# stdlib/http 分析与优化建议

## 模块职责

`stdlib/http` 是当前标准库中职责最重的模块之一，覆盖：

- HTTP/1.1 client：`get/post/put/delete/request`
- HTTP/1.1 server：`route/static/listen/config/response/serverStats`
- URL utility：`urlEncode/urlDecode`
- raw HTTP parser/sender：`parseRequest/sendResponse`
- streaming response：`request({stream:true})` / `readChunk` / `closeStream`
- download：`download/getContentLength`
- multipart form：`formDataNew/formDataAppend/formDataAppendFile/formDataPost`
- proxy/cookie/connection pool
- HTTP/2 client/server：`h2Get/h2Post/h2Request/h2CreateServer/h2Listen/h2Stop/h2Push`

模块已经拆成多个 C 文件，但 public module surface、builtin 类型声明、LSP、HTTP/1/HTTP/2 行为和测试覆盖之间仍存在明显漂移。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/http/http.h` | module context、loader、listen/config/response/serverStats 声明 |
| `stdlib/http/http.c` | module binding、HTTP/1 client wrapper、URL encode/decode、server route/static、stream handle |
| `stdlib/http/http_client.c` | HTTP/1 request 内核、connection pool、response parse、chunked/decompress/cookie |
| `stdlib/http/http_server.c` | HTTP/1 server/router、request read、response send、GC root marking |
| `stdlib/http/http_listen.c` | yieldable listen loop、server config、response helper、server stats |
| `stdlib/http/http_parser.*` | HTTP request/response parser |
| `stdlib/http/http_stream.*` | streaming download、range、redirect config |
| `stdlib/http/http_multipart.*` | multipart form data |
| `stdlib/http/http_cookie.*` | cookie parser/jar |
| `stdlib/http/http_proxy.*` | proxy config/no_proxy |
| `stdlib/http/http2_*` | HTTP/2 frame/HPACK/client/server/binding |
| `stdlib/net/conn_pool.*` | TCP/TLS connection pool |
| `stdlib/compress/compress_zlib.c` | gzip/deflate decompress helpers |

## 当前 API 与导出

### HTTP/1 client

| API | 声明 | 实现要点 |
|---|---|---|
| `get(url, options?)` | `(url: string, options?: Json): HttpResponse` | 实现只读取 url，不读取 options |
| `post(url, body?, options?)` | `(url: string, body?: string, options?: Json): HttpResponse` | 实现第三参数是 contentType string，不是 options |
| `put(url, body?, options?)` | 同 post | 实现第三参数是 contentType string |
| `delete(url, options?)` | `(url: string, options?: Json): HttpResponse` | 实现只读取 url |
| `request(...)` | 声明为 `(method, url, options?)` | 实现实际要求单个 Json options object |

### HTTP/1 server

- `route(method, path, handler)`
- `static(...)`
- `setConnHandler(...)` / `__getConnHandler()`
- `listen(port)` yieldable accept loop
- `config(opts)`
- `response(status, body?, headers?)`
- `serverStats()`
- `ws(...)` legacy route entry, WebSocket 已拆到 `ws` module

### Utility/advanced

- `urlEncode()` / `urlDecode()`
- `parseRequest()` / `sendResponse()`
- `download()` / `getContentLength()`
- `formData*()`
- `setProxy()` / `clearProxy()`
- `h2*()`
- `readChunk()` / `closeStream()`

## 当前架构优点

- HTTP context 是 per-isolate，避免全局 server/context 的明显冲突。
- client 使用 connection pool，TCP/TLS 连接可复用。
- client binding 使用 `XRS_EXPORT_SLOW` 标识阻塞路径，能触发 runtime 对慢调用的处理。
- server listen 使用 yieldable C function，可配合 coroutine I/O。
- server route closure 有 GC root marking，避免 route handler 被回收。
- HTTP/1 parser、multipart、cookie、proxy、HTTP/2 已拆文件，职责比单文件集中更清晰。
- response body 使用 length-aware string，可以承载二进制 body。
- chunked response 和 Content-Encoding 解压已在 client 内核处理。
- `config()` 使用 atomic 保存 server limits/stats。

## 关键问题

### 问题 1：builtin 声明与实现参数形态不一致

最明显的是 `http.request`：

```text
builtin: (method: string, url: string, options?: Json): HttpResponse
implementation: http.request(options: Json)
```

实现读取 fields：`url/method/body/timeout/headers/stream`。

影响：

- 静态分析提示用户错误调用方式。
- 真实测试 `http.request({ ... })` 只测试对象构造和 API 存在，没有执行请求，因此未暴露。

建议：

- 统一为 `request(options: Json)`。
- 或实现兼容 `(method, url, options?)` wrapper。
- LSP/analyzer/README/snippet 同步。

### 问题 2：`get/delete` 声明有 options，但实现忽略

`http_get()` 和 `http_delete()` 只读取 URL，不读取 timeout、headers、stream、proxy 等 options。

影响：

- 用户传 headers/timeout 无效。
- 和 `request()` 能力不一致。

建议：

- `get(url, options?)` 内部构造 `XrHttpRequestConfig`。
- `delete(url, options?)` 同理。

### 问题 3：`post/put` 第三个参数语义漂移

声明是 `options?: Json`，实现却把第三参数当 content type string：

```text
post(url, body?, contentType?)
put(url, body?, contentType?)
```

建议：

- 统一为 `post(url, body?, options?)`，其中 `options.headers` 或 `options.contentType` 设置 Content-Type。
- 或修改声明为 `(url, body?, contentType?: string)`。

### 问题 4：LSP HTTP 符号极不完整

LSP 仅暴露：

```text
route listen get post
```

builtin/analyzer 已有 23 个 HTTP functions 和 3 个 handles。LSP 漏掉 `request/delete/put/urlEncode/urlDecode/download/formData/h2/readChunk` 等。

建议：

- 以 builtin generated 为单一来源同步 LSP。
- 避免手写 LSP symbol 继续漂移。

### 问题 5：HTTP/2 binding 声明与实现也不一致

`h2_request()` 实现要求：

```text
h2Request(options: Json)
```

builtin 声明却是：

```text
(method: string, url: string, options?: Json): HttpResponse
```

`h2_get/h2_post` 是否完整处理 options 也需要进一步对齐。

建议：

- 统一 HTTP/1 `request` 与 HTTP/2 `h2Request` 的 options-object 风格。

## Client 行为与安全

### 问题 6：timeout 变量在 HTTP/1 internal request 中未实际使用

`xr_http_request_internal()` 计算：

```c
int timeout_ms = config->timeout_ms > 0 ? config->timeout_ms : XR_HTTP_DEFAULT_TIMEOUT;
(void) timeout_ms;
```

随后连接池 read/write 未显式传入 timeout。

影响：

- `request({ timeout })` 可能无效。
- 网络调用可能长期阻塞，虽然标记 SLOW，但用户级 timeout 不可靠。

建议：

- 将 timeout 贯穿 DNS/connect/TLS/read/write。
- 为 response header/body read 分别设置 deadline。

### 问题 7：HTTP client response size limit 固定且不可配置

client receive buffer capacity 超过 100MB 时失败。server request body 限制硬编码 10MB。multipart 有默认限制。

建议：

- 将 `maxResponseBytes/maxRequestBodyBytes/maxHeaderBytes` 暴露到 options/config。
- 默认值合理，但用户可调整。

### 问题 8：Content-Encoding 解压没有 output cap

HTTP client 使用：

```c
xr_zlib_gzip_decompress(...)
xr_zlib_deflate_decompress(...)
```

这些 helper 动态扩容，没有 max output 限制。WS 路径已有 `max_out` zip-bomb guard。

建议：

- HTTP client 根据 `maxResponseBytes` 使用 bounded decompress。
- 解压后大小也必须计入 response limit。

### 问题 9：解压失败静默返回原始压缩 body

Content-Encoding 解压失败时当前逻辑 copy raw body 并返回成功 response。

影响：

- 用户看到乱码/压缩数据，但 `error` 仍为 null。
- checksum/corrupt gzip 错误不可观测。

建议：

- 在 response 中加 `decodeError`。
- 或解压失败时设置 `error`。
- 提供 `autoDecompress: false` option。

### 问题 10：redirect 语义不统一

`http_stream` 有 `follow_redirects` config，但普通 `get/request` 未见一致 redirect handling。测试 `1506_http_redirect` 只是本地状态码分类逻辑，不测试真实 redirect。

建议：

- client options 暴露 `followRedirects/maxRedirects`。
- GET/POST redirect 方法重写语义明确化。

### 问题 11：header 表示会丢失重复 header

`result_to_json()` 将 headers 写入 Json object，以 header name 为 key。重复 header（如 `Set-Cookie`）会被覆盖或只保留最后/某一个。

虽然内部 cookie jar 会收集最多 64 个 Set-Cookie，但返回给用户的 `headers` 不保留多值。

建议：

- `headers` 改为 Map<string, string | Array<string>> 不符合类型系统时可用 `headersList: Array<{name,value}>`。
- 至少为 `setCookie` 提供数组字段。

### 问题 12：header 名长度截断风险

stream mode 构造返回 headers 时使用 `char buf[128]`，header name 超过 127 会截断。

建议：

- 使用 length-aware string key 或动态分配。
- 对超长 header name 报错。

### 问题 13：URL encode/decode 对非法 percent encoding 宽松保留

`urlDecode()` 遇到无效 `%XX` 时原样保留 `%`。空字符串测试也允许 null 或空串。

建议：

- 文档明确 lenient 行为。
- 增加 strict decode 或错误返回。

## Server 行为与并发

### 问题 14：server 是 per-isolate singleton

`http.route()` auto-create `ctx->server`，一个 isolate 下默认单 server。多 server/multi-port 需求不明确。

建议：

- 明确当前是 singleton server API。
- 如需多 server，提供 `createServer()` handle API。

### 问题 15：`http.static` 声明与实现不一致

builtin 声明：

```text
static(prefix: string, dir: string): void
```

实现注释和代码是：

```text
http.static(method, path, content)
```

它并不是目录静态文件服务。

建议：

- 改名或修正实现/声明。
- 若需要目录静态服务，增加文件安全边界：路径归一化、防目录穿越、MIME、cache。

### 问题 16：request body read 没有处理 header 后已读 body leftover

`xr_http_read_request()` 读取 header 时一旦找到 `\r\n\r\n` 会将 `total` 截到 header end，然后后续 body read 从 socket 读取 `Content-Length` 字节。若原始 read 已经读到了部分 body，截断后这部分数据丢失。

影响：

- client 在同一 TCP packet 中发送 headers+body 时，server 可能阻塞等待已经丢弃的 body。

建议：

- 保留 header_end 之后的 leftover body bytes。
- body buffer 先复制 leftover，再继续 socket read 剩余部分。

### 问题 17：request header size/body size 常量不统一

`http_server.h` 定义 `XR_HTTP_MAX_HEADER_SIZE` 和 `XR_HTTP_MAX_BODY_SIZE`，但 server body read 使用硬编码 10MB；header read 依赖调用方 buffer size。

建议：

- 所有限制统一到 config/default constants。
- `http.config()` 暴露并使用这些限制。

### 问题 18：route handler GC root 绑定 owner coroutine GC，生命周期复杂

server route closure 注册到当前 coroutine 的 GC root。若 server 长期运行，owner coroutine 结束或 GC 生命周期变化，需要确认 root unregister 和 closure lifecycle 正确。

建议：

- 将 route handler roots 绑定 isolate/module context，而非某个临时 coroutine。
- 增加 server start/stop/GC lifecycle 测试。

## HTTP/2 边界

### 问题 19：HTTP/2 server callback 未接入脚本 handler

`h2_request_callback()` 设置 `current_h2_ctx` 后立即清空，注释说明 request processing 依赖脚本 handler，但当前片段未见实际调用。

建议：

- 明确 HTTP/2 server 是否可用。
- 如果未完成，API 应标 experimental 或不导出。

### 问题 20：HTTP/2 tests 只验证 API 存在

`1510_http2_client.xr` 没有真实连接、frame、TLS、HPACK 或 response 验证。

建议：

- 增加本地 h2 loopback 或 mock frame parser 单元测试。
- 对 h2 client/server 标记成熟度。

## 测试覆盖

现有覆盖：

- API 存在：HTTP client/server/H2。
- URL encode/decode 基础和 edge cases。
- headers/options object 构造。
- status code 分类辅助逻辑。
- response body 模拟对象。
- redirect status code 分类辅助逻辑。
- 少量 network tests 使用外部域名/httpbin。

主要缺口：

1. `http.request()` 真实调用参数形态测试。
2. `get/post/put/delete` options 是否生效。
3. timeout 是否真的中断 connect/read。
4. redirects 真实 follow/maxRedirects。
5. chunked response decode。
6. gzip/deflate Content-Encoding 成功、失败、zip-bomb cap。
7. duplicate headers / multiple Set-Cookie 返回语义。
8. cookie jar domain/path/secure/httpOnly/sameSite。
9. proxy/no_proxy 行为。
10. connection pool reuse、close、TLS reuse。
11. streaming `readChunk/closeStream` lifecycle 和 slot exhaustion。
12. server headers+body same packet leftover。
13. server request size/header size limit。
14. route closure GC lifecycle。
15. static route semantics。
16. HTTP/2 real client/server interop。
17. TLS certificate verification、SNI、hostname mismatch。
18. IPv6 host formatting、default ports、URL parse edge cases。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | `request` 声明与实现不一致 | 用户按签名调用会失败 | 统一 options-object 或实现 wrapper |
| 高 | server 丢弃 headers 后已读 body leftover | POST body 可能阻塞/损坏 | 保留 leftover body |
| 高 | timeout 计算后未使用 | 网络调用可能长期阻塞 | timeout 贯穿 connect/read/write |
| 高 | HTTP 解压无 output cap | Content-Encoding zip-bomb 风险 | bounded decompress |
| 中 | `get/delete` options 无效 | headers/timeout 不生效 | 实现 options |
| 中 | `post/put` 第三参数语义漂移 | contentType/options 混乱 | 统一 API |
| 中 | LSP HTTP 符号严重不完整 | 补全/hover 误导 | 从 builtin 生成 LSP |
| 中 | `http.static` 声明与实现不一致 | API 语义错误 | 修正声明或实现目录静态服务 |
| 中 | duplicate headers 丢失 | Set-Cookie 等多值不可见 | 增加 headersList |
| 中 | HTTP/2 API 成熟度不清 | 用户误用未完成能力 | 标 experimental 或补实现/测试 |
| 低 | urlDecode lenient 未文档化 | 非法输入语义不清 | strict variant 或文档化 |

## 后续实施建议

建议优先处理会导致用户代码误用或运行错误的问题：

1. 修正 `http.request` / `h2Request` builtin 与实现签名。
2. 修正 `get/post/put/delete` options 语义。
3. 修复 server header/body leftover 读取。
4. 将 timeout 传入连接池、socket read/write、TLS handshake。
5. 给 HTTP Content-Encoding 解压增加 output cap 和错误暴露。
6. 同步 LSP HTTP 符号与 handle 字段。
7. 明确 `http.static` 语义。
8. 为 streaming/download/HTTP2 增加真实 loopback 测试。
9. 将 header 多值语义显式化。
10. 标注 HTTP/2/server singleton/experimental 边界。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
