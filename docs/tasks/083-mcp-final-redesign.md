# 083 - MCP Server 最终重构方案

> 状态说明：本文记录最终设计原则和完成判定。当前实现状态与真实 backlog 以 `015-mcp-improvement.md` 为准。

> 原则：Xray 是全新语言，MCP server 不承担历史兼容成本。所有设计选择以协议正确、结构清晰、安全可控、易扩展为准；发现旧实现不合理时直接删除和重写，不保留兼容层。

## 1. 结论

当前 `src/app/mcp` 已经具备可运行基础：专用 NDJSON transport、JSON-RPC validation、registry 驱动的 tools/resources/prompts、generated knowledge、runner opt-in 和 MCP knowledge 回归测试均已落地。但它仍有协议 transcript、runner 沙盒细节、schema validator 深层结构和搜索排序等后续优化空间。

本方案建议将 MCP server 作为一个独立、严格、可测试的 IDE/AI 集成子系统重写，目标是：

- stdio transport 严格按 MCP 规范实现，不复用 LSP/DAP 的 Content-Length framing。
- JSON-RPC 生命周期和错误语义严格化。
- tools/resources/prompts 全部由统一 registry 驱动，capabilities 自动推导。
- `xray_check` 升级为真正 parser + analyzer 诊断工具。
- `xray_run` 变为隔离 runner，默认有超时、输出限制和 toolset 开关。
- 语言知识库和 stdlib 索引改为生成物，避免手写 C 字符串长期漂移。
- 测试从静态单测升级为协议端到端、工具集成、安全边界三层覆盖。

## 2. 非兼容原则

MCP 重构不提供迁移期，不保留旧行为。

| 旧设计 | 新设计 | 处理方式 |
|---|---|---|
| Content-Length stdio framing | line-delimited JSON / NDJSON stdio transport | 直接删除旧路径 |
| server 全局 `current_progress_token` | per-request context | 直接替换 |
| 手动 `has_tools/has_resources/has_prompts` | registry 自动推导 capabilities | 删除手动 flag |
| tool 参数在 handler 内零散校验 | schema + request validator 统一校验 | handler 只处理已验证参数 |
| `xray_check` 只做 parser 检查却声明 type check | 真正 parser + analyzer 诊断 | 直接修正语义（合并为 `xray_analyze`） |
| `xray_diagnostics` 与 `xray_check` 重复 | 合并为结构化 `xray_analyze` | 删除重复 tool |
| 静态手写 knowledge 拼接字符串 | 结构体数组 + ranked search | 第一步手写但结构化，后续考虑生成脚本 |
| `xray_run` 同进程执行 | 隔离 runner | 删除不安全执行路径 |
| prompt 中旧语法示例 | 当前语法单一真相源 | 旧 prompt 全部改写 |

不提供 legacy aliases，例如不再同时保留 `xray_check` 和 `xray_analyze` 两套名字。没有外部用户时，接口清晰比临时兼容更重要。

## 3. 当前问题清单

### 3.1 协议层

- ~~stdio transport 复用了 Content-Length framing~~ 已完成：NDJSON。
- ~~`initialize` 没有严格处理重复初始化~~ 已完成：三态 lifecycle。
- ~~JSON-RPC request validation 不完整~~ 已完成：`xmcp_jsonrpc_validate_message`。
- ~~`notifications/cancelled` silently ignored~~ 已完成：顺序 dispatch 下明确记录 no-op 语义。
- ~~progress token 存在 `XmcpServer` 全局字段~~ 已完成：改为 per-call `XmcpCallContext`。
- 当前协议版本固定为 `2025-03-26`，仍需决定版本固定、协商或升级策略。

### 3.2 工具层

