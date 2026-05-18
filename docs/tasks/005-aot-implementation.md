# Xray AOT 实施文档

> Version: 2.0 | Date: 2026-04-23
> 基于 6 个参考项目源码分析：Nim, V, QuickJS, Go, libdill, kotlinx.coroutines
> 前置文档：`docs/design/aot-design.md`（设计方案）

### ⚡ 开发原则

- **不考虑向后兼容性** — Xray 是全新语言，每个阶段选择最佳设计
- **最大化复用** — 现有 `src/coro/` 17,600 行成熟调度器直接复用，不重写
- **CPS 优先** — 并发协程通过 CPS (Continuation-Passing Style) 状态机编译，与 VM/JIT 语义 100% 等价
- **AOT 产出独立原生二进制** — runtime 编译进 binary（Go 模式），不依赖 VM 解释器

---

## 〇、当前状态与差距

### 已完成（Phase 0, 2026-04）

| 特性 | 状态 | 测试 |
|------|------|------|
| 单模块 XIR→C 翻译 | ✅ | 9 个测试全通过 |
| 基本类型特化 (int64/f64) | ✅ | arithmetic.xr |
| 控制流 (if/while/for/goto) | ✅ | control_flow.xr |
| 闭包 + 逃逸分析 | ✅ | shared_vars.xr |
| 基础 class (构造/字段/方法) | ✅ | class_basic/methods.xr |
| 同函数 try/catch | ✅ | try_catch.xr |
| ARC bump allocator | ✅ | xrt_arc.h |
| 模块 export 框架 | ✅ 框架 | export_const/math_lib.xr |

### 关键差距

| 差距 | 严重度 | 参考 |
|------|--------|------|
| 多模块 import 消费未接线 | 🔴 | Nim cgen.nim + V cgen.v |
| xrt_*.h 直接用 malloc/free | 🔴 | 违反项目规范 |
| ARC retain/release 未插入到生成代码 | 🔴 | Nim ccgexprs.nim |
| class 继承/vtable 未测试 | 🟡 | V struct.v |
| finally 子句未实现 | 🟡 | Nim ccgstmts.nim |
| defer 语句未实现 | 🟡 | V defer.v |
| match 表达式未实现 | 🟡 | V match.v |
| 并发 (go/channel/select/scope) — CPS + 复用 src/coro/ | 🟡 | Go runtime + Kotlin coroutines |
| 高阶集合方法 (map/filter/reduce) | 🟡 | QuickJS quickjs.c |
| enum/泛型/接口 | 🟡 | 长期 |
| 间歇性 SIGBUS (XIR passes) | 🔴 | 独立于 AOT，但阻塞可靠性 |

---

## 一、Phase A — 运行时修复与多模块基础（1-2 周）

### A.1 修复 xrt_*.h 的 malloc/free 违规

**问题**：`xrt_coll.h`, `xrt_class.h`, `xrt_arc.h` 直接调用 `malloc/calloc/realloc/free`。

**参考**：QuickJS `quickjs.c` — 所有分配通过 `js_malloc/js_free/js_realloc` 间接层，
运行时上下文持有分配器：

```c
// quickjs.c:
static void *js_malloc(JSContext *ctx, size_t size) {
    void *ptr = ctx->rt->mf.js_malloc(&ctx->rt->malloc_state, size);
    ...
}
```

**实施**：

1. 在 `xrt_arc.h` 顶部新增 AOT 分配器适配宏：

```c
// xrt_arc.h — AOT allocator adapter
// In standalone AOT mode, map to system allocator.
// In embedded mode, map to xr_malloc/xr_free.
#ifndef XRT_MALLOC
  #define XRT_MALLOC(sz)       malloc(sz)
  #define XRT_CALLOC(n, sz)    calloc(n, sz)
  #define XRT_REALLOC(p, sz)   realloc(p, sz)
  #define XRT_FREE(p)          free(p)
#endif
```

2. 替换 `xrt_coll.h`、`xrt_class.h`、`xrt_arc.h` 中所有裸 malloc → `XRT_MALLOC` 等
3. 编译 xray 主项目时定义 `-DXRT_MALLOC=xr_malloc` 等

**文件清单**：

| 文件 | malloc 调用数 | 需替换 |
|------|-------------|--------|
| `xrt_coll.h` | 8 (array_new, strbuf, map, closure) | ✅ |
| `xrt_class.h` | 1 (xrt_obj_alloc calloc) | ✅ |
| `xrt_arc.h` | 2 (bump block calloc, arc_alloc calloc) | ✅ |

### A.2 多模块编译管线

**参考**：

- **Nim** `cgen.nim` 的 `BModuleList` — 维护所有模块的编译状态，每个模块一个 `BModule`，
  最后 `cgenWriteModules()` 遍历所有模块输出：

```nim
# Nim cgendata.nim:
BModuleList* = ref object
    modules*: seq[BModule]     # all compiled modules
    generatedHeader*: BModule  # shared header
    graph*: ModuleGraph
```

- **V** `cgen.v` 的 `Gen` struct — 单一 Gen 实例遍历所有文件，按文件分段输出 C 代码。
  V 用 `cheaders`/`definitions`/`typedefs` 等 Builder 分段拼装。

**实施步骤**：

1. **重构 `xcgen.h` 数据结构**：

