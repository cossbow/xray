# Runtime 控制面与状态归属分析 V2（`024`）

> 这是 `024-runtime-phase0-control-plane.md` 的重写版本。
>
> 写法上严格对齐当前源码，并按"不考虑向后兼容、直接最佳设计"原则给出建议。
> 旧文档不动，本文与之并列存在；最后会做 V1↔V2 取舍。
>
> **范围**：`src/runtime/` 顶层（`xisolate_*`、`xexec_state.h`、`xexec_frame.h`、`xglobals_table.*`、`xshared.h`、`xerror*`、`xstrbuf.*`、`xray_debug_hooks.*`），以及必要消费者：`src/api/xisolate*.c`、`src/api/xvm_compile.c`、`src/vm/xvm_api.c`、`src/vm/xvm_internal.h`、`src/vm/xvm.c`、`src/vm/xvm_helpers.c`、`src/vm/xvm_dispatch_*.inc.c`、`src/runtime/xray_debug_hooks.*`、`src/coro/xcoro.c`、`src/coro/xmachine.c`、`src/module/xmodule.c`。

## 1. 七条核心结论

- **`XrayIsolate` 是 runtime 控制面的状态总装配点。** 真实拥有 GC、system heap、main coroutine、模块、符号、调试、source cache、全局对象、shape registry、扩展类型位图、stdlib cache、profiler 等所有跨执行体共享状态。

- **活跃执行状态的唯一权威入口是 `xr_vm_current_ctx(isolate)`，定义在 `src/vm/xvm_api.c`。** 解析顺序：worker.coro.vm_ctx → worker.vm_ctx → main_coro.vm_ctx → `&isolate->vm_ctx`。前三层是真实活状态，第四层是 bootstrap/static fallback。

- **`XrVMState` 是 storage host，`XrVMContext` 是 access path。这是设计意图，不是过渡。**
  - 单线程模式：`isolate->vm_ctx` 的指针 alias 到 `isolate->vm.*` 的 fixed-size 数组。
  - 多协程模式：每个 `XrCoroutine` / `XrMachine` 自己持有 `XrVMContext`，stack/frames/handlers 单独 malloc。
  - 这套契约写在 `xexec_state.h` 头注释里，源码层面是闭合的。

- **真问题不是契约缺失，而是消费者侧"绕过 ctx"还在多处出现。**
  - VM 直接读 `isolate->debug_hooks`（`xvm.c`、`xvm_dispatch_exception.inc.c`）。
  - VM 直接读 `isolate->current_module`（`xvm_dispatch_module.inc.c`）。
  - runtime error 打印走 ctx 拿 frame，但仍直接读 `isolate->source_cache`（`xvm_helpers.c`）。
  - value/ 层用 TLS fallback `xray_isolate_current()` 取 isolate，导致控制面"反向暴露"。

- **`XrGlobalsTable` 不是闲置，但定位错位。**
  - 真实消费者是 frontend codegen（`xoop_class*`、`xoop_interface.c`），用 `xr_globals_get` 在编译期查全局值。
  - VM 运行期的"全局样"状态走的是 `vm.builtins[]` + `XrSharedArray vm.shared` + `global_object`。
  - 因此它实际上是"compile-time global value lookup"，不是 runtime global storage。

- **真正的执行期 isolate-wide 共享状态远不止"builtins/shared"。**
  - 还包括 `strings_map`（intern 表）、`coro_state`、`runtime`、`main_locals`（REPL 局部）、`defer_stack`、`ctor_call_stack`、`bytes_allocated/next_gc`、JIT 状态等。
  - V1 文档描述偏简化，V2 必须精确到字段。

- **trace_execution 漂移问题真实存在，且能精确定位。**
  - VM dispatch 用 `VM_TRACE` 宏 = `vm_ctx->trace_execution`（`xvm.c:217`）。
  - `xray_isolate_set_trace()` 只写 `params.trace_execution` 和 `vm.trace_execution`。
  - `xr_coro_sync_vm_ctx()` / `xmachine_init` 把 `ctx->trace_execution` 重置为 false。
  - 结果：在 worker / coroutine 路径上 `xray_isolate_set_trace(true)` 不生效。

## 2. 关键模块逐项核对

### 2.1 `xisolate_internal.h` — `XrayIsolate` 完整字段

源码层（`xisolate_internal.h:81`）实际包含 9 个分组。V1 文档列出的字段不完整，V2 全列：

- **核心对象与类型**
  - `core` (`XrayCoreClasses *`)
  - `type_registry` (`XrTypeRegistry *`)
  - `type_infer_context` (`XrTypeInferContext *`)
  - `type_table` (`XrTypeTable *`)
  - `analyzer_pool` (`XrTypePool *`)
  - `current_type_pool` (`XrTypePool *`，per-isolate active pool，替代 TLS)
  - `json_value_type` (`XrType *`，per-isolate cached union type)
  - `native_type_classes[XR_NATIVE_TYPE_MAX]`

