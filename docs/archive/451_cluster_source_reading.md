# 分布式协程源码阅读计划

为 xray cluster 模块设计提供参考，按优先级阅读 7 个开源项目的核心源码。

---

## 阅读顺序与重点

### 1. Go runtime — 协程 + Channel 实现标杆

**路径**：`/Users/xuxinglei/workspace/xray_v5/go/src/runtime/`

| 文件 | 行数 | 重点 |
|------|------|------|
| `chan.go` | ~700 | Channel 实现：ring buffer、sudog wait queue、select |
| `proc.go` | ~5,600 | M:N 调度器：goroutine 创建、调度、抢占 |
| `netpoll.go` | ~500 | 网络轮询抽象层 |
| `netpoll_kqueue.go` | ~120 | kqueue 具体实现 |
| `select.go` | ~400 | select 多路复用 |

**xray 参考价值**：
- Channel 的 send/recv 阻塞/唤醒机制（与 xchannel.c 直接对标）
- sudog 等待队列管理（对标 xray 的 WaitQueue）
- 网络轮询与协程调度的集成方式

### 2. nng (nanomsg-next-gen) — 纯 C 消息传输

**路径**：`/Users/xuxinglei/workspace/xray_v5/nng/src/`

| 文件 | 行数 | 重点 |
|------|------|------|
| `core/message.c/h` | ~500 | 消息帧格式、零拷贝设计 |
| `core/aio.c/h` | ~700 | 异步 I/O 抽象（与 xray yieldable 对标） |
| `core/pipe.c/h` | ~350 | 传输管道抽象 |
| `core/socket.c/h` | ~1,100 | Socket 核心逻辑 |
| `core/taskq.c/h` | ~200 | 任务队列 |
| `core/msgqueue.c/h` | ~300 | 消息队列 |
| `sp/transport/tcp/` | TCP 传输实现 |
| `core/dialer.c` | ~400 | 连接管理、重连 |

**xray 参考价值**：
- 纯 C 消息帧设计（与 cluster_proto.c 直接相关）
- 异步 I/O + 消息队列架构
- 连接管理、自动重连机制
- 传输层抽象（TCP/IPC 统一接口）

### 3. NATS server — 高性能消息路由 + 集群

**路径**：`/Users/xuxinglei/workspace/xray_v5/nats/server/`

| 文件 | 行数 | 重点 |
|------|------|------|
| `server.go` | ~3,500 | 服务器核心：启动、监听、路由 |
| `client.go` | ~5,000 | 客户端连接管理、消息解析 |
| `route.go` | ~2,500 | 集群路由：节点间消息转发 |
| `parser.go` | ~650 | 协议解析器 |
| `sublist.go` | ~1,000 | 订阅列表（高性能 trie） |
| `gateway.go` | ~2,600 | 跨集群网关 |
| `proto.go` | ~140 | 协议常量定义 |

**xray 参考价值**：
- 高性能消息路由设计（>10M msg/s）
- 集群节点间的路由同步
- 订阅/发布模式（对标 Named Channel 路由）
- 背压和流控机制

### 4. Akka cluster — Gossip + 故障检测

**路径**：`/Users/xuxinglei/workspace/xray_v5/akka/akka-cluster/`

| 文件 | 重点 |
|------|------|
| `src/main/scala/akka/cluster/Cluster.scala` | 集群入口 |
| `src/main/scala/akka/cluster/ClusterHeartbeat.scala` | 心跳 + Phi Accrual 故障检测 |
| `src/main/scala/akka/cluster/Gossip.scala` | Gossip 协议实现 |
| `src/main/scala/akka/cluster/MembershipState.scala` | 成员状态管理 |
| `src/main/scala/akka/cluster/ClusterEvent.scala` | 集群事件（join/leave/unreachable） |

**xray 参考价值**：
- Phi Accrual Failure Detector（自适应故障检测，比固定超时更健壮）
- Gossip 协议的工业级实现
- 集群成员状态机
- 分片路由策略

### 5. Phoenix — Channel/PubSub 抽象

**路径**：`/Users/xuxinglei/workspace/xray_v5/phoenix/lib/phoenix/`