```c
// Current: XcgenModule (single module)
// Target:  XcgenCompilation → XcgenModuleUnit[]

typedef struct XcgenModuleUnit {
    const char   *module_name;
    const char   *module_path;
    XcgenBuf      sections[XCGEN_SEC_COUNT];
    XcgenFunc   **funcs;
    int           nfuncs, funcs_cap;
    XcgenExport  *exports;
    int           nexports;
    int16_t       module_id;
} XcgenModuleUnit;

typedef struct XcgenCompilation {
    XcgenModuleUnit *units;
    int              nunits, units_cap;
    XcgenProtoEntry *proto_map;     // global, cross-module
    int              proto_count, proto_cap;
    XcgenStructRegistry struct_reg; // global
    const char      *output_path;
    bool             single_file;
} XcgenCompilation;
```

2. **扩展 `cmd_build_native`（`src/api/xcli_build.c`）**：

```
for each module in XrBundle (topo order):
    compile .xr → XrProto
    lower XrProto → XIR
    run XIR optimization passes
    xcgen_compile_module(compilation, xir_module)
xcgen_emit_all(compilation, output_path)
```

3. **跨模块函数引用**：

参考 Nim `findPendingModule()` — 通过符号查找所属模块，生成 `extern` 声明。

```c
// In module A, calling module B's function:
// xcgen 检查 proto_map 找到目标模块 ID
// 生成: extern int64_t xr_math_add(XrtContext, int64_t, int64_t);
// 调用: result = xr_math_add(ctx, a, b);
```

4. **模块初始化顺序**：

参考 Nim 的 `cfsInitProc` + `cfsDatInitProc` 两阶段模式：
- `datInit`: 初始化数据（全局变量赋值）
- `init`: 执行模块顶层代码

```c
// Generated main.c:
int main(int argc, char **argv) {
    xrt_runtime_init();
    // Phase 1: data init (in topo order)
    mod_math__dat_init();
    mod_main__dat_init();
    // Phase 2: code init (in topo order)
    mod_math__init(NULL);
    mod_main__init(NULL);
    xrt_runtime_cleanup();
    return 0;
}
```

### A.3 ARC 生命周期插入

**参考**：Nim `ccgexprs.nim` 的 `genAssignment()` — 每次赋值前对旧值 release，对新值 retain：

```nim
# Nim ccgexprs.nim:
proc genAssignment(p: BProc; dest, src: TLoc) =
  if optSeqDestructors in p.config.globalOptions:
    genGenericAsgn(p, dest, src)       # includes =copy hooks
  elif containsManagedMemory(dest.t):
    if dest.storage == OnStack:
      p.s(cpsStmts).addAssignment(...)  # simple for new vars
    else:
      genRefAssign(p, dest, src)        # retain src, release old dest
```

**实施**：

1. 在 `xcgen_expr.c` 的 `MOV` / `STORE_FIELD` 指令翻译中，对 PTR 类型值插入 ARC 操作：

```c
// Before:
//   v5 = v3;
// After (when v3 is PTR type):
//   xrt_val_retain(v3);
//   xrt_val_release(v5);  // release old value
//   v5 = v3;
```

2. **Last-use 优化**（参考 Nim `injectdestructors.nim`）：

XIR SSA 形式天然支持 last-use 分析。在 defuse 信息中，如果某 vreg 只剩一个 use 且该 use 是 MOV/赋值：
- 跳过 retain（move 语义）
- 在 last-use 处不 release（ownership 转移）

这是 Phase D 的优化项，Phase A 先做保守的 retain/release。

### A.4 测试补充

| 新增测试 | 验证内容 |
|---------|---------|
| `tests/aot/modules/import_fn.xr` | 跨模块函数调用 |
| `tests/aot/modules/import_class.xr` | 跨模块类使用 |
| `tests/aot/basic/closures.xr` | 闭包逃逸/非逃逸 |
| `tests/aot/basic/string_methods.xr` | 字符串方法 |
| `tests/aot/basic/array_ops.xr` | 数组操作 |

---

## 二、Phase B — 语言特性完善（2-3 周）

### B.1 finally 子句

**参考**：Nim `ccgstmts.nim` 的 `genTry()` — finally 通过在所有出口（正常 + 异常）都插入 finally 代码实现：

```nim
# Nim ccgstmts.nim genTry():
# 1. Push exception frame
# 2. Generate try body
# 3. At normal exit: run finally, goto after
# 4. At exception: run finally, re-raise or goto catch
```

**实施**：

在 `xcgen_stmt.c` 的 terminator 翻译中：

```c
// try { body } catch(e) { handler } finally { cleanup }
// → C translation:
XrtExcFrame _ef0;
_ef0.prev = xrt_exc_top; xrt_exc_top = &_ef0;
int _ef0_state = 0;  // 0=normal, 1=exception
if (setjmp(_ef0.buf) != 0) { _ef0_state = 1; goto L_catch; }

// try body...
goto L_finally;

L_catch:
    xrt_exc_top = _ef0.prev;
    // catch handler...

L_finally:
    // finally code (always runs)
    { cleanup_code; }
    if (_ef0_state == 1 && !caught) {
        xrt_throw_exc(_ef0.exception);  // re-throw if not caught
    }
    xrt_exc_top = _ef0.prev;
```

### B.2 defer 语句

**参考**：V `defer.v` — 每个 defer 分配一个 bool flag，在函数/块出口 LIFO 顺序执行：

