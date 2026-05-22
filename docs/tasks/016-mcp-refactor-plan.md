# 016 - MCP 模块重构方案

> 状态说明：本文保留为参考实现调研与历史施工路线。当前实现状态、已关闭问题和真实剩余 backlog 以 `015-mcp-improvement.md` 为准。
> 文中关于 Content-Length、无 prompts、无 resource templates、手写 knowledge 等描述已经过时，阅读时应只把相应章节作为设计背景。

> 基于对 12 个参考实现的源码阅读，结合 015 现状审计，给出 xray MCP 模块的目标架构与实施路线。
> 015 关注**问题清单**，本文关注**目标设计与施工路径**。

## 0. 阅读路径

| 你是谁 | 推荐阅读 |
|--|--|
| 想了解为何重构 | 第 1、2 节 |
| 要 review 设计 | 第 3、4 节 |
| 要落地实施 | 第 5、6 节 |
| 关心兼容性 | 第 4.10、第 7 节 |

**前置文档**：`docs/tasks/015-mcp-improvement.md`（现状问题清单）。

---

## 1. 背景与目标

### 1.1 当前现状（一句话）

xray 已内置 `xray mcp-server` 子命令，实现 `2025-03-26` 协议的子集（7 tools / 5 prompts / 3+2 resources），但存在 **6 个 P0 协议合规问题**——其中最严重的 stdio framing 用了 LSP 的 `Content-Length` 头，**与所有主流 MCP 客户端不兼容**。

详细问题清单见 015。本文不重复审计，专注重构。

### 1.2 重构目标

按重要程度排列：

1. **协议合规**：升级到 `2025-11-25`（最新稳定版），使用规范规定的 NDJSON stdio framing。
2. **客户端互操作性**：跑通 Claude Desktop / Cursor / Windsurf / VS Code Copilot / Cline / Codex CLI。
3. **架构清晰**：注册表驱动，工具/资源/提示三套统一抽象，capability 自动推导。
4. **数据自治**：知识库从 xray 源码自动提取（stdlib 模块清单、语法规范），不再手维护 889 行 C 字符串。
5. **可中断**：长运行 tool（`xray_run`）能被 `notifications/cancelled` 中断。
6. **沙盒安全**：`xray_run` 有超时、内存、模块白名单。
7. **可测试**：协议一致性测试 + 工具单测 + Inspector 联调。

### 1.3 不做什么

- 不做 Streamable HTTP 传输（xray 是开发工具，不是远程服务，stdio 足够）。
- 不做 OAuth / 鉴权（同上）。
- 不做 sampling client capability（xray 不主动调用 LLM）。
- 不做 elicitation 的复杂 UI 交互（最小化支持）。
- 不在 MCP server 里启动 LSP 子进程 — xray 的 LSP 已经在同一二进制内，**直接 in-process 复用**（这正是相对 mcp-language-server 的天然优势）。

---

## 2. 参考实现关键学习总结

下面每条都注明**借鉴自哪个仓库的哪个文件**，并提炼出**对 xray 的具体启示**。

### 2.1 NDJSON 框架（修复 P0 #1）

| 实现 | 文件 | 做法 |
|---|---|---|
| TypeScript SDK | `packages/core/src/shared/stdio.ts` (51 行) | `ReadBuffer.append/readMessage()`：累积 buffer，按 `\n` 切分；非 JSON 行容错 skip；`\r\n` 兼容 |
| Go SDK | `mcp/transport.go:351-433` (`ioConn`) | `json.NewDecoder(rwc)` 流式 decode，trailing 字节是 `\n` 或 `\r\n` 才合法 |
| Python SDK | `src/mcp/server/stdio.py` | `for line in stdin: validate_json(line)` + `stdout.write(json + "\n")` |
| mcp-go | `server/stdio.go:480-499` | `bufio.Reader.ReadString('\n')`，goroutine + channel 让阻塞读可被 ctx 取消 |
| Dart MCP | `pkgs/dart_mcp/lib/stdio.dart` (28 行) | `stdioChannel`：`LineSplitter` + `'$data\n'` |

**对比 LSP 框架**：同一份 Dart MCP 代码库里 `pkgs/dart_mcp_server/lib/src/lsp/wire_format.dart` 用 `Content-Length: N\r\n\r\n` 给 LSP 用，**两者明确区分**。

**xray 当前实现**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/app/mcp/xmcp_server.c:117-207` 用的是 `xr_frame_parse`（LSP framing）。直接错。

**对 xray 的启示**：
- 丢掉 `xr_frame_parse` / `xr_frame_write_header`，新建 `xmcp_transport_stdio.c`，约 80-100 行 C。
- Read：累积 buffer + `memchr(buf, '\n', n)` 切行；非 JSON 行（容错 hot-reload 工具的干扰输出）记 debug log 但不报错（参考 TS SDK 注释）。
- Write：`xjson_stringify(msg)`（确保不带裸 `\n`） + 直接 `write(fd, json, len); write(fd, "\n", 1);`。

### 2.2 注册表模式（核心架构）

四种风格，按 C 实现的可移植性排序：

#### (a) Go SDK `featureSet[T]` —— 泛型容器

`@/Users/xuxinglei/workspace/参考代码/mcp/modelcontextprotocol/go-sdk/mcp/features.go`（115 行）：

```go
type featureSet[T any] struct {
    uniqueID   func(T) string
    features   map[string]T
    sortedKeys []string  // lazy
}
```

只有 `add / remove / get / len / all / above(uid)` 6 个方法。lazy-sorted keys 用于稳定枚举。`above` 用于 cursor-based 分页（`tools/list?cursor=xxx`）。

#### (b) GitHub MCP `inventory` —— Stateless builder + handler 工厂

`@/Users/xuxinglei/workspace/参考代码/mcp/github/github-mcp-server/pkg/inventory/server_tool.go`：

```go
type ServerTool struct {
    Tool        mcp.Tool       // 静态描述
    Toolset     ToolsetMetadata
    HandlerFunc HandlerFunc    // func(deps any) mcp.ToolHandler
    FeatureFlagEnable string
    FeatureFlagDisable string
    Enabled func(ctx) (bool, error)
    RequiredScopes []string
}
```

Handler 不是 `ToolHandler`，是 `func(deps) ToolHandler` —— **handler 在注册时按依赖生成**，工具元数据本身是 stateless 的，可以被 filter / read-only 模式 / feature flag 任意筛掉。

#### (c) TypeScript SDK `registerTool` —— ergonomic 链式 API

`@/Users/xuxinglei/workspace/参考代码/mcp/modelcontextprotocol/typescript-sdk/packages/server/src/server/mcp.ts:867-921`：

```typescript
server.registerTool(name, {
    title, description, inputSchema, outputSchema, annotations, _meta
}, async (args) => ({ content, structuredContent }))
```

inputSchema 是 zod schema，自动转 JSON Schema 输出给 client。

#### (d) Dart MCP Mixin —— feature composition

`@/Users/xuxinglei/workspace/参考代码/mcp/dart-lang/ai/pkgs/dart_mcp_server/lib/src/server.dart:38-61`：

```dart
final class DartMCPServer extends MCPServer
    with LoggingSupport, ToolsSupport, ResourcesSupport,
         RootsTrackingSupport, DartAnalyzerSupport, PubSupport,
         FlutterLauncherSupport, GrepSupport, ...
