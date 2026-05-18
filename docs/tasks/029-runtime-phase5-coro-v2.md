# Runtime `src/coro/` 层分析 V2（`029`）

> 旧版 `029-runtime-phase5-coro.md` 的对账重写。严格对齐当前源码，按"无兼容性"原则给最佳设计。
>
> **范围**：`src/coro/` 全部 57 文件 + 必要消费者（`runtime/object/xexception.c` 已在 027 V2 处理；`vm/xvm_dispatch_coro.inc.c`）。

## 1. 七条核心结论

- **`src/coro/` 不是单一调度模块**，而是 runtime 中负责跨 owner 迁移的控制层。V1 准确。

- **`XrCoroutine` 三层分配模式**（`xcoro.c:130-138, 409-428, 490-509, 555-574`）：
  ```
  pool_get(runtime) → sysheap_alloc_coro → xr_malloc fallback
  ```
  4 处分配点逻辑相同。executor 是 sysheap/pool/malloc owned shell，内部 `coro->coro_gc` 是私有 Immix 堆。V1 准确。

- **`XrTask` 走 per-coro Immix**（`xtask.c:61` `xr_alloc(executor, sizeof(XrTask), XR_TTASK)`）。task 是用户可见的 await/monitor handle，executor 是可 recycle 的执行体。这层分离是设计意图。V1 准确。

- **`XrChannel` 永远是 shared system heap object**（`xchannel.c:156, 224`）：单次 `xr_sysheap_alloc_shared(sizeof(XrChannel) + buffer_size * sizeof(XrValue), XR_TCHANNEL)` + `xr_shared_set_refc(1)`。inline buffer 紧跟 header。V1 准确。

- **deep-copy 是 cross-coro 真实数据边界**：
  - `XR_GC_IS_SHARED(obj)` → `xr_shared_incref` 直接返回（`xdeep_copy.c:394, 449, 683`）。
  - `dst_coro_gc` 存在 → `xr_coro_gc_newobj`；否则 → fixedgc fallback（`xdeep_copy.c:71-74`）。
  - `xdeep_copy.c:680-700` 还有 `deep_copy_to_shared` 路径，目标域是 sysheap shared。
  - 三类目标域都覆盖。V1 准确。

- **`XrInstance` 在 deep-copy 路径下落 dst_coro Immix，但 normal allocation 落 fixedgc**（与 028 V2 §3.2 协同发现）。`xdeep_copy_instance_with_ctx`（`xdeep_copy.c:314`）通过 `copy_ctx_alloc` 走 dst_coro_gc，**与 `xinstance.c:37` 的 fixedgc 分配路径 owner 不一致**。这是真正的设计漂移。

- **structured concurrency 双链表（task tree + scope child list）真实存在**（V1 准确）：
  - task: `parent->first_child` + `_Atomic bool child_lock` 自旋锁
  - scope: `XrScopeContext.first_child` + `coro->scope_sibling` + `child_lock`
  - 同一个 child completion 事件需要在两套链表里各做一次同步移除。

## 2. 模块地图（修正版）

实际 57 个文件，按职责分 12 类：

| 子域 | 文件 |
|---|---|
| **coroutine core** | `xcoro.c`、`xcoroutine.h`、`xcoro_flags.h`、`xcoro_pool.h/c`、`xcoro_registry.h/c`、`xcoro_tuning.h`、`xcoro_debug.h/c`、`xcoro_monitor.c` |
| **machine 抽象** | `xmachine.h/c` |
| **proc** | `xproc.h/c` |
| **worker core** | `xworker.h/c`、`xworker_internal.h` |
| **worker 子模块** | `xworker_blocked.c`、`xworker_exec.c`、`xworker_handoff.c`、`xworker_pool.c`、`xworker_runq.c`、`xworker_sched.c`、`xworker_sysmon.c` |
| **task** | `xtask.h/c` |
| **channel** | `xchannel.h/c`、`xchan_wake_cmd.c` |
| **deep copy** | `xdeep_copy.h/c` |
| **timer** | `xtimer_wheel.h/c` |
| **netpoll** | `xnetpoll.h/c`、`xnetpoll_epoll.c`、`xnetpoll_iocp.c`、`xnetpoll_iouring.c`、`xnetpoll_kqueue.c`、`xnetpoll_select.c` |
| **socket** | `xsocket.h/c` |
| **杂项** | `xasync.h/c`、`xbalance.h/c`、`xresume.h/c`、`xsteal_queue.h/c`、`xmpsc_queue.h/c`、`xyieldable.h/c`、`xjit_hooks.h/c`、`xsched_trace.h` |