- ~~`xray_check` / `xray_diagnostics` 重复~~ 已完成：合并为 `xray_analyze`。
- ~~tool result 缺少 `structuredContent` / `outputSchema`~~ 已完成主要 tools，包括 `xray_run`。
- `xray_format` 语法错误路径返回纯 text 错误，该返回 structured diagnostics。
- ~~`xray_run` 会污染 MCP stdout、缺少 timeout/outputSchema~~ 已完成：isolate stdout 捕获、timeout、output limit、structured output 已落地。
- `xray_run` 已有 unit-level 安全边界测试和 transcript 级 stdout 协议隔离测试；仍需更细粒度 allowlist 行为验证。
- `tools/call` 参数错误已通过 schema validator 返回 JSON-RPC `invalid params`；后续可补 object / array 的浅层结构校验。

### 3.3 知识库与 prompts

- ~~knowledge 以拼接 `char[]` 存在 `xmcp_knowledge.c`~~ 已完成：cards + generated C table。
- ~~`xray_stdlib_search` 只有拼接 markdown~~ 已完成：保留 text content，同时返回 structured matches。
- ~~topic 枚举错误信息从硬编码字符串产生~~ 已完成：lookup failure 从 generated knowledge 列出可用 topic。
- ~~prompts 示例未进入 parser smoke test~~ 已完成：MCP knowledge 回归覆盖 prompt smoke examples。
- prompt 正文仍是 C 中手写摘要，后续可从 generated knowledge 派生。

### 3.4 测试层

- ~~缺少真实 stdio 协议端到端测试~~ 部分完成（transport、lifecycle、parse error、unknown method、notification、runner stdout isolation、resources/templates/list、prompts/list、prompts/get、mixed NDJSON、tools/call invalid params / structuredContent）。
- 仍需 resources/read 和 prompts/get 错误路径的 transcript 覆盖。
- ~~缺少 `xray_run` 超时、输出截断、stdlib 白名单测试~~ 已完成 unit coverage。
- ~~缺少 knowledge/prompt 语法示例可解析的回归测试~~ 已完成：topic fences 和 prompt smoke examples 已进入 MCP knowledge 回归。

## 4. 目标架构

### 4.1 模块布局

目标结构：

```text
src/app/mcp/
  xmcp_server.c/.h            # server lifecycle and request loop
  xmcp_transport_stdio.c/.h   # MCP stdio NDJSON transport only
  xmcp_jsonrpc.c/.h           # JSON-RPC validation, response, error helpers
  xmcp_protocol.c/.h          # initialize / capabilities
  xmcp_registry.c/.h          # tools/resources/prompts registry
  xmcp_tools.c/.h             # all tool schemas + handlers + dispatch
  xmcp_resources.c/.h         # resources and templates
  xmcp_prompts.c/.h           # prompt registry and handlers
  xmcp_knowledge.c/.h         # structured language/stdlib metadata + search
```

关于 `xmcp_tools.c` 是否拆分：当前所有 tool 共享 schema builder、structured-result helper、参数校验路径，统一在一个文件里维护内聚度高。文件超过 2000 行（红线 3000）再按 `internal helpers / table / handlers` 维度拆。**不**按 toolset 拆 `_core/_runner/_knowledge`，那会强制重复 helper。

架构约束：

- `src/app/mcp` 属于 app 层，可依赖 base/frontend/module/vm/api 等下层。
- MCP 不直接 include LSP/DAP 内部头，避免 app 子系统互相缠绕。
- 若 MCP 和 LSP 需要共享诊断/definition 能力，应把公共能力下沉到 frontend/service 类模块。
- 所有可复用内部函数使用 `XR_FUNC`；只在单文件内使用的函数必须 `static`。
- 所有动态内存使用 `xr_malloc/xr_free` 系列。

### 4.2 请求处理流水线

```text
stdin line
  -> transport decode
  -> JSON parse
  -> JSON-RPC validate
  -> method dispatch
  -> tool argument validate (centralized, schema-driven)
  -> handler with optional XmcpCallContext (progress_token)
  -> response encode
  -> stdout line
```

核心规则：

