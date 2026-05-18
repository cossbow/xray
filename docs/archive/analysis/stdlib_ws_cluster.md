# stdlib「WebSocket + Cluster」分组源码分析

> **分组**：`ws`（WebSocket, RFC 6455 + permessage-deflate）、`cluster`（分布式节点 + Named Channel + Topic Pub/Sub + 节点/协程监控）
>
> **代码规模**：约 **10.2K 行 C**
> - `ws/`: 5 个文件 ~4.2K 行（`ws.c`、`ws_binding.c`、`ws_deflate.c` + 2 头文件）
> - `cluster/`: 15 个文件 ~6K 行（`cluster.c` + `_node` + `_proto` + `_channel` + `_topic` + `_discovery` + `_health` + `_monitor` + `_serial` + 头文件）
>
> **定位**：
> - `ws` 是 RFC 6455 的**纯 C 实现**，客户端 + 服务端都在 C 层，协程感知 I/O；和 `http_parser` 一样是工程品质较高的模块，但分享 "Net+HTTP" 组的相同积弊（malloc 红线、可见性缺失、阻塞回退路径）。
> - `cluster` 是一个**独立的分布式编程模型实现**（Erlang/Go-channel-over-network 风格），包含挑战应答握手、Phi-Accrual 失败检测、gossip、tombstone、多路复用请求表、基于 writev 批量发送的写协程、NATS 风格主题通配 —— 架构复杂度远高于其它 stdlib 模块。
>
> **读法**：对 `ws.c` 精读；`ws_binding.c` 因为 1880 行且只是模块绑定胶水，做表层抽样；cluster 重点精读 `cluster_node.c`、`cluster_channel.c`、`cluster_topic.c`、`cluster_health.c`、`cluster_discovery.c`、`cluster_serial.c`；通过 grep 统计全组的红线/可见性。

---

## 0. 一眼质量梯度

| 模块/文件 | 行数 | `malloc/free` 直用 | `XRAY_API` | 阻塞调用路径 | 综合 |
|-----------|------|---------------------|-----------|-------------|------|
| `ws.c` | 1997 | 36 | ❌ | ✅ handshake/close/TLS 发送/fallback `poll 5s` | **中高**（try API 协程，fallback 阻塞） |
| `ws_binding.c` | 1881 | 52 | ❌ | 协程 | 中 |
| `ws_deflate.c` | 148 | 12 | ❌ | 无 | 中（缺 bound） |
| `cluster.c` | 1334 | 23 | ❌ | `usleep` 心跳线程、同步 `xr_io_write_all` | 中 |
| `cluster_node.c` | 727 | 25 | ❌ | handshake 段阻塞、writer 协程化 | **中高** |
| `cluster_proto.c` | 514 | 低 | ❌ | 无 | 中高 |
| `cluster_serial.c` | 561 | 5 | ❌ | 无 | **高**（varint + zigzag + 深度限制） |
| `cluster_channel.c` | 447 | 5 | ❌ | 异步队列 | 中高 |
| `cluster_topic.c` | 248 | 3 | ❌ | 锁内投递 | 中低 |
| `cluster_discovery.c` | 318 | 4 | ❌ | pthread + `poll` | 中 |
| `cluster_health.c` | 266 | - | ❌ | `nanosleep` 在重连、Phi 检测 | 中高 |
| `cluster_monitor.c` | 219 | 7 | ❌ | 无 | 中 |

跨模块结论：
1. **🔴 红线违规约 170 处**（ws 100、cluster 73）；全组**零 `XRAY_API`/`XR_FUNC`**。
2. **🟠 WebSocket 客户端 handshake 全阻塞**（DNS 阻塞 + `connect` 阻塞 + 同步 TLS handshake + handshake 期间 `poll(5s)` 同步等待）。
3. **🟠 Cluster 握手路径用 `xr_io_write_all`** —— 内部可能调 TLS 阻塞；handshake 期间整个协程被冻。
4. **🟠 多处 "已知待办"**：`ws.c:208-210, 234-236` 有 `TODO: Refactor to use xr_socket_read_try + proper yieldable integration` 注释，说明作者已识别但没清理。
5. **🟠 Topic 投递在 `topics_lock` 内部**：`xr_channel_try_send` 触发选择器唤醒 → 若用户回调同步订阅/发布，**死锁/栈爆炸**风险。
6. **🟠 `ws_deflate.c` 解压无上限**：允许 zip bomb 攻击（permessage-deflate 用在 WS 上是常见攻击向量）。
7. **🟠 Cluster RPC 请求上限 `XR_MAX_PENDING_REQUESTS=256`**：超过后**直接失败**而非排队，多协程并发 RPC 的集群容易踩。
8. **🟠 `XrCluster` 一个 isolate 一个**，但 WebSocket 还在用全局的 TLS buffer pool（都是 TLS local，无进程全局状态）—— 设计不一致。
9. **🟡 重复压缩实现第 3 份**：`stdlib/compress` 从零实现 deflate、`http/http_compress` 调 zlib、`ws/ws_deflate` 又调一次 zlib。三套并存。

