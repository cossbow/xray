# xray 分布式 cluster 模块设计方案

基于 Erlang 分布式源码分析和 xray 现有基础设施，设计一套以 Named Channel 为核心的分布式方案。

---

## 一、Erlang 分布式架构分析

### 1.1 三层架构

| 层 | 文件 | 行数 | 职责 |
|---|---|---|---|
| **ERTS C 核心** | `dist.c/h` (7.7K) + `external.c/h` (7.1K) + `erl_node_tables.c/h` (3.3K) | ~18,000 | 消息路由、序列化、节点表 |
| **Kernel Erlang** | `net_kernel.erl` (3.1K) + `global.erl` (3.4K) + `dist_util.erl` (1.3K) + 其他 | ~16,400 | 握手、全局注册、RPC |
| **EPMD** | `epmd*.c/h` | ~2,800 | 节点发现守护进程 |

### 1.2 核心机制

1. **DistEntry**（`erl_node_tables.h:130`）：每个远程节点对应一个 `DistEntry`，包含连接状态、输出队列、atom cache、monitor/link 列表
2. **分布式操作码**（`dist.h:140-181`）：30+ 种 DOP（SEND, LINK, EXIT, MONITOR, SPAWN...），每种对应一种跨节点操作
3. **External Term Format**（`external.c`）：6800 行序列化/反序列化代码，支持所有 Erlang term 类型
4. **能力协商**（`dist.h:30-121`）：64-bit DFLAG 位图，握手时协商双方支持的特性
5. **连接状态机**：IDLE → PENDING → CONNECTED → EXITING
6. **消息分片**（`ERTS_DIST_FRAGMENT_SIZE = 64KB`）：大消息自动分片
7. **Atom Cache**（`dist.c:248-279`）：高频 atom 用索引代替完整字符串

### 1.3 关键设计决策

- PID 内嵌节点信息 → 实现位置透明
- 连接断开时自动触发所有 link/monitor 的 EXIT 信号（`con_monitor_link_seq_cleanup`）
- 输出队列有 busy limit（1MB），防止慢节点拖垮发送方
- 支持 yield-based 编码（`TTBEncodeContext`），大消息序列化不阻塞调度器

---

## 二、xray 现有基础设施分析

### 2.1 网络层

| 模块 | 文件 | 行数 | 能力 |
|---|---|---|---|
| **net 标准库** | `stdlib/net/` | ~5,200 | TCP/TLS/UDP/DNS，完整的 API |
| **I/O 层** | `io.c/h` | ~1,000 | 协程友好的阻塞 I/O（kqueue/epoll 集成） |
| **Socket** | `xsocket.c/h` | ~634 | 底层 socket 操作 + yieldable API |
| **连接池** | `conn_pool.c/h` | ~535 | TCP 连接复用 |
| **缓冲池** | `xbuffer_pool.c/h` | ~293 | 内存复用 |

**关键能力**：`xr_io_connect`, `xr_io_read`, `xr_io_write_all`, `xr_io_accept` 全部支持协程 yield，无需额外适配。

### 2.2 Channel

| 文件 | 行数 | 关键特性 |
|---|---|---|
| `xchannel.c/h` | ~820 | 缓冲/无缓冲、wait queue、spinlock、timer channel |

**关键结构**：
- `XrChannel` 存储在 system heap（`xr_sysheap_alloc_shared`），引用计数管理
- 支持跨 Worker 唤醒（MPSC inbox + park/unpark）
- Go 风格语义：send 阻塞满、recv 阻塞空、close 唤醒全部

### 2.3 序列化

| 文件 | 行数 | 能力 |
|---|---|---|
| `xdeep_copy.c/h` | ~563 | 递归深拷贝，支持环检测 |
| `json.c/h` | ~1,000 | JSON 序列化/反序列化 |

**`xr_value_copy_kind` 分类**：IMMEDIATE(int/float/bool/null) → SHARED(string) → DEEP(array/map/closure)

### 2.4 值类型

NaN Boxing，8 字节统一表示：int(48-bit), float(64-bit), ptr(48-bit), null, true, false

GC 对象类型：STRING, ARRAY, MAP, SET, BYTES, JSON, FUNCTION, CHANNEL, INSTANCE 等

---

## 三、cluster 模块详细设计

### 3.1 总体架构

纯 C 模块，无 .xr 层。所有函数通过 C 层直接 export。

遵循 ws/net 模块设计原则：纯 I/O 函数 → C 层直接 export。
需要 `go` 或调用用户闭包的逻辑由用户代码负责，不在库内。

