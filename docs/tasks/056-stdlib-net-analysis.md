# stdlib/net 分析与优化建议

## 模块职责

`stdlib/net` 是标准库网络栈的底层能力集合，向脚本层暴露 TCP、TLS、UDP 和 DNS API，并向上层 `http`、`ws`、`cluster` 提供统一的 I/O、TLS、DNS、连接池和错误码基础。

当前模块实际包含多层职责：

- 脚本层 handle-based API：`dial/listen/accept/read/write/close/fd/lookup/hasTLS/dialTLS/upgradeTLS/udpBind/sendTo/recvFrom`
- C 层 I/O API：`xr_io_connect/read/write/listen/accept` 等 coroutine-friendly 封装
- C 层 TLS API：OpenSSL context、client/server handshake、ALPN、mTLS、session cache、OCSP
- C 层 DNS API：LRU cache、TTL、IPv4/IPv6、round-robin、async pool 接口
- C 层连接池：HTTP keep-alive TCP/TLS connection pool
- 统一错误码：`XrNetError` 被 net/tls/http/ws/cluster 共用

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/net/net.h` | net 模块公开类型和 C API 声明 |
| `stdlib/net/net.c` | 脚本绑定、handle-based TCP/TLS/UDP/DNS API、部分 UDP C API |
| `stdlib/net/io.h` | coroutine-friendly TCP/TLS I/O 抽象 |
| `stdlib/net/io.c` | netpoll 集成、TCP/TLS connect/read/write/listen/accept |
| `stdlib/net/tls.h` | TLS context/connection API、ALPN/mTLS/session/OCSP 声明 |
| `stdlib/net/tls.c` | OpenSSL 实现与 TLS disabled stub |
| `stdlib/net/dns.h` | DNS resolver API |
| `stdlib/net/dns.c` | DNS LRU cache、TTL、round-robin、async resolve 接口 |
| `stdlib/net/conn_pool.h` | HTTP connection pool API |
| `stdlib/net/conn_pool.c` | per-host keep-alive pool、idle eviction、TLS connection reuse |
| `stdlib/net/xneterror.h` | 统一网络错误码 |
| `stdlib/net/xneterror.c` | 错误码字符串 |
| `stdlib/net/xnetbuf.*` | reusable network buffer pool |
| `tests/regression/10_stdlib/1430_net_basic.xr` | DNS/API 基础回归测试 |
| `tests/regression/10_stdlib/1431_net_extended.xr` | API 存在性和 DNS 类型测试 |

## 当前脚本 API

实际 `xr_load_module_net()` 导出：

| API | 导出方式 | 当前行为 |
|---|---|---|
| `dial(host, port, timeout?)` | yieldable | 非阻塞 TCP connect，返回 Json handle 或 null |
| `listen(port, backlog?)` | normal | 创建 TCP listener handle |
| `accept(listener)` | yieldable | accept 新连接，返回 connection handle |
| `read(conn, maxlen?)` | yieldable | TCP/TLS read，返回 string/null |
| `write(conn, data)` | yieldable | TCP/TLS write，返回写入字节数或 -1 |
| `close(handle)` | normal | 关闭 connection/listener/UDP handle |
| `fd(handle)` | normal | 返回 fd，closed 后为 -1 |
| `lookup(hostname)` | normal | 返回首个解析 IP string/null |
| `hasTLS()` | normal | 返回构建是否启用 TLS |
| `dialTLS(host, port, timeout?)` | yieldable，TLS enabled only | TCP connect 后 TLS handshake |
| `upgradeTLS(conn, hostname)` | yieldable，TLS enabled only | 将已有 TCP handle 升级为 TLS |
| `udpBind(port, addr?)` | normal | 创建 UDP socket handle |
| `sendTo(handle, data, host, port)` | yieldable | UDP sendto，返回字节数或 -1 |
| `recvFrom(handle, maxlen?)` | yieldable | UDP recvfrom，返回 `{data, addr}` 或 null |

## 当前架构优点

- 脚本层使用 handle-based API，避免直接把 raw fd 暴露为唯一抽象。
- `dial/accept/read/write/sendTo/recvFrom` 都是 yieldable，能与 coroutine scheduler 和 netpoll 协作。
- TLS read/write/handshake 使用 `*_try` 接口处理 WANT_READ/WANT_WRITE，适合非阻塞协程。
- DNS 有 256-entry LRU cache、5 分钟 TTL 和 round-robin 地址选择。
- TCP listener 默认尝试 IPv6 dual-stack，再 fallback 到 IPv4。
- TLS client 默认加载系统 CA，启用 peer verify 和 hostname verification。
- 统一 `XrNetError` 为 http/ws/cluster 共享错误分类奠定基础。
- `net.read` 使用 per-coroutine reusable I/O buffer，减少热路径分配。
- `net.close` 会先清理 netpoll fd map，避免 fd reuse 后继承 stale poll descriptor。

## API 与工具链漂移

### 问题 1：runtime 导出与 analyzer 不一致

`XR_DEFINE_BUILTIN` 声明和 generated analyzer 包含 23 个函数，其中包含：

```text
waitIO closeFd listenTcp acceptFast readFast writeFast getReadBuffer connectStart connectFinish
```

但 `xr_load_module_net()` 实际只导出 handle-based API 和 UDP/TLS/DNS 子集，没有导出上述 fast fd API。

影响：

- 静态分析认为函数存在，运行时实际不可调用。
- 用户可能基于补全写出运行时报错的代码。
- 低层 fd API 是否仍是正式接口不明确。

建议：

- 如果 fast fd API 已废弃，从 `XR_DEFINE_BUILTIN`/generated analyzer 中删除。
- 如果仍需保留，必须恢复 runtime export 并补齐测试。
- 明确只有 handle-based API 是稳定脚本 API。

### 问题 2：LSP 暴露严重不足且签名漂移

LSP 的 `net_symbols` 只有：

```text
dial listen accept close lookup
```

缺失：

```text
read write fd hasTLS dialTLS upgradeTLS udpBind sendTo recvFrom
```

并且 LSP 把 `lookup` 写成 `Array<string>`，实际 `net.lookup()` 返回单个 `string?`。

建议：

- 由 builtin metadata 自动生成 LSP，避免手写漂移。
- 将 LSP 签名与实际 runtime export 对齐。

### 问题 3：handle 类型字段声明与实际值不一致

builtin handle 声明：

```text
Connection { fd: int, type: string, tls: bool }
Listener   { fd: int, type: string, port: int }
UdpSocket  { fd: int, type: string }
```

实际 Json 字段中 `type` 是整数枚举：

```text
0 = conn, 1 = tls conn, 2 = listener, 3 = udp
```

影响：

- 静态类型提示与实际运行值不同。
- 用户如果读取 `handle.type`，会得到 int 而不是 string。

建议：

- 将 handle 声明改为 `type: int`。
- 或把 runtime handle 改成 string，但这会牺牲当前紧凑表示。
- 更好的方式是引入 opaque native handle，脚本不依赖内部字段。

## DNS 边界

### 问题 4：`net.dial` 仍可能在 DNS miss 时阻塞 worker

`net_tcp_connect()` 调用 `xr_dns_resolve()`。该函数虽然有 cache，但 cache miss 会走同步 `getaddrinfo()`。

影响：

- 大量不可达域名或慢 DNS 会阻塞 worker。
- `dial` 是 yieldable，但 DNS 阶段尚未真正 yield。

建议：

- 在脚本 `dial/dialTLS/sendTo/lookup` 中使用 `xr_dns_resolve_on_async()`。
- 或引入 resolver pool，把 DNS miss 作为 async task。
- 将 DNS timeout、cache TTL、family preference 暴露为配置。

### 问题 5：`net.lookup()` 只返回一个地址

C API `xr_dns_lookup()` 最多返回 8 个地址，但脚本 `net.lookup()` 只返回第一个 IP string。

影响：

- 无法在脚本层做 failover、Happy Eyeballs 或多地址诊断。
- LSP 反而声明为 `Array<string>`，进一步混淆。

建议：

- 增加 `lookupAll(hostname): Array<string>`。
- 保留 `lookup()` 返回首地址，或统一改为数组但需同步测试和文档。

### 问题 6：DNS cache 是进程全局状态

`dns.c` 使用全局 `g_dns`，所有 isolate 共享 cache、TTL 和 round-robin index。

影响：

- 多 isolate 间共享解析结果与轮询状态。
- 测试和长期运行服务之间可能互相影响。
- 不利于每个 isolate 独立配置 DNS policy。

建议：

- 短期明确 DNS cache 是 process-global。
- 长期将 DNS context 下沉到 isolate/runtime，保留可选 global shared cache。

## TCP 与 I/O 边界

### 问题 7：`net.read/write` 没有可配置操作 timeout

`dial` 有 connect timeout，但 `read/write` 在 WANT_READ/WANT_WRITE 时使用 `-1` 无限等待。

影响：

- peer 挂起时 coroutine 可永久等待。
- 上层很难实现统一 deadline。

建议：

- handle 增加默认 read/write timeout。
- `read(conn, maxlen?, timeout?)` 和 `write(conn, data, timeout?)` 支持可选 timeout。
- 或提供 `setTimeout(handle, ms)`。

### 问题 8：`net.write` 跨 yield 保存 raw `XrString` 指针

`NetWriteHandleState` 保存 `state->data = XR_STRING_CHARS(data)`，并注释认为 XrString immutable 且 yielded 期间 GC 不运行。这个约束需要由 runtime 明确保证，否则跨 yield 的 raw pointer 有生命周期风险。

建议：

- 如果 runtime 确认 yield 期间不会移动/回收该字符串，写入函数 doc comment 明确不变量。
- 否则在 would-block 前复制未写完数据到 owned buffer。
- 添加 GC 压力 + slow socket write 测试。

### 问题 9：fallback buffer 分配后可能泄漏

`net_read_handle_yieldable()` 在无 coroutine 时 fallback `xr_malloc(max_len)`，但 `NetReadHandleState` 只记录 `buf`，没有记录 ownership。`net_read_handle_step()` 完成时只释放 state，不释放 fallback buffer。

影响：

- 非 coroutine 调用路径可能泄漏 read buffer。
- 当前脚本 yieldable 调用通常有 coroutine，但 C API 或特殊路径仍有风险。

建议：

- 在 state 中增加 `buf_owned`。
- 或要求该 API 必须运行在 coroutine，并无 coroutine 时直接返回错误。

### 问题 10：`net.close` 对 UDP/listener 也执行 `shutdown(fd, SHUT_WR)`

`net_close_fd()` 无差别执行：

```text
shutdown(fd, SHUT_WR);
close(fd);
```

对 listener/UDP 来说 `shutdown` 语义不必要，可能产生无意义 errno。

建议：

- 按 handle type 区分 close 策略。
- 对 TCP connection 做 shutdown；listener/UDP 只 close。

## TLS 边界

### 问题 11：脚本层无法配置 TLS policy

C 层支持：

- verify on/off
- custom CA bundle
- ALPN
- mTLS client cert
- session cache
- OCSP stapling

脚本层 `dialTLS/upgradeTLS` 只有 hostname，无法传 options。

建议：

- 增加 `dialTLS(host, port, options?)`。
- options 至少覆盖：`timeout`, `verify`, `caFile`, `alpn`, `clientCert`, `clientKey`, `serverName`。
- 默认仍保持 verify=true。

### 问题 12：TLS fd-indexed storage 是进程全局

`net.c` 在 TLS enabled 时维护全局 `g_tls_conns[fd]`，按 fd 映射 `XrTlsConn*`。

风险：

- fd reuse 依赖 close path 必须正确清理。
- 多 isolate 共用同一 fd namespace，错误 handle 或 stale handle 可能跨 isolate 影响。
- 连接所有权不在 handle 内部，debug 和生命周期分析困难。

建议：

- 将 TLS connection pointer 放入 native handle / per-isolate registry。
- 短期对 stale handle、double close、fd reuse 增加测试。

### 问题 13：TLS handshake timeout 没有复用用户传入 timeout

`dialTLS` TCP connect 使用用户传入 timeout，但 TLS handshake loop 中固定使用 30000ms。`upgradeTLS` 也固定 30000ms。

建议：

- `NetDialTLSState` 保存 timeout_ms。
- `upgradeTLS` 支持 timeout 参数。
- 所有 wait 使用同一 deadline，而不是每次 WANT_READ/WANT_WRITE 重置 30s。

### 问题 14：TLS disabled 时 analyzer 仍声明 TLS API

`XR_DEFINE_BUILTIN` 总是声明 `dialTLS/upgradeTLS`，但 loader 只有 `#ifdef XR_ENABLE_TLS` 时才导出。

