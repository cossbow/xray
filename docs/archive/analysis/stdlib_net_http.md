# stdlib「网络基础 + HTTP」分组源码分析

> **分组**：`net`（基础网络）、`http`（HTTP/1.1 + HTTP/2）
> **代码规模**：~17.5K 行 C
> - `net/`: 8 个文件 ~4.2K 行（net, dns, tls, conn_pool, io, xnetbuf, xneterror）
> - `http/`: 24 个文件 ~13.3K 行（http, parser, client, server, listen, router, cookie, multipart, stream, compress, proxy, http2×4）
>
> **定位**：这一组是 xray "能做服务器/能写爬虫"的门面，但它的**技术债密度在整个 stdlib 里最高**：30+ 源文件、无一处使用 `XRAY_API` / `XR_FUNC`、~400 处直接 `malloc/free/calloc/realloc`、同步阻塞调用混在协程 API 里、HTTP/2 HPACK Huffman 解码直接缺失。
>
> **本文档风格**：只对关键文件做抽样精读（`net.c`、`dns.c`、`tls.c`、`conn_pool.c`、`http.c`、`http_client.c`、`http_parser.c`、`http2.c`、`http_listen.c`、`http_stream.c`、`http_multipart.c`、`http_cookie.c`、`http_compress.c` 等），通过 `grep` 统计其它文件的红线与可见性，汇总成跨模块问题清单。

---

## 0. 质量梯度与体积概览

| 模块 | 行数 | 红线（`malloc`/`free` 直用） | 可见性 | 阻塞调用 | 协议完整度 | 综合 |
|------|------|------------------------------|--------|----------|-----------|------|
| `http_parser` | 1208 | 0 | ❌ | 无（纯 CPU） | 高（SIMD、Zero-copy、Chunked） | **高** |
| `xnetbuf` | 194 | 9 | ❌ | 无 | N/A | 中高 |
| `xneterror` | 44 | 0 | ❌ | 无 | N/A | 高 |
| `http_router` | 518 | 15 | ❌ | 无 | Radix tree | 中高 |
| `http_cookie` | 470 | 24 | ❌ | 无 | RFC 6265 基础 | 中 |
| `tls` | 598 | 12 | ❌ | **协程友好接口 + 阻塞接口混用** | TLS 1.2+、SNI、ALPN | 中 |
| `io` | 578 | 8 | ❌ | 无（netpoll） | 协程 IO 层 | 中 |
| `net` | 1600 | 56 | ❌ | **udp 同步 sendto/recvfrom** | TCP/UDP/TLS/DNS 封装 | 中 |
| `http_compress` | 475 | 23 | ❌ | 同步 zlib | 与 `stdlib/compress` 功能重复 | 中 |
| `http_server` | 610 | 6 | ❌ | 协程 | HTTP/1.1 | 中 |
| `http_listen` | 1175 | 15 | ❌ | 协程（native-stackless + VM 栈） | HTTP/1.1 + WS 升级 | 中 |
| `http_stream` | 310 | 7 | ❌ | **同步 fopen/write/select** | 文件下载 | 低 |
| `http_multipart` | 305 | 19 | ❌ | 无 | 有 `rand()` boundary 安全问题 | 中低 |
| `http_proxy` | 245 | 13 | ❌ | 无 | HTTP/SOCKS | 中 |
| `http_client` | 938 | 35 | ❌ | **阻塞 recv/send 走 conn_pool** | HTTP/1.1 + Keep-Alive | 中 |
| `conn_pool` | 424 | 12 | ❌ | **阻塞 connect/recv/send/TLS** | 简易 LRU | 低 |
| `dns` | 561 | 7 | ❌ | **阻塞 `getaddrinfo`** | LRU + 轮询 | 中低 |
| `http` | 1451 | 46 | ❌ | 混用 | 顶层入口 | 中 |
| `http2` | 1151 | 38 | ❌ | 阻塞 `h2_recv` | **HPACK Huffman 未实现** / **PADDED 不处理** | **低** |
| `http2_client` | 669 | 39 | ❌ | 阻塞 | 基础 | 低 |
| `http2_server` | 746 | 5 | ❌ | 阻塞 | 基础 | 低 |
| `http2_binding` | 261 | - | ❌ | - | - | - |

### 跨模块结构性问题（一眼结论）

