# xray Transpile-to-C 完整方案

> 设计原则：**不考虑向后兼容性**，全新语言，直接采用最佳设计

## 1. 现状与参考分析

### 1.1 xray 现有编译管线

```
源代码 (.xr)
    ↓
Parser → AST → Analyzer (类型推断)
    ↓
Compiler → Bytecode (XrProto)
    ↓                        ↓
  解释器 (VM)           JIT Compiler (热函数)
    ↓                        ↓
  tagged XrValue        XIR (SSA IR)
                             ↓
                   xir_pass (DCE/ConstProp/GVN/LICM)
                             ↓
                   xir_codegen → ARM64 native (JIT)
                             ↓
                   xir_transpile_c → C source (AOT)  ← 当前仅支持纯计算
```

### 1.2 现有 transpiler 能力 (`xir_transpile_c.c`, ~460 行)

| 已支持 | 未支持 |
|--------|--------|
| 算术/逻辑运算 (ADD/SUB/MUL...) | 字符串操作 |
| 浮点运算 (FADD/FSUB...) | 数组/Map 操作 |
| 整型/浮点比较 | 对象/类字段访问 |
| 类型转换 (I2F/F2I) | GC 集成 (BOX/UNBOX/BARRIER) |
| 常量/MOV/NOP | 协程/Channel |
| 自递归调用 (CALL_SELF_DIRECT) | 通用函数调用 (CALL/CALL_KNOWN) |
| PHI 节点 (SSA→C变量) | 异常处理 (TRY/CATCH/THROW) |
| 条件分支/跳转/返回 | 闭包/Upvalue |
| | 运行时类型分发 (RT_ADD 等) |

### 1.3 Nim 与 V 的 C codegen 架构对比

| 维度 | Nim | V | xray (目标) |
|------|-----|---|------------|
| **输入** | AST | AST | **XIR (SSA IR)** |
| **类型系统** | 静态强类型 | 静态强类型 | 渐进式类型 (静态+动态) |
| **GC** | 多种 (refcount/mark-sweep/ORC) | autofree + GC | 三色标记 GC |
| **并发** | async/await → 状态机 | spawn(pthread) + go(photon) | 协程 + Channel |
| **异常** | setjmp/longjmp | Option/Result | try/catch + defer |
| **闭包** | struct{fn,env} | struct{fn,env} | Upvalue 链表 |
| **C文件组织** | 多 section (headers/types/procs/data) | 多 Builder (30+) | 单 section (简化) |
| **名称修饰** | Itanium ABI 风格 | `no_dots()` 简单替换 | `xr_` 前缀 |
| **代码行数** | ~12000 行 (ccg*.nim) | ~30000 行 (gen/c/) | 目标 ~3000 行 |

### 1.4 关键洞察

**Nim/V 从 AST 直接生成 C，xray 从 XIR (SSA IR) 生成 C。这是本质区别。**

- Nim/V 需要在 codegen 中处理作用域、变量遮蔽、表达式求值顺序等
- xray 的 XIR 已经把这些全部解决了：SSA 保证单赋值，基本块保证线性控制流
- 生成的 C 代码是低级的 (labels + goto)，但 cc -O2 完全能优化

**优势**：
1. XIR 已经过优化 (DCE/GVN/LICM)，生成的 C 质量高
2. XIR 的类型信息已解析 (i64/f64/ptr/tagged)，C 类型映射简单
3. 同一套 XIR 优化管线服务 JIT 和 AOT
4. 新增 XIR opcode = JIT 和 AOT 同时受益

---

## 2. 架构设计

### 2.1 总体架构

```
                    ┌─────────────┐
                    │  源代码 .xr  │
                    └──────┬──────┘
                           ↓
                    Frontend (parser + analyzer + compiler)
                           ↓
                    XrProto (bytecode + type info)
                           ↓
              ┌────────────┼────────────┐
              ↓            ↓            ↓
          解释器(VM)   JIT Runtime   AOT Build
              │            │            │
              │      ┌─────┴─────┐      │
              │      │ XIR Build │      │
              │      │ + Passes  │      │
              │      └─────┬─────┘      │
              │            ↓            ↓
              │     ┌──────┴──────┬─────────────┐
              │     │ ARM64 Code  │ C Transpiler │
              │     │  (JIT)      │   (AOT)      │
              │     └─────────────┴──────┬──────┘
              │                          ↓
              │                   cc -O2 → native binary
              ↓                          ↓
          解释执行              原生执行 (链接 libxray_rt)
```

### 2.2 模块划分（方案 B：三目录拆分）

将现有 `src/jit/` 按职责拆为三个独立目录：

