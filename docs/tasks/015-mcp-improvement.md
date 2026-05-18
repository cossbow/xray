# Xray MCP 模块优化建议

> 基于对官方 TypeScript SDK、Go SDK、mcp-language-server、everything reference server 的深入源码阅读，
> 对照 Xray 现有 MCP 模块（`src/app/mcp/`）产出的改进方案。

---

## 一、现状分析

### 1.1 Xray MCP 模块当前架构

```
xmcp_server.c      — stdio 传输 + JSON-RPC 2.0 分发主循环
xmcp_protocol.c/h  — initialize 响应构建
xmcp_tools.c/h     — tools/list + tools/call (3 个工具)
xmcp_resources.c/h — resources/list + resources/read (3 个静态资源)
xmcp_knowledge.c/h — 内嵌知识库 (语法主题 + stdlib 索引)
```

### 1.2 已实现的功能

| 功能 | 状态 |
|------|------|
| stdio 传输 | ✅ 基本可用 |
| JSON-RPC 2.0 请求/响应 | ✅ |
| initialize / initialized 握手 | ✅ |
| ping | ✅ |
| tools/list, tools/call | ✅ 3 个工具 |
| resources/list, resources/read | ✅ 3 个静态资源 |
| xray_check (语法检查) | ✅ |
| xray_syntax_lookup (语法查询) | ✅ |
| xray_stdlib_search (stdlib 搜索) | ✅ |

### 1.3 与参考实现的差距总结