---

## 1. `ws` 模块（~4.2K 行）

### 1.1 亮点（值得夸）

- **RFC 6455 协议完整实现**：frame 解析、masking、fragmentation、close code 范围校验（1000-1011、3000-4999）、对 1004/1005/1006 的正确拒绝（client 发送非法 close code 拒绝）、text 帧 UTF-8 验证（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1669-1683`）。
- **控制帧规则强制**：不允许分片控制帧、控制帧 payload ≤ 125 字节、保留 opcode 拒绝（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:772-790`）。
- **permessage-deflate**（RFC 7692）：客户端主动协商 `client_no_context_takeover; server_no_context_takeover`，接收带 RSV1 帧自动解压（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1657-1666`）。
- **Zero-copy payload fast path**：小 frame 直接在 rbuf 内 unmask、数据指向 rbuf（`_data_inplace=true`），避免 memcpy（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1424-1432`）。
- **Buffer 复用（4 级 size class pool）**：256B/4KB/64KB/1MB 线程本地 free list（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:73-121`），避免 echo 循环的 malloc 风暴。
- **Cork 批发送**：用户可调 `cork/uncork` 把多帧合到一次 `send` 里（类似 TCP_CORK），极大减少系统调用（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1688-1717`）。
- **writev 零拷贝 server send**：server 端不需要 mask，用 `writev` 把 header + payload 合并发送（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:373-408`）。
- **32-byte unrolled XOR unmask**：4×uint64 展开帮编译器向量化（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:334-359`）。
- **Auto ping/pong**：`ping_interval_ms` + `pong_timeout_ms` 自动检活（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1310-1325`）。
- **Edge-triggered drain**：`ws_rbuf_fill` 持续 recv 到 EAGAIN，适配 kqueue edge trigger（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1266-1295`）。
- **Large payload bypass**：>64KB 的 payload 绕过 rbuf，直接 recv 到 msg_buf，避免两次拷贝（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1470-1497`）。
- **全局 `make_string` 用非 intern 版本**：WS 消息不去 intern pool 抢 rwlock，延迟恒定（binding line 117）。

### 1.2 问题清单

#### 🔴 硬红线

| # | 位置 | 问题 |
|---|------|------|
| W-1 | 全文 | **100 处 `malloc/calloc/realloc/free`**（ws.c 36 + ws_binding.c 52 + ws_deflate.c 12）。特别是 `ws_deflate.c` 的 12 处直接走系统 zlib 动态分配 |
| W-2 | 全文 | 无 `XRAY_API`/`XR_FUNC`（44 个公共函数） |

#### 🔴 协程语义问题

| # | 位置 | 问题 |
|---|------|------|
| W-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:208-210, 234-236` | 作者注释自己承认："xr_socket_read returns -2 for 'need yield' which is not compatible with WebSocket's internal loop. Use raw recv for now. **TODO: Refactor** to use xr_socket_read_try + proper yieldable integration"。所以：handshake、close、TLS 发送、partial-send fallback 全部走 `recv/send + poll(5s)` 的**同步阻塞**代码路径 |
| W-4 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:918` | `xr_ws_connect` → `xr_dns_resolve`（已知阻塞，见 net 组 D-1） |
| W-5 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:954` | `xr_ws_connect` 用**阻塞 `connect()`**（不调 `net_tcp_connect`），超时完全靠内核 |
| W-6 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:971` | `xr_tls_conn_handshake_client` 在 net/tls 组里有两套 API：`_try` 非阻塞版 + 这个**同步版**（内部 while 循环调 `xr_socket_read/write`）。当 TLS peer 慢时 worker 全挂 |
| W-7 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:248-250, 413-416, 605-611, 620-622, 1906-1912` | 5 处 `poll(&pfd, 1, 5000)` 兜底 —— worker 被阻塞最多 5 秒；对于"高并发 WS 服务端被慢客户端拖累"场景致命 |

