# Windows IOCP 后端实现计划

**目标**：让 xray 在 Windows 上跑起来，网络/IO 路径与 macOS（kqueue）/Linux（epoll/io_uring）行为一致。

**原则**：不考虑向后兼容，直接最佳设计；现存的 `netpoll_iocp.c`（175 行半成品）和 `os_poll_iocp.h`（同样的"假定 IN+OUT 都就绪"错误假设）整体推倒重写。

---

## 1. 现状回顾

### 1.1 xray 现有架构（`src/coro/xnetpoll.{h,c}`）

- **统一抽象**：`XrNetpollOps` 函数指针表（`init/cleanup/add_fd/del_fd/poll_events/wakeup`）
- **可读/可写就绪模型**（readiness-based），与 kqueue/epoll 一致
- 全局 `XrNetpoll` 持有 fd→pd 二级映射、descriptor 缓存池、wakeup pipe、backend ops
- 每个 worker 还有 `XrLocalPoll`（per-worker kqueue/epoll fd），缓解共享 poller 上的争用
- 上层 stdlib（net/http/ws/cluster）流程：
  ```
  set_nonblock(fd) → try recv() → if EWOULDBLOCK → xr_yield_for_io(pd, READ) → recv()
  ```
  这是**典型 readiness 模型**：netpoll 只管"告诉我什么时候可读/可写"，实际 I/O 由调用者完成。

### 1.2 Windows 现状（坏的 / 缺的）

| 文件 | 状态 |
|---|---|
| `src/coro/xnetpoll.c:264` | 直接 `#error` 阻止 Windows 编译 |
| `src/os/netpoll/netpoll_iocp.c` | 175 行半成品，**未接入** `netpoll_default_ops()`，函数名与 unified API 撞符号 |
| `src/os/os_poll_iocp.h` | 给 LSP/MCP transport 用的简单 poll；**同样错误地假定**每次 completion 都是 "IN \| OUT 同时就绪" |
| `src/os/win/*.c` | codemem/dir/dylib/fd/fs/mem/proc/random/thread/time **全部已实现**（这是好消息） |

简单说：**网络栈是 Windows 平台支持的唯一空白**。

---

## 2. 关键技术决策

### 2.1 选择 wepoll-style AFD readiness 包装，**不**走 Go-style 真完成模型

读完 wepoll / libuv / Go runtime 三种参考后，三种 Windows IOCP 集成方式对比：

| 方式 | 代表 | 模型 | 特点 |
|---|---|---|---|
| **AFD readiness** | wepoll, libuv | IOCP 上层包成 readiness | 通过 `\Device\Afd` 注册 socket 状态变化通知；上层仍走 `recv()/send()` |
| **真完成模型** | Go runtime | IOCP 原生 completion | 用户代码直接发 `WSARecv(socket, buf, OVERLAPPED)`，完成时 IOCP 回调 |
| **MSAFD via WSAIoctl** | libuv | AFD readiness 但通过公共 API | 同 wepoll，但绕过 NT 私有 API；代码量大、需要 ping-pong 双请求 |

**xray 选 AFD readiness（wepoll 路线），原因**：

1. **stdlib 全部已写成 readiness 模型**（`try recv() → EWOULDBLOCK → wait`）。改为真完成模型意味着重写所有 stdlib 网络代码加 Windows 分支，并维护两套并行实现 —— 违反"代码简洁"原则。
2. `XrNetpollOps` 抽象本身就是 readiness 语义（add_fd 表达"我想知道何时可读/可写"），换 backend 不应改契约。
3. wepoll 的 AFD 路线是 libuv / Node.js / Rust mio / Bun / Deno 在用的事实标准；性能差距 vs 真完成模型在 ms 级 I/O 上不可见。
4. wepoll 单文件 ~2000 行 C 代码，读懂即可移植，不引入 NT 内核 API 之外的依赖。

**不选真完成模型的代价**：
- 损失约 1 次 syscall/操作（用户态 `recv()` 后 EWOULDBLOCK，再次 `recv()` 拿数据）
- 这部分在 hot path 上可量化，但不是当前优先级；如果未来证明是瓶颈，再做 Windows 专属真完成路径并不破坏现有契约。

