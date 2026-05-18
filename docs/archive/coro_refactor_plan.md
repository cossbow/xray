# `src/coro` 模块重构规划

> **目标**：把 `src/coro/` 50 个文件、16 839 行代码调整到"代码简洁 / 性能最优 / 设计一致"的状态。
>
> **原则**：不考虑向后兼容性 —— 直接采用最佳设计；旧接口可以删、可以改名；不做兼容层。
>
> **规模基线**：`xworker.c` 3 167 行（超 3 000 行硬线），`xcoro.c` 1 422 行，`xtimer_wheel.c` 992 行，`xnetpoll.c` 895 行。
>
> **诊断来源**：参阅根提交审计笔记（`git log -- src/coro`）及最近一次静态分析。下文关键判断均带文件行号引用。

## 总体原则

1. **每阶段独立可回滚**：单阶段内不跨多个架构面；每阶段结束必通过 `ctest --output-on-failure` + `scripts/run_regression_tests.sh`。
2. **硬性规则对齐**：
   - C 文件 ≤ 3 000 行 / H 文件 ≤ 800 行 / 单函数 ≤ 150 行
   - 禁止直接 `malloc/free`（统一 `xr_malloc/xr_free`）
   - 不需外部调用的函数必须 `static`
   - 禁止文件作用域的可变全局变量
   - GC / 调度锁互相独立
3. **性能 KPI**（每阶段给出指标基线 + 目标）：
   - `pingpong`（两 coro 互发 channel）吞吐
   - `spawn1M`（百万协程创建 + join）耗时
   - `chan_fanout`（1 sender → N receivers）平均唤醒延迟
   - `io_echo`（1k 客户端 echo）QPS
4. **可测量验收**：每阶段结束更新 `docs/engineering/coro_refactor_plan.md`，勾选复选框并记录实测数字。

## 阶段总览

| # | 阶段 | 目标 | 风险 | 工时 | 主要文件 |
|---|------|------|------|------|----------|
| 0 | 基础设施：常量集中 + 真正 spinlock | 消除魔数，落实命名一致 | 低 | 0.5 d | `xconstants.h`, `xchannel.h`, `base/xmutex.h` |
| 1 | `xworker.c` 拆分 | 合规（≤3000 行）+ 职责清晰 | 中 | 1 d | 新建 6 个 `xworker_*.c` |
| 2 | 正确性 bug 修复 | UB / 泄漏 / 命名违规清零 | 低 | 0.5 d | `xnetpoll.c`, `xworker.c`, `xbalance.h` |
| 3 | 死代码 + 全局变量 + 命名 | 清理 `XrScheduler` / `dist_hooks` / `g_counters` | 低 | 0.5 d | `xcoroutine.h`, `xchannel.c`, `xyield_closure.c` |
| 4 | 数据结构 lock-free 化 | 消除 `sched_lock` / `pool_lock` 争用 | 中 | 1 d | `xworker.c`(split 后), `xcoro_pool.c`, `xasync.c` |
| 5 | 巨函数拆分 | `xr_coro_run_on_worker` / `worker_loop` / `run_cfunc_coro` 都 ≤150 行 | 中 | 1 d | `xworker_exec.c`, `xworker_sched.c` |
| 6 | 热路径算法优化 | `select` O(N) → O(waiters)，channel helper 去重 | 中 | 1 d | `xworker_blocked.c`, `xchannel.c` |
| 7 | 健壮性与扩展 | 背压 / 迁移 deadline / seen hash 扩容 / 忙循环改 futex / Windows netpoll | 中 | 1.5 d | `xnetpoll*.c`, `xdeep_copy.c`, `xworker.c` |

**总计估算**：约 7 个工作日（不含性能调优和 review）。

---

## 阶段 0：基础设施（常量集中 + 诚实命名）

### 目标
把散落在 coro 模块各处的调优常量收拢；删除误导性的 `XrSpinlock` 别名，直接用 `XrMutex`（诚实命名）。

### 动机
- 魔数满天飞：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:518` `fast_dispatch_budget = 64`、`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:934` `XR_MAX_LIFO_POLLS 3`、`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:1416` `XR_SPIN_COUNT 20`、`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:1732` worker 栈 `8*1024*1024`、`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:2524` `XR_CORO_BATCH_SIZE 32`、`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:2595` `XR_ARENA_BATCH_SIZE 64`。部分还以 `#define` 写在函数体内，语法不规范。
- 伪 spinlock：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xchannel.h:72-75` 把 `XrSpinlock` `typedef` 成 `XrMutex`。注释写 "lock held for short durations only" 暗示是 spinlock，但底层其实是 futex-backed mutex，命名误导。

### 设计决策修订（2026-04-19）

**不引入真 spinlock**。理由：
- `src/base/xmutex.h` 已是 3 态自适应锁（active spin 4 × 30 pause → passive spin 1 × yield → futex sleep）。无竞争时就是一次 CAS，性能上已逼近真 spinlock。
- `xr_channel_close` 路径持锁遍历 waitq 构造 recv/send list，持锁时间线性于 waiter 数；真 spinlock 在此场景下会让所有等待线程烧 CPU，弊大于利。
- registry 的 `find_slot + insert/grow` 临界区也不够短。

**最佳设计**：**直接删除 `XrSpinlock` 别名**，channel / registry 结构体字段改为 `XrMutex`，所有调用点改 `xr_mutex_*`。诚实命名，不引入新概念。

### 步骤

1. **新增 `src/coro/xcoro_tuning.h`**
   - 所有 coro 调优常量集中定义
   - 每个常量附上：单位、默认值、影响维度、（如需）环境变量 override 名
   - 例：
     ```
     #define XR_FAST_DISPATCH_BUDGET   64   // exec_with_cont_stealing 连续快派上限
     #define XR_MAX_LIFO_POLLS          3   // LIFO slot 连续命中上限
     #define XR_SPIN_COUNT             20   // worker_loop 进入 park 前自旋轮数
     #define XR_WORKER_STACK_BYTES     (8*1024*1024)
     #define XR_CORO_BATCH_SIZE        32   // 从全局 free list 批量偷取
     #define XR_ARENA_BATCH_SIZE       64   // 每 worker 从 arena 批量申请
     ```
   - `xworker.c` / `xworker_*` 全部改用。
2. **删除 `XrSpinlock` 别名，统一用 `XrMutex`**
   - `@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xchannel.h:72-75` 删掉 `typedef XrMutex XrSpinlock` + 三个 `#define`
   - `XrChannel::lock` 字段类型改为 `XrMutex`
   - `XrCoroRegistry::lock` 字段类型改为 `XrMutex`
   - 所有 `xr_spinlock_init/lock/unlock` 改为 `xr_mutex_init/lock/unlock`（全局 replace）
   - 修正 `XrChannel` invariant 5 的注释："channel spinlock" → "channel mutex"

