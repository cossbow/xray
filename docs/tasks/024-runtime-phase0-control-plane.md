# Runtime 控制面与状态归属分析（`024`）

> 按《Runtime Module Analysis Plan》执行的第 0 轮。
>
> 本轮只处理 runtime 控制面，不深入 `value/`、`gc/`、`object/`、`class/`、`symbol/`、`src/coro/` 的语义细节。
>
> **范围**：`src/runtime/xisolate_*`、`xexec_*`、`xglobals_table.*`、`xshared.h`、`xerror.*`，以及必要消费者 `src/api/xisolate*.c`、`src/api/xvm_compile.c`、`src/vm/xvm_api.c`、`src/vm/xvm_internal.h`、`src/vm/xvm_helpers.c`、`src/runtime/xray_debug_hooks.*`、`src/coro/xcoro.c`、`src/module/xmodule.c`。

## 1. 本轮结论

- **`XrayIsolate` 是 runtime 的真实控制面**。
  - 生命周期、主协程、GC 核心、模块系统、符号表、调试挂钩、source cache、全局对象、extension type 位图都挂在这里。

- **活跃执行状态的权威入口不是 `isolate->vm`，而是 `xr_vm_current_ctx()`**。
  - 当前契约已经写在 `src/vm/xvm_internal.h`：先取 worker 当前协程 `vm_ctx`，再取 worker `vm_ctx`，再退回 `main_coro->vm_ctx`，最后才是 `&isolate->vm_ctx`。

- **`XrVMState` 与 `XrVMContext` 的职责已经分裂，但控制面还没完全收口**。
  - `XrVMState` 里保留了 builtins/shared、scheduler、runtime、JIT、bootstrap stack/frame 等。
  - `XrVMContext` 才承载真实执行 ABI：stack、frames、handlers、current_exception、current_coro、IC tables、struct area。

- **`xisolate_api` 试图把 `xisolate_internal.h` 从星型依赖中心降级成实现细节，但落地还不彻底**。
  - 仍有多条路径直接读写 `isolate->debug_hooks`、`isolate->source_cache`、`isolate->current_module`。

- **控制面里已经出现“状态漂移”信号**。
  - `module_registry` 在模块执行后需要兜底恢复。
  - `trace_execution` 的 isolate 级配置与 live `vm_ctx` 同步不一致。
  - `source_cache` 的分配、填充、消费路径不闭合。

- **`src/coro` 已经进入控制面范围**。
  - `main_coro` 的创建、升级、`vm_ctx` 同步都在 `xcoro.c`，因此后面分析 runtime 时必须把 `src/coro` 一并算进来。

## 2. 关键职责

### 2.1 `XrayIsolate` 负责什么

`src/runtime/xisolate_internal.h` 当前聚合了以下控制面状态：

- **对象与类型**
  - `core`
  - `type_registry`
  - `type_infer_context`
  - `type_table`
  - `analyzer_pool`
  - `current_type_pool`
  - `json_value_type`
  - `native_type_classes[]`

- **内存与运行时宿主**
  - `gc`
  - `sys_heap`
  - `memory_tracker`
  - `main_coro`
  - `shape_entries/root_shape_cache`

- **全局与模块**
  - `globals`
  - `global_string_pool`
  - `global_object`
  - `module_registry`
  - `current_module`
  - `current_storage_mode`

- **执行与调试**
  - `vm`
  - `vm_ctx`
  - `source_cache`
  - `debug_state`
  - `debug_hooks`
  - `repl_symbols`
  - `profiler`

- **扩展与附加能力**
  - `cluster`
  - `channel_dist_hooks`
  - extension type bitmaps/callbacks
  - `stdlib_cache`

结论：`XrayIsolate` 不是单纯的 public API handle，而是 runtime 的**状态总装配点**。

### 2.2 `xexec_state.h` 与 `xexec_frame.h` 负责什么

这两个头文件承担的是**执行 ABI**，不是单纯的 VM 私有定义。

- **`xexec_state.h`**
  - 定义 `XrVMState`
  - 保存 builtins/shared、scheduler 指针、runtime 指针、JIT 指针、bootstrap stack/frame/handlers

- **`xexec_frame.h`**
  - 定义 `XrBcCallFrame`
  - 定义 `XrExceptionHandler`
  - 定义 `XrVMContext`
  - 声明 `xr_vm_ctx_free_ic_tables()`，让 `src/coro` 可以释放 IC tables，而不必反向 include `src/vm`

这说明当前设计已经默认：**VM/JIT 是执行者，不是执行状态的拥有者**。