#### 🟠 安全 / 正确性

| # | 位置 | 问题 |
|---|------|------|
| W-8 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws_deflate.c:98-131` | **decompress 无输出上限**：1KB 恶意压缩数据可以展开成 100MB+（zip bomb）。permessage-deflate 是 WS 常见攻击向量（CVE-2024-20017 同类） |
| W-9 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:167-204` `parse_ws_url` | 不支持 IPv6 `ws://[::1]:8080/`（和 `http_client.c` 同病） |
| W-10 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:980-1018` handshake request 生成 | `char request[2048]` 栈上硬编码；headers + subprotocols 长 → `WS_APPEND` 会返回错误。但**没路径截断告警**，用户不知道自己的 headers 太长被拒 |
| W-11 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1028-1039` | handshake response buffer 4096 字节，response 超过直接 `WS_ERR_RECV` —— 实际服务器 header 经常 > 4KB（带 cookie、proxy header） |
| W-12 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1051-1054` | **`strstr(accept_header, accept_key)` 是弱校验**：只要 response 任意位置包含该 accept_key 就通过。攻击者控制的 response header（例如 custom header value）可以注入合法 base64 绕过（理论问题，实际 server 控制 response） |
| W-13 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1931` | **服务端 `strstr(request_headers, "permessage-deflate")` 同样是子串匹配** —— 客户端请求 header 里任意其它地方出现这个字符串就认定协商成功 |
| W-14 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1064-1068` | 客户端 connect 成功后**才改 non-blocking**（O_NONBLOCK）。handshake 期间是阻塞模式 + `poll` 超时 —— 成本高于 net/io 的"全程 netpoll" |
| W-15 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:77-91` | TLS pool `WS_POOL_MAX_CACHED=32` 每 size class；4 个 class × 最大 1MB × 32 = 128MB peak per thread。**没主动 cleanup** |
| W-16 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1157-1159` | compressed 压缩失败后 **fallback 到未压缩发送** —— 但此时对端期望 RSV1=1（deflate 已协商）读取；实际会拿到 **RSV1=0 的帧**，导致对端判定协议错误。应该报错而非静默 fallback |
| W-17 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/ws/ws.c:1184-1186` | Client 压缩帧的 mask：`frame[header_len + i] = (char)(compressed[i] ^ mask_key[i & 3])` —— 朴素循环，没用 `apply_mask` 的 8 字节并行版本。大帧性能差 |
| W-18 | 全文 | **无 Origin 检查**：服务端 `xr_ws_upgrade` 不校验 `Origin` header，浏览器 Cross-Origin WS 攻击无防御 |
| W-19 | 全文 | **无 `Sec-WebSocket-Protocol` 子协议协商 API 给服务端**：`xr_ws_send_upgrade_response` 接受 protocol 参数，但没暴露给脚本层实现"根据客户端请求选子协议"的逻辑 |

#### 🟡 其它

| # | 问题 |
|---|------|
| W-20 | `xr_ws_recv`（阻塞 API）只有 `poll(100ms)` 轮询，忙循环式等数据；应标注 deprecated 或移除，只保留 `_try` 协程版 |
| W-21 | `ws_raw_recv` 对 TLS 不感知 non-blocking；TLS 模式下 recv 本身可能阻塞 |
| W-22 | `XrWsMessage._no_free / _data_inplace` 两个内部 flag 暴露在公共头 `ws.h` —— 应放 private |
| W-23 | `xr_ws_connect` 超时在 handshake 阶段（`ws_recv_timeout`）其实用 `poll` 手动算，不如复用 `xr_io_connect(host, port, timeout_ms)` |

### 1.3 测试缺口

