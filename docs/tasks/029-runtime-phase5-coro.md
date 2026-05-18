# Runtime `src/coro/` 层分析（`029`）

> 本轮聚焦 `src/coro/`：`xcoro.*`、`xworker.*`、`xtask.*`、`xchannel.*`、`xdeep_copy.*`、timer/select/scope/blocked queue。
>
> 这一层不是单纯的“调度器实现”，而是 runtime 里最强的 ownership 汇合点：
>
> - per-coro heap
> - isolate fixedgc
> - system heap shared/refcount
> - worker-private state
> - task/scope/await 这些跨线程同步协议
>
> 因此前两轮的 object/class 结论都会在这里重新出现：instance、json、closure、shared object、fixedgc object 如何跨 coroutine 流动，都是 `src/coro/` 真正的边界问题。

## 1. 本轮结论

- **`src/coro/` 的本质不是“一个调度模块”，而是 runtime 中负责跨 owner 迁移和状态同步的控制层。**

- **`XrCoroutine` 本体通常来自 sysheap/pool，不是 per-coro GC object；但它又携带自己的 `XrCoroGC`。**

- **`XrTask` 才是 per-coro GC 上的用户可见 handle。**
  - executor (`XrCoroutine`) 是执行体
  - task (`XrTask`) 是 await / structured concurrency / monitor 的稳定句柄

- **`XrChannel` 是 shared system-heap object。**
  - refcount shared
  - wait queue / buffer / close state 直接跨 coroutine 使用

- **deep-copy 是 `src/coro/` 的真实数据边界。**
  - shared object：直接 incref
  - deep-copy object：复制到目标 coro heap 或 fixedgc/shared heap
  - non-deep types：按值或原指针透传

- **worker/runtime 绝大部分是 isolate-owned malloc graph，不走 GC。**
  - `XrRuntime`
  - `XrWorker[]`
  - `XrMachine[]`
  - async pool / timer wheel / blocked bucket / inbox / poll / handoff state

- **高风险同步点仍然集中在“跨 worker child list / scope list / wake routing”上。**

## 2. 模块地图

| 子域 | 关键文件 | 真实职责 |
|---|---|---|
| coroutine core | `xcoro.c` `xcoroutine.h` | coroutine 创建/初始化/ready/wake/scope bookkeeping |
| runtime/worker | `xworker.h` `xworker.c` `xworker_*` | worker lifecycle、runq/inbox/blocked/timer/handoff/exec |
| task | `xtask.h` `xtask.c` | GC handle、await、child tree、completion listeners |
| channel | `xchannel.h` `xchannel.c` | shared channel、buffer、wait queue、close/wake |
| deep copy | `xdeep_copy.c` | object 跨 coroutine / shared heap 迁移协议 |
| scope/select/timer | `xcoroutine.h` + worker blocked/timer 逻辑 | structured concurrency 与阻塞唤醒辅助机制 |

## 3. owner 与堆域

### 3.1 `XrCoroutine`：sysheap/pool 执行体 + 自带 per-coro GC

`xr_coro_create()` 的真实分配路径：

- 优先 `xr_coro_pool_get(runtime)`
- 否则 `xr_sysheap_alloc_coro(X->sys_heap)`
- 再 fallback `xr_malloc(sizeof(XrCoroutine))`

bootstrap main coroutine 也是类似：

- `xr_sysheap_alloc_coro()`
- 或 `xr_malloc`

这说明 coroutine 本体不是“分配在自己的 coro_gc 上”的对象，反而是：

- isolate/system-owned executor shell
- 可被 pool recycle
- 内部再挂一份 `coro->coro_gc`

也就是说这里的 owner 分裂为两层：

- **外壳 (`XrCoroutine`)：sysheap/pool/malloc**
- **对象堆 (`XrCoroGC`)：该 coroutine 私有 GC heap**

### 3.2 `XrTask`：per-coro heap 上的用户句柄

`xr_task_create()`：

- `xr_alloc(executor, sizeof(XrTask), XR_TTASK)`

这意味着 task 的 owner 是：

- executor coroutine 的分配语义
- 即优先 executor 的 `coro_gc`，必要时 fallback

代码注释也直接说明：

- task 要通过 `coro->task` 被 mark root 持有
- 不能在 parent 仍引用它时先回收 executor，否则 await 路径会扫到悬空内存

因此 task 的设计目的很明确：

- 把可 await、可监控、可挂 child tree 的状态放到 GC handle 上
- 避免直接把这些协议绑在可 recycle 的 executor 外壳上

