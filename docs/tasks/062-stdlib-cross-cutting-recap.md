# stdlib 横切复盘与实施优先级

## 范围

本复盘基于 `037` 到 `061` 的标准库分析文档，覆盖：

- 横切基础层：`common`、`common_io`、`common_parser`、`common_writer`、`ctxbuf`、`stdlib_cache`
- core：`time`、`math`、`gc`、`path`、`base64`、`encoding`、`url`、`datetime`、`log`、`json`、`regex`
- data：`csv`、`toml`、`yaml`、`xml`
- system：`io`、`os`
- crypto/compress：`crypto`、`compress`
- network：`net`、`http`、`ws`、`cluster`
- test/support：`test_yield`

`036` 中列出的 24 个 native stdlib 模块已经全部完成独立分析。`062` 的目标不是重复每个模块的问题，而是把重复出现的风险合并为可执行的重构主线。

## 总体判断

标准库当前的核心问题不是单个模块功能缺失，而是横切契约没有统一：

- runtime export、`XR_DEFINE_BUILTIN`、analyzer generated metadata、LSP symbols 经常不一致。
- CMake 编译条件、loader declaration、module registration 存在分叉。
- blocking、slow、yieldable 的标记不成体系。
- 错误返回模型混合使用 `null`、`false`、空字符串、空 Map、runtime error、errno code、Json error。
- 数据格式模块在 `Json`、`Map`、runtime object 之间没有统一对象表示。
- 多个模块仍有全局状态、进程级副作用或 isolate 生命周期边界不清晰。
- 资源上限、安全默认值、测试覆盖和工具链声明经常后置。

因此后续不应按“一个库一个库随手修”推进，而应先收敛公共契约，再批量落地到模块。

## 实施优先级总表

| 优先级 | 主线 | 目标 | 典型受影响模块 |
|---:|---|---|---|
| 1 | 构建与注册一致性 | CMake、loader、registration 同源，不暴露测试模块 | `test_yield`、`cluster`、`net`、`http`、`ws` |
| 2 | API 类型真相源 | runtime/analyzer/LSP/文档一致 | 全部模块，重点 `regex`、`net`、`http`、data formats |
| 3 | 错误模型统一 | 参数错误、I/O 错误、parse 错误、OOM 可诊断 | `io`、`os`、`csv`、`toml`、`yaml`、`xml`、`compress` |
| 4 | blocking/yieldable 审计 | 不让同步慢操作卡住 worker | `io`、`os`、data file API、`net`、`http`、`ws` |
| 5 | 资源上限与安全默认值 | 不可信输入不导致无界内存、注入或进程副作用 | `compress`、`xml`、`yaml`、`regex`、`http`、`ws`、`os` |
| 6 | 状态与生命周期 | per-isolate ownership、handle 生命周期、GC root 清晰 | `log`、`net`、`http`、`ws`、`cluster`、`test_yield` |
| 7 | 测试矩阵 | 用回归/单测锁住横切契约 | 全部模块 |

## 主线 1：构建与模块注册一致性

### 现状

标准库模块的可用性由多处共同决定：

- CMake 是否把 `.c` 编译进 `xray_core`。
- `xmodule_loaders.h` 是否声明 loader。
- `xmodule.c` 是否注册到 stdlib table。
- `XR_DEFINE_BUILTIN` 是否生成 analyzer metadata。
- LSP 是否有 symbols。

这些来源目前不是单一真相源。

### 典型问题

- `test_yield` 被注册在 filesystem group，但 modular filesystem build 只编译 `io/os`，可能出现 loader undefined symbol。
- TLS disabled 时 runtime 不导出部分 TLS API，但 analyzer 仍可能声明它们。
- `cluster` 属于 optional 能力，但体量、依赖和安全边界接近独立子系统。
- full stdlib 会把测试支撑模块一起编译和暴露。

### 建议