### 验证
- `/build`
- `/test`
- `/build-asan` + `/test`
- 常量集中不影响性能基线；mutex 改名是纯文字替换，零语义变化。

### 风险
- 低。全是重命名 + 集中化，无运行时行为变化。
- 回滚：单个 commit 即可 revert。

---

## 阶段 1：`xworker.c` 拆分

### 目标
把 3 167 行的 `xworker.c` 拆成 6-7 个职责单一的文件，每个 ≤ 800 行；同时确保每个函数 ≤ 150 行（超出的留给阶段 5）。

### 目标布局
```
src/coro/
├── xworker.c               ~300 行：仅 runtime_create/destroy/start/stop/spawn
├── xworker.h
├── xworker_internal.h      （已存在）
├── xworker_runq.c          ~280 行：runq / LIFO / push / pop
├── xworker_sched.c         ~900 行：worker_loop + park/unpark + spinning + stealing
├── xworker_exec.c          ~700 行：xr_coro_run_on_worker + run_cfunc_coro + worker_exec_with_cont_stealing
├── xworker_pool.c          ~200 行：xr_coro_pool_get/put + arena cache + recycle
├── xworker_blocked.c       ~350 行：blocked bucket hash + worker_block/unblock + wake_one/all + dequeue_blocked
├── xworker_handoff.c       ~250 行：entersyscall / exitsyscall / handoff_thread_entry
├── xworker_sysmon.c        （已存在，保持不动）
└── xworker_internal.h      （扩展：把跨 .c 的 static→XR_FUNC 函数声明集中）
```

### 步骤

1. **准备 `xworker_internal.h`**
   - 把下列内部函数声明迁入（当前是 `xworker.c` 内部或无前向声明导致的隐式外部）：
     - `void worker_unpark(XrWorker *);`
     - `void wake_idle_worker(XrRuntime *);`
     - `bool worker_blocked_list_remove(XrWorker *, XrCoroutine *);`
     - `XrBlockedBucket *worker_blocked_bucket_find_or_create(XrWorker *, void *);`
     - `XrBlockedBucket *worker_blocked_bucket_find(XrWorker *, void *);`
     - `void worker_blocked_list_add(XrWorker *, XrCoroutine *);`
     - `void worker_drain_inbox(XrWorker *);`
     - `void worker_poll_sources(XrWorker *);`
     - `bool worker_process_blocked(XrWorker *, XrCoroutine *);`
     - `void worker_exec_with_cont_stealing(XrWorker *, XrCoroutine *);`
     - `void *worker_loop(void *arg);`
     - `void idle_worker_push/pop/remove(XrRuntime *, int)`（或改为 lock-free 后移到阶段 4 再处理）
   - 全部 `XR_FUNC`；加注释 "internal across xworker_*.c"。
2. **拆分执行顺序**（按依赖自底向上）
   1. **`xworker_runq.c`**（无其他 xworker_*.c 依赖）
      - 移入：`xorshift32`、`xr_runq_init/destroy/enqueue/dequeue/steal`、`xr_worker_push/push_lifo/pop`。
      - 保留在原文件的函数：无。
   2. **`xworker_blocked.c`**
      - 移入：`blocked_bucket_hash`、`worker_blocked_bucket_find*`、`worker_blocked_list_add/remove`、`xr_worker_block/unblock/wake_one/wake_all/dequeue_blocked`。
      - 与 `xworker_sysmon.c` 的 `xr_worker_block_select/wake_select/unblock_select` 保持分离（select 走独立路径）。
   3. **`xworker_pool.c`**
      - 移入：`xr_coro_pool_get`、`xr_coro_pool_put`、arena cache 相关、deferred recycle flush。
      - 常量 `XR_CORO_BATCH_SIZE` / `XR_ARENA_BATCH_SIZE` 搬到 `xcoro_tuning.h`。
   4. **`xworker_handoff.c`**
      - 移入：`xr_worker_entersyscall`、`xr_worker_exitsyscall`、`xr_handoff_thread_entry`。
      - 引用 `wake_idle_worker` / `worker_exec_with_cont_stealing` 通过 internal.h。
   5. **`xworker_exec.c`**
      - 移入：`xr_coro_run_on_worker`、`run_cfunc_coro`、`worker_exec_with_cont_stealing`、`worker_process_blocked`、`worker_blocked_post_check`、`worker_handle_vm_result`。
      - 巨函数留到阶段 5 再拆。
   6. **`xworker_sched.c`**
      - 移入：`worker_loop`、`worker_park`、`worker_unpark`、`wake_idle_worker`、`idle_worker_push/pop/remove`、`try_immigrate`、`runtime_has_work`、`xr_worker_inbox_enqueue`、`worker_poll_sources`、`worker_drain_inbox`、`worker_sleep_timeout_callback`、`xr_worker_add_sleep_timer`、`xr_worker_cancel_timer`。
   7. **`xworker.c` 精简**
      - 仅保留：`xr_runtime_create/destroy/start/stop/ensure_workers`、`xr_runtime_spawn/spawn_local`、`xr_worker_init/destroy`、`xr_runtime_print_stats`、`get_current_time_us`、TLS 定义。
3. **CMakeLists.txt 更新**
   - 在 coro 模块 source 列表加入 6 个新 `.c`。
4. **命名 / 可见性一次性整改**
   - `worker_blocked_list_remove` / `sysmon_thread_func` 等当前**无修饰符**的非 static 函数：凡跨文件使用统一加 `XR_FUNC`（并声明到 internal.h）；仅本文件使用的改为 `static`。参照规则：
     ```
     @/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:44-45
     bool worker_blocked_list_remove(XrWorker *worker, XrCoroutine *coro);
     void *sysmon_thread_func(void *arg);
     ```
     这两行违反 "非 static 函数必须带 `XRAY_API` / `XR_FUNC`" 硬线。

