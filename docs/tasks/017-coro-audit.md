# Coro 模块审计实施文档（`src/coro`）

> **⚡ 开发原则：不考虑向后兼容性！**
>
> `src/coro` 是当前 VM 的并发运行时底座，也是后续 AOT/CPS 并发方案计划直接复用的核心模块。
>
> 因此这里已经确认的 correctness / ownership / architecture 问题，不应该继续维持“能跑就先不动”的状态：
>
> - 不保留错误的跨 worker 访问模型
> - 不保留“超时有时生效、有时静默失效”的语义
> - 不保留已知分层违规
> - 不保留“语言级回归不少，但底层单测几乎没有”的工程状态
>
> **本文定位**：把这次 `src/coro` 源码审计结论收敛成可执行实施方案，优先处理 correctness、并发所有权、架构边界和测试覆盖。
>
> **与 `docs/archive/coro_refactor_plan.md`（已完成归档）的关系**：
>
> - `coro_audit_plan.md`：先收敛审计确认的高风险问题
> - `coro_refactor_plan.md`：再推进大文件拆分、广义性能优化、长期演进
> - 两者冲突时，以本文件的 P0/P1 风险收敛优先
>
> **本文不是**对现有 `coro_refactor_plan.md` 的替代，而是一次“先止血、再重构”的补充。

## 1. 背景

这次审计覆盖了 `src/coro` 的主要热点路径和语义节点，包括：

- `xworker.c` / `xworker_runq.c` / `xworker_sched.c` / `xworker_exec.c`
- `xworker_blocked.c` / `xworker_sysmon.c`
- `xchannel.c`
- `xtimer_wheel.c`
- `xnetpoll.c`
- `xyieldable.c` / `xresume.c`
- `xtask.c`
- `xasync.c`

同时也检查了测试覆盖和架构边界：

- `tests/regression/11_coroutine`
- `tests/coroutine_safety`
- `tests/unit/coro`
- `src/coro` 对 `vm/jit` 的 include 方向

### 1.1 当前基线

`src/coro` 的优点很明确：

- 调度器快路径做得比较成熟，已经有 continuation stealing、LIFO locality、分层 poll 等优化意识
- channel、timer wheel、netpoll、async pool 都不是“玩具实现”，而是已经进入可用 runtime 级别
- 语言级回归中，`channel/select/await/linked scope/supervisor/pipeline` 等主路径已有不少 `.xr` 用例

但当前也存在明显的“不成熟边角已经进入主路径”的问题：

- 跨 worker 唤醒路径没有完全服从 owner-private 模型
- timeout / deadline 语义在 worker 迁移下不稳定
- structured concurrency 的任务树没有明确单线程 owner
- `time.after` 仍和 `timer wheel` 双轨并存
- `coro` 按项目层次本应是 L3，但目前明显反向 include 到了 L5 `vm/jit`
- `tests/unit/coro` 目前只有 `test_mpsc_queue.c` 和 `test_steal_queue.c`

### 1.2 这次审计的核心判断

这不是一个“继续小修小补、顺便做点 benchmark”的任务，而是一个 **correctness + ownership + architecture + testing** 的联合收敛任务。

如果继续按当前状态迭代，风险会集中在四个方面：

1. **并发正确性风险**
   - 跨 worker 直接改 owner-private 结构
   - 普通链表承载跨 worker 生命周期事件

2. **语义漂移风险**
   - deadline 在迁移场景下静默失效
   - `time.after` 与 `timer wheel` 并存导致 timeout 模型不统一

3. **架构沉积风险**
   - `coro` 逐步吸收 `vm/jit` 内部细节，后续更难拆边界

4. **回归盲区风险**
   - 语言级黑盒能证明“经常工作”
   - 但不能证明底层 ownership / wake routing / task tree / timer cleanup 没有竞态

---

## 2. 当前问题总览

### 2.1 P0：正确性 / 并发所有权问题

### CORO-01 `xr_runtime_wake_channel()` 的跨 worker 唤醒路径破坏 owner-private 模型

当前路径：

- `xcoro.c` 的 `xr_runtime_wake_channel()` 会线性扫描所有 worker
- 对远端 worker 直接调用 `xr_worker_wake_one()` / `xr_worker_wake_select()`
- `xworker_blocked.c` 的 blocked queue 明确是 owner-private
- `xr_worker_wake_one()` 最后会走到 `xr_worker_push_lifo()`
- `xr_worker_push_lifo()` 在跨 worker 情况下会退化到 `xr_worker_push()`
- `xworker_runq.c` / `xsteal_queue.h` 又明确 Chase-Lev push 是 owner-thread-only

