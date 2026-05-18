# IO runtime 重构实施方案

## 背景

Xray 当前已经有较完整的网络 I/O 基础：`os_net` 统一平台 socket，`xnetpoll` 负责 kqueue/epoll/io_uring 风格事件收集，`xyieldable` 能把 C 函数挂起为 coroutine continuation，`stdlib/net/http/ws` 已经部分使用 yieldable I/O。

但源码中仍存在两条语义不同的等待路径：

- 真正的 coroutine 挂起：`xr_yield_for_io` 保存 continuation，返回 `XR_CFUNC_BLOCKED`，worker 继续执行其他 coroutine。
- 线程阻塞等待：`xr_netpoll_block` 使用 `xr_cond_wait` 阻塞当前 worker thread，调用方包括 `xsocket` 的 blocking-style API 以及部分 `stdlib/net/io.c` 路径。

文件 I/O 也仍是同步 `fopen/fread/fwrite/fclose`，`stdlib/common_io.h` 和 `stdlib/io/io.c` 都明确说明它会阻塞 worker thread。

本方案目标是把 IO 体系收敛成一个统一、可维护、性能正确的 runtime 子系统。

## 开发原则

Xray 是全新语言，没有外部兼容包袱。本次 IO runtime 重构遵循以下原则：

- 不保留旧接口，不做兼容层。
- 直接采用最佳设计，而不是给现有问题打补丁。
- 所有可能等待外部资源的操作都必须能协作挂起 coroutine。
- 同步阻塞 API 只允许存在于明确的内部 bootstrap/tooling 边界，不进入 worker 热路径。
- 脚本层 API、runtime export、analyzer、LSP、文档必须来自同一个事实源。
- 资源生命周期必须由 typed native handle 管理，不再用 Json 模拟 fd handle。
- 错误必须可诊断，不再用 `null/false/-1` 吞掉 errno、timeout、TLS、DNS、closed 等原因。

## 目标架构

```text
脚本层 stdlib
  ├─ io.File / io.Dir / io.Path ops
  ├─ net.Conn / net.Listener / net.UdpSocket
  ├─ http / ws / cluster
  ↓
stdlib native binding
  ├─ 只暴露 yieldable 或 fast API
  ├─ typed native handles
  ├─ typed error / exception
  ↓
IO runtime
  ├─ XrIoRuntime：netpoll、async pool、DNS、TLS policy、handle registry
  ├─ XrIoOp：read/write/connect/accept/dns/fs/exec 等统一 operation
  ├─ XrIoWait：netpoll wait、async job wait、timeout/cancel wait
  ↓
runtime/coro
  ├─ xyieldable continuation 协议
  ├─ worker scheduler
  ├─ timer wheel
  ├─ cancellation/deadline
  ↓
OS abstraction
  ├─ os_net
  ├─ os_fs / os_dir
  ├─ platform netpoll backend
  └─ platform process/fs async backend
```

核心设计：

1. `XrIoRuntime` 归属于 `XrRuntime` 或 `XrayIsolate`，禁止 `g_io`、`g_dns`、thread-local isolate 作为长期架构。
2. 所有 stdlib 网络和文件 API 统一通过 `XrIoOp` 或等价 operation state machine 进入 runtime。
3. netpoll-ready、async-ready、timer-ready、cancel-ready 都统一恢复 coroutine，并通过 `XrResumeStatus` 传递恢复原因。
4. C 层不再提供容易误用的 blocking-style socket API；需要同步能力时使用 `_sync` 命名并限制在非-worker 上下文。

## 现状问题清单

### 问题 1：同一 IO 子系统内同时存在 coroutine 挂起和 worker 线程阻塞

`xr_yield_for_io` 是正确模型；`xr_netpoll_block` 会阻塞 worker thread。二者混用会让 API 名称中的 coroutine-friendly 语义不可靠。

影响：

- 一个慢连接可以占住 worker thread。
- `xsocket` 看起来像 coroutine-safe，实际部分路径是 cond wait。
- 上层模块难以判断某个 API 是否真正让出调度权。

目标：

- 删除 worker 热路径对 `xr_netpoll_block` 的依赖。
- 所有 socket read/write/connect/accept/wait 都改为 yieldable continuation。
- 保留同步等待只用于测试、bootstrap 或非-runtime 工具，并通过命名和断言隔离。

### 问题 2：`stdlib/net/io.c` 有进程全局状态

当前 `g_io` 保存 owned netpoll、TLS context、initialized flag；`tls_isolate` 用 thread-local 保存当前 isolate。

影响：

- 多 isolate 之间共享隐式状态。
- 生命周期和 shutdown 次序不清晰。
- runtime 已经有 `runtime->netpoll`，stdlib 还能创建 owned netpoll，存在双事实源。