### 验证
- 每拆一个子文件独立 commit，commit 后 `/build` + `/test`。
- 拆完后：
  - `wc -l src/coro/xworker*.c` 每个 ≤ 800 行
  - `grep -nE '^[A-Za-z_][A-Za-z0-9_ *]+\(' src/coro/xworker*.c | grep -v 'static\|XR_FUNC\|XRAY_API'` 应为空。

### 风险
- 拆分引入的 link error 最常见（前向声明缺失、循环依赖）。先在 `xworker_internal.h` 一次性写齐声明，再分步搬移函数。
- 回滚：每一个子拆分独立 commit，`git revert` 粒度最小。

---

## 阶段 2：正确性 bug 修复

### 清单（严重度从高到低）

#### 2.1 `xr_poll_cache_alloc` 对已初始化 mutex 重复 `pthread_mutex_init`
**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xnetpoll.c:60-79`

**问题**：从 Treiber stack 弹出的 `XrPollDesc` 在上次 free 时没有 `pthread_mutex_destroy`，这里又调用 `pthread_mutex_init`。POSIX 明确规定对已初始化 mutex 再 init 是 **UB**。

**修复策略**（采用 "永不 destroy，只 init 一次" 模式）：
```c
XrPollDesc *pd;
bool is_new = false;
do {
    pd = atomic_load_explicit(&cache->head, memory_order_acquire);
    if (!pd) {
        pd = (XrPollDesc *)xr_calloc(1, sizeof(XrPollDesc));
        if (!pd) return NULL;
        pthread_mutex_init(&pd->block_mu, NULL);   // <-- 仅新分配时 init
        pthread_cond_init(&pd->block_cond, NULL);
        is_new = true;
        break;
    }
} while (!atomic_compare_exchange_weak_explicit(
    &cache->head, &pd, pd->link,
    memory_order_acq_rel, memory_order_acquire));

// Reset 除 mutex/cond 外的所有字段
pd->link = NULL;
pd->fd = -1;
atomic_store(&pd->fdseq, 0);
/* ... 其余字段 reset，略 ... */
```
同时：`xr_poll_cache_cleanup` 在释放前需要 `pthread_mutex_destroy`/`pthread_cond_destroy`。

#### 2.2 `XrBlockedBucket` 只分配不释放
**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:2708-2717`

**修复**：在 `xr_worker_wake_one` / `xr_worker_wake_all` / `xr_worker_unblock` 路径上，当 bucket 三条队列全空时从 hash 桶链里摘除并 `xr_free`。
- 注意 bucket 摘除仅能在 owner worker 线程执行（hash 桶本身就是 per-worker，天然安全）。

#### 2.3 `worker_blocked_list_remove` / `sysmon_thread_func` 可见性违规
**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:44-45`

**修复**：阶段 1 拆分过程中一次性处理（加 `XR_FUNC` 并声明到 `xworker_internal.h`）。

#### 2.4 `XrBalanceInfo` 重复定义
**位置**：
- `@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xbalance.h:68-74`（定义 `XrBalanceInfo`）
- `@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.h:91-96`（在 `XrRuntime` 里又写了一遍匿名 struct）

**修复**：`XrRuntime` 直接 `XrBalanceInfo balance_info;`；删除匿名 struct。

#### 2.5 `xr_runq_steal` dst 溢出路径潜在链交叉
**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:109-116`

**检查**：被偷走的 coro 此时是否一定不在任何 overflow 链上？submit_time 路径使得它是 deque 里的，但 `sched_link` 同名字段被 overflow 和 MPSC inbox 复用。
**修复**：加 `XR_DCHECK(c->sched_link == NULL, "stolen coro sched_link must be NULL")` 断言；如果触发再深入分析。

#### 2.6 `xr_coro_pool_init` 错误路径泄漏检查
**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xcoro_pool.c:94-134`

**问题**：`pthread_mutex_init` 失败后 `pool->blocks` 虽被释放，但 `pool->current_block` 仍指向已 free 的块。虽然 `xr_coro_pool_destroy` 有 `initialized` 保护，不会二次 free；但应把指针置 NULL。
**修复**：释放后统一 `pool->blocks = NULL; pool->current_block = NULL;`。

### 验证
- `/build-asan` + `/test`（特别是 netpoll / channel / task 相关测试）
- Valgrind（如环境可用）跑 `benchmarks/chan_fanout.xr` 看是否仍有 bucket 泄漏
- `scripts/check_architecture.sh` 的 warning 数应减少

### 风险
- bucket 回收要小心：正在遍历 hash 桶链的地方若删除，需要保留 `next` 指针。这类 bug 通过 ASan 能即时捕获。

---

## 阶段 3：清理死代码 + 全局变量 + 命名一致性

### 3.1 删除 `XrScheduler` 多数字段
**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xcoroutine.h:454-463`

GMP 改造后 `ready_head/tail/current/coro_count/next_id/total_created` 已无意义，实际唯一活字段是 `coro_registry`。

**重构**：
- 改名为 `XrCoroState`（或 `XrIsolateCoroState`），只保留 `XrScopeContext *current_scope;` 和 `XrCoroRegistry *coro_registry;`
- 所有引用 `XrScheduler *` 的位置（主要在 `xworker.c` 里形如 `(XrScheduler *)runtime->isolate->vm.scheduler`）统一改名
- `isolate` 侧的字段名也对齐（vm.scheduler → coro_state）

### 3.2 `xr_channel_dist_hooks` 全局化违规
**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xchannel.c:42`
```c
XrChannelDistHooks *xr_channel_dist_hooks = NULL;
```
违反"禁止文件作用域的可变全局变量"。

**修复**：
- 迁移到 `XrayIsolate` 结构：`XrChannelDistHooks *dist_hooks;`
- 所有 `xr_channel_*` 函数签名增加 `XrChannel *ch` → 从 `ch->isolate->dist_hooks`（需要给 `XrChannel` 添加 `XrayIsolate *isolate` 字段）取 hooks。
- 如果 `XrChannel` 已存 isolate 指针则直接复用。

### 3.3 `g_channel_close_count` 全局计数
**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xchannel.c:39`

**修复**：挪到 `XrSystemHeap::stats.channel_close_count`（与 `channel_create_count` 对齐，`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xchannel.c:183` 已有前例）。

### 3.4 `xyield_closure.c` 并入 `xyieldable.c`
131 行的独立文件没有单独存在的价值。并入后 `xyieldable.c` ≈ 380 行，仍 ≤800 行。