影响：

- 无 TLS 构建下 analyzer 认为 API 存在，运行时不存在。

建议：

- 即使 TLS disabled，也导出 stub，返回 null，并由 `hasTLS()` 告知能力。
- 或 generated builtin 支持条件编译能力标记。

## UDP 边界

### 问题 15：UDP API 没有 timeout 参数

`sendTo` EAGAIN 后固定等待 5000ms，`recvFrom` EAGAIN 后无限等待。

建议：

- `sendTo(handle, data, host, port, timeout?)`。
- `recvFrom(handle, maxlen?, timeout?)`。

### 问题 16：UDP 数据以 string 表示，二进制语义不清

`recvFrom` 把 datagram data 构造成 string，`sendTo` 也只接受 string。

影响：

- UDP 通常承载二进制，string API 可能和 UTF-8/字符串 intern 语义冲突。
- 当前 `xr_string_intern()` 用于接收数据，二进制数据如果含 NUL 或大量不同 payload，会给 intern table 造成压力。

建议：

- 支持 typed `Array<u8>` / Bytes。
- `recvFrom` 对二进制返回 byte array，text 作为便利包装。
- 避免把网络 payload intern 到全局字符串表。

### 问题 17：UDP bind 端口 0 不返回实际端口

`udpBind(0)` 让 OS 分配端口，但 handle 只有 fd/type/tls，没有 local port 字段。