| 维度 | Xray 现状 | 参考实现标准 |
|------|-----------|-------------|
| 协议版本 | `2025-03-26` | 最新 `2025-06-18` 已增加 tasks、elicitation 等 |
| 能力声明 | 仅 tools + resources(且 resources.listChanged=false） | 应根据实际注册动态推导，含 prompts、logging、completions |
| 错误处理 | 基本的 JSON-RPC error | 缺少标准错误码枚举、method not found 等 |
| 进度通知 | 无 | 长运行工具应发 `notifications/progress` |
| Prompts | 无 | 重要特性：预定义提示模板 |
| Resource Templates | 无 | URI 模板 + 动态资源生成 |
| 日志通知 | 无 | `notifications/message` 日志级别 |
| 工具 annotations | 无 | priority、audience、readOnlyHint 等元数据 |
| 分页 | 无 | 大列表需 cursor-based 分页 |
| 动态注册 | 无 | listChanged 通知 |
| 输入/输出 schema 验证 | 无 | Go SDK 和 TS SDK 都做严格验证 |
| 结构化输出 | 无 | structuredContent + outputSchema |
| 优雅关闭 | 部分 | 缺少 parent process 监控 |

---

## 二、优化建议（按优先级分阶段）

### Phase 1：协议合规与健壮性（高优先级）

#### P1.1 完善 JSON-RPC 错误处理

**问题**：当前 `xmcp_server.c` 遇到未知方法时直接 `MCP_LOG_DEBUG` 忽略，不返回错误响应。

**建议**：

- 定义标准错误码枚举（参考 TypeScript SDK `schemas.ts`）：
  ```c
  #define XMCP_ERR_PARSE       (-32700)
  #define XMCP_ERR_INVALID_REQ (-32600)
  #define XMCP_ERR_NOT_FOUND   (-32601)
  #define XMCP_ERR_INVALID_PARAMS (-32602)
  #define XMCP_ERR_INTERNAL    (-32603)
  ```
- 对每个带 `id` 的请求，**必须**返回 response（成功或错误）
- 对 notification（无 `id`）的未知方法，可安全忽略但应 debug log

**参考**：Go SDK `server.go:748` 对 unknown tool 返回 `jsonrpc.CodeInvalidParams` 错误。

#### P1.2 修正 Capabilities 声明

**问题**：当前 capabilities 是硬编码的，`resources.listChanged` 设为 false，且缺少 logging 能力。

**建议**：

- 采用 Go SDK 的**自动推导模式**：注册了 tools 则自动声明 `tools: {listChanged: true}`，注册了 resources 同理
- 添加 `logging: {}` 能力声明
- 如果后续增加 prompts，自动声明 `prompts: {listChanged: true}`
- 将 capabilities 构建逻辑从 `xmcp_protocol.c` 移至可动态查询注册状态的函数

```c
/* Pseudocode for dynamic capability inference */
static XrJsonValue *build_capabilities(XmcpServer *server) {
    XrJsonValue *caps = xlsp_json_new_object();
    if (server->tool_count > 0) {
        XrJsonValue *tc = xlsp_json_new_object();
        XLSP_JSON_SET_BOOL(tc, "listChanged", true);
        xlsp_json_object_set(caps, "tools", tc);
    }
    // ... similarly for resources, prompts
    xlsp_json_object_set(caps, "logging", xlsp_json_new_object());
    return caps;
}
```

#### P1.3 工具输入 Schema 验证

**问题**：当前 `tools/call` 不验证参数是否符合声明的 inputSchema。错误的参数可能导致段错误或不可预测行为。

**建议**：

- 在 `xmcp_handle_tools_call` 中，调用工具前**至少**验证必需字段存在且类型正确
- 短期方案：手动校验（当前工具只有简单的 string 参数）
- 长期方案：构建轻量 JSON Schema 验证器（或仅验证 required + type）

**参考**：Go SDK `toolForErr()` 函数使用 `jsonschema.Resolved` 对入参做严格 validate + 默认值填充。

#### P1.4 优雅关闭

**问题**：当前主循环在 stdin EOF 时退出，但没有处理 SIGTERM/SIGINT，也没有监控父进程。

**建议**：

- 添加信号处理（SIGTERM, SIGINT → 设置 `server->shutdown = true`）
- 参考 `mcp-language-server` 的父进程监控：轮询 `getppid()` 检测父进程退出
- 关闭时清理 isolate 和 knowledge 资源

```c
/* In server main loop */
signal(SIGTERM, handle_signal);
signal(SIGINT, handle_signal);
```

---

### Phase 2：功能扩展（中优先级）

#### P2.1 添加 Prompts 支持

**为什么重要**：Prompts 让 AI 助手获取预定义的提示模板，极大提升交互效率。对于 Xray 语言，这是让 AI 快速了解语言特性的最高效方式。

**建议添加的 Prompts**：

| Prompt 名称 | 描述 | 参数 |
|-------------|------|------|
| `code-review` | 审查 Xray 代码的最佳实践 | `code: string` |
| `explain-error` | 解释 Xray 编译/运行时错误 | `error: string` |
| `convert-to-xray` | 将其他语言代码转换为 Xray | `code: string`, `sourceLanguage: string` |
| `concurrency-pattern` | 推荐并发模式 | `description: string` |
| `write-test` | 为 Xray 代码生成测试 | `code: string` |

**实现方式**：

新建 `xmcp_prompts.c/h`，参考 everything server 的 prompts 目录结构：

```c
typedef struct {
    const char *name;
    const char *title;
    const char *description;
    /* args schema */
    int arg_count;
    struct { const char *name; const char *description; bool required; } args[4];
    /* handler returns GetPromptResult as JSON */
    XrJsonValue *(*handler)(XmcpServer *server, XrJsonValue *args);
} XmcpPromptDef;
```

Handler 返回 `messages` 数组，每个 message 含 `role` + `content`，将 Xray 语言规范嵌入 system prompt 中。

#### P2.2 添加 Resource Templates

**问题**：当前只有 3 个硬编码的静态资源。

**建议**：增加 URI 模板资源，支持动态查询：

| URI Template | 功能 |
|-------------|------|
| `xray://spec/topic/{topicName}` | 按主题查询语法规范 |
| `xray://stdlib/{moduleName}` | 查询特定 stdlib 模块详情 |
| `xray://project/{filePath}` | 读取项目文件（如有工作区） |

**实现**：

```c
typedef struct {
    const char *uri_template;
    const char *name;
    const char *description;
    const char *mime_type;
    XrJsonValue *(*read_handler)(XmcpServer *server, const char *uri,
                                  /* parsed variables */ ...);
} XmcpResourceTemplateDef;
```

需要实现简单的 URI template 匹配（RFC 6570 Level 1，仅 `{variable}` 替换）。

#### P2.3 进度通知（notifications/progress）

**问题**：`xray_check` 对大文件可能耗时较长，AI 客户端无法获知进度。

**建议**：

- 在工具回调中增加 `progressToken` 检查（来自 `_meta.progressToken`）
- 对 `xray_check` 等可能耗时的工具，发送进度通知

```c
/* Check if client requested progress tracking */
XrJsonValue *meta = xlsp_json_get(params, "_meta");
int64_t progress_token = -1;
if (meta) {
    progress_token = xlsp_json_get_int(meta, "progressToken", -1);
}

/* During long operation, send progress */
if (progress_token >= 0) {
    xmcp_send_progress(server, progress_token, current, total);
}
```

**参考**：everything server 的 `trigger-long-running-operation.ts`。

#### P2.4 日志通知（notifications/message）

**建议**：

- 将当前的 `MCP_LOG_*` 宏改为同时发送 MCP logging notification
- 日志级别映射：`debug/info/warning/error/critical`
- 仅在 initialized 后发送

```c
void xmcp_send_log(XmcpServer *server, const char *level, const char *message) {
    if (!server->initialized) return;
    XrJsonValue *params = xlsp_json_new_object();
    XLSP_JSON_SET_STRING(params, "level", level);
    xlsp_json_object_set(params, "data", xlsp_json_new_string(message));
    xmcp_send_notification(server, "notifications/message", params);
}
```

#### P2.5 工具 Annotations

**建议**：为每个工具添加 MCP annotations 元数据，帮助 AI 客户端理解工具的行为特征：

```c
/* xray_check: read-only analysis, no side effects */
"annotations": {
    "title": "Xray Code Checker",
    "readOnlyHint": true,
    "destructiveHint": false,
    "openWorldHint": false
}

/* Future: xray_format could be non-read-only */
"annotations": {
    "readOnlyHint": false,
    "destructiveHint": false
}
```

**参考**：everything server 的 `get-annotated-message.ts` 展示了 content annotations（priority、audience）。

---

### Phase 3：新工具（中优先级）

基于 mcp-language-server 的工具设计和 Xray 的 LSP 能力，建议添加以下工具：

#### P3.1 `xray_format` — 代码格式化

```json
{
    "name": "xray_format",
    "description": "Format Xray source code according to standard style",
    "inputSchema": {
        "type": "object",
        "properties": {
            "code": {"type": "string", "description": "Xray source code to format"}
        },
        "required": ["code"]
    }
}
```

**实现**：复用 `src/frontend/format/xfmt.c` 的格式化器。

#### P3.2 `xray_run` — 执行代码片段

```json
{
    "name": "xray_run",
    "description": "Execute a small Xray code snippet and return output",
    "inputSchema": {
        "type": "object",
        "properties": {
            "code": {"type": "string", "description": "Xray code to execute"},
            "timeout": {"type": "number", "description": "Timeout in milliseconds", "default": 5000}
        },
        "required": ["code"]
    },
    "annotations": {
        "readOnlyHint": false,
        "openWorldHint": true
    }
}
```

**注意**：需要沙盒化执行，限制时间和内存。可以设置 isolate 的资源限制。

#### P3.3 `xray_diagnostics` — 获取文件诊断信息

类似 mcp-language-server 的 `diagnostics` 工具，但直接调用 Xray 的分析器：

```json
{
    "name": "xray_diagnostics",
    "description": "Get detailed diagnostic information (errors, warnings, hints) for Xray source code",
    "inputSchema": {
        "type": "object",
        "properties": {
            "code": {"type": "string", "description": "Xray source code"},
            "filePath": {"type": "string", "description": "Optional file path for context"}
        },
        "required": ["code"]
    }
}
```

#### P3.4 `xray_definition` — 符号定义查找

参考 mcp-language-server 的 `definition` 工具：

```json
{
    "name": "xray_definition",
    "description": "Find the definition of a symbol in the Xray standard library or project",
    "inputSchema": {
        "type": "object",
        "properties": {
            "symbolName": {"type": "string"},
            "filePath": {"type": "string", "description": "File context for resolving symbols"}
        },
        "required": ["symbolName"]
    }
}
```

---

### Phase 4：架构改进（低优先级但重要）

#### P4.1 请求分发表驱动

**问题**：当前 `xmcp_server.c` 用 `strcmp` 链做方法分发，每添加一个方法需要修改主循环。

**建议**：改为表驱动分发（参考 Go SDK 的 methodHandler 模式）：

```c
typedef struct {
    const char *method;
    bool        is_notification;  /* true = no response expected */
    XrJsonValue *(*handler)(XmcpServer *server, XrJsonValue *params);
} XmcpMethodEntry;

static const XmcpMethodEntry METHOD_TABLE[] = {
    {"initialize",                  false, xmcp_handle_initialize_wrapper},
    {"notifications/initialized",   true,  xmcp_handle_initialized},
    {"ping",                        false, xmcp_handle_ping},
    {"tools/list",                  false, xmcp_handle_tools_list_wrapper},
    {"tools/call",                  false, xmcp_handle_tools_call_wrapper},
    {"resources/list",              false, xmcp_handle_resources_list_wrapper},
    {"resources/read",              false, xmcp_handle_resources_read_wrapper},
    {"prompts/list",                false, xmcp_handle_prompts_list_wrapper},
    {"prompts/get",                 false, xmcp_handle_prompts_get_wrapper},
    /* ... */
    {NULL, false, NULL}
};
```

主循环只需遍历表查找匹配项，代码更清晰、更易扩展。

#### P4.2 工具注册表驱动

**问题**：当前工具定义和 handler 混在一起，添加工具需要修改多处。

**建议**：参考 Go SDK 的 `featureSet` 和 TypeScript SDK 的 `RegisteredTool` 模式，将工具注册为运行时数据结构：

```c
typedef struct {
    const char    *name;
    const char    *title;
    const char    *description;
    XrJsonValue   *input_schema;   /* JSON Schema object */
    XrJsonValue   *output_schema;  /* optional */
    XrJsonValue   *annotations;    /* optional */
    bool           enabled;
    XrJsonValue *(*handler)(XmcpServer *server, XrJsonValue *args);
} XmcpRegisteredTool;

typedef struct XmcpServer {
    /* ... existing fields ... */
    XmcpRegisteredTool tools[32];
    int tool_count;
    /* Similar for resources, prompts */
} XmcpServer;
```

这样 `tools/list` 和 `tools/call` 的实现就是纯数据驱动的，添加新工具只需要在初始化时 `xmcp_register_tool()`。

#### P4.3 通知发送基础设施

**建议**：抽象出通用的通知发送函数：

```c
/* Send a JSON-RPC notification (no id, no response expected) */
XR_FUNC void xmcp_send_notification(XmcpServer *server,
                                      const char *method,
                                      XrJsonValue *params);

/* Convenience wrappers */
XR_FUNC void xmcp_send_progress(XmcpServer *server,
                                  int64_t token, int progress, int total);
XR_FUNC void xmcp_send_log(XmcpServer *server,
                             const char *level, const char *message);
XR_FUNC void xmcp_send_tool_list_changed(XmcpServer *server);
XR_FUNC void xmcp_send_resource_list_changed(XmcpServer *server);
```

#### P4.4 分页支持

**问题**：如果后续工具/资源数量增长，需要支持 cursor-based 分页。

**建议**：参考 Go SDK 的 `paginateList` 实现，使用 opaque cursor（实际上是排序后的最后一个 key 的编码）：

- `tools/list` 接受 `cursor` 参数
- 返回 `nextCursor` 字段
- 默认 page size 可配置（Go SDK 默认 1000）

当前工具只有 3 个，暂不紧急，但架构应预留扩展点。

#### P4.5 独立 JSON 模块

**问题**：当前依赖 `../lsp/xlsp_json.h`，MCP 和 LSP 共用 JSON 工具函数。

**建议**：

- 短期：维持现状，共用是合理的
- 长期：如果 MCP 和 LSP 的 JSON 需求分化（例如 MCP 需要 JSON Schema 验证），考虑将通用 JSON 操作下沉到 `src/base/` 或 `src/frontend/`

---

### Phase 5：测试与质量（持续）

#### P5.1 MCP 协议一致性测试

**建议**：参考 Go SDK 的 `conformance_test.go`（txtar 格式的测试用例），创建协议一致性测试：

- 验证 initialize 握手完整流程
- 验证 unknown method 返回正确错误码
- 验证 tools/list 返回格式合规
- 验证 tools/call 参数校验
- 验证 notification 不返回 response

可以用 shell 脚本 + 管道模拟 stdio MCP 会话：

```bash
# tests/mcp/test_initialize.sh
echo 'Content-Length: N\r\n\r\n{"jsonrpc":"2.0","id":1,"method":"initialize",...}' | xray mcp-server | verify_response
```

#### P5.2 参考 everything server 的测试模式

everything server 有独立的测试文件：
- `__tests__/tools.test.ts`
- `__tests__/resources.test.ts`
- `__tests__/prompts.test.ts`

建议为 Xray MCP 创建类似的单元测试：`tests/unit/mcp/test_mcp_tools.c` 等。

---

## 三、实施路线图

### 第一阶段（1-2 周）— 协议合规

1. P1.1 错误码枚举 + unknown method 返回 error response
2. P1.2 capabilities 动态推导
3. P1.3 工具入参基本验证
4. P1.4 信号处理 + 优雅关闭
5. P5.1 基础协议一致性测试

### 第二阶段（2-3 周）— 功能扩展

1. P4.1 方法分发表驱动重构
2. P4.2 工具注册表驱动重构
3. P4.3 通知发送基础设施
4. P2.1 Prompts 支持（3-5 个实用 prompt）
5. P2.5 工具 annotations
6. P2.4 日志通知

### 第三阶段（2-3 周）— 新工具 + 资源

1. P3.1 xray_format 工具
2. P3.3 xray_diagnostics 工具
3. P2.2 Resource Templates
4. P2.3 进度通知
5. P3.2 xray_run 工具（需沙盒设计）

### 第四阶段（持续改进）

1. P3.4 xray_definition 工具（需 LSP 集成）
2. P4.4 分页支持
3. 协议版本升级到 2025-06-18（tasks、elicitation）
4. 更多 prompts 和 resource templates

---

## 四、参考源码清单

| 仓库 | 关键文件 | 学习重点 |
|------|---------|---------|
| typescript-sdk | `packages/core/src/types/schemas.ts` | 完整 MCP 协议 schema 定义 |
| typescript-sdk | `packages/server/src/server/mcp.ts` | 高层 server API、工具/资源/prompt 注册管理、task augmentation |
| go-sdk | `mcp/server.go` | featureSet 注册模式、capabilities 自动推导、分页、change 通知 debounce |
| go-sdk | `mcp/features.go` | 泛型 featureSet 实现，可作为 C 版本数据结构参考 |
| mcp-language-server | `main.go`, `tools.go` | LSP→MCP 桥接模式、工具注册、父进程监控、优雅关闭 |
| servers/everything | `server/index.ts` | server factory 模式、capabilities 全量声明、post-init 条件注册 |
| servers/everything | `tools/*.ts` | 丰富的工具模式：annotations、long-running、structured content |
| servers/everything | `prompts/*.ts` | prompt 注册、completions、embedded resource prompt |
| servers/everything | `resources/templates.ts` | URI template 资源、completer 自动补全 |

---

## 五、关键设计决策建议

### 5.1 C 语言下的注册表模式

由于 C 没有泛型和闭包，建议用**函数指针表 + void\* 上下文**模式：

```c
typedef XrJsonValue *(*XmcpToolHandler)(XmcpServer *server, XrJsonValue *args);

typedef struct {
    const char       *name;
    const char       *description;
    XrJsonValue      *input_schema;
    XmcpToolHandler   handler;
    bool              enabled;
} XmcpTool;
```

注册时用静态数组或链表存储，分发时线性查找（工具数量 <50 时性能足够）。

### 5.2 通知 debounce

Go SDK 在 `changeAndNotify` 中使用 10ms 的 timer debounce 批量变更通知。由于 Xray MCP 是单线程 stdio，这不是必须的，但如果后续添加动态工具注册（如按文件类型启用工具），则应考虑。

### 5.3 协议版本协商

当前硬编码 `2025-03-26`。未来应支持：
- 从客户端 initialize 请求中读取 `protocolVersion`
- 服务器回复**自己支持的最高兼容版本**
- 根据协商版本启用/禁用功能（如 tasks 仅在 2025-06-18+ 可用）

### 5.4 与 LSP 的关系

mcp-language-server 展示了 LSP→MCP 桥接模式。Xray 已有强大的 LSP（`src/app/lsp/`），长期可以：
- MCP 工具复用 LSP 的分析能力（如 diagnostics、definition、references）
- 但 MCP server 和 LSP server 应保持独立进程，避免耦合
- 共享知识库和格式化器是合理的

---

*文档生成日期：基于 MCP 协议 2025-03-26/2025-06-18 规范*
*参考源码版本：typescript-sdk latest, go-sdk latest, mcp-language-server v0.0.2, servers/everything v2.0.0*