```
stdlib/cluster/
├── cluster.c              // 顶层初始化、shutdown、模块注册 + xray 绑定
├── cluster.h              // 公共头文件
├── cluster_serial.c       // XrValue 二进制序列化/反序列化
├── cluster_serial.h
├── cluster_proto.c        // 协议帧编解码
├── cluster_proto.h
├── cluster_node.c         // 节点连接管理（TCP 连接池 + 心跳）
├── cluster_node.h
├── cluster_channel.c      // Named Channel 分布式路由
└── cluster_channel.h
```

**不需要 .xr 层的原因**：
- 所有函数（start/join/call/reply 等）都是纯 I/O 或 yieldable 操作
- service API 设计为"返回 Channel"而非"传入回调"，`go` 和闭包调用由用户代码负责
- 对比：http.xr 需要 .xr 因为 `go _connHandler()` + 调用用户 `fn(req)` 闭包；cluster 不需要

### 3.2 运行时唯一改动：Channel 扩展点

在 `XrChannel` 结构体中添加一个指针字段：

```c
// xchannel.h — 唯一改动
typedef struct XrChannel {
    // ... 现有字段全部不变 ...

    void *dist;   // NULL = 纯本地（默认，零开销）
                  // 非 NULL = 指向 XrDistChannel
} XrChannel;
```

运行时核心通过回调钩子与 cluster 模块交互：

```c
// xchannel.h — 新增钩子接口
typedef struct XrChannelDistHooks {
    // Named Channel 的 send/recv 完全由 dist 层处理
    XrChanResult (*on_send)(struct XrChannel *ch, XrValue value, XrCoroutine *coro);
    XrChanResult (*on_recv)(struct XrChannel *ch, XrValue *out, XrCoroutine *coro);
    // close 时：通知远程节点
    void (*on_close)(struct XrChannel *ch);
} XrChannelDistHooks;

extern XrChannelDistHooks *xr_channel_dist_hooks;  // NULL = 无分布式
```

**关键语义**：Named Channel（`ch->dist != NULL`）的 send/recv **完全走 dist 路径**，
不使用本地缓冲区逻辑。只有 Owner 节点持有真实缓冲区，其他节点持有远程代理。

```c
// xr_channel_send() 中的分发逻辑
XrChanResult xr_channel_send(XrChannel *ch, XrValue value, XrCoroutine *coro) {
    if (ch->dist) {
        // Named Channel: 完全由 dist 层处理（Owner 写本地，非 Owner 走网络）
        return xr_channel_dist_hooks->on_send(ch, value, coro);
    }
    // 纯本地 Channel: 现有逻辑不变
    ...
}
```

**原则**：`ch->dist == NULL` 时（纯本地 Channel），所有路径与现有代码完全一致，零运行时开销。

### 3.3 二进制序列化协议

#### 值编码格式

```
XrValue → Binary:
┌──────────┬──────┬──────────┬─────────────┐
│ Ver(1B)  │ Tag  │ Length   │ Payload     │
│ 目前=0x01│ 1B   │ 0-4B    │ ...         │
└──────────┴──────┴──────────┴─────────────┘

首字节为序列化协议版本号（当前 0x01），用于节点间版本协商。

Tag 定义：
0x01  null                        → 1 byte total
0x02  bool   + 1 byte (0/1)      → 2 bytes
0x03  int    + varint             → 2-9 bytes
0x04  float  + 8 bytes IEEE754   → 9 bytes
0x05  string + len(4B) + UTF-8   → 5+N bytes
0x06  Bytes  + len(4B) + data    → 5+N bytes
0x07  Array  + count(4B) + elems → 递归
0x08  Map    + count(4B) + KV    → 递归
0x09  Set    + count(4B) + elems → 递归
0x0A  Json   + JSON string       → 使用现有 json.c
0x0B  int48  + 6 bytes           → 大整数紧凑编码
```

**不可序列化类型**：Channel、Closure、Instance（运行时报错）

#### 帧协议