1. 建立一个 stdlib module manifest，记录模块名、源码目录、group、feature macro、loader、公开状态、测试状态。
2. CMake source inclusion、loader declaration、registration、metadata generation 都从 manifest 派生。
3. 把 `test_yield` 移出正式 stdlib group，只在测试构建中注册。
4. optional feature 下的 analyzer/LSP metadata 必须可按 feature 裁剪。
5. 给 modular build 增加 CI smoke：至少覆盖 core-only、filesystem、network、full、no-TLS。

### 验收

- 任意 feature combination 下无 undefined loader symbol。
- runtime 可 import 的模块一定有 analyzer/LSP metadata，除非标记为 internal test module。
- release/full stdlib 不暴露 `test_yield`。
- CMake 与 `xmodule.c` 不再手写重复模块列表。

## 主线 2：API 类型真相源

### 现状

多个模块存在 runtime、builtin declaration、analyzer、LSP 的签名漂移。

典型表现：

- `regex.find()` runtime 返回 match object，但 analyzer/LSP 描述为 `string?` 或其他形状。
- data format 模块声明 `Json?`，实际返回 `Map` 或混合对象。
- `compress` 脚本层使用 binary-safe string，但 LSP 中出现 `Bytes` 概念。
- `net/http/ws` 的 handle field、options 参数、TLS conditional API 与工具链不一致。
- 很多 loader 头和 `XR_DEFINE_BUILTIN` 使用 `any` 或过窄类型来绕过类型系统。

### 建议

1. 定义标准库类型描述语言的最小集合：module function、native handle、record shape、optional feature、method、getter、slow/yieldable 属性。
2. `XR_DEFINE_BUILTIN` 只保留为过渡入口，最终由 manifest/descriptor 生成 C declaration、analyzer、LSP、文档片段。
3. 建立以下标准类型或 shape：
   - `Regex`
   - `RegexMatch`
   - `Bytes` 或明确 binary-safe string 策略
   - `PathLike` 或 path 参数约定
   - `NetConn`、`NetListener`、`UdpSocket`、`HttpResponse`、`WsConnection`
   - `ParseResult` / `Diagnostic`
4. data formats 统一对象表示：要么全部返回 `Json`，要么全部返回 `Map<string, any>`，不能每个模块各自漂移。
5. 加一个 generated metadata consistency test：runtime export 表、analyzer generated、LSP symbols 互相校验。

### 验收

- 每个公开模块函数的 runtime export 都能在 generated metadata 中找到。
- 每个 LSP symbol 都能追溯到 runtime export 或类型 method。
- `regex`、data formats、network handles 的返回 shape 在文档、analyzer、LSP、测试中一致。

## 主线 3：错误模型统一

### 现状

当前错误语义高度分散：

- 参数错误有时返回 `null`、`false`、空字符串、空数组，有时 runtime error。
- I/O 错误经常丢失 errno，只返回 `false/null`。
- parser 错误有的返回空 Map，有的返回 `{data, errors}`，有的静默容错。
- compression、crypto、network 有内部错误码，但脚本层常不可观测。
- OOM 路径有时返回旧值或空值，用户无法区分。

### 建议

制定统一规则：

1. **参数 arity/type 错误**：默认 runtime error，不静默返回默认值。
2. **资源/I/O 错误**：返回 detailed result 或抛标准 error，必须包含 code/message。
3. **parse/validation 错误**：区分 `parse()`、`parseStrict()`、`parseDetailed()` 三类语义。
4. **OOM**：统一走 runtime OOM/error，不返回伪成功值。
5. **可预期失败**：例如 `isValid()`、`exists()` 可返回 bool，但不得吞掉系统性错误。

推荐最小公共 shape：

```text
{
  ok: bool,
  value: any?,
  error: {
    code: string,
    message: string,
    line?: int,
    column?: int,
    offset?: int,
    cause?: string
  }?
}
```

如果不想引入通用 `Result` 类型，至少每类模块要内部统一。

### 验收

- data formats 的 `parse/parseStrict/parseDetailed` 命名和返回一致。
- file API 不再用空 Map 或 `false` 掩盖 I/O 错误。
- `compress` 能区分 header/checksum/data/size limit 错误。
- `net/http/ws` 暴露可诊断错误码，而不是只有 `null/-1`。