- transport 只负责字节流和消息边界，不理解 MCP method。
- JSON-RPC 层只负责 request/notification/response 语义。
- MCP method 层只处理 MCP 协议对象，不再持有任何 per-request 状态。
- tool handler 只接收已验证参数，不重复猜测 JSON 类型。
- 顺序 dispatch 模型，详见 §5.5。

## 5. 协议设计

### 5.1 stdio transport

最终实现只支持 MCP stdio line-delimited JSON：

- read：按 `\n` 切分消息，兼容 `\r\n`。
- write：每条 JSON-RPC message 序列化为单行 JSON，追加 `\n`。
- 拒绝 Content-Length header，不做兼容解析。
- 设置最大输入行长度，例如 16 MiB，超过直接关闭连接或返回 parse/invalid request error。
- stdout 只能写协议消息；日志统一写 stderr 或 log file。

### 5.2 JSON-RPC validation

统一实现 `xmcp_jsonrpc_validate_request`：

- `jsonrpc` 必须为字符串 `"2.0"`。
- `method` 必须为非空字符串。
- request `id` 只允许 string/integer/null 之外按规范明确处理；项目内建议禁止 `null` request id，notification 使用无 `id`。
- `params` 若存在必须为 object。
- batch request 暂不支持，直接返回 invalid request。
- parse error、invalid request、method not found、invalid params、internal error 都由统一 helper 构造。

### 5.3 initialize 生命周期

只支持一个当前稳定 MCP protocol version，不维护多版本兼容矩阵。

- 启动后状态为 `created`。
- 收到合法 `initialize` 后进入 `initialized_pending`，返回 server info 和 capabilities。
- 收到 `notifications/initialized` 后进入 `ready`。
- `ready` 后再次 `initialize` 返回 already initialized error。
- 除 `initialize`、`ping`、必要 notification 外，其他 request 在 `ready` 前全部拒绝。

### 5.4 progress 参数化

progress token 不挂在 `XmcpServer` 上，而是 per-call 局部传递：

```c
typedef struct XmcpCallContext {
    int64_t progress_token;  /* -1 means no progress requested */
} XmcpCallContext;

typedef XrJsonValue *(*XmcpToolHandler)(XmcpServer *server,
                                        const XmcpCallContext *ctx,
                                        XrJsonValue *arguments);
```

规则：

- `tools/call` 在 dispatch 时从 `params._meta.progressToken` 提取，构造栈上 `XmcpCallContext` 传给 handler。
- handler 通过 `xmcp_send_progress_notification(server, ctx->progress_token, ...)` 发送进度。
- `XmcpServer` 不再保存 `current_progress_token` 字段。
- 不引入 `request_id`、`cancelled`、`scratch_arena` 等更复杂状态——这些字段在当前顺序 dispatch 模型下没有真正意义（见 §5.5）。

### 5.5 dispatch 模型与 cancellation

stdio MCP 采用顺序 dispatch：read stdin → parse → validate → handle → write stdout，单线程一条流水线。

含义：

- 同一时刻只有一个 in-flight request。
- `notifications/cancelled` 在当前 dispatch 完成前无法被读取，因此**抢占式取消不可实现**。
- 长任务（runner、analyze 大文件）会阻塞所有后续请求；用 timeout 限制最坏阻塞时间（见 §7.3）。
- `notifications/cancelled` handler 不返回错误，但记录 debug 日志说明 cancel 在顺序模型下是 no-op。

这是**显式的简单性选择**。未来如需支持并发：

- 引入 worker pool / async runtime。
- 把当前 `XmcpCallContext` 升级为完整 `XmcpRequestContext`（含 `request_id`、`cancelled` flag、`scratch_arena`）。
- 主线程 dispatch 仅做 transport + 验证，把 handler 派发给 worker 并维护 in-flight 表，cancel 通过原子 flag 抢占。
- 当前不预留任何这类 hook，避免过早抽象。

## 6. Registry 设计

### 6.1 统一 feature registry