```
TCP 帧格式：
┌──────────────┬──────────┬──────────────────┐
│ Length (4B)   │ Type(1B) │ Payload          │
│ big-endian   │          │                  │
└──────────────┴──────────┴──────────────────┘

Type 定义：
0x01  HANDSHAKE_REQ    { name, nonce, flags }
0x02  HANDSHAKE_ACK    { name, nonce, proof, flags }
0x03  HANDSHAKE_DONE   { proof }
0x04  HANDSHAKE_ERR    { error_code, message }
0x05  HEARTBEAT_PING   { timestamp }
0x06  HEARTBEAT_PONG   { timestamp }
0x07  CHANNEL_SEND     { channel_name_len, channel_name, value_bytes }
0x08  CHANNEL_RECV_REQ { channel_name_len, channel_name }
0x09  CHANNEL_RECV_RSP { value_bytes | empty }
0x0A  CHANNEL_CLOSE    { channel_name_len, channel_name }
0x0B  SERVICE_CALL     { request_id, service_name, args_bytes }
0x0C  SERVICE_REPLY    { request_id, result_bytes | error }
0x0D  NODE_INFO        { node_list }
0x0E  CHANNEL_SYNC     { channel_name, owner_node, buffer_size }
```

### 3.4 节点连接管理

```c
// cluster_node.h

typedef enum {
    XR_NODE_IDLE,        // 未连接
    XR_NODE_CONNECTING,  // TCP 握手中
    XR_NODE_HANDSHAKING, // 协议握手中
    XR_NODE_CONNECTED,   // 已连接
    XR_NODE_CLOSING      // 关闭中
} XrNodeState;

typedef struct XrClusterNode {
    char name[64];              // 节点名
    char host[256];             // 主机地址
    uint16_t port;              // 端口
    XrNodeState state;
    XrIOConn *conn;             // TCP 连接（复用现有 net 模块）
    int64_t last_heartbeat;     // 最后心跳时间
    uint32_t flags;             // 能力标志
    // 输出队列（类似 Erlang 的 DistEntry.out_queue）
    XrDistOutputBuf *out_first;
    XrDistOutputBuf *out_last;
    int64_t out_queue_size;
    int64_t busy_limit;         // 背压阈值（默认 1MB）
    struct XrClusterNode *next; // 链表
} XrClusterNode;

typedef struct XrCluster {
    char self_name[64];         // 本节点名
    uint16_t listen_port;       // 监听端口
    char secret[64];            // 集群密钥
    int listen_fd;              // 监听 socket
    XrClusterNode *nodes;       // 已连接节点链表
    int node_count;
    XrSpinlock nodes_lock;      // 节点列表锁（连接/断开时，低频）
    // Named Channel 注册表
    XrDistChannel **channels;   // 哈希表
    int channel_count;
    XrSpinlock channels_lock;   // Channel 注册锁（注册/注销时，低频）
    // Service 注册表
    XrServiceEntry *services;
    int service_count;
    _Atomic(bool) running;
} XrCluster;
```

### 3.5 Named Channel 所有权模型（Owner Model）

第一个创建 `Channel(N, "name")` 的节点拥有真实缓冲区（Owner），
其他节点创建同名 Channel 时得到远程代理（Proxy）。与 Erlang 的直接 PID 寻址不同，此模型与 Phoenix Channel 的 topic → owner 进程模式一致。

```
Owner 节点 A                    Proxy 节点 B/C
┌─────────────────┐            ┌──────────────────┐
│ XrChannel       │            │ XrChannel        │
│   buffer[N]  ←──┼── send ───┤   buffer = NULL  │
│   sendq/recvq   │── recv ──►│   dist → Remote  │
│   dist → Owner  │            │                  │
└─────────────────┘            └──────────────────┘
```

```c
// cluster_channel.h

typedef struct XrDistChannel {
    char name[128];             // Channel 名称
    bool is_owner;              // true = 本节点持有缓冲区
    XrClusterNode *owner_node;  // Owner 节点（is_owner=false 时有效）
    XrCluster *cluster;
} XrDistChannel;
```

**路由语义**：

| 操作 | Owner 节点 | Proxy 节点 |
|------|-----------|------------|
| send | 写入本地缓冲区（与本地 Channel 一致） | 序列化 → TCP 发送到 Owner → Owner 写入缓冲区 |
| recv | 从本地缓冲区读取（与本地 Channel 一致） | TCP 请求 Owner → Owner 从缓冲区取出 → 序列化返回 |
| close | 通知所有 Proxy 节点 | 通知 Owner 节点 |

**优势**：
- 语义清晰：缓冲区只有一份，FIFO 顺序由 Owner 保证
- 实现简单：Proxy 的 send/recv 就是 RPC 调用
- 无一致性问题：不需要分布式队列协调

### 3.6 握手流程（挑战-响应双向验证）

