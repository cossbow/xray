# stdlib/cluster 分析与优化建议

## 模块职责

`stdlib/cluster` 是标准库的分布式通信模块，目标是在多个 Xray isolate/进程之间提供节点连接、鉴权握手、分布式 Channel、RPC Service、Topic Pub/Sub、节点监控、协程监控、健康检测和 LAN 自动发现。

当前模块不是单一网络 API，而是一个完整的分布式 runtime 层：

- 每个 isolate 最多一个 `XrCluster` 实例。
- 每个远端节点对应一个 `XrClusterNode`。
- 节点间使用单 TCP/TLS 连接复用所有 frame。
- writer coroutine 串行化写出，reader coroutine 分发所有入站 frame。
- service call、channel recv 等需要响应的操作通过 pending request table 多路复用。
- Named Channel 使用 owner/proxy 模型避免分布式队列一致性。
- Topic Pub/Sub 使用 NATS-style wildcard trie，并通过 hop-limit 控制转发。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/cluster/cluster.h` | Cluster 核心结构、C API、脚本模块入口 |
| `stdlib/cluster/cluster.c` | lifecycle、脚本绑定、accept/heartbeat coroutine、frame dispatch、info API |
| `stdlib/cluster/cluster_node.h` | node、output queue、pending request、metrics、phi detector 定义 |
| `stdlib/cluster/cluster_node.c` | 节点连接、HMAC 握手、frame I/O、writer/reader coroutine、pending table |
| `stdlib/cluster/cluster_proto.h` | frame type、wire limit、handshake/version 常量、encode/decode API |
| `stdlib/cluster/cluster_proto.c` | big-endian length-prefixed frame 编解码 |
| `stdlib/cluster/cluster_serial.h` | XrValue binary serialization API |
| `stdlib/cluster/cluster_serial.c` | value encode/decode、CRC32、typed array 快路径 |
| `stdlib/cluster/cluster_channel.h` | distributed channel hook API |
| `stdlib/cluster/cluster_channel.c` | owner/proxy channel routing、select push/subscriber 模型 |
| `stdlib/cluster/cluster_topic.c` | topic trie、wildcard matching、local delivery、remote publish forwarding |
| `stdlib/cluster/cluster_health.c` | heartbeat、phi detector、reconnect、gossip、tombstone |
| `stdlib/cluster/cluster_monitor.c` | node/coroutine monitor notification |
| `stdlib/cluster/cluster_discovery.h` | discovery state and constants |
| `stdlib/cluster/cluster_discovery.c` | UDP multicast discovery coroutine |

## 当前脚本 API

`xr_load_module_cluster()` 实际导出 14 个函数：

| API | 当前签名/语义 | 备注 |
|---|---|---|
| `start(config)` | 启动 cluster，返回 bool | config 支持 `name/port/secret/tls` |
| `join(addr)` | 连接 `"host:port"`，返回 bool | LSP 写成 host+port 两参，实际是一参 string |
| `self()` | 返回当前节点名 string | 未启动时返回空字符串 |
| `nodes()` | 返回已连接节点名数组 | 只列远端节点，不含 self |
| `channel(name, size?)` | 创建/获取 distributed named channel | cluster 未启动时退化为本地 channel |
| `serve(name)` | 注册 service，返回 request Channel | LSP 写成带 handler，实际没有 handler 参数 |
| `reply(req, result)` | 回复 service request | 也支持 legacy `reply(id, from, result)` |
| `call(name, args, timeout?)` | 调用 remote/local service | timeout 当前未使用 |
| `monitor(node, coro?)` | 监控节点或远端协程 | 一参 node monitor；两参 coroutine monitor |
| `discover()` | 启动 LAN multicast discovery | 返回 bool |
| `stop()` | 停止 cluster | 返回 null |
| `info()` | 返回诊断 Json | 包含 nodes、metrics、TLS posture 等 |
| `publish(topic, value)` | topic publish | 返回 bool |
| `subscribe(pattern)` | topic subscribe | 返回 Channel |

## 当前架构优点

- Cluster state 挂在 isolate 上，符合“每 isolate 一个 cluster runtime”的直觉。
- 长生命周期任务已改为 native coroutine：accept、heartbeat、discovery 都不再使用裸 pthread 长阻塞。
- 停止路径使用 `stop_pipe` 唤醒 sleep 中的 cluster coroutine，降低 shutdown latency。
- 节点连接有独立 writer coroutine 和 output queue，避免多个业务 coroutine 并发写 socket 导致 frame interleaving。
- output queue 有高低水位背压，默认 4MB/1MB。
- pending request table 使用 32 bucket hash，避免响应 frame 被多个 coroutine 竞争读取。
- 握手协议升级为 HMAC-SHA256，proof 比较使用 constant-time compare。
- 握手有 5s read/write deadline，降低慢连接拖住 accept loop 的风险。
- `cluster.info()` 暴露 per-node metrics、RTT、phi、outq、slow consumer 和 TLS posture，便于诊断。
- Topic subscription 使用 segment trie，避免 publish 时线性扫描全部 subscription。
- Topic forwarding 有 hop-limit 和 split-horizon，降低转发风暴风险。
- 序列化支持 typed array 快路径，并带 CRC32 corruption check。

## API 与工具链漂移

### 问题 1：analyzer generated 完全没有 cluster 模块

`xanalyzer_builtins_generated.h` 的 module registry 包含 `base64/compress/crypto/.../ws/xml/yaml` 等模块，但没有 `cluster`。

影响：

- `import cluster` 的静态分析、补全和类型检查无法依据真实 builtin metadata 工作。
- 与其他标准库模块的生成流程不一致。

建议：

- 在 cluster 的绑定区补齐可被生成器识别的 builtin declarations。
- 将 `cluster` 加入 generated builtin registry。
- 增加“runtime export 与 analyzer registry 同步”测试。

### 问题 2：LSP 的 cluster API 不完整且签名错误

LSP 当前只列：

```text
start join self nodes channel serve call reply monitor stop
```

缺失：

```text
discover info publish subscribe
```

签名漂移：

- `start`：LSP 是 `void`，实际返回 `bool`。
- `join`：LSP 是 `(host: string, port: int)`，实际是单个 `"host:port"` string。
- `channel`：LSP 缺少 `size?`。
- `serve`：LSP 写成 `(name, handler)`，实际返回 request Channel。
- `call`：LSP 写成 `(node, service, data)`，实际是 `(serviceName, args, timeout?)`，路由到第一个 connected node 或本地 service。
- `monitor`：LSP 缺少远端 coroutine monitor 的两参形式。

建议：

- 由 builtin metadata 统一生成 LSP。
- 先修正手写 LSP，避免用户被错误签名误导。

## RPC 与 Channel 语义

### 问题 3：`cluster.call()` 没有真正等待响应

`cluster_call_fn()` 注释写明会等待 pending response，但实现中：

- timeout 参数被显式忽略。
- local service path 发送 request 后立即 `xr_channel_try_recv(rsp_ch, &ok)`。
- remote service path enqueue `SERVICE_CALL` 后也立即 `xr_channel_try_recv(rsp_ch, &ok)`。

这意味着只要响应不是在同一调用栈内已经到达，`cluster.call()` 就会返回 `null`。对真正的远端 RPC，这几乎必然发生。

影响：

- RPC API 表面存在，但可靠性很差。
- pending request 可能已经注册、frame 已发送，但调用方已经返回 null，后续 reply 会投递到无人等待的 temporary Channel。
- timeout 参数没有效果。

建议：

- 将 `cluster.call` 改成 yieldable C function。
- 使用 `xr_channel_recv` 或 VM timeout-aware receive 等待 `rsp_ch`。
- timeout 到期必须 `xr_cluster_node_take_pending(target, req_id)` 清理 pending。
- 如果不想 blocking/yielding，改为返回 future/channel：`callAsync()`。

### 问题 4：service routing 只选第一个 connected node

远端 `cluster.call(name, args)` 不知道 service 位于哪个节点，当前逻辑选择链表中的第一个 connected node。

影响：

- 服务不存在时会静默丢失或返回 null。
- 多节点同名服务没有负载均衡/路由策略。
- 节点顺序影响业务行为。

建议：

- 引入 service registry gossip：节点广播自己提供的 services。
- `call(service, args, options?)` 支持目标节点、负载均衡策略和 timeout。
- 无服务时返回明确错误。

### 问题 5：distributed Channel 的 owner/proxy 模型语义需要外显

当前设计是：

- owner channel 拥有真实 buffer。
- proxy send 序列化 value 并发 `CHANNEL_SEND` 到 owner。
- proxy recv 发 `CHANNEL_RECV_REQ`，由 owner reader 尝试从 owner buffer 取值并回包。
- select 支持通过 subscribe/push 模型，让 owner 把值推给一个 round-robin subscriber。

这是清晰且可实现的 at-most-once 模型，但脚本层没有显式说明：

- send 成功只代表 owner 接收了 frame 或本地 enqueue 成功，不代表远端业务已处理。
- push 给 subscriber 失败时会丢弃。
- select push 每次只选择一个 subscriber，不是广播。

建议：

- 文档/API 明确 distributed Channel 是 at-most-once。
- 如果需要 ack/retry，提供 higher-level RPC 或可靠队列。
- `cluster.info()` 增加 dropped push、serialize failure、subscriber full counters。

### 问题 6：remote Channel recv 没有 timeout

`dist_recv()` 使用 pending request + `xr_channel_recv()`，但没有 deadline。owner 不响应、连接半断、pending 丢失时，调用协程可能无限等待。

建议：

- Channel proxy recv 支持默认 cluster operation timeout。
- pending request 按 deadline 定期清理。
- node disconnect 时已经 close pending response channel，但连接仍活着而逻辑不响应时还需要 timeout。

## 协议与安全边界

### 问题 7：plain TCP + shared secret 是默认路径

`cluster.start(config)` 只有 `tls` 子对象时才启用 TLS；默认使用 plain TCP，并通过 HMAC secret 完成节点认证。

优点：

- 简单，适合本机/可信内网调试。
- HMAC 防止无 secret 节点加入。

风险：

- plain TCP 暴露所有 frame 内容。
- HMAC secret 本身不加密传输，但攻击者可离线观察 traffic metadata。
- discovery 中广播 `hash(secret)`，虽然不是 secret，但仍是 cluster fingerprint。

建议：

- 生产配置默认建议启用 TLS。
- `cluster.start` 在 `secret` 为空且非 localhost 时返回错误或告警。
- discovery hash 使用 HMAC 固定 label，而不是直接 hash secret，降低跨协议复用风险。

### 问题 8：TLS policy 仍偏基础

脚本层 TLS config 支持：

```text
enabled, caFile, certFile, keyFile, insecure
```

不足：

- 没有 ALPN。
- 没有明确 serverName / hostname verify 策略。
- 没有 cert rotation/reload。
- `insecure` 的危险性只在注释里体现，脚本层没有运行时告警接口。

建议：

- TLS options 增加 `serverName`, `alpn`, `minVersion`, `reload`。
- `cluster.info()` 将 insecure/verify 状态拆成可读字段，而不是仅 bitmap。
- 对 insecure 模式提供显式 warning hook。

### 问题 9：frame 读取缓冲与最大 frame limit 不一致

协议层 `XR_FRAME_MAX_PAYLOAD` 是 16MB，但 `xr_cluster_process_node()` 分配的 `recv_buf` 只有 65536。`xr_cluster_node_recv_frame()` 如果 payload 大于 buf_size 会返回错误并断开。

影响：

- 真实可接收 payload 上限约 64KB，而协议声明允许 16MB。
- 大消息会断连而不是返回明确错误。
- `cluster_serial` 和 topic/channel/RPC 上层没有提前限制消息大小。

建议：

- 统一 `XR_CLUSTER_MAX_FRAME_SIZE`。
- 小 frame 走栈/固定 buffer，大 frame heap allocate 到声明上限。
- 脚本层返回 `message too large` 错误，而不是断连。
- 对 publish/channel/RPC 添加 max payload config。

### 问题 10：序列化没有 schema/version 协商

`XR_SERIAL_VERSION` 当前为 `0x03`，cluster handshake 只有 handshake version，未协商 serialization version 或 feature flags。

影响：

- 滚动升级时，不同节点若 serial version 不同，会在业务 frame decode 时失败。
- frame type 增加或 payload 改造缺少能力协商。

建议：

- handshake flags 带上 protocol version、serial version、feature bitmap。
- 不兼容版本在握手阶段拒绝，而不是运行时业务 frame decode 失败。

### 问题 11：字符串 decode 使用 intern，可能造成内存压力

`cluster_serial.c` decode string 时使用 `xr_string_intern()`。网络输入如果包含大量唯一字符串，会给 intern table 带来压力。

建议：

- 对网络 payload 解码默认创建普通字符串。
- 只有 field name、枚举等低基数字符串才适合 intern。

### 问题 12：反序列化容器大小缺少总量预算

decode 有深度限制和 frame 上限，但 array/map/set/json 的元素数量仍可由 payload 指定。虽然最终受 frame size 限制，但缺少更直接的 allocation budget。

建议：

- decode context 增加 remaining allocation budget。
- 对容器 count 做上限检查。
- `cluster.start` 支持 `maxMessageBytes/maxContainerItems`。

## 生命周期与并发边界

### 问题 13：stop/free 使用 bounded spin wait，超时后仍可能 UAF

`xr_cluster_stop()` 对 accept/heartbeat/discovery coroutine 采用最多 1s spin wait。`xr_cluster_node_free()` 对 writer 采用最多 500ms spin wait。超时后继续释放状态。

优点：

- 避免 stop 永久挂起。

风险：

- 如果 worker 被长任务饿死或调度延迟，超时后 coroutine 仍可能引用被释放的 cluster/node/outq。

建议：

- 引入 coroutine join/refcount 生命周期。
- cluster/node 释放前等待引用归零，或延迟回收到 runtime safe point。
- `cluster.info()` 暴露 background coroutine exit 状态，便于诊断 stop 卡顿。

### 问题 14：`cluster_process_node` 释放 node，health/stop 也可能释放 node

reader loop 断开时会：

```text
remove subscribers -> fire monitors -> remove_node -> node_free
```

health checker 检测 dead node 也会做类似操作。虽然都有 nodes_lock 的 remove 操作，但 node 指针可能被收集后锁外释放。需要确保不会双重收集同一个 node。

建议：

- node 加 atomic refcount 或 closing flag CAS。
- 所有释放路径统一走 `xr_cluster_node_retire()`。
- 增加 disconnect 与 heartbeat timeout 竞态测试。

### 问题 15：`cluster.info()` 读取 outq 字段未持有 outq lock

`cluster.info()` 在 nodes_lock 内读取 node fields，但 `outq.total_bytes/frame_count` 由 outq.lock 保护。当前读取是诊断用途，允许短暂不一致，但严格来说存在 data race 风险。

建议：

- 将 outq counters 改成 atomic。
- 或读取每个 node 时短暂持有 outq.lock。

### 问题 16：`join` 地址解析不支持 IPv6 literal

`cluster.join(addr)` 使用 `strrchr(addr, ':')` 拆 host/port。IPv6 literal 如 `[::1]:9000` 或 `::1:9000` 无法可靠处理。

建议：

- 复用 `net` 的地址 parser。
- 或调整 API 为 `join(host, port)`，再兼容 `join("host:port")`。

## Discovery 与 gossip 边界

### 问题 17：LAN discovery 只支持 IPv4 multicast

`cluster_discovery.c` 使用 `AF_INET`、`ip_mreq`、`239.x` 风格 multicast，未支持 IPv6 multicast 或指定网卡。

建议：

- options 支持 `interface`, `group`, `port`, `ttl`, `ipv6`。
- `discover(options?)` 返回更详细错误。

### 问题 18：discovery 自动 join 在 coroutine 内串行执行

收到 announce 后直接调用 `xr_cluster_join()`。如果多个 peer 同时出现，discovery coroutine 会串行完成 TCP connect + handshake。

建议：

- discovery 只 enqueue join task，由独立 join worker/coroutine 并发处理。
- 对并发 join 做上限和去重。

### 问题 19：tombstone 没有周期性 sweep 调用点

`xr_cluster_sweep_tombstones()` 存在，但当前需要确认是否在 heartbeat tick 周期调用。若没有调用，dead node tombstone 可能长期阻止重连。

建议：

- 在 heartbeat loop 中定期 sweep。
- 将 tombstone window 暴露到 config/info。

## Topic Pub/Sub 边界

### 问题 20：Topic publish 是 best-effort flood，缺少 delivery/backpressure 反馈

`publish()` 返回 bool 仅代表本地 publish 过程成功，不代表所有远端 subscriber 收到。远端 enqueue 失败、decode 失败、subscriber channel full 可能静默丢弃。

建议：

- 保持 at-most-once 默认，但增加 metrics。
- 提供 `publishAck` 或 request/reply 风格可靠 topic。
- 对 hop-limit、max fanout、queue full 暴露配置。

### 问题 21：wildcard pattern 语义较强，应暴露验证 API

Topic 使用 NATS 语义：`*` 匹配一个 segment，`>` 匹配一个或多个剩余 segment 且必须在末尾。当前脚本层没有 `validatePattern()` 或错误详情。

建议：

- `subscribe(pattern)` 对非法 pattern 返回错误而不只是 null。
- 提供 `cluster.match(pattern, topic)` 或仅 C 层测试覆盖。

## 错误模型

### 问题 22：脚本层大多只返回 bool/null

失败原因会丢失：

- start 已运行、bind 失败、TLS 配置错误、pipe 创建失败。
- join DNS/connect/handshake/secret/version/TLS verify 失败。
- call service not found、timeout、queue full、decode error。
- publish serialization fail、queue full、frame too large。

建议：

- 引入 cluster error enum，复用/扩展 `XrNetError` 中网络错误。
- `cluster.lastError()` 或 `{ok, error, value}` 结果结构。
- `cluster.info()` 增加累计 error counters。

## 测试覆盖

当前未发现 `tests/` 下有 cluster 相关测试文件，`grep import cluster/cluster.` 无结果。

主要缺口：

1. module import/export：确认 14 个 runtime API 全部可见。
2. analyzer/LSP：确认 cluster builtin registry 与 LSP 签名同步。
3. start/stop：正常启动、重复 start、stop 幂等、端口冲突、port 0。
4. join：loopback 两节点连接、错误地址、IPv6、secret mismatch。
5. TLS：plain、TLS、mTLS、bad CA、hostname mismatch、insecure 模式。
6. handshake：version mismatch、timeout、bad proof、malformed frames。
7. RPC：local service、remote service、timeout、service missing、reply error、并发 256+ pending。
8. distributed Channel：owner/proxy send/recv、select push、subscriber full、channel close。
9. Topic：exact/star/greater-than wildcard、multi-hop forwarding、loop prevention、subscriber full。
10. Discovery：same secret join、different secret ignore、duplicate announce、IPv4 multicast unavailable。
11. Health：heartbeat RTT、phi timeout、node removal、monitor notification、tombstone sweep/rejoin。
12. Backpressure：outq high/low watermark、slow consumer metrics、large frame rejection。
13. Serialization：all supported types、unsupported types、CRC corruption、depth limit、large containers。
14. Lifecycle races：stop during accept/handshake/call/discovery、reader/health simultaneous disconnect。
15. Resource leaks：node free, pending cleanup, TLS context cleanup, discovery socket cleanup。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | analyzer generated 无 cluster | 静态分析缺失 | 补 builtin declarations 并生成 |
| 高 | LSP 签名严重漂移 | IDE 误导用户 | 用 builtin metadata 生成 LSP |
| 高 | `cluster.call` 不等待响应且 timeout 未用 | RPC 基本不可用 | 改 yieldable 或返回 Future/Channel |
| 高 | frame 上限 16MB 但接收 buffer 64KB | 大消息断连 | 统一 max frame 策略 |
| 高 | stop/node free bounded wait 后释放 | 极端调度下 UAF 风险 | coroutine join/refcount |
| 高 | 默认 plain TCP | 生产安全不足 | 默认/文档推荐 TLS，增强告警 |
| 中 | service routing 只选第一个节点 | RPC 路由不可控 | service registry gossip |
| 中 | Channel/proxy recv 无 timeout | 协程可无限等待 | pending deadline |
| 中 | `info()` 读取 outq counter 有 data race 风险 | 诊断不一致/并发风险 | atomic counter 或 outq lock |
| 中 | discovery 串行 join | 多节点启动慢且阻塞 discovery | join task queue |
| 中 | serialization string decode intern | intern table 压力 | 网络字符串默认非 intern |
| 中 | decode 缺少 allocation budget | 恶意 payload 内存压力 | decode budget/count limit |
| 中 | Topic/Channel delivery 静默丢弃 | 难排障 | 增加 drop metrics |
| 低 | `join("host:port")` 不支持 IPv6 | IPv6 可用性差 | 支持 `join(host, port)` |
| 低 | discovery 仅 IPv4 multicast | 网络环境受限 | 支持 IPv6/interface options |

## 后续实施建议

建议优先顺序：

1. 修复 `cluster.call`：改为 yieldable、实现 timeout、超时清理 pending。
2. 将 cluster 加入 analyzer builtin registry，并同步 LSP 签名。
3. 建立 cluster loopback 测试框架，至少覆盖 start/join/RPC/channel/topic/stop。
4. 统一 frame payload 上限与 receive buffer 策略。
5. 为 pending request 增加 deadline 和 cleanup tick。
6. 明确并文档化 at-most-once 语义，同时增加 drop/error metrics。
7. 引入 service registry gossip 和可控 routing。
8. 改善 lifecycle：node refcount/retire、cluster coroutine join、安全延迟回收。
9. 增强 TLS/discovery 配置与可观测性。
10. 为 serialization decode 增加 allocation budget，并避免高基数字符串 intern。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