### 3.5 `run_cfunc_coro` / `xr_coro_run_on_worker` 重复的状态切换三连
反复出现的片段：
```c
atomic_compare_exchange_strong_explicit(&coro->coro_state, RUNNING, BLOCKED, ...);
atomic_fetch_and_explicit(&coro->flags, ~RUNNING, ...);
atomic_fetch_or_explicit(&coro->flags, BLOCKED, ...);
```
**修复**：在 `xcoro_flags.h` 中添加 inline helper：
```c
static inline bool xr_coro_transition_to_blocked(XrCoroutine *coro);
static inline void xr_coro_transition_to_running(XrCoroutine *coro);
static inline void xr_coro_transition_to_ready(XrCoroutine *coro);
static inline void xr_coro_transition_to_done(XrCoroutine *coro);
```

### 3.6 `xr_coro_resume_with_unroll` 的调用位点收敛
目前 `run_cfunc_coro` 的 inline fast path 与 slow path 有重复的 switch/case 处理（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:1982-2038`）。可合并到一个 helper。

### 验证
- `/build`
- `/test`
- `grep -rn 'XrScheduler\|g_channel_close_count\|xr_channel_dist_hooks' src/ | wc -l` 应 = 0
- `grep -n 'static XrChannelDistHooks\|static.*_Atomic' src/coro/` 仅返回本阶段允许保留的（如 TLS）。

### 风险
- `XrScheduler` 改名涉及 vm/、app/ 多处，用 `sed -i '' 's/XrScheduler/XrCoroState/g'` + 手动 review 可快速完成。

---

## 阶段 4：数据结构 lock-free 化

### 4.1 `sched_lock` 改造（最高价值）

**现状**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.h:55` `pthread_mutex_t sched_lock` 保护三份数据：
- `idle_worker_stack[]` + `idle_worker_count`
- `idle_p_head`（XrProc 链表）
- `idle_m_head`（XrMachine 链表）

被以下路径频繁锁：
- `idle_worker_push/pop/remove`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:130-158`）
- `worker_poll_sources` 里仅为读 `idle_worker_count`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:302-310`）
- `xr_get_idle_m/xr_put_idle_m`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xmachine.c:113-137`）
- `xr_get_idle_p/xr_put_idle_p`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xproc.c:190-215`）

**重构方向**：三份数据独立化，全部 lock-free：

#### (a) `idle_worker_stack` → Treiber stack
- 字段改名：`idle_worker_list_head`（`_Atomic(XrMachine *)`）
- 复用 `XrMachine::idle_link` 作链节点
- `worker_park` 路径 push；被唤醒路径 pop 后不需要 "remove specific worker"，因为：
  - 原 `idle_worker_remove` 的唯一用途是 "worker 自唤醒后把自己从 stack 移走" —— 其实不需要，下次 `wake_idle_worker` pop 到已唤醒的 worker 时检查 `m->state != M_PARKED/M_PARKING` 跳过即可（带重试上限）。
- 伪代码：
  ```c
  static void idle_worker_push_lf(XrRuntime *rt, XrMachine *m) {
      XrMachine *head;
      do {
          head = atomic_load_explicit(&rt->idle_worker_list, memory_order_relaxed);
          m->idle_link = head;
      } while (!atomic_compare_exchange_weak_explicit(
          &rt->idle_worker_list, &head, m,
          memory_order_release, memory_order_relaxed));
  }
  static XrMachine *idle_worker_pop_lf(XrRuntime *rt) {
      for (int retry = 0; retry < 4; retry++) {
          XrMachine *head = atomic_load_explicit(&rt->idle_worker_list, memory_order_acquire);
          if (!head) return NULL;
          if (atomic_compare_exchange_weak_explicit(
                  &rt->idle_worker_list, &head, head->idle_link,
                  memory_order_acq_rel, memory_order_acquire)) {
              head->idle_link = NULL;
              return head;
          }
      }
      return NULL;
  }
  ```
- `idle_worker_count` 保留为 `_Atomic int` 供 `worker_poll_sources` 非精确读取（EWMA/启发式用途）。

#### (b) `idle_p_head` / `idle_m_head` 同步改 Treiber stack
- 字段从 `XrProc *` / `XrMachine *` 改为 `_Atomic(XrProc *)` / `_Atomic(XrMachine *)`
- `xr_get_idle_p/m` `xr_put_idle_p/m` 改为 lock-free 循环

#### (c) ABA 风险评估
- **worker stack**：worker 生命周期 = runtime 生命周期，永不销毁，不存在 A→free→A 的 ABA；且 pop 后立即使用，重入间隔很短。
- **P / M 链表**：P 是 pre-allocated 1:1 with worker，永不销毁；M 会动态增长（handoff），但不会 free。ABA 问题不存在。

#### (d) 移除 `sched_lock`
改造完三项后 `sched_lock` 可以删除；`XrRuntime` 字段减少一个 pthread mutex。

### 4.2 协程池锁 → Treiber stack

**现状**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xcoro_pool.c:72-80`
```c
pthread_mutex_t free_lock;    // 保护 free_list
pthread_mutex_t pool_lock;    // 保护 grow
```

**重构**：
- `free_list` → `_Atomic(XrCoroutine *)`，lock-free Treiber stack
  - push：CAS head
  - 批量窃取（`xworker_pool.c` 里从全局 free list 偷 32 个）：`atomic_exchange` 一次拿到整条链，本地切片取所需数量，剩余再 CAS push 回去
- `grow` 仍用 mutex（低频路径，扩容本身内含 `xr_malloc`），但改名 `grow_lock` 避免误解
- free list 中断链节点复用 `coro->next`（现有设计）

### 4.3 `async_pool` 任务队列 → MPSC

**现状**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xasync.c:74-113` 全局 mutex + cond_wait/signal 推任务队列。

**重构**：
- 使用现有 `xmpsc_queue.h`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xmpsc_queue.h`），lock-free push、owner pop
- 唤醒机制改为 futex（`xr_park_futex_wait/wake`，已在 `xmachine.h` 有跨平台封装）
- 异步线程数 = `XR_ASYNC_THREAD_COUNT`（已是常量），若改成多消费者，需要带 steal

### 4.4 统计字段 cache line 对齐

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xproc.h:155-162`
```c
uint64_t executed_count;   // owner 写
uint64_t stolen_count;     // owner 写，sysmon 读
uint64_t yielded_count;    // owner 写
uint64_t cont_steal_count; // stealer 写，owner 读 ← 关键！
uint64_t completed_count;
uint64_t spawned_count;
```
`cont_steal_count` 被偷取者写、被偷者读，与其他 owner-only 字段同 cache line → 伪共享。

