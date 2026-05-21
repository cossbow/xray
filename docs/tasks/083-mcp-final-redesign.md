# 083 - MCP Server 最终重构方案

> 原则：Xray 是全新语言，MCP server 不承担历史兼容成本。所有设计选择以协议正确、结构清晰、安全可控、易扩展为准；发现旧实现不合理时直接删除和重写，不保留兼容层。

## 1. 结论

当前 `src/app/mcp` 已经具备可运行雏形：CLI 接入、JSON-RPC 分发、tools/resources/prompts/knowledge 基本齐全，单测也覆盖了主要静态接口。但它仍是“能用的原型”，不是最终设计。

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
| `xray_check` 只做 parser 检查却声明 type check | 真正 parser + analyzer 诊断 | 直接修正语义 |
| `xray_diagnostics` 与 `xray_check` 重复 | 合并为结构化 `xray_analyze` | 删除重复 tool |
| 静态手写 knowledge 大字符串 | spec/stdlib 元数据生成 | 生成文件替代手写表 |
| `xray_run` 同进程执行 | 隔离 runner | 删除不安全执行路径 |
| prompt 中旧语法示例 | 当前语法单一真相源 | 旧 prompt 全部改写 |

不提供 legacy aliases，例如不再同时保留 `xray_check` 和 `xray_analyze` 两套名字。没有外部用户时，接口清晰比临时兼容更重要。

## 3. 当前问题清单

### 3.1 协议层

- stdio transport 复用了 Content-Length framing，更像 LSP/DAP transport，不应作为 MCP 最终实现。
- `initialize` 没有严格处理重复初始化、协议版本、client capabilities。
- JSON-RPC request validation 不完整：缺少 `jsonrpc == "2.0"`、`id` 类型、`params` 类型、batch policy 等校验。
- `notifications/cancelled` 当前没有真正取消正在执行的 tool。
- progress token 存在 server 全局状态里，不适合未来并发或嵌套调用。

### 3.2 工具层

- `xray_check` 描述包含 type check，但实现主要是 parser diagnostics。
- `xray_diagnostics` 与 `xray_check` 功能重复，只是输出 Markdown table。
- `xray_format` 与 CLI formatter 配置不完全一致。
- `xray_run` 同进程执行用户代码，没有超时、隔离、模块权限、stderr 捕获和取消支持。
- tool result 以文本为主，缺少稳定的 `structuredContent` 和 `outputSchema`。

### 3.3 知识库与 prompts

- knowledge 是手写 C 字符串，语言规范和 stdlib 改动后容易漂移。
- stdlib 只有模块级描述，缺少函数级签名、参数、返回值、示例和安全说明。
- prompts 中仍可能出现旧语法示例，会误导 AI 生成过时代码。

### 3.4 测试层

- 现有单测覆盖 registry/schema/静态返回较多，但缺少真实 stdio 协议端到端测试。
- 缺少 `xray_run` 超时、取消、输出截断、安全限制测试。
- 缺少 knowledge/prompt 语法示例可解析的回归测试。

## 4. 目标架构

### 4.1 模块布局

建议重排为以下结构：

```text
src/app/mcp/
  xmcp_main.c                 # CLI entry and option parsing
  xmcp_server.c/.h            # server lifecycle and request loop
  xmcp_transport_stdio.c/.h   # MCP stdio NDJSON transport only
  xmcp_jsonrpc.c/.h           # JSON-RPC validation, response, error helpers
  xmcp_context.c/.h           # per-request context, cancellation, progress
  xmcp_registry.c/.h          # tools/resources/prompts registry
  xmcp_schema.c/.h            # minimal JSON schema builders and validators
  xmcp_tools_core.c           # analyze, format
  xmcp_tools_runner.c         # run toolset, isolated execution
  xmcp_tools_knowledge.c      # doc lookup, stdlib lookup
  xmcp_resources.c/.h         # resources and templates
  xmcp_prompts.c/.h           # prompt registry and handlers
  xmcp_knowledge_generated.c/.h
```

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
  -> XmcpRequestContext create
  -> method dispatch
  -> handler result/error
  -> response encode
  -> stdout line