```
节点 A                              节点 B
  │                                    │
  │──── TCP connect ──────────────────►│
  │                                    │
  │──── HANDSHAKE_REQ ────────────────►│
  │     {version=1, name="node-a",     │
  │      nonce_a=random_bytes(16),     │
  │      flags=0x01}                   │
  │                                    │
  │◄──── HANDSHAKE_ACK ───────────────│
  │      {version=1, name="node-b",    │
  │       nonce_b=random_bytes(16),    │
  │       proof_b=SHA256(secret+nonce_a),│
  │       flags=0x01}                  │
  │                                    │
  │──── HANDSHAKE_DONE ───────────────►│
  │     {proof_a=SHA256(secret+nonce_b)}│  ← 双向验证完成
  │                                    │
  │◄──── NODE_INFO ───────────────────│
  │      {已知节点列表}                 │ ← Gossip: B 告诉 A 它知道的其他节点
  │                                    │
  │◄──►  CHANNEL_SYNC ◄──────────────►│  ← 互相同步 Named Channel Owner 信息
  │                                    │
  │◄──►  HEARTBEAT (每 5 秒) ◄──────►│
```

**安全性**：使用随机 nonce + SHA-256 挑战-响应，防止重放攻击。双向验证确保两端都持有正确密钥。

### 3.7 API 设计（纯 C export）

所有函数从 C 层直接 export，无 .xr 封装。

#### C 层 export 列表

| 函数 | 类型 | 说明 |
|------|------|------|
| `cluster.start(config)` | cfunction | 启动节点，config 为 Json |
| `cluster.join(addr)` | yieldable | 连接到已有节点 |
| `cluster.self()` | cfunction | 返回当前节点名 |
| `cluster.nodes()` | cfunction | 返回已连接节点列表 |
| `cluster.stop()` | cfunction | 停止集群 |
| `cluster.wait()` | yieldable | 阻塞直到集群关闭 |
| `cluster.serve(name)` | cfunction | 注册服务 + 返回请求 Channel（原子操作） |
| `cluster.reply(reqId, from, result)` | yieldable | 回复远程请求 |
| `cluster.call(name, args, timeout)` | yieldable | 远程调用，timeout 毫秒超时（默认 30000） |
| `cluster.monitor(nodeName)` | cfunction | 监控节点状态 |

`cluster.serve(name)` 内部完成两件事：向集群注册服务名 + 创建并返回请求 Channel。
不拆分为两个调用，避免注册与获取 Channel 不一致的 bug。

#### 用户代码示例

```xray
// 节点 A: 服务端
import cluster

cluster.start({name: "node-a", port: 9000, secret: "key"})
const ch = cluster.serve("echo")  // 注册服务 + 返回请求 Channel

for (req in ch) {
    go fn() {
        cluster.reply(req.id, req.from, req.args)
    }()
}
```

```xray
// 节点 B: 客户端
import cluster

cluster.start({name: "node-b", port: 9001, secret: "key"})
cluster.join("localhost:9000")

let result = cluster.call("echo", "hello")       // 默认 30s 超时
let fast = cluster.call("echo", "hello", 5000)  // 5s 超时
print(result)
```

```xray
// Named Channel 用法（分布式任务队列）
import cluster

cluster.start({name: "worker", port: 9002, secret: "key"})
cluster.join("localhost:9000")

const jobs = Channel(10, "jobs")  // Named Channel，集群可见
for (job in jobs) {
    go fn() {
        let result = processJob(job)
        const results = Channel(10, "results")
        results.send(result)
    }()
}
```

```xray
// 串行处理（不用 go，严格保序）
const ch = cluster.serve("ordered")
for (req in ch) {
    cluster.reply(req.id, req.from, process(req.args))
}
```

```xray
// 限流（最多 4 并发）
const ch = cluster.serve("heavy")
const sem = Channel(4)
for (let i = 0; i < 4; i++) { sem.send(true) }
for (req in ch) {
    sem.recv()
    go fn() {
        cluster.reply(req.id, req.from, compute(req.args))
        sem.send(true)
    }()
}
```

### 3.8 Channel 构造函数扩展

修改 Channel 的构造函数，支持可选的 name 参数：

```c
// VM 层：Channel(size) 或 Channel(size, "name")
// 当传入 name 参数时，自动注册为 Named Channel
static XrValue builtin_channel_new(XrayIsolate *X, int argc, XrValue *args) {
    int buf_size = (argc > 0) ? xr_to_int(args[0]) : 0;
    XrChannel *ch = xr_channel_new(X, buf_size);

    // 可选第二参数：Channel 名称
    if (argc > 1 && XR_IS_PTR(args[1])) {
        const char *name = xr_to_cstr(args[1]);
        if (name && xr_cluster_is_running()) {
            xr_cluster_register_channel(name, ch);
        }
    }

    return xr_value_from_channel(ch);
}
```