### 2.2 不引入 wepoll 库；自己写 ~600 行 C

wepoll 单文件 2254 行，但其中**至少 60% 是我们用不到的**：
- `tree_*` / `queue_*` / `reflock_*` / `ts_tree_*`：xray 已有等价基础设施（XrFdMap、Treiber stack、xr_mutex/cond）
- 多个 epoll 实例：xray 是 runtime，每进程**一个 IOCP**，不需要多 port_state
- 跨线程引用计数：xray 用 worker-bound + deferred free 方案

实际需要从 wepoll 移植的核心：
- AFD 设备打开（~30 行）
- AFD_POLL_INFO 结构和 IOCTL_AFD_POLL 调用（~50 行）
- epoll↔AFD 事件位掩码转换（~30 行）
- 单 socket 状态机（IDLE/PENDING/CANCELLED/CLOSED）（~150 行）
- update queue 重新发起 poll（~80 行）
- IOCP completion 派发（~80 行）
- base_socket 解析 via SIO_BASE_HANDLE（~30 行）

合计 ~450 行核心代码 + ~150 行胶水 ≈ **600 行**，可控。

### 2.3 共享 IOCP，不做 per-worker IOCP（当前阶段）

- Linux/macOS 当前已实现 per-worker `XrLocalPoll`（kqueue/epoll fd 各 worker 一份）
- Windows 上 IOCP **天然多线程友好**：所有 worker 调 `GetQueuedCompletionStatusEx` 共享同一 IOCP
- 当前阶段：Windows 走单 IOCP，所有 worker 在同一 IOCP 上等。不做 per-worker IOCP
- 等基础跑稳后，根据 perf 数据再决定是否上 per-worker IOCP（参考 Tokio multi-runtime 模式）

### 2.4 接入策略：扩展 XrPollDesc，**不**新增 Windows 专属 desc 类型

每个被 watch 的 socket 在 wepoll 里有 `sock_state_t`（包含 IO_STATUS_BLOCK / AFD_POLL_INFO / 用户事件位）。
xray 已有 `XrPollDesc`，是每 fd 独占的"运行时状态"对象。

**做法**：在 `XrPollDesc` 上加一段**仅 Windows 编译可见**的字段，集中表达 IOCP 状态机：

```c
typedef struct XrPollDesc {
    // ... existing fields (rg, wg, rseq, wseq, ...)

#ifdef XR_OS_WINDOWS
    // IOCP / AFD state
    IO_STATUS_BLOCK iocp_iosb;        // submitted to IOCP, completion lands here
    AFD_POLL_INFO afd_poll_info;      // single-handle variant (32 bytes)
    SOCKET base_socket;               // SIO_BASE_HANDLE result
    uint32_t user_events;             // EPOLLIN/EPOLLOUT bitmap user wants
    uint32_t pending_events;          // events submitted in current AFD poll
    uint8_t poll_status;              // IDLE / PENDING / CANCELLED
    uint8_t in_update_queue;          // 1 if in port->update_queue
    XrPollDesc *update_link;          // intrusive update queue link
#endif
} XrPollDesc;
```

**理由**：
- 非 Windows 平台编译时这段被剔除，零 ABI 浪费
- Windows 上每个 fd 一份 sock_state，正好对应 pd 一份；分配/生命周期已被 `XrPollCache` 管理
- 不增加额外的 hash table / lookup 路径

### 2.5 wakeup 通过 `PostQueuedCompletionStatus`，丢弃 wakeup_pipe

- 删掉 `xnetpoll.c:264` 那个 `#error`：Windows 上 `wakeup_pipe[]` 字段保留为 -1 即可（成员没用到）
- `iocp_ops.wakeup` 调用 `PostQueuedCompletionStatus(iocp, 0, KEY_WAKEUP, NULL)`
- `iocp_ops.poll_events` 在收到 completion 时根据 completion key 区分：
  - `KEY_WAKEUP`（哨兵值，例如 1）→ wakeup，跳过
  - 其他（XrPollDesc 指针）→ 正常 socket 完成