- **内存与运行时宿主**
  - `gc` (`XrGC`，fixedgc list 管理器)
  - `sys_heap` (`XrSystemHeap *`，coroutine pool + class arena)
  - `memory_tracker` (`XrMemoryTracker *`)
  - `main_coro` (`XrCoroutine *`，自带 large heap GC，4MB)
  - `shape_entries` / `shape_count` / `shape_capacity`（hidden class registry）
  - `root_shape_cache[32]`（json shape cache）

- **全局与模块**
  - `globals` (`XrGlobalsTable *`)
  - `global_string_pool` (`XrGlobalStringPool *`)
  - `global_object` (`XrGlobalObject *`)
  - `module_registry` (`XrModuleRegistry *`)
  - `current_module` (`XrModule *`，loader 临时槽)
  - `current_storage_mode` (`uint8_t`，0=normal/1=shared)
  - `suppress_exception_print` (`bool`，测试模式)

- **执行 storage（VM）**
  - `vm` (`XrVMState`，inline，详见 §2.2)
  - `vm_ctx` (`XrVMContext`，bootstrap/static fallback ctx)

- **调试**
  - `source_cache` (`XrSourceCache *`)
  - `debug_state` (`void *`, `XrDebugState *`)
  - `debug_hooks` (`void *`, `XrDebugHooks *`)
  - `repl_symbols` (`XrReplSymbolTable *`)
  - `profiler` (`void *`, `VMProfiler *`，编译开关 `XR_ENABLE_VM_PROFILER`)

- **配置**
  - `params` (`XrayIsolateParams`)
  - `config` (`XrayConfig *`)
  - `userdata` (`void *`)
  - `init_flags` (`uint32_t`，`XR_INIT_*`)
  - `current_arena` (`XrArena *`，parser-set，NULL=use malloc)
  - `symbol_table` (`XrSymbolTable *`)

- **集群（可选 `XR_HAS_CLUSTER`）**
  - `cluster` (`void *`, `XrCluster *`)
  - `channel_dist_hooks` (`void *`, `XrChannelDistHooks *`)

- **扩展类型系统（dlopen 包）**
  - `ext_type_next` (`uint8_t`，下一个待分配 type id，从 `XR_TTASK + 1` 起)
  - `ext_type_names[XGC_MAX_TYPES]`
  - `ext_finalize_bitmap` / `ext_has_refs_bitmap` (`uint64_t`)
  - `ext_destroy_funcs[XGC_MAX_TYPES]` / `ext_traverse_funcs[XGC_MAX_TYPES]`

- **stdlib per-isolate cache**
  - `stdlib_cache` (`void *`, `XrStdlibCache *`，lazy)

V1 漏掉的：`channel_dist_hooks`、`stdlib_cache`、`profiler`、`json_value_type`、`current_type_pool`、`current_arena`、`shape_entries`/`root_shape_cache`、`suppress_exception_print`、扩展类型系统全部、`memory_tracker`。

### 2.2 `xexec_state.h` — `XrVMState` 真实字段

V1 仅列了 6 个字段，实际是 14 个分组，分别是：

```c
typedef struct XrVMState {
    XrValue stack[XR_STACK_MAX];                                 // 1
    XrValue *stack_top;
    XrBcCallFrame frames[XR_FRAMES_MAX];                         // 2
    int frame_count;
    int module_base_frame;
    XrExceptionHandler exception_handlers[XR_EXCEPTION_HANDLERS_MAX]; // 3
    int handler_count;
    XrValue current_exception;
    void *strings_map;                                            // 4 interned strings
    XrValue builtins[XR_GLOBALS_MAX];                             // 5 builtins
    int builtin_count;
    XrSharedArray shared;                                         // 6 dynamic shared
    size_t bytes_allocated;                                       // 7 GC bookkeeping
    size_t next_gc;
    bool trace_execution;                                         // 8 debug toggle
    int last_nret;
    XrCtorCallEntry ctor_call_stack[XR_CTOR_CALL_STACK_MAX];      // 9 ctor tracking
    int ctor_call_depth;
#ifdef XRAY_HAS_JIT
    struct XirJitState *jit;                                      // 10 JIT
    int jit_threshold;
    int jit_opt_threshold;
#endif
    void *coro_state;                                             // 11 coroutine
    void *current_coro;
    struct XrMap *main_locals;                                    // 12 REPL locals
    void *runtime;                                                // 13 multi-core
    bool multicore_enabled;
    XrValue *defer_stack;                                         // 14 defer
    int defer_count;
    int defer_capacity;
    int *defer_frame_marks;
} XrVMState;
```

