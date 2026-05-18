# Xray 协程系统最佳重构方案

> **原则**: 不考虑向后兼容性，直接采用最佳设计，一步到位。

## 一、当前问题诊断

### 1.1 核心矛盾

```
XrCoroutine 结构体 (~640B) + VM栈 (1KB) + Frames (640B) ≈ 2.3KB/协程
   ↓ 从 Pool 分配（不在 GC 堆上）
   ↓ GC 无法自动回收
   ↓ 完成后只释放 Immix heap，不回收结构体
   ↓ 500K × 8 轮 = 400万个结构体泄漏 → OOM
```

**根本矛盾**: `go` 返回 `XrCoroutine*` 作为 XrValue，用户持有引用可以 re-await。但 `XrCoroutine*` 又是 pool 分配的重型对象，必须及时回收才不会泄漏。**回收 vs 引用存活** 不可调和。

### 1.2 问题链

| 问题 | 影响 | 根因 |
| ---- | ---- | ---- |
| 内存泄漏 (500K OOM) | P0 | Pool 分配脱离 GC，await 后不回收 struct |
| recycle_local 导致 re-await 挂起 | P0 | 回收清空 result + DONE 标志 |
| release_resources 导致 SIGSEGV | P0 | 回收后 struct 被重用，旧引用 UAF |
| XrCoroutine 过大 (~640B) | P1 | 冷热数据混在一起，缓存不友好 |
| await 状态机复杂 (CAS + 多分支) | P2 | 直接操作协程内部状态 |

### 1.3 设计缺陷本质

**Kotlin 的启示**: `Deferred` 对象和协程执行体是分离的。`Deferred` 只缓存结果，GC 管理生命周期。协程执行体（Continuation 链）在完成后被 GC 回收。两者独立。

**xray 当前**: `XrCoroutine` = 用户句柄 + 执行体 + 结果 + 调度元数据，全部耦合在一个 pool 分配的巨型结构体中。

## 二、最佳设计方案：Task/Executor 分离

### 2.1 核心理念

将 `XrCoroutine` 拆分为两个独立对象：

```
XrTask (~48B, GC 堆分配)          XrExecutor (~600B, Pool 分配)
┌─────────────────────┐           ┌──────────────────────────┐
│ XrGCHeader gc       │           │ vm_ctx (stack/frames)    │
│ XrValue result      │  ──coro─► │ coro_gc (Immix heap)     │
│ XrValue error       │           │ flags, reductions        │
│ XrExecutor *coro    │           │ sched_link, next, prev   │
│ _Atomic uint8_t     │           │ entry (closure/cfunc)    │
│   state             │           │ args                     │
│ _Atomic(XrTask*)    │           │ waiter, await_state      │
│   waiter_task       │           │ channel blocking fields  │
│ int waiter_index    │           │ jit suspend/resume       │
└─────────────────────┘           │ ...                      │
  用户持有，GC 管理                └──────────────────────────┘
  多次 await 安全                   调度器持有，Pool 管理
  完成后 coro→NULL                  完成后立即回收到 Pool
```

### 2.2 生命周期

```
go fn(args):
  1. Pool 分配 XrExecutor (含 stack/frames)
  2. GC 堆分配 XrTask (48B)
  3. XrTask.coro = executor
  4. XrExecutor.task = task (弱引用)
  5. base[a] = xr_value_from_task(task)  // 用户拿到 Task
  6. 调度器拿到 executor，加入 run queue

executor 完成:
  1. task->result = executor->vm_ctx.stack[0]  (deep copy)
  2. task->state = COMPLETED
  3. task->coro = NULL                         // 断开引用
  4. xr_coro_recycle_local(worker, executor)    // 立即回收！
  5. wake waiter

await task:
  if (task->state == COMPLETED)
      return task->result       // 直接返回，无论第几次 await
  else
      挂起等待 task->state 变为 COMPLETED

GC 阶段:
  - Task 是普通 GC 对象，无引用时自动回收
  - Executor 不在 GC 堆上，通过 Pool 管理
  - Task.result 是 GC 堆上的值，正常 mark
```

