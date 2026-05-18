# Xray AOT Transpile-to-C 功能设计方案

> Version: 1.0 | Date: 2026-04-22
> Principle: **No backward compatibility — always pick the best design.**

---

## 一、设计目标

将 xray 源码通过 `XIR SSA → C` 管线编译为独立可执行二进制，支持：

1. **多文件模块系统** — 入口文件 + 所有 import 依赖统一 AOT 编译
2. **类型特化** — typed 路径走 C 原生类型，untyped 走 tagged dispatch
3. **结构化数据** — class/struct/json 统一映射为 C struct
4. **闭包** — 逃逸分析 + 非逃逸内联 + 逃逸堆分配
5. **并发** — coroutine 映射为线程池 + channel
6. **独立二进制** — 不依赖 VM，自含轻量运行时

### 设计选型对比（借鉴四语言）

| 维度 | Nim | V | Zig | Cython | **Xray (选型)** |
|------|-----|---|-----|--------|-----------------|
| 编译单元 | 每模块→1个.c | 所有模块→1个.c | 每函数→1个MIR | 每模块→1个.c | **每模块→1个.c** (Nim式) |
| 类型系统 | 完全静态 | 完全静态 | 完全静态 | 渐进式 | **渐进式** (Cython式) |
| GC | ARC/ORC | 无/手动 | 无 | CPython GC | **ARC** (Nim式) |
| 闭包 | env struct + fn ptr | 同 | N/A | PyObject | **env struct + fn ptr** (Nim式) |
| 并发 | thread+channels | pthread/coroutine | async | GIL | **M:N thread pool** (V式) |
| 异常 | setjmp/goto | option/result | error union | CPython | **setjmp/longjmp** (Nim式) |

---

## 二、整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                     xray build --native                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Phase 1: Dependency Collection                              │
│  ┌──────────┐    ┌───────────┐    ┌──────────────────┐      │
│  │ entry.xr │───→│ XrBundle  │───→│ ordered modules  │      │
│  └──────────┘    │ (existing)│    │ [topo-sorted]    │      │
│                  └───────────┘    └──────────────────┘      │
│                                                              │
│  Phase 2: Per-Module Compilation                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────┐  │
│  │ .xr src  │───→│ XrProto  │───→│ XIR SSA  │───→│ .c   │  │
│  │ (parse)  │    │ (compile)│    │ (optimize)│    │ file │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────┘  │
│     ↑ for each module in topo order                         │
│                                                              │
│  Phase 3: Assembly & Linking                                 │
│  ┌──────────────────────────────────────────┐               │
│  │ xrt_runtime.h + module_*.c + main.c      │               │
│  │         ↓                                │               │
│  │    cc -O2 → standalone binary            │               │
│  └──────────────────────────────────────────┘               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 核心数据结构

```c
// xcgen_module.h - Module-level AOT compilation unit
typedef struct XcgenModuleUnit {
    const char   *module_name;     // "math", "./utils"
    const char   *module_path;     // absolute path
    XcgenBuf      sections[XCGEN_SEC_COUNT];  // per-module C sections
    XcgenFunc    *funcs;           // compiled functions in this module
    int           nfuncs;
    int           funcs_cap;

    // Module-level state
    XcgenExport  *exports;         // exported symbols
    int           nexports;
    int16_t       module_id;       // unique id for cross-module refs
} XcgenModuleUnit;

// Top-level compilation context (replaces XcgenModule)
typedef struct XcgenCompilation {
    XcgenModuleUnit *units;        // all modules being compiled
    int              nunits;
    int              units_cap;

    // Global proto registry (cross-module)
    XcgenProtoEntry *proto_map;
    int              proto_map_count;
    int              proto_map_cap;

    // Global struct promotion registry
    XcgenStructRegistry struct_reg;

    // Global type table (for cross-module type consistency)
    XcgenTypeTable  type_table;

    // Output configuration
    const char     *output_dir;    // nimcache-style: ~/.xray/cache/<hash>/
    bool            emit_debug;    // #line directives
    bool            single_file;   // combine all into one .c (default for small projects)
} XcgenCompilation;
```

---

## 三、Phase 1: 多模块依赖收集

### 3.1 设计（复用 xbundle + 扩展）