## 主线 4：blocking、slow、yieldable 审计

### 现状

标准库中存在三类 native 调用：

- 普通快速函数。
- 同步慢函数，需要标记 `SLOW`。
- 可协作挂起的 yieldable 函数。

当前标记不够系统：

- `time.sleep()` 已正确 yieldable。
- `net`、`ws` 部分 I/O 已 yieldable。
- `http` client 使用 `SLOW`，但 timeout/deadline 传递不完整。
- data formats 的 `parseFile/writeFile` 同步全量 I/O 仍可能阻塞 worker。
- `os.exec` 是高风险同步命令执行，且有 stdout/stderr 管道和输出上限问题。
- `io`/`os` 中部分系统调用会阻塞或产生进程级副作用。

### 建议

1. 给所有 stdlib export 增加 execution class：`fast`、`slow`、`yieldable`、`process_side_effect`、`blocking_file_io`。
2. 静态检查所有 `XRS_EXPORT` 调用：凡是可能同步等待 I/O、进程、DNS、锁、线程 join 的 API 必须显式标注。
3. 文件 API 统一规划 yieldable/streaming 版本：`readFileAsync` 不一定需要新 API，但 native 层应能不阻塞 worker。
4. `os.exec` 单独制定策略：超时、输出上限、stderr/stdout 并发读取、shell 注入警告、参数数组 API。
5. network read/write/accept/recv 默认带 timeout 或可配置 deadline。

### 验收

- 有一份自动生成的 stdlib export execution class 表。
- 阻塞 API 均为 `SLOW` 或 yieldable。
- `os.exec` 不会因 pipe buffer 填满导致死锁。
- file read/write 大文件路径不会无限占用 worker。

## 主线 5：资源上限与安全默认值

### 现状

多个模块面对不可信输入，但资源上限不统一：

- `compress` 解压缺少脚本层 `maxOutputBytes`，HTTP decompression 有 zip bomb 风险。
- `xml` 需要明确 entity/DTD/XXE 默认禁用策略。
- `yaml` 需要 alias/depth/document boundary/duplicate key 边界。
- `regex` 有引擎上限，但 `findAll/split` 存在隐藏 cap，需要显式语义。
- `http/ws` body、header、frame、message、deflate output 需要统一上限。
- `os.exec` 需要命令执行、环境、工作目录、输出上限和超时安全边界。
- `path/io` 需要 path traversal、symlink、absolute/relative 策略。

### 建议

建立默认安全策略表：

| 类别 | 默认策略 |
|---|---|
| 最大输入 | parser/file/network API 有默认上限和可配置 options |
| 最大输出 | compress/replace/stringify/HTTP body 有 cap |
| 最大深度 | JSON/YAML/TOML/XML/regex AST 有 depth limit |
| 外部实体 | XML 默认禁止外部实体和 DTD 网络访问 |
| 命令执行 | 默认推荐 args-array API，不鼓励 shell string |
| TLS | 默认验证证书和 hostname，可显式关闭 |
| 随机数 | crypto random 与 math random 明确区分 |
| 路径 | 提供 safeJoin/normalize/insideBase 语义 |

### 验收

- 每个 parser 模块有 depth/input/output 上限测试。
- HTTP/WS/compress 有 zip bomb 和 oversized body/frame 测试。
- TLS 默认验证策略有正反测试。
- `os.exec` 有 timeout/output cap/stderr flooding 测试。

## 主线 6：状态、生命周期与 isolate 边界

### 现状

若干模块仍存在进程级状态或生命周期不清晰：

- `log` 虽已放入 stdlib cache，但 output sink、async queue、FILE* 生命周期仍有并发风险。
- `net` 有 DNS cache、TLS fd registry、shape cache 等跨 isolate 风险。
- `http/ws` server、connection、route closure、stream slot、deflate state 的 owner 和 GC root 需要更清晰。
- `cluster` 涉及 distributed channel、discovery、health、serialization，边界接近独立 runtime subsystem。
- `test_yield` 有进程级 atomic counter，不应作为正式 stdlib 暴露。
- `os.chdir/setenv/unsetenv/exit/kill` 本质是进程级副作用，需要文档和测试隔离。