| 文件 | 重点 |
|------|------|
| `channel.ex` | Channel 抽象（join/leave/push） |
| `socket.ex` | Socket 管理 |
| `pubsub/` | 跨节点发布订阅 |
| `presence.ex` | 分布式在线状态跟踪（CRDT） |

**xray 参考价值**：
- Channel 的高级抽象设计
- 跨节点 PubSub 实现（基于 Erlang 分布式）
- Presence（分布式状态同步，CRDT 思路）

### 6. Ray — 分布式任务调度

**路径**：`/Users/xuxinglei/workspace/xray_v5/ray/src/ray/`

| 目录 | 重点 |
|------|------|
| `core_worker/` | Worker 进程管理 |
| `raylet/` | 本地调度器 |
| `gcs/` | 全局控制服务 |
| `object_manager/` | 跨节点对象传输 |
| `common/` | 通用基础设施 |

**xray 参考价值**：
- 分布式任务调度器设计
- 对象序列化和跨节点传输
- Worker 管理和资源调度

### 7. libuv — 事件循环参考

**路径**：`/Users/xuxinglei/workspace/xray_v5/libuv/src/`

| 文件 | 行数 | 重点 |
|------|------|------|
| `unix/core.c` | ~400 | 事件循环核心 |
| `unix/kqueue.c` | ~200 | kqueue 实现 |
| `threadpool.c` | ~300 | 线程池 |
| `timer.c` | ~130 | 定时器（最小堆） |
| `unix/tcp.c` | ~400 | TCP 连接管理 |

**xray 参考价值**：
- 跨平台事件循环的最佳实践
- TCP 连接管理模式
- 定时器实现

---

## 阅读记录

每个项目阅读完后，在此记录关键收获和对 xray cluster 设计的影响。

### 1. Go runtime

**阅读状态**：✅ 已完成

**阅读文件**：`chan.go`（971行完整）、`netpoll.go`（150行结构+API定义）

**关键收获**：

#### chan.go — Channel 实现

1. **hchan 结构**：单一 mutex 保护，ring buffer（`buf` + `sendx`/`recvx`），两个 waitq（`sendq`/`recvq`），`closed` 标志
2. **快速路径优化**：`chansend` 在加锁前先做非阻塞快速检查（`!block && full(c)`），避免无谓加锁
3. **直接传递**：当有等待的 receiver 时，`send()` 直接将数据从 sender 拷贝到 receiver 的栈上（绕过 buffer），减少一次拷贝
4. **sudog 复用池**：等待节点 `sudog` 从全局池分配，避免频繁内存分配
5. **select 非阻塞**：`selectnbsend`/`selectnbrecv` 只是调用 `chansend`/`chanrecv` 并设 `block=false`
6. **waitq 是双向链表**：`enqueue` 追加到尾部，`dequeue` 从头部取出，支持 select 竞争检测（`isSelect` + `selectDone` CAS）
7. **closechan**：关闭时唤醒所有等待者，sender 收到 panic，receiver 收到零值 + `received=false`
8. **timer channel 特殊处理**：`chanlen`/`chancap` 对 timer channel 返回 0，表现为无缓冲

#### netpoll.go — 网络轮询抽象

1. **pollDesc 状态机**：`rg`/`wg` 字段使用原子操作，状态为 `pdNil` → `pdWait` → `G指针` → `pdReady`
2. **二元信号量设计**：每个 fd 有独立的 read/write 信号量（`rg`/`wg`），而非共享锁
3. **fd 序列号**：`fdseq` 防止过期的 pollDesc 被误用（与 xray 的 fd 复用问题同源）
4. **deadline timer 内嵌**：读写超时 timer 直接嵌入 `pollDesc`，无需额外分配
5. **平台抽象接口**：`netpollinit`、`netpollopen`、`netpollclose`、`netpoll`、`netpollBreak` 五个函数完成平台适配

**对 xray cluster 的启示**：
- xray 的 `xchannel.c` 已采用类似 Go 的 spinlock + wait queue 设计，分布式扩展应保持一致
- Go channel 的"直接传递"优化在跨节点场景不适用，但可在本地 Named Channel 的 Owner 模式中借鉴
- netpoll 的 fd 序列号机制可用于分布式连接管理中的连接复用安全检查

### 2. nng

**阅读状态**：✅ 已完成