建议：

- 对 UDP handle 增加 `port` 或提供 `localAddr(handle)`。
- listener `listen(0)` 也应返回实际绑定端口，而不是传入的 0。

## 连接池边界

### 问题 18：connection pool 属于 HTTP 但位于 net 模块，职责边界不清

`conn_pool.*` 文件名和实现都聚焦 HTTP keep-alive，但放在 `stdlib/net`。

合理性：

- pool 复用 TCP/TLS 连接，是网络层通用设施。

问题：

- 当前 key 包含 `host:port:https`，API 也直接使用 `is_https`，更像 HTTP client 私有实现。
- 没有暴露通用 pooling policy，也没有独立测试。

建议：

- 短期标注为 HTTP client internal dependency。
- 长期要么迁回 `stdlib/http`，要么泛化为 protocol-agnostic connection pool。

### 问题 19：连接池限制和 eviction 策略较固定

当前默认：

```text
per-host 6, hosts 64, idle 60s
```

建议：

- 允许 HTTP client 配置 max idle、max per host、idle timeout。
- 增加 idle eviction 和 closed socket reuse 测试。

## 错误模型

### 问题 20：脚本层错误信息过少

大多数失败只返回 null 或 -1，无法区分：

- DNS 失败
- connect timeout
- connect refused
- TLS verify failed
- read EOF
- read timeout
- write partial/error