```v
// V defer.v:
fn (g &Gen) defer_flag_var(stmt &ast.DeferStmt) string {
    return '${g.last_fn_c_name}_defer_${stmt.idx_in_fn}'
}
// At defer site:   _fn_defer_0 = true;
// At return/exit:  if (_fn_defer_2) { ... }
//                  if (_fn_defer_1) { ... }
//                  if (_fn_defer_0) { ... }
```

**实施**：

1. 在 `XcgenFunc` 中增加 `defer_count` 字段
2. 遇到 XIR 的 defer 指令时：
   - 记录 defer 体的代码到一个延迟缓冲区
   - 在当前位置生成 `_defer_N = 1;`
3. 在所有 return 路径和函数末尾，LIFO 顺序插入：

```c
if (_defer_1) { /* defer body 1 */ }
if (_defer_0) { /* defer body 0 */ }
```

### B.3 match 表达式

**参考**：V `match.v` — match 翻译为 C 的链式 `if/else if`，对枚举类型优化为 `switch`：

```v
// V match.v:
fn (mut g Gen) match_expr(node ast.MatchExpr) {
    // For enum types with many branches → switch
    if is_enum && branches > 5 {
        g.write('switch(')
        g.expr(node.cond)
        g.writeln(') {')
        // ... case per branch ...
    } else {
        // chain of if/else if
    }
}
```

**实施**：

```c
// xray match:
// let result = match x {
//     1 => "one",
//     2, 3 => "few",
//     10..20 => "teen",
//     _ => "default"
// }

// → C translation:
XrtValue _match_tmp;
if (x == 1) {
    _match_tmp = xrt_box_str("one");
} else if (x == 2 || x == 3) {
    _match_tmp = xrt_box_str("few");
} else if (x >= 10 && x <= 20) {
    _match_tmp = xrt_box_str("teen");
} else {
    _match_tmp = xrt_box_str("default");
}
XrtValue result = _match_tmp;
```

对于整数 cond，在分支数 > 5 时生成 `switch/case`。

### B.4 class 继承完善

**参考**：V `struct.v` + Nim `ccgtypes.nim` 的 vtable 生成。

当前 `xrt_class.h` 已有 vtable 框架，需补充：

1. **super 调用**：编译为直接调用父类函数（不走 vtable）
2. **字段继承**：子类 struct 包含父类所有字段（相同偏移量）
3. **abstract 方法**：vtable slot 填 NULL，调用时检查并报错
4. **override 标记**：编译期检查（已在前端完成），codegen 只需覆盖 vtable slot

```c
// super.speak() → direct call to parent
xr_Animal_speak(ctx, (XrtObj_Animal*)self);

// Polymorphic call (type unknown at compile time)
typedef XrValue (*SpeakFn)(XrtContext, void*);
XrtTypeInfo *ti = &xrt_type_table[((XrtObjHeader*)obj)->type_id];
((SpeakFn)ti->vtable[SLOT_SPEAK])(ctx, obj);
```

### B.5 集合方法扩展

**参考**：QuickJS `quickjs.c` 的 `js_array_proto_funcs` — 每个方法一个 C 函数实现。

**优先实现**（覆盖 80% 使用场景）：

| 方法 | 实现位置 | 策略 |
|------|---------|------|
| `array.map(fn)` | `xrt_method.h` | 新分配 array，逐元素调用 fn |
| `array.filter(fn)` | `xrt_method.h` | 新分配 array，条件 push |
| `array.forEach(fn)` | `xrt_method.h` | 逐元素调用，无返回值 |
| `array.indexOf(val)` | `xrt_method.h` | 线性查找 |
| `array.slice(s,e)` | `xrt_method.h` | 新分配 sub-array |
| `array.sort(cmp?)` | `xrt_method.h` | qsort wrapper |
| `string.split(sep)` | `xrt_method.h` | strtok 风格 |
| `string.replace(old,new)` | `xrt_method.h` | strstr + 拼接 |
| `string.trim()` | `xrt_method.h` | 跳过首尾空白 |
| `string.toLower/Upper()` | `xrt_method.h` | 逐字符转换 |
| `map.keys()` | `xrt_method.h` | 返回 key 数组 |
| `map.values()` | `xrt_method.h` | 返回 value 数组 |
| `map.delete(key)` | `xrt_method.h` | 线性查找删除 |

### B.6 enum 翻译

```xray
enum Color { Red, Green, Blue }
```

```c
// → C:
enum { COLOR_RED = 0, COLOR_GREEN = 1, COLOR_BLUE = 2 };
static const int64_t Color_member_count = 3;
static const char *Color_names[] = {"Red", "Green", "Blue"};
```

### B.7 测试补充

| 新增测试 | 验证内容 |
|---------|---------|
| `tests/aot/basic/finally.xr` | try/catch/finally |
| `tests/aot/basic/defer.xr` | defer LIFO 执行 |
| `tests/aot/basic/match.xr` | match 表达式 |
| `tests/aot/classes/inheritance.xr` | 继承 + vtable |
| `tests/aot/classes/super_call.xr` | super 调用 |
| `tests/aot/basic/enum.xr` | 枚举 |
| `tests/aot/basic/collection_methods.xr` | map/filter/reduce |

---

## 三、Phase C — CPS 并发（3-4 周）

### C.0 核心架构决策

**决策：CPS 状态机 + 直接复用 `src/coro/` 调度器**

CPS (Continuation-Passing Style) 把 VM 在运行时做的事提前到编译期：

