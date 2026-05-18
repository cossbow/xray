# Xray Task 基础设施升级方案（Phase 0）

> **定位**：本文档是 820_coroutine_association.md 的前置依赖。
> 先升级 Task 基础设施，再实现关联语法（linked go / monitored go / supervisor scope）。
>
> **设计哲学**：xray 协程默认独立（go = 无关联），只有显式声明（linked/monitored）
> 或在 scope 内才建立 parent-child 关系。这与 Kotlin（默认结构化）不同。
>
> **参考**：kotlinx.coroutines (Job.kt, JobSupport.kt, Deferred.kt)

## 一、现状与目标

### 1.1 已完成

| 已完成项 | 说明 |
|----------|------|
| XrTask 类型引入 | GC 管理的轻量句柄 (~64B)，缓存 result/error/state |
| go 返回 Task | vm_spawn_cont 分配 Task，base[a] = task |
| vm_await Task 路径 | fast path: state==COMPLETED 直接返回；slow path: CAS 协调 |
| executor 回收 | vm_task_consume_result deep copy + recycle |
| Task GETPROP | .done/.cancelled/.result/.error |
| Task INVOKE | vm_invoke_task_handle: cancel()/toString() |
| 500K×8 压力测试 | 400万次 go+await 无 OOM |

### 1.2 本文档要解决的问题

当前 XrTask 是纯结果缓存（~64B），不具备关联能力。820 文档需要以下基础设施：

| 820 需要的能力 | 当前缺失 | 本文档解决 |
|---------------|---------|-----------|
| linked go 错误传播 | 无 parent-child | Phase A: parent/first_child/next_sibling |
| cancel 级联 | 无 cancel_tree | Phase A: xr_task_cancel_tree |
| monitored go 通知 | 无 completion listener | Phase A: CompletionNode |
| supervisor scope | 无 supervisor 标志 | Phase A: flags + SUPERVISOR bit |
| linked scope 等子完成 | 无 Completing 状态 | Phase A: 6 状态机 |
| Task.monitor() API | waiter 一对一 | Phase B: 多 listener |
| Coro 瘦身（可选优化） | await 协调在 Coro 上 | Phase C: 迁移到 Task |

### 1.3 与 820 的关系

```
本文档 (Phase 0)                   820 (Phase 1-2)
================                   ================
Phase A: Task 状态机 + 层级结构
Phase B: await 迁移 + CompletionNode
Phase C: Coro 瘦身
                                   Phase 1: 前端语法 (linked/monitored/supervisor)
                                   Phase 2: 关联运行时 (link/monitor/scope 行为)
                                   Phase 3: cluster 统一 + JIT 适配
```

**Phase A 是 820 的硬依赖**。Phase B/C 可与 820 Phase 1 并行但建议先做。

### 1.4 与 Kotlin 的核心差异

| 维度 | Kotlin | xray |
|------|--------|------|
| 默认关联 | 所有 launch/async 自动 parent-child | go = 独立，无关联 |
| 显式关联 | 无（总是关联） | linked go / monitored go |
| scope | coroutineScope 默认子失败取消父 | scope 只等待；linked scope 才传播 |
| monitor | invokeOnCompletion 回调 | Channel-based（和 cluster 一致） |
| 语法 | launch { } / supervisorScope { } | linked go fn() / supervisor scope { } |

**parent-child 是可选的**（只在 linked/scope 场景使用），普通 go 的 Task 无 parent 开销。

## 二、XrTask 状态机升级

### 2.1 从 4 状态到 6 状态

**当前**（xtask.h）：PENDING(0) / COMPLETED(1) / FAILED(2) / CANCELLED(3)

**升级为**：
```c
typedef enum {
    XR_TASK_ACTIVE      = 0,  // executor running (renamed from PENDING)
    XR_TASK_COMPLETING  = 1,  // self done, waiting for children
    XR_TASK_CANCELLING  = 2,  // cancel requested, children still running
    XR_TASK_COMPLETED   = 3,  // final: success
    XR_TASK_FAILED      = 4,  // final: error
    XR_TASK_CANCELLED   = 5,  // final: cancelled
} XrTaskState;
```

**兼容性**：`XR_TASK_PENDING` → `XR_TASK_ACTIVE`，终态数值从 1/2/3 变为 3/4/5，
所有比较需更新。新增 `xr_task_is_active()` / `xr_task_is_done()` helper 替代裸比较。

### 2.2 XrTask 结构体扩展