### 建议

1. 所有模块 state 优先挂到 `XrStdlibCache` 或模块 context。
2. 所有 native handle 带 owner/isolate token 或改为 opaque native object。
3. GC root 和 native resource cleanup 要有统一模式：module unload/isolate destroy/coroutine cancel/handle close。
4. 进程级副作用 API 标记为 `process_side_effect`，测试隔离运行。
5. 后台线程、async queue、server listener 必须在 isolate destroy 时可停止且可验证。

### 验收

- 多 isolate 同进程测试不互相污染 log/net/test state。
- fd reuse 后不会继承 stale TLS/netpoll state。
- HTTP route closure 和 WS connection 在 GC 压力下不悬垂。
- `os` 进程级 API 的测试可恢复环境。

## 主线 7：数据格式模块统一

### 现状

`json/csv/toml/yaml/xml` 重复出现类似问题：

- 返回对象表示不统一。
- strict/detailed 命名不一致。
- parseFile/writeFile 同步全量 I/O。
- serializer 对 Map/Json/Instance 的遍历方式不同。
- duplicate key、null、datetime、Unicode escape、cycle、ordering 策略不统一。

### 建议

1. 统一 parser API 命名：
   - `parse(input, options?)`
   - `parseDetailed(input, options?)`
   - `stringify(value, options?)`
   - `readFile(path, options?)`
   - `writeFile(path, value, options?)`
2. strict 语义只保留一种：errors 非空时失败，或改名 detailed。
3. 统一返回对象类型。
4. 提供 runtime object iterator API，serializer 不直接依赖 Map/Json internal。
5. 文件 API 共享 common I/O 错误和 blocking/yieldable 策略。

### 验收

- data format 模块的 analyzer/LSP 签名风格一致。
- duplicate key、invalid unicode、deep nesting、cycle、stable output 都有测试。
- `parseFile/writeFile` 的错误可观测。

## 主线 8：网络栈边界

### 现状

`net/http/ws/cluster` 共享很多能力，但边界尚未完全清晰：

- `net` 既有 TCP/TLS/DNS，也有 HTTP connection pool 倾向。
- `http` 复用 net/compress，但 timeout、TLS、decompression 上限未完整贯通。
- `ws` 复用 HTTP handshake、net I/O、compress deflate，但 frame/message/backpressure 边界需要更强测试。
- `cluster` 架在 net/serialize/channel/discovery 上，体量和故障模式接近独立系统。

### 建议

1. `net` 只保留 transport：TCP/UDP/TLS/DNS/native handle。
2. HTTP connection pool 归属 HTTP，除非抽象为通用 transport pool。
3. HTTP/WS/cluster 共享的 timeout、TLS options、error、bounded decompression、handle owner 走公共结构。
4. cluster 默认不作为核心 stdlib 能力，保留 optional build 和明确安全边界。
5. 给 network stack 建 loopback integration tests，而不仅是 API existence tests。

### 验收

- TLS disabled/enabled 构建下 runtime/analyzer/LSP 一致。
- HTTP/WS 真实 loopback 覆盖 connect、TLS、compression、timeout、close。
- cluster discovery/health/serialization 有隔离测试和故障注入。

## 主线 9：loader 可见性与 include 边界

### 现状

多个 stdlib 模块的 loader 定义没有统一 `XR_FUNC`，部分 header 只需要声明 loader 却 include VM 或 runtime internal。`XRS_EXPORT_YIELDABLE` 当前也推动 stdlib include VM header。

### 建议

1. 所有 `xr_load_module_*` 定义与声明统一 `XR_FUNC`。
2. 模块私有 `.h` 若只声明 loader，应只 forward declare `XrayIsolate` 和 `XrModule`。
3. 深层 `../../src/vm/...` include 要被清理或封装。
4. `XRS_EXPORT_YIELDABLE` 所需 constructor 通过 module 层或稳定 bridge 暴露。
5. 增加 include DAG 检查和 stdlib header self-contained compile test。