```
VM 模式:  函数执行 → OP_CHAN_RECV → 保存 pc/stack → return BLOCKED → 调度器切换
CPS 模式: 状态机执行 → xr_channel_recv() → 保存 state → return BLOCKED → 调度器切换
```

两者在数学上同构。VM 的 `XrVMContext`（stack + frames + pc）就是运行时 CPS frame。
CPS 把它变成编译期确定的 C struct，消除 VM 解释器开销。

**为什么不重写调度器**：

现有 `src/coro/` 是 17,600 行成熟调度器，包含：

| 组件 | 文件 | VM 耦合 |
|------|------|---------|
| 调度循环 / work stealing / park | `xworker_sched.c` (890) | **零** |
| Lock-free steal queue | `xsteal_queue.c` | **零** |
| MPSC inbox + Dekker fence | `xmpsc_queue.c` | **零** |
| Channel (ring + waitq + deep copy) | `xchannel.c` (759) | **低** — 只引用 `XrCoroutine*` |
| Task 状态机 / 结构化并发 | `xtask.c` (436) | **零** |
| Timer wheel | `xtimer_wheel.c` (992) | **零** |
| I/O (kqueue/epoll/iouring) | `xnetpoll*.c` (935+) | **零** |
| 执行分发 + 结果处理 | `xworker_exec.c` (1015) | **高** — 调用 VM |
| 协程生命周期 | `xcoro.c` (1462) | **高** — 管理 XrVMContext |

**结论**：17,600 行中只有 ~2,500 行和 VM 耦合，其余 ~15,000 行是纯调度/并发基础设施。
重写 = 丢掉 work stealing / continuation stealing / LIFO fast dispatch / sysmon 等全部优化。

**正确做法**：`#ifdef XRAY_AOT` 让 `XrCoroutine` 支持 CPS 执行模式，~300 行改动复用全部调度器。

### C.1 XrCoroutine CPS 适配

**原则**：AOT 编译时 `#define XRAY_AOT`，strip 掉 VM/JIT 字段，替换为 CPS 字段。

```c
// xcoroutine.h — CPS mode adaptation

typedef struct XrCoroutine {
    // ===== HOT ZONE (调度器用，100% 复用，不改) =====
    XrGCHeader gc;
    _Atomic uint32_t flags;
    int32_t reductions;
    struct XrCoroutine *sched_link;
    struct XrCoroutine *next, *prev;
    _Atomic int resume_status;
    _Atomic int affinity_p;
    int id;
    // ...

#ifdef XRAY_AOT
    // ===== CPS 执行上下文 (替代 XrVMContext) =====
    XrVMResult (*cps_step)(void *frame);   // CPS 状态机 step 函数
    void       *cps_frame;                 // CPS frame struct (编译期确定大小)
    uint16_t    cps_frame_size;            // for pool reuse
#else
    // ===== VM 执行上下文 (原有) =====
    XrVMContext vm_ctx;
    struct XrJitScratch *jit_ctx;
    XrJitSuspendState *jit_suspend;
    // ...
#endif

    // ===== 以下全部 100% 复用 =====
    XrValue result, error;
    struct XrTask *task;
    _Atomic int wait_count;
    struct XrScopeContext *parent_scope;
    struct XrCoroutine *wait_link;        // channel waitq
    void *wait_channel;
    XrValue send_value;
    XrValue *recv_slot;
    struct XrSelectWait *select_wait;
    struct XrCoroutine *pending_spawn;    // continuation stealing
    struct XrScopeContext *current_scope;
    // ...
} XrCoroutine;
```

**内存对比**：

| 字段 | VM 模式 | CPS 模式 |
|------|---------|---------|
| 执行上下文 | `XrVMContext` (~128B) + stack (4KB) + frames | `cps_step` (8B) + `cps_frame` (8B) + frame struct (~50-200B) |
| JIT | `jit_ctx` + `jit_suspend` (~400B) | 无 |
| **每协程总开销** | ~400B header + 4KB stack | ~200B header + ~100B frame |

百万协程：VM 模式 ~4.4GB → CPS 模式 ~300MB。

### C.2 执行分发适配

改动仅在 `xworker_exec.c` 的 `xr_coro_run_on_worker()` 中加一个分支：

```c
// xworker_exec.c — CPS dispatch (仅新增部分)
#ifdef XRAY_AOT
    // CPS mode: call step function directly
    XR_DCHECK(coro->cps_step != NULL, "CPS coro has NULL step function");
    coro->reductions = XR_CORO_REDUCTIONS;  // fair scheduling
    result = coro->cps_step(coro->cps_frame);
#else
    // VM mode: existing dispatch (closure/cfunc/JIT)
    if (coro->entry_type == XR_CORO_ENTRY_CLOSURE) {
        result = run(isolate, coro_ctx);
    }
    // ...
#endif
```

**关键**：返回码完全复用。CPS step 函数返回 `XR_VM_OK / XR_VM_YIELD / XR_VM_BLOCKED / XR_VM_SPAWN_CONT`。
`worker_handle_vm_result()`、`worker_exec_with_cont_stealing()`、`worker_process_blocked()` — **一行不改**。

### C.3 CPS 状态机代码生成

**函数着色**：编译器从显式挂起点向上传播 `may_suspend` 标记：

```
XIR pass: propagate_suspend_mark
  1. 标记所有直接挂起函数 (含 channel send/recv/await/yield/sleep/select)
  2. 遍历调用图: f 调用了 may_suspend 的 g → f 也标记 may_suspend
  3. 只对 may_suspend 函数做 CPS 变换
  4. 不含挂起点的函数编译为普通 C 函数（零开销）
```