**修复**：把 stats 块用 `__attribute__((aligned(XR_CACHE_LINE)))` 独立出一个 sub-struct：
```c
typedef struct XrProcStats {
    uint64_t executed_count;
    uint64_t stolen_count;
    uint64_t yielded_count;
    uint64_t completed_count;
    uint64_t spawned_count;
    /* hot cross-worker: put last */
    uint64_t cont_steal_count;
} XrProcStats __attribute__((aligned(64)));
```
XrProc 里 `XrProcStats stats;` 替换 6 个独立字段。

### 验证
- `/build` + `/test`
- `/build-asan` + `/test`（lock-free 代码必跑 ASan）
- TSan（如果项目已集成）
- 微基准：
  - `pingpong` 目标 +10~20%
  - `spawn1M` 目标 +5~15%

### 风险
- lock-free pop 最常见 bug：pop 后忘记清 `head->link`，下次被 push 时污染。每个 pop 点 `XR_DCHECK(old->idle_link == NULL || "just popped")`。
- 分阶段合并：4.1 / 4.2 / 4.3 每个单独 PR / commit，可独立回滚。

---

## 阶段 5：巨函数拆分

### 5.1 `xr_coro_run_on_worker` 拆分
**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:2076-2519`（约 440 行，超限 3 倍）

**当前结构**：
```
xr_coro_run_on_worker
├── channel resume fast path
├── cfunc 入口（first-run + resume 公用 run_cfunc_coro）
├── closure first exec fast path（label: first_exec_setup）
├── native entry
├── cfunc entry（second branch）
├── first exec slow path（goto first_exec_setup）
├── resume slow path
│   ├── JIT resume
│   ├── cont-stealing resume
│   ├── debug resume
│   └── unroll resume
└── handle_result（各分支汇聚）
```

**目标拆分**：
```
XrVMResult xr_coro_run_on_worker(worker, coro) {
    // 仅做 dispatch
    switch (coro->entry_type) {
        case XR_CORO_ENTRY_NATIVE: return run_native_coro(worker, coro);
        case XR_CORO_ENTRY_CFUNC:  return run_cfunc_coro(worker, coro);
        case XR_CORO_ENTRY_CLOSURE:
            if (!(flags & STARTED)) return run_closure_first(worker, coro);
            return run_closure_resume(worker, coro);
    }
}

static XrVMResult run_closure_first(worker, coro);    // ~80 行
static XrVMResult run_closure_resume(worker, coro);   // ~120 行
  ├── run_closure_resume_jit(worker, coro)            // ~50 行
  ├── run_closure_resume_channel(worker, coro)        // ~40 行
  └── run_closure_resume_unroll(worker, coro)         // ~40 行

static void finalize_coro_result(coro, result, ctx);  // ~60 行（handle_result 公共逻辑）
```

### 5.2 `worker_loop` 拆分
**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:1235-1511`（约 270 行）

**拆分**：
```
void worker_loop(worker) {  // ~70 行
    worker_setup_tls(worker);
    worker_setup_affinity(worker);
    while (running) worker_schedule_step(worker);
    worker_teardown(worker);
}

static XrCoroutine *worker_find_work(worker, poll_skip);  // ~60 行：local pop → poll → balance → steal → spin
static void worker_enter_spinning(worker);                // ~40 行
static void worker_schedule_step(worker);                 // ~60 行：主循环内一步
```

### 5.3 `run_cfunc_coro` 拆分
**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:1905-2068`（约 160 行）

**拆分**：
```
static XrVMResult run_cfunc_coro(worker, coro, isolate) {
    if (!(flags & STARTED)) return cfunc_first_call(worker, coro, isolate);
    return cfunc_resume(worker, coro, isolate);  // 内含 inline fast path + slow path
}
```

### 5.4 `xr_coro_pool_get` 拆分
**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:2527-2623`（约 95 行）

**拆分**：
```
XrCoroutine *xr_coro_pool_get(runtime) {
    worker_flush_pending_recycle(worker);
    if (worker->p.local_free_list) return pool_get_from_local(worker);
    if (pool_steal_batch(worker, pool)) return pool_get_from_local(worker);
    return pool_get_from_arena(worker, pool);  // 可能返回 NULL
}
```

### 验证
- `/build` + `/test`
- `find src/coro -name '*.c' -exec awk '/^[a-zA-Z_][a-zA-Z0-9_ *]+\([^;]*$/{f=$0; n=NR} /^}$/{if(f){print FILENAME":"n"-"NR": "f; f=""}}' {} \;` 找出 ≥150 行函数（应为空）
- 性能对比：拆分后由于 inline 边界变化，需确认 hot path 不退化（可能需要 `static inline` 标注）

### 风险
- 拆分导致 inline 失效 → 热路径退化。措施：
  - 对每 hot 小函数加 `static inline`
  - LTO 开启时问题较小
  - 最后一步做 `benchmarks/pingpong` 对比，若退化超 5% 定位到具体函数拆回 `__attribute__((always_inline))`

---

## 阶段 6：热路径算法优化

### 6.1 `select` 唤醒从 O(N) 降到 O(waiters_on_channel)

**现状**：
- `@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker_sysmon.c:373-403`：`xr_worker_block_select` 只把 coro 挂到 **第一个** channel 的 bucket select 队列。
- `@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker_sysmon.c:412-457`：`xr_worker_wake_select` 为找某 channel 上的 select waiter，**线性扫全 worker 的 blocked_head**。

**问题规模**：1000 个 coro 做 `select(M channels)` → 每次 channel 事件 O(1000) 扫描。