Tools、resources、prompts 使用同一种注册思想：静态定义 + 启动期过滤 + 排序索引。

```text
XmcpRegistry
  tools[]
  resources[]
  resource_templates[]
  prompts[]
```

每类 feature 都有：

- name / uri / uriTemplate
- title
- description
- schema 或 argument definition
- annotations
- toolset id
- handler

capabilities 从 registry 自动生成：

- registry 有 enabled tools -> 声明 tools capability。
- registry 有 enabled prompts -> 声明 prompts capability。
- registry 有 resource 或 template -> 声明 resources capability。
- logging 始终声明。

### 6.2 Toolset

工具按 toolset 分组：

| Toolset | 默认 | 内容 |
|---|---:|---|
| `core` | 是 | `xray_analyze`、`xray_format` |
| `knowledge` | 是 | `xray_syntax_lookup`、`xray_stdlib_search`、`xray_definition` |
| `runner` | 否 | `xray_run` |

`runner` 默认不启用，因为它执行用户代码。

关于 "project" toolset：先前草案里包含的 "project info / resource read" 不属于 tool——`resources/read` 是 MCP 独立 method，`project info` 还没具体设计。删除该分组，避免占位。

CLI：

```text
xray mcp-server                                       # core + knowledge
xray mcp-server --enable-runner                       # + runner
xray mcp-server --enable-runner --run-timeout-ms=3000 # custom timeout (see §7.3)
```

CLI 暂用 `--enable-runner` 单 flag，未来若新增 toolset 再升级为 `--toolsets=...`。

## 7. Tool 设计

### 7.1 `xray_analyze`

替代 `xray_check` 和 `xray_diagnostics`。

输入：

- `code: string`
- `filename?: string`
- `mode?: "syntax" | "semantic" | "full"`

输出：

- text summary
- structured diagnostics
- `ok: bool`
- parser/analyzer version metadata

诊断结构：

```text
line
column
endLine
endColumn
severity
code
message
source
```

实现要求：

- `syntax` 只跑 parser。
- `semantic/full` 必须跑 analyzer。
- MCP 和 CLI `xray check` 共享同一底层诊断服务。
- 错误数量有上限，但返回中必须标注是否 truncated。

### 7.2 `xray_format`

输入：

- `code: string`
- `filename?: string`
- `config?: object`
- `checkOnly?: bool`

输出：

- `formattedCode`
- `changed`
- `diagnostics`
- optional diff summary

实现要求：

- 与 CLI formatter 使用同一配置结构。
- 语法错误返回 structured diagnostics，不返回伪格式化结果。
- 默认配置来自项目统一 formatter defaults。

### 7.3 `xray_run`

只属于 `runner` toolset，默认关闭。

输入：

- `code: string`
- `timeoutMs?: int`（默认 3000，最大 30000）

输出（structured）：

- `ok: bool`
- `exitCode: int`
- `stdout: string`
- `timedOut: bool`
- `truncated: bool`
- `outputBytes: int`

实现要求：

- **同进程 + 一次性 isolate**：每次调用 `xray_isolate_new` 创建新 isolate，调用结束销毁。同进程是显式选择，xray 当前没有 IPC 框架，子进程会和 stdio MCP 抢 stdin/stdout，工程成本远高于收益。
- **stdlib 白名单**：runner isolate **不**调用 `xray_isolate_setup_full`。默认只启用 `print/json/math/string` 等纯计算 stdlib，**不**启用 `net/io/os/process/cluster`。如需放开，必须新增显式 CLI flag（先不实现）。
- **stdout 隔离**：用 isolate 自带 stdout 重定向 API（或临时替换 isolate 内 stdout），**绝不 dup2 全局 stdout**——MCP 协议响应流必须保持纯净。
- **wall-clock timeout**：进入 `xray_isolate_dostring` 前记录 deadline，VM 在循环边界协作式检查（需要 `xr_vm_check_deadline`）。超时后回收 isolate，返回 `timedOut=true`。
- **stderr 不返回**：runner 出错通过 `exitCode != 0` + `stdout` 末尾追加 error 文本表达，简化输出契约。
- **不实现 cancel**：受 §5.5 顺序模型限制，notification/cancelled 抢占不可行。
- **stdout 上限**：默认 8 KiB（保持 MCP 响应紧凑；运行长输出的场景应该用 CLI `xray run`，不是 MCP）。超出 truncated=true。