**Completion Key 编码方案**：
```
key = 0       → wakeup（约定 PostQueuedCompletionStatus 用 0）
key != 0      → (uintptr_t)pd（指向 XrPollDesc）
```
Go 用了 4-bit tagged pointer，xray 简化为：约定 0 是 wakeup 哨兵，其他都是 pd 指针。

### 2.6 base_socket 解析

AFD 必须作用于 **base socket**（最底层服务提供者的 socket handle），不是用户拿到的 socket。
通过 `WSAIoctl(sock, SIO_BASE_HANDLE, ...)` 获取。

- 标准 TCP/UDP socket 始终能解析成功
- 某些 LSP（Layered Service Provider）可能拦截 SIO_BASE_HANDLE，**Windows 7 之后** 提供 `SIO_BSP_HANDLE_POLL` 作为更可靠的备选
- xray 实现：先试 `SIO_BSP_HANDLE_POLL`，失败回退 `SIO_BASE_HANDLE`，再失败 → 报错（不做 select fallback，保持代码简洁）

---

## 3. 新模块结构

### 3.1 文件清单

```
src/os/netpoll/
├── netpoll_iocp.c              [完全重写, ~600 行]
├── netpoll_iocp_afd.h          [新增, AFD 私有头文件 / IOCTL 定义]
├── netpoll_iocp_afd.c          [新增, ~150 行, AFD device 打开/IOCTL 调用]
├── netpoll_kqueue.c            [无修改]
├── netpoll_epoll.c             [无修改]
├── netpoll_iouring.c           [无修改]
└── netpoll_select.c            [无修改]

src/coro/
├── xnetpoll.h                  [+ Windows 字段 in XrPollDesc; 删掉 wakeup_pipe 在 Windows 上的 #error 注释]
└── xnetpoll.c                  [删除 #error; netpoll_default_ops() 加 Windows 分支]
```

新增 `netpoll_iocp_afd.{h,c}` 而不是把 AFD 代码塞进 `netpoll_iocp.c`，原因：AFD 是 NT 私有 API（非公开 SDK），把它和 IOCP backend 分开能让 ops 文件保持"高层调度逻辑"，AFD 文件保持"低层 syscall 包装"。这是 wepoll 自己的分层（`afd.c` vs `port.c` vs `sock.c`）。

### 3.2 ops 表（新 `netpoll_iocp.c` 的对外接口）

```c
static const XrNetpollOps iocp_ops = {
    .name        = "iocp",
    .init        = iocp_init,           // CreateIoCompletionPort + AFD device + WSAStartup
    .cleanup     = iocp_cleanup,        // CloseHandle + WSACleanup
    .add_fd      = iocp_add_fd,         // resolve base_socket + init pd Windows fields + start poll
    .del_fd      = iocp_del_fd,         // cancel pending poll + mark closing
    .poll_events = iocp_poll_events,    // GetQueuedCompletionStatusEx + dispatch
    .wakeup      = iocp_wakeup,         // PostQueuedCompletionStatus(KEY_WAKEUP)
};
```

签名与 kqueue_ops / epoll_ops 完全相同。

### 3.3 状态机（每个 socket）

```
   add_fd                                                       del_fd
     ↓                                                            ↓
  [IDLE] ──submit AFD_POLL──▶ [PENDING] ──completion──▶ [IDLE]  ──▶ done
                                  │                       │
                                  │ event-mask change     │
                                  │ via xr_netpoll_wait   │
                                  ▼                       │
                              [CANCELLED] ──completion──▶─┘
                              (NtCancelIoFileEx)
                                  │
                                  ▼
                              re-arm with new mask
```

- **每次 completion 后**：如果 user_events 还有未触发的位 → 重新 submit；否则 IDLE
- **状态切换** 严格 lock-free（pd 自身的字段，per-pd CAS）

### 3.4 update queue

按 wepoll 设计，把"需要重新提交 AFD_POLL"的 pd 进队列，在 `iocp_poll_events` 主循环里批量处理。
xray 复用现有 intrusive 链表（`pd->update_link`），不引入 std container。

---