建议：

- 增加 `lastError(handle?)` 或返回 `{ok, value, error}` 风格结果。
- 至少在 `dial/dialTLS/upgradeTLS/recvFrom` 失败时返回带 `error` 字段的 Json，或提供 `net.errorString(code)`。

### 问题 21：`XrNetError` C 层统一了，但脚本层未使用

`XrNetError` 已被 TLS/HTTP/WS 共用，但脚本 `net` API 没有把错误码暴露出来。

建议：

- 将 `XrNetError` 映射为脚本可见 enum/int/string。
- 上层 HTTP/WS 也复用同一错误报告结构。

## 全局状态与 isolate 边界

当前全局状态包括：

- `dns.c` 的 `g_dns`
- `io.c` 的 `g_io`
- `tls.c` 的 `tls_initialized`
- `net.c` 的 `g_tls_conns`、`g_tls_client_ctx`
- `net.c` 的 `shape_conn/shape_listener` 静态变量

风险：

- 多 isolate 之间共享网络配置、TLS context、DNS cache、shape 指针。
- `shape_conn/shape_listener` 注释说每 isolate rebuild，但变量是文件静态；如果多个 isolate 共存，后加载 isolate 会覆盖前一个 isolate 的 shape 指针。
- 脚本 handle 是普通 Json，fd 整数可能被用户构造或跨 isolate 传递。