### 3.3 `XrChannel`：shared system heap object

`xr_channel_new()`：

- `xr_sysheap_alloc_shared(..., XR_TCHANNEL)`
- 然后 `xr_shared_set_refc(&ch->gc_header, 1)`

buffer 采用单次分配：

- `sizeof(XrChannel) + buffer_size * sizeof(XrValue)`
- inline buffer 指向 `ch + 1`

所以 channel 的 owner 很清楚：

- shared/refcount object
- 所有 coroutine 都能直接持有和传递
- 不需要 deep copy 本体

### 3.4 `XrRuntime` / `XrWorker` / `XrMachine`：isolate-owned malloc graph

`xr_runtime_create()`：

- `runtime = xr_calloc(1, sizeof(XrRuntime))`
- `machines = xr_calloc(num_workers, sizeof(XrMachine))`
- `workers = xr_calloc(num_workers, sizeof(XrWorker))`
- `async_pool = xr_calloc(1, sizeof(XrAsyncPool))`

worker 内部再初始化：

- timer wheel
n- local poll
- MPSC inbox
- blocked buckets
- continuation deque
- jit guard page

这些对象都是：

- isolate lifetime
- 手动 init/destroy
- 与 GC 毫无关系

### 3.5 worker-private state 是 owner 明确的“不可共享热数据”

`XrWorker`/`XrProc` 里大量结构天然是 owner-private：

- runq
- lifo slot
- blocked queue
- timer wheel
- local poll
- jit scratch
- local free list

这部分的核心不变式是：

- **只能由 owning worker 直接修改**
- 跨 worker 必须走 inbox / chan wake queue / handoff 协议

## 4. coroutine/task/channel 三层职责划分

### 4.1 coroutine 是执行体，不是稳定 handle

`XrCoroutine` 持有：

- `vm_ctx`
- `jit_ctx`
- `coro_gc`
- entry/args
- blocked/select/suspend state
- `task` back-pointer

同时它又可能被 pool 回收、复用。

所以它更像：

- executor frame carrier
- worker scheduling unit

而不是给用户长期引用的语义对象。

### 4.2 task 是 await / completion / structured concurrency 的协议核心

`XrTask` 持有：

- `result`
- `error`
- `state`
- `parent/first_child/next_sibling`
- `links`
- `on_completion`
- `await_state/waiter/waiter_index`

这层已经不是简单 handle，而是：

- await 协议状态机
- task tree
- monitor channel 通知源
- linked/supervisor scope 的一部分执行契约

### 4.3 channel 是共享同步原语

`XrChannel` 持有：

- ring buffer
- sendq/recvq
- mutex
- timer channel state
- waiter worker mask
- dist hooks binding

这使 channel 不是 per-coro object，而是：

- shared coordination object
- wake routing root
- distributed channel hook adapter

## 5. deep-copy 才是 coroutine 边界的真实入口

### 5.1 `xr_deep_copy_to_coro()` 的协议

核心逻辑：

- 非指针：直接返回
- shared object：`xr_shared_incref(obj)` 后直接返回
- 非 deep-copy kind：直接返回
- 否则复制到目标 coroutine 的 Immix heap；没有 `dst_coro_gc` 时退回 isolate gc

所以 `src/coro/` 真正维护的是一套对象跨域规则，而不是只管“排队执行”。

### 5.2 当前 deep-copy 覆盖的对象

`xr_deep_copy_with_ctx()` 当前显式覆盖：

- string：直接返回
- array
- map
- closure
- set
- instance
- json
- datetime

值得注意的点：

- `XR_TINSTANCE` 明确会 deep-copy
- 这与上一轮发现的“normal instance 实际落 fixedgc”形成交叉张力
- `task/channel` 不在 deep-copy switch 里，更多依赖 shared / non-deep 语义

### 5.3 deep-copy 输出堆域并不总是 per-coro

`copy_ctx_alloc()`：

- 有 `dst_coro_gc` → `xr_coro_gc_newobj(...)`
- 否则 → fixedgc (`xr_isolate_get_gc(X)`) 路径

所以 deep-copy 不是简单“复制到目标协程私有堆”，而是：

- 有目标 coro_gc：复制到目标 coro heap
- 无目标 coro_gc：复制到 isolate fixedgc

### 5.4 shared copy 支持另一条路径

`xdeep_copy.c` 还存在显式 shared copy：