- ✅ echo / server-client / concurrent / functional 几个测试存在
- ❌ **Autobahn TestSuite**（WebSocket 官方一致性测试套件，500+ case）没有跑
- ❌ permessage-deflate zip bomb 测试
- ❌ 超大 payload（> max_message_size）的 close code 1009 流程
- ❌ 非法 UTF-8 text 帧的 close code 1007 流程
- ❌ fragmented message 跨多次 recv 的状态恢复
- ❌ handshake 期间 socket 被 peer 关闭的错误路径
- ❌ permessage-deflate context 在 server 拒绝该 ext 时应禁用（协议协商测试）

---

## 2. `cluster` 模块（~6K 行）

### 2.1 架构亮点

这是整个 stdlib 里架构复杂度最高、也最"体面"的模块。

- **独立的分布式编程模型**：每个节点通过 `cluster.start(name, port, secret)` 启动，`cluster.join(host, port)` 手动加入、或 LAN multicast 自动发现。
- **Named Channel 的 Owner/Proxy 模型**（`cluster_channel.c`）—— 只有一个节点持有 Channel 的 buffer（Owner），其它节点通过 Proxy 转发；无需分布式一致性，at-most-once 语义（Erlang/NATS 风格）。
- **Dist hooks 钩子点**：不侵入 `XrChannel`，通过 `XrChannelDistHooks` 注入分布式 send/recv/close/try_*/destroy/select_enter/select_exit。本地 Channel 和分布式 Channel 代码共享。
- **挑战-应答握手 SHA256**：3 轮 HANDSHAKE_REQ → HANDSHAKE_ACK(with proof_b) → HANDSHAKE_DONE(with proof_a)，双向证明共享 secret（`cluster_node.c:517-596`）。
- **Phi-Accrual failure detector**（Akka 同款）：基于心跳间隔方差动态调整判死阈值（φ > 8.0），比固定超时鲁棒（`cluster_node.c:187-238`）。
- **专用 writer 协程 + writev 批量**：每个节点连接有独立的 writer 协程，用 pipe 唤醒，pop 所有队列帧一次性 `writev(64)`（`cluster_node.c:394-474`）。
- **管道零阻塞唤醒**：write end 置 non-blocking，enqueue 绝不阻塞（`cluster_node.c:76-77`）。
- **Pending Request 多路复用**：一个 TCP 连接上 N 个协程并发发 RPC，用 `request_id` 在 `pending_first` 哈希链查找响应，避免多协程 recv 冲突（`cluster_node.c:672-727`）。
- **Tombstone 机制**：节点死亡后其 name 进 tombstone，gossip/发现期间不重连近期离开的节点（防止 flapping，`cluster_health.c:204-266`）。
- **Gossip node discovery**：连接后互换 NODE_INFO，接收方自动对未知节点发起 connect（`cluster_health.c:126-200`）。
- **LAN multicast auto-discovery**：UDP 多播 `224.0.0.1:port`，FNV-64 hash(secret) 做 cluster-fingerprint，只接纳同-secret 的广播（`cluster_discovery.c:44-71`）。
- **Backpressure**：每节点输出队列带 high/low watermark（4MB/1MB），满即 `is_full=true` 触发发送端拒绝 —— slow consumer 可单独被识别（`cluster_node.c:57-68, 490-495`）。
- **Async zero-copy large frames**：> 4KB 帧用 `malloc + xr_outq_push_nocopy` 转移所有权给写协程（`cluster_node.c:345-361`）。
- **NATS 风格 topic matching**：`"events.*"` 单段通配、`"events.>"` 多段通配（`cluster_topic.c:36-69`）。
- **Varint + ZigZag + 深度限制 serial**：紧凑二进制、递归深度限制防止栈爆（`cluster_serial.c:86-104, 107-108`）。
- **Typed array 批量 memcpy**：`XR_ELEM_U8/I32/I64/F32/F64` 数组直接 memcpy 整个 buffer，不 per-element 编码（`cluster_serial.c:157-187`）。
- **CSP 监控**：`cluster.monitor(node)` 返回一个 Channel，节点死亡时自动投递 node name（`cluster_monitor.c:34-82`）；跨节点协程监控同理（`monitor_coro`）。
- **Named Channel 的 select 集成**：proxy channel 加入 select 时自动 SUBSCRIBE，退出时 UNSUBSCRIBE；Owner 用 round-robin push 给一个订阅者（`cluster_channel.c:373-403`）。