当前 `xbundle.c` 已经实现了递归 import 分析和拓扑排序。AOT 需要扩展它：

```c
// New: AOT-specific bundle that preserves AST/Proto for each module
typedef struct XcgenBundleEntry {
    const char *path;
    const char *module_name;
    XrProto    *proto;          // compiled proto tree (not serialized)
    bool        is_stdlib;      // stdlib modules get special treatment
    int         dep_count;
    int        *dep_indices;    // indices into bundle entries
} XcgenBundleEntry;

typedef struct XcgenBundle {
    XcgenBundleEntry *entries;
    int               count;
    int               entry_index;  // main module
} XcgenBundle;
```

### 3.2 关键决策

**Q: stdlib 模块怎么处理？**

借鉴 V 的做法：stdlib C 模块直接链接，不重新编译。

```
用户模块 (math.xr)  → AOT 编译为 C
stdlib 模块 (time)   → 已有 C 实现，直接 #include 或链接
```

**Q: 循环 import？**

xray 当前的 module system 已经有循环检测（`xbundle.c` 的 `visited` hashmap）。
AOT 用拓扑排序确保初始化顺序，不需要额外处理。

---

## 四、Phase 2: 模块级变量与跨模块调用

### 4.1 GETSHARED/SETSHARED 翻译

当前 AOT 忽略了 GETSHARED/SETSHARED。这是跨模块调用的核心。

**设计：每个模块生成一个 init 函数 + 导出变量表**

参考 Nim 的 `datInit` + `init` 两阶段初始化：

```c
// Generated for module "math.xr":
// --- math_exports.c ---

// Module-level variables (top-level let/const bindings)
static XrValue math__pi;          // const pi = 3.14159
static XrValue math__add;         // fn add(a, b) = a + b  (closure value)

// Export table for runtime lookup
static XrtModuleExport math__exports[] = {
    {"pi",  &math__pi,  XRT_EXPORT_CONST},
    {"add", &math__add, XRT_EXPORT_FN},
};
static const int math__nexports = 2;

// Module init (runs once, in topo order)
static void math__init(XrtContext ctx) {
    math__pi = xrt_box_float(3.14159265358979);
    math__add = (XrValue){.tag = XRT_TAG_FN, .ptr = (void*)xr_math_add};
}
```

跨模块引用变为直接 C `extern` 引用：

```c
// In user.c that does: import "./math" as math   (then math.add)
extern XrValue math__add;  // resolved at link time

static void xr_user_main(XrtContext ctx) {
    // call add(1, 2) → direct C call if known function
    int64_t result = xr_math_add(ctx, 1, 2);
}
```

### 4.2 跨模块函数调用策略

```
                    ┌─ typed, same module ──→ direct C call
                    │
function call ──────┼─ typed, cross-module ──→ extern + direct C call
                    │
                    ├─ untyped, known fn ──→ tagged dispatch via xrt_call
                    │
                    └─ dynamic (closure) ──→ fn pointer indirect call
```

**关键优化：** 如果被调函数有类型标注，跨模块调用也走直接 C 调用（extern 声明）。
这是比当前 JIT thunk 方式更好的设计，完全消除间接调用开销。

---

## 五、类型系统与值表示

### 5.1 统一值表示（简化当前设计）

当前 `XrValue` 是 16 字节 tagged union。AOT 保留它作为 untyped 路径，
但 typed 路径直接用 C 原生类型：

```c
// Tagged value (untyped path)
typedef struct {
    union {
        int64_t i;
        double  f;
        void   *ptr;
    };
    uint32_t tag;
    uint32_t _pad;
} XrValue;
// sizeof = 16, same as current

// Type tags (simplified for AOT)
enum {
    XRT_TAG_NULL   = 0,
    XRT_TAG_BOOL   = 1,
    XRT_TAG_I64    = 2,
    XRT_TAG_F64    = 3,
    XRT_TAG_PTR    = 4,   // GC-managed pointer (string, array, object, ...)
};
```

**AOT 类型映射表：**

