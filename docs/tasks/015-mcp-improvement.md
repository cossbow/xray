# Xray MCP 模块当前审计与后续优化

本文记录 `src/app/mcp/` 的当前状态和真实后续 backlog。旧版审计中的若干问题已经完成，后续开发以本文的状态表为准。

## 当前实现概览

### 协议与传输

- 专用 NDJSON stdio transport，MCP 不再复用 LSP/DAP 的 `Content-Length` framing。
- JSON-RPC 请求验证、错误响应 helper 和 method dispatch 已拆分为独立路径。
- 生命周期使用明确状态：created、initialize 已响应、ready。
- `notifications/cancelled` 在当前顺序 dispatch 模型下不可抢占，handler 明确记录 no-op 语义。
- `progressToken` 已经是 per-call `XmcpCallContext`，不再放在 server 全局状态中。
- 当前协议版本仍是 `2025-03-26`。

### Registry 与 capabilities

- Tools、resources、resource templates、prompts 由 `XmcpRegistry` 统一持有。
- `initialize` capabilities 从 registry 动态推导。
- logging capability 始终声明。
- runner toolset 通过 `--enable-runner` 显式启用，默认关闭。

### 已有 tools

| Tool | 默认启用 | 说明 |
|---|---:|---|
| `xray_analyze` | 是 | Parser + analyzer 诊断，返回 text summary 和 structured diagnostics |
| `xray_format` | 是 | 格式化 Xray 源码，返回 formatted code 和结构化字段 |
| `xray_syntax_lookup` | 是 | 查询 generated syntax topic |
| `xray_stdlib_search` | 是 | 查询 generated stdlib module / symbol index |
| `xray_definition` | 是 | 综合查询 syntax topic 与 stdlib symbol |
| `xray_run` | 否 | 一次性 isolate 运行短代码，带 timeout、output limit 和 structured output |

### 已有 resources 与 prompts

- 静态 resources：cheatsheet、concurrency model、stdlib module list。
- Resource templates：`xray://spec/topic/{name}`、`xray://stdlib/{module}`。
- Prompts：`code-review`、`explain-error`、`convert-to-xray`、`concurrency-pattern`、`write-test`。

### Knowledge 生成链

- 语言语义和权威示例来自 `docs/spec/source/sections/*.md`。
- MCP 投影来自 `docs/spec/source/cards/**/*.json`。
- `docs/knowledge/**`、`LANGUAGE_SPEC_CN.md`、`LANGUAGE_SPEC.md` 由 `scripts/gen_language_docs.py` 生成。
- `src/app/mcp/xmcp_knowledge_generated.c` 由 generated knowledge 和 analyzer builtin dump 生成。
- stdlib API 签名来自 analyzer builtin metadata，不在 card 中手写复制。

## 已关闭的旧问题

| 旧问题 | 当前状态 |
|---|---|
| MCP stdio 使用 `Content-Length` | 已切换为 NDJSON transport |
| 缺少 JSON-RPC validation | 已有 validator 和错误 helper |
| capabilities 纯硬编码 | 已从 registry 动态推导 |
| 无 prompts | 已有 5 个 prompt |
| 无 resource templates | 已有 topic 和 stdlib template |
| 无 tool annotations / output schema | tools/list 已输出 annotations；主要 tools 已有 outputSchema |
| progress token 是全局状态 | 已改为 per-call context |
| knowledge 为手写大块 C 字符串 | 已改为 cards + generated C table |
| prompt 示例缺少语法 smoke test | 已进入 MCP knowledge 回归 |
| sections/cards 示例可能漂移 | 已增加 card fence 引用和 orphan fence coverage 门禁 |

## 真实剩余 backlog

### 1. 协议一致性测试

当前测试覆盖了 transport、protocol handler、knowledge generation 和核心 stdio transcript 路径。

已覆盖：