后果：

- 热路径复杂度是 `O(worker_count)`
- 更严重的是存在跨 worker 直接改远端 blocked queue / runq 的正确性风险
- close / buffered send / unbuffered rendezvous / select 唤醒都可能受影响

这是当前优先级最高的问题。

> **源码审查补充**：`xchannel.c` 的 `chan_direct_send/recv → channel_wake_coro_ex()` 路径实际上 **已经是安全的**：
> - non-close wake → 拉到当前 worker 的 LIFO（owner-local 操作）
> - close wake + 跨 worker → 走 `xr_worker_inbox_enqueue()`（MPSC，安全）
>
> 真正破坏 ownership 的是 `xcoro.c` 的 `xr_runtime_wake_channel()` / `xr_runtime_wake_channel_all()` 这两个函数——它们绕过 channel waitq，直接扫描远端 worker 的 blocked bucket 并操作远端 runq。
>
> 同时，`xr_coro_ready()` 是已有的正确跨 worker 唤醒范例：`next=true` push 当前 worker LIFO，`next=false` 走 inbox。Phase 0 可直接参考这两个正确路径。

### CORO-02 `xnetpoll.c` 的 deadline 在 worker 迁移下会静默失效

当前实现里，`xr_netpoll_set_deadline()` 在发现当前协程不在 `pd` owner worker 上时，会直接跳过 timer wheel 的 set/cancel。

后果：

- API 层看起来已经设置 timeout
- 运行时却可能没有真正挂上 deadline
- 表现不是 crash，而是偶发“某些 I/O 超时失灵”
- 这类 bug 最难排查，因为它依赖调度时机

### CORO-03 `xtask.c` 的父子任务层级是普通链表，但生命周期事件可能跨 worker 并发发生

`xtask.c` 中：

- `parent`
- `first_child`
- `next_sibling`

都是普通指针和普通链表操作。

但以下路径可能跨 worker 发生：

- 子任务终态进入 `xr_task_finalize()`
- 父任务取消进入 `xr_task_cancel_tree()`
- 失败向上传播进入 `xr_task_fail_with_propagation()`

后果：

- 可能出现重复 detach、链表断裂、父任务过早 finalize
- linked / supervisor / monitored 语义建立在不稳定底层之上
- 这类问题不一定在普通回归中稳定复现，但属于结构性竞态风险

### CORO-04 `sysmon` 跨线程直接修改 `XrCFunction` 元数据

`xworker_sysmon.c` 会直接写：

- `cfn->auto_slow_count++`
- `cfn->cfunc_class = XR_CFUNC_SLOW`

但这些字段当前不是原子，也没有同步保护。

后果：

- 这是标准数据竞争
- 在 ARM64 这类内存模型下更不应该靠“通常没事”侥幸通过
- sysmon 原本是观察者，当前却直接越权改执行元数据

### CORO-10 `xr_coro_wake_waiter()` 中 scope child list 与 CORO-03 同类竞态

`xcoro.c` 的 `xr_coro_wake_waiter()` 在子协程完成时操作 `XrScopeContext` 的 child list：

- `scope->first_child` / `coro->scope_sibling` 是普通链表
- 遍历 + unlink 操作（`xcoro.c:1337-1345`）在任意 worker 上执行
- 多个子协程同时完成 → 并发修改同一条 scope child 链表

更严重的是 linked scope cancel 路径（`xcoro.c:1350-1364`）在遍历中同时调用 `xr_coro_cancel()` + `xr_task_cancel()` + `xr_task_wake_waiter()`，可能触发递归跨 worker 操作。

后果：

- 与 CORO-03 的 task child list 是同一类结构性竞态
- scope 是 linked/supervisor/wait 语义的底层载体，竞态影响面大
- 应在 Phase 2 中与 task child list 一并改造

### 2.2 P1：语义一致性 / 性能与生命周期问题

### CORO-05 `time.after` 和 `timer wheel` 仍是双轨并存

当前：

- `sleep` / I/O deadline 主要走 `xtimer_wheel.c`
- `time.after` 仍通过 `xchannel.c` 的 timer channel + `xr_channel_timer_ready()` 轮询路径实现