V1 仅列了 6 大类（coroutine/runtime-worker/task/channel/deep-copy/scope），漏掉：proc/machine 抽象、worker 7 个子文件、netpoll 5 个 backend、socket、balance、steal_queue、async、yieldable、coro_pool、coro_registry、coro_monitor、coro_debug、jit_hooks、resume、mpsc_queue、sched_trace、timer_wheel。**实际 coro/ 是 V1 描述规模的 ~10 倍**。

## 3. 关键论断核对

### 3.1 `XrCoroutine` 分配三层

V1 准确。源码 4 处分配点（`xr_coro_create_bootstrap`、`xr_coro_create`、`xr_coro_create_with_args`、`xr_task_run_*`）全部使用相同三层 fallback 协议。

值得注意（V1 漏）：

- pool 路径走 worker-local `xr_current_worker()` LIFO 优先（`xworker_pool.c:30-40`），命中率高。
- pool miss 时才走 sysheap arena。
- `XR_TCOROUTINE` 的 GC type 标记用于 deep-copy 判断（不复制）+ debug 显示。

### 3.2 `XrTask` per-coro Immix

`xtask.c:55-75` 的注释明确写出设计契约：

> task is allocated on the executor's Immix heap (GC-marked in `mark_coro_roots`). The OP_AWAIT paths must NOT recycle the executor while parent still references the task — parent's GC would scan freed Immix memory (use-after-free).

这是**显式的 lifetime invariant**：

- task 在 executor 的 Immix 上。
- parent coroutine 通过 `coro->task` 持有 task PTR。
- executor recycle 必须确认无任何 parent task tree 引用。

V2 加正向：这套契约已经在源码注释里清楚表达，是良好设计。

### 3.3 `XrChannel` 单次分配 + 单一 owner

`xchannel.c:153-160` 和 `:221-228` 两个构造函数都用同一模式：

```c
size_t alloc_size = sizeof(XrChannel) + buffer_size * sizeof(XrValue);
XrChannel *ch = xr_sysheap_alloc_shared(..., alloc_size, XR_TCHANNEL);
xr_shared_set_refc(&ch->gc_header, 1);
```

V2 加正向：channel 是 V1 已经识别 + V2 强化的"shared object 模型干净案例"——分配/refcount/owner 全单一。

### 3.4 deep-copy 三目标域

V1 已识别，V2 精确化路径：

| 路径 | 入口 | 目标域 |
|---|---|---|
| Per-coro deep copy | `xr_deep_copy_to_coro(value, dst_coro)` | `dst_coro->coro_gc` Immix |
| Bootstrap fallback | 同上，`dst_coro_gc == NULL` | isolate fixedgc |
| Shared deep copy | `xr_deep_copy_to_shared(X, value)` | sysheap shared + refc=1 |
| Shared passthrough | `XR_GC_IS_SHARED(obj)` | incref，no copy |

`copy_ctx_alloc` 在 `xdeep_copy.c:71-74` 实现统一 alloc：

```c
static inline void* copy_ctx_alloc(XrCopyContext *ctx, size_t size, uint8_t type) {
    if (ctx->dst_coro_gc) {
        return xr_coro_gc_newobj(ctx->dst_coro_gc, type, size);
    }
    // fallback: isolate fixedgc
    ...
}
```