```

每个 mixin 自己 `registerTool(...)` —— 加新功能不改主类，只加 mixin。

**对 xray 的启示**：

C 没泛型也没 mixin，但综合上面四种思路，得到一个 **C-friendly 的工具注册表设计**：

```c
// 静态自描述（编译期）
typedef struct {
    const char *name;
    const char *title;
    const char *description;
    const char *toolset;            // "core" / "lsp_bridge" / "runner" / "knowledge"
    bool        read_only;
    bool        open_world;
    bool        default_enabled;    // 是否在 default 模式下启用
    XmcpSchemaBuilder build_schema; // 函数指针，按需构建 JSON Schema
    XmcpToolHandler   handler;      // (server, args) -> result
} XmcpToolDef;

// 工具集是 toolset_id -> ToolDef[] 的索引
typedef struct {
    const char         *id;
    const char         *description;
    bool                default_enabled;
    const XmcpToolDef  *tools;
    size_t              tool_count;
} XmcpToolset;

// 运行时注册表（启动时按客户端 capabilities 和 CLI 选项过滤填充）
typedef struct {
    const XmcpToolDef **enabled_tools;  // 指针数组（指向静态表）
    size_t              enabled_count;
    // 按 name 排序好，便于 cursor 分页
} XmcpRegistry;
```

每个 .c 文件提供 `static const XmcpToolDef MY_TOOLS[]`，启动时 `xmcp_registry_register(reg, MY_TOOLS, count, &my_toolset)`。

### 2.3 Capability 自动推导

Go SDK `mcp/server.go:576-624`：注册表里有什么就声明什么。

```go
func (s *Server) capabilities() *ServerCapabilities {
    caps := &ServerCapabilities{Logging: &LoggingCapabilities{}}  // 总是有 logging
    if s.tools.len() > 0 {
        caps.Tools = &ToolCapabilities{ListChanged: true}
    }
    if s.prompts.len() > 0 {
        caps.Prompts = &PromptCapabilities{ListChanged: true}
    }
    if s.resources.len() > 0 || s.resourceTemplates.len() > 0 {
        caps.Resources = &ResourceCapabilities{ListChanged: true}
        if s.opts.SubscribeHandler != nil {
            caps.Resources.Subscribe = true
        }
    }
    if s.opts.CompletionHandler != nil {
        caps.Completions = &CompletionCapabilities{}
    }
    return caps
}
```

Dart 的 `analyzer.dart:79-110` 反着用：客户端不支持 `roots` → 不注册分析工具，capability 自动不出现。

**对 xray 的启示**：

`xmcp_protocol.c` 的 `build_capabilities` 改成根据 `XmcpRegistry` 的实际内容生成；移除 `XmcpServer.has_tools/has_resources/has_prompts` 这三个手维护的 bool。

### 2.4 Cancellation —— Preempter 模式

Go SDK `mcp/transport.go:200-218`：

```go
type canceller struct{ conn *jsonrpc2.Connection }

func (c *canceller) Preempt(ctx, req) (any, error) {
    if req.Method == notificationCancelled {
        var p CancelledParams
        json.Unmarshal(req.Params, &p)
        id, _ := jsonrpc2.MakeID(p.RequestID)
        go c.conn.Cancel(id)  // 触发对应 request 的 ctx.Done()
    }
    return nil, jsonrpc2.ErrNotHandled
}
```

设计核心：cancel notification **抢占式**处理（preempt），不入正常请求队列；通过 ctx cancellation 传递给正在执行的 handler。

**spec 2025-11-25** 明确（`docs/specification/2025-11-25/basic/utilities/cancellation.mdx`）：cancellation 是 fire-and-forget notification，receiver SHOULD stop processing 但 MAY ignore 已完成的。

**对 xray 的启示**（C 没有 ctx，方案如下）：

1. 注册表里每个正在执行的 request 维护 `volatile sig_atomic_t cancelled` flag。
2. tools/call dispatch 时把 flag 指针挂到 thread-local。
3. handler 可周期性 `if (xmcp_check_cancelled()) return`。
4. `xray_run` 接 isolate 的 instruction-count hook，每 10000 条指令检查一次（已有机制）。
5. Cancel notification 走快速路径：不入队列，直接 lookup pending request 表，置 flag。

### 2.5 Worker pool —— 主循环不阻塞

mcp-go `server/stdio.go:443-473`：tool calls 进 channel，由 worker pool 异步执行；主循环只读消息、分发、不等待。带 panic recovery 防止单个 handler 崩溃整个 server。

**对 xray 的启示**：

xray 已经有协程系统（`src/coro/`），**直接复用**：
- 主循环（在系统 worker 上）只做 read + dispatch。
- 每个 tool/call 包装成一个 xray 协程提交到 worker。
- 写响应走单一 mutex（stdio 没有 fd-level 并发）。
- 这彻底解决"`xray_run` 卡死期间无法收 cancel"。

### 2.6 Lazy resource lifecycle

Dart `mixins/analyzer.dart:40-69`：

```dart
Process? _liveAnalysisServer;
Timer? _lspInactivityTimer;
static Duration lspInactivityDuration = const Duration(minutes: 10);
int _activeLspRequests = 0;
```

LSP 子进程**第一次 tool 调用时才启动**；10 分钟无活动自动关闭；活跃请求计数防止误关。

**对 xray 的启示**：

xray 不需要外部 LSP 子进程（已 in-process），但同样思路适用于：
- `xmcp_server` 的分析 isolate：lazy create，每次 `xray_check`/`xray_diagnostics` 后 reset 状态（避免符号表累积）。
- 知识库：lazy load（启动延迟更小）。

### 2.7 Toolset 分组（GitHub MCP 的扩展性策略）

`@/Users/xuxinglei/workspace/参考代码/mcp/github/github-mcp-server/pkg/github/`：30+ 个 `.go` 文件，按功能域分（issues / pull_requests / actions / code_scanning / discussions / gists / git / copilot ...）。

CLI 选项 `--toolsets repos,issues,pull_requests` 启用一组。`--read-only` filter 掉所有 write 工具。`Default: true` 标记决定 `--toolsets default` 默认启用哪些。

**对 xray 的启示**：

将 7 个工具按主题分 3-4 组：

| Toolset | 工具 | 默认 |
|--|--|--|
| `core` | `xray_check` / `xray_diagnostics` / `xray_format` | ✅ |
| `knowledge` | `xray_syntax_lookup` / `xray_stdlib_search` / `xray_definition` | ✅ |
| `runner` | `xray_run` | ⚠️ 需 `--enable-runner` |
| `lsp_bridge`（未来）| `xray_completion` / `xray_hover` / `xray_references` / `xray_rename` | ❌ |

CLI：`xray mcp-server --toolsets core,knowledge --read-only`。

### 2.8 Schema 验证 —— 缓存编译

mcp-go `server/input_validation.go:33-110`：JSON Schema **编译一次缓存**，用 `(toolName, schemaDigest)` 双键，schema 改了自动失效。即使有 100 次 tools/call 也只编译一次。

**对 xray 的启示**：

- xray 的工具 schema 多数是简单 string/int/bool，最严格只到 `required` + `type`。
- 不引入完整 JSON Schema 验证器（依赖太重）；自己写**轻量验证器**（300 行 C 内），只支持：
  - `type: string|integer|number|boolean|array|object`
  - `required: [...]`
  - `enum: [...]`（用于 syntax topic 等）
  - `minLength` / `maxLength`（字符串）
  - `minimum` / `maximum`（数字）
- 编译期把 schema 直接编入 `XmcpToolDef.input_schema_json`（静态字符串），运行时只跑验证器。

### 2.9 父进程监控 + 优雅关闭

mcp-language-server `main.go:148-168`：

```go
go func() {
    ppid := os.Getppid()
    ticker := time.NewTicker(100 * time.Millisecond)
    for {
        currentPpid := os.Getppid()
        if currentPpid != ppid && (currentPpid == 1 || ppid == 1) {
            // 父进程死了（被 init/launchd 接管），触发 cleanup
        }
    }
}()
```

注释明确写："Claude desktop does not properly kill child processes for MCP servers"。这是踩坑总结的实战代码。

`cleanup()` 用 `context.WithTimeout(5s)` 包装关闭，避免无限挂起。

**对 xray 的启示**：

新建 `xmcp_lifecycle.c`：
- SIGTERM/SIGINT/SIGHUP/SIGPIPE 处理（signal.h）。
- 100ms 一次的 `getppid()` 轮询（Windows 用 `OpenProcess(PROCESS_QUERY_INFORMATION, FALSE, ppid)` + `GetExitCodeProcess`）。
- 收到任一 shutdown 信号 → 设置 `volatile sig_atomic_t shutdown = 1` → 主循环退出。
- 5s timeout 包装 isolate 销毁。

### 2.10 Mixin / feature 配置

Dart `pkgs/dart_mcp_server/lib/src/features_configuration.dart`：CLI 命令行参数决定哪些 mixin 启用，启用后才注册对应 tools。

**对 xray 的启示**：

CLI 选项设计：

```
xray mcp-server [options]

  Tool selection:
    --toolsets <ids>      启用的工具集，逗号分隔 (default: "core,knowledge")
                          可用: core, knowledge, runner, all
    --enable-runner       等价 --toolsets += "runner"，单独开关因为有副作用
    --read-only           过滤掉所有 write/destructive 工具
    --list-toolsets       打印所有 toolset 描述并退出

  Runner sandbox (仅当 runner 启用):
    --runner-timeout <ms>     超时（默认 5000，max 30000）
    --runner-output-max <kb>  输出截断（默认 8）
    --runner-modules <list>   stdlib 白名单（默认 math,json,time）
    --runner-deny-net         禁用 net/http/ws（默认禁）

  Logging:
    --log-level <level>   error/warn/info/debug (默认 info)
    --log-file <path>     额外写入文件

  Protocol:
    --protocol-version <ver>  强制使用某协议版本（调试用）