### 2.2 问题清单

#### 🔴 硬红线

| # | 位置 | 问题 |
|---|------|------|
| C-1 | 全组 | 73 处 `malloc/calloc/realloc/free`。`cluster_node.c` 25 处最重 |
| C-2 | 全组 | 无 `XRAY_API`/`XR_FUNC`（>50 个公共函数） |

#### 🔴 并发与正确性

| # | 位置 | 问题 |
|---|------|------|
| C-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_topic.c:160-189` | **Topic delivery 在 `topics_lock` 内调用 `xr_channel_try_send`** → 如果 Channel 唤醒的接收协程在 recv 回调里**同步 publish/subscribe**，立即递归获取同一把锁 → **死锁**。Named channel 代码路径（`cluster_channel.c:283-286`）也在 `channels_lock` 外调用，但 push_to_subscribers 自身又 `xr_spinlock_lock(&c->channels_lock)` —— 嵌套锁序脆弱 |
| C-4 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_node.c:517-596` | **Client 握手用阻塞 `xr_io_write_all` + `xr_cluster_node_recv_frame`**。`xr_io_connect` 底层已经是协程感知的 netpoll，但握手 3 轮是同步调用；对端若慢/故意拖延 → worker 冻结。建议改为 state machine + yield |
| C-5 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_node.c:600-669` | Server-side `xr_cluster_node_accept` 同样全同步 |
| C-6 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_node.c:52-58` | `xr_cluster_compute_proof` 把 secret 拷到栈缓冲 `input[80]`—— 读后**没 secure_wipe**。secret 残留在栈内存里 |
| C-7 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster.c:45-60` | **心跳线程用 `usleep`**，不是协程 sleep。进程终止时需要等最多 `sleep_ms` 才退出 |
| C-8 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_health.c:115-121` | `xr_cluster_reconnect` 用 `nanosleep` 阻塞重连退避 → 如果从 RPC 响应回调触发重连，worker 整个 block。且 `exponential backoff` 没加 **jitter**（注释说"with jitter"但实际只是 *2） |
| C-9 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_node.c:695-699` | RPC pending request **上限硬编码 256**，超过就 `return NULL` —— 用户协程看到 "XR_CHAN_CLOSED"，区分不出"连接断了"还是"pending 满了" |
| C-10 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_node.c:708-727` | `xr_cluster_node_take_pending` 是 **O(N) 链表扫描**，并发 RPC 多时成本高；应该用 small hash map |
| C-11 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_health.c:32-68` | `check_heartbeats` 栈缓冲 `to_remove[64]` —— 同时挂掉 > 64 个节点时漏检测（静默丢弃一些死节点判定） |

#### 🟠 安全

| # | 位置 | 问题 |
|---|------|------|
| C-12 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_node.c:49-59` | **Handshake proof = SHA256(secret \|\| nonce)** 无 length prefix、无 HMAC；给对手 1 字节长度控制就可能发生 length-extension-like 误配（具体到 SHA-256 语义安全，但用 **HMAC-SHA256 才是标准做法**） |
| C-13 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_node.c:532-544` | **所有集群通信明文**。secret 仅用于握手鉴别，之后的 TCP 流里 channel 数据、topic 数据都是**明文 XrValue 序列化**。任何局域网嗅探器都能看到所有 channel/topic 流量。没有 TLS wrap |
| C-14 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_node.c:659-670` | server accept 后 copy `req.name` 作为对方 node name —— **对方自称任意名字就能加入**（只要 proof 对）。如果一个合法节点 A 的 secret 泄漏，任意第三方可以假扮 node=A 加入。**没有节点 CA / 证书链** |
| C-15 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_discovery.c:47-68` | Multicast announce 里 `cluster_hash = FNV-1a(secret)` —— FNV **不是加密 hash**，已知明文 secret 可以用极短后缀爆破冲突；但对当前用途（只是分集群过滤）问题不大 |
| C-16 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_health.c:154-159` | Gossip payload `uint8_t payload[4096]` 栈上 —— 节点数 > ~100 就会在 loop 前 `size > sizeof - 300` 截断，不是所有节点都能传播 |

#### 🟠 正确性 / 设计