设计上这是 isolate-wide storage host：所有 XR_*_MAX fixed-size 数组都在这里；`shared` 是动态扩容的 `XrSharedArray`。`vm_ctx` 在单线程模式下指针 alias 进这些数组，多协程模式下 worker/coro 各自分配。

### 2.3 `xexec_frame.h` — `XrVMContext` + 其他类型

V1 把这个文件描述为只放 frame/handler/context。**实际还包含 C function 系列类型**：

- `XrBcCallFrame`（call frame，含 `XR_CALL_*` flags）
- `XrExceptionHandler`
- `XrCFuncResult`（DONE / YIELD / BLOCKED / ERROR / CALL_CLOSURE）
- `XrCFunctionPtr` / `XrYieldableCFunctionPtr`（C 函数签名）
- `XrCFuncClass`（FAST / SLOW，sysmon 调度分类）
- `XrCFunction`（带 `XrGCHeader` 的 C 函数对象）
- `XrVMContext`（per-execution-entity 状态）
- `xr_vm_ctx_free_ic_tables()`（声明，定义在 `src/vm/xic*.c`）
- `VMCTX_*` 访问宏

`XrVMContext` 完整字段：

```c
typedef struct XrVMContext {
    // 1. value stack
    XrValue *stack; XrValue *stack_top; int stack_capacity;
    // 2. call stack
    XrBcCallFrame *frames; int frame_count, frame_capacity;
    int module_base_frame;
    // 3. exception handlers
    XrExceptionHandler *handlers; int handler_count, handler_capacity;
    XrValue current_exception;
    // 4. owning coroutine
    void *current_coro;
    // 5. execution control
    uint32_t instruction_count;        // preempt counter
    bool preempt_pending;              // yield at next safe point
    int last_nret;
    bool trace_execution;              // debug toggle (read by VM_TRACE)
    struct XrStrBuf *tmp_strbuf;       // per-ctx scratch buffer
    XrayIsolate *isolate;              // back pointer
    // 6. per-frame struct storage (lazy)
    uint8_t **struct_areas;
    uint16_t *struct_area_caps;
    int struct_areas_cap;
    // 7. struct return arena (frame_count==1 reset)
    uint8_t *struct_ret_arena;
    uint32_t struct_ret_arena_used, struct_ret_arena_cap;
    // 8. per-proto IC tables (immutable proto + ctx-local IC)
    struct XrICFieldTable   **ic_field_tables;
    struct XrICMethodTable  **ic_method_tables;
    struct XrICBuiltinTable **ic_builtin_tables;
    uint32_t ic_tables_capacity;
} XrVMContext;
```

V1 漏掉的：`tmp_strbuf`、`instruction_count`、`preempt_pending`、`struct_ret_arena`，以及对"IC tables 为什么必须 ctx-local"的解释。

### 2.4 `xisolate_api.h` / `.c` — accessor 层

- `xisolate_api.h` 提供 ~50 个 accessor，覆盖 GC、type、class、module、globals、coro、VM state、debug、REPL、扩展类型、TLS。
- `xisolate_api.c` 实现绝大部分；**例外**：
  - `xr_isolate_get_symbol_table()` 声明在 `xisolate_api.h:39`，定义在 `src/api/xisolate.c:196`。这是 V1 指出的边界泄漏，仍未修。
  - `xr_isolate_get_coro_gc()` 为了取 `main_coro->coro_gc`，让 `xisolate_api.c` 必须 include `../coro/xcoroutine.h`。这也让"轻量 accessor 层"事实上非轻量。

### 2.5 `xglobals_table.*` — 真实定位

V1 说"业务消费者很少"是错的。实际消费者：

- `src/api/xisolate.c:84,117,182` — 创建 / 失败回滚 / 销毁
- `src/frontend/codegen/xoop_class.c:21`、`xoop_class_descriptor_builder.c:23`、`xoop_interface.c:19` — 编译期通过 `xr_globals_get` 查类/接口的全局槽

VM dispatch 路径上看不到 `xr_globals_get/set`，因为 VM 走 `OP_GETBUILTIN` → `vm.builtins[]`、`OP_GETSHARED` → `vm.shared.data[]`、property access → `global_object`。

设计契约：`XrGlobalsTable` 是 **compile-time global value index lookup**，由 codegen 在生成字节码时通过 index 查回 isolate 上的全局值。它不是 VM hot path。

但实现里仍有 layer violation：`xr_globals_destroy()` forward-declare 了 `XrEnumType` 并直接调 `xr_enum_type_free()`（`xglobals_table.c:18-20, 45-47`）。

### 2.6 `xshared.h` — shared 对象 refcount 协议

正确的描述：