后果：

- timeout 语义分裂成两套实现
- 维护成本翻倍
- 轮询式 timer channel 更容易产生额外检查开销和行为漂移

### CORO-06 `xasync.c` 的 ready queue 消费路径还有明显低效点

现状：

- producer 是 MPSC
- consumer `xr_async_check_ready()` 仍逐个 CAS 弹出
- 注释里已经明确提示可以优化成 `atomic_exchange` 一次性 whole-list drain

后果：

- 完成事件多的时候会有不必要 CAS 消耗
- correctness 风险不高，但属于已经确认的热路径低效点

### CORO-07 `xtimer_wheel.c` 的取消/销毁路径仍有生命周期债务

已确认的问题：

- cross-worker cancel 每次分配 `XrCanceledTimerNode`
- cancel 风暴下会放大分配压力
- destroy 时若 `head == stub`，仅从 `head` 开始释放可能遗漏 `stub.next` 后面的残留链

这部分更偏生命周期和清理正确性，不是第一优先级 crash 风险，但应该在 timer 模型收敛前一起处理。

> **可提前修复的独立 fix**：`xr_timer_wheel_destroy()` 的内存泄漏是一个 standalone bug——MPSC Vyukov 队列的 `head` 初始值 = `&cq->stub`，destroy 循环因 `node == &cq->stub` 直接退出，`stub.next` 后面的未消费 cancel 节点全部泄漏。修复方案：改为从 `cq->stub.next` 开始遍历释放。此 fix 不依赖任何 Phase 改动，可随时合入。

### 2.3 P1 / P2：架构与工程质量问题

### CORO-08 `src/coro` 明显存在 L3 → L5 的反向依赖

按项目规则：

```text
L0 base → L1 runtime/value → L2 runtime/gc, runtime/object
→ L3 runtime/class, runtime/symbol, coro → L4 module, frontend/*
→ L5 vm, jit → L6 aot → L7 api → L8 app/*
```

当前 `src/coro` 已确认有多个文件直接 include：

- `../vm/xvm_internal.h`
- `../vm/xvm.h`
- `../jit/xir_jit.h`
- `../jit/xjit_compile_queue.h`
- `../jit/xir_jit_debug.h`

> **逐文件清单（源码审查补充）**：
>
> | 文件 | 反向依赖 |
> |------|----------|
> | `xworker_internal.h` | `../vm/xvm_internal.h`（**传染源**，所有 worker 文件间接依赖） |
> | `xcoro.c` | `../vm/xvm_internal.h` |
> | `xresume.c` | `../vm/xvm.h`, `../vm/xvm_internal.h` |
> | `xyieldable.c` | `../vm/xvm_internal.h` |
> | `xnetpoll.c` | `../vm/xvm_internal.h` |
> | `xsocket.c` | `../vm/xvm_internal.h` |
> | `xworker_exec.c` | `../jit/xir_jit.h`, `../jit/xjit_compile_queue.h`, `../jit/xir_jit_debug.h` |
> | `xworker_sysmon.c` | `../jit/xir_jit_debug.h` |
> | `xworker.c` | `../jit/xir_jit_debug.h` |
>
> 最大传染源是 `xworker_internal.h` → `xvm_internal.h`，一个 include 污染所有 worker 文件。Phase 4 应优先处理此头文件。

后果：

- `coro` 不再是独立调度层，而是逐渐吸收了 VM/JIT 内部实现细节
- 后续任何运行时、AOT、JIT 路径都更难抽象复用
- 这违反仓库架构红线，不只是“风格问题”

### CORO-09 测试覆盖偏黑盒，底层单测不足

当前测试结构：

- `tests/regression/11_coroutine`：语言级回归较多
- `tests/coroutine_safety`：有 channel 深拷贝、pipeline、monitoring 等脚本
- `tests/unit/coro`：只有 `test_mpsc_queue.c`、`test_steal_queue.c`

缺失的关键单测包括：

- cross-worker channel wake routing
- blocked queue owner-only 约束
- task tree 并发完成/取消
- **scope child list 并发完成/取消**（CORO-10，与 task tree 同类）
- netpoll deadline + migration
- timer cancel / destroy cleanup
- async ready queue whole-list drain

---

## 3. 重构目标

### 3.1 最终目标

1. **owner-private 结构只允许 owner 操作**
   - runq / lifo / blocked queue / timer wheel slot / task child list / **scope child list** 都不能被远端线程直接改