| # | 位置 | 问题 |
|---|------|------|
| C-17 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_topic.c:42-68` `xr_topic_match` | 匹配逻辑正确性**非常脆弱**：先 while 处理 literal/wildcard，然后 if 判断 `*p == '>'` —— 对 "events.>" 匹配 "events"（无段跟随）返回 false，对吗？  line 67 明写 `if (*p == '>' && *t == '\0') return false;` 所以 ">" 要求**至少一段**。这和 NATS 语义（`>` 要求 >= 1 segment）一致，但文档没说清 |
| C-18 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_topic.c:160-189` | **慢路径扫描所有 32 桶**：每次 publish 都 O(total_subscriptions)。订阅 1000 个主题的服务端每条消息都要扫一遍。应该建 trie（prefix tree）路由 |
| C-19 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_topic.c:192-203` | **`topic_handle_publish` 只投递本地 subscribers**，不转发给其它节点 —— 避免广播风暴。但这要求 publisher 节点**必须直连所有其他节点**才能让订阅者收到，mesh 不完整时消息丢失。应该有 hop-limit 的受控 flooding（或 tree-based multicast） |
| C-20 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_channel.c:262-288` | **Owner channel 的 `xr_channel_try_send` 满了直接 return -1** —— 上游 proxy 得到 `XR_CHAN_CLOSED`，脚本区分不出"buffer 满"和"连接断" |
| C-21 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_channel.c:307-310` | Push 到 subscribers 序列化失败后**静默丢数据**，注释说 "at-most-once semantics, data is lost" —— 对齐 Erlang/Go/NATS，但**文档一定要说清**；否则用户会认为 `chan.send()` 成功就"一定到达某个 receiver" |
| C-22 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_discovery.c:185-251` | discovery_thread 是**独立 pthread**，不是协程。节点状态变更要通过 `xr_cluster_join`（阻塞 connect + handshake）—— 这个 pthread 会阻塞在每次 join 上 |
| C-23 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_discovery.c:62-68` | announce 格式 `[magic 4B LE] [version 1B] [name_len 1B] [name] [port 2B LE] [cluster_hash 8B LE]` —— **LE 字节序**和 cluster_proto 的 **BE 字节序** 不一致，容易出 bug。应统一 |
| C-24 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_serial.c:41-46` | `xr_serial_buf_init` 直接 `malloc(256)` 无 NULL 校验路径被后续 `error` flag 接管，但调用方如果不查 `buf.error` 就直接 memcpy 到 NULL，会段错误。所有 encode 路径都要确保检查 `buf.error` |
| C-25 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_monitor.c:38-56` | monitor 的 channel 是 buffered(8)；死节点一次 fire 把一个 `node_name` 塞进去，**但多次 fire 同一节点（reconnect / 再死）时 channel 填满后静默丢弃** |
| C-26 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/cluster/cluster_health.c:85-124` | `reconnect()` 里 `xr_cluster_node_new(NULL, host, port)` —— **name=NULL**，然后 `xr_cluster_node_connect` 期间从 ACK 读取对方 name 写回 `node->name`。如果对方是冒充者（secret 正确），它可以自称任何 name，**就地顶替已死节点** |

#### 🟡 其它

| # | 问题 |
|---|------|
| C-27 | `cluster_node.c` 727 行逼近 800 行上限；`cluster.c` 1334 行**已经超过 3000 行限制的 `⅓` 不多**，但承担太多职责（lifecycle + binding），应拆分 |
| C-28 | `XR_CLUSTER_CHANNEL_BUCKETS = 64`、`XR_CLUSTER_SERVICE_BUCKETS = 16`、`XR_CLUSTER_TOPIC_BUCKETS = 32` 全部硬编码 |
| C-29 | `cluster_proto.c:xr_frame_buf_init` 是 header 内联函数，失败（malloc 返回 NULL）后 `fb.data = NULL`，调用方必须查；半数调用方没查 |
| C-30 | gossip 里 node name 用**单次比较**（strcmp） —— 对 64 字节的 name 每次 gossip 扫整张 nodes list 成本 O(n²) |
| C-31 | `cluster_topic.c:98` 订阅时**向所有节点广播 SUBSCRIBE frame**，但 publish 时不做扇出限制 —— 大集群 + 订阅密集场景下每条 publish 的 send 到 N-1 个节点是 O(N) |