| xray type | C type (typed path) | C type (untyped path) |
|-----------|--------------------|-----------------------|
| `int` | `int64_t` | `XrValue` |
| `float` | `double` | `XrValue` |
| `bool` | `int64_t` (0/1) | `XrValue` |
| `string` | `XrtStr` | `XrValue` |
| `[T]` (array) | `XrtArray*` | `XrValue` |
| `{K:V}` (map) | `XrtMap*` | `XrValue` |
| `fn(A)->R` | C function pointer | `XrValue` |
| `T` (class) | `XrtObj_T*` | `XrValue` |
| `any` / untyped | `XrValue` | `XrValue` |

### 5.2 Box/Unbox 生成策略

```c
// Typed → Tagged (boxing): only at module boundaries or dynamic dispatch
static inline XrValue xrt_box_int(int64_t v) {
    return (XrValue){.i = v, .tag = XRT_TAG_I64};
}
static inline XrValue xrt_box_float(double v) {
    return (XrValue){.f = v, .tag = XRT_TAG_F64};
}
static inline XrValue xrt_box_ptr(void *p) {
    return (XrValue){.ptr = p, .tag = XRT_TAG_PTR};
}

// Tagged → Typed (unboxing): with type guard
static inline int64_t xrt_unbox_int(XrValue v) {
    XRT_ASSERT(v.tag == XRT_TAG_I64);
    return v.i;
}
```

**关键原则：** 同一函数内部 typed 变量**永远不 box/unbox**。
只在跨类型边界时插入（函数调用参数/返回值类型不同时）。

---

## 六、Class/OOP 翻译

### 6.1 Class → C Struct 映射

借鉴 V 的 struct 生成 + Nim 的 vtable 策略：

```xray
class Animal {
    name: string
    age: int

    fn speak() -> string {
        return this.name + " says hello"
    }
}

class Dog extends Animal {
    fn speak() -> string {
        return this.name + " barks"
    }
    fn fetch() -> string {
        return "fetching..."
    }
}
```

生成的 C 代码：

```c
// --- Type descriptors ---
typedef struct XrtTypeInfo {
    uint16_t type_id;
    uint16_t parent_id;     // 0 = no parent
    const char *name;
    void (**vtable)(void);  // virtual method table
    int vtable_size;
    void (*destructor)(void *);
} XrtTypeInfo;

// --- Instance layout ---
// Header: every object has type_id + refcount for ARC
typedef struct {
    uint32_t type_id;
    uint32_t refcount;      // ARC reference count
} XrtObjHeader;

typedef struct {
    XrtObjHeader hdr;       // type_id = TYPE_ID_ANIMAL
    XrtStr name;
    int64_t age;
} XrtObj_Animal;

typedef struct {
    XrtObjHeader hdr;       // type_id = TYPE_ID_DOG
    XrtStr name;            // inherited fields at same offsets
    int64_t age;
    // no extra fields for Dog
} XrtObj_Dog;

// --- VTable ---
// Slot 0: speak
enum { ANIMAL_VTABLE_SPEAK = 0, ANIMAL_VTABLE_SIZE = 1 };
enum { DOG_VTABLE_SPEAK = 0, DOG_VTABLE_FETCH = 1, DOG_VTABLE_SIZE = 2 };

static XrValue xr_Animal_speak(XrtContext ctx, XrtObj_Animal *self);
static XrValue xr_Dog_speak(XrtContext ctx, XrtObj_Dog *self);
static XrValue xr_Dog_fetch(XrtContext ctx, XrtObj_Dog *self);

static void (*animal_vtable[])(void) = {
    (void(*)(void))xr_Animal_speak,
};
static void (*dog_vtable[])(void) = {
    (void(*)(void))xr_Dog_speak,   // override slot 0
    (void(*)(void))xr_Dog_fetch,   // new slot 1
};

// --- Method dispatch ---
// Known type → direct call (most cases, inferred by type checker)
// Unknown type → vtable dispatch
static inline XrValue xrt_vcall_speak(XrtContext ctx, void *obj) {
    XrtObjHeader *h = (XrtObjHeader *)obj;
    XrtTypeInfo *ti = &xrt_type_table[h->type_id];
    typedef XrValue (*SpeakFn)(XrtContext, void*);
    return ((SpeakFn)ti->vtable[ANIMAL_VTABLE_SPEAK])(ctx, obj);
}
```

### 6.2 类型检查