**阅读文件**：`message.c/h`、`aio.h`（245行）、`msgqueue.h`、`pipe.h`、`dialer.h`、`socket.c`（320行）

**关键收获**：

#### 消息帧设计（message.c）

1. **nng_msg 结构**：header（固定大小 uint32 数组）+ body（nni_chunk 动态缓冲）+ pipe_id + refcount + sockaddr
2. **nni_chunk 双指针**：`ch_buf`（底层缓冲起始）和 `ch_ptr`（有效数据起始），支持高效的 trim/insert 操作（移动指针而非拷贝数据）
3. **引用计数消息**：`nni_msg_clone` 增加引用，`nni_msg_unique` 在修改前确保独占，避免不必要的拷贝
4. **header + body 分离**：header 用于协议路由元数据（TTL 等），body 用于用户数据，`nni_msg_pull_up` 可合并

#### 异步 I/O 抽象（aio.h）

1. **nng_aio 结构**：统一的异步操作句柄，包含 timeout、result、iov、msg、4个input/output 槽位
2. **cancel 机制**：provider 注册 `cancel_fn`，aio 被取消时调用
3. **completion list**：批量完成通知，减少上下文切换（`nni_aio_completions_run`）
4. **expire queue**：独立的超时管理队列

#### Socket 架构（socket.c）

1. **双消息队列**：`s_uwq`（上层写队列）和 `s_urq`（上层读队列），类似 Go channel 的 buffered 模式
2. **Pipe 抽象**：每个连接是一个 pipe，socket 管理多个 pipe（listeners + dialers + active pipes）
3. **重连配置**：`s_reconn`（初始重连间隔）和 `s_reconnmax`（最大重连间隔），指数退避
4. **协议操作分离**：`nni_proto_pipe_ops`/`nni_proto_sock_ops`/`nni_proto_ctx_ops` 三层操作接口

**对 xray cluster 的启示**：
- **消息帧设计直接可借鉴**：header + body 分离方式适合 cluster 协议帧（header = type + channel_name，body = serialized value）
- **引用计数消息**：broadcast 场景下一条消息发给多个节点时避免拷贝
- **重连指数退避**：cluster 节点断连后的重连策略应采用类似机制
- **aio completion list**：批量处理完成通知的思路适合 cluster 的批量消息确认

### 3. NATS server

**阅读状态**：✅ 已完成

**阅读文件**：`route.go`（3315行核心部分）、`client.go`（370行结构定义）

**关键收获**：

#### 连接类型分层（client.go）

1. **5种连接类型**：`CLIENT`（终端用户）、`ROUTER`（集群路由）、`GATEWAY`（跨集群网关）、`LEAF`（叶节点）、`SYSTEM`（内部系统）
2. **client 结构**：统一 struct 用于所有连接类型，通过 `kind` 字段区分，`route`/`gw`/`leaf`/`ws`/`mqtt` 可选扩展
3. **outbound 缓冲**：`net.Buffers`（向量 I/O writev）+ 三级缓冲池（512B/4KB/64KB）+ 背压信号 `stc chan`
4. **慢消费者检测**：`isSlowConsumer` 标志，pending bytes 超限时标记并断开
5. **clientFlag 位图**：16-bit 紧凑布尔集合，比多个 bool 字段节省内存

#### 集群路由（route.go）

1. **路由池化**：`RoutePoolSize` 控制每对节点间的连接数（默认多连接），通过 `poolIdx` 分配
2. **createRoute 流程**：TCP连接 → 可选TLS → initClient → 设置重连 → 发送 CONNECT + INFO → 启动 readLoop/writeLoop
3. **processRouteInfo**：接收 INFO 后检测重复路由、集群名冲突、压缩协商，最终调用 `addRoute`
4. **addRoute 去重**：通过 `s.routes[remoteID]` map + `poolIdx` 精确定位，防止双向同时连接导致的重复
5. **Gossip 式路由发现**：`forwardNewRouteInfoToKnownServers` 将新节点信息转发给已知节点（隐式路由 `processImplicitRoute`）
6. **sendSubsToRoute**：新路由建立后同步订阅兴趣，避免消息丢失
7. **processRouteConnect**：CONNECT 验证集群名一致性，支持动态集群名协商（较小名称方让步）
8. **removeRoute**：清理路由表、网关URL、LeafNode URL，触发重连