deep-copy 覆盖 7 个 GC 类型（`xdeep_copy.c:399-410`）：Array、Map、Closure、Set、Instance、Json、DateTime。**Task / Channel / Coroutine / Cell / BoundMethod / String / BigInt / Range 不在 deep-copy switch 里**（依赖 shared / non-deep / 不可跨）。

### 3.5 `XrInstance` 跨域漂移（与 028 V2 §3.2 同源问题）

deep-copy 路径（`xdeep_copy.c:314`）通过 `copy_ctx_alloc(ctx, ..., XR_TINSTANCE)` 走 `dst_coro_gc`。

normal 路径（`xinstance.c:37, 96`）走 `xr_gc_alloc(&isolate->gc, ..., XR_TINSTANCE)` = fixedgc。

shared 路径（`xdeep_copy.c:586`）走 `xr_sysheap_alloc_shared(..., XR_TINSTANCE)`。

**同一个 `XR_TINSTANCE` 类型在三种路径下落在三个 owner**。这是 028 V2 推荐"normal instance 走 per-coro"修复的关键证据。如果 028 V2 修复后，三路径就退化成"per-coro 默认 + shared 显式"两路径，与其他 deep-copy-able 对象一致。

### 3.6 task tree + scope child list 双链表（P1）

V1 准确。两套链表：

- `XrTask.first_child` + `XrTask.parent` + `XrTask._Atomic bool child_lock` (`xtask.h`)
- `XrScopeContext.first_child` + `XrCoroutine.scope_sibling` + `XrScopeContext.child_lock` (`xcoroutine.h`)

`xr_coro_wake_waiter`（`xcoro.c`）在协程完成时同时操作两套链表。这是真实的 owner 复杂度。

V2 修复方向：

- **方案 A**：合并两套链表，task tree 是 lifecycle owner，scope 只是 view（迭代时按 task tree 过滤 scope 标记）。
- **方案 B**：保留两套，但写清楚 happens-before 契约：scope sibling 移除 → task child detach → fire completion。

V2 推荐 A：scope 只在 linked/supervisor 模式有用，可作为 task 的 metadata bit。

### 3.7 `xr_coro_wake_waiter` 责任过重（V1 准确）

V1 已识别。在 `xcoro.c` 中此函数同时处理：

- scope counter decrement
- linked/supervisor error policy
- scope child list removal
- sibling cancel (linked mode)
- task wake_waiter delegate

V2 修复：拆 4 个独立函数：`scope_dec_count`、`scope_apply_error_policy`、`scope_detach_sibling`、`scope_cancel_siblings`。`xr_coro_wake_waiter` 只做 orchestration。

### 3.8 channel wake routing 严格 worker-private（V1 准确）

`xworker.h` 暴露 worker-side wake API：

- `xr_worker_inbox_enqueue(worker, ...)`：cross-worker 远端唤醒入口
- `xr_worker_dispatch_chan_wake(worker, ch, ...)`：本 worker 调度 channel wake
- `xr_worker_drain_chan_wake_queue(worker)`：本 worker 处理积压唤醒

cross-worker wake 必须走 inbox / chan wake queue，不允许直接操作对方 worker 的 blocked queue。V2 加正向：这套契约清晰，是好设计。

V1 提到 `xchannel.c` 的 `channel_wake_coro_ex()` 已经安全（local LIFO 或 inbox），仅 `xcoro.c` 的 `xr_runtime_wake_channel*` 系列需要 audit。这是 026 audit plan 已经识别的 CORO-01。

### 3.9 worker pool L2 全局共享（与 026 V2 §6.4 协同）

V2 §6.4 已点出 `xcoro_gc.c:50-52` 的 `g_gc_pool_*`。同样的模式在 `xworker_pool.c` 是否存在？