```

---

## 3. 目标架构

### 3.1 文件清单与职责

```
src/app/mcp/
├── xmcp.h                    # 顶层公共 API（仅 cmd_mcp_server 暴露给 CLI）
│
├── xmcp_server.c/h           # 主循环 + 生命周期
├── xmcp_lifecycle.c/h        # 信号 / 父进程监控 / 优雅关闭
├── xmcp_transport_stdio.c/h  # NDJSON 读写（替代 xr_frame_*）
│
├── xmcp_protocol.c/h         # initialize / 版本协商 / capability 推导
├── xmcp_dispatch.c/h         # JSON-RPC 方法分发表
├── xmcp_session.c/h          # pending requests 表 / cancellation flag
│
├── xmcp_registry.c/h         # XmcpToolDef / XmcpResourceDef / XmcpPromptDef 注册表
├── xmcp_schema.c/h           # 轻量 JSON Schema 验证器（300 行内）
├── xmcp_pagination.c/h       # cursor-based 分页（list 系列方法共用）
│
├── xmcp_notifications.c/h    # log / progress / list_changed 通知发送
├── xmcp_completion.c/h       # completions/complete 实现（topic 名/模块名补全）
│
├── xmcp_tools_core.c         # toolset "core": xray_check / xray_diagnostics / xray_format
├── xmcp_tools_knowledge.c    # toolset "knowledge": xray_syntax_lookup / xray_stdlib_search / xray_definition
├── xmcp_tools_runner.c       # toolset "runner": xray_run（带沙盒）
├── xmcp_tools_lsp.c          # toolset "lsp_bridge"（预留）：xray_completion / xray_hover / ...
│
├── xmcp_resources.c          # resources/list / read / templates
├── xmcp_prompts.c            # prompts/list / get
│
├── xmcp_knowledge.c/h        # 知识库（重写：从源码自动提取）
├── xmcp_knowledge_loader.c   # 启动时扫描 stdlib/ 和 docs/rules/ 生成索引
└── xmcp_knowledge_data.h     # configure_file 生成（从 docs/rules/language-spec.md 等）
```

文件数从 6 增加到 ~18，但每个 .c 都在 200-500 行（远低于 c-coding-standards 3000 行上限），职责清晰。

### 3.2 依赖图

```
   xcli (cmd_mcp_server)
        │
        ▼
    xmcp_server   ◄────── xmcp_lifecycle (信号 / ppid)
        │
   ┌────┼─────────┬─────────┬─────────┐
   ▼    ▼         ▼         ▼         ▼
xmcp_  xmcp_   xmcp_     xmcp_     xmcp_
trans  dispa   proto     sessi     notif
port_  tch     col       on        ication
stdio                              s
        │
        ▼
   xmcp_registry  ◄─────  xmcp_schema
        │              ◄─────  xmcp_pagination
   ┌────┴────┬───────┬────────┐
   ▼         ▼       ▼        ▼
tools_*   resources prompts completion
   │         │       │        │
   └─────────┴───────┴────────┘
              │
              ▼
        xmcp_knowledge ◄──── xmcp_knowledge_loader
              │              ◄──── xmcp_knowledge_data.h
              ▼
        (复用) xray frontend / api / lsp
```

**关键约束**：
- `xmcp_*` 单向依赖 `src/api`、`src/frontend`、`src/app/lsp`，反向不存在。
- `xmcp_transport_stdio` 不依赖任何 mcp 内部，便于单元测试。
- `xmcp_tools_runner` 单独 link `src/api`（执行需要完整 isolate），其他 tools 只需要 analyzer/parser。

### 3.3 核心数据结构

```c
/* xmcp_registry.h */