- shared 对象通过 `xr_shared_init` 把 storage 标志位设成 `XR_GC_STORAGE_SHARED`，refcount 初值 1。
- refcount **复用 `XrGCHeader.gc_next`**（`_Atomic(uintptr_t)`）。条件：shared 对象不挂在 GC 链表上，所以 `gc_next` 闲置可重用。
- `xr_shared_incref` / `xr_shared_decref` 是 atomic 原子操作。decref 返回 0 时调用方负责调用 `xr_shared_destroy`。

设计契约（来自头注释，不在类型系统里显式表达）：

- `shared const x = ...`：原子 refcount，所有 coroutine 可读，immutable。
- `shared let x = ...`：必须经过 `Channel` 串行化访问，禁止直接读。
- 词法作用域，跨作用域同名报错。

V1 没有引用三类语义，V2 应明确写出来，因为这影响后面 deep-copy / channel 分析。

### 2.7 `xerror*` — 名义模块 vs 实际职责

- `xerror.h`：`XrErrorCode` 枚举（lexer/syntax/compile/type/analyze/runtime/memory/io/module/coroutine/json/regex/internal）+ ANSI 颜色宏。
- `xerror_codes.h`：`XR_ERR_*` 三位数 macro（E01xx-E05xx），VM/JIT 运行期用。
- `xerror.c`：完全空文件（`xerror.c:1-13`），头注释说所有 XrResult/XrError 函数已经因为零外部调用方被删。
- 真正的运行期异常机制走 `src/runtime/object/xexception.*` + VM `VM_RUNTIME_ERROR` 宏。

V1 描述对，V2 直接给最佳设计建议：删掉 `xerror.c`，把颜色宏挪到 `base/xansi.h`，错误码模块改名 `xerror_codes.h` → `xerror.h`，原 `xerror.h` 的枚举合并进新 `xerror.h`，整个目录从 4 文件降到 2 文件（甚至 1 个）。

### 2.8 `xstrbuf.*` — per-ctx scratch buffer

V1 完全没提，但这是 `XrVMContext` "per-execution-entity 状态" 的最干净例子：

- `xr_strbuf_tmp(X)`：先取当前 worker 的 `m->vm_ctx`，否则 fallback 到 `xr_isolate_get_vm_ctx(X)`（即 isolate 的 bootstrap ctx）。lazy-allocate `tmp_strbuf` 字段。
- `xr_strbuf_to_string()`：转成 interned `XrString`，自动 reset buffer，可立即复用。
- 释放路径：`xmachine.c:81` 在 `xmachine_destroy` 里释放 worker ctx 的 buffer；coro ctx 的 buffer 由 coro 释放路径处理。

这套设计已经是最佳实践（lazy alloc、auto reset、per-ctx 隔离），V2 应当作为正面示范引用。

唯一的问题：`xr_strbuf_tmp` 内部用了与 `xr_vm_current_ctx` 几乎一样的解析逻辑（`xstrbuf.c:70`），但是自己实现的，不是调 `xr_vm_current_ctx`。这违反了"单一权威 ctx 入口"原则。

### 2.9 `xray_debug_hooks.*` — VM 调试钩子契约

设计干净：

- `XrDebugHooks` 三个回调：`on_line`、`on_exception`、`is_enabled`。
- 没附 debugger 时 hooks 指针为 NULL，VM 零成本跳过。

但消费者侧打破了 accessor 层：

- `xvm.c:179, 353` 直接 cast `isolate->debug_hooks` 为 `XrDebugHooks *`。
- `xvm_dispatch_exception.inc.c:164` 同样直接 cast。

`xisolate_api.h` 已经提供 `xr_isolate_get_debug_hooks()`，但 VM 没用。

## 3. 生命周期与状态归属（精确版）

### 3.1 创建路径（`src/api/xisolate.c:48-123`）

```
xray_isolate_new(params)
  ├─ xr_malloc + memset(0)
  ├─ ext_type_next = XR_TTASK + 1
  ├─ params 注入（包含 init_extra/cleanup_extra 回调）
  ├─ global_string_pool 创建 + xr_global_pool_init
  ├─ xr_gc_init(&gc, isolate)
  ├─ sys_heap 创建 + xr_sysheap_init
  ├─ main_coro = xr_coro_create_bootstrap(isolate)   ← 这里就建好了
  ├─ globals = xr_globals_create(64)
  ├─ xr_vm_init(isolate)                              ← VMState 初始化
  ├─ profiler 分配（XR_ENABLE_VM_PROFILER）
  ├─ xr_shape_registry_init(isolate)
  ├─ params.init_extra(isolate)                       ← full runtime 才接
  └─ xray_isolate_enter(isolate)                       ← 写 TLS
```

### 3.2 full runtime（`src/api/xisolate_full.c:49-141`）

`init_extra` 回调串了 17 步：