目标：

- `XrIoRuntime` 成为 IO 状态唯一所有者。
- DNS/TLS/netpoll/async pool/deadline policy 都挂到 runtime 或 isolate。
- stdlib 不直接拥有第二套 netpoll。

### 问题 3：DNS cache miss 仍可同步阻塞

`xr_dns_resolve()` cache miss 会调用同步 resolver。虽然已有 `xr_dns_resolve_on_async()`，但脚本 `dial/lookup/sendTo` 路径需要统一接入。

目标：

- DNS miss 默认走 async resolver operation。
- DNS cache、TTL、family preference 归属 `XrIoRuntime`。
- `lookupAll` 和 Happy Eyeballs 所需地址列表成为一等能力。

### 问题 4：文件 I/O 全部同步阻塞

`stdlib/io/io.c` 和 `stdlib/common_io.h` 使用同步 stdio / filesystem syscall。

影响：

- 大文件、慢盘、网络挂载、深目录递归会阻塞 worker。
- data format 的 `parseFile/writeFile` 也继承此问题。

目标：

- 文件 read/write/stat/copy/removeAll/readDirRecursive 等 slow operation 进入 async pool。
- 脚本层 `io.readFile` 等默认就是 yieldable，不新增兼容 async 后缀。
- 内部同步 helper 改名为 `_sync` 并仅限非-worker 或 async thread 使用。

### 问题 5：handle 用 Json 暴露 fd 细节

`net` 现有 handle 字段和 analyzer/LSP 声明存在漂移；脚本可观察内部 `fd/type/tls` 字段。

目标：

- 引入 opaque native handle：`NetConn`、`NetListener`、`UdpSocket`、`File`、`Dir`。
- handle 持有 fd、poll desc、deadline、TLS state、closed flag、owner runtime。
- handle finalizer 和显式 `close()` 走同一释放路径。
- 脚本不能依赖内部 fd 字段；需要诊断时提供受控 `debugFd()` 或测试专用 API。

### 问题 6：错误模型不可诊断

当前大量 API 使用 `null/false/-1` 表示失败。

目标：

- I/O 失败默认抛 typed error，例如 `IoError`、`NetError`、`DnsError`、`TlsError`、`TimeoutError`、`ClosedError`。
- 错误对象至少包含 `code`、`message`、`operation`、`path/host/fd`、`errno/nativeCode`。
- 可预期探测类 API 如 `exists()` 返回 bool，但不得吞掉权限错误或路径格式错误。

### 问题 7：deadline、timeout、cancel 没有统一契约

部分 connect 有 timeout，read/write 经常无限等待，HTTP/WS/cluster 各自传递策略。

目标：

- 每个 `XrIoHandle` 有默认 deadline policy。
- 每个 operation 可覆盖 timeout。
- timeout 和 cancellation 都作为 resume status 进入 continuation。
- close handle 必须唤醒所有等待 coroutine。

### 问题 8：stdlib metadata 漂移

runtime export、`XR_DEFINE_BUILTIN`、analyzer generated metadata、LSP symbols、文档经常不一致。

目标：

- IO 相关模块从 manifest/descriptor 生成 metadata。
- descriptor 包含 signature、yieldable/fast/slow、handle type、error behavior、feature gate。
- 加一致性测试锁住 runtime/analyzer/LSP 三方。

## 实施计划

### 阶段 1：确定 IO runtime 单一所有权

交付：

- 新增或整理 `XrIoRuntime` 设计，明确归属 `XrRuntime` 或 `XrayIsolate`。
- 移除 stdlib 层 owned netpoll 的长期设计。
- 明确 `netpoll`、`async_pool`、DNS cache、TLS default context、handle registry 的生命周期。
- 增加 worker 上下文断言：禁止在 worker 热路径调用未标注的阻塞等待。

验收：

- IO 初始化和 shutdown 只有一条权威路径。
- 多 isolate/runtime 的 IO 状态不会隐式共享，除非显式配置 shared cache。
- `g_io`、`tls_isolate` 有清晰替代方案和删除清单。

### 阶段 2：统一 yieldable IO operation 协议

交付：

- 定义 `XrIoOp` 或等价 state machine 基础结构。
- read/write/connect/accept/wait 全部通过 continuation 恢复。
- netpoll wait 不再需要阻塞 worker thread。
- `XrResumeStatus` 覆盖 `IO_READY`、`TIMEOUT`、`CANCELLED`、`CLOSED`、`ERROR`。

验收：

- `xsocket` 不再作为 stdlib 热路径的 blocking facade。
- `xr_netpoll_block` 从 worker 热路径移除，或只保留在 `_sync` 调试/测试 API。
- 慢 socket 测试证明一个阻塞连接不会阻塞同 worker 的其他 coroutine。