```
src/xir/                    # ═══ 共享 IR 基础设施 ═══
├── xir.h/c                 # XIR 数据结构
├── xir_builder.h/c         # Bytecode → XIR 翻译
├── xir_pass.h/c            # 优化管线 (DCE/GVN/LICM...)
├── xir_printer.h/c         # XIR 打印/调试输出
├── xir_tfa.h/c             # 类型流分析
└── xir_feedback.h/c        # 类型反馈收集

src/jit/                    # ═══ JIT 运行时编译 ═══
├── xir_jit.h/c             # JIT 入口、热函数检测
├── xir_codegen.h/c         # XIR → ARM64 代码生成
├── xir_arm64.h/c           # ARM64 指令编码
├── xir_code_alloc.h/c      # JIT 代码内存分配
├── xir_offsets.h           # VM 结构体偏移定义
└── xir_offsets_verify.c    # 偏移验证

src/aot/                    # ═══ AOT transpile-to-C ═══
├── xcgen.h                 # C codegen 主入口
├── xcgen.c                 # C codegen 主控逻辑
├── xcgen_expr.c            # 表达式翻译
├── xcgen_stmt.c            # 语句/控制流翻译
├── xcgen_call.c            # 函数调用翻译
├── xcgen_type.c            # 类型映射 + 值操作
└── xcgen_runtime.c         # Runtime 交互代码生成
```

**依赖方向**：`src/jit/` → `src/xir/` ← `src/aot/`（共享 XIR，互不依赖）

需要删除的旧文件（Phase 1 执行）：
- `src/jit/xir_transpile_c.h/c` — 被 `src/aot/xcgen` 替代
- `src/jit/xir_codegen_aot.c` — ARM64 AOT，不再使用
- `src/jit/xir_aot.h/c` — ARM64 AOT 模块管理，不再使用

### 2.3 命名规范

| 前缀 | 含义 | 示例 |
|------|------|------|
| `xcgen_` | C codegen 公共 API | `xcgen_compile()` |
| `xcg_` | C codegen 内部函数 | `xcg_emit_binary_op()` |
| `xrt_` | 生成的 C 代码中调用的 runtime API | `xrt_string_concat()` |
| `xr_` | xray 现有 runtime 函数 | `xr_channel_send()` |

---

## 3. 核心数据结构

### 3.1 C Codegen 上下文

```c
// xcgen.h

typedef struct XcgenBuf {
    char  *data;
    size_t len;
    size_t cap;
} XcgenBuf;

// C file sections (inspired by Nim's TCFileSection)
typedef enum {
    XCGEN_SEC_HEADERS,       // #include directives
    XCGEN_SEC_FORWARD,       // forward declarations
    XCGEN_SEC_TYPES,         // struct/typedef
    XCGEN_SEC_DATA,          // static data (string literals, constants)
    XCGEN_SEC_FUNCS,         // function bodies
    XCGEN_SEC_MAIN,          // main() wrapper
    XCGEN_SEC_COUNT,
} XcgenSection;

// Per-function codegen state
typedef struct XcgenFunc {
    XirFunc       *xfunc;          // source XIR function
    const char    *c_name;         // generated C function name (e.g. "xr_fib")
    XcgenBuf       body;           // function body buffer
    int            tmp_count;      // temp variable counter
    bool           needs_runtime;  // true if function calls runtime APIs
    bool           needs_gc;       // true if function allocates GC objects
} XcgenFunc;

// Module-level codegen state
typedef struct XcgenModule {
    XcgenBuf       sections[XCGEN_SEC_COUNT];
    XcgenFunc     *funcs;          // all functions being compiled
    int            nfuncs;

    // String literal dedup table
    struct { const char *str; int id; } *string_table;
    int            nstrings;

    // Compilation flags
    bool           standalone;     // true = --native (no runtime)
    bool           emit_debug;     // true = #line directives
    const char    *opt_flag;       // "-O2" etc.
} XcgenModule;
```

### 3.2 类型映射

```c
// xcgen_type.c

// XIR type → C type string
// Extends current c_type_for_xir() with tagged value support
static const char *xcg_c_type(uint8_t xir_type) {
    switch (xir_type) {
        case XIR_TYPE_I64:    return "int64_t";
        case XIR_TYPE_F64:    return "double";
        case XIR_TYPE_PTR:    return "XrObject*";
        case XIR_TYPE_TAGGED: return "XrValue";
        case XIR_TYPE_VOID:   return "void";
        default:              return "int64_t";
    }
}
```

runtime `XrValue` 已经是 16B tagged-union 值布局；AOT 侧使用独立的
standalone `XrtValue` 表示，并通过源码级命名别名复用部分源码级名字，
不做任何 NaN-boxing 兼容。

---