---

## 四、实施阶段

### Phase 1: 序列化层（~2,000 行，1-2 周）

**目标**：实现 XrValue ↔ binary 双向转换

| 文件 | 行数 | 内容 |
|---|---|---|
| `cluster_serial.h` | ~80 | 序列化 API 定义 |
| `cluster_serial.c` | ~1,200 | encode/decode 所有值类型 |
| `tests/unit/test_cluster_serial.c` | ~500 | 单元测试 |

**依赖**：`xvalue.h`, `xstring.h`, `xarray.h`, `xmap.h`, `xset.h`, `xbytes.h`, `xjson.h`

**验证**：
```c
// 单元测试：encode → decode → 比较
XrValue original = xr_int(42);
uint8_t buf[256]; int len;
xr_cluster_encode(original, buf, &len);
XrValue decoded = xr_cluster_decode(buf, len);
assert(xr_to_int(decoded) == 42);
```

### Phase 2: 协议帧 + 节点连接（~2,500 行，2 周）

**目标**：TCP 连接管理、握手、心跳

| 文件 | 行数 | 内容 |
|---|---|---|
| `cluster_proto.h` | ~100 | 帧格式定义 |
| `cluster_proto.c` | ~600 | 帧编解码 |
| `cluster_node.h` | ~120 | 节点管理结构 |
| `cluster_node.c` | ~1,200 | 连接管理、握手、心跳 |
| `cluster.h` | ~80 | 顶层 API |
| `cluster.c` | ~400 | 初始化、shutdown |

**依赖**：Phase 1 + `stdlib/net/io.h`（TCP 连接）

**验证**：两个 xray 进程能建立 TCP 连接并完成握手

### Phase 3: Named Channel 分布式路由（~2,500 行，2 周）

**目标**：跨节点 Channel send/recv

| 文件 | 行数 | 内容 |
|---|---|---|
| `cluster_channel.h` | ~100 | Named Channel 结构 |
| `cluster_channel.c` | ~1,500 | 路由逻辑、batch fetch |
| `xchannel.h/c` 改动 | ~50 | 添加 `dist` 字段和钩子 |
| `cluster_binding.c` | ~800 | xray 绑定 |

**依赖**：Phase 2 + `xchannel.h`

**验证**：
```xray
// 节点 A
cluster.start({name: "a", port: 9000})
const ch = Channel(10, "test")
ch.send("hello from A")

// 节点 B
cluster.start({name: "b", port: 9001})
cluster.join("localhost:9000")
const ch = Channel(10, "test")
let msg = ch.recv()  // → "hello from A"
```

### Phase 4: Service + Remote Spawn（~2,000 行，1-2 周）

**目标**：请求-响应模式 + 远程协程启动

| 文件 | 行数 | 内容 |
|---|---|---|
| `cluster.c` 扩展 | ~1,000 | serve/reply/call C export |
| `cluster_node.c` 扩展 | ~600 | spawn 请求处理 |
| 测试 | ~400 | 集成测试 |

**验证**：
```xray
// 节点 A
const ch = cluster.serve("echo")
for (req in ch) {
    cluster.reply(req.id, req.from, req.args)
}
// 节点 B
let result = cluster.call("echo", "hello")       // 默认 30s 超时
let fast = cluster.call("echo", "hello", 5000)  // 5s 超时
```

### Phase 5: 故障检测 + 健壮性（~1,500 行，1 周）

| 功能 | 行数 |
|---|---|
| 断线检测（心跳超时） | ~300 |
| 自动重连 | ~400 |
| 节点 monitor 回调 | ~300 |
| Gossip 协议（节点发现传播） | ~500 |

---

## 五、代码量汇总

| 阶段 | 预估行数 | 周期 |
|---|---|---|
| Phase 1: 序列化 | ~2,000 | 1-2 周 |
| Phase 2: 协议+连接 | ~2,500 | 2 周 |
| Phase 3: Named Channel | ~2,500 | 2 周 |
| Phase 4: Service | ~2,000 | 1-2 周 |
| Phase 5: 健壮性 | ~1,500 | 1 周 |
| **总计** | **~10,500** | **7-9 周** |

对比 Erlang 的 ~37,000 行，约为其 28%。原因：
- xray 值类型更少（序列化简单）
- 无需 30 年兼容历史
- 复用已有 net 模块
- 不实现 atom cache 等优化