1. **🔴 红线违反规模最大**：`net` 104 处 `malloc/free`、`http` 285 处（整个 stdlib 其它组加起来不如这组多）。`grep` 确认**零** `XRAY_API`/`XR_FUNC` 使用。
2. **🔴 协程友好性分裂**：同一份代码里"非阻塞 netpoll 协程 API" 和 "阻塞 `connect/recv/send/fopen/getaddrinfo/select`" 并存 —— `http_client.c` + `conn_pool.c` + `dns.c` + `http_stream.c` 共四条阻塞 IO 链路会**把整个 worker 线程冻住**。
3. **🔴 HTTP/2 不完整**：`http2.c:178-183` 注释 "Huffman not implemented yet, just copy"；大多数现代服务器会发 Huffman 编码的 header，因此 xray 实际解出来是乱码；`PADDED` flag 未处理；`connection_window` 无 overflow 校验。
4. **🟠 两套连接池、两套压缩**：`conn_pool`（HTTP/1）+ `http2_client` 的 `XrH2Pool`、`stdlib/compress` 从零实现的 deflate + `http_compress` 再调系统 zlib —— 功能重复、配置不互通。
5. **🟠 设计漂移**：`XrHttpContext` 承诺"每 isolate 一个连接池"，但 `http_client.c` 实际用**进程全局** `g_http_pool`（文件静态变量），单测过但多-isolate 场景等于无隔离。
6. **🟠 错误码统一了，错误传播没统一**：`xneterror.h` 做了正确的事，但 binding 层全是 `return xr_null()`；脚本层拿不到 `XR_NERR_*`。
7. **🟠 安全问题**：multipart boundary 用 `rand()+srand(time)` → 1 秒窗口内可预测；cookie parser 对 `Set-Cookie` 输入长度无上界；HTTP 头 Host 域名长度无校验。

---

## 1. `net` 子模块

### 1.1 `net/net.c`（1600 行，56 处 `malloc/free`）