### 7.4 `xray_syntax_lookup`

输入：

- `topic: string`（如 `class`、`channel`、`coroutine`）

输出（structured）：

- `topic: string`
- `found: bool`
- `content: string`（markdown body）

要求：

- topic 集合从结构化数据表（§10）枚举，错误信息列出真实可用 topic 而非硬编码字符串。

### 7.5 `xray_stdlib_search`

输入：

- `query: string`
- `module?: string`（可选过滤）

输出（structured）：

- `query: string`
- `module: string?`
- `matchCount: int`
- `matches: array<StdlibMatch>`
  - `module: string`
  - `summary: string`
  - `score: number`（用于 ranking）

要求：

- 返回 ranked `matches` 数组而不是拼接 markdown。MCP client 可自行渲染。
- query 太短（< 2 字符）时返回 `matchCount=0` 而不是噪声匹配。
- stdlib 索引最终从 analyzer builtin 元数据生成（§10）；先从结构体数组手写。

### 7.6 `xray_definition`

综合查询：先查语法 topic，再查 stdlib symbol。

输入：

- `symbol: string`

输出（structured）：

- `symbol: string`
- `kind: "syntax" | "stdlib" | "none"`
- `found: bool`
- `content: string`

要求：

- `kind` 显式表达定义来源，便于 client 区分跳转目标。

## 8. Resources 设计

保留 MCP resources，但行为严格化。

资源 URI：

| URI | 内容 |
|---|---|
| `xray://spec/cheatsheet` | 语言速查 |
| `xray://spec/topic/{name}` | 语法主题 |
| `xray://stdlib/modules` | 模块列表 |
| `xray://stdlib/{module}` | 模块详情 |
| `xray://stdlib/{module}/{symbol}` | 符号详情 |

规则：

- unknown URI 返回 JSON-RPC invalid params，不返回空 contents。
- 缺少 `uri` 返回 invalid params。
- URI template 必须严格解析和 decode。
- resource content 使用 `text/markdown; charset=utf-8`。
- resources/list 和 templates/list 按稳定排序返回，支持 cursor。

## 9. Prompts 设计

保留 5 类 prompt，但全部按当前 Xray 语法重写：

- `code-review`
- `explain-error`
- `convert-to-xray`
- `concurrency-pattern`
- `write-test`

要求：

- prompt 中不得出现旧函数返回语法、旧 closure 语法、旧 channel 规则。
- prompt 示例必须进入测试，以 parser smoke test 验证。
- prompt 不复制大段静态知识；需要知识时引用 generated knowledge 的摘要。
- 如果 MCP 当前 message role 不支持 `system`，则把系统指令作为第一条 user message 的明确 instruction，不伪造不被协议接受的 role。

## 10. Knowledge 结构化与生成化

当前实现已经完成生成化：

- 语言语义和权威示例来自 `docs/spec/source/sections/*.md`。
- MCP topic/resource/stdlib 投影来自 `docs/spec/source/cards/**/*.json`。
- `scripts/gen_language_docs.py` 生成 `LANGUAGE_SPEC_CN.md`、`LANGUAGE_SPEC.md` 和 `docs/knowledge/**`。
- `scripts/gen_mcp_knowledge.py` 结合 `docs/knowledge/**` 与 analyzer builtin dump 生成 `src/app/mcp/xmcp_knowledge_generated.c`。
- stdlib symbol 签名来自 analyzer builtin metadata，不在 cards 中手写复制。
- `xmcp_knowledge.c` 只保留加载、lookup 和 search/ranking 逻辑。