---

## 六、风险与注意事项

### 6.1 对现有代码的影响

- **Channel 改动最小化**：仅添加 `void *dist` 字段 + 全局钩子指针
- **不影响单机性能**：`dist == NULL` 时零开销
- **不影响 GC**：Named Channel 仍在 system heap，引用计数不变

### 6.2 序列化边界

| 可序列化 | 不可序列化 |
|---|---|
| int, float, bool, null | Channel |
| string, Bytes | Closure/Function |
| Array, Map, Set | Class Instance（除非实现 Serializable） |
| Json 对象 | |

跨节点传递不可序列化类型时，运行时报错：`"cannot send <type> across cluster nodes"`

### 6.3 一致性模型

- **Channel 语义**：FIFO，由 Owner 节点保证顺序
- **无全局一致性**：不保证跨 Channel 的消息顺序
- **at-most-once delivery**：断线时进行中的 send/recv 返回错误，不做自动重传。
  由应用层决定重试策略。这与 Go channel、Erlang、NATS、Akka 一致——所有主流分布式系统基础层都是 at-most-once
- **不支持事务**：与 Erlang 一致，分布式事务由应用层处理

### 6.4 安全性

- 握手使用挑战-响应双向验证（SHA-256(secret + nonce)），防重放攻击
- 可选 TLS 传输（复用现有 `stdlib/net/tls.c`）
- 集群内节点互信，不做逐消息认证

---

## 七、文件结构总览

```
stdlib/cluster/
├── cluster.c              // 顶层初始化、shutdown、模块注册 + xray export 绑定
├── cluster.h              // 公共头文件（XrCluster, API）
├── cluster_serial.c       // XrValue 序列化/反序列化
├── cluster_serial.h
├── cluster_proto.c        // 帧编解码
├── cluster_proto.h
├── cluster_node.c         // 节点连接、握手、心跳
├── cluster_node.h
├── cluster_channel.c      // Named Channel 分布式路由
└── cluster_channel.h

// 无 .xr 层（纯 C 模块，同 net 模块）

// 运行时改动（#ifdef XR_HAS_CLUSTER 保护）
src/coro/xchannel.h        // 添加 void *dist + 钩子接口
src/coro/xchannel.c        // send/recv 中检查钩子（1 行 if）

// 测试
tests/unit/test_cluster_serial.c
tests/regression/15_cluster/
    1500_basic_connect.xr
    1501_named_channel.xr
    1502_service_call.xr
    1503_multi_node.xr
```

---

## 八、条件编译

cluster 模块遵循现有 stdlib 的条件编译模式，支持在嵌入式/纯净版场景下禁用。

### 8.1 现有模式分析

xray stdlib 使用三层条件编译机制：

**CMakeLists.txt（第 68-313 行）：**
```cmake
option(XR_STDLIB_FULL    "Full stdlib (ignores group options below)" ON)
option(XR_STDLIB_NET     "Network modules (net, http)" ON)
# ... 其他分组

if(XR_STDLIB_FULL)
    file(GLOB_RECURSE STDLIB_ALL_SRC "stdlib/*/*.c")
    # ...
else()
    add_compile_definitions(XR_STDLIB_MODULAR=1)
    if(XR_STDLIB_NET)
        file(GLOB STDLIB_NET_SRC "stdlib/net/*.c" "stdlib/http/*.c" "stdlib/ws/*.c")
        list(APPEND STDLIB_SRC ${STDLIB_NET_SRC})
        add_compile_definitions(XR_HAS_NETWORK=1)
    endif()
endif()
```

**xmodule.c（第 1024-1033 行）：**
```c
#if defined(XR_HAS_NETWORK) || !defined(XR_STDLIB_MODULAR)
    extern XrModule* xr_load_module_net(XrayIsolate *isolate);
    xr_module_register_native(isolate, "net", xr_load_module_net);
    // ...
#endif
```

**模式**：`#if defined(XR_HAS_XXX) || !defined(XR_STDLIB_MODULAR)` — 非模块化构建时全部启用，模块化构建时按开关启用。

### 8.2 cluster 条件编译方案

#### CMakeLists.txt 改动