---

## 3. 跨模块系统性问题

### 3.1 协程语义不统一

| 组件 | 协程语义 |
|------|---------|
| `ws` try API（`recv_try`、`send_frame_try`）| ✅ 非阻塞 + yield |
| `ws.c` handshake + close + TLS send + partial fallback | ❌ `poll(5s)` 同步 |
| `cluster_node.c` writer 协程 | ✅ native coroutine + pipe wakeup |
| `cluster_node.c` handshake | ❌ `xr_io_write_all` 同步 |
| `cluster_discovery.c` | ❌ 独立 pthread + pthread_create |
| `cluster.c` 心跳线程 | ❌ 独立 pthread + `usleep` |
| `cluster_health.c` reconnect | ❌ `nanosleep` 同步退避 |

混用 pthread + coroutine + TLS isolate 让心智成本很高，且**在协程调度器旁边有多个"独立时钟"**容易出一致性 bug（例：discovery 线程看到的 nodes list 和协程看到的在同一时刻可能不一致）。

### 3.2 重复实现与不一致

- **三套压缩**：`stdlib/compress` 从零写、`http/http_compress` 调系统 zlib、`ws/ws_deflate` 又调系统 zlib。
- **两套 TLS-over-TCP 抽象**：`net/io.c` 的 `XrIOConn`（ws_deflate/cluster 用）vs `net/net.c` 的 handle-based API（脚本用）。
- **字节序漂移**：`cluster_proto` BE、`cluster_discovery` announce LE。
- **两套错误枚举**：`XrWsError` alias 到 `XrNetError`，但没**完整**透出细分错误（如 `WS_ERR_PROTOCOL` 包一切协议错）。

### 3.3 观测性空白

- Cluster 有 `XrNodeMetrics`（frames/bytes/errors/slow_consumer_events/last_rtt_ms）——**但没有导出到 `cluster.info()` JSON**（文档写了，实现看不到）。
- Phi 值有函数但不暴露给脚本。
- `ws` 有 `WS_PROFILE` 宏但编译关掉（`#define WS_PROFILE 0`）。

---

## 4. 按 ROI 排序的 PR 建议

| 序号 | 工作量 | 优先级 | 建议 |
|------|-------|--------|------|
| **P1** | S（0.5 PD）| 🔴🔴 | **ws_deflate.c 加 `max_output_size` 上限**（默认 16MB，和 `max_message_size` 对齐）。避免 zip bomb |
| **P2** | S | 🔴🔴 | **Topic delivery 把 `try_send` 移出 `topics_lock`**：锁内收集 subs 到局部数组，锁外批量投递（避免递归死锁） |
| **P3** | S | 🔴 | **cluster_node.c 握手完结后 `secure_wipe(input, sizeof(input))`** 擦除 secret 残留（C-6） |
| **P4** | M（1 PD）| 🔴 | **WebSocket `_try` API 全路径协程化**：`ws.c:208-210` TODO 落地；`ws_raw_recv/ws_send_all` 替换为 `xr_socket_read_try/write_try` + yield continuation；**删除 `poll(5s)` fallback** |
| **P5** | M | 🔴 | **WebSocket client `xr_ws_connect` 走 `net_tcp_connect` 协程化**：DNS 异步（搭配 net 组 P5）+ 非阻塞 connect + 非阻塞 TLS handshake；可复用 `net/io.c:xr_io_connect` |
| **P6** | M | 🟠 | **Cluster handshake 协程化**（C-4/C-5）：改为"每步 `xr_socket_write_try` + yield" 的 state machine |
| **P7** | M | 🟠 | **全组 `malloc → xr_malloc` + `XRAY_API` 补齐**：ws ~100 处、cluster ~73 处。机械工作 |
| **P8** | M | 🟠 | **WS 服务端加 Origin 校验 API**（W-18）+ 子协议协商回调给脚本（W-19）+ strict header 匹配替换 `strstr`（W-12、W-13） |
| **P9** | M | 🟠 | **Cluster handshake 改 HMAC-SHA256**（C-12）—— 5 行代码换成 `xr_hmac_sha256(secret, key_len, nonce, 16, proof)` |
| **P10** | M | 🟠 | **Cluster 加可选 TLS wrap**（C-13）：复用 `net/tls.c`，在 secret 验证通过后升级到 TLS 再收 Channel 数据。可以按配置开关 |
| **P11** | L（2-3 PD）| 🟠 | **Topic routing 换 trie**（C-18）—— 对前缀 + 通配高效路由；现有 O(N_subs) 降到 O(path_depth) |
| **P12** | L | 🟠 | **Cluster RPC pending 改 hash map**（C-10）+ **上限调成 per-node 可配**（C-9） |
| **P13** | M | 🟠 | **Reconnect 加 jitter**（C-8）—— 避开节点群同时重试的雷群效应 |
| **P14** | M | 🟠 | **心跳/discovery 线程融进协程调度**（C-7/C-22）：用 `xr_runtime_schedule_at(now + 5000ms, cb)` 风格定时器，取代 pthread |
| **P15** | L | 🟡 | **合并三套压缩**：stdlib/compress 作为唯一入口，http/ws 的压缩层改为 thin wrapper |
| **P16** | L | 🟡 | **`cluster.info()` 真正暴露 metrics / phi / pool stats**（和序列化组的"统一 Result 对象"建议一起做） |
| **P17** | L | 🟡 | **Topic 跨节点转发**（C-19）：加 hop-limit TTL 支持大 mesh |