xray 的挂起点全部是显式语法 (`await`/`ch.send()`/`ch.recv()`/`select`/`yield`/`sleep`)，
且编译器已有 `is_coro_safe` 标记，传播分析是自然的。

**CPS 变换示例**：

```xray
fn producer(ch: Channel<int>, n: int) {
    for i in 0..n {
        ch.send(i)          // 挂起点
    }
    ch.close()
}
```

生成的 C 代码：

```c
// CPS frame (编译期确定，只包含活跃局部变量)
typedef struct {
    uint8_t    state;
    XrChannel *ch;
    int64_t    n;
    int64_t    i;
} XrtFrame_producer;

// CPS step 函数 (每次调用执行到下一个挂起点或结束)
static XrVMResult xr_producer_step(void *raw) {
    XrtFrame_producer *f = (XrtFrame_producer *)raw;
    switch (f->state) {
    case 0:
        f->i = 0;
        f->state = 1;
        // fallthrough
    case 1:
        if (f->i >= f->n) { f->state = 3; goto done; }
        {
            // 获取当前 coroutine 以设置 send_value
            XrCoroutine *self = xr_current_coro();
            self->send_value = xr_int(f->i);
            XrChanResult cr = xr_channel_send(f->ch, xr_int(f->i), self);
            if (cr == XR_CHAN_OK) {
                f->i++;
                return XR_VM_YIELD;  // 让出一次，公平调度
            }
            if (cr == XR_CHAN_BLOCK) {
                f->state = 2;
                return XR_VM_BLOCKED;  // 调度器处理阻塞
            }
        }
        // fallthrough on error
    case 2:  // 从 send 阻塞恢复
        f->i++;
        f->state = 1;
        return XR_VM_YIELD;
    done:
    case 3:
        xr_channel_close(f->ch);
        return XR_VM_OK;
    }
    return XR_VM_OK;
}
```

**非 may_suspend 函数不受影响**（零开销）：

```c
// 纯计算函数：直接编译为普通 C 函数，无状态机
static int64_t xr_fibonacci(int64_t n) {
    if (n <= 1) return n;
    return xr_fibonacci(n - 1) + xr_fibonacci(n - 2);
}
```

### C.4 `go` 表达式翻译

```xray
let data = [1, 2, 3]
let task = go compute(data)
```

```c
// → 创建 CPS frame + 深拷贝参数 + spawn 到调度器
XrtFrame_compute *_frame = xr_malloc(sizeof(XrtFrame_compute));
_frame->state = 0;
_frame->data = xr_deep_copy_to_coro(isolate, data, /*dst=*/NULL);

XrCoroutine *_coro = xr_coro_create_cps(
    isolate,
    xr_compute_step,       // CPS step 函数
    _frame,                // CPS frame
    sizeof(XrtFrame_compute)
);
XrTask *_task = xr_task_create(parent_coro, _coro);

// scope 跟踪 (复用现有逻辑，一行不改)
if (parent_coro->current_scope) {
    _coro->parent_scope = parent_coro->current_scope;
    atomic_fetch_add(&parent_coro->current_scope->count, 1);
}

// spawn 到调度器 (复用现有 xr_runtime_spawn，一行不改)
xr_runtime_spawn(runtime, _coro);
```

**复用清单**：`xr_runtime_spawn()` / `xr_task_create()` / `xr_task_attach_child()` /
scope 跟踪 / continuation stealing / deep copy — **全部原样复用**。

### C.5 Channel 复用

**直接复用 `xchannel.c`**。CPS 协程和 VM 协程在 channel 视角完全相同：
都是 `XrCoroutine*`，有 `wait_link`、`send_value`、`recv_slot` 字段。

代码生成的 channel 操作直接调用现有函数：

```c
// ch.send(val) → 编译为:
self->send_value = xr_deep_copy_value(val);
XrChanResult cr = xr_channel_send(ch, val, self);  // 现有函数，不改
if (cr == XR_CHAN_BLOCK) { f->state = N; return XR_VM_BLOCKED; }

// ch.recv() → 编译为:
self->recv_slot = &f->recv_val;
XrChanResult cr = xr_channel_recv(ch, &recv_val, self);  // 现有函数，不改
if (cr == XR_CHAN_BLOCK) { f->state = N; return XR_VM_BLOCKED; }

// ch.trySend/tryRecv/close/isClosed → 非阻塞，直接调用，无需 CPS
```

### C.6 select 复用

**直接复用现有 `XrSelectWait` / `XrSelectCase` / `xr_worker_block_select()`**：

```xray
select {
    msg from ch1 => { print(msg) }
    100 to ch2 => { print("sent") }
    after 1000 => { print("timeout") }
}
```