#### 协议设计

1. **文本+二进制混合协议**：控制消息用文本（`INFO`、`CONNECT`、`PING`、`PONG`），数据用二进制（`RMSG`）
2. **CONNECT 携带 JSON**：认证信息、特性协商、集群名等以 JSON 编码
3. **nonce 机制**：`generateNonce` 生成随机 nonce，但路由间不做签名验证（比客户端简化）
4. **压缩协商**：支持 s2 压缩，通过 INFO 交换协商级别

**对 xray cluster 的启示**：
- **路由池化**是高吞吐的关键，xray 初期可用单连接，后期扩展为连接池
- **CONNECT + INFO 双向交换**模式适合 xray 的握手：HANDSHAKE_REQ/ACK/DONE 三步
- **Gossip 隐式路由发现**值得借鉴：节点 A 连接 B 后，B 告诉 A 它知道的 C、D 节点
- **慢消费者检测+背压**是必须的：xray 设计中 busy limit（1MB）与 NATS 的 `mp`（max pending）一致
- **统一 client 结构**简化代码：xray 的 `XrClusterNode` 可类似设计，内部/外部连接共用结构

### 4. Akka cluster

**阅读状态**：✅ 已完成

**阅读文件**：`Gossip.scala`（200行）、`ClusterHeartbeat.scala`（350行）、`ClusterDaemon.scala`（450+行）、`MembershipState.scala`（250行）、`Cluster.scala`（140行）、`ClusterEvent.scala`（490行）

**关键收获**：

#### Gossip 协议（Gossip.scala）

1. **Gossip 结构**：`members`（有序集合）+ `overview`（reachability + seen）+ `version`（VectorClock）+ `tombstones`
2. **VectorClock 版本控制**：每个节点有独立时钟，通过 `merge` 合并冲突版本
3. **merge 五步**：合并墓碑 → 合并向量时钟 → 合并成员（选最高优先级状态）→ 合并可达性 → 清空 seen
4. **seen 集合**：记录哪些节点已看到当前 gossip 版本，用于判断收敛
5. **tombstone 机制**：已移除节点留墓碑，防止被重新加入（时间清理）

#### 心跳与故障检测（ClusterHeartbeat.scala）

1. **HeartbeatNodeRing**：将节点排列在一致性哈希环上，每个节点只监控环上相邻的 N 个节点（`monitoredByNrOfMembers`）
2. **不是全连接**：N 个节点只需 O(N) 条心跳连接，而非 O(N²)
3. **heartbeat 流程**：定期发送 `Heartbeat` → 收到 `HeartbeatRsp` → 调用 `failureDetector.heartbeat(address)`
4. **ExpectedFirstHeartbeat**：首次心跳延迟触发，给对方回复时间
5. **tick 间隔监控**：检测实际 tick 间隔是否超过预期 2 倍（可能因 GC 或 CPU 过载导致误判）
6. **oldReceiversNowUnreachable**：成员变更时，旧的不可达接收者保留监控直到恢复或移除

#### 成员状态机（MembershipState.scala）

1. **状态流转**：`Joining` → `WeaklyUp` → `Up` → `Leaving` → `Exiting` → `Removed`；`Down` 可从任何状态转入
2. **收敛条件**：所有 DC 内 Up/Leaving 成员都 seen 当前 gossip，且不可达节点都是 Down/Exiting
3. **Leader 选举**：DC 内可达的、状态在 leaderMemberStatus 集合中的最小地址节点
4. **跨 DC**：`dcReachability` 过滤掉不同 DC 的观察者，各 DC 独立选 Leader
5. **gossipTargets**：优先 gossip 给 seen 不同的节点（加速收敛）

#### 集群守护进程（ClusterDaemon.scala）

1. **三个定时任务**：gossipTask（定期 gossip）、failureDetectorReaperTask（收割不可达节点）、leaderActionsTask（Leader 动作）
2. **receiveGossip 三种结果**：`Older`（忽略）、`Newer`（接受）、`Merge`（合并冲突）
3. **down 节点时立即 gossip**：加速传播 down 状态（STONITH 模式）
4. **coordinated shutdown**：优雅退出协调，等待 exiting 确认

