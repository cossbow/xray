# Xray 升级优化方案 v1.0

> 基于对 Xray 全代码库的深入分析，结合 Yo 语言等同类项目的经验，制定分阶段优化路线图。
> 日期：2026-04-21

---

## 目录

1. [现状评估](#1-现状评估)
2. [Phase 1：性能天花板突破（1-2 月）](#2-phase-1性能天花板突破)
3. [Phase 2：AOT 增强（2-3 月）](#3-phase-2aot-增强)
4. [Phase 3：开发者体验（1-2 月）](#4-phase-3开发者体验)
5. [Phase 4：语言特性完善（持续）](#5-phase-4语言特性完善)
6. [Phase 5：生态与发布（持续）](#6-phase-5生态与发布)
7. [不做清单](#7-不做清单)

---

## 1. 现状评估

### 架构优势（已有）

| 模块 | 状态 | 亮点 |
|------|------|------|
| **VM** | ✅ 成熟 | 寄存器式字节码，类型化栈，handler 机制 |
| **JIT** | ✅ 成熟 | SSA IR（XIR），ARM64 后端，SCCP/GVN/Loop Opt/寄存器分配，~150KB 核心 |
| **AOT** | ✅ 可用 | XIR→C 翻译，xrt.h 自包含 runtime，类型特化，struct promotion |
| **GC** | ✅ 先进 | Per-Coroutine Immix Mark-Region，无全局 STW，增量 GC |
| **调度器** | ✅ 先进 | GMP 模型，MPSC inbox，work stealing，continuation stealing，LIFO fast dispatch |
| **I/O** | ✅ 完整 | kqueue/epoll netpoll，timer wheel，async thread pool |
| **标准库** | ✅ 丰富 | net/http/ws/json/yaml/toml/xml/csv/regex/crypto/compress/cluster |
| **工具链** | ✅ 完整 | CLI/REPL/LSP/DAP/formatter/test runner/package manager |

### 需要突破的瓶颈

| 瓶颈 | 影响 | 原因 |
|------|------|------|
| VM 解释开销 | 计算密集型慢 3-5x vs C | dispatch + type checking 在热路径上 |
| AOT 覆盖不全 | 无法生成独立二进制 | 缺 class/coroutine/module/stdlib AOT |
| xrt.h 单体化 | AOT 编译时间长 | 884 行全 include，无按需选择 |
| HTTP 吞吐天花板 | 被 Go/Rust 压制 | 标准库还未做极致优化 |
| 编译到原生码无协程支持 | AOT 代码无法用 go/chan | 协程完全依赖 VM |

---

## 2. Phase 1：性能天花板突破

**目标：在 HTTP benchmark 中达到 Go 级别吞吐量**

### 1.1 Netpoll 零拷贝 fast-path

**现状**：`worker_poll_sources()` 每次调用 `xr_netpoll_poll()` 返回 `XrReadyList`，遍历唤醒协程。

**优化**：
- 当 I/O ready 的协程是当前 worker 的 affinity 协程时，跳过入队直接执行（类似 Yo 的 bounded-inline sync-completion）
- 为 LIFO slot 增加 I/O 优先级：I/O ready 协程优先于 compute 协程

```
// 伪代码
XrCoroutine *io_coro = ready.head;
if (io_coro && io_coro->affinity_p == worker->p.id && !io_coro->sched_link) {
    // 单个 I/O 唤醒：直接执行，跳过队列
    worker_reset_spinning(worker, runtime);
    worker_exec_with_cont_stealing(worker, io_coro);
    return;
}
```

**预估收益**：HTTP echo server 吞吐 +15-25%
**文件**：`src/coro/xworker_sched.c`

### 1.2 Netpoll 混合加速（共享 + per-worker local poll）

**现状**：
- 所有 worker 共享一个 `runtime->netpoll`（单 kqueue/epoll fd）
- 已有 `XrLocalPoll` 完整实现（`xnetpoll.c`）和 `XrProc.local_poll` 字段，但**未接入调度循环**
- 已有 `pd->owner_worker_id` fd 亲和性绑定机制

**方案**：混合模式（保持共享 netpoll + 叠加 local poll fast-path），**不做纯独立替换**。

> ⚠️ 纯 per-worker netpoll 与 GMP work stealing 存在根本矛盾：
> 协程迁移后 fd 仍在原 worker 的 local_poll 上，新 worker 收不到事件。
> Yo 没有此问题因为 Yo 没有协程迁移。Go 也选择了共享 netpoll。

**混合模式设计**：

```
┌─────────────────────────────────────────────────┐
│               runtime->netpoll                  │
│   (共享 kqueue/epoll, 兜底, 处理所有 fd)         │
└─────────────────┬───────────────────────────────┘
                  │  fd 同时双注册
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
 Worker 0      Worker 1      Worker 2
 local_poll    local_poll    local_poll
 (只含绑定到   (只含绑定到   (只含绑定到
  本 worker     本 worker     本 worker
  的 fd)        的 fd)        的 fd)
```

**调度逻辑改造**（`worker_poll_sources`）：

```c
void worker_poll_sources(XrWorker *worker) {
    XrProc *p = &worker->p;

    // Fast path: local poll（本 worker 绑定的 fd，零竞争）
    if (p->local_poll.poll_fd >= 0) {
        XrReadyList local_ready = {0};
        xr_local_poll_events(&p->local_poll, 0, &local_ready);
        dispatch_ready_list(worker, &local_ready);  // 直接入 LIFO
    }

    // Slow path: shared netpoll（未绑定 fd + 跨 worker 事件）
    XrReadyList shared_ready = xr_netpoll_poll(&runtime->netpoll, 0);
    dispatch_ready_list(worker, &shared_ready);

    // ... 其余不变（async pool, timers, inbox）
}
```

**实现步骤**：
1. `xr_worker_init()` 中调用 `xr_local_poll_init(&worker->p.local_poll)`
2. `xr_netpoll_bind_worker()` 中同时把 fd 注册到 owner 的 `local_poll`
3. `worker_poll_sources()` 先 poll `local_poll` 再 poll 共享 `netpoll`
4. 共享 `netpoll` 收到 `owner_worker_id != self` 的事件时，已有 `affinity_p` + inbox 路由机制

**为什么不做纯 per-worker 独立 netpoll**：

| 问题 | 纯独立的风险 | 混合模式如何规避 |
|------|-------------|----------------|
| 协程 work stealing 后 fd 归属错乱 | fd 在原 worker local_poll，新 worker 收不到事件 | 共享 netpoll 兜底，迁移后事件仍可被收到 |
| Listen socket thundering herd | 需要 SO_REUSEPORT 改造 stdlib/net | Listen fd 走共享 netpoll，无需改动 |
| fd 数量不均衡 | Worker 间 I/O 负载无法均衡 | 共享 netpoll 自然均衡 |
| 短连接频繁注册/注销 | 双 syscall 开销 | local_poll 注册是可选的加速，非必须 |

**预估收益**：
- 绑定 fd 的 I/O 事件延迟降低（跳过共享 kqueue 竞争）
- HTTP 长连接场景吞吐 +10-20%（多数事件走 local fast-path）
- 风险极低（共享 netpoll 始终兜底，可随时回退）

**文件**：`src/coro/xworker_sched.c`, `src/coro/xnetpoll.c`, `src/coro/xworker.c`

### 1.3 stdlib/net TCP_NODELAY + 响应合并

**现状**：`stdlib/net/net.c`（51KB）功能完善，但可能缺少性能微调。

**优化**：
- TCP server 默认启用 `TCP_NODELAY`
- 连接池预热（`conn_pool.c` 已有，确认冷启动路径）
- HTTP keep-alive 响应头合并写入（减少 syscall）
- writev 批量发送

**预估收益**：HTTP 延迟降低 20-40%
**文件**：`stdlib/net/net.c`, `stdlib/http/`

### 1.4 JIT 编译队列优化

**现状**：`src/jit/xjit_compile_queue.c` 存在后台编译队列。

**优化**：
- Tier-1 快速编译（无优化 pass，仅寄存器分配）用于首次 JIT
- Tier-2 完整优化（SCCP + GVN + Loop Opt）用于热点函数
- 对应 Yo 的"先编译后优化"两阶段策略

**预估收益**：首次 JIT 延迟降低 50%，长期性能不变
**文件**：`src/jit/xir_jit.c`, `src/jit/xjit_compile_queue.c`

---

## 3. Phase 2：AOT 增强

**目标：支持 `xray build --native` 生成独立可执行文件**

### 2.1 xrt.h 分层化

**现状**：`xrt.h`（884 行）单个 header，全部 `static inline`。

**优化**：

```
src/aot/
├── xrt_core.h      → 值表示 + boxing/unboxing（~80 行，必选）
├── xrt_arith.h     → 算术 + 比较（~60 行，需要 tagged 运算时）
├── xrt_array.h     → Array 操作（~80 行，需要数组时）
├── xrt_map.h       → Map 操作（~50 行，需要 map 时）
├── xrt_string.h    → String 操作（~80 行，需要字符串时）
├── xrt_arc.h       → ARC + bump allocator（~100 行，需要对象时）
├── xrt_closure.h   → 闭包运行时（~30 行，需要闭包时）
├── xrt_method.h    → 方法分派（~200 行，需要方法调用时）
├── xrt_print.h     → print/println（~30 行，需要输出时）
├── xrt_compat.h    → 源码级别名（XrValue / XR_TAG_* 等）
└── xrt.h           → AOT runtime umbrella header (#include all)
```

`xcgen.c` 中细化 `needs_runtime` 为按子系统：
```c
typedef struct XcgenFunc {
    ...
    bool needs_array;    // 用了 Array
    bool needs_map;      // 用了 Map
    bool needs_string;   // 用了 String
    bool needs_arc;      // 用了 ARC 对象
    bool needs_closure;  // 用了闭包
    bool needs_method;   // 用了方法调用
    ...
};
```

**收益**：纯算术函数零 #include 开销，编译更快，产物更小
**工程量**：小（纯重构，不改功能）

### 2.2 AOT + JIT 协程协同优化

**这是最大的技术挑战，但也是最高价值的改进。**

**目标**：
- AOT：让生成的 C 代码能使用 `go`/`chan`/`await`，不依赖 VM
- JIT：补全协程 opcode 支持，消除 deopt 热点

#### 现状分析：JIT 协程的 CPS 寄存器快照模型

JIT 已有成熟的协程挂起/恢复机制（`XIR_SUSPEND`），但与 AOT 需要的状态机是**不同策略**：

```
JIT 路径（ARM64 原生码）：
  挂起 = 保存全部 22 个 GP 寄存器 + spill slots 到 coro->jit_suspend_state
  恢复 = resume entry stub reload 寄存器 + suspend_id 跳转表

AOT 路径（C 代码）：
  挂起 = f->state = N; return BLOCKED（只保存跨 yield 活跃变量）
  恢复 = switch(f->state) { case N: ... }（C 编译器优化）
```

| 维度 | JIT（CPS 快照） | AOT（状态机） |
|------|-----------------|--------------|
| 挂起开销 | 22 个 STP ~200B 码 | 1 个赋值 + return |
| 恢复开销 | reload + 跳转 ~300B | switch + 跳转（编译器优化） |
| 协程帧 | 固定 ~200B（全量寄存器） | 只有活跃变量（通常 <100B） |
| 支持范围 | await/channel ✅, go/yield ❌(deopt) | 全部 ✅（规划中） |

#### JIT 可复用资产（AOT 直接复用）

1. **Block Helper 函数**：`xr_jit_await_block`、channel send/recv helper
   已封装调度器交互 + gopark 安全模式 + inline resume fast-path
   → AOT 状态机的 `return BLOCKED` 前直接调用同一套 helper

2. **suspend_id 机制**：JIT 已有 `suspend_id` 编号每个挂起点
   → AOT 的 `switch(f->state)` 分支与 `suspend_id` 一一对应

3. **gopark 模式**：在 block_helper 设 BLOCKED 之前预存恢复信息
   → AOT 必须遵守同样模式：`f->state = N` 在调 helper 之前

#### AOT 协程设计（状态机翻译）

```c
// AOT 生成的协程帧（只包含跨 yield 活跃变量）
typedef struct {
    int state;                  // 当前状态（suspend_id）
    int64_t v0, v1;             // 跨 yield 存活的局部变量
    XrtValue result;            // await/channel 结果
    XrtValue send_val;          // channel send 值
} xrt_coro_frame_fib;

// AOT 生成的步进函数
static int xrt_coro_step_fib(xrt_coro_frame_fib *f, XrtContext ctx) {
    switch (f->state) {
    case 0:
        f->v0 = 1; f->v1 = 1;
        f->send_val = xrt_box_int(f->v0);
        f->state = 1;
        // 调用与 JIT 相同的 block helper
        if (!xrt_channel_send_try(ctx, f->chan, f->send_val))
            return XRT_CORO_BLOCKED;
        // fall through (inline resume)
    case 1:
        f->v0 = f->v0 + f->v1;
        // ...
        return XRT_CORO_DONE;
    }
    return XRT_CORO_DONE;
}
```

#### JIT 协程补全（消除 deopt 热点）

当前 JIT 遇到 `OP_GO/OP_YIELD/OP_SLEEP` 全部 deopt 回解释器。逐步补全：

| opcode | 优先级 | 方案 | 预估收益 |
|--------|--------|------|---------|
| `OP_SPAWN_CONT` cont stealing | **P0** | 返回 `XIR_JIT_SPAWN_CONT` 让 worker 执行 cont stealing（当前跳过了） | spawn 密集 +20-30% |
| `OP_SLEEP` | P1 | 类似 SUSPEND，CALL_C 调用 `xr_jit_sleep_block` | 消除 sleep deopt |
| `OP_YIELD` | P1 | 保存寄存器 + return `XIR_JIT_YIELD` | 消除 yield deopt |
| `OP_GO` | P2 | 类似 SPAWN_CONT 但无 scope | 消除 go deopt |
| SUSPEND 寄存器瘦身 | P2 | 只保存 liveness 标记的活跃寄存器 | 代码体积 -15% |

#### 统一 Dispatch 架构（⚡ 关键设计决策）

**原则**：不强迫不同执行模式共享一个为 VM 字节码设计的路径。每种模式有自己的最短路径。

**当前问题**（`xr_coro_run_on_worker` 中的 3 个设计迁就）：

1. **JIT 被排除在 channel resume fast path 之外**：
   `!coro->jit_resume_entry` 条件把 JIT 协程强制推到 `run_resume_path` 慢路径，
   因为 JIT 需要把 `recv_slot` 拷贝到 `jit_suspend_state.result`。

2. **`jit_suspend_state`（320B）嵌入每个 `XrCoroutine`**：
   `caller_saved[15] + callee_saved[8] + result + result_tag + spill[15]` = 320 字节，
   对 VM-only 协程完全浪费。1 万协程浪费 3.2MB。

3. **`run_resume_path` 是万能函数**：JIT/VM/continuation/debug 四种 resume 混在一起。

**最佳方案**：Dispatch Shell 只做路由，每种执行模式有独立 fast path。

```
xr_coro_run_on_worker（dispatch shell，纯路由）
│
├─ jit_resume_entry != NULL  →  run_jit_resume()     // JIT 专用 fast path（新增）
├─ ENTRY_AOT                 →  run_aot_coro()        // AOT 专用（新增）
├─ ENTRY_CFUNC               →  run_cfunc_coro()      // 已有
├─ ENTRY_NATIVE              →  native callback       // 已有
├─ ENTRY_CLOSURE + channel   →  VM channel fast path  // 已有
└─ ENTRY_CLOSURE + other     →  run_resume_path()     // VM 通用
```

**AOT 入口类型**：新增 `XR_CORO_ENTRY_AOT`，不走 VM 路径：

```c
// XrCoroEntryType 新增：
typedef enum {
    XR_CORO_ENTRY_CLOSURE,
    XR_CORO_ENTRY_NATIVE,
    XR_CORO_ENTRY_CFUNC,
    XR_CORO_ENTRY_AOT       // 新增
} XrCoroEntryType;

// XrCoroEntry union 新增：
typedef struct {
    void *frame;        // AOT 状态机帧（xrt_coro_frame_xxx*）
    XrtStepFn step;     // 步进函数指针
} XrAotEntry;

// run_aot_coro: ~30 行极简函数
static XrVMResult run_aot_coro(XrWorker *worker, XrCoroutine *coro,
                                XrayIsolate *isolate) {
    XrAotEntry *aot = &coro->entry.aot;
    xr_coro_transition_to_running(coro);
    int rc = aot->step(aot->frame, isolate);
    switch (rc) {
    case XRT_CORO_DONE:    coro->result = ((XrtFrameBase*)aot->frame)->result;
                           return XR_VM_OK;
    case XRT_CORO_BLOCKED: return XR_VM_BLOCKED;
    case XRT_CORO_YIELD:   return XR_VM_YIELD;
    default:               return XR_VM_RUNTIME_ERROR;
    }
}
```

为什么不兼容 `xr_coro_run_on_worker` 现有 VM 路径：
- AOT 协程**没有** `vm_ctx`/`closure`/`proto`，兼容需要伪造这些结构（浪费 ~2KB/协程）
- Xray 是全新语言，不需要兼容层，直接采用最佳设计

**JIT resume 独立 fast path**：把当前 `run_resume_path` 中的 JIT 分支抽出：

```c
// run_jit_resume: ~40 行，直达 xir_jit_resume
static XrVMResult run_jit_resume(XrWorker *worker, XrCoroutine *coro,
                                  XrayIsolate *isolate) {
    xr_coro_transition_to_running(coro);
    coro->jit_ctx = &worker->p.jit_scratch;
    int rs = xr_coro_resume_load(coro);

    if (rs == XR_RESUME_CHANNEL_CLOSED) {
        coro->jit_resume_entry = NULL;  // deopt to VM
        return run_resume_path(...);
    }
    if (rs == XR_RESUME_CHANNEL) {
        // recv_slot → jit_suspend_state.result（现在不用绕 run_resume_path）
        XrValue rv = coro->vm_ctx.stack[0];
        coro->jit_suspend->result = rv.i;
        coro->jit_suspend->result_tag = rv.tag;
    }
    // await result propagation ...
    xr_coro_resume_store(coro, XR_RESUME_OK);

    XrValue jit_result;
    int rc = xir_jit_resume(coro, &jit_result);
    // OK / SUSPEND / DEOPT 处理
    ...
}
```

**`jit_suspend_state` 按需分配**：从 `XrCoroutine` 内联改为指针 + 懒分配：

```c
// 当前：320B 嵌入每个 XrCoroutine（多数协程永远不用）
struct { int64_t caller_saved[15]; ... } jit_suspend_state;  // 320B 浪费

// 优化：指针 + 首次 JIT suspend 时分配
XrJitSuspendState *jit_suspend;  // 8B，NULL = 无 JIT 挂起状态
// 1 万协程省 ~3.1MB
```

#### AOT 调度器选择

- **嵌入式场景**：AOT 代码链接 `libxray_rt.a`，新增 `XR_CORO_ENTRY_AOT` 入口
  → worker 调度循环（work stealing / cont stealing / LIFO）自动生效，零改动
- **独立 CLI 场景**：轻量 mini scheduler（单线程 round-robin）
  生成在 AOT C 文件内，无外部依赖

#### 演进路线

```
阶段 0（1 周）：Dispatch 重构
  0.1  抽出 run_jit_resume()，JIT 协程不再走 run_resume_path
  0.2  jit_suspend_state 改指针 + 懒分配（省 320B/协程）
  0.3  channel resume fast path 去掉 !jit_resume_entry 排除条件

阶段 A（2 周）：JIT 协程补全
  A1: SPAWN_CONT 支持 continuation stealing
  A2: OP_SLEEP / OP_YIELD JIT 化
  A3: SUSPEND 寄存器瘦身（只保存活跃寄存器）

阶段 B（3 周）：AOT 状态机基础
  B1: XR_CORO_ENTRY_AOT + run_aot_coro dispatch 分支
  B2: xcgen 识别含 SUSPEND 的函数，生成状态机帧 + step 函数
  B3: 复用 JIT block helpers（xrt_coro.h 暴露 C 接口）
  B4: 生成 mini scheduler（单线程版）

阶段 C（2 周）：AOT 调度器集成
  C1: AOT 协程帧适配 XrCoroutine + ENTRY_AOT
  C2: 链接 libxray_rt.a 使用完整 GMP 调度器
  C3: go + chan + select + timeout 完整支持

远期：纯 JIT 协程（类型已稳定、无 deopt 点）可跳过 vm_ctx 分配
  → 本质上 JIT 自动产出 AOT 级别的轻量协程
```

**预估收益**：
- 阶段 0：每协程省 320B，JIT channel resume 延迟降低
- 阶段 A：JIT spawn 密集场景 +20-30%，消除协程相关 deopt
- 阶段 B/C：AOT 协程代码接近 C 性能，每协程帧 <100B
**工程量**：阶段 0（1 周）+ A（2 周）+ B/C（5 周），可独立推进

### 2.3 AOT Runtime 静态库（libxray_rt.a）

**目标**：对于需要完整语言特性的 AOT，提供静态链接的 runtime 库。

**方案**：
```
libxray_rt.a 包含：
├── xrt_gc.o        → 精简版 Immix GC（从 xcoro_gc.c 裁剪）
├── xrt_class.o     → 类系统核心（从 runtime/class/ 裁剪）
├── xrt_channel.o   → Channel 实现
├── xrt_scheduler.o → 精简版 GMP 调度器
├── xrt_stdlib.o    → 常用标准库（string/array/map）
└── xrt_io.o        → 网络 I/O（从 stdlib/net/ 裁剪）
```

AOT 生成的 C 代码 + `libxray_rt.a` → 独立可执行文件：
```bash
xray build --native app.xr
# 生成 app.c + 编译命令：
# clang -O3 app.c -L. -lxray_rt -o app
```

**收益**：完整语言特性的 AOT 支持
**工程量**：中（主要是从现有代码中裁剪 + 定义稳定 ABI）

### 2.4 不逃逸闭包优化

**现状**：所有闭包都通过 `xrt_cl->upvals[i]` 间接访问 upvalue。

**优化**：编译期检测不逃逸的闭包，将 upvalue 提升为函数参数：

```c
// 优化前
static XrValue lambda_1(XrtContext ctx, xrt_closure_t *xrt_cl) {
    int64_t x = xrt_cl->upvals[0].i;   // 间接
    int64_t y = xrt_cl->upvals[1].i;   // 间接
    return xrt_box_int(x + y);
}

// 优化后（不逃逸闭包）
static XrValue lambda_1(XrtContext ctx, int64_t x, int64_t y) {
    return xrt_box_int(x + y);         // 零间接
}
```

**实现**：在 `xcgen_compile_func()` 预扫描阶段增加逃逸分析：
- 如果闭包只被 `CALL_KNOWN` 调用，不存储到堆上 → 不逃逸
- 不逃逸闭包：upvalue 变为额外参数
- 逃逸闭包：保持现有 `xrt_cl->upvals[]` 方式

**工程量**：中
**文件**：`src/aot/xcgen.c`, `src/aot/xcgen_call.c`

---

## 4. Phase 3：开发者体验

### 3.1 HTTP Benchmark 工具

**目标**：内置 benchmark 命令，验证优化效果。

```bash
xray bench --http tests/demos/http_server.xr
# 自动：启动服务器 → wrk 压测 → 输出结果 → 与基准比较
```

**实现**：
- `src/app/cli/xcmd_bench.c` — 新子命令
- 内置 wrk 核心逻辑（HTTP/1.1 client + 统计）
- 与 Go/Bun/Node.js 的自动对比

### 3.2 AI Skill Pack（.xray/skills/）

**借鉴 Yo 的 `.github/skills/`**：

```
.xray/
├── skills/
│   ├── xray-syntax/          → 教 AI 写 .xr 代码
│   ├── xray-stdlib/          → 标准库用法示例
│   ├── xray-async/           → 协程和 channel 模式
│   └── xray-embedding/       → C 嵌入 API 用法
├── rules/                    → 已有的编码规范
└── workflows/                → 已有的工作流
```

每个 skill 包含：
- `README.md` — 概述
- `examples/` — 代码示例
- `patterns.md` — 常见模式
- `gotchas.md` — 常见坑点

**收益**：AI 编码助手能更好地写 Xray 代码

### 3.3 REPL 增强

- 多行编辑支持
- 自动补全（复用 LSP 的补全逻辑）
- `go` 语句在 REPL 中可用（后台 worker 已启动）
- `.time` 命令测量表达式执行时间

### 3.4 错误信息改进

- 编译错误：显示代码上下文 + 修复建议（类似 Rust）
- 运行时错误：彩色 stack trace + 源码位置
- 类型错误：expected vs actual 对比

---

## 5. Phase 4：语言特性完善

### 4.1 类型系统增强

| 特性 | 优先级 | 说明 |
|------|--------|------|
| 泛型（Generics） | 高 | `fn map<T, U>(arr: [T], f: fn(T) -> U) -> [U]` |
| 接口（Interface） | 高 | `interface Reader { fn read(buf: Buffer) -> int }` |
| 枚举（Enum） | 中 | `enum Color { Red, Green, Blue(int) }` |
| 模式匹配（Pattern Match） | 中 | `match value { Color.Red => ..., _ => ... }` |
| 可选类型（Optional） | 中 | `fn find(key: string) -> int?` |

### 4.2 错误处理改进

**现状**：try/catch 机制。

**增强**：
- `Result<T, E>` 类型（配合泛型）
- `?` 操作符自动传播错误
- `defer` 语句确保资源清理

### 4.3 编译期求值（CTFE）

- 编译期常量表达式求值
- `const fn` 编译期执行的函数
- 编译期字符串插值
- 条件编译（`#if`/`#else`/`#endif`）

---

## 6. Phase 5：生态与发布

### 5.1 包管理器完善

- `xray pkg init` — 初始化项目
- `xray pkg add <name>` — 添加依赖
- `xray pkg publish` — 发布到 registry
- `xray.lock` — 版本锁定文件
- 中央 registry（xray-pkg.org）

### 5.2 文档站

- API 文档自动生成（从源码注释）
- 教程系列（Getting Started → Advanced）
- Playground（浏览器中运行 .xr 代码）
- 性能对比页面

### 5.3 跨平台

| 平台 | 状态 | 待办 |
|------|------|------|
| macOS ARM64 | ✅ 主平台 | - |
| Linux x86_64 | ✅ CI | JIT x86_64 后端 |
| Linux ARM64 | 🔜 | 测试覆盖 |
| Windows | 🔜 | IOCP netpoll + MinGW/MSVC 构建 |
| WASM | 💡 | 无协程的子集可行 |

### 5.4 CI/CD 增强

- Release 自动化（GitHub Actions 已有基础）
- 性能回归检测（每次 PR 跑 benchmark）
- 模糊测试覆盖率提升（`tests/fuzz/` 已有基础）

---

## 7. 不做清单

以下明确**不做**，避免方向偏移：

| 不做 | 原因 |
|------|------|
| 改用 RC 替换 Mark-Sweep GC | Immix GC 更适合高并发，Per-Coro GC 已避免 STW |
| 改用 Lisp 式语法 | Xray 的 C-like 语法更主流、更易读 |
| 用 TS/Rust 重写编译器 | 纯 C 实现是核心优势（自举、嵌入、移植性） |
| transpile-only（去掉 VM） | VM 是脚本模式/REPL/调试的基础，不可去掉 |
| 模仿 Go 的 goroutine 栈管理 | Xray 的 stackless 协程 + Immix 更内存高效 |

---

## 优先级总览

```
紧急+高价值:
  [P0] 1.1 Netpoll 零拷贝 fast-path         → 性能立竿见影
  [P0] 1.3 TCP_NODELAY + writev              → HTTP 延迟降低
  [P0] 2.1 xrt.h 分层化                      → AOT 编译体验改善

高价值:
  [P1] 1.2 Per-Worker kqueue                  → 多核网络线性扩展
  [P1] 2.4 不逃逸闭包优化                    → AOT 闭包性能
  [P1] 3.1 HTTP Benchmark 工具               → 量化优化效果
  [P1] 4.1 泛型 + 接口                       → 语言表达力飞跃

中价值:
  [P2] 1.4 JIT 两阶段编译                    → 首次 JIT 延迟
  [P2] 2.2 AOT 协程状态机                    → 独立二进制关键
  [P2] 2.3 libxray_rt.a                      → AOT 完整语言支持
  [P2] 3.2 AI Skill Pack                     → 开发者生态

持续迭代:
  [P3] 3.3 REPL 增强
  [P3] 4.2 错误处理改进
  [P3] 5.x 生态建设
```

---

## 附录：关键指标

| 指标 | 当前值 | Phase 1 目标 | Phase 2 目标 |
|------|--------|-------------|-------------|
| HTTP echo RPS（单核） | 待测 | >200K | >300K |
| HTTP echo RPS（8核） | 待测 | >1M | >1.5M |
| fib(35) vs C | ~3-5x 慢 | ~2-3x 慢（JIT） | ~1.1x 慢（AOT） |
| 协程创建速度 | 待测 | >1M/s | >2M/s |
| AOT 编译时间 | 待测 | <1s（小文件） | <1s |
| 独立二进制大小 | N/A | N/A | <1MB（hello world） |
