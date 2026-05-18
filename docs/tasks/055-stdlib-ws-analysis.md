# stdlib/ws 分析与优化建议

## 模块职责

`stdlib/ws` 提供 WebSocket client/server 能力，覆盖 RFC 6455 的握手、frame 编解码、masking、ping/pong、close、fragmentation、UTF-8 校验，以及 permessage-deflate 压缩扩展。

当前模块还承担高性能 echo server、HTTP server upgrade 集成和脚本层连接对象管理：

- client：`connect/send/recv/recvData/close/ping/state/isOpen`
- server：`serve/echoServe/stopServer/isServerRunning`
- HTTP 集成：`xr_ws_upgrade_and_wrap()` 供 `http` 模块升级连接
- 压缩：`ws_deflate` 薄封装复用 `stdlib/compress/compress_zlib.c`

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/ws/ws.h` | WebSocket C API、状态、配置、消息结构、upgrade API |
| `stdlib/ws/ws.c` | RFC 6455 协议实现：URL parse、handshake、frame、mask、fragment、ping/pong、close、deflate |
| `stdlib/ws/ws_binding.c` | xray module binding、per-isolate context、连接表、yieldable send/recv/server |
| `stdlib/ws/ws_deflate.c` | permessage-deflate wrapper，调用 compress zlib stream API |
| `stdlib/ws/ws_deflate.h` | deflate wrapper 声明 |
| `stdlib/http/http_listen.c` | HTTP route 命中 WS upgrade 时调用 `xr_ws_upgrade_and_wrap()` |
| `tests/network/ws_*` | WebSocket 功能/并发/benchmark/interop 测试脚本 |
| `tests/network/ws_tests/*` | Python websockets interop 测试和 xray client 测试 |
| `tests/benchmarks/ws/*` | WebSocket benchmark 工具 |

## 当前脚本 API

实际 module loader 导出：

| API | 导出方式 | 当前行为 |
|---|---|---|
| `connect(url, options?)` | slow | 连接 ws/wss，返回 Json connection object |
| `send(conn, data, binary?)` | yieldable | 发送 text/binary frame |
| `sendData(conn, data, binary?)` | yieldable | 与 `send` 指向同一实现，高性能别名 |
| `recv(conn, timeout?)` | yieldable | 返回 `{data, binary}` 或 `{error}` |
| `recvData(conn, timeout?)` | yieldable | 直接返回 string/null，跳过 Json wrapper |
| `close(conn, code?, reason?)` | normal | 关闭连接并释放 registry slot |
| `ping(conn)` | normal | 发送 ping frame |
| `state(conn)` | normal | 返回 connecting/open/closing/closed |
| `isOpen(conn)` | normal | 判断 open |
| `serve(port, handler)` | yieldable | 启动脚本 handler server |
| `echoServe(port)` | yieldable | 启动纯 C echo server |
| `stopServer()` | normal | 停 server |
| `isServerRunning()` | normal | 查询 server 状态 |

## 当前架构优点

- client 和 server 协议核心都在 C 层实现，热路径少 VM dispatch。
- client connect 在 TCP connect 前设置 nonblocking，并通过 netpoll yield，避免长时间阻塞 worker。
- send/recv 是 yieldable，慢路径通过 `xr_yield_for_io()` 等待读写就绪。
- server 使用 coroutine-per-connection 模型，`echoServe` 提供纯 C 高性能路径。
- receive fast path 尽量不分配：rbuf 内 payload 可 inplace unmask，返回 embedded `last_msg`。
- 支持 text/binary，binary 接收时转换为 typed `Array<u8>`。
- 对 client/server masking 方向做了校验。
- close frame code 和 reason UTF-8 做了基本校验。
- text message 会做 UTF-8 校验。
- permessage-deflate 解压复用 `xr_inflate_bounded()`，使用 `max_message_size` 防 zip bomb。
- connection registry 是 per-isolate，带 mutex 和 ID free-list。

## API 与工具链漂移

### 问题 1：runtime 导出、builtin generated、LSP 三者不同步

实际 `xr_load_module_ws()` 导出 13 个函数，但 `xanalyzer_builtins_generated.h` 只有 7 个：

```text
connect send recv close ping state isOpen
```

缺失：

```text
recvData sendData serve echoServe stopServer isServerRunning
```

LSP 则包含：

```text
connect send recv close ping state isOpen hasError serve stopServer isServerRunning
```

其中 `hasError` 未在 loader 中导出；LSP 又缺失 `recvData/sendData/echoServe`。

影响：

- analyzer、LSP、runtime 对用户暴露不同 API。
- 用户可运行的函数在编辑器中无补全。
- 编辑器提示存在的函数可能运行时报不存在。

建议：

- 以 `XR_DEFINE_BUILTIN` 作为唯一声明源重新生成 analyzer。
- 给 `serve/stopServer/isServerRunning/sendData` 补 `XR_DEFINE_BUILTIN`。
- 移除 LSP 中不存在的 `hasError`，或实现该 API。
- LSP 由 builtin metadata 自动生成，避免手写漂移。

### 问题 2：`WsConn` handle 字段与实际 Json 不一致

builtin handle 声明：

```text
WsConn { wsid: int, url: string, state: string }
```

实际连接对象使用：

```text
{ _wsId, url?, state, error?, _isServer? }
```

测试和用户代码常访问 `conn.error`，但 handle 未声明该字段；内部函数依赖 `_wsId`，但 handle 声明是 `wsid`。

建议：

- 将 handle 改为实际字段：`_wsId`, `url?`, `state`, `error?`, `_isServer?`。
- 或把内部 ID 字段从 `_wsId` 迁移为公开 `wsid`，并统一所有 binding。
- 对失败返回值明确：当前 `connect` 声明为 `WsConn?`，但失败返回 `{ _wsId: -1, error, state: closed }`，不是 null。

### 问题 3：`WsMessage` 类型只支持 string data，实际 binary 返回 Array<u8>

`make_recv_result()` 对 binary message 返回 typed `Array<u8>`，但 builtin handle 声明：

```text
WsMessage { data: string, binary: bool }
```

建议：

- 类型层支持 `data: string?` + `bytes: Array<int>?`，或引入 Bytes 类型。
- 短期把文档写清楚：`binary=true` 时 `data` 是 byte array。

## Client 配置与握手

### 问题 4：C 配置支持 headers/subprotocols，但脚本 `connect` 不解析

`XrWsConfig` 支持：

```text
subprotocols, subprotocol_count, headers, header_count
```

client handshake 会发送 `Sec-WebSocket-Protocol` 和 custom headers。但 `ws_connect()` 脚本 options 当前只解析：

```text
timeout, pingInterval, pongTimeout, maxMessageSize
```

建议：

- 支持 `options.headers` 和 `options.subprotocols`。
- 返回连接对象中暴露 negotiated protocol。
- 对 header name/value 做 CRLF injection 防护。

### 问题 5：permessage-deflate 总是由 client 提议，脚本无法关闭

client handshake 默认发送：

```text
Sec-WebSocket-Extensions: permessage-deflate; client_no_context_takeover; server_no_context_takeover
```

如果 server 接受，所有 text/binary 消息默认压缩。脚本层没有 `compression: false` option，也不能设置 compression threshold。

建议：

- `connect` options 增加 `compression: bool` 和 `compressionMinBytes`。
- server upgrade options 也需要显式 compression policy。

### 问题 6：TLS/证书验证需要单独审计和测试

`wss://` 路径使用 net/tls，设置 hostname 并执行 client handshake。但当前 WS 测试主要依赖外部 echo 服务或本地 ws://，缺少本地 wss 证书验证、SNI、hostname mismatch 测试。

建议：

- 增加 wss loopback test。
- 明确是否默认验证证书和 hostname。
- 暴露 `tlsVerify` / CA 配置前需谨慎设计。

### 问题 7：DNS 仍是同步 getaddrinfo 路径

`xr_ws_connect()` 注释已说明 `xr_dns_resolve()` 仍依赖同步 resolver。高并发连接不可达域名时仍可能阻塞 worker。

建议：

- 统一 net/ws/cluster 的 async DNS 方案。
- 或将 DNS 标记为 SLOW 并交给 resolver pool。

## Frame 与协议边界

### 问题 8：fragmented message 总长度缺少 max_message_size 限制

接收单 frame 时会在分配前检查 `payload_len > max_message_size`，但 fragmentation 路径中 `ws_frag_append()` 只按累计长度扩容，没有检查累计长度是否超过 `max_message_size`。

风险：

- 攻击者可发送多个小 fragment 绕过单 frame payload limit。
- `frag_buf` 可持续增长导致内存压力。

建议：

- 在 `ws_frag_start/append` 前检查 `frag_buf_len + len <= max_message_size`。
- 超限发送 1009 close。

### 问题 9：send path 的 masked client frame 在 EAGAIN 时回退到 blocking send_all

`xr_ws_send_frame_try()` 的 client masking 路径在 `EAGAIN` 或 partial send 时无法重建相同 mask buffer，因此调用 `ws_send_all()` 完成剩余发送。

影响：

- `send` 虽然是 yieldable，但该路径可能阻塞 worker。
- 大 payload 或慢网络下影响并发。

建议：

- 将 masked frame buffer 保存在 send state 中跨 yield。
- `xr_ws_send_frame_try()` 不应在 would-block 后阻塞补发。

### 问题 10：control frame 约束需要更完整测试

实现处理了 close/ping/pong，但需要确认并测试：

- control frame payload <= 125。
- control frame 不允许 fragmentation。
- reserved opcode 关闭。
- continuation frame 状态机正确。

建议：

- 增加 raw socket frame fuzz/negative tests。

### 问题 11：zero-copy inplace message 生命周期容易误用

`xr_ws_recv_try()` 可能返回 data 指向 `rbuf` 的 embedded message，并用 `_flags` 标记不释放。binding 立即复制到 runtime string/array 后释放消息，当前脚本层安全。

但 C API 直接调用者如果保存 `msg->data`，下次 recv 可能覆盖。

建议：

- C API doc 明确 message data 生命周期只到下一次 recv 或 free。
- 或默认返回 owned buffer，zero-copy 只限内部 fast path。

## Server 与生命周期

### 问题 12：server singleton，`max_conns` 未真正生效

`XrWsContext` 有 `max_conns` 字段，但 `store_ws()` 没有检查它。server 只有一个 `listen_fd` 和 `server_running`，等价于 per-isolate singleton server。

建议：

- 明确 singleton 语义。
- 实现 `maxConns` 配置和 accept 前/after upgrade 限制。
- 未来如需多 server，设计 server handle。

### 问题 13：server handler closure 的 GC root/lifetime 需要审计

`WsServeListenCtx` 把 `handler_val` 存在 C continuation context 中跨 yield。相比 `WsRecvState` 自己注释明确“不跨 yield 存 XrValue”，server handler 这里存在 GC root 风险。

建议：

- handler closure 注册到 module context GC root。
- 或只保存 stable closure pointer，并在 GC mark callback 中标记。
- 增加 server 长时间运行 + GC 压力测试。

### 问题 14：upgrade request buffer 固定 4096

server echo/serve upgrade 使用 `WS_UPGRADE_BUF_SIZE 4096`。超长 header 会失败，且不可配置。

建议：

- 暴露 maxHeaderBytes。
- 失败时返回明确 HTTP 431 或 400。

### 问题 15：Origin/subprotocol policy 的 C API 未暴露到脚本 server

`xr_ws_upgrade_ex()` 支持 `allowed_origins` 和 `server_protocols`，但 `ws.serve` / `echoServe` / HTTP integration 默认走 legacy `xr_ws_upgrade()`，没有 origin/subprotocol policy。

建议：

- `ws.serve(port, handler, options?)` 支持 `allowedOrigins/serverProtocols/compression/maxMessageSize`。
- HTTP `ws` route 也应支持相同 policy。

## 压缩边界

### 问题 16：压缩发送没有 threshold 和背压控制

一旦 deflate negotiated，所有非空 text/binary send 都压缩。小消息压缩成本可能高于收益。

建议：

- 增加 compression threshold。
- 大消息压缩应与 yield/backpressure 结合，避免单次 CPU 占用过长。

### 问题 17：压缩失败时协议处理保守但错误不可观测

发送压缩失败直接返回 `WS_ERR_SEND`；接收解压失败关闭连接并设置 1009。脚本层只看到 `{error: Connection closed}` 或 send false，无法知道是 deflate 失败/zip-bomb。

建议：

- `recv` error 加 close code/reason。
- 连接对象暴露 `closeCode/closeReason/lastError`。

## 依赖与可见性

### 问题 18：握手 SHA1 依赖 CommonCrypto/OpenSSL，而项目已有 crypto SHA1

`ws.c` 在 macOS 使用 CommonCrypto，其他平台 include OpenSSL SHA1。这与模块“纯 C WebSocket”描述不完全一致，也引入平台依赖差异。

建议：

- 复用 `stdlib/crypto` 的 `xr_sha1()` 或抽出 base hash helper。
- 减少 OpenSSL 仅为 SHA1 handshake 的依赖。

### 问题 19：部分非 static C 函数定义缺少 `XR_FUNC`

头文件声明使用 `XR_FUNC`，但实现中如 `xr_ws_deflate_compress()`、`xr_ws_deflate_decompress()` 以及多个 `xr_ws_*` definition 未带可见性修饰符。

建议：

- 对所有跨文件 C API 的 definition 同步 `XR_FUNC`。
- 不对外的 helper 保持 `static`。

## 测试覆盖

现有覆盖：

- xray client 连接 Python echo server。
- Python websockets client 连接 xray server。
- text echo、多消息、binary echo、empty message、大消息、并发连接、快速连接关闭、ping/pong、close code。
- 外部 wss echo 服务功能测试。
- 并发/benchmark 脚本。

主要缺口：

1. 这些测试多在 `tests/network` / benchmark 下，不一定进入快速 regression。
2. `connect` options：timeout、pingInterval、pongTimeout、maxMessageSize。
3. `headers/subprotocols` 暴露后需要测试。
4. `recvData/sendData/serve/echoServe/stopServer/isServerRunning` 的 analyzer/LSP 同步测试。
5. fragmented message 累计超限。
6. control frame fuzz：oversize ping、fragmented control、reserved opcode、bad RSV。
7. invalid UTF-8 text close code 1007。
8. client receives masked server frame、server receives unmasked client frame。
9. permessage-deflate negotiation、small/big message、zip-bomb、corrupt deflate。
10. wss TLS certificate verification/SNI/hostname mismatch。
11. Origin allowlist 和 subprotocol negotiation。
12. server handler closure 在 GC 压力下是否仍安全。
13. slow network / partial send / EAGAIN 下 masked client send 是否阻塞 worker。
14. connection registry ID reuse、close 后 stale conn object 行为。
15. max connection limit。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | runtime/builtin/LSP API 不同步 | 用户可见 API 漂移 | 统一生成工具链 |
| 高 | `WsConn` 字段与实际对象不一致 | 静态类型误导 | 修正 handle 字段 |
| 高 | fragment 累计长度无 max 限制 | 可绕过 message limit | 累计检查并 1009 close |
| 高 | server handler `XrValue` 跨 yield 生命周期不明 | GC 后 handler 可能失效 | 注册 GC root 或存稳定 closure |
| 高 | masked send would-block 回退 blocking send | 慢网络可能 pin worker | send state 保存 frame buffer 跨 yield |
| 中 | connect options 不支持 headers/subprotocols | C 能力未暴露 | 解析 options 并返回 negotiated protocol |
| 中 | compression 无开关/threshold | 小消息浪费 CPU | compression options |
| 中 | close/error 信息过少 | 用户难以诊断 | 暴露 closeCode/reason/lastError |
| 中 | Origin policy 未暴露脚本层 | 浏览器跨站 WS 风险 | serve/http ws route options |
| 中 | TLS 行为缺少测试 | wss 安全边界不明 | 增加 wss loopback tests |
| 低 | SHA1 handshake 重复依赖外部库 | 依赖和实现分散 | 复用内部 SHA1 |
| 低 | 非 static C API definition 缺 `XR_FUNC` | 可见性规则不一致 | 补修饰符 |

## 后续实施建议

建议优先顺序：

1. 同步 `XR_DEFINE_BUILTIN`、generated analyzer、LSP 和 runtime exports。
2. 修正 `WsConn/WsMessage` handle 字段。
3. 修复 fragmented message 累计大小限制。
4. 审计并修复 server handler closure GC root。
5. 改造 masked send would-block 路径，避免 blocking send_all。
6. 暴露 `connect` 的 headers/subprotocols/compression options。
7. 为 `serve` 增加 options：allowedOrigins、serverProtocols、maxMessageSize、compression。
8. 增加协议负向测试和 permessage-deflate 测试。
9. 增加 wss TLS loopback 测试。
10. 复用内部 SHA1，收敛平台依赖。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