源码 `xcoro_pool.c` 是 per-runtime pool，不是 process-wide。**worker / coroutine 的进程级 mutable global 仅 GC pool 一处**（验证）。V2 加正向：worker / coro pool 设计干净。

## 4. 状态归属（修正版）

| 对象 | 真实所有者 | 释放路径 |
|---|---|---|
| `XrRuntime` | isolate（calloc graph） | `xr_runtime_destroy` |
| `XrMachine[]` / `XrWorker[]` | isolate（calloc array） | `xr_runtime_destroy` |
| `XrCoroutine` | runtime pool / sysheap arena / malloc | recycle to pool 或 sysheap free |
| `XrCoroGC` | per-coroutine（lazy） | `xr_coro_gc_destroy` 或 reset to pool |
| `XrTask` | per-executor Immix | Immix sweep（要求 parent 释放 ref） |
| `XrChannel` | sysheap shared + refc | `xr_shared_decref` → sysheap free |
| `XrScopeContext` | parent coroutine（栈或 heap） | parent 完成时 destroy |
| timer wheel | per-worker（malloc） | `xr_worker_destroy` |
| netpoll backend | per-worker（malloc + fd） | `xr_worker_destroy` |
| async pool | runtime（calloc） | `xr_runtime_destroy` |
| MPSC inbox / continuation deque / blocked queue | per-worker（malloc） | `xr_worker_destroy` |
| `XrSocket` | per-coro Immix | Immix sweep |

## 5. 高风险点（精确版）

### 5.1 `XrInstance` 三域漂移（与 028 V2 协同 P0）

§3.5 已展开。修复路径同 028 V2 §5.1：normal `xr_instance_new` 改走 `xr_alloc(coro, ...)`。

### 5.2 task tree + scope child list 双链表同步（P1）

§3.6 已展开。

### 5.3 `xr_coro_wake_waiter` 责任过重（P1）

§3.7 已展开。

### 5.4 `xr_runtime_wake_channel*` cross-worker 直接操作（P0，已在 audit plan）

V1 提及，与 `coro_audit_plan.md` CORO-01 协同。这是已识别的 known issue，不在 V2 重复细化。

### 5.5 `XrTask.on_completion` listener 链表无独立同步原语（V1 准确，P2）

`xr_task_add_completion`（`xtask.c`）做简单头插，`xr_task_fire_completion` 取整链。当前测试场景下 listener 注册都在 single-thread 上下文，但缺乏文档化的 happens-before 契约。

V2 修复：要么加 `task->listener_lock`（与 `child_lock` 同模式），要么文档化"listener 注册必须发生在 task 创建之后、await 之前"。

### 5.6 deep-copy 不覆盖某些 GC 对象（V1 准确）

`Task` / `Cell` / `BoundMethod` / `Coroutine` / `Channel` / `String` / `BigInt` / `Range` 不在 deep-copy switch。

| 类型 | 跨 coro 行为 |
|---|---|
| `Task` | 不跨（不应跨） |
| `Cell` | 不跨（per-closure） |
| `BoundMethod` | 不跨（fixedgc） |
| `Coroutine` | 不跨（owner shell） |
| `Channel` | 跨（shared incref） |
| `String` | 跨（intern / shared） |
| `BigInt` | 跨? 当前无明确路径 |
| `Range` | 跨? 当前无明确路径 |

V2 修复：明确 `BigInt` / `Range` 的 cross-coro 协议（添加 deep-copy 或标 shared-only）。

### 5.7 deep-copy `dst_coro_gc == NULL` 退化到 fixedgc（P2）

`xdeep_copy.c:71-74` 在没有 `dst_coro_gc` 时静默退化到 fixedgc。这违反"deep-copy 总到目标 coro"的语义直觉。

V2 修复：要么 fail-fast（`dst_coro_gc` 必须存在），要么文档化"bootstrap fallback only"。