```c
typedef struct XrTask {
    XrGCHeader gc;                          // 16B
    XrValue result;                         // 16B
    XrValue error;                          // 16B
    struct XrCoroutine *coro;               //  8B
    _Atomic uint8_t state;                  //  1B
    uint8_t flags;                          //  1B
    uint16_t child_count;                   //  2B
    uint32_t _pad;                          //  4B

    /* Parent-Child (only used with linked/scope) */
    struct XrTask *parent;                  //  8B
    struct XrTask *first_child;             //  8B
    struct XrTask *next_sibling;            //  8B

    /* Completion listeners */
    struct XrCompletionNode *on_completion; //  8B

    /* Await coordination (Phase B: from XrCoroutine) */
    _Atomic int await_state;                //  4B
    struct XrCoroutine *waiter;             //  8B
    int waiter_index;                       //  4B
} XrTask;
// ~112B total
```

**flags**: `XR_TASK_FLG_SUPERVISOR(1)`, `XR_TASK_FLG_SCOPE_TASK(2)`, `XR_TASK_FLG_HAS_PARENT(4)`

**零开销**：普通 `go fn()` 时 parent/first_child/on_completion 都是 NULL，无运行时开销。

### 2.3 CompletionNode

```c
typedef struct XrCompletionNode {
    struct XrCompletionNode *next;
    uint8_t type;  // WAKE / CHANNEL / CLOSURE / AWAIT_ALL / AWAIT_ANY
    union {
        struct XrCoroutine *waiter;        // WAKE
        struct XrChannel *channel;         // CHANNEL (for monitor)
        XrValue closure;                   // CLOSURE (onComplete)
        struct { void *ctx; int index; } await_all;
        void *await_any;
    } as;
} XrCompletionNode;
```

`XR_COMPLETION_CHANNEL` 为 820 的 `task.monitor()` 准备（向 Channel 发送事件）。

## 三、核心操作

### 3.1 Parent-Child

```c
void xr_task_attach_child(XrTask *parent, XrTask *child);  // 建立关联
void xr_task_detach_child(XrTask *parent, XrTask *child);  // 解除关联
```

单写者假设（spawn 在父上下文中）。child_completed 从不同 worker 调用需 CAS 保护。

### 3.2 完成流程

- `xr_task_try_complete(task, result)` — 有子 → COMPLETING，无子 → finalize
- `xr_task_child_completed(parent, child)` — detach + 检查是否最后一个
- `xr_task_finalize(task, state)` — 设终态 + 通知 parent + fire listeners

### 3.3 Cancel 级联

`xr_task_cancel_tree(task)` — CAS → CANCELLING，cancel executor，递归 cancel 子。

### 3.4 Supervisor

`XR_TASK_FLG_SUPERVISOR` — 子失败时 `xr_task_fail_with_propagation` 跳过向上传播。

## 四、Await 协调迁移（Phase B）

### 4.1 核心变化

CAS `await_state`、`waiter` 从 XrCoroutine 迁移到 XrTask：

| 迁移前（Coro 上） | 迁移后（Task 上） |
|-------------------|-------------------|
| `coro->await_state` | `task->await_state` |
| `coro->waiter` | `task->waiter` |
| `coro->waiter_index` | `task->waiter_index` |
| `coro->wait_count` | CompletionNode chain |
| `coro->any_done` | CompletionNode chain |
| `coro->await_results` | AwaitAllCtx |
| `xr_coro_wake_waiter()` | `xr_task_fire_completion()` |

### 4.2 vm_await 改动要点

- fast path: `xr_task_is_done(task)` → 读 `task->result`
- slow path: CAS 在 `task->await_state`（不再在 `coro->await_state`）
- waiter 注册在 `task->waiter`
- 完成通知: `xr_task_fire_completion()` 替代 `xr_coro_wake_waiter()`

### 4.3 await_all / await_any

旧设计在 caller coro 上维护 wait_count/any_done/await_results。
新设计通过 CompletionNode 链表：每个被等待的 task 注册一个 AWAIT_ALL/AWAIT_ANY node，
task 完成时触发 node 回调，递减 remaining 计数或标记 first result。

## 五、实施计划（3 个 Phase）

### Phase A：Task 状态机 + Parent-Child + CompletionNode

**目标**：XrTask 从纯结果缓存升级为可关联的 Job 对象

| 文件 | 改动 |
|------|------|
| `coro/xtask.h` | 6 状态机 + flags + parent/child/sibling + on_completion + CompletionNode |
| `coro/xtask.c` | attach/detach_child, cancel_tree, try_complete, finalize, child_completed, fire_completion |
| `runtime/gc/xcoro_gc_traverse.c` | mark parent/children/siblings/on_completion |

**验证**：回归测试 + parent-child 单元测试 + 状态转换测试

**风险**：低 — 纯新增，不改动旧路径

### Phase B：Await 协调迁移 (Coro → Task)

**目标**：CAS/waiter 迁移到 Task，await_all/any 用 CompletionNode 重写

| 文件 | 改动 |
|------|------|
| `coro/xtask.h` | 新增 await_state, waiter, waiter_index |
| `vm/xvm_cold_paths.c` | vm_await/await_all/any/timeout: 操作 task 而非 coro |
| `coro/xcoro.c` | 删除 xr_coro_wake_waiter |
| `coro/xworker.c` | worker_handle_vm_result: 调用 xr_task_try_complete |
| `jit/xir_jit.c` | xr_jit_await_block: CAS 在 task->await_state |

