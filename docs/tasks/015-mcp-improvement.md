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
| `xray_format` | 是 | 格式化 Xray 源码，返回 formatted code；语法错误返回 structured diagnostics |
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
- resources/templates/list、prompts/list、prompts/get 的完整协议路径。
- resources/read 静态 URI 与 template URI 的协议路径。
- prompts/get unknown prompt 与缺少 name 的错误路径。
- 连续多条 NDJSON request 中混合 request、notification 和 error。
- tools/call transcript 覆盖 invalid params 与 structuredContent。

仍建议覆盖：

- 新增 method 或新 tool 时同步补 transcript smoke test。

### 2. 协议版本策略

当前版本固定为 `2025-03-26`。需要决定是否继续固定版本，还是实现版本协商。

可选策略：

- 保守策略：继续固定 `2025-03-26`，只实现该版本明确能力。
- 协商策略：读取 initialize 的 client protocolVersion，在服务器支持范围内选择版本。
- 升级策略：评估 `2025-06-18` 或更新版本的 breaking changes 后一次性升级。

### 3. 参数验证与错误分层

`tools/call` 已有轻量 schema validator，覆盖 required 字段、primitive type、string enum、integer range，并把这些协议参数错误返回为 JSON-RPC invalid params。

已覆盖：

- required 字段。
- primitive type。
- enum。
- integer range。

仍建议支持：

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

## 2026-05-22 源码审计发现

下面 16 条来自对 `src/app/mcp/` 的逐文件源码审计，独立于上面的业务 backlog。开发原则：Xray 无外部用户、可大胆重写、不保留旧接口、不做兼容层；每条直接采用最佳设计。

### P0 — 协议正确性偏差

**#1 `progressToken` 只接受 number，不支持 string**

MCP 规范 `ProgressToken = string | number`，多数 client（含 Claude Desktop、Cursor）发字符串 UUID。当前实现把 token 装在 `int64_t progress_token` 字段里、用 `xjson_get_int_or` 提取，字符串 token 被静默丢弃，progress 永不回报。

最佳设计：`XmcpCallContext.progress_token` 改成 `XrJsonValue *`（call 入口 `xjson_clone` 一份，call 结束统一释放）；`xmcp_send_progress_notification` 接收 `XrJsonValue *progress_token` 直接挂到 params 里；不存在时传 `NULL`。哨兵 `-1` 取消。

**#2 `prompts/get` 两条消息都是 `role: "user"`**

`build_prompt_messages` 注释写"system message"，role 却写 `user`。MCP `PromptMessage.role` 只允许 `user | assistant`，不允许 `system`。当前两条都是 `user`，client 无法区分指令和实际输入。

最佳设计：合并成单条 user message，把 SYSTEM_PREAMBLE + system_text 拼到 user_text 之前。

**#3 `mcp_dispatch` 用 `strcmp("initialize")` 特判生命周期**

整个 dispatch 已经表驱动，唯独"initialize 不可重入"是字符串比较。

最佳设计：`XmcpMethodEntry` 加 `XmcpLifecycleRequirement required_state` 字段（枚举：`ANY` / `MUST_BE_CREATED` / `MUST_BE_READY`），`needs_init` 合并进去；dispatch 表自洽。

### P1 — 性能 / 架构

**#4 `tool->build_schema()` 每次请求都重建 + 释放 JSON DOM**

`tools/list`、`tools/call` 参数校验都会触发。Schema 是不可变常量，却在每次请求时分配/释放整棵树。

最佳设计：把 `build_schema / build_output_schema` 改成 lazy getter（首次调用建好挂静态指针，后续返回同一指针）。调用方只读不释放。`tools/list` 输出时 `xjson_clone`；validator 直接读静态对象。

**#5 `xmcp_server.c` 反向依赖 `app/cli/*`**

CLI 和 MCP 都是 `src/app/` 兄弟模块，不应互相依赖。当前 MCP 通过 `xcli_isolate.h` 取 isolate 工厂，通过 `xcli_spec.h / xcli_diag.h` 实现 `cmd_mcp_server` CLI 入口。

最佳设计：
- CLI 入口 `cmd_mcp_server` 抽到 `src/app/cli/xcmd_mcp_server.c`，反方向依赖 `xmcp_server.h`；
- `xr_cli_isolate_new(profile)` 这种 isolate factory 下沉到 `src/api/`（或新设的共享层），CLI 和 MCP 都从公共层取。

**#6 `xray://stdlib/{module}` 资源用全文搜索代替直接取模块**

`read_stdlib_resource` 把模块名同时作为 query 和 module_filter 走 `xmcp_knowledge_search_stdlib`，等于让排序、tokens 切分、8KB 文本生成替代了 `O(1)` 取 body。

最佳设计：在 `xmcp_knowledge.[ch]` 加 `xmcp_knowledge_get_module(kb, name)` 返回 `const XmcpModule *`；resources 直接用，搜索路径只留给 `xray_stdlib_search` 工具。

**#7 `xmcp_knowledge_search_stdlib` 静默截断 + `xmcp_knowledge_load` 静默丢失**

- search_stdlib 用 8KB 固定 buffer，超出 `break` 但不告知，`structuredContent.matchCount` 与 `text` 不一致。
- knowledge_load 在 topics > 128 / modules > 64 时 silent break，新 topic 会神秘消失。