**亮点**
- **Handle API 设计优雅**：把底层 fd 藏在 `XrJson` handle 里，通过 Shape cache 让 handle 字段访问是 O(1) 内联访问（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/net.c:407-441`）。
- `net.dial/accept/read/write/dialTLS/upgradeTLS/sendTo/recvFrom` 全部是 yieldable C 函数，用 `xr_yield_for_io` + continuation 正确实现协程非阻塞 IO（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/net.c:497-888`）。
- **TCP_NODELAY 默认开启**（`net_tcp_connect` line 91）+ `TCP_NOTSENT_LOWAT=16KB`（io.c:71）—— 低延迟默认值专业。
- `net_close_fd` 正确处理 `netpoll deregister + shutdown + close` 三步（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/net.c:63-73`），避免 netpoll 悬垂指针。
- **Per-coroutine I/O 缓冲复用**：`xr_coro_ensure_io_buf`（line 638-650）把 4KB ~ 256KB 的 read buffer 挂到 `XrCoroExt`，消除每次 read 的 malloc。

**问题清单**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| N-1 | 全文 | 🔴 | 56 处 `malloc/free/calloc` + 1 处 `realloc` —— 对 state 结构（`NetDialState`、`NetAcceptState`、`NetReadHandleState`、`NetWriteHandleState`、`NetDialTLSState`、`NetUpgradeTLSState`、`NetSendToState`、`NetRecvFromState`）每次请求都分配一次 |
| N-2 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/net.c:181-201`、`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/net.c:203-226` | 🟠 | `xr_udp_send_to` / `xr_udp_recv_from` 直接 `sendto`/`recvfrom` —— **阻塞**。script 一旦调用 `XrUdpConn` 旧 API 就冻结 worker。新的 handle-based `net_send_to_yieldable` 正确，但旧接口还对外暴露 |
| N-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/net.c:645-649` | 🟠 | `xr_coro_ensure_io_buf` 内部 `char *buf = (char *)realloc(ext->io_buf, cap)` —— 无临时指针中转，失败时 `ext->io_buf` 保留；但调用方假设成功后 `ext->io_buf = buf`，失败路径返回 NULL 后上层走 `malloc(max_len)` fallback 又**没配套释放**（line 753 + 762 只在 `!coro` 时 free，但状态里只保留了 state 而没保留 buf owned 标记） |
| N-4 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/net.c:278-301` | 🟠 | `xr_net_parse_addr` 用 `strrchr(addr_str, ':')` 查端口分隔符 —— **对 IPv6 `[::1]:8080` 格式彻底错误**，会把倒数第二个 `:` 当分隔符 |
| N-5 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/net.c:388-389` | 🟠 | `g_udp_recv_buf[65536]` / `g_udp_recv_addr` 作为**TLS 静态变量**跨请求共享，多协程在同一 OS 线程跑时若两个 `recvFrom` 交错就会相互覆盖。目前靠"同一协程跑到 return 前不会切出去再回来"保护，脆弱 |
| N-6 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/net.c:883-885` | 🟡 | `state->data = XR_STRING_CHARS(data)` 把 `XrString.data` 指针零拷贝穿透到 yield 之后。注释声称 "coroutine arena GC doesn't run while yielded, so direct reference is safe" —— 这依赖 GC 不主动搬迁协程数据的实现细节，**一旦未来 GC 改成增量/整理式就会静默错误** |
| N-7 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/net.c:1100` | 🟡 | `strncpy(state->hostname, XR_STRING_CHARS(host), 255)` —— hostname 截断到 255 字节且不报错，SNI 超长直接失败 |
| N-8 | 全文 | 🟡 | 所有 yieldable 失败路径**只返回 `XR_NULL_VAL`，没有错误码**。脚本 `net.dial()` 拿 null 分不清 "DNS 失败 / 连接拒绝 / 超时 / OOM" |

### 1.2 `net/dns.c`（561 行）

**亮点**
- 256 entries + LRU + 5 分钟 TTL + round-robin 选址 —— 设计合理
- 双栈解析（IPv4 优先后 IPv6）
- 有 async pool 集成点（`xr_dns_resolve_on_async`）

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| D-1 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/dns.c:349-363` | 🔴🔴 | `xr_dns_resolve` **直接调 `getaddrinfo`，blocking 调用**。这是整个 stdlib 最严重的协程语义 bug 之一：`conn_pool.c` + `http_client.c` + `http_stream.c` + `xr_io_connect` 都依赖 `xr_dns_resolve`，任何一个走缓存未命中的路径都会**把 worker 线程 chen 几十到几百毫秒**。应该默认走 `xr_dns_resolve_on_async` |
| D-2 | 全文 | 🔴 | 7 处 `malloc/strdup/free` —— 缓存 entry 全走系统堆 |
| D-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/dns.c:388-431` | 🟠 | `xr_dns_resolve_on_async` 把 coro 挂起后返回 `true`，但注释说"Result available via coro->async_result" —— 这是**隐式协议**，没抽象成可审查的 API，调用者很容易忽略 |
| D-4 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/dns.c:472-514` | 🟠 | `connect_with_timeout` 用 `select()` —— FD_SETSIZE 限制，大 fd（> 1024）UB。另外成功后 `fcntl(fd, F_SETFL, flags)` 把 fd 改回阻塞，调用者如果想继续用 netpoll 就**已经错过非阻塞设置** |
| D-5 | 全文 | 🟠 | **无负缓存**：NXDOMAIN / SERVFAIL 没缓存，对坏域名反复 syscall |
| D-6 | 全文 | 🟠 | **无 Happy Eyeballs**（RFC 8305）：`xr_dns_dial` 串行尝试所有地址，单个 IP 超时要等满 `per_ip_timeout` 才换下一个；现代 HTTP client 标配并行 IPv4/IPv6 连接 |
| D-7 | 全文 | 🟡 | 无 DoH / DoT 支持 |
| D-8 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/dns.c:134-153` | 🟡 | LRU evict 走到 cache 满才触发，没有**主动过期清理**，冷数据占满 256 槽位 |

### 1.3 `net/tls.c`（598 行）

**亮点**
- OpenSSL 现代 API（`OPENSSL_init_ssl` / `SSL_CTX_set_min_proto_version(TLS1_2_VERSION)`）
- SNI + hostname 验证（`SSL_set1_host` + `X509_CHECK_FLAG_NO_PARTIAL_WILDCARDS`）—— **证书校验正确**
- ALPN 客户端/服务端支持 + 默认 h2 > http/1.1 选择
- 协程友好 `_try` API 返回 `-1/-2/-3` 语义清晰

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| T-1 | 全文 | 🔴 | 12 处 `calloc/malloc/free` |
| T-2 | 全文 | 🟠 | **两套 API 并存**：`xr_tls_conn_handshake_client` / `_read` / `_write`（内部 while 循环 + yield via `xr_socket_read`）vs `_try` 版本。新代码用 try，但旧 API 还在。技术债 |
| T-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/tls.c:228-234` | 🟡 | `alpn_str` 是 `_Thread_local` 静态 32 字节，跨两次 `xr_tls_conn_get_alpn` 调用之间会被覆盖。调用方必须立即拷贝。文档未说明 |
| T-4 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/tls.c:545-550` | 🟠 | `xr_tls_conn_close` 只做 `SSL_shutdown(ssl)` 单向，不等 peer close_notify —— 对短连接 OK，对复用的 TLS 连接会遗留 `close_notify` 在线上 |
| T-5 | 全文 | 🟠 | **无 session resume** / **无 client cert** / **无 OCSP stapling** / **无自定义 CA store**。面向生产的 TLS 客户端这些都是必需 |
| T-6 | 全文 | 🟡 | `SSL_CTX_set_default_verify_paths` 在一些发行版（Alpine）找不到 CA bundle 会静默失败；应探测并 fallback 到 mozilla bundle |

### 1.4 `net/conn_pool.c`（424 行）

**问题**（几乎全是硬伤）

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| P-1 | 全文 | 🔴🔴 | **`create_connection` 用阻塞 `connect()`**（line 78）+ **阻塞 `xr_tls_conn_handshake_client`**（line 108）+ **`xr_pooled_conn_read/write` 直接走 `recv/send`**（line 411/423）。全链路阻塞。因此 `http_client.c` 走的 fast path 下连接池取得的每一个连接，IO 都是**同步的**，协程毫无作用 |
| P-2 | 全文 | 🔴 | 12 处 `calloc/strncpy/free` |
| P-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/conn_pool.c:224-227` | 🟠 | 连接复用跳过 liveness 检测（不做 `recv(MSG_PEEK)`）—— 注释说"lazy detection"。实际后果：对方 FIN 后下次 send 才能 detect，用户请求会**收到 connection reset 失败一次，有 retry 才成功**；如果业务没 retry → 间歇失败 |
| P-4 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/conn_pool.c:310-315` | 🟠 | 每 host 限 `XR_POOL_MAX_CONNS_PER_HOST` 个 idle，**超出直接 close** —— 没有等待队列，突发请求会把 keep-alive 踢掉 |
| P-5 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/conn_pool.c:145-162` | 🟡 | `idle_timeout_ms` 来自宏 `XR_POOL_MAX_IDLE_TIME`，无 per-pool 设置 |