## 3. 生命周期与状态归属

### 3.1 Isolate 创建路径

`src/api/xisolate.c:xray_isolate_new()` 的主流程是：

1. 分配并清零 `XrayIsolate`
2. 初始化 `ext_type_next`、`params`、`userdata`、`init_flags`
3. 创建 `global_string_pool`
4. 初始化 `gc`
5. 创建 `sys_heap`
6. 创建 bootstrap `main_coro`
7. 创建 `globals`
8. 执行 `xr_vm_init()`
9. 分配 profiler（按编译开关）
10. 初始化 shape registry
11. 调用 `params.init_extra`（full runtime 才接上）
12. `xray_isolate_enter(isolate)`，写入 TLS

### 3.2 full runtime 附加初始化

`src/api/xisolate_full.c:isolate_init_full()` 继续装配：

- `config`
- `analyzer_pool`
- `current_type_pool`
- symbol table
- type registry
- core class system
- reflect/json API
- global object
- module system
- source cache
- builtins 数组中的核心类入口

这里还有一个值得注意的细节：

- `current_type_pool` 已经挂到 isolate 上
- 但初始化期间仍会调用 `xr_type_set_current_pool(...)` 写 TLS fallback

这说明**显式 isolate 所有权**与**TLS fallback** 当前是并存模型。

### 3.3 主协程与 live `vm_ctx`

主协程不是第一次执行时才创建，而是在 isolate 初始化时就由 `xr_coro_create_bootstrap()` 创建。

之后 `xr_execute()`：

1. 用 `main_coro` 创建主闭包
2. 调 `xr_coro_setup_main(main_coro, isolate, closure)`
3. `xr_coro_setup_main()` 内部再调 `xr_coro_sync_vm_ctx()`
4. 最终运行解释器或 runtime 主线程

`xr_coro_sync_vm_ctx()` 会重置：

- `stack_top`
- `frame_count`
- `module_base_frame`
- `handler_count`
- `current_exception`
- `current_coro`
- `instruction_count`
- `preempt_pending`
- `last_nret`
- `trace_execution`
- `isolate`

因此**主协程的 `vm_ctx` 才是单线程运行时的真实活状态**，而不是 `isolate->vm`。

### 3.4 `xr_vm_current_ctx()` 是唯一权威入口

`src/vm/xvm_internal.h` 与 `src/vm/xvm_api.c` 已明确给出解析顺序：

1. 当前 worker 的 `M.vm_ctx.current_coro -> coro->vm_ctx`
2. 当前 worker 的 `M.vm_ctx`
3. `isolate->main_coro->vm_ctx`
4. `&isolate->vm_ctx`

这条契约非常关键，因为它直接说明：

- `isolate->vm.stack/frame/handlers` 只是 bootstrap/static fallback
- 一旦进入协程或 worker 执行，**live stack/frame/exception 状态就不再由 `isolate->vm` 持有**

## 4. 边界判断

### 4.1 当前的正面边界

- **`xexec_frame.h` 放在 runtime 层是正确的**
  - 它让 `src/coro`、`src/vm`、`src/jit` 共用执行 ABI，而不是让 `coro/runtime` 反向 include `vm`

- **`xisolate_api.h` 作为轻量 accessor 层方向是对的**
  - module、debug hook 注册、部分 runtime 模块可以不直接看 `struct XrayIsolate`

### 4.2 当前的边界泄漏

- **VM 直接碰 `isolate->debug_hooks`**
  - `src/vm/xvm.c`
  - `src/vm/xvm_dispatch_exception.inc.c`

- **runtime error 打印直接碰 `isolate->source_cache`**
  - `src/vm/xvm_helpers.c`

- **VM module dispatch 直接碰 `isolate->current_module`**
  - `src/vm/xvm_dispatch_module.inc.c`

- **`xr_isolate_get_symbol_table()` 的声明在 `runtime/xisolate_api.h`，实现却在 `src/api/xisolate.c`**
  - accessor 边界被拆到 runtime/api 两处，说明控制面收口还不干净

- **`xr_isolate_get_coro_gc()` 为了拿 `main_coro->coro_gc`，在 `runtime/xisolate_api.c` 直接 include `../coro/xcoroutine.h`**
  - 这让“轻量 accessor 层”并不轻

## 5. 依赖与控制面热点

### 5.1 `XrGlobalsTable` 不是当前主路径权威全局存储

本轮 grep 的结果比较意外：

- `xr_globals_*()` 几乎没有业务消费者
- `isolate->globals` 的观察到的引用基本只在创建、销毁、accessor

与此相对，真正活跃的“全局样”状态却在：