**验证**：全量回归 + 500K×8 压力 + ASan

**风险**：高 — 核心路径。策略：先新增 Task 路径，旧路径 fallback，测试通过后删旧

### Phase C：Coro 瘦身

**目标**：删除 Coro 中已迁移字段，节省 ~80B/coro

| 字段 | 大小 | 去向 |
|------|------|------|
| result/error | 32B | task->result/error |
| await_state/waiter/waiter_index | 16B | task 上 |
| wait_count/any_done/await_results | 16B | CompletionNode |
| parent_scope/await_target | 16B | task->parent / 删除 |

| 文件 | 改动 |
|------|------|
| `coro/xcoroutine.h` | 删除上述字段 |
| `coro/xcoro.c` | 更新 init/recycle/reset |
| `coro/xworker.c` | worker_blocked_post_check: 从 task 读 |
| `vm/xvm.c` | OP_RETURN: result 写 task |

**验证**：sizeof 验证 + 全量回归 + 压力测试

**风险**：中 — 大量引用点更新，但逻辑已在 Phase B 验证

## 六、删除清单（Phase C）

| 删除项 | 位置 | 替代 |
|--------|------|------|
| `coro->result/error` | xcoroutine.h | task->result/error |
| `coro->await_state/waiter/waiter_index` | xcoroutine.h | task 上 |
| `coro->wait_count/any_done/await_results` | xcoroutine.h | CompletionNode |
| `coro->parent_scope/await_target` | xcoroutine.h | task->parent / 删除 |
| `xr_coro_wake_waiter()` | xcoro.c | xr_task_fire_completion |
| `waiter_index` 负值约定 | xcoro.c | CompletionNode type 字段 |
| `vm_await_recycle_coro()` | xvm_cold_paths.c | xr_task_finalize 统一回收 |

**净效果**：
```
XrCoroutine: ~640B → ~560B  (节省 ~80B/coro)
XrTask:       ~64B → ~112B  (增加 ~48B/task)
净: 每 go 调用节省 ~32B + 删除 waiter_index 魔数 + 统一完成通知
```

## 七、Kotlin 概念映射

| Kotlin | xray | 来源 |
|--------|------|------|
| `Job` | `XrTask` | 本文档 Phase A |
| `Job._state` (6 状态) | `task->state` (6 状态) | 本文档 Phase A |
| `Job.parent/children` | `task->parent/first_child` | 本文档 Phase A |
| `SupervisorJob` | `XR_TASK_FLG_SUPERVISOR` | 本文档 Phase A |
| `Job.cancel()` / 父取消子 | `xr_task_cancel_tree` | 本文档 Phase A |
| `Deferred.await()` | `await task` (Task CAS) | 本文档 Phase B |
| `awaitAll` | `await.all` (CompletionNode) | 本文档 Phase B |
| `invokeOnCompletion` | `task.onComplete(fn)` | 本文档 Phase A |
| `coroutineScope {}` | `linked scope {}` | 820 Phase 1-2 |
| `supervisorScope {}` | `supervisor scope {}` | 820 Phase 1-2 |
| 子失败 → 取消父 | `linked go` 错误传播 | 820 Phase 2 |
| `launch/async` | `go fn()` / `linked go fn()` | 820 Phase 1 |

**不实现**：CoroutineContext / Dispatchers（固定 work-stealing）/ Flow

## 八、风险与缓解

| 风险 | 等级 | 缓解 |
|------|------|------|
| Phase B 核心路径改动导致死锁 | P0 | 双路径过渡 + 500K 压力 + ASan |
| parent-child GC 遍历遗漏 | P1 | xcoro_gc_traverse 全覆盖 + ASan |
| CompletionNode 内存泄漏 | P1 | fire_completion 后释放 + GC 扫描 |
| 状态枚举值变化导致旧代码误判 | P1 | 全局搜索替换 + 编译器 -Wswitch 检查 |

## 九、执行顺序与里程碑

```
Phase A (状态机+层级+CompletionNode)
    │  回归 + parent-child 测试
    ▼
Phase B (await 迁移)
    │  回归 + 500K + ASan
    ▼
Phase C (coro 瘦身)
    │  sizeof 验证 + 回归 + 500K
    ▼
─── 本文档完成，进入 820 ───
    ▼
820 Phase 1 (前端语法: linked/monitored/supervisor)
    ▼
820 Phase 2 (关联运行时: link/monitor/scope 行为)
    ▼
820 Phase 3 (cluster 统一 + JIT 适配)
```

**每个 Phase 结束时**：
1. 全量回归测试通过
2. 500K×8 压力测试无 OOM
3. Git commit 备份