1. `xr_malloc(XrayConfig)` + `xr_config_init`
2. `xr_type_global_init`（process-level idempotent）
3. `analyzer_pool = xr_type_pool_new()`
4. 直接写 `current_type_pool = analyzer_pool`
5. **同时**调 `xr_type_set_current_pool` 写 TLS（双轨）
6. `symbol_table = xr_symbol_table_create()` + init_builtins
7. `xr_registry_init(isolate)`
8. `xr_core_init(isolate)`（创建 Object/String/Array/Map/Set/Json 等核心类）
9. `xr_reflect_api_init`
10. `xr_json_api_init`
11. `global_object = xr_global_object_create(isolate)`
12. 注册 core classes + builtin functions 到 global object
13. `xr_module_system_init`
14. 注入 compiler hooks（parser/compile_ast/compile_source/program_destroy）
15. `xr_regex_init_native_type`
16. `source_cache = xr_source_cache_new()`
17. 把 core classes 写进 `vm.builtins[XR_GLOBAL_VAR_*]`，并把 `vm.builtin_count` 推到 `XR_USER_GLOBALS_START`

第 4-5 步是控制面"双轨"的真实例子：显式 isolate 字段 + TLS fallback 同时存在。

### 3.3 销毁路径

- `xray_isolate_delete()`：profile_report → exit TLS → free main_coro → cleanup_extra(`isolate_cleanup_full`) → vm_cleanup → shape_destroy → gc_cleanup → sysheap_destroy → string_pool_free → stdlib_cache_free → globals_destroy → config_free → free isolate。
- `isolate_cleanup_full()` 顺序：stdlib_cache_free → source_cache_free → module_system_free → global_object_destroy → core_free → registry_free → symbol_table_destroy → analyzer_pool_free → repl_symbols_free。

顺序敏感点：`stdlib_cache_free` 必须在 `module_system_free` 之前，因为它持有 stdlib 注册的 shape 引用。

### 3.4 主协程与 live `vm_ctx`

- `xr_coro_create_bootstrap(isolate)` 在 isolate 创建时就建好 main coroutine。
- `xr_execute()`：用 main_coro 创建主闭包 → `xr_coro_setup_main(main_coro, isolate, closure)` → 内部调 `xr_coro_sync_vm_ctx()`。
- `xr_coro_sync_vm_ctx()`（`xcoro.c:101-107`）重置 `instruction_count`、`preempt_pending`、`last_nret`、`trace_execution=false`、`isolate=X`。**stack/frames/handlers 不在这里重置**。
- 之后所有 VM dispatch 走 `vm_ctx->*`，因此 main coroutine 的 ctx 才是真实活状态。

### 3.5 `xr_vm_current_ctx` 解析顺序（`xvm_api.c:28-50`）

```
1. xr_current_worker() 有效 ⇒ worker 当前 coro 有效 ⇒ &coro->vm_ctx
2. xr_current_worker() 有效 ⇒ &worker->m->vm_ctx
3. isolate->main_coro 有效 ⇒ &main_coro->vm_ctx
4. &isolate->vm_ctx           // bootstrap/static fallback
```

注释明确指出：**前三层是真实活状态，第四层只在 init/teardown 阶段使用**。其他代码必须 ctx 入口拿，不允许读 `isolate->vm.{stack,frames,handlers}` 当 live。

## 4. 边界判断（精确化）

### 4.1 已经成立的设计边界

| 边界 | 体现 |
|---|---|
| VM 不反向 include runtime | `xexec_frame.h` / `xexec_state.h` 都在 `src/runtime/` |
| VMState/VMContext 二分 | storage host vs access path（`xexec_state.h` 头注释） |
| `xr_vm_current_ctx` 单一入口 | `xvm_api.c` 实现 + `xvm_internal.h` 注释强约束 |
| coroutine 释放 IC 不依赖 vm/ | `xr_vm_ctx_free_ic_tables` 在 runtime 层声明，vm/ 实现 |
| 扩展类型 dlopen 隔离 | `xisolate_api.h` 显式 API（`xr_alloc_extension_type` 等） |

### 4.2 真实的边界泄漏（按消费者侧定位）