## 4. 上层 API 不变

`xr_netpoll_init/open/wait/close` 等公共 API 在 Windows 上行为与 macOS/Linux **完全一致**。stdlib（net/http/ws/cluster）**零修改**。

唯一的语义差异：
- **AFD_POLL 是 one-shot**：每次 completion 后 backend 内部自动 re-arm。这对上层透明。

---

## 5. 阶段计划

> **实施状态（2026-05-10 起）**：Phase 1-3 + Phase 5 全部在 Mac 完成并 113/113 ctest pass。Phase 4（Windows VM 端到端验证）需要实机/CI 跑通，等待用户验证。
>
> 提交链（feat/windows-iocp）：
> ```
> cc19847  Drop dead select backend, mark known xpoll IOCP misreport bug
> 6cc7c8c  Fix close-time UAF in IOCP backend via async ref handoff
> d76bd7b  Wire IOCP readiness through AFD with per-pd state machine
> 44ade64  Add AFD private API wrapper for IOCP backend
> c960b8b  Open Windows compile path for netpoll, scaffold IOCP backend ops
> ```

### Phase 1：解除编译阻塞 + 接口骨架（**Mac 完成 ✓ commit c960b8b**）
- [ ] 删除 `xnetpoll.c:264` 的 `#error` 块；把 `create_wakeup_pipe`/`close_wakeup_pipe` 在 Windows 下变成 no-op
- [ ] 完全重写 `netpoll_iocp.c`：所有函数加 `iocp_` 前缀，提供完整 `XrNetpollOps` 表
- [ ] `netpoll_default_ops()` 加 `XR_OS_WINDOWS` 分支
- [ ] `xnetpoll.c` 在 `XR_OS_MACOS / XR_OS_LINUX` 之后加 `#elif defined(XR_OS_WINDOWS) #include "../os/netpoll/netpoll_iocp.c"`
- [ ] 用 clang/Windows headers 做语法检查（如果可行）；至少保证 macOS 上 `#ifdef XR_OS_WINDOWS` 之外的部分照常编译

**验收**：Mac 上 `cmake --build build` 不动。`grep '#error' src/coro/xnetpoll.c` 为空。

### Phase 2：AFD 私有 API 包装（**Mac 写 + Windows VM 编译**）
- [ ] 新增 `netpoll_iocp_afd.h`：声明 AFD_POLL_INFO / IOCTL_AFD_POLL / NtDeviceIoControlFile / NtCancelIoFileEx 等
- [ ] 新增 `netpoll_iocp_afd.c`：实现 `xr_afd_create_device(iocp, &afd)` / `xr_afd_poll(afd, info, iosb)` / `xr_afd_cancel(afd, iosb)`
- [ ] 实现 `xr_afd_get_base_socket(SOCKET, SOCKET*)`（SIO_BSP_HANDLE_POLL → SIO_BASE_HANDLE 回退）
- [ ] 在 `netpoll_iocp.c` 中：iocp_init → 创建 IOCP + AFD device，iocp_cleanup → 关闭

**验收**：Windows VM 上单元测试 `test_afd_basic`：能开 IOCP、AFD device、提交一次 listen socket 的 AFD_POLL_RECEIVE 请求并收到完成事件。

### Phase 3：state machine + update queue（**Windows VM**）
- [ ] 实现 `iocp_add_fd`：resolve base_socket、初始化 pd Windows 字段、提交首个 AFD_POLL
- [ ] 实现 `iocp_poll_events`：GetQueuedCompletionStatusEx 拿 entries，区分 KEY_WAKEUP / pd*
- [ ] 实现 socket completion 处理：AFD events → epoll-style mask → 调用 `xr_netpoll_ready`
- [ ] 实现 update queue：completion 后如果还有 user_events 未触发 → enqueue；poll 入口处批量重 submit
- [ ] 实现 `iocp_del_fd`：NtCancelIoFileEx + 等待 STATUS_CANCELLED completion
- [ ] 实现 `iocp_wakeup`：PostQueuedCompletionStatus(0)

**验收**：单元测试 `test_iocp_loopback`（TCP loopback：accept/recv/send 三组配对，跨 worker），所有用例 pass。