## 4. 分阶段实施计划

### Phase 1: 重构基础框架（替换现有 transpiler）

**目标**：用新的 `xcgen` 模块替换 `xir_transpile_c.c`，功能等价但架构可扩展。

**工作内容**：
1. 新建 `xcgen.h/c` 模块框架
2. 迁移现有纯计算翻译逻辑到新模块
3. 实现 section-based C 文件生成
4. 实现 forward declaration 自动生成
5. 更新 `cmd_build.c` 调用新 API
6. 删除 `xir_transpile_c.h/c`

**文件清单**：
| 文件 | 操作 | 行数估算 |
|------|------|---------|
| `src/xir/` 目录 | 新建，从 `src/jit/` 迁移 12 个共享 IR 文件 | 0 (纯迁移) |
| `src/aot/xcgen.h` | 新建 | ~120 |
| `src/aot/xcgen.c` | 新建 | ~300 |
| `src/aot/xcgen_expr.c` | 新建 | ~250 |
| `src/aot/xcgen_stmt.c` | 新建 | ~150 |
| `src/jit/xir_transpile_c.h/c` | 删除 | -500 |
| `src/jit/xir_aot.h/c` | 删除 | -200 |
| `src/jit/xir_codegen_aot.c` | 删除 | -300 |
| `CMakeLists.txt` | 修改，新增 xir/aot 源文件路径 | ~30 |
| `src/cli/cmd_build.c` | 修改 | ~20 |

**验证**：fib(40) --native 输出结果和性能不变。

---

### Phase 2: 通用函数调用 + 字符串

**目标**：`--native` 支持多函数互调和字符串操作。

#### 4.2.1 函数调用

当前只支持 `XIR_CALL_SELF_DIRECT` (自递归)。需要扩展：

```
XIR_CALL_SELF_DIRECT → 已有：直接 C 递归调用
XIR_CALL_KNOWN       → 新增：已知 callee proto 的直接 C 调用
XIR_CALL             → 新增：通用调用 (通过 runtime 分发)
XIR_CALL_C           → 新增：调用 C runtime 函数
```

**生成策略（借鉴 Nim 的 ccgcalls.nim）**：

```c
// XIR_CALL_KNOWN: callee 的 XIR 也被 transpile 了
// 直接生成 C 函数调用，参数类型匹配
v5 = xr_helper(v2, v3);

// XIR_CALL: 通用调用 (callee 可能是闭包、method 等)
// 通过 runtime dispatch
v5 = xrt_call(closure, 2, (XrValue[]){v2, v3});

// XIR_CALL_C: 调用已知的 C runtime 函数
// 直接内联调用
v5 = xr_string_concat(v2, v3);
```

**多函数编译**：

所有 AOT 候选函数编译到同一个 C 文件，共享 forward declarations。
函数间调用直接使用 C 函数名，cc 处理链接。

```c
// Generated C file structure:
#include <stdint.h>
#include <xray_runtime.h>   // only if needs_runtime

// Forward declarations
static int64_t xr_fib(int64_t v0);
static int64_t xr_helper(int64_t v0, int64_t v1);

// Function bodies
static int64_t xr_fib(int64_t v0) { ... }
static int64_t xr_helper(int64_t v0, int64_t v1) { ... }

// Entry point
int main(int argc, char **argv) { ... }
```

#### 4.2.2 字符串操作

xray 字符串是 GC 管理的不可变对象 (`XrString`)。
Transpile 到 C 时，通过 runtime API 操作：

```c
// XIR 中的字符串操作 → C runtime 调用
XrValue v5 = xrt_string_concat(v2, v3);    // a + b
XrValue v6 = xrt_string_len(v2);           // a.length
int64_t v7 = xrt_string_eq(v2, v3);        // a == b
XrValue v8 = xrt_string_slice(v2, i, j);   // a.slice(i, j)
```

**Runtime thin wrapper (`xray_runtime.h`)**：

```c
// 精简的 runtime header，只暴露 AOT 需要的 API
// 内部调用现有的 xr_string_* 函数
typedef struct XrValue XrValue;
typedef struct XrString XrString;

XrValue xrt_string_concat(XrValue a, XrValue b);
XrValue xrt_string_len(XrValue s);
int64_t xrt_string_eq(XrValue a, XrValue b);
```

**文件清单**：
| 文件 | 操作 | 行数估算 |
|------|------|---------|
| `src/aot/xcgen_call.c` | 新建 | ~300 |
| `src/aot/xcgen_expr.c` | 修改 | +100 |
| `include/xray_runtime.h` | 新建 | ~150 |
| `src/api/xray_runtime.c` | 新建 | ~200 |