**重构**：
1. `XrSelectCase` 添加 `struct XrSelectCase *next_on_channel;` 字段（或通过 XrSelectWaitNode 节点分离）
2. `xr_worker_block_select` 对每个 case 都 push 到 `bucket->select_head`（通过 case 节点，而非 coro 本体）
3. `xr_worker_wake_select` 改为 "查 bucket → 遍历 bucket 的 select_head"，复杂度 = 该 channel 上 select waiters 数
4. 多 channel case 同一 coro：仍通过 `XrSelectWait::triggered` 的 CAS 避免重复唤醒
5. `xr_worker_unblock_select`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker_sysmon.c:465-516`）需从**所有** channel 的 select 队列摘除

**数据结构**：
```c
typedef struct XrSelectWaitNode {
    struct XrSelectWaitNode *next;  // bucket->select_head 链
    XrCoroutine *coro;              // 回指所属 coro
    int case_index;                  // 本节点对应 cases[] 的下标
} XrSelectWaitNode;
```
`XrSelectWait` 额外持有 `XrSelectWaitNode *nodes` 数组（大小 = case_count），池化分配避免频繁 malloc。

**测试**：`benchmarks/select_fanout.xr`（1 个 sender → M channels → N 个 `select` 接收者），对比 N=10/100/1000 时的吞吐。

### 6.2 channel send/recv 三路径去重

**现状**：`xr_channel_send` / `xr_channel_try_send` / `xr_channel_notify_send` 的 buffer push + 直接传递逻辑重复 3 次。

**重构**：抽 inline helpers（全部 **要求调用方持有 ch->lock**）：
```c
static inline bool chan_try_direct_send(XrChannel *ch, XrValue v);  // 若 recvq 非空，直接交付并 wake，返回 true
static inline bool chan_try_buffer_push(XrChannel *ch, XrValue v);  // 若 buffer 有空位，推入返回 true
static inline bool chan_try_direct_recv(XrChannel *ch, XrValue *out);
static inline bool chan_try_buffer_pop(XrChannel *ch, XrValue *out);
```
然后 `xr_channel_send` ≈ 15 行、`xr_channel_try_send` ≈ 15 行、`xr_channel_notify_send` ≈ 15 行。

### 6.3 channel close 大扇出分发

**现状**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xchannel.c:521-524` close 时所有 waiter 都挤到当前 worker LIFO → 串行瓶颈。

**重构**（已实现，简化版）：
```c
static void channel_wake_coro_ex(XrCoroutine *coro, bool is_close) {
    /* ... existing state transitions ... */
    int target = atomic_load(&coro->affinity_p);
    if (is_close && target != current) {
        xr_worker_inbox_enqueue(runtime, target_id, coro);  // cross-worker via inbox
        return;
    }
    xr_worker_push_lifo(current, coro);  // same-worker or normal wake
}
```
实际实现省略了阈值常量 `XR_CLOSE_WAKE_LIFO_THRESHOLD`——对于 close 扇出，跨 worker 的 waiter **一律**走 MPSC inbox，同 worker 的仍走 LIFO。inbox 单次入队开销约 1 个 CAS + 1 次 futex wake，远低于串行唤醒 N 个跨 worker coro 的代价，无需阈值判断。

### 6.4 `xr_channel_remove_waiter` 改 O(1)

**现状**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xchannel.c:80-104` 线性扫 waitq 找特定 coro。
**重构**：把 `XrWaitQueue` 从单链改为**双链**（`wait_link` 已存在，增加 `wait_prev`），remove 变 O(1)。
- `XrCoroutine` 结构已有足够字段，也可以在 `XrCoroutine` 内增加 `wait_prev`。
- 所有 enqueue/dequeue/remove 一次性改。

### 验证
- `/build-asan` + `/test`（select 改动涉及多线程协作，ASan 必跑）
- 新增 benchmark：
  - `benchmarks/select_1000.xr`
  - `benchmarks/chan_close_fanout.xr`
- 对比前后 QPS。

### 风险
- select 改造影响正确性最高：增加专用测试 `test/unit/test_select_fanout.c` 覆盖 1/10/100/1000 waiters 的唤醒次序。

---

## 阶段 7：健壮性与扩展

### 7.1 Runtime 忙循环改 futex

**位置 1**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:1714-1716`
```c
while (atomic_load(&runtime->started_workers) < expected_workers) {
    sched_yield();
}
```
**修复**：用 futex-based 计数等待：
```c
while (atomic_load_explicit(&runtime->started_workers, memory_order_acquire) < expected_workers) {
    xr_park_futex_wait(&runtime->started_workers, current_value, 1000 /* us */);
}
```
`worker_loop` 启动时在 `atomic_fetch_add` 后 `xr_park_futex_wake(&runtime->started_workers);`。

**位置 2**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:3040-3048` `exitsyscall` 自旋。
**修复**：带超时的 futex 等待；超时后降级到当前的 `nanosleep(100us)` 慢路径，并打 warning log。

### 7.2 `xr_netpoll_bind_worker` 支持迁移

**现状**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xnetpoll.c:807-823` 首次绑定后永久不变；coro 被偷走后 deadline 静默丢失。

**重构**：
- 当 `set_deadline` 发现 `current != owner`：
  - **选项 A**（简单）：先 `xr_timer_queue_cancel` 老 wheel 里的 timer，再把 pd 的 owner 改为 current，然后在 current 的 wheel 上 `xr_twheel_set_timer`
  - **选项 B**（保守）：仍跨 worker 不动 timer，但把此事实记录到 `pd->migration_count++`，sysmon 周期性根据该计数决定是否真正迁移
- 推荐选项 A 配合 "只有 rrun/wrun 都为 false 时才允许迁移" 的安全条件。

### 7.3 Deep copy seen hash 动态扩容

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xdeep_copy.c:47-51`
```c
#define SEEN_BUCKET_COUNT 32
```
**重构**：
- `XrCopyContext::bucket_count` 已存在，改为可增长字段
- 每次 record 时检查 count 是否 > buckets * 0.75，超过就 rehash 到 2x
- buckets 仍按需分配（首次 record 时才 calloc 32 bucket）
- 参考 `xr_coro_registry` 的 grow 实现

### 7.4 Runtime 背压

**现状**：
- `xr_runq_enqueue` 溢出到无界链（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:82-87`）
- MPSC inbox 无界（`total_inbox_len` 只是统计不限流）
- `async_pool` 队列无界

**重构**：新增配置字段
```c
typedef struct XrRuntimeLimits {
    int  max_runq_overflow;      // 每 worker 溢出链上限（默认 100_000）
    int  max_inbox_depth;        // 全局 inbox 上限（默认 1_000_000）
    int  max_async_queue;        // async 池任务上限（默认 10_000）
} XrRuntimeLimits;
```
- 超限行为：spawn 时返回 `XrSpawnError::BACKPRESSURE`，由语言侧暴露为抛异常
- 提供环境变量 `XRAY_RUNQ_LIMIT` / `XRAY_INBOX_LIMIT` / `XRAY_ASYNC_LIMIT` 在 runtime_create 时读取