建议：

- 为脚本 `net` 模块引入 per-isolate `XrNetContext`。
- 把 symbol/shape cache、TLS fd registry、默认 TLS context 放入 context。
- handle 改为 opaque native handle 或至少带 owner/isolate token。

## 测试覆盖

现有覆盖：

- `1430_net_basic.xr`：`lookup(localhost)`、invalid hostname 不崩溃、模块/API 存在。
- `1431_net_extended.xr`：core/TLS/UDP API 存在、`hasTLS()` 类型、lookup 多次调用类型。

主要缺口：

1. TCP loopback：listen/dial/accept/read/write/close。
2. connect timeout、connection refused、invalid port。
3. read/write timeout 与 peer half-close。
4. large write partial/EAGAIN。
5. TLS loopback：self-signed CA、hostname verify fail/pass、ALPN、mTLS。
6. TLS disabled build 的 API 行为。
7. UDP loopback：bind(0)、sendTo/recvFrom、binary payload、大 datagram、timeout。
8. DNS cache TTL、LRU eviction、round-robin、多地址返回。
9. `lookupAll` 如新增后的行为。
10. 多 isolate 并发加载 net，shape/static context 是否安全。
11. stale handle/double close/fd reuse。
12. 连接池 get/put/reuse/idle eviction/closed socket cleanup。
13. analyzer/LSP 与 runtime export 同步测试。
14. GC 压力下跨 yield read/write state 生命周期。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | analyzer 声明未导出的 fast fd API | 静态/运行时漂移 | 删除或恢复导出 |
| 高 | LSP 缺失大量 API 且 `lookup` 签名错误 | IDE 误导 | 自动生成 LSP |
| 高 | `type` handle 字段声明为 string，实际 int | 类型错误 | 修正 handle 声明 |
| 高 | DNS miss 同步 `getaddrinfo()` | yieldable API 仍可阻塞 worker | async resolver pool |
| 高 | `shape_conn/shape_listener` 为文件静态 | 多 isolate 潜在错用 shape | per-isolate context |
| 高 | TLS fd registry 进程全局 | fd reuse/跨 isolate 生命周期风险 | native handle/per-isolate registry |
| 中 | read/write/UDP recv timeout 缺失 | coroutine 可无限等待 | 增加 timeout/deadline |
| 中 | `net.write` 跨 yield 保存 raw string pointer | 依赖 runtime 不变量 | 复制或明确 GC 保证 |
| 中 | fallback read buffer 可能泄漏 | 特殊路径内存泄漏 | 增加 ownership 标记 |
| 中 | 脚本层 TLS policy 不可配置 | 不能配置 CA/ALPN/mTLS | `dialTLS` options |
| 中 | 脚本层错误只返回 null/-1 | 难以诊断 | 暴露 `XrNetError` |
| 中 | UDP payload 使用 interned string | 二进制与 intern 压力 | 支持 Bytes/Array<u8> |
| 低 | listener/UDP close 调用 shutdown | 语义不精确 | 按 handle type close |
| 低 | connection pool 边界偏 HTTP | 模块职责模糊 | 迁回 HTTP 或泛化 |

## 后续实施建议

建议优先顺序：

1. 统一 runtime export、`XR_DEFINE_BUILTIN`、generated analyzer、LSP。
2. 修正 `Connection/Listener/UdpSocket` handle 字段类型。
3. 引入 per-isolate `XrNetContext`，迁移 shape/symbol/TLS registry/default TLS context。
4. 将 DNS miss 接入 async resolver，避免 `dial/lookup/sendTo` 阻塞 worker。
5. 增加 read/write/UDP/TLS handshake deadline 语义。
6. 明确脚本错误模型，暴露 `XrNetError`。
7. 支持 TLS options：verify、CA、ALPN、client cert、serverName、timeout。
8. 支持 UDP/TCP binary payload 和 `lookupAll/localAddr`。
9. 审计并测试跨 yield raw pointer 生命周期。
10. 为 TCP/UDP/TLS/DNS/connection pool 增加真实 loopback 和负向测试。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