**验证**：
- 多函数互调测试用例
- 字符串拼接/比较测试用例
- 性能对比：JIT vs native

---

### Phase 3: 数组/Map + 对象字段访问

**目标**：`--native` 支持容器操作和对象访问。

#### 4.3.1 数组操作

```c
// XIR_LOAD_FIELD / XIR_STORE_FIELD on array
XrValue v5 = xrt_array_get(arr, idx);
xrt_array_set(arr, idx, val);
int64_t v6 = xrt_array_len(arr);
XrValue v7 = xrt_array_push(arr, val);
```

#### 4.3.2 Map 操作

```c
XrValue v5 = xrt_map_get(map, key);
xrt_map_set(map, key, val);
int64_t v6 = xrt_map_len(map);
```

#### 4.3.3 对象字段访问

两种策略（根据类型信息选择）：

```c
// 静态路径：编译期已知字段偏移（类型标注 + shape 固定）
// 借鉴 Nim 的 dotField(accessor, field.loc.snippet)
int64_t v5 = ((XrInstance*)obj)->fields[3].i;  // field offset known

// 动态路径：运行时字段查找
XrValue v5 = xrt_field_get(obj, "name");
xrt_field_set(obj, "name", val);
```

**文件清单**：
| 文件 | 操作 | 行数估算 |
|------|------|---------|
| `src/aot/xcgen_expr.c` | 修改 | +200 |
| `include/xray_runtime.h` | 修改 | +80 |
| `src/api/xray_runtime.c` | 修改 | +300 |

**验证**：数组/Map/对象 CRUD 测试用例。

---

### Phase 4: Tagged Value 操作 (BOX/UNBOX/RT_*)

**目标**：支持动态类型值的操作，使 `any` 类型的函数也能 AOT。

这是从"纯类型函数 AOT"扩展到"全功能 AOT"的关键步骤。

#### 4.4.1 BOX/UNBOX

```c
// XIR_BOX_I64: raw int64 → XrValue
XrValue v5 = xrt_box_int(v2);
// 内联展开（Tagged Union 16B）:
XrValue v5 = (XrValue){ .i = v2, .tag = XR_TAG_INT };

// XIR_UNBOX_I64: XrValue → raw int64 (with tag check)
int64_t v6 = xrt_unbox_int(v5);
// 内联展开:
assert(v5.tag == XR_TAG_INT);
int64_t v6 = v5.i;
```

#### 4.4.2 Runtime 混合类型操作

```c
// XIR_RT_ADD: mixed-type addition
XrValue v5 = xrt_add(v2, v3);
// Runtime 内部根据 tag 分发:
//   int + int → int
//   int + float → float
//   string + string → string concat
//   其他 → type error
```

#### 4.4.3 Guard + Deopt

```c
// XIR_GUARD_TAG: 类型守卫
if (v2.tag != XR_TAG_INT) goto deopt_L3;

// XIR_DEOPT: 退化点
// --native 模式下不支持 deopt（回退解释器）
// 改为 runtime error:
deopt_L3:
    xrt_type_error("expected int, got %s", xrt_tag_name(v2.tag));
```

**文件清单**：
| 文件 | 操作 | 行数估算 |
|------|------|---------|
| `src/aot/xcgen_type.c` | 新建 | ~250 |
| `src/aot/xcgen_expr.c` | 修改 | +150 |
| `include/xray_runtime.h` | 修改 | +60 |

---

### Phase 5: GC 集成

**目标**：AOT 生成的代码与 xray GC 正确交互。

#### 4.5.1 Safepoint

```c
// XIR_SAFEPOINT → GC safepoint poll
// 在循环回边和函数调用前插入
xrt_safepoint();
// 内联展开:
if (__builtin_expect(xr_gc_should_collect, 0)) {
    xrt_gc_collect();
}
```

#### 4.5.2 Write Barrier

```c
// XIR_BARRIER_FWD → forward write barrier
// 当把新对象存入老对象的字段时
xrt_write_barrier(parent, child);
```

#### 4.5.3 对象分配

```c
// XIR_ALLOC → GC bump allocation
XrObject *v5 = xrt_alloc(sizeof(XrInstance) + nfields * sizeof(XrValue));
```

**设计要点**（借鉴 Nim 的 ccgtrav.nim）：
- Nim 为每个类型生成专门的 GC 遍历函数
- xray 不需要：xray 的 GC 通过 tag 自动识别引用类型
- AOT 代码只需在适当位置调用 safepoint 和 write barrier

**文件清单**：
| 文件 | 操作 | 行数估算 |
|------|------|---------|
| `src/aot/xcgen_runtime.c` | 新建 | ~200 |
| `include/xray_runtime.h` | 修改 | +40 |

---