### 1.5 `net/io.c`（578 行）

**亮点**
- 把 `dns + tls + netpoll + socket` 打包成 `XrIOConn` 抽象
- `xr_io_writev` 做了 iovec + EAGAIN 混合路径，TLS 自动 fallback to 顺序 write

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| I-1 | 全文 | 🔴 | 8 处 `calloc/free` + `malloc` |
| I-2 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/io.c:44-52` | 🟠 | `_Thread_local XrayIsolate *tls_isolate` —— **只支持每线程 1 个 isolate**。对多 isolate 场景需要在每次调用时显式传，TLS 隐式耦合 |
| I-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/io.c:162-164` | 🟠 | `xr_io_connect` 调 `xr_dns_resolve`（阻塞）+ 然后 netpoll 非阻塞 connect —— **前半同步、后半异步**，不一致 |
| I-4 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/io.c:448-475` | 🟠 | `xr_io_listen` IPv6 dual-stack 优先、IPv4 fallback —— 逻辑正确但没做 `SO_REUSEPORT`（多 worker 共享 listen fd 的现代做法） |

### 1.6 `net/xnetbuf.c`（194 行）

**亮点**
- Consume/reserve 模式 + 自动 compact + 指数增长（1MB 以下 2x、以上 1.25x）
- **TLS 级回收池**（线程本地，无锁、无 atomic）避免跨线程争用

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| B-1 | 全文 | 🔴 | 9 处 `malloc/free` |
| B-2 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/xnetbuf.c:138-193` | 🟠 | TLS 池**线程退出时不清理**，长期运行的线程累积内存（每个池 8 × 上限 cap） |
| B-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/xnetbuf.c:67-86` | 🟡 | `xr_netbuf_reserve` 先尝试 compact，失败才 realloc —— **实际 realloc 时没用 `xr_realloc` 的安全中转**，失败会丢原缓冲 |

---

## 2. `http` 子模块

### 2.1 `http/http_parser.c`（1208 行）—— 整组最亮眼的模块

**亮点**
- **SSE4.2 / NEON SIMD 加速**的字符扫描（`findchar_fast`、`get_token_to_eol`）
- **零拷贝**：所有 header name/value 指向原始 buffer，不做任何 `malloc`
- **Chunked 解码就地改写**：`xr_http_decode_chunked` 把 chunked encoding 直接压回 buffer，节省一次拷贝
- Token 字符查找表（RFC 7230 tchar）
- 防 slowloris 的 `is_complete` 快速检测
- **全文 0 处 `malloc`**，零红线违反 —— 是全组唯一干净的文件

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| PR-1 | 全文 | 🟠 | **无 `XRAY_API`/`XR_FUNC`**（和其它文件一样） |
| PR-2 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http_parser.h:295-311` | 🟠 | `xr_http_parse_content_length` 是 header line 线性扫描 —— 已有结构化 header 数组，应该复用 |
| PR-3 | 全文 | 🟡 | **`strict` 模式缺失**：对 folded header（过时但存在的 RFC 7230 obs-fold）、非法 `\r\0`、长度超限 header **默认容忍**，不符合严谨代理的需求 |

### 2.2 `http/http.c`（1451 行）+ `http_listen.c`（1175 行）