```c
// instanceof → type_id 比较 (含继承链)
static inline bool xrt_instanceof(void *obj, uint16_t target_type_id) {
    XrtObjHeader *h = (XrtObjHeader *)obj;
    uint16_t tid = h->type_id;
    while (tid != 0) {
        if (tid == target_type_id) return true;
        tid = xrt_type_table[tid].parent_id;
    }
    return false;
}
```

---

## 七、内存管理 (ARC)

### 7.1 设计决策

**选择 ARC（引用计数），不选择 tracing GC。**

理由（参考 Nim 的 ARC/ORC 演进）：
- ARC 的 C 代码生成更简单，retain/release 可以精确插入
- 确定性析构，适合 AOT 独立二进制
- 无需 root set scanning，无 STW 暂停

**循环引用：** 采用 ORC (cycle collector) 按需处理，
或要求用户用 `weak` 标注打破循环（V 的方式）。

### 7.2 ARC 实现

```c
// Object header (for all heap-allocated objects)
typedef struct {
    uint32_t type_id;    // runtime type
    uint32_t refcount;   // reference count, 0 = immortal/stack
} XrtObjHeader;

static inline void xrt_retain(void *obj) {
    if (!obj) return;
    XrtObjHeader *h = (XrtObjHeader *)obj;
    if (h->refcount > 0) h->refcount++;  // 0 = immortal
}

static inline void xrt_release(void *obj) {
    if (!obj) return;
    XrtObjHeader *h = (XrtObjHeader *)obj;
    if (h->refcount == 0) return;  // immortal
    if (--h->refcount == 0) {
        xrt_type_table[h->type_id].destructor(obj);
        xrt_dealloc(obj);
    }
}

// For XrValue (tagged):
static inline void xrt_retain_val(XrValue v) {
    if (v.tag == XRT_TAG_PTR && v.ptr) xrt_retain(v.ptr);
}
static inline void xrt_release_val(XrValue v) {
    if (v.tag == XRT_TAG_PTR && v.ptr) xrt_release(v.ptr);
}
```

### 7.3 生成的代码示例

```c
static void xr_example(XrtContext ctx, XrtStr name) {
    xrt_retain(name.ptr);              // retain param
    XrtObj_Animal *a = xrt_new_Animal(ctx);
    a->name = name;
    xrt_retain(name.ptr);             // retained by field assignment
    // ... use a ...
    xrt_release(a);                    // release at scope end
    xrt_release(name.ptr);            // release param
}
```

**优化：** move semantics — 如果编译器知道变量最后一次使用（last use），
省略 retain + 在最后使用处 sink（不 release），直接移动所有权。
这是 Nim ARC 的核心优化，XIR SSA 形式天然支持 last-use 分析。

---

## 八、闭包翻译

### 8.1 统一闭包模型

**所有闭包** = env struct + function pointer（参考 Nim `ClEnv`）。

```xray
fn make_adder(x: int) -> fn(int) -> int {
    return fn(y: int) -> int { return x + y }
}
```

生成：

```c
// Closure environment (ARC-managed)
typedef struct {
    XrtObjHeader hdr;      // for ARC
    int64_t x;             // captured variable
} XrtEnv_adder_lambda;

// The inner function
static int64_t xr_adder_lambda(XrtContext ctx, XrtEnv_adder_lambda *env, int64_t y) {
    return env->x + y;
}

// Closure value = { fn_ptr, env_ptr }
typedef struct {
    void *fn;
    void *env;
} XrtClosure;

static XrValue xr_make_adder(XrtContext ctx, int64_t x) {
    XrtEnv_adder_lambda *env = xrt_alloc(sizeof(XrtEnv_adder_lambda));
    env->hdr.type_id = TYPE_ID_CLOSURE_ENV;
    env->hdr.refcount = 1;
    env->x = x;
    return xrt_box_closure((XrtClosure){
        .fn = (void*)xr_adder_lambda,
        .env = env
    });
}
```

### 8.2 逃逸分析优化（保留当前设计）

当前 xcgen 的闭包逃逸分析是一个亮点，保留并增强：

- **非逃逸闭包** → env 在栈上，upvalue 作为直接参数传递（零分配）
- **逃逸闭包** → env 在堆上，ARC 管理
- **新增：部分逃逸** → env 在栈上，但如果逃逸则 COW (copy-on-write) 到堆