### Phase 6: 异常处理 + defer

**目标**：`--native` 支持 try/catch/throw 和 defer。

#### 4.6.1 异常处理

**策略：setjmp/longjmp**（与 Nim 相同，V 也是类似模式）

```c
// XIR_TRY_BEGIN
jmp_buf _try_buf_1;
if (setjmp(_try_buf_1) == 0) {
    // try body
    // XIR_TRY_END
} else {
    // XIR_CATCH
    XrValue _exception = xrt_get_current_exception();
    // catch body
}

// XIR_THROW
xrt_throw(exception_value);
// 内部: longjmp(current_try_buf, 1)
```

#### 4.6.2 defer

**策略：flag 变量**（与 V 的 `defer.v` 相同）

```c
// defer { cleanup() }
bool _defer_0 = false;

// 在 defer 声明点标记
_defer_0 = true;

// 在函数返回/异常前执行
if (_defer_0) { cleanup(); }
```

**文件清单**：
| 文件 | 操作 | 行数估算 |
|------|------|---------|
| `src/aot/xcgen_stmt.c` | 修改 | +200 |
| `include/xray_runtime.h` | 修改 | +30 |

---

### Phase 7: 闭包 + Upvalue

**目标**：`--native` 支持闭包和变量捕获。

**策略**（与 Nim/V 相同：struct{fn, env}）：

```c
// 闭包结构
typedef struct {
    void *fn;            // function pointer
    XrUpvalueBox *env;   // captured variables
} XrClosure;

// XIR_LOAD_UPVAL
XrValue v5 = closure->env->values[upval_index];

// XIR_STORE_UPVAL
closure->env->values[upval_index] = v5;

// 闭包调用
XrValue result = ((XrClosureFn)closure->fn)(closure->env, arg0, arg1);
```

**文件清单**：
| 文件 | 操作 | 行数估算 |
|------|------|---------|
| `src/aot/xcgen_call.c` | 修改 | +150 |
| `include/xray_runtime.h` | 修改 | +30 |

---

### Phase 8: 协程 + Channel

**目标**：`--native` 支持 `spawn` 和 Channel 通信。

这是复杂度最高的部分，也是 transpile-to-C 方案的最大优势所在。

#### 4.8.1 协程

**策略**：调用 runtime API（协程调度器已是 C 实现）

```c
// spawn worker(ch)
xrt_spawn(xr_worker, (XrValue[]){ch}, 1);

// yield (在协程内部)
xrt_yield();
```

**当前 VM runtime 的协程模型**：
xray 协程在 native/C 栈层面是 stackless 的；每个协程拥有独立的 VM value stack 和 bytecode frame array。
阻塞与恢复通过 VM frame、continuation、JIT suspend state 和调度器状态完成，不通过 `setjmp/longjmp` 切换 C 栈。
因此 AOT 并发若要复用当前调度器，需要生成可恢复状态，不能假设存在 native stackful yield。

#### 4.8.2 Channel

```c
// ch.send(value)
xrt_channel_send(ch, value);

// let x = ch.recv()
XrValue x = xrt_channel_recv(ch);
```

#### 4.8.3 V 的 spawn 实现参考

V 的 `spawn_and_go.v` 为每个 spawn 调用点生成：
1. wrapper struct（包含函数指针 + 所有参数）
2. wrapper function（解包参数，调用目标函数）
3. `pthread_create` 调用

xray 更简单：协程由 runtime 调度器管理，spawn 只需将函数 + 参数入队：

```c
// V 的方式（重量级）:
typedef struct thread_arg_worker {
    void (*fn)(XrValue);
    XrValue arg0;
} thread_arg_worker;
void* worker_thread_wrapper(thread_arg_worker *arg) { arg->fn(arg->arg0); }
pthread_create(&thread, NULL, worker_thread_wrapper, arg);

// xray 的方式（轻量级）:
xrt_spawn(worker_fn_ptr, args, nargs);
// runtime 内部：创建协程，加入调度队列
```

**文件清单**：
| 文件 | 操作 | 行数估算 |
|------|------|---------|
| `src/aot/xcgen_runtime.c` | 修改 | +150 |
| `include/xray_runtime.h` | 修改 | +30 |

---

### Phase 9: 模块 import + 多文件工程

**目标**：`xray build --native` 支持多文件工程。

#### 4.9.1 编译期解析

在 AOT 编译时，静态解析所有 `import`，收集全部函数：

```
project/
├── main.xr       → import "utils"
├── utils.xr      → export fn helper()
└── build output:
    └── app.c      → xr_main() + xr_helper() + main()
```

所有函数编译到单个 C 文件（或少量 C 文件），由 cc 一次编译链接。

#### 4.9.2 全程序优化