### 2.3 XrTask 结构体定义

```c
/* GC-managed lightweight handle returned by `go` expression.
 * Survives independently of the executor — safe to re-await.
 * Size: ~48 bytes (fits in one cache line). */
typedef struct XrTask {
    XrGCHeader gc;                  // 16B: GC header (type = XR_TTASK)
    XrValue result;                 // 16B: cached result (deep-copied on completion)
    XrValue error;                  // 16B: error value (if failed)
    struct XrExecutor *coro;        //  8B: executor (NULL after completion)
    _Atomic uint8_t state;          //  1B: PENDING / COMPLETED / FAILED / CANCELLED
    uint8_t _pad[7];               //  7B: alignment padding

    /* Await coordination (moved from XrCoroutine) */
    _Atomic int await_state;        //  4B: NONE / WAITING / RESOLVED
    struct XrTask *waiter_task;     //  8B: parent task waiting on this
    int waiter_index;               //  4B: index in await_all array
    _Atomic int wait_count;         //  4B: for await_all/await_any
    _Atomic bool any_done;          //  1B: for await_any
    struct XrArray *await_results;  //  8B: for await_all results
    struct XrScopeContext *parent_scope; // 8B: structured concurrency
} XrTask;
```

### 2.4 XrCoroutine 瘦身（重命名为 XrExecutor 可选）

从 `XrCoroutine` 中移除所有与「用户句柄」和「await 协调」相关的字段：

**删除的字段**（移到 XrTask）：
- `result`, `error` → Task.result, Task.error
- `await_state` → Task.await_state
- `waiter`, `waiter_index` → Task.waiter_task, Task.waiter_index
- `wait_count`, `any_done` → Task.wait_count, Task.any_done
- `await_results` → Task.await_results
- `parent_scope` → Task.parent_scope

**新增字段**：
- `XrTask *task` — 指向关联的 Task（完成后写入 result 用）

**保留的字段**（执行体核心）：
- `gc`, `flags`, `reductions` — 调度
- `vm_ctx` — VM 执行上下文
- `sched_link`, `next`, `prev` — 队列链接
- `coro_gc` — GC 堆
- `entry`, `args` — 入口点
- `channel blocking fields` — channel 等待
- `jit suspend/resume` — JIT 挂起恢复
- `name`, `source_file`, `source_line` — 调试信息

### 2.5 await 实现（极简化）

```c
int vm_await(XrayIsolate *isolate, XrVMContext *vm_ctx,
             XrInstruction instr, XrValue *base,
             XrBcCallFrame *frame, XrInstruction *pc) {
    int a = GETARG_A(instr);
    int b = GETARG_B(instr);
    int discard_result = GETARG_C(instr);

    XrValue task_val = base[b];
    if (!xr_value_is_task(task_val)) {
        VM_COLD_THROW(frame, pc, XR_ERR_TYPE_MISMATCH, "await: expected task");
    }

    XrTask *task = xr_value_to_task(task_val);

    /* Fast path: task already completed — works for re-await too! */
    uint8_t state = atomic_load_explicit(&task->state, memory_order_acquire);
    if (state == XR_TASK_COMPLETED) {
        base[a] = discard_result ? xr_null() : task->result;
        return VM_COLD_BREAK;
    }
    if (state == XR_TASK_FAILED) {
        /* propagate error */
        base[a] = xr_null();
        return VM_COLD_BREAK;
    }
    if (state == XR_TASK_CANCELLED) {
        base[a] = xr_null();
        return VM_COLD_BREAK;
    }

    /* Slow path: suspend and wait (CAS on task->await_state) */
    XrCoroutine *current_coro = vm_cold_get_coro(vm_ctx);
    XrTask *current_task = current_coro ? current_coro->task : NULL;
    if (current_task) {
        int expected = XR_AWAIT_NONE;
        if (atomic_compare_exchange_strong_explicit(&task->await_state,
                &expected, XR_AWAIT_WAITING,
                memory_order_acq_rel, memory_order_acquire)) {
            task->waiter_task = current_task;
            frame->pc = pc - 1;
            return VM_COLD_BLOCKED;
        }
        /* CAS failed → must be RESOLVED */
        base[a] = discard_result ? xr_null() : task->result;
        return VM_COLD_BREAK;
    }

    /* Main thread: spin wait (same as before) */
    // ...
}
```