- `isolate->vm.builtins[]`
- `isolate->vm.shared`
- `global_object`

这意味着当前控制面至少有**两套全局状态模型**：

- `XrGlobalsTable`
- `vm.builtins/shared + global_object`

后面必须确认 `XrGlobalsTable` 是尚未接通、保留给某条特定路径，还是已经事实性闲置。

### 5.2 `xglobals_table.c` 有低层泄漏高层语义的问题

`xglobals_table.c` 自己写了：

- globals 比 class 更低层
- 为避免 layer violation，只做 forward declaration

但实现里仍然会：

- 检查 `XR_TENUM_TYPE`
- 调 `xr_enum_type_free()`

这本质上仍是**globals table 知道 enum 的具体释放语义**，只是把 include 关系藏起来了，没有真正清掉依赖。

### 5.3 `xshared.h` 是低层协议头，但承载了隐式布局契约

`xshared.h` 用 `gc_next` 复用 shared refcount：

- shared 对象不走普通 GC linked list
- refcount 存在 `gc_next`
- 通过 `_Atomic(uintptr_t)` 操作

这设计很轻，但也意味着：

- shared 生命周期协议依赖 `XrGCHeader` 内部布局
- 这个契约不在类型系统里显式体现，而是靠注释与 helper 函数维护

### 5.4 `xerror.*` 已经不是 runtime error 的单一边界

当前状态是：

- `xerror.h`：分析器/编译期错误码 + ANSI 颜色
- `xerror_codes.h`：VM/JIT runtime error code 宏
- `xexception.*` / VM 宏：真正的运行期异常传播
- `xerror.c`：空文件，仅保留“dead code removal 后为空”的说明

因此 `xerror.*` 在控制面里更像**残留兼容壳**，而不是权威 runtime error 模块。

## 6. 状态归属表

| 状态 | 真实所有者 | 初始化路径 | 活跃消费者 | 清理路径 |
|---|---|---|---|---|
| `gc` | `XrayIsolate` | `xray_isolate_new` | GC/object/runtime | `xr_gc_cleanup` |
| `sys_heap` | `XrayIsolate` | `xray_isolate_new` | coro/class/module | `xr_sysheap_destroy` |
| `main_coro` | `XrayIsolate` | `xr_coro_create_bootstrap` | execute、GC stats、live ctx fallback | `xr_coro_free` |
| `vm` | `XrayIsolate` | `xr_vm_init` | builtins/shared/JIT/runtime/bootstrap | `xr_vm_cleanup` |
| live `vm_ctx` | `worker` 或 `coro` | `init_vm_context` / `xr_coro_sync_vm_ctx` | VM/JIT/runtime error path | `xr_vm_ctx_free_ic_tables` + coro/free |
| `module_registry` | `XrayIsolate` | `xr_module_system_init` | module loader、VM module glue | `xr_module_system_free` |
| `current_module` | `XrayIsolate` | module load 临时设置 | export collection、module execute | 加载路径 restore |
| `source_cache` | `XrayIsolate` | `isolate_init_full` | runtime error printing | `xr_source_cache_free` |
| `debug_hooks` | `XrayIsolate` | debugger register | VM line/exception safe points | isolate free |
| `symbol_table` | `XrayIsolate` | `isolate_init_full` | compiler/serializer | `xr_symbol_table_destroy` |
| `globals` | `XrayIsolate` | `xray_isolate_new` | 当前扫描到的业务消费很少 | `xr_globals_destroy` |

## 7. 已确认高风险点

### 7.1 `vm_ctx` 契约已存在，但控制面没有彻底收口

风险不是“没有权威入口”，而是：

- 权威入口已经有了：`xr_vm_current_ctx()`
- 但并不是所有消费者都真正围绕它组织

后续任何直接把 `isolate->vm.stack/frame/handlers` 当 live 状态读写的代码，都很容易在 worker/coro 场景下读到**陈旧镜像**。

### 7.2 `trace_execution` 存在状态漂移

`xray_isolate_set_trace()` 当前只更新：

- `isolate->params.trace_execution`
- `isolate->vm.trace_execution`

但 `xr_coro_sync_vm_ctx()` 会把 live `ctx->trace_execution` 重置为 `false`。

如果 VM 真正读取的是 `vm_ctx->trace_execution`，那当前 trace 开关在真实协程执行路径上就存在明显漂移风险。

### 7.3 `module_registry` 需要执行后兜底恢复

`src/module/xmodule.c` 在 `xr_vm_execute_module()` 前后会：