### Phase 4：协程联调 + stdlib 跑通（**Windows VM**）
- [ ] `tests/coroutine_safety/` 在 Windows 上跑通
- [ ] `stdlib/net` / `stdlib/http` 的现有测试在 Windows 上跑通
- [ ] CI 添加 `windows-latest` runner，进入主 matrix
- [ ] 修复在此过程中暴露的所有 Windows-only bug（fd 复用、close-while-pending、跨 worker timer、wakeup 风暴等）

**验收**：完整 regression `scripts/run_regression_tests.sh` 在 Windows 上 ≥ 90% pass（剩余 fail 必须有 `docs/known_bugs.md` 记录 + 复现步骤）。

### Phase 5：清理 + 副产物（**Windows VM**）
- [ ] 顺手修 `src/os/os_poll_iocp.h` 的"假定 IN+OUT 同时就绪"错误（用 LSP/MCP transport 的 simple poll，可以采用更轻量的 select 方案，因为 transport 流量极小）
- [ ] 删除 `netpoll_select.c` 如果确认无目标平台用到（FreeBSD/Solaris 不在 supported list 上）
- [ ] 把这份 plan 拆成 `docs/known_bugs.md` 的 closed-issue 记录

---

## 6. 测试矩阵

| 层级 | 测试 | 目的 |
|---|---|---|
| **单元** | `test_afd_basic` | AFD device 打开 / 单次 poll 提交 / 接收完成 |
| **单元** | `test_iocp_loopback` | TCP loopback I/O，readiness 触发正确 |
| **单元** | `test_iocp_wakeup` | wakeup 不丢失、不重复唤醒 |
| **单元** | `test_iocp_close_while_pending` | 关 fd 时取消 outstanding poll，无 leak |
| **单元** | `test_iocp_fd_reuse` | OS 复用 fd 号，旧 pd 通知不串到新 pd |
| **集成** | `tests/coroutine_safety/*.xr` | go/select/channel 路径不挂 |
| **集成** | `stdlib/net/test_*` | TCP echo / accept / connect 全跑通 |
| **回归** | `scripts/run_regression_tests.sh` | ≥ 90% pass |
| **CI** | GitHub Actions `windows-latest` | 持续守护 |

---

## 7. 风险与缓解

| 风险 | 概率 | 缓解 |
|---|---|---|
| `\Device\Afd` 是 NT 私有 API，未来 Windows 版本可能改 | 低 | wepoll/libuv/mio 全在用，Microsoft 自己 WSL2 网络栈也在用，事实标准 |
| 某些 LSP 拦截 SIO_BASE_HANDLE | 中 | 优先 SIO_BSP_HANDLE_POLL（W7+），失败直接报错（不引入 select fallback） |
| pd 复用时旧 completion 串扰 | 中 | 复用 xray 已有的 `fdseq` 序列号机制；completion 处理时检查 |
| update queue 与 close 并发死锁 | 中 | wepoll 用单 critical section，xray 用 worker-bound 模式 + deferred free |
| AFD_POLL 不监听 connect 完成 | 低 | `AFD_POLL_CONNECT_FAIL` + `AFD_POLL_SEND` 可联合表达 connect 完成（参考 wepoll mapping） |
| close 时异步 cancel 长尾 | 低 | xray 已有 `pd->closing` + deferred free，与现有平台一致 |

---

## 8. 不做什么（明确边界）

- ❌ **不做 I/O Rings**（Windows 11 22H2+ 的 io_uring 等价）。可用版本太窄、API 不全、生态零。等 Windows 10 EOL（2025-10）后再评估
- ❌ **不做 RIO**（Registered I/O）。难用、性能优势仅在 ≥10 Gbps 网卡 + 极端低延迟场景体现，与 xray 定位不符
- ❌ **不做 select fallback**。AFD 失败直接报错，不增加代码分支
- ❌ **不做 per-worker IOCP**。共享 IOCP 已经多线程友好，等 perf 数据驱动再做
- ❌ **不修 stdlib 任何代码**。这是 backend 替换，对上不可见
- ❌ **不引入 wepoll 库**。手抄需要的部分 ~600 行 C，避免依赖