**亮点**
- `URL_COPY_BEGIN/END` 栈缓冲优化（2KB 以下零 `malloc`）
- `shape_conn` / `shape_listener` 复用同一 Shape —— Json handle 字段访问 O(1)
- Streaming 响应槽位（`XR_HTTP_MAX_STREAMS=16`）—— 支持 `http.request({stream:true})` + `http.readChunk`
- 预构建响应头（`HTTP_200_PLAIN` 等）+ C-level uint→ascii —— `response` 组装零 `sprintf`
- **native-stackless 协程 accept 循环**（独立 VM 栈，不切换 C 栈）性能对标裸 C server

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| H-1 | 整个 http/ | 🔴🔴 | **285 处直接 `malloc/free/calloc/realloc`**。比整个 stdlib 其它目录加起来还多 |
| H-2 | 所有文件 | 🔴 | 完全无 `XRAY_API`/`XR_FUNC` |
| H-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http.c:86-94` | 🟠 | `URL_COPY_BEGIN` fallback 分支用 `malloc`，失败不处理（`_url_need_free = true` 无 NULL 校验）→ 后续 `memcpy(NULL, ...)` 崩 |
| H-4 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http.c:162-170` | 🟠 | `result_to_json` 每个 header 都 `malloc(name_len+1)` 只为了做 null-termination 给 `xr_json_set_by_key` —— 应该提供 `_by_key_len` 版本省掉这个 |
| H-5 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http.c:444-467` | 🟠 | `XR_HTTP_MAX_STREAMS = 16` 硬编码。同一 isolate 并发 17 个 streaming 下载直接失败 |
| H-6 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http.c:479-482` | 🟠 | `sr->headers[i].name_len >= 128` 就截断 header 名 —— 静默错误 |
| H-7 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http.c:934-938` | 🟠 | `http.setConnHandler` 对 closure 有 upvalue 时**只打印 warning 继续接受**，脚本侧看不到 race condition 风险 |
| H-8 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http.c:987-1000` | 🟠 | `http_parse_request_fast` 只 `read` 一次就决定成败，**默认 4096 字节** —— 大 header 直接失败且不做增量 parse |

### 2.3 `http/http_client.c`（938 行）—— 技术债重灾区

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| HC-1 | 全文 | 🔴🔴 | **走阻塞 conn_pool 路径**（见 P-1）—— `xr_http_request_internal` 的 send/recv 都 block |
| HC-2 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http_client.c:36-50` | 🟠 | **`static XrConnPool *g_http_pool = NULL`** 进程全局 —— 违反 `XrHttpContext` 文档声明的"per-isolate pool"。`ctx->conn_pool` 字段根本没被 `xr_http_request` 使用（全局 `g_http_pool` 优先） |
| HC-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http_client.c:104-191` `xr_http_url_parse` | 🟠 | URL 解析器手写。**对 userinfo `http://user:pass@host`、IPv6 `http://[::1]:8080/`、URL-encoded host、非法字符都不支持** —— 直接和 `stdlib/url` 模块功能重复且质量更低 |
| HC-4 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http_client.c:260-268` | 🟠 | `build_request` 的 `buf_size = 1024 + body_len + path + host + headers` —— **没计算 Cookie / User-Agent / Host / Accept-Encoding 等自动添加 header**。长 Cookie + 极端场景**缓冲区溢出** |
| HC-5 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http_client.c:63-100` | 🟡 | `connect_with_timeout` 标注 `__attribute__((unused))` —— **死代码**（走 conn_pool 后已不用）但还编译进 binary |
| HC-6 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http_client.c:456-461` | 🟡 | 303 重定向强制 GET 的代码路径 **在 `xr_http_result_free(&result)` 之后读 `result.status_code`** —— `status_code` 是 `int` 确实没被释放，但顺序依赖 `status_code` 在 `free` 内不被清零，脆弱 |
| HC-7 | 全文 | 🟠 | **无请求取消**：`XrHttpReqContext.cancelled` 标志字段存在但 hot path 完全不检查。长查询只能等超时 |
| HC-8 | 全文 | 🟠 | **无 HTTP/2 自动协商路径到 http_client**：用户调 `http.get()` 永远是 HTTP/1.1，只有显式 `http2_client` API 才走 h2 |

### 2.4 `http/http2.c`（1151 行）—— 协议不完整

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| H2-1 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http2.c:178-183` | 🔴🔴🔴 | **HPACK Huffman 解码整段没实现**：注释直接写 "Huffman not implemented yet, just copy"。**实际后果**：几乎所有现代 HTTP/2 服务器（nginx、envoy、h2o）都会对 `:path`、`content-type` 之类高频 header 做 Huffman 编码发来；当前代码把它当普通 ASCII memcpy，脚本拿到的 header 是**乱码**。**HTTP/2 模块在生产里基本不可用** |
| H2-2 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http2.c:740-760` | 🔴 | `DATA` frame 完全不识别 `XR_H2_FLAG_PADDED`：pad length 字节 + padding 字节全部当作 body 写进 `stream->data_buf`。用户拿到的 HTTP body **前后多出垃圾字节**。同样问题出现在 `HEADERS`、`PUSH_PROMISE` |
| H2-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http2.c:745` | 🔴 | `realloc(stream->data_buf, ...)` 无临时指针中转 —— **OOM 时原 buffer 丢失 → leak + 下次 read NULL 解引用** |
| H2-4 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http2.c:785-795` | 🟠 | `WINDOW_UPDATE` 累加 `conn->connection_window += inc` —— **不检查 overflow**（RFC 7540 6.9.1: 不能超过 2³¹-1，超出是 FLOW_CONTROL_ERROR） |
| H2-5 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http2.c:678-846` | 🟠 | `xr_h2_recv` + `xr_h2_recv_stream_data` 是**同步 while 循环**，peer 慢就冻 worker。应该写成 yieldable continuation |
| H2-6 | 全文 | 🟠 | 38 处直接 `malloc/free/realloc/calloc` |
| H2-7 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http2.c:211-232` | 🟠 | HPACK 动态表插入是 O(N) 线性查尾节点 —— 应该改双向链表 |
| H2-8 | 全文 | 🟠 | **无 PRIORITY 树**：收到 PRIORITY frame 仅存最后一个，没 dependency tree；多 stream 场景调度全靠 OS |
| H2-9 | 全文 | 🟠 | `XR_H2_STREAM_HASH_SIZE = 64` 固定；超过 64 个并发 stream hash 退化 |

### 2.5 `http/http_compress.c`（475 行）

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| HCP-1 | 全文 | 🟠🟠 | **功能与 `stdlib/compress` 重复**：后者从零实现了 deflate + gzip + zlib，这里却用 `#ifdef XR_ENABLE_ZLIB` 包系统 zlib。一个进程里两套压缩实现互不调用 |
| HCP-2 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http_compress.c:77-80` | 🟠 | `realloc` 用临时指针 `new_buf` 正确，但 line 107 最终 shrink `realloc` 没用临时（仅仅侥幸 shrink 不失败） |
| HCP-3 | 全文 | 🟡 | **无 Brotli / Zstd**，但 `Accept-Encoding` 服务器端常见；这导致 xray HTTP client 不能享受 `br`/`zstd` 压缩 |
| HCP-4 | 全文 | 🟡 | 同步 `inflate` 循环没有最大输出长度上限 —— **zip bomb 风险** |

### 2.6 `http/http_cookie.c`（470 行）

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| CK-1 | 全文 | 🔴 | 24 处 `malloc/strdup_n/free` |
| CK-2 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http_cookie.c:93-112` | 🟠 | `parse_cookie_date` 用 `strptime` —— POSIX 扩展，Windows 不可用。应该自写 RFC 1123 / RFC 850 parser |
| CK-3 | 全文 | 🟠 | **无 Public Suffix List**：`domain_matches` 允许 `.com` 这种 public suffix cookie 匹配所有 `*.com`，**隐私与安全问题** |
| CK-4 | 全文 | 🟠 | **无 `SameSite` 支持**（现代浏览器强制的 CSRF 防御）—— xray 作为 client 不发，作为 server 不解析 |
| CK-5 | 全文 | 🟠 | `XrCookieJar` 无上限（内存或条目数）—— DoS 风险：恶意 server 发 10000 个 Set-Cookie |