2. **deadline / timeout 语义稳定**
   - 无论 worker 是否迁移，I/O timeout 都必须成立
   - `time.after` / `sleep` / I/O deadline 最终收敛到同一套 timer 语义

3. **structured concurrency 语义可信**
   - 任务树的 attach / detach / cancel / finalize 必须有明确 owner 模型

4. **`coro` 回到正确层级**
   - 不再直接依赖 `vm/jit` 内部头
   - 调度层只依赖抽象回调或接口

5. **测试覆盖能保护关键并发不变量**
   - 不再只靠 `.xr` 黑盒回归兜底
   - 对 wake routing、deadline、task tree、timer cleanup 有白盒单测

### 3.2 非目标

本次实施文档 **不要求** 一次完成以下内容：

- 所有 `src/coro` 文件拆分到最理想布局
- 所有 benchmark 的极致优化
- 一次性重写整个 worker / netpoll / timer 子系统
- 为所有路径补齐 TSan 级并发测试基础设施

策略是：**先修 correctness 和 ownership，再做大规模性能演进。**

---

## 4. 目标设计

### 4.1 Worker ownership 硬约束

未来需要明确这条规则：

- **只有 owner worker** 可以直接修改：
  - 本地 runq / lifo slot
  - 本地 blocked buckets / select waiters
  - 本地 timer wheel slot 链表
  - 以该 worker 为 owner 的 task child list
  - 以该 worker 为 owner 的 scope child list（`XrScopeContext.first_child / scope_sibling`）
- 远端线程只能做两类事情：
  - 原子标记
  - 投递命令 / 消息到 owner 的 inbox / command queue

这条规则应该成为 `src/coro` 的第一不变量。

### 4.2 Channel 唤醒模型

目标不是“找到 waiter 就直接远端唤醒”，而是：

```text
channel state change
→ 查询该 channel 可能有哪些 worker 有 waiter
→ 对当前 worker：本地直接 wake
→ 对远端 worker：投递 wake-channel 命令
→ 远端 worker 在自己的线程上执行 xr_worker_wake_one()/wake_select()
→ 再本地 push 到自己的 runq/lifo
```

关键点：

- `xr_runtime_wake_channel()` 不再直接改远端 blocked queue
- `xr_runtime_wake_channel()` 不再直接把 coro push 到远端 runq
- channel 自身要维护“可能有哪些 worker 在等待它”的 owner 索引

实现形式可以是：

- `worker_count <= 64` 时用 `_Atomic uint64_t waiter_worker_mask`
- worker 数更大时用动态 bitset / 小型 owner set

接口语义固定，内部实现可按 worker 上限决定。

### 4.3 Task tree owner 模型

任务树不应再允许“谁先到终态谁就随手改父链表”。

目标模型：

- 父任务 child list 有明确 owner
- 子任务完成时，只上报“child completed / child failed / child cancelled”事件
- 真正的 detach / finalize / child_count 归零判断，由父 owner 执行
- `cancel_tree` 采用“两阶段语义”：
  - 第一阶段：原子标记 cancel intent
  - 第二阶段：由 owner 遍历 children 并完成传播

### 4.4 Deadline owner 模型

`XrPollDesc` 的 timeout 语义需要明确：

- 要么 I/O wait 期间 pin 到 `pd` owner worker
- 要么所有 deadline set/cancel 都通过 owner 命令队列转发
- 绝不能再保留“非 owner 时直接跳过”的静默退化分支

推荐策略：

- **短期**：active wait 期间 `pd` owner 固定，非 owner 的 deadline 操作转发给 owner
- **长期**：仅在 `pd` 空闲且无 active timer 时允许显式迁移 owner

### 4.5 Timer 统一模型

最终希望保留的语义是：

- `sleep` / `time.after` / I/O deadline 都由同一套 timer runtime 驱动
- 语言层仍可以保留“`time.after` 返回 channel”的接口
- 但底层不再需要单独维护一套 polling timer channel 逻辑

即：

```text
time.after(ms)
→ 创建只读 channel / promise-like wait object
→ 在 timer wheel 上注册一次性 timer
→ timer 到期后向该对象投递结果
```

而不是在 VM 每次 recv 时主动检查 `xr_channel_timer_ready()`。

---

## 5. 分阶段实施计划

### 阶段总览