```cmake
# 在 option 区域（第 73 行附近）添加：
option(XR_STDLIB_CLUSTER   "Cluster/distributed modules (cluster)" ON)

# 在 modular 编译区域（第 313 行 endif 前）添加：
    if(XR_STDLIB_CLUSTER)
        # cluster 依赖 network
        if(NOT XR_STDLIB_NET)
            message(FATAL_ERROR "XR_STDLIB_CLUSTER requires XR_STDLIB_NET")
        endif()
        file(GLOB STDLIB_CLUSTER_SRC "stdlib/cluster/*.c")
        list(APPEND STDLIB_SRC ${STDLIB_CLUSTER_SRC})
        add_compile_definitions(XR_HAS_CLUSTER=1)
        message(STATUS "Stdlib: +cluster")
    endif()
```

#### xmodule.c 改动

```c
    // ========== Cluster module ==========
#if defined(XR_HAS_CLUSTER) || !defined(XR_STDLIB_MODULAR)
    extern XrModule* xr_load_module_cluster(XrayIsolate *isolate);
    xr_module_register_native(isolate, "cluster", xr_load_module_cluster);
#endif
```

#### xchannel.h 改动

```c
typedef struct XrChannel {
    // ... 现有字段 ...

#ifdef XR_HAS_CLUSTER
    void *dist;   // NULL = local-only, non-NULL = XrDistChannel*
#endif
} XrChannel;

#ifdef XR_HAS_CLUSTER
typedef struct XrChannelDistHooks {
    bool (*on_send)(struct XrChannel *ch, XrValue value);
    bool (*on_recv)(struct XrChannel *ch, XrValue *out);
    void (*on_close)(struct XrChannel *ch);
} XrChannelDistHooks;

extern XrChannelDistHooks *xr_channel_dist_hooks;
#endif
```

#### xchannel.c 改动

Named Channel 完全走 dist 路径，不进入本地缓冲区逻辑：

```c
// xr_channel_send() 顶部（进入 spinlock 之前）：
#ifdef XR_HAS_CLUSTER
    if (ch->dist) {
        // Named Channel: 完全由 dist 层处理
        return xr_channel_dist_hooks->on_send(ch, value, coro);
    }
#endif
    // 以下为纯本地 Channel 逻辑（不变）
```

#### Channel 构造函数 named 参数

```c
// builtin_channel_new 中：
    if (argc > 1 && XR_IS_PTR(args[1])) {
#ifdef XR_HAS_CLUSTER
        const char *name = xr_to_cstr(args[1]);
        if (name && xr_cluster_is_running()) {
            xr_cluster_register_channel(name, ch);
        }
#endif
    }
```

### 8.3 构建场景

| 场景 | CMake 选项 | 效果 |
|---|---|---|
| **默认全量构建** | `XR_STDLIB_FULL=ON`（默认） | 包含 cluster，`GLOB_RECURSE` 收集所有 `stdlib/*/*.c` |
| **嵌入式纯净版** | `-DXR_STDLIB_FULL=OFF -DXR_STDLIB_CLUSTER=OFF` | 不编译 cluster，Channel 无 `dist` 字段 |
| **网络版无集群** | `-DXR_STDLIB_FULL=OFF -DXR_STDLIB_NET=ON -DXR_STDLIB_CLUSTER=OFF` | 有 net/http/ws，无 cluster |
| **隔离测试** | `-DXR_STDLIB_FULL=OFF -DXR_STDLIB_CLUSTER=ON -DXR_STDLIB_NET=ON` | 只编译 cluster + net 依赖 |

### 8.4 构建说明

cluster 是纯 C 模块。禁用时不编译 C 源码 → 模块注册不存在 → `import cluster` 报 "module not found" 错误。

### 8.5 二进制大小影响

| 构建 | 预估大小 |
|---|---|
| 全量（含 cluster） | ~2.8MB（+80-120KB） |
| 纯净版（无 cluster） | ~2.7MB（基线） |
| 最小嵌入式 | ~1.5MB（仅核心模块） |

---

## 九、源码阅读收获与设计决策确认

基于对 Go runtime、nng、NATS server、Akka cluster、Phoenix Channel、Ray、libuv 七个项目的源码阅读（详见 `docs/451_cluster_source_reading.md`），以下设计决策得到确认或调整：

### 9.1 确认的设计决策

1. **去中心化架构**：Akka 用 Gossip 协议实现去中心化，Ray 用中央 GCS。xray 选择去中心化（全连接 + Gossip 发现），与 Akka/NATS/Erlang 一致，避免单点故障
2. **Owner 模式 Named Channel**：Akka 用 VectorClock 合并冲突，复杂度高。xray 的 Owner 模型更简单——单一 Owner 即权威，无需分布式队列协调
3. **CONNECT + INFO 双向握手**：NATS 的 `processRouteConnect` + `processRouteInfo` 模式验证了 xray 的 HANDSHAKE_REQ/ACK/DONE 三步握手设计
4. **at-most-once 语义**：与 Go channel 一致——连接断了就是断了，不做自动重传
5. **分离锁**：Ray 用 `absl::Mutex` 读写锁分离，xray 的 `nodes_lock` + `channels_lock` 分离策略正确