**关键简化**：
1. **re-await 零成本**: `task->state == COMPLETED` 直接返回缓存的 result
2. **无 UAF 风险**: Task 是 GC 对象，result 也在 GC 堆上
3. **Executor 已经回收**: 不影响 Task 的 result

### 2.6 Executor 完成时的处理

```c
// worker_handle_vm_result 中 XR_VM_OK 分支:
static void executor_complete(XrWorker *worker, XrCoroutine *coro, XrVMResult result) {
    XrTask *task = coro->task;
    if (task) {
        /* Deep copy result from executor heap to parent's/task's GC heap.
         * Task.result lives on GC heap, accessible after executor recycled. */
        XrayIsolate *isolate = worker->p.runtime->isolate;
        if (!XR_IS_PTR(coro->vm_ctx.stack[0])) {
            task->result = coro->vm_ctx.stack[0];  // primitives: no copy
        } else {
            /* Result copied to isolate's main GC heap (shared, long-lived).
             * Alternative: copy to waiter's coro_gc if known. */
            task->result = xr_deep_copy_value(isolate, coro->vm_ctx.stack[0]);
        }

        /* Set state BEFORE wake (release ordering ensures result visible) */
        atomic_store_explicit(&task->state, XR_TASK_COMPLETED, memory_order_release);

        /* Disconnect: executor no longer referenced by task */
        task->coro = NULL;

        /* Wake waiter */
        xr_task_wake_waiter(isolate, task);

        /* Clear back-pointer so recycle doesn't touch stale task */
        coro->task = NULL;
    }

    /* Immediately recycle executor back to pool — safe! */
    xr_coro_recycle_local(worker, coro);
}
```

### 2.7 `go` 表达式的改动

```c
// vm_spawn_cont 中:
XrCoroutine *coro = xr_coro_create(isolate, closure, args, c, ...);

/* Allocate task on GC heap (~48B) */
XrTask *task = (XrTask *)xr_gc_alloc(current_coro_gc, sizeof(XrTask));
task->gc.type = XR_TTASK;
task->result = xr_null();
task->error = xr_null();
task->coro = coro;
atomic_store_explicit(&task->state, XR_TASK_PENDING, memory_order_relaxed);
atomic_store_explicit(&task->await_state, XR_AWAIT_NONE, memory_order_relaxed);
task->waiter_task = NULL;
task->waiter_index = -1;
task->parent_scope = scope;

coro->task = task;  // back-pointer

base[a] = xr_value_from_task(task);  // 用户拿到的是 Task，不是 Coro
```

## 三、实施计划

### Phase 1: 引入 XrTask 类型（基础设施）

**修改的文件**：

| 文件 | 改动 |
| ---- | ---- |
| `base/xconstants.h` | 新增 `XR_TTASK` 类型标签 |
| `coro/xtask.h` (新) | XrTask 结构体定义 |
| `coro/xtask.c` (新) | xr_task_wake_waiter, xr_task_alloc |
| `vm/xvm_value.h` | xr_value_is_task, xr_value_to_task, xr_value_from_task |
| `runtime/gc/xgc.c` | GC mark/traverse for XrTask |
| `runtime/gc/xcoro_gc.c` | XrTask 的 GC 遍历（mark task->result） |

### Phase 2: 修改 `go` / await 路径

| 文件 | 改动 |
| ---- | ---- |
| `vm/xvm_cold_paths.c` | vm_spawn_cont: 分配 Task，base[a] = task |
| `vm/xvm_cold_paths.c` | vm_await: 操作 Task 而非 Coro |
| `vm/xvm_cold_paths.c` | vm_await_all/any/timeout: 同上 |
| `coro/xworker.c` | worker_handle_vm_result: executor_complete |
| `coro/xcoro.c` | xr_coro_wake_waiter → xr_task_wake_waiter |

### Phase 3: XrCoroutine 瘦身