### 2.7 `http/http_multipart.c`（305 行）

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| M-1 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http_multipart.c:23-38` | 🔴 | `generate_boundary` **用 `rand()` + `srand(time(NULL))`**：时钟 1 秒粒度 → 同一秒内生成的 boundary **完全相同**；`rand()` 本身非加密。**如果攻击者能猜中 boundary，可在上传内容里注入假的 multipart part**。应改用 `xr_random_bytes`（crypto 模块已实现）或 `getentropy`/`BCryptGenRandom` |
| M-2 | 全文 | 🔴 | 19 处 `malloc/calloc/strdup/free` |
| M-3 | 全文 | 🟠 | 没有**服务端 multipart 解析**，只有客户端编码；上传表单服务器拿不到 file parts |

### 2.8 `http/http_stream.c`（310 行）

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| HS-1 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http_stream.c:84` | 🔴🔴 | `fopen` + `fwrite` + `select(2)` —— **全同步 IO**。下载大文件期间整个 worker block。应该走 `stdlib/io` 的异步写路径 |
| HS-2 | 全文 | 🔴 | 7 处 `malloc/strdup/free` |
| HS-3 | 全文 | 🟠 | 无断点续传（Range 头）尽管文档暗示 "resume support" |

### 2.9 `http/http_router.c`（518 行）

**亮点**
- Radix Tree 路由，O(k) 匹配
- 支持 param (`:id`) 和 wildcard (`*`) 两种特殊子节点
- 静态路由预构建响应（bypass VM 快路径）

**问题**