| 泄漏点 | 文件 | 行 | 应替换为 |
|---|---|---|---|
| 直接 `isolate->debug_hooks` cast | `xvm.c` | 179, 353 | `xr_isolate_get_debug_hooks()` |
| 直接 `isolate->debug_hooks` cast | `xvm_dispatch_exception.inc.c` | 164 | 同上 |
| 直接 `isolate->source_cache` 读 | `xvm_helpers.c` | 48, 62, 64 | `xr_isolate_get_source_cache()` |
| 直接 `isolate->current_module` 读 | `xvm_dispatch_module.inc.c` | 93 | `xr_isolate_get_current_module()` |
| `xr_isolate_get_symbol_table` 跨边界实现 | `xisolate.c` | 196 | 移回 `xisolate_api.c` |
| `xglobals_destroy` 调 enum free | `xglobals_table.c` | 45-47 | enum 用 destructor 钩，table 不应 typecheck |
| TLS isolate 在 value 层 | `xvalue.c`, `xvalue_print.c`, `xtype.c`, `xjson_pool.c` | 多处 | 显式 `XrayIsolate *X` 参数 |
| `xr_strbuf_tmp` 自实现 ctx 解析 | `xstrbuf.c` | 70 | 调 `xr_vm_current_ctx` |
| 双轨：current_type_pool + TLS | `xisolate_full.c` | 63-66 | 删 TLS 路径，统一 isolate 字段 |

### 4.3 死代码 / 空声明

- `xerror.c` 整个文件无定义（13 行注释）
- `xray_isolate_init_common()` / `xray_isolate_cleanup_common()` 在 `xisolate_internal.h:264, 267` 声明，无任何定义和调用方

## 5. 已确认的高风险点

### 5.1 `trace_execution` 漂移（精确链路）

```
xray_isolate_set_trace(X, true)   写 X->params.trace_execution + X->vm.trace_execution
                                                                          │
                                                                          │ 永远是 isolate 的 bootstrap state
                                                                          ▼
xr_execute() / xr_coro_setup_main() ─→ xr_coro_sync_vm_ctx(X, main_coro)
                                                  │
                                                  └─ ctx->trace_execution = false  ★
                                                  
VM dispatch:    VM_TRACE = vm_ctx->trace_execution ★
```

主协程一进入执行就把 ctx 的开关清成 false，VM 永远读不到用户设的 true。worker / coro 路径同理（`xmachine.c:67`）。

修复：`xr_coro_sync_vm_ctx` 应当读 `isolate->vm.trace_execution` 同步进 ctx；或者更彻底，把 trace 配置直接放到 `XrayIsolate.params`，让 VM 读 isolate 而不是 ctx。

### 5.2 `module_registry` 模块执行后兜底恢复

`xmodule.c:885-892`：

```c
void *saved = xr_isolate_get_module_registry(isolate);
int result = xr_vm_execute_module(isolate, code);
if (!xr_isolate_get_module_registry(isolate) && saved) {
    xr_isolate_set_module_registry(isolate, saved);
}
```

含义：模块代码执行可能把 `isolate->module_registry` 改成 NULL。这是非常强的"VM 副作用污染控制面"信号。

### 5.3 `current_module` 单槽位上下文

- 模块 loader 写入 → VM 执行隐式读取 → 完成后 restore。
- 多 worker / 嵌套 module 加载会撞同一槽。

正确设计：把 current_module 作为 ctx 字段（`XrVMContext.current_module`），随调用栈走，loader 直接传参不需要写 isolate。

### 5.4 `source_cache` 在 module 加载路径不闭合

- `xisolate_scripting.c:170, 208` 显式 `xr_source_cache_add(filename, source)`（dostring/dofile）。
- `xmodule.c` 的模块加载路径**没有**调 `xr_source_cache_add`。
- 后果：imported module 的 runtime error 找不到源行，`xvm_helpers.c:62` 取 `xr_source_cache_get_line` 返回 NULL，错误提示退化。

修复：模块 loader 在解析源代码后立即 add 到 source cache。

### 5.5 TLS isolate 在 value 层穿透

- `xvalue.c:325`、`xvalue_print.c:46, 189, 314`、`xtype.c:86, 92, 99, 714`、`xjson_pool.c:27`。
- 这些函数都有"想接 isolate 但参数没传"的形态，靠 `xray_isolate_current()` 兜底。
- 真实问题：value 层不应该需要 isolate；需要的是 type pool / json shape 这种依赖。

修复路径见 §6.4。

### 5.6 死代码契约漂移

- `xerror.c` 应当删除。
- `xray_isolate_init_common/cleanup_common` 应当删声明。
- `XrError` typedef（`xerror.h:149`）已无消费者，应当删。

## 6. 最佳设计建议（不考虑兼容性）

按用户原则"避免技术债务，每阶段选最佳设计"，给出**直接可执行**的最佳设计。

### 6.1 收紧 ctx 权威入口

- **删除所有"直接读 `isolate->vm.{stack,frames,handlers,current_exception}`" 路径**，要求消费者全部走 `xr_vm_current_ctx(isolate)`。
- VM init / teardown 路径例外，但要在头注释明确标记。
- 增加 `XR_DCHECK` 在所有 ctx 入口做"是否有有效 worker / coro"断言。

### 6.2 `XrVMState` 收成纯 storage host

把 VMState 字段分成两类：