### 阶段 3：重建网络 handle 与 stdlib/net API

交付：

- 使用 opaque native handle 替代 Json fd handle。
- `dial/listen/accept/read/write/close/lookup/lookupAll/udpBind/sendTo/recvFrom` 全部重写为统一 yieldable operation。
- TLS handshake 使用同一 wait/read/write continuation，不走阻塞路径。
- DNS miss 接入 async resolver。

验收：

- 脚本层不再读取 `handle.fd/type/tls`。
- `net.read/write/accept/dial/lookup` 的 timeout 行为一致。
- GC 压力 + slow socket write/read 测试通过，跨 yield 不保存悬空 raw pointer。

### 阶段 4：文件 IO 改为默认 yieldable

交付：

- `io.readFile/writeFile/appendFile/copyFile/readDirRecursive/removeAll/stat` 等 slow operation 接入 async pool。
- `common_io` 的同步 helper 改为 async thread 内部 helper，不在 worker 上直接调用。
- data format `parseFile/writeFile` 统一通过 IO runtime。
- 引入 `File` handle 和 streaming API：`open/read/write/seek/close`。

验收：

- 大文件读写不阻塞 worker。
- data format file API 不阻塞 worker。
- short read、ferror、fclose failure 都能转成 typed error。

### 阶段 5：统一错误、deadline、cancel

交付：

- IO 错误对象和错误码表。
- deadline/cancel/close 唤醒协议。
- `close()` 与 finalizer 共享释放路径。
- HTTP/WS/cluster 统一使用底层 deadline 和 error。

验收：

- timeout、cancel、closed、dns failure、tls failure、permission denied、disk full 都有可区分测试。
- close 正在等待的 conn/file/listener 会唤醒 coroutine，不产生悬挂等待。

### 阶段 6：metadata、测试和删除旧路径

交付：

- IO/stdnet descriptor 生成 runtime/analyzer/LSP 文档片段。
- 删除旧 blocking facade、Json handle、过期 fast fd API 声明。
- 增加 execution class 表：`fast`、`yieldable`、`async_worker`、`process_side_effect`。

验收：

- runtime export、analyzer、LSP 完全一致。
- `ctest --output-on-failure` 通过。
- 网络回归、HTTP/WS 回归、文件 IO slow path 测试通过。
- 架构检查通过：低层不 include 高层，stdlib 不绕过 runtime IO 契约。

## 推荐新/调整文件

候选拆分：

```text
src/coro/xio_runtime.h/.c        IO runtime owner and lifecycle
src/coro/xio_op.h/.c             yieldable IO operation state machine
src/coro/xio_handle.h/.c         native handle lifecycle
src/coro/xio_error.h/.c          typed IO error mapping
src/coro/xio_fs.h/.c             async filesystem operations
src/coro/xio_dns.h/.c            async DNS resolver context
stdlib/net/net.c                 thin script binding only
stdlib/io/io.c                   thin script binding only
```

要求：

- `src/coro` 可依赖 `src/os`、`src/base`、runtime value 基础类型，但不能反向依赖 stdlib。
- `stdlib/*` 只做 binding，不拥有 netpoll、DNS cache 或 async pool。
- OS 差异必须收敛在 `src/os/*` 或 IO runtime backend，不能散落到脚本 binding。

## 测试矩阵

必须覆盖：

- netpoll read/write ready、timeout、close wake、cancel wake。
- connect timeout、DNS failure、TLS handshake failure。
- slow reader / slow writer 下的 coroutine fairness。
- 多 worker 下 fd 绑定、迁移、deadline rebind。
- 大文件 read/write/copy 不阻塞其他 coroutine。
- directory recursive slow path 不阻塞其他 coroutine。
- data format `parseFile/writeFile` 不阻塞 worker。
- GC 压力下跨 yield buffer/handle 生命周期。
- runtime/analyzer/LSP metadata 一致性。

推荐命令：

```bash
ctest --output-on-failure
scripts/run_regression_tests.sh
tests/network/run_network_tests.sh
```

若新增 ASAN 配置，增加：

```bash
ctest --output-on-failure -R "io|net|http|ws|coro"
```

## 完成标准

本任务完成时应满足：

- worker 热路径没有未标注的同步阻塞 IO。
- 网络 IO、文件 IO、DNS、TLS、HTTP、WS 使用同一 IO runtime 契约。
- stdlib 脚本 API 使用 typed native handle，不暴露 Json fd 结构。
- 所有 IO 错误可诊断。
- timeout、cancel、close 语义一致。
- runtime export、analyzer、LSP、文档由单一事实源生成或校验。
- 旧接口和旧注释被删除，不留下兼容层。