| Phase | 目标 | 优先级 | 产出 |
|------|------|--------|------|
| 0 | 收敛 channel/select 跨 worker 唤醒正确性 | P0 | 不再跨线程直接改远端 blocked queue / runq |
| 1 | 收敛 netpoll deadline 和 timer 生命周期正确性 | P0 | 不再静默丢 timeout，timer cleanup 有明确不变量 |
| 2 | 收敛 task tree 与 sysmon 元数据竞态 | P0/P1 | structured concurrency owner 明确，cfunc 元数据不再数据竞争 |
| 3 | 收敛 timer 双轨和 async 热路径 | P1 | `time.after` 走统一 timer runtime，async ready queue whole-list drain |
| 4 | 修复架构边界 | P1 | `coro` 不再直接 include `vm/jit` 内部头 |
| 5 | 补齐测试与回归体系 | P1 | `tests/unit/coro` 和回归脚本覆盖关键并发路径 |

---

### 5.1 Phase 0 — Channel / Select 跨 worker 唤醒正确性收敛

### 目标

1. 删除远端 blocked queue 的直接操作
2. 删除远端 runq 的直接 push
3. 保留当前 channel 语义，但重建 wake routing 管线

> **实施范围精确化**：`xchannel.c` 的 `chan_direct_send/recv → channel_wake_coro_ex()` 路径已经是安全的（见 CORO-01 源码审查补充）。
> Phase 0 的核心改动集中在 **`xr_runtime_wake_channel()` / `xr_runtime_wake_channel_all()` 这两个函数**，以及它们调用的远端 `xr_worker_wake_one()` / `xr_worker_wake_select()` 路径。
> 可直接参考 `channel_wake_coro_ex()` 和 `xr_coro_ready()` 两个已有正确范例。

### 主要改动

#### 0.1 给 channel 增加 waiter owner 索引

涉及文件：

- `src/coro/xchannel.c`
- `src/coro/xchannel.h`
- `src/coro/xworker_blocked.c`
- `src/coro/xworker_sysmon.c`

要求：

- send waiter / recv waiter / select waiter 在 block 时，把 owner worker 标记到 channel owner 索引里
- unblock / wake 后，及时清理索引位
- select 要对每个关联 channel 都建立 owner 标记，而不是只在某个“首 channel”上挂钩

#### 0.2 把 `xr_runtime_wake_channel()` 改成“调度唤醒命令”，而不是直接远端唤醒

涉及文件：

- `src/coro/xcoro.c`
- `src/coro/xworker_sched.c`
- `src/coro/xworker.h`

要求：

- `xr_runtime_wake_channel()` 只做两件事：
  - 本地 worker 可直接处理的本地 wake
  - 向远端 worker inbox / command queue 投递 wake-channel 请求
- 不再对远端 worker 直接调用 `xr_worker_wake_one()` / `xr_worker_wake_select()`

#### 0.3 远端 worker 只在本线程执行 local wake

涉及文件：

- `src/coro/xworker_blocked.c`
- `src/coro/xworker_sched.c`
- `src/coro/xworker_runq.c`

要求：

- `xr_worker_wake_one()` / `xr_worker_wake_select()` 只用于 owner worker 本地执行
- `xr_worker_push()` / `xr_worker_push_lifo()` 不再从远端线程写入目标 worker runq
- 若发现跨 worker 代码路径仍能触达这些函数，应直接视为 bug

#### 0.4 close / buffered / rendezvous / select 统一到同一条 wake routing 模型

涉及文件：

- `src/coro/xchannel.c`
- `src/coro/xcoro.c`

要求：

- channel close 唤醒也必须走同样的 owner routing
- 不允许保留“普通 send/recv 走一套，close 走另一套，select 再走第三套”的状态

### 验收标准

- 源码中不再存在远端直接调用 `xr_worker_wake_one(remote_worker, ...)` 的主线路径
- 源码中不再存在远端直接 `xr_worker_push(remote_worker, coro)` 的主线路径
- `buffered send/recv`、`unbuffered rendezvous`、`select`、`close` 在跨 worker 情况下行为稳定
- `xr_runtime_wake_channel()` 的复杂度从“扫全部 worker + 远端直接改结构”收敛为“查询 owner 索引 + 命令投递”
- **防御性断言**：`xr_worker_push()` 和 `xr_worker_wake_one()` 入口加上 `XR_DCHECK(xr_current_worker() == worker)` 捕捉残留违规