- 保存 `saved_module_registry`
- 执行 module code
- 若执行后 registry 变成 NULL，再手动恢复

这说明当前**模块执行路径可能破坏 isolate 控制面状态**，属于很强的架构异味。

### 7.4 `current_module` 是 isolate 全局槽位，属于脆弱上下文

`current_module` 现在的用法是：

- module loader 临时写入 isolate
- VM export/execute 路径隐式读取
- 结束后再 restore

这意味着它是一个**进程内单槽位上下文**，而不是显式参数传递。对后面多 worker / module 语义收敛来说，这种设计非常脆弱。

### 7.5 `source_cache` 的分配、写入、读取没有完全闭环

当前观察到：

- `isolate_init_full()` 分配 `source_cache`
- `api/xisolate_scripting.c` 调 `xr_source_cache_add()` 填充 entry script/source
- `xr_runtime_error()` 直接读取 `isolate->source_cache`

但本轮没有看到 `xmodule.c` 的模块加载路径显式补 `xr_source_cache_add()`。

这意味着 imported module 的 runtime error 上下文是否完整，目前至少值得下一轮顺手核实。

### 7.6 TLS fallback 还在穿透到 value/type 路径

`xray_isolate_current()` / `xray_isolate_get_current()` 当前被这些路径直接使用：

- `runtime/value/xvalue.c`
- `runtime/value/xvalue_print.c`
- `runtime/value/xtype.c`
- `runtime/object/xjson_pool.c`

这说明“显式传 isolate”还没有完全成为底层统一契约。后面分析 `value/` 时要把这些点当成重点看。

### 7.7 `xray_isolate_init_common()` / `xray_isolate_cleanup_common()` 只有声明，没有观察到实现

`xisolate_internal.h` 仍声明这两个函数，但本轮源码扫描没有找到对应定义与调用链。

这属于明显的**控制面契约漂移**，要么删声明，要么补回真实实现与用途。

## 8. 改进建议

### 8.1 先把控制面权威入口收紧

建议把下面三条明确成硬约束：

- **live 执行态只通过 `xr_vm_current_ctx()` 取**
- **非装配代码默认不用 `xisolate_internal.h`，优先用 accessor**
- **TLS isolate 只做 fallback，不做权威状态来源**

### 8.2 明确 `XrVMState` 与 `XrVMContext` 的边界

推荐后续按下面方式收口：

- `XrVMState`
  - builtins/shared
  - scheduler/runtime/JIT
  - bootstrap fallback 存储

- `XrVMContext`
  - stack/frame/handler/current_exception/current_coro
  - preempt/trace/IC tables/struct areas

核心目标是：**不再让“状态镜像”看起来像“状态所有者”**。

### 8.3 决定 `XrGlobalsTable` 的去留

当前更像三选一问题：

- 真正接通并明确职责
- 合并到更清晰的 global storage 设计
- 如果已事实性闲置，就删掉

不能继续维持“isolate 一定分配，但业务几乎不读写”的状态。

### 8.4 让 module/source/debug 都走同一层 accessor 习惯

优先收口这些访问：

- `debug_hooks`
- `source_cache`
- `current_module`
- `module_registry`

至少应该做到：**消费者不再直接碰这些字段，统一走 accessor 或专门 helper**。

### 8.5 把 trace 配置同步问题作为短平快修复项单独跟踪

如果后面验证 `vm_ctx->trace_execution` 确实没有随 isolate trace 设置同步，就应该单独修。

这类问题不需要等大重构，属于控制面状态同步缺口。

## 9. 给下一轮 `value/` 分析的输入

下一轮重点不要只盯 `XrValue` 位布局，还要盯这些控制面交点：

- `xvalue.c` / `xvalue_print.c` / `xtype.c` 是否把 TLS isolate 当默认依赖
- `value` 层哪些 API 允许 `XrayIsolate *X == NULL`
- `xvalue_format.c` 是否已经从“值层工具”演化成横切服务
- `json shape` / `type pool` / `json_value_type` 这类 per-isolate cache 是否在 value 层泄漏了高层语义

---

## 10. 本轮状态

- **已完成**
  - 控制面文件与关键消费者的第一轮源码审计
  - `XrayIsolate` / `XrVMState` / `XrVMContext` 的职责分离判断
  - state ownership、TLS fallback、module/source/debug 三类热点识别

- **未完成**
  - 尚未进入 `value/` 的逐文件分析
  - 尚未核 `source_cache` 在 imported module 场景的完整性
  - 尚未做横向 include 统计与 consumers 分层计数

- **下一步**
  - 开始 `025-runtime-phase1-value.md`