| # | 位置 | 严重 | 问题 |
|---|------|------|------|
| R-1 | 全文 | 🔴 | 15 处 `malloc/calloc/realloc/free` |
| R-2 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http_router.c:59-62` | 🔴 | `realloc` 不用临时指针（`parent->children = realloc(parent->children, ...)`）—— 失败丢原指针 |
| R-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/http/http_router.c:78-86` | 🟠 | `find_child` 线性扫描子节点 —— k 大时慢；应按字符排序 + 二分，或用 256-entry 跳转表（Radix 经典实现） |
| R-4 | 全文 | 🟠 | 无 method-specific 路由（path 相同 method 不同）的智能分派，依赖上层先 match method |

---

## 3. 跨模块系统性问题与抽象缺失

### 3.1 协程语义"瑞士奶酪"

| 对外 API | 底层 | 实际是否非阻塞 |
|---------|------|--------------|
| `net.dial` | `net_tcp_connect` + yield | ✅ |
| `net.accept` | `xr_socket_accept_try` + yield | ✅ |
| `net.read/write` | handle-based yield | ✅ |
| `net.dialTLS` | yield TCP + yield handshake | ✅ |
| **`xr_udp_send_to` / `xr_udp_recv_from`（旧 API）** | **直接 `sendto`/`recvfrom`** | ❌ |
| **`http.get`/`http.post`** | `xr_http_request` → `conn_pool` → blocking `connect`/`recv`/`send` | ❌ |
| **`xr_dns_resolve` 缓存 miss** | `getaddrinfo` | ❌ |
| **`http2.c` 任意 `xr_h2_recv`** | 同步 while | ❌ |
| **`http_stream.xr_http_download`** | `fopen + select` | ❌ |

这意味着**脚本以为在用协程**的时候（`http.get("http://slow.com/")`），实际上整个 worker 线程被冻住直到远端响应 —— **默认 xray 'concurrent' 宣传与实际行为严重不符**。

### 3.2 连接池碎片化

- `conn_pool.c` (HTTP/1) —— 全局 `g_http_pool` + 文档声明的 `ctx->conn_pool`（实际未使用）
- `http2_client.c` `XrH2Pool` —— legacy global + 新的 per-isolate
- `xnetbuf.c` 缓冲池 —— per-thread TLS pool
- 三套生命周期管理互不协同

### 3.3 错误传播断层

`xneterror.c` 定义了完整错误枚举（`XR_NERR_DNS/CONNECT/READ/WRITE/TLS/*`），但：
- `net.*` binding 全部 `return XR_NULL_VAL`（无错误字段）
- `http.*` binding 在 `result_to_json` 里有 `error` 字段，但**只返回 `xr_http_error_string(err)` 字符串**，不返回枚举；脚本无法 `match`
- 协程路径的 state 结构里完全没有 error 字段

### 3.4 HTTP/2 协议不完整 —— **不可用于生产**

最直白：**HPACK Huffman 解码没实现**。只要对端发 Huffman 编码（RFC 7541 5.2 允许，且主流服务器默认开），xray 就解出乱码 header。这是**P0 bug**。

---

## 4. 按 ROI 排序的 PR 建议

| 序号 | 工作量 | 优先级 | 建议 |
|------|-------|--------|------|
| **P1** | S（半天） | 🔴🔴🔴 | **修 HPACK Huffman 解码**（`http2.c:178-183`）。RFC 7541 Appendix B 有完整 Huffman 表，200 行代码 + 1 个查找树。不修 HTTP/2 客户端在生产几乎不可用 |
| **P2** | S | 🔴🔴 | **修 HTTP/2 PADDED flag + realloc 丢指针**（H2-2、H2-3）。~30 行 |
| **P3** | S | 🔴🔴 | **multipart `generate_boundary` 改用 `xr_random_bytes`**（M-1）。5 行改动，消除 CSRF/上传注入风险 |
| **P4** | S | 🔴 | **`XrHttpContext.conn_pool` 真正生效 + 删掉 `g_http_pool`**（HC-2）。把 `xr_http_request_internal` 里的 `g_http_pool` 改成 `xr_http_get_context(X)->conn_pool` |
| **P5** | M（2-3 天） | 🔴 | **DNS 默认走 async**（D-1）。`xr_dns_resolve` 在有 coroutine 上下文时自动跳到 `xr_dns_resolve_on_async`；加 yield wait。彻底消除"缓存 miss 冻线程"问题 |
| **P6** | L（1 周） | 🔴 | **`conn_pool.c` 全链路重写为协程友好**：`create_connection` 走 `net_tcp_connect` + yield handshake；`xr_pooled_conn_read/write` 走 `xr_socket_read/write` 非阻塞路径 |
| **P7** | L | 🔴 | **`http_stream.c` 重写**：`fopen/fwrite` → `xr_io.write_async` / 协程 `write`（同 `http_listen.c` 的写法） |
| **P8** | M | 🟠 | **三模块红线替换**：`malloc/free` → `xr_malloc/xr_free`；机械工作约 400 处。可以一个 PR 按模块切分提交 |
| **P9** | M | 🟠 | **公共 API 统一加 `XRAY_API`/`XR_FUNC`**（配合 P8） |
| **P10** | M | 🟠 | **IPv6 地址解析支持**：`xr_net_parse_addr` + `xr_http_url_parse` 支持 `[::1]:8080` 格式（N-4、HC-3）。最简单是把 `http_client.c` 的 URL 解析换成复用 `stdlib/url` 模块 |
| **P11** | L | 🟠 | **HPACK 动态表改双向链表**（H2-7）+ **Stream hash 改动态扩容**（H2-9） |
| **P12** | L | 🟠 | **Happy Eyeballs（RFC 8305）** —— IPv4/IPv6 并行连接（D-6） |
| **P13** | M | 🟠 | **Cookie**：`SameSite` 属性 + Public Suffix List 防御（CK-3、CK-4） |
| **P14** | L | 🟡 | **合并 `http_compress` 与 `stdlib/compress`**：HTTP layer 只做 Content-Encoding 分派，不重复实现 zlib（HCP-1） |
| **P15** | L | 🟡 | **统一 stdlib 错误对象** `{ok, value, error:{code, msg, detail}}`（和 serialization/crypto 组建议一起做） |
| **P16** | L | 🟡 | **零拷贝 header API**：避免 `result_to_json` 每个 header 都 `malloc` 一次（H-4） |
| **P17** | L | 🟡 | **TLS 补齐生产特性**：session resume、client cert、OCSP stapling（T-5） |