---

## 九、异常处理

### 9.1 setjmp/longjmp 方案（Nim 式）

```c
// Thread-local exception state
typedef struct {
    jmp_buf     buf;
    XrValue     exception;   // current exception value
    const char *file;
    int         line;
} XrtExceptionFrame;

// Per-thread exception stack
_Thread_local XrtExceptionFrame *xrt_exc_stack = NULL;
_Thread_local int xrt_exc_depth = 0;

#define XRT_TRY(frame) \
    do { \
        (frame).prev = xrt_exc_stack; \
        xrt_exc_stack = &(frame); \
        xrt_exc_depth++; \
        if (setjmp((frame).buf) == 0) {

#define XRT_CATCH \
        } else {

#define XRT_END_TRY \
        } \
        xrt_exc_stack = (frame).prev; \
        xrt_exc_depth--; \
    } while(0)

// Throw: longjmp to nearest catch
static _Noreturn void xrt_throw(XrValue exc, const char *file, int line) {
    if (!xrt_exc_stack) {
        fprintf(stderr, "Uncaught exception at %s:%d\n", file, line);
        abort();
    }
    xrt_exc_stack->exception = exc;
    xrt_exc_stack->file = file;
    xrt_exc_stack->line = line;
    longjmp(xrt_exc_stack->buf, 1);
}
```

### 9.2 生成的代码

```xray
try {
    let x = might_fail()
} catch e {
    println(e.message)
}
```

→

```c
{
    XrtExceptionFrame _ef;
    XRT_TRY(_ef) {
        XrValue x = xr_might_fail(ctx);
    } XRT_CATCH {
        XrValue e = _ef.exception;
        xrt_println(ctx, xrt_get_message(e));
    } XRT_END_TRY;
}
```

---

## 十、并发模型

### 10.1 设计选型

借鉴 V 的 `spawn` → pthread / `go` → coroutine 双模型：

```
go func(args)  →  提交到线程池 (M:N 调度)
channel         →  线程安全 MPMC 队列
```

### 10.2 Go 表达式翻译

```xray
go compute(data)
```

→

```c
// Wrapper struct for arguments (V 的 thread_arg_ 模式)
typedef struct {
    // captured arguments
    XrValue data;
} XrtGoArgs_compute;

static void xrt_go_wrapper_compute(void *raw) {
    XrtGoArgs_compute *args = (XrtGoArgs_compute *)raw;
    XrtContext ctx = xrt_thread_ctx();
    xr_compute(ctx, args->data);
    xrt_release_val(args->data);  // ARC release captured args
    xrt_dealloc(args);
}

// At call site:
XrtGoArgs_compute *_go_args = xrt_alloc(sizeof(XrtGoArgs_compute));
_go_args->data = data;
xrt_retain_val(data);  // ARC retain for async ownership
xrt_pool_submit(xrt_go_wrapper_compute, _go_args);
```

### 10.3 Channel 翻译

```c
// Channel: bounded MPMC queue (lock-free or mutex-based)
typedef struct {
    XrtObjHeader hdr;
    XrValue     *buffer;
    uint32_t     capacity;
    _Atomic uint32_t head;
    _Atomic uint32_t tail;
    pthread_mutex_t  mtx;
    pthread_cond_t   not_empty;
    pthread_cond_t   not_full;
    bool             closed;
} XrtChannel;

// ch <- value
static void xrt_chan_send(XrtChannel *ch, XrValue val) {
    pthread_mutex_lock(&ch->mtx);
    while (xrt_chan_is_full(ch) && !ch->closed)
        pthread_cond_wait(&ch->not_full, &ch->mtx);
    if (ch->closed) { pthread_mutex_unlock(&ch->mtx); xrt_throw(...); }
    ch->buffer[ch->tail % ch->capacity] = val;
    xrt_retain_val(val);
    ch->tail++;
    pthread_cond_signal(&ch->not_empty);
    pthread_mutex_unlock(&ch->mtx);
}

// value = <-ch
static XrValue xrt_chan_recv(XrtChannel *ch) {
    pthread_mutex_lock(&ch->mtx);
    while (xrt_chan_is_empty(ch) && !ch->closed)
        pthread_cond_wait(&ch->not_empty, &ch->mtx);
    if (ch->closed && xrt_chan_is_empty(ch)) {
        pthread_mutex_unlock(&ch->mtx);
        return xrt_null();
    }
    XrValue val = ch->buffer[ch->head % ch->capacity];
    ch->head++;
    pthread_cond_signal(&ch->not_full);
    pthread_mutex_unlock(&ch->mtx);
    return val;  // ownership transferred, no extra retain
}
```