---

## 9. 设计审查清单

实施前需要用户确认的开放问题：

1. **`os_poll_iocp.h` 的修复时机**：现在顺手修（Phase 5）还是另开 ticket？
   - 推荐：另开（避免本分支膨胀），但本分支 commit 注释中明确指出
2. **是否在 Phase 1 就改 CMake 加 `windows-latest` CI**？
   - 推荐：是。让 Phase 1 的代码改动立即被 Windows 编译验证，不要等到 Phase 4
3. **是否需要在 `XrPollDesc` 上做 size assertion**（防止 Windows 字段意外加大结构）？
   - 推荐：是，加 `_Static_assert(sizeof(XrPollDesc) <= 512)` 之类
4. **基础设施（`xr_atomic.h` / `xr_mutex.h`）已经覆盖 Windows 吗？**
   - 已确认：`os_thread.h` 在 Windows 上用 `SRWLOCK` + `CONDITION_VARIABLE`，`stdatomic.h` 由 `/std:c11 /experimental:c11atomics` 启用 ✓

---

## 10. 实施起点

**第一刀**：Phase 1，从 Mac 上开始。

具体步骤（每步独立可 commit）：
1. 删 `#error`，让 `xnetpoll.c` 在 Windows 上空实现编译过
2. 重写 `netpoll_iocp.c`：完整 ops 表 + 占位实现（all return -1）
3. `netpoll_default_ops()` 加 Windows 分支
4. macOS 编译检查：`cmake --build build`
5. `XrPollDesc` 加 Windows 平台字段（条件编译）
6. CI 配置：`.github/workflows/ci.yml` 加 `windows-latest` matrix（仅 build，不跑 test）
7. push → 让 GitHub Actions 在 Windows 上验证编译

之后逐 Phase 推进，每个 Phase 一个 commit，commit message 严格遵守 main.md 的"事实+原因"原则，不出现 Phase X 字样。

---

## 11. 开发工作流（macOS + Parallels Win11 ARM）

主开发机是 Apple Silicon macOS，IDE 和源码主副本都在 macOS 上；Parallels Win11 ARM VM 只承担 **build/test runner** 角色。三脚本 + SSH + rsync 串起来。

### 11.1 一次性配置