```

核心规则：

- transport 只负责字节流和消息边界，不理解 MCP method。
- JSON-RPC 层只负责 request/notification/response 语义。
- MCP method 层只处理 MCP 协议对象。
- tool handler 只接收已验证参数，不再重复猜测 JSON 类型。
- cancellation notification 走抢占路径，不排队等待慢 tool 完成。

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

### 5.4 cancellation 与 progress

新增 `XmcpRequestContext`：

```text
request_id
method
progress_token
cancelled
server
scratch_arena
```

规则：

- 每个 request 独立保存 progress token。
- `notifications/cancelled` 按 request id 标记 context cancelled。
- 长任务必须在阶段边界检查 cancellation。
- progress notification 只从 request context 发出，不允许读写 server 全局 progress 字段。

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
| `core` | 是 | analyze、format |
| `knowledge` | 是 | doc lookup、stdlib lookup |
| `project` | 是 | project info、resource read |
| `runner` | 否 | run |

`runner` 默认不启用，因为它执行用户代码。用户需要通过 CLI 显式启用。

CLI 建议：

```text
xray mcp-server --toolsets=core,knowledge,project
xray mcp-server --toolsets=core,knowledge,project,runner --run-timeout-ms=3000
```

不保留旧开关语义；命令行参数以最终设计为准。

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

只属于 `runner` toolset。

输入：

- `code: string`
- `args?: string[]`
- `timeoutMs?: int`
- `stdin?: string`

输出：

- `exitCode`
- `stdout`
- `stderr`
- `timedOut`
- `cancelled`
- `truncated`

实现要求：

- 不再同进程直接执行用户代码。
- 使用隔离子进程或专用 worker。
- 默认 timeout，例如 3000ms。
- stdout/stderr 都有上限，例如各 64 KiB。
- cancellation 必须能终止 runner。
- 网络、文件系统、危险 stdlib 模块的策略必须显式，不靠默认开放。

### 7.4 `xray_doc_lookup`

替代旧的 syntax lookup 和 definition 文档查询。

输入：

- `query: string`
- `kind?: "syntax" | "stdlib" | "symbol" | "all"`

输出：

- ranked matches
- title
- kind
- summary
- markdown body
- examples

要求：

- 不只返回第一个模糊匹配。
- query 太短时返回候选提示，不做噪声匹配。
- 所有示例必须通过语法 smoke test。

### 7.5 `xray_stdlib_lookup`

输入：

- `module?: string`
- `symbol?: string`
- `query?: string`

输出：

- module description
- symbol signature
- params
- return type
- examples
- safety notes

要求：

- stdlib 索引从真实 builtin/module 元数据生成。
- 函数签名必须和 analyzer builtin 表一致。

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

## 10. Knowledge 生成化

最终状态：`xmcp_knowledge_generated.c/.h` 是生成物，手写源数据不在 C 文件里维护。

生成来源：

- language topic metadata
- stdlib module metadata
- builtin signature metadata
- examples metadata

生成内容：

```text
topic id
title
aliases
summary
markdown body
examples
stdlib module
stdlib symbol
signature
params
return type
```

生成规则：

- C 文件只包含数据表，不写复杂逻辑。
- 搜索和 ranking 逻辑在手写 `xmcp_knowledge_search.c` 中。
- 生成脚本必须有测试，确保关键语法主题存在。
- stdlib symbol 签名必须与 analyzer builtin 输出一致。

## 11. 错误处理策略

统一分两类错误：

- JSON-RPC error：协议错误、method 错误、参数 schema 错误、资源不存在。
- Tool result `isError=true`：工具执行成功但 Xray 代码本身有语法/语义/运行时错误。

例子：

| 场景 | 返回 |
|---|---|
| `tools/call` 缺少 `name` | JSON-RPC invalid params |
| unknown tool | JSON-RPC invalid params |
| `xray_analyze` 收到语法错误代码 | tool result，`isError=true`，含 diagnostics |
| unknown resource URI | JSON-RPC invalid params |
| runner timeout | tool result，`isError=true`，`timedOut=true` |
| server 内部分配失败 | JSON-RPC internal error |

这样客户端能区分“调用 MCP 错了”和“Xray 程序有问题”。

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
- `xray_format` 正常格式化、checkOnly、语法错误。
- `xray_doc_lookup` exact/alias/fuzzy/too-short query。
- `xray_stdlib_lookup` module/symbol/signature。
- `xray_run` 正常退出、非零退出、timeout、cancel、输出截断。

### 12.3 Knowledge 和 prompt 回归测试

覆盖：

- 所有 prompt 示例片段可解析。
- knowledge 中关键语法存在：`_ ->`、`after ms ->`、`#{ key: value }`、`fn(...) -> T`、`shared const` channel。
- stdlib symbol signature 与 generated builtin/analyzer 数据一致。

### 12.4 安全边界测试

覆盖：

- runner 未启用时 `xray_run` 不在 tools/list。
- runner timeout 后 server 仍可响应 ping。
- runner cancel 后 server 仍可继续处理新 request。
- 超大输入被拒绝且不导致内存失控。

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

- runner toolset 默认关闭。
- 隔离执行。
- timeout/cancel/output limit。
- runner 安全测试。

## 14. 完成判定

MCP 重构完成必须同时满足：

- `xray mcp-server` 可被主流 MCP 客户端通过 stdio 正常初始化和调用。
- 不再存在 MCP Content-Length transport。
- 初始化、错误、notification、cancel、progress 行为有端到端测试。
- tools/list 只由 registry 产生，capabilities 只由 registry 推导。
- `xray_analyze` 能返回 parser + analyzer structured diagnostics。
- `xray_run` 不会无超时卡死 server。
- knowledge 和 stdlib 索引不再靠手写 C 大字符串维护。
- prompt 示例全部符合当前 Xray 语法。
- 完整 `ctest --output-on-failure` 通过。

## 15. 后续维护原则

- 新增 MCP feature 必须先注册 schema 和 tests，再接 handler。
- 新增 Xray 语法时，必须同步 knowledge 生成源和 prompt 示例测试。
- 新增 stdlib builtin 时，必须进入 stdlib symbol index。
- 任何不符合最终协议的旧入口直接删除，不添加兼容分支。
- MCP server 是 AI 客户端的可信语言接口，宁可少暴露 tool，也不要暴露语义不准或安全不可控的 tool。