**对 xray cluster 的启示**：
- **HeartbeatNodeRing** 是关键优化：xray V1 全连接可行（小集群），V2 应采用环形心跳降低开销
- **Phi Accrual Failure Detector**（Akka 默认）比固定超时更健壮，但 xray V1 可先用固定超时+3次容忍
- **VectorClock 对 xray 过重**：xray 的 Named Channel Owner 模式不需要冲突合并，单一 Owner 即权威
- **gossip 优先给 seen 不同的节点**：xray 的 NODE_INFO 传播应采用类似策略
- **tombstone 机制**：xray 也需要，防止已退出节点被重新发现后尝试连接

### 5. Phoenix

**阅读状态**：✅ 已完成

**阅读文件**：`channel.ex`（120行文档+API）、`channel/server.ex`（120行核心逻辑）、`presence.ex`（120行 CRDT 模式）

**关键收获**：

#### Channel 抽象（channel.ex）

1. **Topic 路由**：`"room:*"` 通配符匹配，`join/3` 回调控制授权
2. **三种消息模式**：`handle_in`（客户端→服务端）、`broadcast!`（→所有订阅者）、`reply`（→单个客户端）
3. **reply 带 ref**：每条客户端消息携带 `ref`，reply 关联同一 ref，实现请求/响应语义
4. **binary 支持**：`{:binary, data}` 元组传递二进制载荷

#### Channel Server（channel/server.ex）

1. **每 Channel 一个 GenServer 进程**：join 时启动进程，进程死亡则 Channel 关闭
2. **PubSub 集成**：通过 `Phoenix.PubSub` 实现跨节点消息分发
3. **fastlane 优化**：broadcast 时缓存序列化结果，同一 serializer 的多个订阅者共享编码
4. **dispatch 函数**：`Enum.reduce` 遍历订阅者，跳过发送者自己（`pid == from`）

#### Presence（presence.ex）

1. **CRDT 模式**：分布式在线状态追踪，基于 `Phoenix.Tracker`
2. **track/list 接口**：`Presence.track(socket, user_id, metadata)` 注册，`Presence.list(socket)` 查询
3. **自动 diff 广播**：joins/leaves 差异自动推送给所有订阅者（"presence_diff" 事件）
4. **fetch 回调**：可扩展元数据（如从数据库获取用户详情），避免 N+1 查询

**对 xray cluster 的启示**：
- **Topic 路由模式**可简化 Named Channel 寻址：`channel_name` 本质上就是 topic
- **fastlane 序列化缓存**：xray broadcast 到多个节点时，相同格式只序列化一次
- **Presence 的 CRDT 思路**：xray 的 CHANNEL_SYNC 协议本质上是 Channel 元数据的 CRDT 同步
- **每 Channel 一个进程**太重：xray 应保持轻量级，Named Channel 是数据结构而非协程

### 6. Ray

**阅读状态**：✅ 已完成

**阅读文件**：`gcs/gcs_server.h`（324行完整）、`gcs/gcs_server.cc`（核心启动+事件监听 600行）、`gcs/gcs_node_manager.h`（200行）、`gcs/gcs_node_manager.cc`（210行）、`gcs/gcs_task_manager.h`（150行）、`gcs/gcs_task_manager.cc`（127行）、`gcs/actor/gcs_actor_scheduler.h`（200行）、`raylet/node_manager.h`（250行）

**关键收获**：

#### GCS Server 架构（gcs_server.h/cc）

1. **中央控制面**：GCS（Global Control Service）是集群的"大脑"，管理所有元数据
2. **模块化初始化**：`DoStart` 依次初始化 12+ 个子管理器（Node、Resource、Job、Actor、PlacementGroup、Task...）
3. **gRPC 服务注册**：每个管理器注册为独立 gRPC service，`rpc_server_.RegisterService()`
4. **事件驱动**：`InstallEventListeners` 将节点增/删事件连接到各管理器的回调
5. **定期任务**：资源负载拉取、debug 状态打印、metrics 上报，通过 `PeriodicalRunner`

#### 节点管理（gcs_node_manager.h/cc）