**VM 内**：
1. 启用 OpenSSH Server（Win11 自带，PowerShell 管理员模式）：
   ```powershell
   Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
   Start-Service sshd
   Set-Service -Name sshd -StartupType Automatic
   New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' `
     -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
   ```
2. 把 macOS 的 `~/.ssh/id_ed25519.pub` 加到 VM 的 `C:\Users\<你>\.ssh\authorized_keys`（注意 ACL：仅当前用户可读，否则 sshd 拒绝 key 认证）
3. 装工具链（用 `winget`）：
   ```powershell
   winget install --id Microsoft.VisualStudio.2022.BuildTools --override `
     "--quiet --wait --add Microsoft.VisualStudio.Workload.VCTools `
      --add Microsoft.VisualStudio.Component.VC.Tools.x86.x64 `
      --add Microsoft.VisualStudio.Component.VC.Tools.ARM64 `
      --add Microsoft.VisualStudio.Component.Windows11SDK.22621"
   winget install Kitware.CMake
   winget install Ninja-build.Ninja
   winget install Git.Git
   winget install Python.Python.3.12
   winget install Microsoft.Vcpkg          # 可选：用于装 zlib 等依赖
   ```
   ARM64 host 配 `VC.Tools.x86.x64` 后即可**交叉编译出真 x64 binary**（运行时走 Win11 ARM 的 x64 emulator）。
4. 准备工作目录：
   ```powershell
   mkdir C:\workspace\xray
   mkdir C:\workspace\xray-build
   ```
5. （可选）装 zlib：`vcpkg install zlib:x64-windows-release`，并把 vcpkg 工具链作为 `XRAY_WIN_CMAKE_EXTRA` 传入构建脚本

**macOS 内**：
- `~/.ssh/config` 加 host alias：
  ```
  Host xray-win
      HostName 10.211.55.3       # Parallels 默认 NAT 下 VM 实际 IP
      User <你的-windows-用户名>
      IdentityFile ~/.ssh/id_ed25519
      ServerAliveInterval 30
  ```
- 测试：`ssh xray-win whoami`

### 11.2 三脚本（已落在 `scripts/`）

| 脚本 | 作用 |
|---|---|
| `scripts/win_sync.sh` | rsync 单向同步源码到 `C:\workspace\xray`（删除 build/、.git/、缓存） |
| `scripts/win_build.sh` | 同步 + 远程 `cmake -G Ninja` + `ninja`，先 `call vcvarsamd64_arm64.bat` 拉 MSVC 环境 |
| `scripts/win_test.sh`  | 远程 `ctest --output-on-failure`，把 `LastTest.log` scp 回 `/tmp/win_ctest.log` |

环境变量覆盖（可选）：

- `XRAY_WIN_HOST`（默认 `xray-win`）
- `XRAY_WIN_SRC`（默认 `C:/workspace/xray`）
- `XRAY_WIN_BUILD`（默认 `C:/workspace/xray-build`）
- `XRAY_WIN_BUILD_TYPE`（默认 `Release`）
- `XRAY_WIN_CMAKE_EXTRA`（额外 cmake 参数，如 `-DCMAKE_TOOLCHAIN_FILE=...`）
- `XRAY_WIN_CTEST_ARGS`（默认 `--output-on-failure --timeout 120`）

### 11.3 日常迭代循环

```bash
# 在 macOS Windsurf 改代码 → 切到终端
./scripts/win_build.sh   # 同步 + 编译，看链接/编译错误
./scripts/win_test.sh    # 跑测试，看运行时错误
```

正常情况下 5-30 秒一次循环，比推 GitHub Actions（5-10 min）快一个数量级。

### 11.4 纪律

- **VM 里永不 `git commit`**：源码改动只在 macOS 上做，rsync 单向推送，避免行尾/CRLF 污染
- **Build dir 必须放 VM 本地 NTFS**：不要用 `\\Mac\Home` 共享路径做 build（UNC 路径 + 跨 hypervisor IO 性能差，CMake 个别 generator 还出怪问题）
- **Defender 排除 `C:\workspace`**：实时扫描拖慢 build 30-50%
- **路径浅一些**：Windows 仍有 260 字符 MAX_PATH 限制，CMake 深路径偶尔踩坑

### 11.5 对等性边界

| 验证场景 | Parallels Win11 ARM | 备注 |
|---|---|---|
| 编译/链接错误（`open_memstream` 缺失、符号未定义等） | **100% 对等** | MSVC 在 ARM64 host 上生成 x64 binary 的行为与实体 x64 host 一致 |
| 一般 Windows 平台 API（IOCP、文件、socket、TLS） | ~90% | x64 emulator 跑 .exe 行为基本一致，浮点/SIMD/时序敏感测试可能微小差异 |
| x64 JIT 后端正确性 | ~50% | emulator 行为差异；权威验证仍依赖 GitHub Actions x64 runner |
| 性能数字 | **不可比** | emulator 有 1.5-3x 开销 |

**结论**：Parallels 解决 90% 日常迭代，**关键提交前仍推 GitHub Actions**（真 x64 MSVC）做最终验证。

---

## 12. 参考代码（已克隆到 `/Users/xuxinglei/workspace/参考代码/`）

| 仓库 | 关键路径 | 用途 |
|---|---|---|
| `wepoll/` | `wepoll.c`（单文件） | **核心参考**：AFD/IOCTL/状态机 |
| `mio/` | `src/sys/windows/{afd,iocp,selector}.rs` | wepoll 思路的 Rust 现代封装 |
| `libuv/` | `src/win/poll.c` + `core.c` + `async.c` | MSAFD 路线 + IOCP 主循环 |
| `go/` | `src/runtime/netpoll_windows.go` | 真完成模型（不抄，仅理解差异） |
| `libevent/` | `evthread_win32.c` + `bufferevent_async.c` | 入门理解 IOCP |