维护要求：

- 新增 Xray 语法或语义时，先更新 `sections/*.md` 中英文正文，再更新相关 card。
- 需要 MCP 复用的示例使用 `xray @id=...` fence，并由 card `fences` 引用。
- 新增 `@id` fence 后必须有至少一个 card 引用。
- 新增 topic card 后必须同步 tool/resource metadata 列表。
- MCP knowledge 回归必须保持生成物最新、示例可检查、stdlib symbols 与 analyzer builtin dump 一致。

## 11. 错误处理策略

严格分两层：

- **JSON-RPC error** (`error.code` + `error.message`)：协议层错误。包括 parse error、invalid request、method not found、invalid params（含 tool 名缺失 / 未知 tool / arguments 不是 object / required 参数缺失 / 参数类型错 / unknown resource URI）、internal error。
- **Tool result `isError=true`**：handler 执行成功但业务语义上失败。包括：Xray 代码语法/语义/运行时错误、runner timeout、runner exit != 0。

具体场景表：

| 场景 | 返回 | error code |
|---|---|---|
| malformed JSON | JSON-RPC error | `-32700` parse |
| `jsonrpc != "2.0"` | JSON-RPC error | `-32600` invalid request |
| `tools/call` 缺少 `name` | JSON-RPC error | `-32602` invalid params |
| unknown tool name | JSON-RPC error | `-32602` invalid params |
| `arguments` 不是 object | JSON-RPC error | `-32602` invalid params |
| required 参数缺失 | JSON-RPC error | `-32602` invalid params |
| 参数类型不匹配 schema | JSON-RPC error | `-32602` invalid params |
| `resources/read` 缺 `uri` | JSON-RPC error | `-32602` invalid params |
| `resources/read` unknown URI | JSON-RPC error | `-32602` invalid params |
| ready 前调用非 initialize/ping | JSON-RPC error | `-32002` not initialized |
| 重复 initialize | JSON-RPC error | `-32003` already initialized |
| `xray_analyze` 收到语法错 | tool result | `isError=true` + diagnostics |
| `xray_run` 超时 | tool result | `isError=true` + `timedOut=true` |
| `xray_run` exit != 0 | tool result | `isError=true` + `exitCode` |
| server 内部分配失败 | JSON-RPC error | `-32603` internal error |

**关键区分**：客户端可通过响应是否含 `error` 字段区分“调用 MCP 错了”（需要修复调用代码）与“Xray 程序有问题”（需要修复被调代码）。

**实现说明**：

- `xmcp_validate_tool_arguments` 返回错误代码 + 人读信息，dispatch 层负责包装为 JSON-RPC error 响应。handler 只产生 tool result（成功或 isError）。
- `tools/list`、`resources/list` 等查询接口不产生 tool result，出错一律走 JSON-RPC error。

## 12. 测试方案

### 12.1 协议端到端测试

新增测试覆盖：

- initialize -> initialized -> tools/list。
- ready 前调用 tools/list 被拒绝。
- 重复 initialize 返回错误。
- unknown method 返回 method not found。
- malformed JSON 返回 parse error。
- notification 不产生 response。
- NDJSON 多消息连续输入。
- 拒绝 Content-Length 输入。

### 12.2 Tool 集成测试

覆盖：

- `xray_analyze` 正确代码、语法错误、语义错误。
- `xray_format` 正常格式化、语法错误返回 structured diagnostics。
- `xray_syntax_lookup` exact match / alias / unknown topic。
- `xray_stdlib_search` ranked matches / module 过滤 / too-short query。
- `xray_definition` syntax kind / stdlib kind / none。
- `xray_run` 正常退出、非零退出、timeout、输出截断；stdlib 白名单生效（调用 `net`/`os` 被拒绝）。

### 12.3 Knowledge 和 prompt 回归测试

覆盖：