1. **Register/Unregister 模式**：Raylet 启动时注册到 GCS，关闭时注销
2. **读写锁分离**：`absl::Mutex` 用 `ReaderMutexLock`/`MutexLock` 区分读写
3. **alive/dead 双缓存**：`alive_nodes_` + `dead_nodes_` map，快速查询
4. **Health Check**：`GcsHealthCheckManager` 通过 gRPC channel 定期检查节点存活
5. **事件监听器模式**：`AddNodeAddedListener`/`AddNodeRemovedListener`，松耦合通知

#### Actor 调度（gcs_actor_scheduler.h）

1. **GcsLeasedWorker**：抽象远程租用的 worker，包含地址、资源、关联 actor
2. **Schedule/Reschedule/Cancel**：标准调度三件套
3. **ReleaseUnusedActorWorkers**：释放未使用的 worker，避免资源浪费

#### 任务管理（gcs_task_manager.h/cc）

1. **优先级 GC**：已完成任务最先被回收，actor 任务次之，普通任务最后
2. **多维索引**：`task_index_`（按 TaskID）、`job_index_`（按 JobID）、`worker_index_`（按 WorkerID）
3. **worker 死亡级联处理**：`MarkTasksFailedOnWorkerDead` 标记该 worker 上所有任务失败

**对 xray cluster 的启示**：
- **中央 vs 去中心化**：Ray 用中央 GCS，Akka 用 Gossip 去中心化。xray V1 适合去中心化（无单点故障），但可保留未来加 GCS 的扩展点
- **事件监听器模式**很干净：xray 的 cluster 模块可用类似的回调注册机制通知各子系统
- **读写锁分离**：xray 设计中 `nodes_lock`/`channels_lock` 的分离与 Ray 思路一致
- **Health Check 独立管理**：xray 的 heartbeat 也应独立于业务消息处理
- **worker 死亡级联**：xray 节点断开时需级联处理该节点所有 Named Channel 的 Owner 迁移

### 7. libuv

**阅读状态**：✅ 已完成（之前 xray 开发过程中已深入阅读）

**关键收获**（基于之前的阅读经验）：

1. **事件循环核心**：`uv_run` 的 `default`/`once`/`nowait` 三种模式
2. **handle/request 二分法**：长期存活的 handle（timer、tcp、idle）vs 一次性 request（write、connect）
3. **跨平台抽象**：统一的 `uv_tcp_t`、`uv_timer_t` 接口，底层适配 kqueue/epoll/IOCP
4. **线程池**：`uv_queue_work` 将阻塞操作提交到线程池，完成后回到事件循环
5. **TCP 连接管理**：`uv_tcp_connect` → `uv_write` → `uv_read_start` 标准流程

**对 xray cluster 的启示**：
- xray 已有自己的事件循环（xnetpoll），无需依赖 libuv
- libuv 的 handle/request 分离模式可参考：cluster 连接是 handle（长期），单次消息是 request（一次性）
- 线程池模式不适合 xray：xray 用协程代替线程池，cluster I/O 应全部走协程路径

---

## 综合总结

### 对 xray cluster 设计的核心启示

| 参考项目 | 最重要的借鉴 |
|---------|------------|
| Go | Channel send/recv 语义保持一致，快速路径优化 |
| nng | 消息帧 header+body 分离，引用计数避免拷贝，重连指数退避 |
| NATS | Gossip 路由发现，CONNECT+INFO 握手，慢消费者检测+背压 |
| Akka | HeartbeatNodeRing 降低心跳开销，tombstone 防幽灵节点，收敛检测 |
| Phoenix | fastlane 序列化缓存，Topic 路由模型，Presence CRDT |
| Ray | 事件监听器模式，读写锁分离，Health Check 独立管理 |
| libuv | handle/request 分离思路 |

### 确认的设计决策

1. **去中心化架构**（非 Ray 式中央 GCS），全连接 + Gossip 发现
2. **Owner 模式 Named Channel**（非 VectorClock 冲突合并）
3. **challenge-response 握手**（NATS 简化版）
4. **固定超时心跳 V1**，预留 Phi Accrual 升级路径
5. **at-most-once 语义**，消息帧带版本号
6. **分离锁**：nodes_lock + channels_lock（参考 Ray 的读写锁分离）