- array/map/set/instance/json/stringbuilder/closure/datetime
- 都走 `xr_sysheap_alloc_shared(...)`

这证明 `src/coro/` 同时负责三类目标域：

- per-coro heap
- isolate fixedgc
- shared sysheap

## 6. await / wake / completion 协议

### 6.1 `xr_coro_ready()` 是最基础的 BLOCKED -> READY CAS 协议

它做了三件事：

- CAS 抢占 wake ownership，防 double wake
- `next=true` 且本地 worker 存在时走 `xr_worker_push_lifo()`
- 否则按 `affinity_p` 送到目标 worker inbox

这条路径是整个 coro 层“统一准备就绪协议”的核心。

### 6.2 `xr_task_wake_waiter()` 已经是 await 真正的协调中心

它先：

- `await_state = RESOLVED`
- 原子交换拿走 `task->waiter`
- 根据 `waiter_index` 区分：
  - 单 await
  - scope waiter
  - await.any
  - await.anySuccess
  - await.all

然后在必要时调用 `xr_coro_ready()` 唤醒 waiter。

说明 await 协议已经从 coroutine 迁移到了 task 层。

### 6.3 `xr_task_fire_completion()` 是 monitor/link/onComplete 的扇出点

completion listener 支持：

- wake waiter coro
- 发送 task 到 monitor channel
- closure callback（占位）

并且会在 fire 后释放 listener node。

这说明 task 不是只给 `await` 用，而是所有 completion 订阅的聚合点。

## 7. structured concurrency 与 child list owner

### 7.1 task child list 有锁，但仍是共享链表

`xr_task_attach_child()` / `detach_child()` / `child_completed()`：

- 都围绕 `parent->first_child`
- 用 `_Atomic bool child_lock` 自旋锁保护

这意味着：

- task child list 是 parent-owned mutable shared list
- 允许不同 worker 上的 child 完成后并发触达同一 parent

### 7.2 scope child list 是第二套平行结构

`XrScopeContext` 里还有：

- `first_child`
- `child_lock`
- 每个 child coroutine 上的 `scope_sibling`

`xr_coro_wake_waiter()` 会：

- 从 scope child list 中摘掉当前完成的 coroutine
- linked scope 下在首次失败时遍历 sibling 并取消它们
- supervisor scope 下收集 errors

因此当前 structured concurrency 实际有两套并行树：

- task tree
- scope coroutine list

它们各自解决不同层级的问题，但也增加了状态同步复杂度。

### 7.3 linked scope 取消路径是本轮最敏感的同步区

`xr_coro_wake_waiter()` 在持有 scope lock 时：

- 修改 `scope->first_child`
- 迭代 sibling
- 触发取消和 waiter 递减相关逻辑

这是典型的 cross-worker completion 热点，稍有不慎就会出现：

- list reuse
- double decrement
- sibling iteration 与删除竞态

## 8. channel wake routing 的真实边界

### 8.1 channel 内部队列不是最终 owner，worker blocked queue 才是 wake 目标入口

`XrChannel` 自己维护 `sendq/recvq`，但 worker 侧还有：

- `blocked_head/blocked_tail`
- `blocked_buckets[channel]`

`xr_worker_wake_one()` / `wake_all()` 只允许 owning worker 调用，并用：

- bucket 队列定位 waiter
- 从 worker 线性 blocked list 摘除
- 设置 `resume_status`
- 再 push 回本 worker runq/lifo

这说明真正的 wake owner 是：

- worker-private blocked queue
- channel 只是共享索引和同步点

### 8.2 cross-worker wake 必须走 command queue / inbox，而不是直接操作对方队列

`xworker.h` 明确给了：

- `xr_worker_inbox_enqueue()`
- `xr_worker_dispatch_chan_wake()`
- `xr_worker_drain_chan_wake_queue()`

这意味着远端唤醒的合法协议是：

- 发送命令
- 由目标 worker 自己 drain 并操作本地 blocked queue

一旦绕过这条路径，owner 就错了。

## 9. 正向发现

### 9.1 coroutine / task 分层是对的

当前设计至少把：

- 可 recycle executor
- 可 await / 可监听的 stable task handle

分开了，这比把所有状态都堆在 `XrCoroutine` 上清晰得多。

### 9.2 `xr_coro_ready()` 的 CAS wake 协议比较稳

它把：

- double wake
- 本地优先
- 远端 inbox
- spinner 唤醒

统一到了一个入口，利于后续分析和收敛。