```c
// CPS 状态机中的 select 翻译:
case SELECT_ENTRY: {
    // 快速路径：尝试非阻塞操作
    if (xr_channel_try_recv(ch1, &f->recv_val)) {
        f->state = CASE_0_BODY; return XR_VM_YIELD;
    }
    if (xr_channel_try_send(ch2, xr_int(100))) {
        f->state = CASE_1_BODY; return XR_VM_YIELD;
    }
    // 慢路径：注册到所有 channel 并阻塞 (复用现有 select 基础设施)
    XrCoroutine *self = xr_current_coro();
    XrSelectCase cases[2];
    cases[0] = (XrSelectCase){ .channel = ch1, .is_send = false, .owner = self };
    cases[1] = (XrSelectCase){ .channel = ch2, .is_send = true,
                               .send_value = xr_int(100), .owner = self };
    self->select_wait = &(XrSelectWait){ .cases = cases, .case_count = 2 };
    xr_worker_block_select(worker, self, (void*[]){ch1, ch2}, 2);
    f->state = SELECT_RESUME;
    return XR_VM_BLOCKED;
}
case SELECT_RESUME:
    switch (self->select_ready_case) {
        case 0: f->state = CASE_0_BODY; break;
        case 1: f->state = CASE_1_BODY; break;
    }
    return XR_VM_YIELD;
```

### C.7 scope / supervisor scope / linked go

**全部直接复用** `xtask.c` 的 Task 层级 + `XrScopeContext`：

```c
// scope { go f(); go g() }
// CPS 状态机中:
case SCOPE_ENTER: {
    XrScopeContext *sc = xr_malloc(sizeof(XrScopeContext));
    *sc = (XrScopeContext){ .mode = XR_SCOPE_WAIT, .parent = self->current_scope };
    self->current_scope = sc;
    f->state = SCOPE_BODY;
    return XR_VM_YIELD;
}
// ... scope 内的 go 自动关联到 current_scope (现有逻辑) ...
case SCOPE_EXIT: {
    // 等待所有子协程完成 (复用现有 wait_count 机制)
    if (atomic_load(&self->current_scope->count) > 0) {
        f->state = SCOPE_EXIT;  // resume 后重新检查
        return XR_VM_BLOCKED;
    }
    // scope 结束
    self->current_scope = self->current_scope->parent;
    f->state = AFTER_SCOPE;
    return XR_VM_YIELD;
}
```

`XrScopeContext`、`xr_coro_wake_waiter()`、linked scope 取消传播、supervisor scope
错误收集 — **全部原样复用**，因为它们操作的是 `XrCoroutine` 和 `XrTask`，与执行模式无关。

### C.8 await 翻译

```c
// await task → CPS:
case AWAIT_CHECK: {
    XrTask *task = xr_value_to_task(f->task_val);
    if (xr_task_is_done(task)) {
        // 快速路径：已完成
        f->result = xr_deep_copy_to_coro(isolate, task->result, self);
        f->state = AFTER_AWAIT;
        return XR_VM_YIELD;
    }
    // 慢路径：注册等待 (复用现有 await_state CAS 协议)
    self->await_task = task;
    task->waiter = self;
    task->waiter_index = -1;
    int expected = XR_AWAIT_NONE;
    if (atomic_compare_exchange_strong(&task->await_state, &expected, XR_AWAIT_WAITING)) {
        f->state = AWAIT_RESUME;
        return XR_VM_BLOCKED;
    }
    // CAS 失败 → 在设置 waiter 期间已完成
    f->result = xr_deep_copy_to_coro(isolate, task->result, self);
    f->state = AFTER_AWAIT;
    return XR_VM_YIELD;
}

// await.all / await.any / await(timeout:) — 同理复用现有 wait_count / any_done 机制
```

### C.9 深拷贝

**复用现有 `xdeep_copy.c`**（698 行），适配 AOT 值类型：

AOT 编译时 `#ifdef XRAY_AOT` 路径中，deep copy 的 ARC 值类型适配约 ~100 行改动。
现有 `xr_deep_copy_to_coro()` 的递归遍历逻辑完全复用。

### C.10 AOT 构建配置

```cmake
# CMakeLists.txt — AOT runtime 构建
if(XRAY_AOT_RUNTIME)
    add_definitions(-DXRAY_AOT)
    # 包含 src/coro/ 全部调度器代码
    set(AOT_CORO_SOURCES
        src/coro/xworker.c
        src/coro/xworker_sched.c
        src/coro/xworker_exec.c    # 含 CPS 分发分支
        src/coro/xworker_blocked.c
        src/coro/xworker_sysmon.c
        src/coro/xcoro.c           # 含 xr_coro_create_cps()
        src/coro/xchannel.c
        src/coro/xtask.c
        src/coro/xdeep_copy.c
        src/coro/xsteal_queue.c
        src/coro/xmpsc_queue.c
        src/coro/xtimer_wheel.c
        src/coro/xbalance.c
        src/coro/xnetpoll.c
        # ... 平台特定 netpoll
    )
    # 排除 VM/JIT
    # 不包含 src/vm/, src/jit/, src/frontend/
endif()
```

### C.11 改动量总结

| 文件 | 改动类型 | 行数 |
|------|---------|------|
| `xcoroutine.h` | `#ifdef XRAY_AOT` + CPS 字段 | ~30 行 |
| `xworker_exec.c` | CPS 分发分支 | ~50 行 |
| `xcoro.c` | 新增 `xr_coro_create_cps()` | ~80 行 |
| `xcoro_pool.h` | pool 适配 CPS（不分配 VM stack） | ~30 行 |
| `xdeep_copy.c` | AOT 值类型适配 | ~100 行 |
| `xcgen_stmt.c` | CPS 状态机生成 | ~500 行（新逻辑） |
| `xcgen_expr.c` | 挂起点翻译 (channel/await/select) | ~300 行（新逻辑） |
| XIR pass | `propagate_suspend_mark` 传播分析 | ~200 行（新 pass） |
| CMakeLists.txt | AOT 构建排除 VM/JIT | ~20 行 |
| **总计新增/改动** | | **~1,310 行** |
| **直接复用不动** | | **~15,000 行** |