---

## 5. 测试覆盖补齐清单

### net
- ❌ **并发 DNS 缓存争用**（多协程同时 miss）
- ❌ **IPv6-only host**（Happy Eyeballs 缺失会暴露）
- ❌ **连接池 expired idle 的 first-request race**（P-3）
- ❌ **TLS SNI + cert verify**（强制用 self-signed 证书测 `XR_NERR_TLS_VERIFY`）
- ❌ **SO_REUSEPORT 多 listen fd 行为**
- ❌ **UDP MTU（>65K）边界**

### http
- ❌ **HTTP/2 真实服务器互操作**：对 `h2.akamai.com`、`nghttp2.org` 做端到端测试 —— 能直接暴露 H2-1 Huffman bug
- ❌ **HPACK RFC 7541 官方测试向量**（附录 C）
- ❌ **HPACK 动态表驱逐 + 巨型 header**
- ❌ **Chunked 边界**：`0\r\n\r\n` 最小 chunk、`CRLF` 前后空白、非法 hex size
- ❌ **Slowloris 攻击**：小包慢发请求头，期望 server 超时关闭
- ❌ **multipart 服务器端解析**（根本没实现 → 需要补实现 + 测试）
- ❌ **Cookie 安全套件**：跨域 cookie、SameSite、Secure-only、HttpOnly、Max-Age vs Expires 优先级
- ❌ **URL parser 对 IPv6 / userinfo / UTF-8 host 的兼容**（vs `curl --libcurl`）
- ❌ **Redirect loop**（301 → 302 → 301 → ...，期望 `max_redirects` 生效）
- ❌ **大响应（>100MB）**：测 `XR_HTTP_ERR_TOO_LARGE` 边界 + zip bomb 防御
- ❌ **Keep-Alive 连接复用后 peer FIN 的 first-request 失败**
- ❌ **Fuzz `http_parser`**（AFL/libfuzzer）—— 最关键的攻击面

---

## 6. 代码量与技术债粗算

| 模块 | 行数 | 直 `malloc` 处 | 阻塞调用 | bug 条目（本文档） | 重构工作量 |
|------|------|----------------|---------|--------------------|-----------|
| net | 4159 | 104 | 3 处（udp / dns / tls legacy） | 23 | 4-5 PD |
| http | 13357 | 285 | 5 处（conn_pool / http2 / stream / compress / multipart）| 52 | 10-15 PD |
| **合计** | **17516** | **~389** | **8 处阻塞 API** | **75 条** | **~15-20 PD** |

> 这个数字比基础工具组（~1-2 PD）、序列化组（~10-16 PD）、加解密+压缩+正则组（~10-16 PD）**都高**，且包含**协议级不完整**（HPACK Huffman）。

---

## 7. 附：按严重程度的 P0 清单（再次强调）

必须在"生产可用"标签之前完成的 4 件事：

1. **HTTP/2 HPACK Huffman 解码**（H2-1）—— 不做 xray 的 HTTP/2 客户端连 `curl --http2 https://example.com` 的 header 都解不出来
2. **HTTP/2 PADDED flag 正确处理**（H2-2）—— body 会带垃圾字节
3. **DNS 默认 async**（D-1）+ **conn_pool 非阻塞化**（P-1）—— xray "协程网络"的名声全靠这两个
4. **Multipart boundary 换 CSPRNG**（M-1）—— 上传注入漏洞

做完这 4 个，这组才从"玩具演示级"进入"能用但还有技术债"。