- **真正 isolate-wide**：`builtins[]`、`builtin_count`、`shared`、`strings_map`、`coro_state`、`runtime`、`multicore_enabled`、`jit*`、`bytes_allocated`、`next_gc`、`main_locals`。
- **仅 bootstrap fallback**：`stack[]`、`stack_top`、`frames[]`、`frame_count`、`module_base_frame`、`exception_handlers[]`、`handler_count`、`current_exception`、`trace_execution`、`last_nret`、`ctor_call_stack`、`ctor_call_depth`、`defer_*`。

第二类应当**改名**为 `XrBootstrapVMSlots`，物理放在 `XrayIsolate.bootstrap_vm`，避免读者误以为是 isolate-wide 状态。`isolate->vm_ctx` 在静态模式下 alias 进 `bootstrap_vm.*`。

### 6.3 `XrGlobalsTable` 显式定位为 codegen 辅助结构

- 改名 `XrCodegenGlobalsTable`。
- 头注释明确：仅 codegen 期间使用，VM 不读。
- 移除 `xr_globals_destroy()` 里的 enum_type free，改成调用方在销毁 globals 之前先扫一遍 enum。
- 或者更激进：把 codegen 全局值改为编译期在 `global_object` 上注册，彻底删除 `XrGlobalsTable`。

### 6.4 value 层去 TLS

- `xvalue.c` / `xvalue_print.c` / `xtype.c` / `xjson_pool.c` 所有用 `xray_isolate_current()` 的地方都改为显式 `XrayIsolate *X` 参数。
- 调用方负担：API 增加一个参数，VM 路径上 ctx 已经持有 isolate，所以零成本。
- 删 `xtype.c` 的 `xr_type_set_current_pool()` TLS 写入路径，统一走 isolate 字段。
- `xjson_pool.c` 的 `xr_json_shape(X, json)` 已经接受 isolate，去掉 fallback。

### 6.5 trace 配置走 ctx-pull 模型

两种方案选其一：

- **方案 A**：`xr_coro_sync_vm_ctx` 读 `isolate->vm.trace_execution` 同步进 ctx。简单，但每次 sync 都要拷一次。
- **方案 B**：删掉 `XrVMContext.trace_execution`，VM_TRACE 宏改为 `(vm_ctx->isolate->vm.trace_execution)`。零拷贝，永远生效。

推荐方案 B。trace 不是 hot path，多一次间接寻址无所谓。

### 6.6 `current_module` 入栈

- `XrVMContext` 加 `XrModule *current_module`。
- module loader 不再写 isolate，而是 `xr_vm_execute_module(isolate, code, parent_module)` 显式传。
- 删除 `xmodule.c:885-892` 的兜底恢复代码。

### 6.7 source_cache 由 module loader 主动写

`xmodule.c` 在解析源代码 → 编译之前调 `xr_source_cache_add`。

### 6.8 删死代码

- `xerror.c`（空文件）
- `xerror.h` 里的 `XrError` typedef + 颜色宏（拆到 `base/xansi.h`）
- `xerror_codes.h` 合并进 `xerror.h`
- `xray_isolate_init_common/cleanup_common` 声明

### 6.9 `xshared.h` refcount 显式化

把 `gc_next` 复用 refcount 写成 union：

```c
typedef struct XrGCHeader {
    ...
    union {
        struct XrGCHeader *gc_next;     // managed objects
        _Atomic(uintptr_t)  shared_refc; // shared objects
    } link;
} XrGCHeader;
```

类型系统直接表达，不再靠注释维护。

### 6.10 debug_hooks accessor 强约束

- VM 路径全部走 `xr_isolate_get_debug_hooks(isolate)`，禁止直接 `(XrDebugHooks*)isolate->debug_hooks`。
- 用一个 inline accessor 返回强类型：

```c
static inline XrDebugHooks *xr_get_debug_hooks(XrayIsolate *X) {
    return (XrDebugHooks *)(X ? X->debug_hooks : NULL);
}
```

## 7. 状态归属表（修订版）