---

## 十一、AOT Runtime (`xrt.h`)

### 11.1 运行时层次

```
xrt_core.h      — 值表示、box/unbox、类型 tag
xrt_arc.h       — ARC retain/release
xrt_string.h    — 不可变字符串 (SSO + ARC)
xrt_array.h     — 动态数组 (typed: T*, untyped: XrValue*)
xrt_map.h       — hash map (open addressing)
xrt_channel.h   — MPMC bounded channel
xrt_pool.h      — 线程池
xrt_exception.h — setjmp/longjmp 异常
xrt_io.h        — println, readln, file I/O
xrt_type.h      — 运行时类型信息 (vtable, instanceof)
xrt_module.h    — 模块初始化/导出
```

### 11.2 关键设计决策

**String: SSO (Small String Optimization)**

```c
typedef struct {
    union {
        struct {            // heap string
            char    *ptr;
            uint32_t len;
            uint32_t cap;
        };
        char sso[16];       // inline for strings ≤ 15 bytes
    };
    uint8_t flags;          // bit 0: is_sso, bit 1: is_static
} XrtStr;
```

**Array: Typed + Homogeneous**

```c
typedef struct {
    XrtObjHeader hdr;
    void    *data;
    uint32_t len;
    uint32_t cap;
    uint8_t  elem_type;     // XRT_ELEM_I64 / XRT_ELEM_F64 / XRT_ELEM_PTR / XRT_ELEM_VAL
    uint8_t  elem_size;     // sizeof element
} XrtArray;
```

---

## 十二、输出文件结构

### 12.1 小型项目（单文件输出，默认）

```
/tmp/xray_aot_<hash>/
├── app.c               # 合并所有模块的 C 代码
└── (compile) → a.out
```

### 12.2 大型项目（多文件输出）

```
~/.xray/cache/<project_hash>/
├── xrt_runtime.h       # 自含运行时头文件
├── xrt_runtime.c       # 运行时实现
├── mod_main.c          # 入口模块
├── mod_math.c          # math 模块
├── mod_utils.c         # utils 模块
├── mod_init.c          # 模块初始化序列 + main()
├── Makefile            # 并行编译
└── (make -j8) → app
```

### 12.3 main() 生成

```c
// mod_init.c
#include "xrt_runtime.h"

// Forward declarations of all module inits (topo order)
static void mod_math__init(XrtContext ctx);
static void mod_utils__init(XrtContext ctx);
static void mod_main__init(XrtContext ctx);

int main(int argc, char **argv) {
    XrtRuntime *rt = xrt_runtime_new();
    xrt_runtime_set_args(rt, argc, argv);

    XrtContext ctx = xrt_context_new(rt);

    // Initialize modules in dependency order
    mod_math__init(ctx);
    mod_utils__init(ctx);
    mod_main__init(ctx);    // entry point runs here

    int result = xrt_runtime_result(rt);
    xrt_context_free(ctx);
    xrt_runtime_free(rt);
    return result;
}
```

---

## 十三、实现分阶段计划

### Phase A: 基础重构 (1-2 weeks)

**目标：** 多模块编译管线，消除 VM 依赖

1. **重构 `XcgenModule` → `XcgenCompilation`**
   - 支持多个 `XcgenModuleUnit`
   - 全局 `proto_map` 跨模块共享
   - 移除 `MAX_AOT_PROTOS = 64` 限制

2. **扩展 `cmd_build_native`**
   - 使用 `XrBundle` 收集所有依赖模块
   - 对每个模块调用 `collect_aot_protos` + `xcgen_compile_func`
   - 生成模块初始化函数

3. **实现 GETSHARED/SETSHARED → C 全局变量**
   - 每模块一个 `static XrValue mod_X__varname;`
   - GETSHARED → `extern` 引用
   - SETSHARED → 赋值