### C.12 并发测试

| 新增测试 | 验证内容 |
|---------|---------|
| `tests/aot/concurrency/go_basic.xr` | go + await + CPS 状态机基本流程 |
| `tests/aot/concurrency/channel.xr` | channel send/recv (阻塞/非阻塞) |
| `tests/aot/concurrency/channel_buffered.xr` | 缓冲 channel |
| `tests/aot/concurrency/select.xr` | select 多路复用 |
| `tests/aot/concurrency/scope.xr` | scope 等待 + linked scope 取消 |
| `tests/aot/concurrency/supervisor.xr` | supervisor scope 错误收集 |
| `tests/aot/concurrency/deep_copy.xr` | go 参数 / channel 深拷贝 |
| `tests/aot/concurrency/shared_const.xr` | shared const 零拷贝共享 |
| `tests/aot/concurrency/million_coro.xr` | 百万级 CPS 协程压测 |
| `tests/aot/concurrency/pingpong.xr` | channel ping-pong 吞吐量对比 VM |

---

## 四、Phase D — Stdlib 桥接与优化（2-3 周）

### D.1 Stdlib C 绑定

**参考**：V 的 stdlib — 每个 stdlib 模块有 C wrapper，`#include` 对应系统库。

**策略**：

对 xray 的 `stdlib/` 目录中的每个模块，生成一个 `xrt_stdlib_<name>.h`：

```c
// xrt_stdlib_math.h — AOT bindings for stdlib/math
#include <math.h>
static inline XrtValue xrt_math_floor(XrtValue v) {
    return xrt_box_float(floor(v.tag == XRT_TAG_F64 ? v.f : (double)v.i));
}
static inline XrtValue xrt_math_sqrt(XrtValue v) { ... }
static inline XrtValue xrt_math_pow(XrtValue a, XrtValue b) { ... }
// ...
```

**优先级**（按使用频率）：

| 模块 | 优先级 | C 依赖 |
|------|--------|--------|
| `math` | P0 | `<math.h>` |
| `time` | P0 | `<time.h>`, `clock_gettime` |
| `io` | P0 | `<stdio.h>` |
| `json` | P1 | 自有实现 (参考 cJSON) |
| `os` | P1 | POSIX APIs |
| `path` | P1 | 字符串操作 |
| `regex` | P2 | PCRE2 或 re2 |
| `http` | P2 | `stdlib/http/` 已有 C 实现 |
| `crypto` | P2 | `stdlib/crypto/` 已有 C 实现 |

### D.2 多文件输出

**参考**：Nim 的 `nimcache/` 目录结构 — 每模块一个 `.c` + 共享运行时头文件。

```
~/.xray/cache/<hash>/
├── xrt.h               # 合并运行时头文件
├── mod_math.c
├── mod_utils.c
├── mod_main.c
├── Makefile             # auto-generated
└── bin/app
```

生成 Makefile：

```makefile
CC = cc
CFLAGS = -O2 -Wall -I.
SRCS = mod_math.c mod_utils.c mod_main.c
OBJS = $(SRCS:.c=.o)
TARGET = app

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^ -lm -lpthread

%.o: %.c xrt.h
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)
```

### D.3 ARC Move 语义优化

**参考**：Nim `injectdestructors.nim` 的 last-use 分析。

在 XIR SSA 上做 last-use 分析很自然：

```
// XIR:
v5 = CALL f(v3)    // v3 is last-used here
v6 = v5            // v5 is last-used here

// Without optimization:
xrt_retain(v3); result = f(v3); xrt_release(v3);
xrt_retain(v5); v6 = v5; xrt_release(v5);

// With move optimization:
result = f(v3);    // v3 ownership transferred to f
v6 = v5;           // v5 ownership transferred to v6
// No retain/release at all!
```

### D.4 增量编译

**策略**：基于文件内容 hash 的增量重编译。

```
1. 对每个 .xr 源文件计算 hash
2. 比对上次编译的 hash（存在 .xray/cache/<hash>/manifest.json）
3. 只重新编译 hash 变化的模块
4. 重新链接
```

---

## 五、参考文件索引

### 每个 Phase 需要参考的源码文件

#### Phase A (运行时修复 + 多模块)

| 参考项目 | 文件 | 参考内容 |
|---------|------|---------|
| **Nim** | `compiler/cgendata.nim` | `BModuleList` 多模块管理结构 |
| **Nim** | `compiler/cgen.nim:60-82` | `findPendingModule()` 跨模块符号查找 |
| **Nim** | `compiler/cgen.nim` 末尾 | `cgenWriteModules()` 输出组装 |
| **V** | `vlib/v/gen/c/cgen.v:36-120` | `Gen` struct — sections builder 设计 |
| **V** | `vlib/v/gen/c/consts_and_globals.v` | 全局变量/常量翻译 |
| **QuickJS** | `quickjs.c` 的 `js_malloc/js_free` | 分配器间接层模式 |

#### Phase B (语言特性)