- initialize -> initialized -> tools/list。
- ready 前调用普通 request 被拒绝。
- 重复 initialize 的行为。
- unknown method 返回 method not found。
- malformed JSON 返回 parse error。
- notification 不返回 response。
- 拒绝或忽略 `Content-Length` 输入。
- runner stdout 不污染 MCP stdout 协议流。

仍建议覆盖：

- resources/templates/list、prompts/list、prompts/get 的完整协议路径。
- 连续多条 NDJSON request 中混合 request、notification 和 error。
- 更完整的 tools/call transcript，包括 invalid params 与 structuredContent。

### 2. 协议版本策略

当前版本固定为 `2025-03-26`。需要决定是否继续固定版本，还是实现版本协商。

可选策略：

- 保守策略：继续固定 `2025-03-26`，只实现该版本明确能力。
- 协商策略：读取 initialize 的 client protocolVersion，在服务器支持范围内选择版本。
- 升级策略：评估 `2025-06-18` 或更新版本的 breaking changes 后一次性升级。

### 3. 参数验证与错误分层

部分 handler 仍然包含手动参数检查。后续应统一为轻量 schema validator。

建议支持：

- required 字段。
- primitive type。
- enum。
- integer range。
- object / array 的浅层结构。

错误分层规则：

- 参数缺失、类型错误、未知 tool/resource 属于 JSON-RPC invalid params。
- Xray 源码自身的语法、语义、运行时失败属于 tool result `isError=true`。

### 4. Runner 沙盒加固

`xray_run` 已有默认关闭、一次性 isolate、timeout、output limit 和 structured output；unit coverage 已覆盖 runner 默认关闭、outputSchema、基础输出、截断、timeout 后 server 继续可用，以及危险模块 import 拒绝。

仍建议补齐：

- 更细粒度的 module allowlist 测试：允许模块可用、禁止模块不可用，错误信息稳定。
- 评估内存配额或子进程 runner。

### 5. Knowledge 搜索排序

当前 generated knowledge 已消除源漂移，搜索排序已支持 alias exact match、`module.symbol` 直达和多词 query 的 module-context 加权。

已覆盖：

- alias exact match 优先。
- stdlib `module.symbol` 直达。
- 多词 query 做 token 交集评分。

仍建议优化：

- title、alias、lead、body 分权重。
- 对无命中的 query 返回相近 topic/module 建议。

### 6. Resource completion 与更多 templates

当前已有两个 resource template，但还没有 completion。

建议增加：

- `completion/complete` for topic name。
- `completion/complete` for stdlib module。
- `xray://stdlib/{module}/{symbol}`。
- `xray://spec/anchor/{anchor}`。

### 7. Prompt 内容生成化

Prompt 已进入 smoke test，但内容仍是 C 中手写摘要。

建议：

- prompt 中的固定语言规则尽量从 generated knowledge 摘要派生。
- prompt 示例继续由测试覆盖。
- 不在 prompt 中复制与 sections/cards 冲突的独立语义。

### 8. 文档和任务文件收敛

`015`、`016`、`083` 都记录过 MCP 设计。后续应保持单一当前状态入口，避免旧任务文档继续误导实现判断。

建议：

- `015` 作为当前状态和 backlog。
- `083` 作为最终设计原则和完成判定。
- `016` 作为历史参考，必要时只保留仍有价值的参考实现摘录。

## 维护规则

- 修改 Xray 语法或语义时，先更新 `sections/*.md` 的中文和英文正文，再更新相关 card。
- 需要 MCP 复用的 Xray 示例必须放在 `sections/*.md` 的 `xray @id=...` fence 中，再由 card `fences` 引用。
- 新增 `@id` fence 后必须有至少一个 card 引用。
- 新增 topic card 后必须同步 tool/resource metadata 列表。
- 新增 stdlib builtin 后必须确保 analyzer builtin dump 和 generated MCP table 同步。
- 每次修改后运行 MCP knowledge 回归和完整 `ctest`。