### 7.5 Windows netpoll

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xnetpoll.c:235-239` `// TODO`

**选项**：
- **选项 A**：补 IOCP 唤醒（已有 `xnetpoll_iocp.c`，但未被 `netpoll_default_ops` 映射）。需要为 Windows 启用 `_WIN32` 分支 + 引入 `PostQueuedCompletionStatus` 作为 wakeup。
- **选项 B**（暂不支持 Windows）：直接 `#error` 明确标记不支持，等后续立项。

推荐现阶段走选项 B，把 TODO 清除。

### 7.6 `XrBlockedBucket` bucket 池化（非必须，锦上添花）
阶段 2 已处理释放；阶段 7 可进一步改为定长 pool 分配。

### 验证
- `/build` + `/test` + `/build-asan`
- 新增 stress 测试：
  - `test/stress/test_runq_overflow.c`：无限 spawn 直到命中背压
  - `test/stress/test_deep_copy_cyclic.c`：10k 对象环
  - `test/stress/test_fd_migrate.c`：跨 worker fd 迁移

### 风险
- 背压语义变更：一旦添加 spawn 失败语义，语言侧代码需处理。建议先以环境变量启用为 opt-in。

---

## 阶段尾：一致性收口

每阶段结束固定动作：
1. **跑完整回归**：`/test`（快速）+ `scripts/run_regression_tests.sh`（完整）
2. **跑 ASan**：`/build-asan` + `/test`
3. **检查硬线指标**：
   ```bash
   wc -l src/coro/*.c | awk '$1 > 3000'      # 应为空
   wc -l src/coro/*.h | awk '$1 > 800'       # 应为空
   ```
4. **函数行数检查**：借脚本把 ≥150 行函数列出来
5. **commit 规范**：每阶段一个或多个小 commit；commit message 英文（见 `main.md`）
6. **更新本文档**：把对应阶段的复选框打勾，记录实测 KPI

## KPI 基线记录表（待填）

| 基准测试 | 阶段 0 前 | 阶段 4 后 | 阶段 6 后 | 阶段 7 后 |
|----------|-----------|-----------|-----------|-----------|
| `pingpong` ops/s | _ | _ | _ | _ |
| `spawn1M` ms | _ | _ | _ | _ |
| `chan_fanout` μs/wake | _ | _ | _ | _ |
| `select_1000` ops/s | _ | _ | _ | _ |
| `io_echo_1k` QPS | _ | _ | _ | _ |
| `xworker*.c` 最大行数 | 3167 | _ | _ | _ |
| `sched_lock` 持有时长（perf lock report） | _ | _ | _ | _ |

## 执行进度追踪（复选框）

### 阶段 0：基础设施
- [x] 创建 `src/coro/xcoro_tuning.h`，搬移 7 个常量
- [x] 替换 `xworker.c` 内魔数为 `xcoro_tuning.h` 常量
- [x] 删除 `xchannel.h` 的 `XrSpinlock` typedef
- [x] `XrChannel::lock` / `XrCoroRegistry::lock` 改为 `XrMutex` 并更新所有调用点（13 个文件、164 处调用）

### 阶段 1：`xworker.c` 拆分
- [x] 扩充 `xworker_internal.h` 声明（增加 `xr_xorshift32` inline、6 条 XR_FUNC 声明）
- [x] 拆出 `xworker_runq.c`（228 行）
- [x] 拆出 `xworker_blocked.c`（299 行）
- [x] 拆出 `xworker_pool.c`（187 行）
- [x] 拆出 `xworker_handoff.c`（225 行）
- [x] 拆出 `xworker_exec.c`（1005 行，含 440 行 `xr_coro_run_on_worker` 待 Phase 5 拆分）
- [x] 拆出 `xworker_sched.c`（803 行）
- [x] `xworker.c` 精简到 551 行（原 3167 行）
- [x] 所有非 static 跨文件函数通过 `xworker.h` / `xworker_internal.h` 的 `XR_FUNC` 声明

### 阶段 2：正确性 bug 修复
- [x] 修 `xr_poll_cache_alloc` mutex 双初始化（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xnetpoll.c:54-101`，加 `cleanup` 里 destroy，改为 alloc 时仅首次 init）
- [x] 修 `XrBlockedBucket` 泄漏（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker_blocked.c`，`bucket_reclaim_if_empty` 在 wake_one / dequeue_blocked / wake_all 回收空桶；select 路径遗留给 Phase 6 整改）
- [x] `worker_blocked_list_remove` / `sysmon_thread_func` 可见性（通过 Phase 1 的 `xworker_internal.h` XR_FUNC 声明已修正）
- [x] `XrBalanceInfo` 去重（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.h:91`，直接使用 `XrBalanceInfo` 类型替代匿名 struct）
- [x] `xr_runq_steal` 加 XR_DCHECK（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker_runq.c:68-73`）
- [x] `xr_coro_pool_init` 错误路径指针置 NULL（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xcoro_pool.c:114-127`）

### 阶段 3：清理 + 命名一致性
- [x] `XrScheduler` → `XrCoroState` 重命名 + 字段裁剪（`vm.scheduler` → `vm.coro_state`；删除死字段 `current`（从未被设为非 NULL）和 `next_id`（从未被读取）；保留 `ready_head/tail`、`coro_count`、`total_created`、`current_scope`、`coro_registry`。涉及 12 个源文件 + 2 个 stdlib 文件共 45 处引用全部更新。69/69 ctest 通过。）
- [x] `xr_channel_dist_hooks` 挪到 `XrayIsolate::channel_dist_hooks`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/xisolate_internal.h:147`，所有访问通过 `ch->isolate` 或 `coro->isolate` 或 cold_paths 的 `isolate` 参数，cluster install/uninstall 加 `XrayIsolate *X` 参数）
- [x] `g_channel_close_count` 挪到 `XrSystemHeap::stats.channel_close_count`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xchannel.c:562-564`，`xr_channel_get_close_count(X)` 现需 isolate 参数）
- [x] `xyield_closure.c` 合并入 `xyieldable.c`（131 行文件删除，合并后 `xyieldable.c` = 353 行，仍 ≤ 800 行）
- [x] 添加 `xr_coro_transition_*` 宏封装（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xcoro_flags.h:231-259`，提供 `_to_running/_to_ready/_to_blocked/_wake` 四个语义版本，`xworker_exec.c` / `xworker_sched.c` / `xchannel.c` 的 7 处 hot path 已切换）