单个 C 文件的优势：cc -O2 -flto 可以做跨函数优化，
包括内联、常量传播、死代码消除等。这等效于 LTO（Link-Time Optimization）。

#### 4.9.3 外部 C 库链接

```c
// xray 源码中声明外部 C 函数:
// extern fn printf(fmt: string, ...): int

// 生成 C:
// #include <stdio.h>
// 直接调用 printf()
```

**文件清单**：
| 文件 | 操作 | 行数估算 |
|------|------|---------|
| `src/cli/cmd_build.c` | 修改 | +200 |
| `src/aot/xcgen.c` | 修改 | +150 |

---

## 5. Runtime 层设计

### 5.1 分层 Runtime

```
┌─────────────────────────────────────────┐
│            xray_runtime.h               │  ← AOT 使用的精简 API
│  xrt_string_concat / xrt_array_get /    │
│  xrt_spawn / xrt_channel_send / ...     │
├─────────────────────────────────────────┤
│            libxray_rt.a                 │  ← 精简 runtime 静态库
│  GC / scheduler / string / array /      │
│  channel / class / ...                  │
├─────────────────────────────────────────┤
│            libxray_core.a               │  ← 完整 runtime（含 VM/JIT）
│  bytecode interpreter / JIT /           │
│  module loader / ...                    │
└─────────────────────────────────────────┘
```

| 构建模式 | 链接的库 | 说明 |
|----------|---------|------|
| `xray build app.xr` (默认) | libxray_core | 完整 runtime，含 VM |
| `xray build app.xr --native` | libxray_rt | 精简 runtime，不含 VM |
| `xray build app.xr --native` (纯计算) | 无依赖 | 自包含 C 文件 |

### 5.2 Runtime API 设计原则

1. **薄包装**：`xrt_*` 函数只是对现有 `xr_*` 函数的薄包装，适配 AOT 调用约定
2. **可内联**：简单操作（如 box/unbox/tag_check）定义在 header 中作为 `static inline`
3. **最小化**：只暴露 AOT 需要的 API，不暴露 VM 内部结构
4. **无状态**：AOT 代码不持有全局 VM 状态，通过 TLS 或参数传递 isolate

```c
// xray_runtime.h 示例

#pragma once
#include <stdint.h>
#include <stdbool.h>
#include <setjmp.h>

// === Value representation (Tagged Union 16B) ===
typedef struct XrValue {
    union { int64_t i; double f; void *ptr; };
    uint32_t tag;
    uint32_t _pad;
} XrValue;

#define XR_TAG_NULL   0
#define XR_TAG_INT    1
#define XR_TAG_FLOAT  2
#define XR_TAG_BOOL   3
#define XR_TAG_STRING 4
#define XR_TAG_OBJECT 5

static inline XrValue xrt_box_int(int64_t v)  { return (XrValue){.i=v, .tag=XR_TAG_INT}; }
static inline XrValue xrt_box_float(double v)  { return (XrValue){.f=v, .tag=XR_TAG_FLOAT}; }
static inline int64_t xrt_unbox_int(XrValue v) { return v.i; }
static inline double  xrt_unbox_float(XrValue v) { return v.f; }

// === String ===
XrValue xrt_string_concat(XrValue a, XrValue b);
XrValue xrt_string_len(XrValue s);

// === Array ===
XrValue xrt_array_get(XrValue arr, int64_t idx);
void    xrt_array_set(XrValue arr, int64_t idx, XrValue val);
int64_t xrt_array_len(XrValue arr);

// === Object ===
XrValue xrt_field_get(XrValue obj, const char *name);
void    xrt_field_set(XrValue obj, const char *name, XrValue val);

// === Coroutine ===
void    xrt_spawn(void *fn, XrValue *args, int nargs);
void    xrt_yield(void);

// === Channel ===
void    xrt_channel_send(XrValue ch, XrValue val);
XrValue xrt_channel_recv(XrValue ch);

// === GC ===
void    xrt_safepoint(void);
void    xrt_write_barrier(void *parent, void *child);
void   *xrt_alloc(size_t size);

// === Exception ===
void    xrt_throw(XrValue exception) __attribute__((noreturn));
XrValue xrt_get_current_exception(void);

// === Mixed-type operations ===
XrValue xrt_add(XrValue a, XrValue b);
XrValue xrt_sub(XrValue a, XrValue b);
XrValue xrt_mul(XrValue a, XrValue b);
XrValue xrt_lt(XrValue a, XrValue b);
XrValue xrt_eq(XrValue a, XrValue b);
```

---

## 6. 生成 C 代码示例

### 6.1 纯计算函数（Phase 1, 当前已有）