typedef XrJsonValue *(*XmcpToolHandler)(XmcpServer *s, XrJsonValue *args, XmcpRequestCtx *rctx);
typedef XrJsonValue *(*XmcpSchemaBuilder)(void);  /* 按需构建 schema */

typedef struct XmcpToolDef {
    const char        *name;
    const char        *title;
    const char        *description;
    const char        *toolset;          /* e.g. "core" */
    bool               read_only;
    bool               open_world;
    bool               default_enabled;
    bool               needs_runtime;    /* true => 需要完整 isolate (runner) */
    XmcpSchemaBuilder  build_input_schema;
    XmcpSchemaBuilder  build_output_schema;  /* 可空 */
    XmcpToolHandler    handler;
} XmcpToolDef;

typedef struct XmcpToolset {
    const char         *id;
    const char         *description;
    bool                default_enabled;
    const XmcpToolDef  *tools;
    size_t              tool_count;
} XmcpToolset;

/* 每个 tools_*.c 暴露一个 toolset */
extern const XmcpToolset XMCP_TOOLSET_CORE;
extern const XmcpToolset XMCP_TOOLSET_KNOWLEDGE;
extern const XmcpToolset XMCP_TOOLSET_RUNNER;

/* 运行时注册表 */
typedef struct XmcpRegistry {
    const XmcpToolDef    **tools;
    size_t                 tool_count;
    /* 排序后的 name -> idx，二分查找 + cursor */
    /* 类似的 resources / templates / prompts */
} XmcpRegistry;

XR_FUNC void xmcp_registry_init(XmcpRegistry *r);
XR_FUNC void xmcp_registry_add_toolset(XmcpRegistry *r, const XmcpToolset *ts, const XmcpServerOptions *opts);
XR_FUNC const XmcpToolDef *xmcp_registry_find_tool(XmcpRegistry *r, const char *name);
XR_FUNC void xmcp_registry_free(XmcpRegistry *r);
```

```c
/* xmcp_session.h —— 请求生命周期跟踪 */

typedef struct XmcpRequestCtx {
    XrJsonValue            *id;               /* JSON-RPC id（NULL = notification）*/
    int64_t                 progress_token;   /* 客户端要求进度跟踪时非 -1 */
    volatile sig_atomic_t   cancelled;
    /* 用于 progress / log 通知 */
    XmcpServer             *server;
} XmcpRequestCtx;

typedef struct XmcpSession {
    /* pending requests: id -> XmcpRequestCtx*，cancellation lookup 用 */
    /* hash map（src/base/xhash.h），ID 用 jsonrpc id 字符串化 */
    XrMap *pending;
    /* 协议状态 */
    bool   initialized;
    char  *protocol_version;  /* 协商后 */
    /* 客户端 capabilities 副本（决定哪些 server features 可用）*/
    XrJsonValue *client_caps;
} XmcpSession;
```

```c
/* xmcp_schema.h —— 轻量验证器 */

typedef enum {
    XMCP_SCHEMA_OK = 0,
    XMCP_SCHEMA_REQUIRED_MISSING,
    XMCP_SCHEMA_TYPE_MISMATCH,
    XMCP_SCHEMA_OUT_OF_RANGE,
    XMCP_SCHEMA_ENUM_MISMATCH,
} XmcpSchemaResult;

XR_FUNC XmcpSchemaResult xmcp_schema_validate(
    const char *schema_json,   /* 静态 schema（编译期嵌入） */
    XrJsonValue *value,
    char *err_buf, size_t err_buf_len
);
```

### 3.4 与 xray 已有模块的关系

| MCP 工具 | 复用 xray 模块 | 共用接口 |
|---|---|---|
| `xray_check` / `xray_diagnostics` | `src/frontend/parser/xparse.h` | `xr_parser_init` + `xr_parse_recoverable` |
| `xray_format` | `src/frontend/format/xfmt.h` | `xfmt_format_ast` |
| `xray_run` | `src/api/xray_isolate.h` | `xray_isolate_dostring` + 沙盒选项 |
| `xray_syntax_lookup` / `xray_definition` | `src/app/lsp/xlsp_*` | 共享 hover/symbol 元数据 |
| `xray_stdlib_search` | `stdlib/` 目录 + `src/app/lsp/xlsp_workspace_index` | 索引复用 |
| `xray_completion` / `xray_hover`（未来）| `src/app/lsp/xlsp_analysis.c` | 直接调 LSP 内部函数 |

**关键设计**：`xmcp_tools_lsp.c` 不通过 stdio 启动 LSP 子进程，而是直接 include `xlsp_analysis.h`，**单进程内调用**——这是相对 mcp-language-server 的天然优势。

---

## 4. 关键设计决策

### 4.1 协议版本：默认 `2025-11-25`，向下兼容到 `2024-11-05`

**借鉴**：Go SDK `mcp/shared.go:38-65`。

```c
/* xmcp_protocol.c */
static const char *XMCP_SUPPORTED_VERSIONS[] = {
    "2025-11-25",  /* latest */
    "2025-06-18",
    "2025-03-26",
    "2024-11-05",
    NULL
};
#define XMCP_LATEST_VERSION "2025-11-25"

/* 客户端发的版本若在表里则用之，否则用 latest */
const char *xmcp_negotiate_version(const char *client_version) {
    if (!client_version) return XMCP_LATEST_VERSION;
    for (int i = 0; XMCP_SUPPORTED_VERSIONS[i]; i++) {
        if (strcmp(client_version, XMCP_SUPPORTED_VERSIONS[i]) == 0)
            return XMCP_SUPPORTED_VERSIONS[i];
    }
    return XMCP_LATEST_VERSION;
}
```

**注意**：spec 要求"返回客户端能接受的版本"——但实践上 Go SDK / TS SDK 都是"无脑 latest"，除非客户端断连重试。我们跟主流走。

### 4.2 stdio framing：NDJSON

**借鉴**：TS SDK `ReadBuffer`（最简洁）+ Go SDK `ioConn`（容错）。

伪代码：

```c
/* xmcp_transport_stdio.c */

typedef struct {
    char  *buf;
    size_t cap;
    size_t len;
} XmcpReadBuffer;

void xmcp_rb_append(XmcpReadBuffer *rb, const char *data, size_t n);

/* Returns:
 *   1 on full message extracted (out_msg/out_len 指向 buf 内部，caller dup)
 *   0 partial（需要更多数据）
 *  -1 error */
int xmcp_rb_pop_message(XmcpReadBuffer *rb, const char **out_msg, size_t *out_len);