| 参考项目 | 文件 | 参考内容 |
|---------|------|---------|
| **Nim** | `compiler/ccgstmts.nim` | try/except/finally 翻译 |
| **Nim** | `compiler/ccgexprs.nim` | 表达式翻译 + ARC 插入 |
| **Nim** | `compiler/ccgcalls.nim` | 函数调用 + NRVO 优化 |
| **V** | `vlib/v/gen/c/defer.v` | defer 实现 (flag + LIFO) |
| **V** | `vlib/v/gen/c/match.v` | match → if/else chain 或 switch |
| **V** | `vlib/v/gen/c/struct.v` | struct/class 字段布局 |
| **V** | `vlib/v/gen/c/fn.v` | 函数声明 + 泛型实例化 |
| **QuickJS** | `quickjs.c` 的 `js_array_*` | 集合方法实现 |

#### Phase C (CPS 并发)

| 参考 | 文件 | 参考内容 |
|------|------|---------|
| **xray 自身** | `src/coro/xworker_sched.c` | M:N 调度器 — 直接复用 |
| **xray 自身** | `src/coro/xchannel.c` | Channel (ring + waitq) — 直接复用 |
| **xray 自身** | `src/coro/xtask.c` | Task + scope — 直接复用 |
| **xray 自身** | `src/coro/xworker_exec.c` | 执行分发 — 加 CPS 分支 |
| **xray 自身** | `src/coro/xcoro.c` | 协程创建 — 加 `xr_coro_create_cps()` |
| **Kotlin** | `CancellableContinuation.kt` | CPS 状态机编译参考 |
| **Rust** | async/await → `Pin<Future>` | CPS 状态机生成模式 |
| **Go** | `runtime/proc.go` | GMP 模型（已在 xray 调度器中实现） |

#### Phase D (stdlib + 优化)

| 参考项目 | 文件 | 参考内容 |
|---------|------|---------|
| **Nim** | `compiler/injectdestructors.nim` | ARC last-use / move 优化 |
| **Nim** | `compiler/ccgmerge_unused.nim` | 增量编译合并 |
| **V** | `vlib/v/gen/c/cheaders.v` | C 头文件输出 |
| **QuickJS** | `quickjs-libc.c` | stdlib 绑定模式 |

---

## 六、风险与缓解

| 风险 | 影响 | 缓解方案 |
|------|------|---------|
| XIR passes 间歇性 SIGBUS | 🔴 阻塞 AOT 测试可靠性 | AOT 编译路径禁用后台 JIT；增加 arena debug 校验 |
| ARC 循环引用泄漏 | 🟡 内存泄漏 | Phase A 先不处理；Phase D 加 ORC cycle collector 或 weak ref |
| 多模块重构范围大 | 🟡 工期超出 | 先做单文件多模块（V 风格），再拆分为多文件 |
| CPS 函数着色传播 | 🟡 编译器工作量 | xray 挂起点全部显式，已有 `is_coro_safe` 标记，传播分析自然 |
| `#ifdef XRAY_AOT` 在 src/coro 中的适配 | 🟡 可能触及热路径 | 最小化改动面（~300 行），严格只改执行入口，不动调度逻辑 |

**已消除的风险**（旧方案 v1.0 → 新方案 v2.0）：

| 旧风险 | 消除原因 |
|--------|---------|
| ~~并发运行时复杂度高~~ | 不再重写，复用 15,000 行成熟代码 |
| ~~select 忙等性能差~~ | 直接复用现有阻塞式 select (XrSelectWait + per-bucket queue) |
| ~~简化调度器功能退化~~ | 不简化，原样复用 work stealing / cont stealing / sysmon |
| ~~新 runtime 全是新 bug~~ | 改动仅 ~1,300 行，95% 代码已经过生产压测 |

---

## 七、里程碑与验收标准

| 里程碑 | 验收标准 | 预计时间 |
|--------|---------|---------|
| **A.done** | ≥2 个多模块 import 测试通过；xrt_*.h 无裸 malloc | +2 周 |
| **B.done** | finally/defer/match 测试通过；class 继承测试通过；≥20 个 AOT 测试 | +2 周 |
| **C.done** | CPS 状态机生成 + 复用调度器；go/await/channel/select/scope 测试通过；百万协程压测 | +4 周 |
| **D.done** | math/time/io stdlib 可用；ARC move 优化启用；增量编译 | +2 周 |

**总计**：约 10 周从当前状态到 Phase D 完成。

---

## 八、架构对比：v1.0 (旧) vs v2.0 (新)

| 维度 | v1.0 (旧方案) | v2.0 (新方案) |
|------|--------------|--------------|
| **并发模型** | 简化线程池 (pthread mutex+condvar) | CPS 状态机 + 复用 M:N 调度器 |
| **调度器** | 重写 ~800 行简化版 | 复用 `src/coro/` 15,000 行 |
| **Channel** | 重写 ~400 行 (mutex+condvar) | 复用 `xchannel.c` (lock-free waitq) |
| **select** | 忙等 polling → 后续升级 | 直接复用阻塞式 per-bucket select |
| **scope** | 重写 ~500 行 | 复用 `xtask.c` + `XrScopeContext` |
| **work stealing** | ❌ 无 | ✅ 现有 continuation stealing + LIFO fast dispatch |
| **I/O** | ❌ 无 | ✅ kqueue/epoll/iouring 就绪 |
| **sysmon** | ❌ 无 | ✅ heartbeat 死锁检测 |
| **新代码量** | ~2,300 行全新 | ~1,310 行改动/新增 |
| **功能等价度** | ❌ 大幅降级 | ✅ 100% 语义等价 VM/JIT |
| **新 bug 风险** | 🔴 全新代码 | 🟢 95% 已验证代码 |