### 建议新增测试

- `tests/unit/coro/test_channel_wake.c`
  - 本地 wake
  - 跨 worker wake
  - close wake
- `tests/unit/coro/test_select_wake.c`
  - 多 channel select
  - 同一 channel 多 worker waiters
- `tests/regression/11_coroutine/`
  - 新增跨 worker `select` 和 channel close 场景

---

### 5.2 Phase 1 — Netpoll deadline / timer 生命周期正确性收敛

### 目标

1. 删除 deadline 的静默失效分支
2. 明确 `XrPollDesc` owner 语义
3. 补齐 timer wheel 的销毁/取消生命周期不变量

### 主要改动

#### 1.1 删除 `non-owner -> skip timer ops` 分支

涉及文件：

- `src/coro/xnetpoll.c`

要求：

- `xr_netpoll_set_deadline()` 不再在非 owner 场景下直接跳过 timer set/cancel
- 所有 deadline 操作都必须真正落到某个 owner 上执行

#### 1.2 明确 `XrPollDesc` owner 策略

涉及文件：

- `src/coro/xnetpoll.c`
- `src/coro/xworker_sched.c`
- `src/coro/xyieldable.c`

推荐策略：

- active read/write wait 期间，`pd` owner 固定
- 非 owner 线程设置/取消 deadline 时，通过 owner command queue 执行
- `pd` 只有在空闲态才能迁移 owner

#### 1.3 收敛 timer wheel destroy / cancel cleanup

涉及文件：

- `src/coro/xtimer_wheel.c`

要求：

- destroy 时明确从 `stub.next` 开始完整释放 pending canceled queue
- 为 canceled queue 建立显式不变量：
  - `head/tail/stub.next` 三者关系明确
  - drain 前后状态可断言
- 为 cancel path 增加更强的调试断言，防止重复回收 / 悬挂节点

### 验收标准

- 源码中不再保留“非 owner 直接跳过 deadline”分支
- 在 work stealing / worker 迁移场景下，I/O timeout 仍可稳定触发
- timer wheel destroy 后不存在残留 canceled node 链

### 建议新增测试

- `tests/unit/coro/test_netpoll_deadline.c`
  - set deadline
  - cancel deadline
  - 非 owner 设置 deadline
- `tests/unit/coro/test_timer_wheel.c`
  - cancel queue drain
  - destroy cleanup
- `tests/regression/11_coroutine/`
  - 新增 I/O timeout / sleep / steal 后 timeout 场景

---

### 5.3 Phase 2 — Structured Concurrency / Sysmon 竞态收敛

### 目标

1. task tree 明确 owner
2. 子任务终态不再直接改父链表
3. sysmon 不再直接进行无同步元数据写入

### 主要改动

#### 2.1 给 task tree 建立单 owner 生命周期模型

涉及文件：

- `src/coro/xtask.c`
- `src/coro/xtask.h`
- 相关调用点（`vm/xvm_cold_paths.c`、worker completion 路径）

要求：

- attach / detach / child_count 归零判断由父 owner 执行
- 子任务终态只投递 child-completed / child-failed / child-cancelled 事件
- `xr_task_cancel_tree()` 改成“先标记、再由 owner 展开”的两阶段模型

#### 2.2 收敛 linked / supervisor / monitored 的传播链

要求：

- 先保证底层 owner 模型成立，再保留现有语义
- 不再允许任意 worker 在任意时刻递归修改整棵 task tree

#### 2.2b 统一收敛 `XrScopeContext` child list 竞态（CORO-10）

涉及文件：

- `src/coro/xcoro.c`（`xr_coro_wake_waiter()` 中 scope child list 操作）
- `src/coro/xcoroutine.h`（`XrScopeContext` 定义）

要求：

- `scope->first_child` / `coro->scope_sibling` 的 unlink 必须由 owner 执行，不允许多个 worker 并发修改
- linked scope cancel 路径（`xcoro.c:1350-1364`）改为“标记 cancel + owner 展开”两阶段模型，与 2.1 task tree 的改法保持一致
- 与 task child list 的 owner 模型统一，避免“一套事物两种生命周期管理”

#### 2.3 让 `XrCFunction` 的 slow-promotion 成为同步良好的单向状态迁移

涉及文件：

- `src/coro/xworker_sysmon.c`
- `src/runtime/xexec_frame.h`
- `src/vm/xvm.c`
- `src/vm/xvm_helpers.c`