/* 写一条消息：json + '\n'，单 mutex 保护 */
void xmcp_write_ndjson(const char *json, size_t len);
```

`xmcp_rb_pop_message` 内部：
1. `memchr(buf, '\n', len)` 找换行。
2. 取出该行（去掉末尾 `\r`）。
3. **首字符不是 `{` 也不是 `[`** → 跳过该行（兼容 hot-reload 工具的 stdout 干扰）。
4. 调用方 `xjson_parse` 实际反序列化。
5. 把已消费部分从 buf 里 `memmove` 移除。

### 4.3 Cancellation：抢占式 + 协作式

**借鉴**：Go SDK preempter + spec 2025-11-25 semantics。

```c
/* xmcp_dispatch.c */

void xmcp_dispatch(XmcpServer *s, XrJsonValue *msg) {
    const char *method = xjson_get_string(msg, "method");

    /* 抢占式：cancellation 不入队，直接处理 */
    if (strcmp(method, "notifications/cancelled") == 0) {
        XrJsonValue *params = xjson_get(msg, "params");
        XrJsonValue *id = xjson_get(params, "requestId");
        XmcpRequestCtx *rctx = xmcp_session_lookup(s->session, id);
        if (rctx) {
            rctx->cancelled = 1;  /* 协程会在下一 instr-count check 看到 */
        }
        return;
    }

    /* 其他正常分发 */
    ...
}
```

handler 端（`xray_run`）：

```c
/* 在 isolate 的 instruction hook 里检查 */
static void runner_inst_hook(XrIsolate *iso, void *ud) {
    XmcpRequestCtx *rctx = (XmcpRequestCtx *)ud;
    if (rctx->cancelled) {
        xray_isolate_request_abort(iso, "MCP cancelled");
    }
}
```

### 4.4 输入 schema 验证：轻量内嵌

**借鉴**：mcp-go `compileSchema` 缓存思路 + 自研轻量验证器（不引入 jsonschema lib）。

工具 schema 编译期就是一段静态 JSON 字符串：

```c
/* xmcp_tools_core.c */
static const char SCHEMA_CHECK[] =
    "{\"type\":\"object\","
    "\"properties\":{\"code\":{\"type\":\"string\",\"minLength\":1}},"
    "\"required\":[\"code\"]}";

static XrJsonValue *schema_check(void) {
    return xjson_parse(SCHEMA_CHECK, sizeof(SCHEMA_CHECK)-1);
}
```

dispatch 在 handler 之前先验证：

```c
XrJsonValue *xmcp_dispatch_tools_call(...) {
    const XmcpToolDef *def = xmcp_registry_find_tool(reg, name);
    if (!def) return xmcp_error(...);

    /* 用静态 schema string 编译验证 */
    char err[256];
    if (xmcp_schema_validate(def->input_schema_json, args, err, sizeof(err)) != XMCP_SCHEMA_OK) {
        /* 注意：spec 2025-11-25 SEP-1303 明确：input validation errors 应作为
         * Tool Execution Error 返回（isError: true），而不是 Protocol Error，
         * 让 model 能 self-correct */
        return xmcp_make_error_result(err);
    }

    return def->handler(s, args, rctx);
}
```

**关键合规点**：参数验证错误必须用 `isError: true` 的 CallToolResult，不是 JSON-RPC error code。这是 2025-11-25 才明确的细节。

### 4.5 Worker 分流：复用 xray 协程

**借鉴**：mcp-go worker pool。

xray 已有协程系统，主循环走系统 worker，每个请求开一个用户协程：

```c
void xmcp_server_run(XmcpServer *s) {
    while (!s->shutdown) {
        /* 主线程读 stdin */
        const char *msg = xmcp_rb_read_one(s->rb, s->stdin_fd, &s->shutdown);
        if (!msg) break;

        /* 分发：notification 直接处理；request 投递到协程 */
        XrJsonValue *json = xjson_parse(msg);
        if (xmcp_is_notification(json)) {
            xmcp_dispatch(s, json);  /* 同步 */
        } else {
            xr_coro_spawn(s->isolate, dispatch_request_coro, json);
        }
    }
}
```

**好处**：
- 长 tool 不阻塞读取。
- Cancel notification 立刻被读到。
- xray 自家协程，零额外线程库依赖。

**写时序**：`xmcp_write_ndjson` 内部用 `pthread_mutex` / `mtx_t`，所有协程都通过它写。

### 4.6 父进程监控

**借鉴**：mcp-language-server `main.go:148-168`。

```c
/* xmcp_lifecycle.c */
#include <unistd.h>

static void *parent_watchdog(void *ud) {
    XmcpServer *s = ud;
    pid_t orig_ppid = getppid();
    while (!s->shutdown) {
        usleep(100 * 1000);  /* 100ms */
        pid_t now = getppid();
        if (now != orig_ppid && (now == 1 || orig_ppid == 1)) {
            /* 父进程死了，被 init 接管 */
            s->shutdown = 1;
            return NULL;
        }
    }
    return NULL;
}
```

Windows：`OpenProcess(SYNCHRONIZE, FALSE, ppid)` 拿 handle，`WaitForSingleObject(h, 0)` 检查；超时 0 立刻返回。

### 4.7 知识库自治

**借鉴**：Dart MCP 的"分析器协议直连"思路（不靠手写文本）。

`xmcp_knowledge_loader.c` 启动时执行：

1. 扫描 `stdlib/*/` 目录，从每个模块的 `*.xr` 导出语句和 `@doc` 注解抽取 API 列表 → 生成 `XmcpStdlibIndex`。
2. 解析 `docs/rules/language-spec.md`（已存在，是 xray 语言规范的真值源），按 markdown heading 切分主题 → 生成 `XmcpTopicIndex`。
3. 内存里持有索引；`xray_syntax_lookup` / `xray_stdlib_search` / `xray_definition` 查它。

**编译期 fallback**：如果运行时找不到 stdlib/docs（例如 single-binary 部署），通过 CMake `configure_file` 把这两类内容嵌成 `xmcp_knowledge_data.h` 静态字符串，运行时优先文件系统、退回内嵌。

### 4.8 Toolset 与 CLI

CLI 选项见 2.10。实现要点：

```c
/* xmcp_server.c */
XR_FUNC int cmd_mcp_server(const XrCliInvocation *inv) {
    XmcpServerOptions opts;
    xmcp_options_parse_cli(&opts, inv);  /* 填 toolsets / read_only / runner_* / log_* */

    XmcpServer *s = xmcp_server_new(&opts);
    if (!s) return XR_CLI_EXIT_INTERNAL;

    /* 按 opts.toolsets 注册 */
    if (opts.has_toolset("core"))      xmcp_registry_add_toolset(&s->reg, &XMCP_TOOLSET_CORE, &opts);
    if (opts.has_toolset("knowledge")) xmcp_registry_add_toolset(&s->reg, &XMCP_TOOLSET_KNOWLEDGE, &opts);
    if (opts.has_toolset("runner"))    xmcp_registry_add_toolset(&s->reg, &XMCP_TOOLSET_RUNNER, &opts);

    if (opts.read_only) xmcp_registry_filter_read_only(&s->reg);

    int rc = xmcp_server_run(s);
    xmcp_server_free(s);
    return rc == 0 ? XR_CLI_EXIT_OK : XR_CLI_EXIT_FAIL;
}
```

### 4.9 `xray_run` 沙盒

**借鉴**：参考实现里没有现成（runner 是 xray 独有），但综合 isolate 现有能力：

```c
/* xmcp_tools_runner.c */
static XrJsonValue *tool_xray_run(XmcpServer *s, XrJsonValue *args, XmcpRequestCtx *rctx) {
    const char *code = xjson_get_string(args, "code");
    int64_t timeout_ms = xjson_get_int_or(args, "timeout", s->opts.runner_timeout);
    if (timeout_ms > s->opts.runner_timeout_max) timeout_ms = s->opts.runner_timeout_max;

    XrayIsolateParams p;
    xray_isolate_params_init(&p);
    xray_isolate_setup_full(&p);

    /* 沙盒：屏蔽危险模块 */
    xray_isolate_params_set_module_whitelist(&p, s->opts.runner_modules);
    if (s->opts.runner_deny_net) {
        xray_isolate_params_disable_module(&p, "net");
        xray_isolate_params_disable_module(&p, "http");
        xray_isolate_params_disable_module(&p, "ws");
    }

    /* 时间限制：注入 instruction hook */
    int64_t deadline_ns = xr_time_ns() + timeout_ms * 1000000LL;
    XrayIsolate *iso = xray_isolate_new(&p);
    xray_isolate_set_inst_hook(iso, runner_check_deadline_or_cancel,
                               &(RunnerHookCtx){deadline_ns, rctx});

    /* 输出捕获：用内存管道，不用 dup2 + tmpfile（Windows 不友好） */
    XrIoCapture cap;
    xr_io_capture_begin(&cap, s->opts.runner_output_max);

    int rc = xray_isolate_dostring(iso, code);

    char *out = xr_io_capture_end(&cap);
    bool truncated = xr_io_capture_was_truncated(&cap);

    xray_isolate_delete(iso);

    /* 构造 result */
    char *text = format_runner_output(out, rc, truncated);
    XrJsonValue *result = xmcp_make_text_result(text, rc != 0);
    xr_free(text); xr_free(out);
    return result;
}
```

**关键约束**：
- `runner_timeout_max = 30000ms`（硬上限）。
- `runner_output_max = 64KB`（硬上限）。
- 模块白名单默认仅 `math, json, time, regex` 等纯计算类。
- `--enable-runner` 显式开启（默认关），符合"安全第一"。

### 4.10 Pagination

**借鉴**：Go SDK `featureSet.above(uid)`。

```c
/* xmcp_pagination.c */
typedef struct {
    const char *cursor;       /* 客户端传进来的 */
    int         page_size;    /* 默认 1000 */
    /* 输出 */
    const char *next_cursor;  /* NULL 表示 last page */
} XmcpPaginator;

/* tools/list */
XrJsonValue *xmcp_list_tools(XmcpRegistry *r, XrJsonValue *params) {
    const char *cursor = params ? xjson_get_string(params, "cursor") : NULL;
    XrJsonValue *result = xjson_new_object();
    XrJsonValue *arr = xjson_new_array();

    int emitted = 0;
    for (size_t i = 0; i < r->tool_count; i++) {
        const XmcpToolDef *t = r->tools[i];  /* 已按 name 排序 */
        if (cursor && strcmp(t->name, cursor) <= 0) continue;
        xjson_array_push(arr, build_tool_descriptor(t));
        emitted++;
        if (emitted >= 1000) {
            XJSON_SET_STRING(result, "nextCursor", t->name);
            break;
        }
    }
    xjson_object_set(result, "tools", arr);
    return result;
}
```

resources/templates/prompts list 共用同一套 paginator。

### 4.11 Completions

新功能。**借鉴**：spec `docs/specification/2025-11-25/server/utilities/completion.mdx`。

实现要点：
- 客户端发 `completion/complete`，参数含 `ref: { type: "ref/resource", uri }` 或 `ref: { type: "ref/prompt", name }` + `argument: { name, value }`。
- 我们对 `xray://spec/topic/{name}` 模板的 `{name}` 提供候选：从 `XmcpTopicIndex` 拿所有 topic 名字，按 `value` 前缀过滤。
- `xray://stdlib/{module}` 同理。
- 对 prompt 参数（如 `convert-to-xray.sourceLanguage`）也可提供候选（`Python, JavaScript, Go, Rust, ...`）。

`XmcpServerCapabilities.completions = {}` 此时才声明。

### 4.12 Logging

**借鉴**：spec `2025-11-25/server/utilities/logging.mdx`。

`logging/setLevel` 请求把 server 的 `log_level` 调整到客户端要求的级别。所有 log 通过 `xmcp_send_log_notification` 发到客户端（用 `notifications/message` method）。

### 4.13 Resources subscribe（可选）

不实现 Phase 1。如果要做：
- 客户端发 `resources/subscribe { uri }`。
- server 为该 uri 维护订阅列表。
- 资源底层数据变化时（例如 stdlib 文件 mtime 变了，或 reload 知识库）发 `notifications/resources/updated`。
- capability 增加 `resources.subscribe = true`。

---

## 5. 实施计划

按"先连得上、再加功能、最后优化"。每个阶段独立可发布、独立可回滚。

### 阶段一：连通性（**必须做，1-2 天**）

**目标**：xray MCP 能被 Claude Desktop / Cursor / Inspector 识别并 list tools 成功。

| 任务 | 文件 | 工作量 |
|--|--|--|
| 新建 `xmcp_transport_stdio.c/h`，实现 NDJSON 读写 | 新建 | 半天 |
| `xmcp_server.c` 切换到新传输；删除 `xr_frame_*` 依赖 | 修改 | 2h |
| 协议版本协商升到 `2025-11-25`，新增版本表 | `xmcp_protocol.c` | 1h |
| 修 `xmcp_write_message` 错误处理（write 失败 → log + shutdown） | 同上 | 1h |
| 跑 `npx @modelcontextprotocol/inspector xray mcp-server` 验证 | 测试 | 半天 |

**验收**：
- Inspector 能成功 initialize 并显示 7 tools 列表。
- Claude Desktop 配置后能调 `xray_check`。

### 阶段二：架构骨架（3-5 天）

**目标**：注册表 + dispatch 重构，capability 自动推导，中型重构。

| 任务 | 文件 |
|--|--|
| 抽 `xmcp_registry.c/h`：`XmcpToolDef` / `XmcpToolset` / 增删查改 / 排序 | 新建 |
| 抽 `xmcp_dispatch.c/h`：表驱动方法分发，从 `xmcp_server.c` 拆出 | 新建 |
| 抽 `xmcp_session.c/h`：pending requests 表、cancellation flag | 新建 |
| `xmcp_protocol.c` `build_capabilities` 改为读 registry，删除手维护 bool | 修改 |
| `xmcp_pagination.c/h`：cursor 工具 | 新建 |
| 现有 7 个 tools 迁移到 `xmcp_tools_core.c` / `xmcp_tools_knowledge.c` / `xmcp_tools_runner.c` | 重组 |
| `notifications/cancelled` 改成抢占式（直接置 flag） | `xmcp_dispatch.c` |
| 单测：`tests/unit/mcp/test_mcp_registry.c` + `test_mcp_dispatch.c` | 新建 |

**验收**：
- ctest 全过；`tests/unit/mcp/` 数从 1 增到 4-5 个。
- 添加新工具的步骤：写一个 handler + 一个 schema builder + 一行 `XmcpToolDef` 进表。

### 阶段三：合规与健壮性（2-3 天）

**目标**：堵住所有 015 列出的 P0/P1 问题。

| 任务 | 文件 |
|--|--|
| Schema 验证器（轻量版） | `xmcp_schema.c/h` 新建 |
| 所有 tool dispatch 走 schema 验证 → 失败返回 `isError: true`（spec SEP-1303） | `xmcp_dispatch.c` |
| `logging/setLevel` 请求实现 | `xmcp_dispatch.c` + `xmcp_protocol.c` |
| 父进程监控线程 | `xmcp_lifecycle.c` 新建 |
| `xmcp_write_message` 加 mutex + 错误传播 | `xmcp_transport_stdio.c` |
| `prompts/get` 修 role（合并 system+user 为单条 user） | `xmcp_prompts.c` |
| 每次 `xray_check` / `xray_diagnostics` 用独立 arena + reset isolate（避免污染） | `xmcp_tools_core.c` |

**验收**：
- 015 的 P0/P1 项全部关闭。
- 协议合规测试：用脚本喂错误参数、错误版本、错误方法名，验证响应。

### 阶段四：协程化与沙盒（3-4 天）

**目标**：长 tool 不阻塞主循环；`xray_run` 安全可中断。

| 任务 | 文件 |
|--|--|
| 主循环走 xray 协程：每个 request 投递给协程 | `xmcp_server.c` |
| Worker 写时序：`xmcp_write_ndjson` 加 mutex | `xmcp_transport_stdio.c` |
| `xray_run` 沙盒：timeout / 输出截断 / 模块白名单 / cancel flag check | `xmcp_tools_runner.c` 重写 |
| isolate inst hook + 内存管道捕获（替代 `dup2 + tmpfile`） | 同上 |
| Progress notification 真实接入（解析 + 发送） | `xmcp_notifications.c` |

**验收**：
- 跑 `while (true) {}` 的 `xray_run` 能在 timeout 后被中断并返回 `[exit code: -1]`；如客户端发 cancel 也能立即停。
- `xray_run` 期间客户端能发 `tools/list` 并立即收到响应。

### 阶段五：知识库自治（2-3 天）

**目标**：删除 889 行手写知识库字符串，从源码自动生成。

| 任务 | 文件 |
|--|--|
| `xmcp_knowledge_loader.c`：扫描 `stdlib/` 目录，提取每个模块的导出和 `@doc` | 新建 |
| 解析 `docs/rules/language-spec.md`，按 heading 切主题 | 同上 |
| CMake `configure_file` 嵌入两份索引作为 fallback（部署到 single-binary 时用） | `CMakeLists.txt` + `xmcp_knowledge_data.h.in` |
| `xmcp_knowledge.c` 改为运行时优先文件系统、回退到嵌入数据 | 重构 |
| `xmcp_knowledge.c` 老的硬编码字符串删除（`TOPIC_*`、`STDLIB_LIST`、`CHEATSHEET`、`CONCURRENCY_MODEL`） | 删除 |

**验收**：
- 加一个 stdlib 模块（如 `cluster`）后，无需改 mcp 代码，`xray_stdlib_search "cluster"` 自动有结果。
- `xmcp_knowledge.c` 行数从 889 降到 ~200。

### 阶段六：扩展能力（2-3 天，可分次发布）

| 任务 | 文件 |
|--|--|
| `completion/complete` 实现（topic / module / 部分 prompt 参数补全） | `xmcp_completion.c` 新建 |
| LSP bridge toolset：复用 `xlsp_analysis.h`，提供 `xray_completion` / `xray_hover` / `xray_references` | `xmcp_tools_lsp.c` 新建 |
| `--toolsets` / `--read-only` / `--enable-runner` 等 CLI 选项 | `xcli_spec.c` + `xmcp_server.c` |
| Resources subscribe（如果决定支持） | `xmcp_dispatch.c` |
| MCP-Apps（icons 等 2025-11-25 新元数据） | 各 `XmcpToolDef` |

**验收**：
- `xray mcp-server --toolsets default` 默认启 core+knowledge。
- `xray mcp-server --toolsets all --enable-runner` 全部启用。
- `xray mcp-server --list-toolsets` 打印 toolset 描述。

---

## 6. 测试策略

### 6.1 单元测试（`tests/unit/mcp/`）

| 文件 | 覆盖 |
|--|--|
| `test_mcp_transport_ndjson.c` | NDJSON 读写：partial / multiple-in-one / 容错非 JSON 行 / `\r\n` |
| `test_mcp_registry.c` | 注册 / 查找 / 排序 / read-only filter / toolset filter |
| `test_mcp_schema.c` | 验证器：required / type / enum / range / 嵌套 object |
| `test_mcp_dispatch.c` | 方法路由 / cancellation / unknown method / pre-init guard |
| `test_mcp_session.c` | pending requests / id 匹配 / 并发 cancel |
| `test_mcp_pagination.c` | cursor 边界 / 空列表 / 单元素 |
| `test_mcp_protocol.c`（已有）| 版本协商 / capability 推导 |

每个 `< 300` 行，加起来约 1500-2000 行测试代码。

### 6.2 协议一致性测试（`tests/integration/mcp/`）

**借鉴**：Go SDK `mcp/conformance_test.go` 的 txtar 风格。

新建 `tests/integration/mcp/conformance/`，每个 `.txt` 文件描述一个完整会话：

```
-- request --
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}
-- response --
{"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2025-11-25",...}}
```

`tests/integration/mcp/test_conformance.c` 启动一个 in-process MCP server（用 `XmcpServer` 直接喂 NDJSON，不 fork 子进程），按 .txt 顺序发请求 / 验响应。

### 6.3 端到端（`scripts/test_mcp_e2e.sh`）

```bash
#!/bin/bash
# 用 inspector 跑一遍：list tools / call check / call run
npx @modelcontextprotocol/inspector --cli build/xray mcp-server \
    --tools \
    --call xray_check '{"code": "let x = 1"}' \
    | grep -q '"isError":false'
```

### 6.4 真实客户端 smoke

手工：Claude Desktop / Cursor / Windsurf 各连一次，跑 `tools/list` + 一次 `xray_check`。每次发版前过一遍。

---

## 7. 风险与开放问题

### 7.1 兼容性

- **xray 老用户**：MCP 模块上线时间不长，破坏性改动可接受；CHANGELOG 里说明 framing 变化。
- **CLI 选项命名**：`--toolsets` / `--enable-runner` 是新增，不破坏现有 `xray mcp-server` 用法。
- **TOML 配置**：未来可在 `xray.toml` 里写 `[mcp.toolsets] = ["core", "knowledge"]`，但本次不做。

### 7.2 性能

- xray MCP 是开发工具，QPS 极低，性能不是优化重点。
- 唯一关注：`xray_run` 用完整 isolate，启动开销 ~50ms。如果客户端高频调用（不太会），可考虑 isolate 池。

### 7.3 安全

- `xray_run` 是唯一有副作用的 tool。沙盒方案见 4.9。
- 协议本身（stdio）不需要鉴权，靠操作系统进程隔离。
- 父进程监控防止"客户端崩溃后 server 残留消耗资源"。

### 7.4 开放问题

| 问题 | 建议 |
|--|--|
| Tasks（2025-11-25 实验性）支持否？ | **不支持**。`xray_run` 用 cancel + progress 已足够。 |
| Elicitation 客户端能力检测？ | **能力声明仅作记录**，server 不主动发起。 |
| MCP-Apps icons / sizes 元数据？ | 阶段六加，每个 toolset 选个 emoji 字符当 16x16 SVG。 |
| Resources subscribe？ | 暂缓。除非有人想做"语言规范变化时通知"。 |
| Roots（client capability）？ | 暂不利用。未来 `xray_check` 可基于 client 提供的 roots 做项目级检查。 |
| Logging/setLevel 之后 stderr 还要不要写？ | 保持双写：MCP 通道发 + stderr 也写（被客户端 capture 的话也不会污染 stdout）。 |

### 7.5 工时估算总览

| 阶段 | 工时 | 价值 |
|--|--|--|
| 一、连通性 | 1-2d | 极高（修最严重 bug） |
| 二、架构骨架 | 3-5d | 高（后续所有事的地基） |
| 三、合规与健壮性 | 2-3d | 高 |
| 四、协程化与沙盒 | 3-4d | 中-高 |
| 五、知识库自治 | 2-3d | 中（长期维护成本） |
| 六、扩展能力 | 2-3d | 中（取决于业务需求） |
| **合计** | **13-20d** | |

并行机会有限（阶段二是后面所有阶段的前置）；可独立推进的是阶段五的知识库爬取脚本。

---

## 附录 A：参考实现源码索引

阅读重构方案时如需对照源码：

| 主题 | 入口文件 |
|--|--|
| NDJSON framing | `@/Users/xuxinglei/workspace/参考代码/mcp/modelcontextprotocol/typescript-sdk/packages/core/src/shared/stdio.ts` |
| NDJSON framing (Go) | `@/Users/xuxinglei/workspace/参考代码/mcp/modelcontextprotocol/go-sdk/mcp/transport.go:351-433` |
| NDJSON framing (Dart) | `@/Users/xuxinglei/workspace/参考代码/mcp/dart-lang/ai/pkgs/dart_mcp/lib/stdio.dart` |
| NDJSON vs LSP Content-Length 对比 | `@/Users/xuxinglei/workspace/参考代码/mcp/dart-lang/ai/pkgs/dart_mcp_server/lib/src/lsp/wire_format.dart` |
| Capability 推导 | `@/Users/xuxinglei/workspace/参考代码/mcp/modelcontextprotocol/go-sdk/mcp/server.go:576-624` |
| 版本协商 | `@/Users/xuxinglei/workspace/参考代码/mcp/modelcontextprotocol/go-sdk/mcp/shared.go:32-65` |
| FeatureSet 注册表 | `@/Users/xuxinglei/workspace/参考代码/mcp/modelcontextprotocol/go-sdk/mcp/features.go` |
| Inventory builder | `@/Users/xuxinglei/workspace/参考代码/mcp/github/github-mcp-server/pkg/inventory/builder.go` |
| ServerTool / Toolset 元数据 | `@/Users/xuxinglei/workspace/参考代码/mcp/github/github-mcp-server/pkg/inventory/server_tool.go` |
| RegisterTool ergonomics | `@/Users/xuxinglei/workspace/参考代码/mcp/modelcontextprotocol/typescript-sdk/packages/server/src/server/mcp.ts:867-921` |
| Mixin composition | `@/Users/xuxinglei/workspace/参考代码/mcp/dart-lang/ai/pkgs/dart_mcp_server/lib/src/server.dart:38-61` |
| Cancellation preempter | `@/Users/xuxinglei/workspace/参考代码/mcp/modelcontextprotocol/go-sdk/mcp/transport.go:200-218` |
| Cancellation spec | `@/Users/xuxinglei/workspace/参考代码/mcp/modelcontextprotocol/modelcontextprotocol/docs/specification/2025-11-25/basic/utilities/cancellation.mdx` |
| Worker pool + panic recovery | `@/Users/xuxinglei/workspace/参考代码/mcp/mark3labs/mcp-go/server/stdio.go:443-499` |
| Schema cache | `@/Users/xuxinglei/workspace/参考代码/mcp/mark3labs/mcp-go/server/input_validation.go:33-110` |
| Lazy LSP + idle shutdown | `@/Users/xuxinglei/workspace/参考代码/mcp/dart-lang/ai/pkgs/dart_mcp_server/lib/src/mixins/analyzer.dart:40-160` |
| 父进程监控 | `@/Users/xuxinglei/workspace/参考代码/mcp/isaacphi/mcp-language-server/main.go:148-168` |
| 优雅关闭 | `@/Users/xuxinglei/workspace/参考代码/mcp/isaacphi/mcp-language-server/main.go:193-245` |
| LSP→MCP 工具映射 | `@/Users/xuxinglei/workspace/参考代码/mcp/isaacphi/mcp-language-server/tools.go` |
| Errors 模型 | `@/Users/xuxinglei/workspace/参考代码/mcp/mark3labs/mcp-go/server/errors.go` |
| Transport multi-mode 入口 | `@/Users/xuxinglei/workspace/参考代码/mcp/modelcontextprotocol/servers/src/everything/index.ts` |

## 附录 B：关键 spec 锚点

| 主题 | 文件 |
|--|--|
| stdio transport（NDJSON 强制） | `2025-06-18/basic/transports.mdx#stdio`，line 28 |
| 版本协商 | `2025-11-25/basic/lifecycle.mdx#version-negotiation` |
| Capability 列表 | `2025-11-25/basic/lifecycle.mdx#capability-negotiation` |
| Cancellation 语义 | `2025-11-25/basic/utilities/cancellation.mdx` |
| Logging | `2025-11-25/server/utilities/logging.mdx` |
| Completions | `2025-11-25/server/utilities/completion.mdx` |
| Pagination | `2025-11-25/server/utilities/pagination.mdx` |
| 2025-11-25 changelog | `2025-11-25/changelog.mdx` |

---

**文档版本**：v1（基于 12 个参考实现首版）
**前置任务**：015-mcp-improvement.md
**下一步**：实施阶段一（NDJSON framing）