| 状态 | 真实所有者 | 创建路径 | 活跃消费者 | 销毁路径 |
|---|---|---|---|---|
| `gc` | isolate | `xr_gc_init` | runtime/object/gc | `xr_gc_cleanup` |
| `sys_heap` | isolate | `xr_sysheap_init` | coro pool, class arena | `xr_sysheap_destroy` |
| `main_coro` | isolate | `xr_coro_create_bootstrap` | execute、live ctx fallback、stats | `xr_coro_free` |
| `vm` (VMState) | isolate | `xr_vm_init` | builtins、shared、JIT、coro_state、bootstrap | `xr_vm_cleanup` |
| live `vm_ctx` | worker / coro | `xmachine_init` / `xr_coro_sync_vm_ctx` | VM、JIT、IC、tmp_strbuf、struct arena | `xr_vm_ctx_free_ic_tables` + 协程释放 |
| `globals` (codegen) | isolate | `xr_globals_create` | frontend codegen | `xr_globals_destroy` |
| `global_string_pool` | isolate | `xr_global_pool_init` | string intern | `xr_global_pool_free` |
| `global_object` | isolate | `xr_global_object_create` | core classes、builtins | `xr_global_object_destroy` |
| `module_registry` | isolate | `xr_module_system_init` | loader、import、export | `xr_module_system_free` |
| `current_module` | isolate（应入 ctx） | loader 临时写入 | export collection、execute | restore 兜底 |
| `source_cache` | isolate | `xr_source_cache_new` | runtime error 打印 | `xr_source_cache_free` |
| `debug_hooks` | isolate | DAP `xr_debug_register_hooks` | VM line / exception 安全点 | isolate delete |
| `symbol_table` | isolate | `xr_symbol_table_create` | compiler、serializer | `xr_symbol_table_destroy` |
| `shape_*` | isolate | `xr_shape_registry_init` | hidden class、json | `xr_shape_registry_destroy` |
| `analyzer_pool` | isolate | `xr_type_pool_new` | analyzer / 编译期 | `xr_type_pool_free` |
| `current_type_pool` | isolate | `isolate_init_full` 内直写 | type infer | isolate delete 隐式 |
| `core` | isolate | `xr_core_init` | core classes | `xr_core_free` |
| `type_registry` | isolate | `xr_registry_init` | reflection | `xr_registry_free` |
| `repl_symbols` | isolate | REPL 启动 | REPL incremental | `xr_repl_symbols_free` |
| `profiler` | isolate | `xr_calloc` | VM dispatch（编译开关） | `xr_free` + drain report |
| `stdlib_cache` | isolate | lazy first-use | stdlib（json/yaml/io.stat 等） | `xr_stdlib_cache_free` |
| `cluster` / `channel_dist_hooks` | isolate | cluster 启动 | 分布式 channel | cluster shutdown |
| `ext_*`（dlopen 类型） | isolate | `xr_alloc_extension_type` 等 | GC traverse / finalize | isolate delete |
| `XrSharedArray vm.shared` | isolate | `xr_shared_array_init` | OP_GETSHARED / OP_SETSHARED | `xr_shared_array_free` |

## 8. 给下一轮 `value/` 分析的输入

下一轮重点（按 V2 视角）：

- 量化 `xvalue.c` / `xvalue_print.c` / `xtype.c` / `xjson_pool.c` 用 TLS isolate 的所有点，分类：能改显式参数的、必须保留 fallback 的。
- 检查 `xtype.c` 的 type pool 双轨（`current_type_pool` 字段 + TLS）实际有没有竞争。
- `xvalue_format.c` 是否已经从"值层工具"演化成横切服务，跨 layer 调用了多少。
- `json shape` / `type pool` / `json_value_type` 这些 per-isolate cache 在 value 层的边界。
- 量化"哪些 value-layer API 接受 `XrayIsolate *X == NULL`"，按"安全 / 不安全"分类。

## 9. 与 V1 的主要差异

| 点 | V1 | V2 |
|---|---|---|
| `XrGlobalsTable` 定位 | "几乎闲置" | codegen-time global value lookup（frontend 实际用） |
| `XrVMState`/`XrVMContext` 定位 | "职责分裂还没收口" | "storage host vs access path"是设计意图；问题在消费者绕开 |
| `xexec_state.h` 字段表 | 6 项 | 14 个分组，全列 |
| `xexec_frame.h` 字段表 | frame/handler/ctx | 还含 C function 类型族（XrCFunction 等） |
| `XrVMContext` 字段 | 部分 | 全列（含 tmp_strbuf、struct_ret_arena、preempt） |
| `xstrbuf` | 未提 | 列为正面示范 + 唯一的小问题（自实现 ctx 解析） |
| `xshared.h` shared 三类语义 | 未引用 | 完整列出（const / let / channel） |
| trace 漂移 | 风险点 | 精确链路 + 修复方案 |
| `xerror*` | "残留兼容壳" | 死代码，给具体合并路线 |
| 改进建议 | 5 条方向性 | 10 条可执行（按"无兼容性"原则） |

## 10. 本轮状态

- **已完成**
  - `src/runtime/` 顶层 + 关键消费者源码逐文件复核
  - 与 V1 全部论断对账，标出准确 / 不准确 / 不完整
  - 修订状态归属表
  - 给出 10 条最佳设计建议

- **未完成**
  - `src/runtime/value/` 详细 TLS 穿透计数（留给 025 V2）
  - module loader 是否补 source_cache 的实测 patch 验证

- **下一步**
  - 进入 `025-runtime-phase1-value-v2.md`