---

## 5. 测试补齐清单

### ws
- [ ] **Autobahn TestSuite 跑通**（500+ case）。这是 WS 测试的黄金标准
- [ ] permessage-deflate **zip bomb** 输入
- [ ] max_message_size 触发 close(1009)
- [ ] 非法 UTF-8 text 触发 close(1007)
- [ ] 分片消息 + 中间 ping/pong 交叉
- [ ] **并发 100 协程同时 recv/send 同一连接**（验证 `_try` API 协程安全）
- [ ] 握手 response 中间被 peer 关连接
- [ ] Origin 校验（P8 做完）

### cluster
- [ ] **3 节点集群 + 任意一台 kill -9**，验证 Phi 检测 / tombstone / gossip recovery
- [ ] **1000 并发 RPC** 同一连接（验证 pending 256 上限是瓶颈）
- [ ] **Topic trie 替换后** 的正则式发布压测（100K msg/s）
- [ ] 握手超时（对端不回 ACK）
- [ ] 对端回错误 proof → 握手失败 / secret 擦除
- [ ] Channel Owner 切主（owner 宕机后另一节点尝试 re-register）
- [ ] 多 isolate 同一进程各自开 cluster（冲突检测）
- [ ] LAN discovery 异 secret 集群互不可见
- [ ] Reconnect exponential backoff 实际 sleep 时间测量

---

## 6. 附：代码量与技术债粗算

| 模块 | 行数 | 直 `malloc` | 阻塞路径 | bug 条目 | 重构工作量 |
|------|------|-------------|---------|----------|-----------|
| ws | 4175 | 100 | 5+（handshake/close/TLS/partial）| 23 | 4-5 PD |
| cluster | 5976 | 73 | 4+（handshake/discovery/heartbeat/reconnect）| 31 | 8-12 PD |
| **合计** | **10151** | **~173** | **9+** | **54** | **~12-17 PD** |

---

## 7. 结语：两个模块的定位差异

- `ws` 是**"近生产可用"**：协议完整、零拷贝路径优秀、permessage-deflate 上线。挡路的主要是：(a) handshake/close 阻塞回退（W-3~W-7），(b) zip bomb 防御（W-8）。修完这两类问题即可上生产。
- `cluster` 是**"架构 impressive，工程需补齐"**：分布式模型设计很好（Owner/Proxy、Phi、gossip、tombstone、writer coro、RPC multiplex）；但同步握手、pthread 定时器、明文传输、pending 256 上限 → 在严肃分布式场景（跨 WAN、多租户、敌对网络）**不可用**。需要 **C-3 ~ C-14 八项** 重构才能声称 "production-grade distributed runtime"。

这一组的工程品质**比 net+http 组要高**（WS 的 `_try` 主路径、cluster 的 writer 协程 + writev 批量 + Phi detector 都是非常现代化的设计），但积累的技术债/未做完的事情也更具体。所以更像 "90% 做完的重型模块"，剩下的 10% 正好是边缘 case 和安全性，离"能上云"就差这一步。