最佳设计：
- search_stdlib 改成动态 buffer（`xr_arena_vec` 或 `xr_realloc`），或在截断处明确追加 `\n_(truncated)_\n`；
- knowledge_load 用 `XR_DCHECK` 断言不超容量，构建期失败 fail-fast。

**#8 `xray_run` 用 `tmpfile()` 落临时文件**

每次调用都建 `/tmp/...`，再 fseek/fread 回内存，与"轻量沙箱"定位冲突。

最佳设计：POSIX 走 `open_memstream(&buf, &size)`，Windows 走 `tmpfile()` 兜底（或基于 `xarena_vec` 自实现 in-memory `FILE*`）。隐藏在一个 `xmcp_capture_open / xmcp_capture_finish` 小工具里。

### P2 — 可维护性 / 一致性

**#9 `xmcp_tools.c` 1299 行，开始巨石化**

虽然没破 3000 行红线，但已经混合了 table、schema builder、validator、result helper、6 个 handler、tools/list、tools/call dispatch。

最佳设计：拆 5 个文件
- `xmcp_tools.c`：table、tools/list、tools/call dispatch、validator
- `xmcp_tools_schema.c`：所有 `schema_*` builder
- `xmcp_tools_lang.c`：`tool_xray_analyze` / `tool_xray_format`（依赖 frontend）
- `xmcp_tools_run.c`：`tool_xray_run`（依赖 isolate / runtime）
- `xmcp_tools_kb.c`：`tool_xray_syntax_lookup` / `tool_xray_stdlib_search` / `tool_xray_definition`

**#10 tool handler 缺 NULL 防御**

`xjson_get_string` 在 key 缺失或类型错误时返回 NULL；handler 直接 `code[0] == '\0'`，依赖 schema validator 提前拦截。一旦 schema 写错就 SIGSEGV。

最佳设计：所有 `xjson_get_string` 后写成 `if (!s || s[0] == '\0')`，或在 validator 末尾对 string-typed required 字段追加 `XR_DCHECK(value->type == XR_JSON_STRING)`。

**#11 `tools/list` 的 cursor 分页是死代码**

`XMCP_PAGE_SIZE = 1000`，但 `XMCP_REGISTRY_MAX_TOOLS = 16`，永远不会触发。同时 cursor 按 tool name 字典序但 table 按业务顺序排列，真触发也会出错。

最佳设计：删除 cursor 逻辑，无条件返回全部工具；`resources/list` / `prompts/list` 已经无分页，统一行为。未来真要分页再用 table index 重写。

**#12 progress 字段整型不一致**

`xmcp_send_progress_notification(int64_t progress_token, int progress, int total)`：token 是 int64，progress/total 是 int。规范都是 number。

最佳设计：三者统一 `int64_t`。

**#13 `tool_xray_definition` 控制流错综**

两段 `if (text) { match==0 free else use } else error` 重复，加上 module/symbol 切分的二次搜索，5 个 return 点各自管 `xr_free`，新增搜索路径容易漏。

最佳设计：抽 `static XrJsonValue *try_lookup_stdlib(kb, query, module_filter)`，命中返回 result（已 strdup），未命中返回 NULL。definition 主体变成线性 `if (...) return; if (...) return; return not_found;`。

**#14 `xmcp_knowledge.c` 评分常量散落**

`score_symbol` 和 `score_module` 函数各自硬编码 6 个权重数字（`140 / 100 / 55 / 70 / 45 / 25 / 50 / 30 / 18 / 35 / 20 / 10`），无文档。

最佳设计：顶部定义 `XmcpScoreWeights` 表，三档（exact / contains / token）×五个字段（symbol_name / signature / symbol_summary / module_name / module_summary / module_body）；后续调权只改一处。

**#15 `xmcp_resources.c` 两个 reader 返回类型不对称**

`read_topic_resource` 返回 `const char *`（静态），`read_stdlib_resource` 返回 `char *`（owning）。调用处用两个变量分头管理释放。

最佳设计：结合 #6，`read_stdlib_resource` 改为返回 `const char *`（直接拿模块 body），调用处的 `dyn_text` 变量、双分支判断、`xr_free` 全部消失。

**#16 `xmcp_handle_initialize` 完全忽略 `params`**

未读 `clientInfo.name/version` 和 `protocolVersion`。MCP 客户端识别（Claude Desktop / Cursor / Continue）和未来版本协商无从下手。

最佳设计：至少把 client name 记到 info-level log，为 backlog 的协议版本协商打基础：

```c
const char *client_name = NULL;
XrJsonValue *client_info = params ? xjson_get_object(params, "clientInfo") : NULL;
if (client_info)
    client_name = xjson_get_string(client_info, "name");
mcp_log(server, 2, "initialize from client: %s", client_name ? client_name : "(unknown)");
```

### 实施顺序

每条修改后跑 `cmake --build build -j8 --target test_mcp_protocol test_mcp_transport` + 完整 `ctest`，必要时跑 `tests/mcp/test_protocol_transcript.py`。

第一批（P0 协议正确性，安全）：#3 → #11 → #12（最便宜，无外部行为变化）→ #2 → #1。

第二批（P1 架构 / 性能）：#16 → #4 → #7 → #6 + #15（合并）→ #10 → #8 → #5。

第三批（P2 重构）：#9 → #13 → #14。

未列入：015 既有 backlog 的 #2（协议版本协商）、#3 object/array 浅校验、#4 runner 内存配额、#5 ranking 多权重、#6 completion/complete、#7 prompt 内容生成化、#8 文档收敛——这些保留在原 backlog 推进。