```xray
fn fib(n: int): int {
    if (n <= 1) return n
    return fib(n-1) + fib(n-2)
}
```

→ 生成：

```c
static int64_t xr_fib(int64_t v0);
static int64_t xr_fib(int64_t v0) {
    int64_t v1, v2, v3, v4, v5;
L0:
    v1 = (int64_t)(v0 <= INT64_C(1));
    if (v1) { goto L1; } else { goto L2; }
L1:
    return v0;
L2:
    v2 = v0 - INT64_C(1);
    v3 = xr_fib(v2);
    v4 = v0 - INT64_C(2);
    v5 = xr_fib(v4);
    v3 = v3 + v5;
    return v3;
}
```

### 6.2 字符串 + 函数调用（Phase 2）

```xray
fn greet(name: string): string {
    return "Hello, " + name + "!"
}
```

→ 生成：

```c
#include <xray_runtime.h>

static XrValue xr_greet(XrValue v0) {
    XrValue v1, v2, v3;
L0:
    v1 = xrt_string_concat(xrt_string_lit("Hello, "), v0);
    v2 = xrt_string_concat(v1, xrt_string_lit("!"));
    return v2;
}
```

### 6.3 协程 + Channel（Phase 8）

```xray
fn producer(ch: Channel<int>) {
    for i in 0..10 {
        ch.send(i)
    }
    ch.close()
}

fn main() {
    let ch = Channel<int>(0)
    spawn producer(ch)
    for msg in ch {
        print(msg)
    }
}
```

→ 生成：

```c
#include <xray_runtime.h>

static void xr_producer(XrValue v0) {
    int64_t v1;
    XrValue v2;
L0:
    v1 = INT64_C(0);
    goto L1;
L1:
    if ((int64_t)(v1 < INT64_C(10))) { goto L2; } else { goto L3; }
L2:
    v2 = xrt_box_int(v1);
    xrt_channel_send(v0, v2);
    xrt_safepoint();
    v1 = v1 + INT64_C(1);
    goto L1;
L3:
    xrt_channel_close(v0);
    return;
}

int main(int argc, char **argv) {
    xrt_init(argc, argv);
    XrValue ch = xrt_channel_new(0);
    xrt_spawn((void*)xr_producer, &ch, 1);
    XrValue msg;
    while (xrt_channel_recv_iter(ch, &msg)) {
        xrt_print(msg);
    }
    xrt_shutdown();
    return 0;
}
```

---

## 7. 实施优先级与时间估算

| Phase | 内容 | 估计时间 | 依赖 | 价值 |
|-------|------|---------|------|------|
| **1** | 重构框架 (xcgen) | 2-3 天 | 无 | 架构基础 |
| **2** | 函数调用 + 字符串 | 3-4 天 | Phase 1 | **高**：多函数工程可用 |
| **3** | 数组/Map + 对象 | 3-4 天 | Phase 2 | **高**：数据结构可用 |
| **4** | Tagged Value (BOX/UNBOX) | 2-3 天 | Phase 1 | **高**：any 类型可用 |
| **5** | GC 集成 | 2-3 天 | Phase 3,4 | 必要：防止内存泄漏 |
| **6** | 异常 + defer | 2-3 天 | Phase 5 | 中等 |
| **7** | 闭包 + Upvalue | 2-3 天 | Phase 2 | 中等 |
| **8** | 协程 + Channel | 3-4 天 | Phase 5 | **高**：xray 核心特性 |
| **9** | 模块 import + 多文件 | 3-4 天 | Phase 2 | **高**：工程化 |

**总计**：约 22-31 天，分 3 个里程碑：

- **M1 (Phase 1-4)**：基础可用，约 10-14 天
  - 纯计算 + 字符串 + 数组 + 混合类型 → 覆盖大部分计算密集型代码
- **M2 (Phase 5-7)**：GC + 异常 + 闭包，约 6-9 天
  - 完整的内存管理和错误处理 → 生产可用
- **M3 (Phase 8-9)**：协程 + 多文件，约 6-8 天
  - 全功能 AOT → 与 bytecode 模式功能对等

---

## 8. 目录迁移与旧代码清理

### 8.1 Phase 0：目录重组（在 Phase 1 前执行）

从 `src/jit/` 迁移到 `src/xir/` 的文件：

| 原路径 | 新路径 |
|--------|--------|
| `src/jit/xir.h/c` | `src/xir/xir.h/c` |
| `src/jit/xir_builder.h/c` | `src/xir/xir_builder.h/c` |
| `src/jit/xir_pass.h/c` | `src/xir/xir_pass.h/c` |
| `src/jit/xir_printer.h/c` | `src/xir/xir_printer.h/c` |
| `src/jit/xir_tfa.h/c` | `src/xir/xir_tfa.h/c` |
| `src/jit/xir_feedback.h/c` | `src/xir/xir_feedback.h/c` |