### 阶段 4：lock-free 化
- [x] `idle_worker_stack` → Treiber stack（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker_sched.c:40-102`，含 `in_idle_worker_list` 幂等守卫防止双推入环）
- [x] `idle_p_head` / `idle_m_head` → Treiber stack（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xproc.c:190-225`、`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xmachine.c:114-152`）
- [x] 删除 `runtime->sched_lock`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.h:54-79` 字段彻底移除，shutdown 的 idle_m 链唤醒改为无锁 snapshot 遍历）
- [x] 协程池 `free_lock` → Treiber stack（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xcoro_pool.c:170-187`；batch steal 用 `atomic_exchange` 一次拿下整条链 + CAS splice 还回剩余，实现 O(1)）
- [x] `async_pool` 完成队列 → MPSC（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xasync.c:115-148`，XrAsyncReadyQueue 去掉 mutex；任务提交队列保留 mutex+cond 因为是 MPMC 场景，`pthread_cond` 本身已是 futex 封装）
- [x] `XrProc::stats` 子 struct cache-line 对齐（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xproc.h:53-60`，新增 `XrProcStats` aligned(64)，25 处访问改为 `p->stats.X`）

### 阶段 5：巨函数拆分
- [x] `xr_coro_run_on_worker` 443 → 133 行 薄壳 + 3 个 helper（`run_finalize` / `run_first_exec` / `run_resume_path`，`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker_exec.c:583-985`）
- [x] `worker_loop` 275 → 91 行 + 4 个 helper（`worker_bind_cpu` / `worker_housekeeping` / `worker_try_steal` / `worker_spin` + `worker_reset_spinning`，`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker_sched.c:561-835`）
- [x] `run_cfunc_coro` 163 行 → 主函数 60 行 + `run_cfunc_first_exec` / `run_cfunc_resume`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker_exec.c:401-565`）
- [ ] `xr_coro_pool_get` 129 行（**跳过**：仍在 150 行限度内，且逻辑层次已经清晰：fast path → steal → per-worker arena → global arena → grow。进一步拆分边际收益低）

### 阶段 6：热路径算法优化
- [x] `select` 唤醒 O(N) → O(waiters_on_channel)（`XrSelectCase` 新增 `bucket_next` + `owner` 字段嵌入 per-bucket 链；`XrBlockedBucket::select_head/tail` 类型从 `XrCoroutine *` 改为 `XrSelectCase *`；`block_select` 对每个 case 链入对应 bucket；`wake_select` 只遍历目标 bucket 的 select chain；`unblock_select` 遍历所有 case 逐一从 bucket 摘除。`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker_sysmon.c:364-514`）
- [x] channel send/recv 三路径提取 helper（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xchannel.c:326-380` 新增 4 个 inline primitives：`chan_direct_send` / `chan_direct_recv` / `chan_buffer_push` / `chan_buffer_pop`，`notify_send` / `try_send` / `try_recv` / `send` case1+2 / `recv` case1+2 共 5 处 callsite 切换到 helper）
- [x] channel close 大扇出分发（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xchannel.c:548-551` 在 `channel_wake_coro_ex` 中判定 `is_close && target != current` 时走 MPSC inbox 而非 push_lifo，避免单 worker 串行化）
- [ ] `xr_channel_remove_waiter` 改 O(1)（**推迟**：`wait_link` 字段同时用于 channel waitq 和 per-worker `blocked_bucket` 链，单纯加 `wait_prev` 无法解决共用。且函数只在 timeout 冷路径使用。留到后续重构 waitq / bucket 分离时一起处理。）

### 阶段 7：健壮性与扩展
- [x] `ensure_workers` startup wait 改 futex（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:371-376`、`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker.c:403-408`，`worker_loop` 的 `fetch_add` 后加 `xr_park_futex_wake` 触发唤醒）
- [x] `exitsyscall` spin 改 futex（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker_handoff.c:100-113`，新增 `XrProc::handoff_sync` futex word；exitsyscall 64 轮 spin 后 futex wait 1000μs 超时；handoff release 端 `@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xworker_handoff.c:202-204` 在 `current_m=NULL` 后 `handoff_sync=1` + futex wake）
- [ ] Netpoll fd 迁移支持（**推迟**：需要深入理解 poll_desc / timer_wheel 交互，改动复杂度高且与 Phase 6.1 类似需要 ctest 验证）
- [x] Deep copy seen hash 动态扩容（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xdeep_copy.c:116-165`，新增 `seen_hash_grow` 在 record 前检查 75% 负载因子触发 rehash；arena 中的 entry 保持不动，仅重建 bucket 链；`seen_hash_n` 取代固定宏版本）
- [ ] 运行时背压（runq / inbox / async）（**推迟**：涉及语言层 API 语义变更——spawn 失败改为抛异常；需要与语言 runtime 协调，留作独立 phase）
- [x] Windows netpoll `#error` 明确标记（`@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xnetpoll.c:255-261`，把原 `// TODO` 改为 `#error`，要求显式接入 `xnetpoll_iocp.c` 或禁用网络构建）

## 附录 A：每阶段独立 PR 模板

```
refactor(coro): [Phase N] <one-line summary>

Phase N of docs/engineering/coro_refactor_plan.md.

Changes:
- ...
- ...

Metrics:
- <benchmark>: before X -> after Y (+Z%)
- wc -l src/coro/xworker*.c: max <LINE>

Tests:
- ctest: all pass
- ASan: no findings
- Regression: scripts/run_regression_tests.sh PASS
```

## 附录 B：风险缓解策略

1. **每阶段一个 feature branch**：`refactor/coro-phase-N`，merge 前 rebase on main。
2. **性能回归 gate**：pingpong / spawn1M 退化 > 5% 必须定位到具体 commit。
3. **ASan 必跑**：lock-free 改动（阶段 4、6）强制。
4. **TSan 补充**（如环境可用）：阶段 4 合入前跑 TSan 全量测试。
5. **Rollback plan**：每阶段独立 revertable；阶段 4 最关键，合入后观察 3 天无异常再进入阶段 5。
