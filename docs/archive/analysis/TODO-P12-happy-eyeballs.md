# TODO-P12 — Happy Eyeballs v2 (RFC 8305) parallel connect

## 本次已落地

- `stdlib/net/dns.c::do_resolve_all` 现在把 IPv6 / IPv4 **交错输出**
  （RFC 8305 §4 destination address sorting），冷路径
  `getaddrinfo` 的系统排序不再决定实际连接顺序。
- `stdlib/net/conn_pool.c::create_connection` 遍历所有解析地址、
  遇到 `coro_tcp_connect` 失败就清理 netpoll + close(fd) 继续下一个。
  这是 **串行** failover —— 对"v6 路由被黑洞、v4 还活着"的常见场景
  已经能自救。
- 删除 `xr_dns_dial` / `connect_with_timeout`：用 `select()` 且无调用者，
  同时会把 fd 改回阻塞模式。保留它就是技术债。

## 为什么没做真正的并行 HE v2

RFC 8305 §8 要求：

> After initiating a connection attempt on an address of one family,
> a new attempt MUST NOT be started for 250 ms, at which point a
> connection attempt to an address of the other family will be started.

也就是说需要：

1. 同时对 **多个 fd** 非阻塞 `connect()`，
2. 用 `xr_netpoll_block` 挂起 **协程**，但 netpoll 当前的
   `XrPollDesc` 是一 fd 对一 pd，`xr_netpoll_block` 只能等单个 fd 可写，
3. 引入 **timer**（250 ms stagger），runtime 目前没有原生 coroutine
   timer（只有 `sleep` 的粗粒度 yield）。

真正并行版本需要一个新 runtime 原语：

```c
// Wait for ANY of the given pds to become ready, or the timer to fire.
// Returns the index of the ready pd, or -1 on timeout.
int xr_netpoll_block_any(XrRuntime *rt, XrPollDesc **pds, int n,
                          int events, uint64_t timeout_ms);
```

实现层面需要：

- `pd` 上挂 **多个 waiter coroutine**（当前是单 g link，需要换成 list）
- runtime 里加 **timer wheel** 或 `timerfd_create`-style 机制，让
  `xr_netpoll_block_any` 可以在到期时主动 wake
- 被唤醒的 waiter 要能**撤销**仍在 pending 的其它 pd 上的注册
  （避免 ABA：后续 fd ready 时 park 列表已经空了）

## 优先级判断

- **P5 (async DNS)** 和 **P12 (parallel HE)** 阻塞点是同一套 runtime
  能力：多 fd 等待 + timer。
- 合理的节奏：先把 runtime 那层的通用原语做出来
  （参见 `TODO-P5-dns-async.md`），HE v2 只是其上层一个 ~120 行的 helper。
- 在 runtime 原语到位之前，继续用当前的串行 failover 版本。现实
  语义损耗：IPv6 完全不可达时最坏多等一个 TCP SYN 的 RTT
  （典型 ~1 s，服务端 blackhole 则走 TCP 超时 ~20 s）。

## 真要做的时候的实现轮廓

```c
// In conn_pool.c::create_connection(...)
XrSockAddr addrs[XR_DNS_MAX_ADDRS];
int n = xr_dns_resolve_all(host, addrs, XR_DNS_MAX_ADDRS, XR_AF_UNSPEC);

// Spawn up to n non-blocking connects, staggered by 250 ms.
XrConnectAttempt attempts[XR_DNS_MAX_ADDRS] = {0};
int started = 0, winner = -1;
uint64_t next_start_ms = now_ms();

while (winner < 0 && (started < n || has_pending(attempts, started))) {
    if (started < n && now_ms() >= next_start_ms) {
        attempts[started].fd    = socket(addrs[started].family, ...);
        attempts[started].pd    = xr_netpoll_open(runtime, fd);
        xr_io_set_nonblocking(fd);
        connect(fd, sa, sa_len);                 // EINPROGRESS expected
        started++;
        next_start_ms += 250;                    // RFC 8305 stagger
    }

    uint64_t wait_ms = next_start_ms > now_ms()
                        ? next_start_ms - now_ms() : 0;

    int idx = xr_netpoll_block_any(runtime,
                pds_of_pending(attempts, started),
                npending(attempts, started),
                POLLOUT, wait_ms);

    if (idx >= 0) {
        int err = 0; socklen_t len = sizeof(err);
        getsockopt(attempts[idx].fd, SOL_SOCKET, SO_ERROR, &err, &len);
        if (err == 0) { winner = idx; break; }
        // else: this attempt failed; keep waiting for others
        cleanup_attempt(&attempts[idx]);
    }
}

// Close losers, keep the winner.
for (int i = 0; i < started; i++)
    if (i != winner) cleanup_attempt(&attempts[i]);
```

## 测试计划

- 单测无法真造双栈路由，用 **mock DNS + mock socket** 覆盖：
  - v6 全 ECONNREFUSED → 回落 v4
  - v6 黑洞（connect 不返回）→ 250 ms 后起 v4，v4 先 ready
  - 二者都失败 → 返回 -1
- 真实网络 smoke：`curl`-style 命令跑几个公共双栈主机
  （`ipv6.google.com` / `he.net`）。