### 9.2 新增/调整的设计要素

#### 消息帧优化（来自 nng）

- **header + body 分离**：nng 的 `nng_msg` 将路由元数据（header）和用户数据（body）分离。xray 帧协议中 `Type + channel_name` 是 header，`value_bytes` 是 body
- **引用计数消息**：broadcast 场景下一条消息发给多个节点时，仅序列化一次，多个输出队列共享同一 buffer（类似 nng 的 `nni_msg_clone`）

#### 路由发现优化（来自 NATS + Akka）

- **Gossip 隐式路由发现**：NATS 的 `forwardNewRouteInfoToKnownServers` 机制——节点 A 连接 B 后，B 告诉 A 它知道的 C、D 节点。对应 xray 的 `NODE_INFO` 帧
- **优先 gossip 给 seen 不同的节点**（来自 Akka `GossipTargetSelector`）：加速集群状态收敛
- **tombstone 机制**（来自 Akka `Gossip.tombstones`）：已退出节点留墓碑，防止被重新发现后尝试连接。xray 需在 `XrCluster` 中维护 `dead_nodes` 列表

#### 故障检测优化（来自 Akka + NATS）

- **全连接心跳**（与 Erlang/NATS 一致）：每个连接独立 PING/PONG，目标集群规模 <20 节点，O(N²) 开销可接受
- **慢消费者检测 + 背压**（来自 NATS）：`out_queue_size > busy_limit` 时标记节点为 slow，考虑断开。对应 xray 的 `busy_limit`（1MB）设计
- **tick 间隔监控**（来自 Akka `checkTickInterval`）：检测实际心跳间隔是否超过预期 2 倍，避免因 GC 或 CPU 过载导致误判

#### 连接管理优化（来自 NATS + nng）

- **重连指数退避**（来自 nng `s_reconn`/`s_reconnmax`）：xray 断连后重连应采用指数退避（初始 500ms，最大 30s），避免雪崩
- **单连接/节点对**（与 Erlang/Akka 一致）：每对节点间一个 TCP 连接，多路复用。NATS 支持池化但默认不开启
- **向量 I/O**（来自 NATS `net.Buffers` writev）：xray 输出队列应支持 scatter-gather I/O，减少系统调用次数

#### 序列化优化（来自 Phoenix）

- **fastlane 序列化缓存**（来自 Phoenix `dispatch` 函数）：同一消息 broadcast 给多个订阅者时，缓存序列化结果。xray 的 Named Channel broadcast 场景可借鉴

#### 事件驱动架构（来自 Ray）

- **事件监听器模式**（来自 Ray `InstallEventListeners`）：xray cluster 模块对外暴露 `on_node_added`/`on_node_removed` 回调注册接口，上层模块（如 service）通过回调感知节点变化，而非轮询
- **Health Check 独立管理**（来自 Ray `GcsHealthCheckManager`）：心跳逻辑独立于业务消息处理，不受业务负载影响

### 9.3 明确不采用的方案

| 方案 | 来源 | 不采用原因 |
|------|------|-----------|
| VectorClock | Akka | Owner 模型不需要冲突合并 |
| 中央 GCS | Ray | 去中心化更简单，无单点故障 |
| 每 Channel 一个进程 | Phoenix | 太重，Named Channel 应是轻量级数据结构 |
| Phi Accrual FD | Akka | 固定超时+3次容忍即可（与 Erlang/NATS/Ray 一致） |
| 线程池 I/O | libuv | xray 用协程代替线程池 |
| Atom Cache | Erlang | xray 无全局 atom 表 |

### 9.4 实施优先级调整

基于源码阅读，Phase 5（故障检测+健壮性）需增加以下内容：

- **tombstone 机制**：维护已退出节点列表（+100 行）
- **重连指数退避**：初始 500ms，最大 30s（+50 行）
- **慢消费者检测**：`out_queue_size` 超限告警/断开（+80 行）
- **事件监听器接口**：`on_node_added`/`on_node_removed` 回调注册（+120 行）

Phase 5 预估从 ~1,500 行调整为 ~1,850 行，总代码量从 ~10,500 调整为 ~10,850 行。
