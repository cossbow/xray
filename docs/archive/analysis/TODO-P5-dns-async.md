# P5 — DNS 默认走 async：未决 / 待 runtime 层专项

本文件记录 docs/analysis/stdlib_net_http.md 里 P5 的设计调研结论与待决项。

## 1. 现状摘要

- `xr_dns_resolve(hostname, addr, family)` 在缓存命中时 O(1) 返回；
  缓存未命中时直接调用 `getaddrinfo`，**整个 worker 线程阻塞** 数十到
  数百毫秒（@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/dns.c:349-363）。
- `xr_dns_resolve_on_async(pool, coro, worker_id, host, addr, family)`
  已存在，但**全项目零调用者**（@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/dns.c:388-431）。
- `xr_async_submit` 仅把协程标记为 `XR_CORO_FLG_BLOCKED` 并把 job 入队，
  **不触发协程上下文切换**（@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xasync.c:248-260）。
  它依赖调用者在返回后到达某个 "调度检查点"（VM opcode 边界或 CFunc 返回
  `XR_CFUNC_BLOCKED`）才真正 yield。
- `xr_dns_resolve_on_async` 因此在深 C 栈里调用是 **no-op**：函数返回
  `true` 时 `addr` 尚未被写入；doc D-3 已标注这是"隐式协议"。

## 2. 设计空间

| 方案 | 语义 | 协程并发收益 | 工作量 | 风险 |
|------|------|-------------|--------|------|
| A. 加 runtime 新原语 `xr_async_wait_coro(pool, job, X)`：内部做真协程上下文切换 | worker 可以去跑别的协程 | ✅ 有 | ~200 行 runtime 代码 + setjmp/ucontext | 高 — 触及 coro 调度核心 |
| B. 加 runtime 原语但只做 pthread_cond_wait（和 `xr_netpoll_block` 对称） | worker 线程 park，其他协程也跑不了 | ❌ 无 | ~40 行 | 中 — 架构一致但收益为零，属于引入假异步技术债 |
| C. 整条 HTTP 客户端链路状态机化 + CFunc yieldable | worker 可以跑别的协程 | ✅ 有 | ~1 周+，要重写 `http_client.c`、`conn_pool.c` 公共 API 边界 | 高 — 规模巨大 |

## 3. 为什么本轮未动

按项目原则 **"选最佳设计、不做兼容层、不引入技术债"**：

- **方案 B 违反原则**：代码新增但无实际收益，等同不改；把 blocking 位置从 worker 搬到 async thread，worker 还是全程 park。
- **方案 A / C 超出本次 scope**：都属于 runtime 层或 HTTP 架构重写，不是 stdlib 修复。需要专项立项。

## 4. 现阶段缓解手段（已实现）

P6 把 `conn_pool.c` 的 connect/read/write 改成非阻塞 + netpoll yield
（@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/net/conn_pool.c:62-145），
所以**除了 DNS 缓存未命中之外**，HTTP 请求已经不再冻结 worker。
DNS 缓存当前 TTL 5 分钟，命中率在稳态下应 > 95%，影响面可控。

## 5. 下一步可选

若决定做专项，推荐方案 A + 明确的 API 契约：

```c
// 语义：同步等待 async job 完成（coro 真 yield；worker 可以去跑别的）。
// 返回值：job 执行完的结果指针（通常是传入的 data）或 NULL（cancelled）。
// 前提：X 在协程上下文；runtime->async_pool 已就绪。
void *xr_async_wait_coro(XrAsyncPool *pool, XrAsyncJob *job, XrayIsolate *X);
```

调用方：

```c
bool xr_dns_resolve(const char *host, XrSockAddr *addr, XrAddrFamily fam) {
    if (cache_lookup(host, addr, fam)) return true;

    XrayIsolate *X = xr_io_get_isolate();
    if (X && X->vm.runtime) {
        XrRuntime *rt = (XrRuntime *)X->vm.runtime;
        XrCoroutine *coro = xr_current_coro(X);
        XrWorker *w = xr_current_worker();
        if (rt->async_pool && coro && w) {
            XrDnsAsyncData data = { ... };
            XrAsyncJob *job = xr_async_job_create(coro, w->p.id, dns_async_invoke, &data);
            xr_async_wait_coro(rt->async_pool, job, X);  // <- new primitive
            if (data.success) { *addr = data.addr; return true; }
            return false;
        }
    }
    return do_resolve(host, addr, fam);   // fallback: sync (tests, CLI)
}
```

核心工作在 `xr_async_wait_coro` 实现里 —— 要把当前 coro 从执行队列取下
挂到 job 上，让 worker 循环去 `xr_find_runnable` 找下一个协程执行。
复用 `xr_coro_transition_to_blocked` + `xr_waitq` 模式，参照 channel
实现（@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xchannel.c:694-772）
的 send/recv 阻塞语义。

## 6. 交叉引用

- 分析文档：`docs/analysis/stdlib_net_http.md` 第 D-1 / D-3 / P5 条目
- 相关代码：`stdlib/net/dns.c`、`src/coro/xasync.c`、`src/coro/xnetpoll.c`、`src/coro/xchannel.c`