## 6. 正向资产

V1 + V2 共同识别：

- **coroutine / task 分层**（executor 可 recycle，task 是 stable handle）
- **`xr_coro_ready` 单一 wake CAS 协议**
- **channel shared 模型干净**
- **deep-copy 显式编码三目标域**
- **worker-private 状态严格隔离**（cross-worker 必经 inbox / chan wake queue）
- **`XrTask` 注释级 lifetime invariant 文档化**（V2 强化）
- **`xworker.h` API 边界清晰**（worker-side / cross-worker 各有显式入口）

## 7. 最佳设计建议（无兼容性）

10 条直接可执行：

1. **`XrInstance` normal 走 per-coro Immix**（§5.1，与 028 V2 §5.1 P0）
2. **合并 task tree + scope child list**（§3.6 方案 A，P1）
3. **拆 `xr_coro_wake_waiter`**（§3.7，P1）
4. **`xr_runtime_wake_channel*` 走 inbox**（§5.4，已在 audit plan，P0）
5. **`XrTask.on_completion` listener 同步契约文档化或加 lock**（§5.5，P2）
6. **deep-copy 明确所有 GC 类型的跨 coro 行为**（§5.6，P2）
7. **deep-copy `dst_coro_gc == NULL` 决策**（§5.7，P2）
8. **完整 owner 矩阵在 `coroutine.h` 头注释里固化**（让后续维护者一眼看清）
9. **`XrCoroutine` 与 `XrTask` 交互 lifetime contract 在 `xtask.h` 头注释里写清**（已部分在 `xtask.c:55-75`，挪到 .h 更显眼）
10. **timer wheel 释放路径加 `XR_DCHECK`**（与 audit plan CORO-07 协同）

## 8. 给 030 横切复盘的输入

V1 给的方向都对，V2 补充：

- **`XrInstance` 的三路径 owner 漂移**是 027/028/029 共同发现的 P0，030 V2 必须作为代表案例。
- **伪 GC 对象**模式贯穿 027（exception/reflection wrapper）、028（class/instance/wrapper）、029（无新增），在 030 必须独立成章。
- **三种 deep-copy 目标域** vs **四种 owner 模型**（per-coro / fixedgc / sysheap shared / malloc graph）的映射矩阵需要在 030 显式列出。
- **worker / coro 分配池** vs **GC struct pool**（`g_gc_pool_*`）的"进程级 mutable global"是否还有第三处，030 需要全仓 grep 验证。
- **structured concurrency** 的 task tree vs scope child list 是否合并、单一 lifecycle 来源问题。

## 9. 与 V1 的主要差异

| 点 | V1 | V2 |
|---|---|---|
| 文件清单 | 6 大类 | 57 个文件 12 类 |
| coroutine 三层分配 | 准确 | 加 4 处分配点引用 |
| deep-copy 三目标域 | 提到 | 加路径表 + 入口函数 |
| `XrInstance` 漂移 | "deep-copy 与 fixedgc 张力" | 三路径精确：fixedgc / dst_coro_gc / sysheap shared |
| deep-copy 覆盖类型 | 列 7 个 | 加 7 个不覆盖的（含跨 coro 协议建议） |
| task lifetime invariant | "task 通过 coro->task mark root" | 加 `xtask.c:55-75` 注释引文 |
| worker pool global | 未识别 | 验证仅 `g_gc_pool_*` 一处全局 |
| 改进建议 | 5 条方向 | 10 条可执行 |

## 10. 本轮状态

- **已完成**：coro/ 57 文件全核对、V1 论断逐项判断、最佳设计 10 条
- **未完成**：netpoll 各 backend 详查（留给后续 audit）；timer wheel destroy 路径详查（与 CORO-07 协同）；socket / async / balance / yieldable 子系统的 owner 模型
- **下一步**：`030-runtime-cross-cutting-recap-v2.md`