留在 `src/jit/` 的文件（JIT 专用）：
- `xir_jit.h/c` — JIT 入口、热函数检测
- `xir_codegen.h/c` — ARM64 代码生成
- `xir_arm64.h/c` — ARM64 指令编码
- `xir_code_alloc.h/c` — JIT 代码内存分配
- `xir_offsets.h` / `xir_offsets_verify.c` — VM 偏移

### 8.2 需要删除的旧文件

| 文件 | 原因 |
|------|------|
| `src/jit/xir_transpile_c.h/c` | 被 `src/aot/xcgen` 完全替代 |
| `src/jit/xir_codegen_aot.c` | ARM64 AOT codegen，不再使用 |
| `src/jit/xir_aot.h/c` | ARM64 AOT 模块管理（Mach-O 输出），不再使用 |

### 8.3 `#include` 路径批量更新

```c
// 旧：
#include "xir.h"           // 同目录隐式引用
#include "xir_pass.h"

// 新：
#include "xir/xir.h"       // 显式目录前缀
#include "xir/xir_pass.h"
```

`CMakeLists.txt` 添加 include 路径：

```cmake
target_include_directories(xray PRIVATE
    ${CMAKE_SOURCE_DIR}/src      # src/xir/xir.h → "xir/xir.h"
)
```

---

## 9. 与现有系统的集成

### 9.1 XIR 层（`src/xir/`）无需修改

`xir_builder.c` 已能翻译全部字节码操作码到 XIR。
`xir_pass.c` 的优化对 JIT 和 AOT 同样有效。
AOT 使用 `XIR_OPT_FULL` 级别（最大化优化）。
`src/aot/xcgen` 只是 XIR 的另一个消费者（与 `src/jit/xir_codegen` 并列）。

### 9.2 cmd_build.c 改动最小

当前 `cmd_build_pure()` 中的 transpile 逻辑只需替换 API 调用：

```c
// 旧：
#include "jit/xir_transpile_c.h"
XirTranspileCResult tr = xir_transpile_to_c(xfunc, c_name);
// 新：
#include "aot/xcgen.h"
XcgenFunc *cfn = xcgen_compile_func(mod, xfunc, c_name);
```

### 9.3 Tagged Union 迁移协同

Tagged Union (16B XrValue) 迁移完成后，xcgen 的 BOX/UNBOX 代码
自动使用新布局——因为 `xray_runtime.h` 中的 `XrValue` 定义就是新的。

### 9.4 依赖关系图

```
                  src/xir/          (共享 IR 层)
                 ╱        ╲
          src/jit/        src/aot/   (两个独立后端)
                 ╲        ╱
                  src/vm/           (都可被 VM 调用)
```

`src/jit/` 和 `src/aot/` 互不 include，仅共享 `src/xir/` 的头文件。

---

## 10. 参考文件索引

| 参考项目 | 关键文件 | 关注点 |
|----------|---------|--------|
| **Nim** | `compiler/cgen.nim` | 入口架构、section 组织 |
| | `compiler/cgendata.nim` | `BModule`/`BProc` 数据结构 |
| | `compiler/ccgexprs.nim` | 表达式→C：字面量、运算符、转换 |
| | `compiler/ccgstmts.nim` | 语句→C：if/for/try/defer |
| | `compiler/ccgcalls.nim` | 函数调用：参数传递、闭包调用 |
| | `compiler/ccgtypes.nim` | 类型→C：名称修饰、struct 生成 |
| | `compiler/ccgtrav.nim` | GC 遍历函数生成 |
| **V** | `vlib/v/gen/c/cgen.v` | 入口架构、30+ Builder |
| | `vlib/v/gen/c/fn.v` | 函数声明/调用 |
| | `vlib/v/gen/c/spawn_and_go.v` | 并发：wrapper struct + pthread |
| | `vlib/v/gen/c/defer.v` | defer：flag 变量模式 |
| | `vlib/v/gen/c/array.v` | 数组操作生成 |
| | `vlib/v/gen/c/str.v` | 字符串操作生成 |
| **xray** | `src/xir/xir.h` | XIR 指令集、数据结构 |
| | `src/xir/xir_builder.c` | Bytecode → XIR |
| | `src/xir/xir_pass.c` | 优化管线 |
| | `src/jit/xir_transpile_c.c` | 现有 transpiler（待删除，被 `src/aot/xcgen` 替代） |
| | `src/aot/xcgen.h/c` | 新 C codegen 入口 |
| | `src/cli/cmd_build.c` | 构建命令集成 |