- 所有 prompt 示例片段可解析。
- knowledge 中关键语法存在：`_ ->`、`after ms ->`、`#{ key: value }`、`fn(...) -> T`、`shared const` channel。
- stdlib symbol signature 与 generated builtin/analyzer 数据一致。

### 12.4 安全边界测试

覆盖：

- runner 未启用时 `xray_run` 不在 tools/list。
- runner timeout 后 server 仍可响应后续 request。
- runner stdlib 白名单：调用被禁模块返回语义错误，不会造成隐藏副作用。
- runner stdout 被 isolate 捕获，不污染 MCP 响应流。
- 超大输入被拒绝且不导致内存失控。

不覆盖（顺序模型下不适用）：

- 抢占式 cancel——§5.5 明确声明不实现。

## 13. 实施批次

### 批次一：协议骨架重写

目标：先让 MCP transport 和 JSON-RPC 生命周期正确。

交付：

- 新 stdio NDJSON transport。
- 新 JSON-RPC validator/error helper。
- 新 initialize state machine。
- 删除 Content-Length MCP 路径。
- 端到端协议测试通过。

### 批次二：Registry 和 capability 重写

目标：消灭手动 capabilities 和零散 dispatch。

交付：

- `XmcpRegistry`。
- tools/resources/prompts 统一注册。
- capabilities 自动推导。
- cursor list 稳定排序。
- 旧 tool 表直接替换。

### 批次三：Core tools 正确化

目标：让 AI 最常用能力可信。

交付：

- `xray_analyze` 合并 check/diagnostics。
- analyzer 接入。
- `xray_format` 复用 CLI formatter config。
- structured output + outputSchema。

### 批次四：Knowledge 和 prompts 生成化

目标：消灭手写知识库漂移。

交付：

- generated knowledge 数据表。
- stdlib API 级索引。
- prompts 全量改写。
- 示例语法 smoke test。

### 批次五：Runner 安全化

目标：保留运行能力，但不牺牲 MCP server 稳定性。

交付：

- runner toolset 默认关闭（已完成）。
- isolate 内部 stdout 捕获，不再 dup2 全局 stdout。
- stdlib 白名单，默认关闭 `net/io/os/process/cluster`。
- wall-clock timeout + `xr_vm_check_deadline` 协作检查。
- structured output (`ok / exitCode / stdout / timedOut / truncated / outputBytes`) + outputSchema。
- CLI `--run-timeout-ms`。
- runner 安全测试。

## 14. 完成判定

MCP 重构完成必须同时满足：

- `xray mcp-server` 可被主流 MCP 客户端通过 stdio 正常初始化和调用。
- 不再存在 MCP Content-Length transport。
- 初始化、错误、notification、progress 行为有端到端测试（cancel 不覆盖，§5.5）。
- tools/list 只由 registry 产生，capabilities 只由 registry 推导。
- `xray_analyze` 能返回 parser + analyzer structured diagnostics。
- `xray_run` 受 timeout 保护不会无限阻塞 server；stdlib 限于白名单；stdout 不污染 MCP 响应。
- knowledge 使用 cards + generated C table，`xray_stdlib_search` 返回 ranked structured matches。
- prompt 示例全部符合当前 Xray 语法。
- 完整 `ctest --output-on-failure` 通过。

进阶目标（后续）：

- protocol transcript conformance 测试覆盖真实 stdio 会话。
- runner stdlib 重启用有显式 CLI flag 控制。
- worker 模型补齐 cancel 与并发。

## 15. 后续维护原则

- 新增 MCP feature 必须先注册 schema 和 tests，再接 handler。
- 新增 Xray 语法时，必须同步 knowledge 生成源和 prompt 示例测试。
- 新增 stdlib builtin 时，必须进入 stdlib symbol index。
- 任何不符合最终协议的旧入口直接删除，不添加兼容分支。
- MCP server 是 AI 客户端的可信语言接口，宁可少暴露 tool，也不要暴露语义不准或安全不可控的 tool。