| 文件 | 改动 |
| ---- | ---- |
| `coro/xcoroutine.h` | 删除 result, error, await_*, waiter_*, scope 字段 |
| `coro/xcoroutine.h` | 新增 `XrTask *task` 字段 |
| `coro/xcoro.c` | 更新 coro_init_common, recycle_local |
| `coro/xcoro_pool.h` | 调整 slab 大小（struct 变小） |

### Phase 4: fire-and-forget 简化

fire-and-forget 的 `go`（不 await）：
- Task 仍然分配（GC 管理，无泄漏风险）
- Executor 完成后立即回收（不需要 `XR_CORO_GC_RECYCLABLE` 延迟回收）
- 删除 `pending_recycle_coro` 机制

### Phase 5: JIT 适配

| 文件 | 改动 |
| ---- | ---- |
| `jit/xir_builder_misc.c` | OP_AWAIT: xr_jit_await 操作 Task |
| `jit/xir_builder_misc.c` | OP_GO: xr_jit_spawn 返回 Task |

## 四、收益分析

### 4.1 内存泄漏：彻底解决

```
Before:  Executor 完成后不回收 → 每轮 500K × 2.3KB = 1.15GB 泄漏
After:   Executor 完成后立即回收，Task (~48B) 由 GC 管理
         500K × 48B = 24MB（Task） + Pool 循环使用（Executor）
```

### 4.2 re-await：零成本

```
Before:  回收 Executor → result 丢失 → re-await 挂起/crash
After:   result 缓存在 Task 中，Executor 已回收不影响
         re-await 走 fast path: if (state == COMPLETED) return result
```

### 4.3 XrCoroutine 瘦身

```
Before:  XrCoroutine ~640B (含 await 协调、result、error、scope...)
After:   XrCoroutine ~500B (纯执行体)
         XrTask ~48B (轻量句柄)
         节省 ~90B/协程 + 更好的缓存局部性
```

### 4.4 代码简化

```
Before:  vm_await 需要 CAS(await_state) + check(FLG_DONE) + release_heap/recycle
         8+ 个 release_heap/recycle 调用点，每个有不同的前置条件
After:   vm_await 只操作 Task（GC 对象，无需手动释放）
         Executor 回收集中在一个地方（executor_complete）
```

### 4.5 架构清晰度

```
Before:  XrCoroutine = 用户句柄 + 执行体 + 结果 + 调度（全耦合）
After:   XrTask = 用户句柄 + 结果（GC 管理，语义简单）
         XrCoroutine = 执行体 + 调度（Pool 管理，生命周期受控）
```

## 五、风险与缓解

| 风险 | 缓解 |
| ---- | ---- |
| GC 分配 Task 的开销（每次 go） | ~48B 在 Immix 上极快（bumper alloc），远小于 Executor 的 pool alloc |
| Task 的 deep copy 开销 | 只在 Executor 完成时做一次，与现有 vm_await_read_result 等价 |
| Task.result 的 GC 归属 | 分配在父协程的 coro_gc 或 isolate 的共享 GC 堆上 |
| Channel 操作仍操作 Coro | Channel 是调度器概念，不涉及 Task，无改动 |
| 旧测试中 `typeof(go fn())` 返回类型 | 从 "coroutine" 改为 "task"（无向后兼容顾虑） |

## 六、Kotlin 参考总结

| Kotlin 概念 | xray 对应 | 状态 |
| ----------- | --------- | ---- |
| `Deferred<T>` | `XrTask` | 本方案核心 |
| `Job._state` 结果缓存 | `XrTask.result` | 本方案核心 |
| 多次 `await` 返回相同值 | Task.state == COMPLETED 直接返回 | 自然支持 |
| `Job` 状态机 (New→Active→Completing→Completed) | `XrTask.state` (PENDING→COMPLETED/FAILED) | 简化版 |
| `invokeOnCompletion` 回调 | `xr_task_wake_waiter` | 直接映射 |
| 协程体 GC 回收 | Executor pool 回收 | 等价效果 |
| CPS 状态机编译 | 未来方向（先 AOT 实验） | 长期路线 |