### 9.3 channel 明确采用 shared object 模型

这和 object/class 层相比反而更清晰：

- 分配路径单一
- refcount 单一
- shared 语义单一

### 9.4 deep-copy 显式编码了跨域规则

不管实现细节是否仍有缺口，至少这里已经有一个清楚的决策点：

- 什么原样传
- 什么 incref
- 什么 deep-copy
- copy 到哪个 heap

## 10. 已确认高风险点

### 10.1 `XrCoroutine` 本体与其 `coro_gc` owner 分裂

外壳来自：

- pool/sysheap/malloc

对象堆来自：

- per-coro GC

这让“回收 executor”与“仍有 task/await/monitor 引用”的关系必须非常谨慎，否则容易出现 stale pointer。

### 10.2 task child list / scope child list 是两套并行可变链表

当前同时存在：

- `task->first_child`
- `scope->first_child`

两者都可能在不同 worker completion 路径里被并发修改。

### 10.3 `xr_coro_wake_waiter()` 责任过重

它同时处理：

- scope count decrement
- linked/supervisor error logic
- scope child list removal
- sibling cancel
- 再委托 task wake

这是一个明显的 cross-layer 汇合点，未来很容易继续膨胀。

### 10.4 deep-copy 与 fixedgc/shared object 的边界仍有概念张力

尤其是 instance：

- 平时 `xr_instance_new()` 落 fixedgc
- deep-copy to coro 时又可落 dst_coro_gc
- deep-copy to shared 时又可落 sysheap shared

这表示 object owner 在 coro 层被重新解释，说明上层“instance 属于哪个堆域”的叙述还不稳定。

### 10.5 task completion listener 链表没有独立同步原语

`xr_task_add_completion()` 只是简单头插：

- `node->next = task->on_completion`
- `task->on_completion = node`

而 `xr_task_fire_completion()` 会取走整个链表并释放。

如果未来出现更强的跨 worker 并发 listener 注册，这里会变成风险点。

### 10.6 deep-copy 对某些类型依赖“shared 直接 passthrough”，owner 认知必须一致

`XR_GC_IS_SHARED(obj)` 会直接 incref 返回。

如果某个类型在上层被误标 shared 或 shared 语义不闭合，就会直接跨 coroutine 传播，不再走复制防线。

## 11. 改进建议

### 11.1 给 coroutine / task / scope 各自写清 authoritative owner

建议明确三类对象：

- executor owner：runtime/worker/sysheap/pool
- task owner：executor GC handle
- scope owner：parent coroutine / task / runtime 哪一层

### 11.2 收敛 structured concurrency 到一套 authoritative child model

现在 task tree 与 scope child list 并存，虽然各有用途，但需要明确：

- 哪个是 lifecycle 真相源
- 哪个只是调度辅助视图

### 11.3 把 `xr_coro_wake_waiter()` 拆成更窄的职责块

至少可以拆成：

- scope bookkeeping
- linked/supervisor policy
- task await wake

降低 cross-layer 粘连。

### 11.4 明确 deep-copy 的目标域规范

建议把当前隐含规则文档化成固定矩阵：

- source type
- copy kind
- source storage(shared/fixedgc/coro)
- target storage(coro/fixedgc/shared)

### 11.5 对 completion/listener 注册补 owner 说明

需要明确：

- listener node 由谁分配
- 谁能并发注册
- fire 后谁释放
- 与 monitor channel 的 happens-before 保证在哪里

## 12. 给最后横切复盘的输入

下一轮 `030` 需要把以下东西串起来：

- fixedgc / per-coro / shared 三域是否真的是统一模型
- class/object/coro 三层对 instance/shared/reflect/task 的 owner 描述是否一致
- deep-copy / barrier / external memory accounting 是否在所有跨域对象上闭合
- 哪些类型只是“看起来有 GC header”，其实属于独立服务系统

---

## 13. 本轮状态

- **已完成**
  - coroutine/task/channel/runtime/worker owner 梳理
  - deep-copy 目标域与 shared passthrough 规则确认
  - await/completion/scope/child list 交叉路径确认
  - cross-worker wake routing 的 owner 边界确认

- **未完成**
  - 还没把 timer wheel / select case / netpoll resume 全部逐段对齐
  - 还没对 `xr_runtime_wake_channel()` / `wake_channel_all()` 的每个具体路径做完整复核
  - 还没把 coro 层和 DAP/debug stop 语义并起来

- **下一步**
  - 进入 `030` 横切复盘