4. **新 AOT Runtime**
   - 提取 `xrt.h` 为独立运行时（不依赖 VM headers）
   - ARC retain/release 实现（替换当前 no-op）
   - 基础 `XrtStr`, `XrtArray`, `XrtMap`

### Phase B: 类型系统完善 (2-3 weeks)

**目标：** class/OOP, 异常, 完整类型特化

5. **Class → C Struct**
   - `XrtObjHeader` + 字段布局
   - vtable 生成
   - `new` → `xrt_alloc` + init

6. **异常 setjmp/longjmp**
   - `XrtExceptionFrame` 栈
   - try/catch/throw 翻译

7. **完善类型特化**
   - 拆分 `xcg_emit_instruction()` 超长函数
   - String/Array/Map 方法内联
   - 提取 MOV chain 追踪为公共函数

### Phase C: 并发支持 (2-3 weeks)

**目标：** go + channel

8. **线程池**
   - `XrtPool`: 固定大小 worker 线程 + 任务队列
   - `go expr` → 参数捕获 + pool submit

9. **Channel**
   - `XrtChannel`: bounded MPMC queue
   - send/recv 翻译

10. **并发安全**
    - 参数深拷贝 (go 表达式的值语义)
    - shared 类型 → 原子操作

### Phase D: 完善 & 优化 (2-3 weeks)

**目标：** stdlib, 多文件输出, 增量编译

11. **Stdlib 桥接**
    - 生成 stdlib 函数的 extern 声明
    - 链接 stdlib `.a`/`.o` 文件

12. **多文件输出**
    - 每模块独立 `.c` 文件
    - Makefile 生成
    - 增量编译（基于文件 hash）

13. **优化**
    - Move semantics (ARC last-use 优化)
    - 内联小函数
    - 常量折叠提升

---

## 十四、废弃项（不再需要）

基于"不考虑向后兼容"原则，以下现有设施将被替换：

| 废弃 | 替代 | 理由 |
|------|------|------|
| JIT thunk 注册机制 (`xr_proto_set_jit_entry`) | 直接 C extern 调用 | AOT 不需要运行时 patch |
| `xrt_invoke_method_sentinel` | vtable dispatch | class 有 vtable, 其他用 trait |
| VM `XrCoroutine` context passing | `XrtContext` (thin wrapper) | 不依赖 VM 协程 |
| `xr_json_new_with_shape` struct promotion | 直接 C struct 生成 | AOT 编译期确定所有类型 |
| `XRT_TAG_F64/I64` NaN boxing (当前 VM) | 16B tagged union | AOT 用显式 tag, 更简单 |
| Bytecode bundle fallback | 纯 AOT 或报错 | 不要半 VM 半 AOT |

---

## 十五、与参考语言的关键差异

### vs Nim
- Nim 直接 AST → C，xray 经过 XIR SSA（更适合优化）
- Nim 有完整 ORC cycle collector，xray 先用简单 ARC + weak ref
- Nim 的 codegen 是 2600+ 行单文件，xray 拆分为多个子模块（更好）

### vs V
- V 是 AST → C 没有中间优化层，xray 有 XIR 优化管线
- V 所有模块合一个 .c 文件（简单但不支持增量），xray 支持拆分
- V 的 `go` 用 photon coroutine，xray 用线程池（更简单可靠）

### vs Zig
- Zig C backend 非常底层（IR → C），主要为 bootstrap
- xray 的 C backend 是主要输出目标，需要更好的可读性

### vs Cython
- Cython 依赖 CPython runtime，xray 独立运行时
- Cython 用 `PyObject*` 做 untyped，xray 用 `XrValue` (更轻量)

---

## 十六、测试策略

```
tests/aot/
├── basic/           # 基本类型、运算、控制流
├── modules/         # 多模块 import/export
├── classes/         # OOP、继承、vtable
├── closures/        # 逃逸分析、闭包调用
├── exceptions/      # try/catch/throw
├── concurrency/     # go + channel
├── stdlib/          # stdlib 函数调用
└── regression/      # 回归测试

# 每个测试: .xr 源文件 + .expected 输出
# 运行: xray build --native test.xr -o test && ./test
```