### 验收

- 静态检查能列出所有未加 `XR_FUNC` 的非 static stdlib 函数。
- stdlib module header 不 include VM internal。
- `scripts/check_architecture.sh` 对 stdlib include 边界无新增警告。

## 建议的落地顺序

### 第一组：先修构建和测试暴露面

1. 移除或内化 `test_yield` 的正式 stdlib 注册。
2. 修正 modular build 中 CMake/source/loader/registration 不一致。
3. 为 stdlib module manifest 打基础。
4. 增加 feature combination build smoke。

原因：这些问题会直接影响能否构建和发布。

### 第二组：统一工具链类型

1. 设计 module/type descriptor。
2. 先覆盖 `regex`、`net`、`http`、data formats。
3. 生成 analyzer/LSP/runtime export consistency test。
4. 修正最明显的签名漂移。

原因：类型真相源统一后，后续改 API 不会继续产生漂移。

### 第三组：统一错误和 blocking 规则

1. 制定参数错误、I/O 错误、parse 错误、OOM 的统一规则。
2. 生成 execution class 表。
3. 审计所有 `XRS_EXPORT` 是否应为 `SLOW` 或 yieldable。
4. 优先处理 `os.exec`、file API、network timeout。

原因：这是标准库可靠性的核心用户体验。

### 第四组：资源上限和安全默认值

1. compress/http/ws 解压和 body/frame 上限。
2. XML/YAML/TOML/JSON parser depth/input 上限。
3. TLS 默认验证策略。
4. path/os 安全边界。

原因：这是服务端场景的安全底线。

### 第五组：生命周期和对象模型

1. native handle owner/isolate token。
2. per-isolate state context。
3. async queue/server/listener cleanup。
4. data serializer object iterator。

原因：这是大规模重构，适合在契约稳定后推进。

## 横切测试矩阵

建议新增或扩展以下测试类型：

| 测试类型 | 目标 |
|---|---|
| module manifest consistency | CMake、loader、registration、metadata 一致 |
| analyzer/LSP/runtime export diff | 工具链不漂移 |
| execution class audit | 阻塞 API 必须标注 |
| feature combination build | core/full/filesystem/network/no-TLS 均可构建 |
| parser resource limits | depth/input/output cap 有效 |
| zip bomb / oversized body | compress/http/ws 安全上限 |
| multi-isolate state isolation | log/net/test state 不串扰 |
| handle lifecycle/fd reuse | close 后 fd reuse 不污染 |
| GC pressure native resources | route/ws/regex/log/native handle 不悬垂 |
| error shape snapshot | 典型错误返回稳定可诊断 |

## 不建议继续做的事

- 不建议继续为每个模块单独手写 LSP symbols。
- 不建议让 analyzer generated 与 runtime export 分开维护。
- 不建议保留 `parseStrict` 这种名称但返回部分成功结果的语义。
- 不建议把测试支撑模块混入 release stdlib。
- 不建议在用户标准库 API 中继续引入进程级全局状态。
- 不建议新增 blocking file/network/process API 而不标注 execution class。

## 完成标准

标准库重构的第一轮完成标准建议定义为：

1. 所有公开模块来自统一 manifest。
2. runtime/analyzer/LSP 的公开 API 集合一致。
3. 每个公开 export 有稳定签名、错误策略和 execution class。
4. 所有 parser/decoder/network API 有默认资源上限。
5. 测试模块不进入 release stdlib 命名空间。
6. 所有 native handle 和 module state 有明确 owner 和 cleanup。
7. `ctest`、完整回归、feature combination build、architecture/comment checks 全部通过。

## 结论

标准库独立模块分析已经完成，下一步应从“逐库观察”切换到“横切契约收敛”。最先处理的不是业务功能，而是构建注册一致性、API 类型真相源、错误模型和 blocking/yieldable 分类。只有这些公共规则稳定后，逐库修复才不会继续制造 analyzer/LSP 漂移、隐藏阻塞和资源上限缺口。