要求：

- `auto_slow_count` / `cfunc_class` 至少改为原子字段，或迁移到 side-table/owner message 机制
- `FAST -> SLOW` 采用单向 CAS 升级
- sysmon 保持“观察者 + 决策触发者”角色，不直接做无同步共享写入

### 验收标准

- task child list 不再被多个 worker 直接并发修改
- **scope child list**（`XrScopeContext.first_child / scope_sibling`）同样不再被多 worker 直接并发修改
- 父任务 finalize/cancel 与子任务终态处理顺序有明确 owner 语义
- `sysmon` 对 `XrCFunction` 的修改不再是数据竞争

### 建议新增测试

- `tests/unit/coro/test_task_tree.c`
  - child complete / parent cancel 交错
  - linked / supervisor propagation
- `tests/unit/coro/test_sysmon_cfunc.c`
  - auto slow promotion 单向升级
- `tests/regression/11_coroutine/`
  - 新增 linked cancel / nested scope 交错场景

---

### 5.4 Phase 3 — Timer 模型收敛与 Async 热路径优化

### 目标

1. `time.after` 不再依赖 polling timer channel
2. timer cancel path 减少额外分配
3. async ready queue whole-list drain

### 主要改动

#### 3.1 让 `time.after` 改由 timer wheel 驱动

涉及文件：

- `src/coro/xchannel.c`
- `src/vm/xvm.c`
- `src/coro/xtimer_wheel.c`

要求：

- 保留语言层“返回 channel”的语义
- 但底层由 timer wheel 一次性回调投递结果
- 删除 VM recv 路径对 `xr_channel_timer_ready()` 的特殊轮询依赖

#### 3.2 为 canceled timer node 建立 freelist / pooling

涉及文件：

- `src/coro/xtimer_wheel.c`

要求：

- 避免 cancel 风暴下每次都堆分配 `XrCanceledTimerNode`
- 优先使用 owner-local freelist 或 intrusive node 设计

#### 3.3 `xasync.c` 改成 whole-list drain

涉及文件：

- `src/coro/xasync.c`

要求：

- consumer 使用 `atomic_exchange(head, NULL)` 一次拿到整条 ready 链
- 必要时本地 reverse 保序
- `max_per_check` 可继续保留，但处理成本应从“每个 job 一次 CAS”下降

### 验收标准

- `time.after` 不再依赖 VM 轮询 `xr_channel_timer_ready()`
- cancel storm 下 timer 分配压力下降
- async completion 多时 ready drain 的 CAS 次数显著下降

### 建议新增测试

- `tests/unit/coro/test_time_after.c`
  - 单 shot
  - cancel / close / recv 次序
- `tests/unit/coro/test_async_ready.c`
  - 多 producer ready completion
- `tests/regression/11_coroutine/`
  - 新增 `time.after + select` 组合场景

---

### 5.5 Phase 4 — 架构边界修复

### 目标

1. `src/coro` 回到 L3 正确层级
2. worker / timer / netpoll 不再直接依赖 `vm/jit` 内部头
3. 为后续 AOT/CPS 复用建立更干净的调度抽象

### 主要改动

#### 4.1 为 VM/JIT 相关能力定义 callback / ops 抽象

涉及文件：

- `src/coro/xworker_exec.c`
- `src/coro/xyieldable.c`
- `src/coro/xresume.c`
- `src/coro/xworker_sysmon.c`
- `src/coro/xnetpoll.c`
- 新增抽象头（名称可在实施时确定）

要求：

- `coro` 不再直接 include `vm/xvm_internal.h`、`jit/*`
- 调度器只依赖“如何执行/恢复/调试/上报”的抽象接口
- VM/JIT 侧负责提供具体实现

#### 4.2 把 JIT/Debug 特定逻辑从通用调度路径里剥离

要求：

- JIT guard page / debug safepoint / compile queue 相关逻辑应由更高层挂接
- `worker_exec` 保持“执行调度”和“状态转换”的核心职责，不继续吸收上层策略细节

### 验收标准

- `src/coro` 下不再直接 include `../vm/` 或 `../jit/` 内部头
- `scripts/check_architecture.sh` 不再报 `coro -> vm/jit` 方向问题
- 调度层对 VM/JIT 的依赖收敛为抽象回调

### 建议新增测试

- 以架构检查和编译验证为主
- 必要时补一组 smoke tests，确保 callback 接线不破坏现有 VM/JIT 路径

---

### 5.6 Phase 5 — 测试与回归体系补齐

### 目标

把 `src/coro` 从“主要靠语言级回归证明能工作”的状态，提升到“关键所有权和生命周期不变量有专门回归保护”的状态。

### 主要改动

#### 5.1 扩充 `tests/unit/coro`

建议新增：

```text
tests/unit/coro/
├── test_mpsc_queue.c           # 现有
├── test_steal_queue.c          # 现有
├── test_channel_wake.c         # wake routing / close / cross-worker
├── test_select_wake.c          # select waiter registration / wake / cleanup
├── test_task_tree.c            # attach/detach/finalize/cancel ownership
├── test_netpoll_deadline.c     # deadline set/cancel/migration
├── test_timer_wheel.c          # cancel queue / destroy cleanup
├── test_time_after.c           # timer wheel-backed time.after
└── test_async_ready.c          # whole-list drain / fairness basics
```

#### 5.2 扩充语言级回归

建议在：

- `tests/regression/11_coroutine/`
- `tests/coroutine_safety/`

补充以下方向：

- cross-worker channel close fanout
- select + timeout + channel close 交错
- linked/supervisor scope 的嵌套取消
- async job completion 压力
- `time.after` 与 `await/select` 组合

#### 5.3 并发问题默认走 ASan 构建验证

凡涉及：

- wake routing
- task tree 生命周期
- timer cancel/destroy
- netpoll deadline

都应默认执行：

- `/build`
- `/test`
- `/build-asan`
- `/test`

若后续仓库引入可用的 TSan 路径，再补充到本计划中；当前不把 TSan 作为本阶段前置条件。

### 验收标准

- `tests/unit/coro` 不再只有队列测试
- 所有 P0/P1 路径都至少有一类专门测试覆盖
- 文档化的关键不变量有对应测试名可追溯

---

## 6. 与现有 `coro_refactor_plan.md` 的协同顺序

仓库里已存在一份 `docs/archive/coro_refactor_plan.md`（主体工作完成，已归档），它更偏：

- 大文件拆分
- 热路径性能演进
- lock-free 化
- benchmark 驱动重构

这份文档本身没有问题，但执行顺序需要调整。

### 建议顺序

1. **先执行本文件 Phase 0-2**
   - 先把 wake ownership、deadline 语义、task tree owner、sysmon 数据竞争收敛

2. **再视需要穿插 `coro_refactor_plan.md` 中的文件拆分工作**
   - 尤其是 `xworker.c` / `xworker_exec.c` / `xworker_sched.c` 的拆分
   - 这样后续 correctness 修复会更容易落地

3. **本文件 Phase 3 完成后，再推进更激进的性能优化**
   - 如 timer cancel pooling、更多 lock-free 结构、benchmark 微调等

### 不建议的顺序

不建议在以下问题尚未收敛前，就优先推进更激进的结构改造：

- `xr_runtime_wake_channel()` 的跨 worker direct wake
- `xtask` 普通链表承载跨 worker 生命周期事件
- `xnetpoll` deadline 静默失效

否则很容易把 bug 从“大而明显”的旧结构，搬运到“更分散、更难追踪”的新结构里。

---

## 7. 推荐验证方式

每个阶段落地时，至少执行：

```text
/build
/test
```

凡涉及并发正确性、生命周期、close/cancel、deadline、timer cleanup 的阶段，额外执行：

```text
/build-asan
/test
```

如果阶段引入了新的 unit tests：

- 优先先跑对应测试目标
- 再跑完整 `/test`

如果阶段涉及 include 方向或模块边界：

```text
scripts/check_architecture.sh
```

---

## 8. 最终判断

`src/coro` 当前的状态，不是“设计很差、需要推倒重来”，而是：

- **快路径和整体思路已经相当成熟**
- **但 ownership / correctness / architecture 的几个边角已经进入主路径**
- 这些边角如果不先修，后续再做性能优化、AOT 复用、文件拆分，成本都会更高

因此最合理的策略不是直接冲向“大重构”，而是先完成本文件定义的风险收敛：

- 先把跨 worker wake 路线改正确
- 再把 deadline、task tree、sysmon 这些竞态点收敛
- 然后统一 timer 模型、补测试、清边界
- 最后再继续做更大范围的结构与性能演进
