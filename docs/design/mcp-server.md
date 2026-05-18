# Xray MCP Server 详细设计方案

> Model Context Protocol (MCP) server，让 AI 助手能正确编写和调试 Xray 代码。

## 1. 设计目标

### 核心问题
Xray 是全新语言，AI 模型训练数据中不包含 Xray。新手使用 AI 编写 `.xr` 代码时，AI 不知道：
- 语法规则（关键字、控制流、并发原语）
- 标准库 API（模块名、函数签名、用法示例）
- 并发安全模型（shared const / Channel / move 语义）
- 编写的代码是否能通过编译

### 解决方案
提供 MCP Server，让 AI 可以**按需查询**语法和 API，**实时验证**代码正确性。

### 设计原则
1. **内置到 CLI** — `xray mcp-server`，学习 Dart 的做法，零额外安装
2. **stdio 传输** — 最大兼容性（Cursor / Claude Code / Windsurf / VS Code Copilot / Gemini CLI）
3. **渐进式** — Phase 1 用 3 个 tool 解决 80% 问题，后续逐步扩展
4. **数据自治** — 所有知识从 xray 源码自动提取，不需要手工维护独立文档

---

## 2. 架构

### 2.1 在 CLI 中的位置

```
xray CLI (src/app/cli/xcli.c)
├── run        → xcmd_run.c
├── build      → xcmd_build.c
├── check      → xcmd_check.c
├── test       → xcmd_test.c
├── fmt        → xcmd_fmt.c
├── lsp        → src/app/lsp/
├── dap        → src/app/dap/
└── mcp-server → src/app/mcp/     ← 新增
```

### 2.2 模块层次

```
L8: src/app/mcp/          ← MCP Server（与 lsp/dap 同级）
    ├── xmcp_server.c/h       — 主循环、stdio 读写、JSON-RPC 分发
    ├── xmcp_protocol.c/h     — MCP 协议编解码（initialize/tools/resources）
    ├── xmcp_tools.c/h        — Tool handler 实现
    ├── xmcp_resources.c/h    — Resource handler 实现
    └── xmcp_knowledge.c/h    — 知识库索引（语法 spec、stdlib API）
```

### 2.3 依赖关系

```
xmcp_server
  ├── xmcp_protocol        — JSON-RPC over stdio
  ├── xmcp_tools
  │   ├── XrayIsolate      — 编译检查（复用 xray_isolate_dostring）
  │   ├── xr_parse         — 语法检查（复用 frontend/parser）
  │   ├── XaAnalyzer       — 静态分析（复用 frontend/analyzer）
  │   ├── xmcp_knowledge   — 语法/API 查询
  │   └── xfmt             — 代码格式化（复用 app/cli/xfmt）
  └── xmcp_resources
      └── xmcp_knowledge   — 静态 Resource 数据
```

### 2.4 传输协议

- **Transport**: stdio（stdin 读 JSON-RPC，stdout 写 JSON-RPC）
- **日志/调试**: stderr（不影响 MCP 协议通信）
- **编码**: UTF-8，Content-Length 头（复用 LSP transport 的 stdin 读取逻辑）

---

## 3. MCP 协议支持

### 3.1 Capabilities

```json
{
  "capabilities": {
    "tools": {},
    "resources": {
      "listChanged": true
    }
  },
  "serverInfo": {
    "name": "xray-mcp-server",
    "version": "0.1.0"
  }
}
```

### 3.2 支持的 MCP 方法

| 方法 | 说明 |
|------|------|
| `initialize` | 握手，交换 capabilities |
| `initialized` | 客户端确认 |
| `tools/list` | 列出所有可用 tool |
| `tools/call` | 调用 tool |
| `resources/list` | 列出所有可用 resource |
| `resources/read` | 读取 resource 内容 |
| `ping` | 心跳 |

---

## 4. Tools 定义

### 4.1 `xray_check` — 编译检查（Phase 1 核心）

**用途**：AI 写完代码后立即验证是否能通过编译。

```json
{
  "name": "xray_check",
  "description": "Check Xray source code for syntax and type errors. Returns compilation diagnostics. Use this after writing or modifying .xr code to verify correctness.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "code": {
        "type": "string",
        "description": "Xray source code to check"
      },
      "file_path": {
        "type": "string",
        "description": "Optional file path for context (import resolution, project mode). If omitted, checks code as standalone snippet."
      },
      "strict": {
        "type": "boolean",
        "description": "Enable strict type checking (runs analyzer in addition to parser). Default: false",
        "default": false
      }
    },
    "required": ["code"]
  }
}
```

**输出示例**（成功）：
```json
{
  "content": [{
    "type": "text",
    "text": "✓ No errors found (12 lines parsed)"
  }]
}
```

**输出示例**（失败）：
```json
{
  "content": [{
    "type": "text",
    "text": "Found 2 errors:\n\n1. line 3:10: error: undefined variable 'x'\n2. line 7:5: error: type mismatch: expected 'int', got 'string'\n\nSuggestions:\n- Line 3: Did you mean to declare 'x' with 'let x = ...'?\n- Line 7: Use int() to convert string to int"
  }],
  "isError": true
}
```

**实现**：
1. 创建临时 `XrayIsolate`
2. 调用 `xr_parse_with_source()` 做语法检查
3. 若 `strict=true`，进一步调用 `xa_analyzer_analyze()` 做类型检查
4. 收集 diagnostics，格式化返回
5. 销毁 isolate

---

### 4.2 `xray_syntax_lookup` — 语法查询（Phase 1 核心）

**用途**：AI 不确定 Xray 语法时按需查询。

```json
{
  "name": "xray_syntax_lookup",
  "description": "Look up Xray language syntax by topic. Use this when you need to know how to write specific Xray constructs (loops, classes, channels, etc.).",
  "inputSchema": {
    "type": "object",
    "properties": {
      "topic": {
        "type": "string",
        "description": "Topic to look up. Examples: 'variables', 'types', 'functions', 'class', 'struct', 'enum', 'generics', 'for_loop', 'match', 'channel', 'coroutine', 'go', 'select', 'defer', 'scope', 'shared', 'move', 'import', 'test', 'operators', 'array', 'map', 'set', 'string', 'error_handling', 'concurrency_rules', 'builtin_functions', 'keywords'"
      }
    },
    "required": ["topic"]
  }
}
```

**输出示例**（`topic: "channel"`）：
```json
{
  "content": [{
    "type": "text",
    "text": "## Channel\n\nChannels are the primary inter-coroutine communication mechanism.\n\n### Declaration\n```xray\nconst ch = Channel(10)          // buffered, capacity 10\nconst ch = Channel<string>(5)   // typed channel\n```\n\n### Must be `const` (not `let`)\nChannels use system heap + refcount. `let ch = Channel()` is a compile error.\n\n### Operations\n```xray\nch.send(value)                   // blocking send\nlet val = ch.recv()              // blocking receive\nlet ok = ch.trySend(value)       // non-blocking send\nlet val, ok = ch.tryRecv()       // non-blocking receive\nlet ok = ch.sendTimeout(val, ms) // timeout send\nlet val = ch.recvTimeout(ms)     // timeout receive\nch.close()                       // close channel\nch.closed                        // check if closed\n```\n\n### Concurrency rules\n- Can be captured by closures (exception to shared rule)\n- Values sent through channel are deep-copied (pointer types)\n- Channel itself is reference-counted on system heap"
  }]
}
```

**实现**：
- 将 `docs/rules/language-spec.md` 按章节（## 标题）拆分为索引
- 启动时一次性加载到内存，按 topic 关键词匹配
- 支持模糊匹配（"for" → 匹配 "控制流" 章节中的 for 部分）
- Topic 别名映射表（如 "loop" → "控制流", "class" → "面向对象"）

**Topic 索引表**：

| topic 关键词 | 映射到 language-spec.md 章节 |
|-------------|---------------------------|
| `literals` | §1 字面量 |
| `variables`, `let`, `const`, `shared` | §2 变量声明 |
| `types`, `int`, `float`, `string`, `nullable` | §3 类型 |
| `operators` | §4 运算符 |
| `if`, `while`, `for`, `for_loop`, `match`, `control_flow` | §5 控制流 |
| `functions`, `fn`, `arrow`, `closure` | §6 函数 |
| `array`, `map`, `set`, `collections`, `bytes`, `json` | §7 集合 |
| `string_methods` | §8 字符串 |
| `class`, `struct`, `interface`, `oop` | §9 面向对象 |
| `enum` | §10 枚举 |
| `generics` | §11 泛型 |
| `error_handling`, `try`, `catch`, `throw` | §12 异常 |
| `import`, `export`, `module` | §13 模块 |
| `coroutine`, `go`, `await`, `channel`, `select`, `defer`, `scope`, `concurrency`, `move` | §14 协程与并发 |
| `test`, `assert` | §15 测试 |
| `builtin_functions`, `print`, `typeof` | §16 内置函数 |
| `global_variables` | §17 全局变量 |
| `keywords` | §19 关键字 |
| `concurrency_rules`, `shared_rules` | §14.5 并发共享三条规则 |
| `array_methods`, `string_methods`, `map_methods`, `channel_methods` | 附录 A 对应部分 |

---

### 4.3 `xray_stdlib_search` — 标准库搜索（Phase 1 核心）

**用途**：AI 查找 Xray 标准库模块和 API。

```json
{
  "name": "xray_stdlib_search",
  "description": "Search the Xray standard library for modules, classes, and functions. Returns API signatures and brief descriptions.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search query. Can be a module name (e.g., 'http'), a function name (e.g., 'Server'), or a description (e.g., 'websocket client')."
      },
      "module": {
        "type": "string",
        "description": "Optional: filter results to a specific module (e.g., 'http', 'json', 'ws', 'crypto')"
      }
    },
    "required": ["query"]
  }
}
```

**输出示例**（`query: "http server"`)：
```json
{
  "content": [{
    "type": "text",
    "text": "## Module: http\n\nImport: `import http`\n\n### http.Server\nHTTP server class.\n\n```xray\nlet server = new http.Server()\nserver.get('/api/users', (req, res) => {\n    res.json({ users: [] })\n})\nserver.listen(8080)\n```\n\n### Key methods:\n- `server.get(path, handler)` — Register GET route\n- `server.post(path, handler)` — Register POST route\n- `server.listen(port)` — Start listening\n- `server.use(middleware)` — Add middleware\n\n### Related:\n- `http.Client` — HTTP client\n- `http.Request` — Request object\n- `http.Response` — Response object"
  }]
}
```

**实现**：
- 启动时扫描 `stdlib/` 目录结构，提取模块列表
- 解析每个模块的 `_binding.c`（或 `.c`）中的注册函数，提取导出的方法签名
- 可选：从 `stdlib/*.h` 头文件注释提取文档
- 全文搜索匹配 query

**标准库模块清单**（自动从 stdlib/ 目录提取）：

| 模块 | 源码目录 | 主要功能 |
|------|---------|---------|
| `base64` | stdlib/base64/ | Base64 编解码 |
| `cluster` | stdlib/cluster/ | 分布式集群通信 |
| `compress` | stdlib/compress/ | 压缩/解压 (zlib) |
| `crypto` | stdlib/crypto/ | 加密/哈希 |
| `csv` | stdlib/csv/ | CSV 解析/生成 |
| `datetime` | stdlib/datetime/ | 日期时间处理 |
| `encoding` | stdlib/encoding/ | 字符编码转换 |
| `gc` | stdlib/gc/ | GC 控制接口 |
| `http` | stdlib/http/ | HTTP 客户端/服务器/HTTP2 |
| `io` | stdlib/io/ | 文件 I/O |
| `json` | stdlib/json/ | JSON 解析/生成 |
| `log` | stdlib/log/ | 日志系统 |
| `math` | stdlib/math/ | 数学函数 |
| `net` | stdlib/net/ | TCP/UDP/TLS 网络 |
| `os` | stdlib/os/ | 操作系统接口 |
| `path` | stdlib/path/ | 路径操作 |
| `regex` | stdlib/regex/ | 正则表达式 |
| `time` | stdlib/time/ | 时间/定时器 |
| `toml` | stdlib/toml/ | TOML 解析 |
| `url` | stdlib/url/ | URL 解析 |
| `ws` | stdlib/ws/ | WebSocket |
| `xml` | stdlib/xml/ | XML 解析 |
| `yaml` | stdlib/yaml/ | YAML 解析 |

---

### 4.4 `xray_run_code` — 运行代码（Phase 2）

**用途**：AI 运行小段代码验证行为。

```json
{
  "name": "xray_run_code",
  "description": "Run a small Xray code snippet and return the output. Use for verifying behavior of small examples. Has a 5-second timeout.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "code": {
        "type": "string",
        "description": "Xray source code to run"
      },
      "timeout_ms": {
        "type": "integer",
        "description": "Execution timeout in milliseconds. Default: 5000, Max: 10000",
        "default": 5000
      }
    },
    "required": ["code"]
  }
}
```

**输出示例**：
```json
{
  "content": [{
    "type": "text",
    "text": "Output:\nHello, World!\n42\n\nExit code: 0\nExecution time: 3ms"
  }]
}
```

**安全限制**：
- 最大执行时间 10 秒
- 禁用文件写入（只读模式）
- 禁用网络连接
- 内存限制 64MB
- 不支持 import（仅内置函数）

---

### 4.5 `xray_format_code` — 代码格式化（Phase 2）

**用途**：格式化 Xray 代码。

```json
{
  "name": "xray_format_code",
  "description": "Format Xray source code with standard style. Returns formatted code.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "code": {
        "type": "string",
        "description": "Xray source code to format"
      },
      "indent_size": {
        "type": "integer",
        "description": "Indent size in spaces. Default: 4",
        "default": 4
      }
    },
    "required": ["code"]
  }
}
```

**实现**：直接复用 `xfmt.c` 的 AST-based formatter。

---

### 4.6 `xray_analyze_project` — 项目分析（Phase 2）

**用途**：分析整个项目的编译错误。

```json
{
  "name": "xray_analyze_project",
  "description": "Analyze an Xray project directory for all compilation errors and warnings. Equivalent to 'xray check --strict <directory>'.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "Absolute path to the project directory or a single .xr file"
      },
      "strict": {
        "type": "boolean",
        "description": "Enable strict type checking. Default: true",
        "default": true
      }
    },
    "required": ["path"]
  }
}
```

**实现**：复用 `xcmd_check.c` 的逻辑，递归扫描 `.xr` 文件。

---

### 4.7 `xray_run_tests` — 运行测试（Phase 2）

**用途**：运行项目测试并返回结果。

```json
{
  "name": "xray_run_tests",
  "description": "Run Xray tests for a project or specific test file. Returns test results with pass/fail details.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "Path to test file or project directory"
      },
      "filter": {
        "type": "string",
        "description": "Optional: filter test names by pattern (e.g., 'test_http_*')"
      },
      "timeout": {
        "type": "integer",
        "description": "Per-test timeout in milliseconds. Default: 30000",
        "default": 30000
      }
    },
    "required": ["path"]
  }
}
```

**实现**：复用 `xcmd_test.c` 的测试运行器，捕获结果而非打印到 stdout。

---

### 4.8 `xray_example_search` — 示例搜索（Phase 2）

**用途**：按功能搜索 Xray 示例代码。

```json
{
  "name": "xray_example_search",
  "description": "Search for Xray code examples by functionality description. Returns relevant example snippets from the example library.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Description of what you're looking for (e.g., 'concurrent web scraper', 'channel communication', 'class inheritance')"
      },
      "max_results": {
        "type": "integer",
        "description": "Maximum number of examples to return. Default: 3",
        "default": 3
      }
    },
    "required": ["query"]
  }
}
```

**实现**：
- 索引 `demos/**/*.xr`，按目录分类和内容关键词匹配
- 返回最相关的代码片段（截取相关函数/类，不是整个文件）

---

### 4.9 `xray_explain_error` — 错误解释（Phase 3）

```json
{
  "name": "xray_explain_error",
  "description": "Explain an Xray compilation error and suggest fixes. Provide the error message from xray_check or xray_analyze_project.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "error_message": {
        "type": "string",
        "description": "The error message to explain"
      },
      "code_context": {
        "type": "string",
        "description": "Optional: the source code surrounding the error"
      }
    },
    "required": ["error_message"]
  }
}
```

---

## 5. Resources 定义

Resources 是 MCP 的静态数据接口，AI 可以主动读取。

### 5.1 Resource 列表

| URI | 名称 | 说明 |
|-----|------|------|
| `xray://spec/overview` | Language Overview | 语言一句话介绍 + 核心特性列表 |
| `xray://spec/full` | Full Language Spec | 完整语法规范（language-spec.md 全文） |
| `xray://spec/cheatsheet` | Syntax Cheatsheet | 精简速查表（2KB 以内） |
| `xray://spec/concurrency` | Concurrency Model | 并发安全模型详解 |
| `xray://stdlib/modules` | Standard Library Modules | 所有模块列表 + 简介 |
| `xray://stdlib/{module}` | Module API | 特定模块的完整 API 文档 |
| `xray://examples/list` | Example List | 所有示例的标题和分类 |
| `xray://examples/{name}` | Example Code | 特定示例的完整代码 |

### 5.2 关键 Resource 内容设计

#### `xray://spec/cheatsheet`（最重要的 Resource）

AI 在开始写 Xray 代码前应首先读取的资源，控制在 ~2KB：

```markdown
# Xray Language Cheatsheet

## Basics
let x = 1; const PI = 3.14; let name: string = "hello"
fn add(a: int, b: int): int { return a + b }
let double = (x) => x * 2

## Types
int, float, string, bool, void | Array<T>, Map<K,V>, Set<T>
int?, string?  (nullable) | Json, Bytes, BigInt, Channel<T>

## Control flow
if (cond) {} else {}
for (i in 0..10) {} | for (item in arr) {}
match x { 1 => "one", _ => "other" }

## OOP
class Dog extends Animal { constructor(name) { super(name) } }
struct Point { x: float; y: float }
interface Shape { area(): float }
enum Color { Red, Green, Blue }

## Collections
[1,2,3]  arr.map(fn).filter(fn)  arr[1:3]
{"key" => val}  m.get(k)  m.set(k,v)
#[1,2,3]  s.add(v)  s.has(v)

## Concurrency (core differentiator)
let t = go someFunc(args)           // spawn coroutine
let result = await t                // wait for result
const ch = Channel(10)              // buffered channel (must be const)
ch.send(val); let v = ch.recv()     // send/receive
select { msg from ch => handle(msg); after 1000 => timeout() }
shared const CFG = {...}            // immutable cross-coroutine sharing
go fn() { let d = move data }()    // ownership transfer

## Rules: compile pass = concurrency safe
- shared const: zero-copy read across coroutines
- Channel: communication (deep copy on send)
- Function params: deep copy to child coroutine
- Regular let/const: cannot be captured by go closures

## Modules
import http; import json; import time
export fn helper() {}

## Testing
@test fn test_add() { assert_eq(1+1, 2) }
```

#### `xray://spec/concurrency`

并发模型是 Xray 最独特的部分，AI 最容易写错，需要专门的 Resource：

```markdown
# Xray Concurrency Model

## Golden Rule: If it compiles, it's concurrency-safe.

## Three ways to share data across coroutines:

### 1. Channel (communication)
const ch = Channel(10)   // MUST be const
ch.send(value)           // deep copies pointer types
let val = ch.recv()

### 2. shared const (immutable sharing)
shared const CONFIG = { host: "localhost", port: 8080 }
// Zero-copy reads from any coroutine. Cannot modify.

### 3. Function parameters (deep copy)
go processData(myArray)  // myArray is deep-copied

## What you CANNOT do:
let x = [1,2,3]
go fn() { print(x) }()  // COMPILE ERROR: cannot capture 'x'
// Fix: pass as parameter → go fn(data) { print(data) }(x)

## Move semantics:
shared let data = [1,2,3]
go fn() { let local = move data }()  // ownership transferred
print(data)  // COMPILE ERROR: 'data' already moved
```

---

## 6. 实现方案

### 6.1 Phase 1：最小可用（预计 2-3 天）

**目标**：3 个核心 tool + 3 个 resource，让 AI 能写出基本正确的 Xray 代码。

**新增文件**：
```
src/app/mcp/
├── xmcp_server.c       — 主循环、stdio JSON-RPC 收发
├── xmcp_server.h       — 服务器接口
├── xmcp_protocol.c     — MCP 协议消息构造/解析
├── xmcp_protocol.h     — 协议类型定义
├── xmcp_tools.c        — Tool handlers (check, syntax_lookup, stdlib_search)
├── xmcp_tools.h        — Tool 接口
├── xmcp_knowledge.c    — 知识库加载（索引 language-spec.md）
└── xmcp_knowledge.h    — 知识库接口
```

**修改文件**：
```
src/app/cli/xcli.c      — 添加 "mcp-server" 命令路由
src/app/cli/xcli.h      — 添加 cmd_mcp_server 声明
CMakeLists.txt           — 添加 src/app/mcp/ 源文件
```

**Tool 清单**：
1. `xray_check` — 编译检查
2. `xray_syntax_lookup` — 语法查询
3. `xray_stdlib_search` — 标准库搜索

**Resource 清单**：
1. `xray://spec/cheatsheet` — 速查表
2. `xray://spec/concurrency` — 并发模型
3. `xray://stdlib/modules` — 模块列表

### 6.2 Phase 2：完整工具链（预计 1-2 周）

新增 Tool：
- `xray_run_code` — 运行代码片段
- `xray_format_code` — 代码格式化
- `xray_analyze_project` — 项目分析
- `xray_run_tests` — 运行测试
- `xray_example_search` — 示例搜索

新增 Resource：
- `xray://spec/full` — 完整规范
- `xray://stdlib/{module}` — 按模块的 API 文档
- `xray://examples/list` — 示例列表
- `xray://examples/{name}` — 示例代码

### 6.3 Phase 3：高级能力（持续迭代）

- `xray_explain_error` — 错误解释
- DAP 集成（运行时调试）
- 包管理集成（`xray add/remove`）
- 自动生成 Resource 数据的 CI 流程

---

## 7. 数据源映射

每个 tool/resource 的数据从哪里来：

| 功能 | 数据源 | 提取方式 |
|------|--------|---------|
| 语法查询 | `docs/rules/language-spec.md` | 启动时解析 markdown，按 `##` 章节分段索引 |
| 标准库 API | `stdlib/*/` 目录结构 + `.c` 文件 | 扫描目录获取模块列表；解析 C 文件中的 `xr_register_*` 调用提取导出函数名 |
| 内置方法 | `src/module/xbuiltin_method_defs.h` | 解析 X-macro 定义获取类型方法签名 |
| 编译检查 | xray 编译器本身 | 创建 XrayIsolate，调用 parser + analyzer |
| 代码格式化 | `src/app/cli/xfmt.c` | 直接调用 xfmt API |
| 测试运行 | `src/app/cli/xcmd_test.c` | 复用测试运行器 |
| 示例代码 | `demos/**/*.xr` | 启动时扫描并建立关键词索引 |
| 并发规则 | `docs/rules/language-spec.md` §14 + `docs/rules/design-principles.md` | 编译到二进制中的静态字符串 |

### 7.1 知识库嵌入策略

对于不依赖文件系统的部署（如独立二进制分发），关键文档应 **编译时嵌入**：

```c
// xmcp_knowledge.c — 自动生成（build 时 xxd 或 cmake configure_file）
static const char SPEC_CHEATSHEET[] = "# Xray Language Cheatsheet\n...";
static const char SPEC_CONCURRENCY[] = "# Xray Concurrency Model\n...";
static const char STDLIB_MODULES[] = "base64, cluster, compress, ...";
```

同时支持运行时覆盖（如果检测到 `docs/` 目录存在，优先读文件系统）：
1. 先检查 `xray` 二进制同级的 `../docs/rules/language-spec.md`
2. 再检查 `$XRAY_HOME/docs/`
3. 最后 fallback 到编译时嵌入的数据

---

## 8. 配置与启动

### 8.1 CLI 接口

```bash
# 启动 MCP server（stdio 模式）
xray mcp-server

# 带选项
xray mcp-server --log-level debug    # 日志级别
xray mcp-server --log-file /tmp/mcp.log  # 日志文件
xray mcp-server --no-sandbox         # 禁用 xray_run_code 沙箱
```

### 8.2 各 AI IDE 配置

**Windsurf** (`.windsurf/mcp.json`)：
```json
{
  "mcpServers": {
    "xray": {
      "command": "xray",
      "args": ["mcp-server"]
    }
  }
}
```

**Cursor** (`.cursor/mcp.json`)：
```json
{
  "mcpServers": {
    "xray": {
      "command": "xray",
      "args": ["mcp-server"]
    }
  }
}
```

**Claude Code**：
```bash
claude mcp add --transport stdio xray -- xray mcp-server
```

**VS Code Copilot** (`.vscode/settings.json`)：
```json
{
  "mcp": {
    "servers": {
      "xray": {
        "command": "xray",
        "args": ["mcp-server"]
      }
    }
  }
}
```

**Gemini CLI** (`~/.gemini/settings.json`)：
```json
{
  "mcpServers": {
    "xray": {
      "command": "xray",
      "args": ["mcp-server"]
    }
  }
}
```

---

## 9. 协议交互示例

### 9.1 初始化握手

```
→ {"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{"roots":{"listChanged":true}},"clientInfo":{"name":"windsurf","version":"1.0"}}}

← {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2025-03-26","capabilities":{"tools":{},"resources":{"listChanged":true}},"serverInfo":{"name":"xray-mcp-server","version":"0.1.0"}}}

→ {"jsonrpc":"2.0","method":"notifications/initialized"}
```

### 9.2 工具列表

```
→ {"jsonrpc":"2.0","id":2,"method":"tools/list"}

← {"jsonrpc":"2.0","id":2,"result":{"tools":[
  {"name":"xray_check","description":"Check Xray source code for syntax and type errors...","inputSchema":{...}},
  {"name":"xray_syntax_lookup","description":"Look up Xray language syntax by topic...","inputSchema":{...}},
  {"name":"xray_stdlib_search","description":"Search the Xray standard library...","inputSchema":{...}}
]}}
```

### 9.3 AI 调用工具

```
→ {"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"xray_check","arguments":{"code":"let x = 1\nprint(x + \"hello\")","strict":true}}}

← {"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"Found 1 error:\n\n1. line 2:9: error: operator '+' cannot be applied to types 'int' and 'string'\n\nSuggestion: Use string() to convert int, or use string interpolation: \"${x}hello\""}],"isError":true}}
```

### 9.4 AI 读取 Resource

```
→ {"jsonrpc":"2.0","id":4,"method":"resources/read","params":{"uri":"xray://spec/cheatsheet"}}

← {"jsonrpc":"2.0","id":4,"result":{"contents":[{"uri":"xray://spec/cheatsheet","mimeType":"text/markdown","text":"# Xray Language Cheatsheet\n..."}]}}
```

---

## 10. 与 LSP/DAP 的关系

| 维度 | LSP | DAP | MCP |
|------|-----|-----|-----|
| **服务对象** | IDE 编辑器 | IDE 调试器 | AI 助手 |
| **交互方式** | 实时、增量 | 断点/步进 | 请求/响应 |
| **文档管理** | 打开的文件实时同步 | N/A | 无状态，每次传入代码 |
| **代码共享** | LSP 有完整的 parser/analyzer | DAP 有 eval/inspect | MCP 复用 parser/analyzer |

**代码复用**：
- MCP `xray_check` 复用 LSP 的 `xr_parse_with_source` + `xa_analyzer_analyze`
- MCP `xray_format_code` 复用 `xfmt` 格式化器
- MCP `xray_run_tests` 复用 `xcmd_test` 测试运行器
- MCP stdio transport 可参考 LSP 的 `xlsp_transport.c`

**不共享**：
- MCP 是无状态的（每次 tool call 独立），LSP 是有状态的（维护文档、索引）
- MCP 不需要 LSP 的文档同步、增量解析等复杂逻辑

---

## 11. 测试策略

### 11.1 单元测试

```c
// tests/test_mcp_knowledge.c — 知识库索引测试
void test_topic_lookup(void) {
    XrMcpKnowledge *kb = xmcp_knowledge_load("docs/rules/language-spec.md");
    const char *result = xmcp_knowledge_lookup(kb, "channel");
    assert(strstr(result, "Channel") != NULL);
    assert(strstr(result, "ch.send") != NULL);
    xmcp_knowledge_free(kb);
}
```

### 11.2 集成测试

用 `xray mcp-server` 启动进程，通过 stdin/stdout 发送 JSON-RPC 消息：

```bash
# tests/test_mcp_integration.sh
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test"}}}' | xray mcp-server
```

### 11.3 端到端测试

用真实 AI IDE 配置 MCP，验证：
1. AI 能查到 Channel 语法 → 写出正确的 Channel 代码
2. AI 能查到 http.Server API → 写出能编译的 HTTP 服务器
3. AI 写的代码通过 `xray_check` 后确实能运行

---

## 12. 未来扩展

### 12.1 Prompts（MCP Prompts 能力）

MCP 协议还支持 Prompts — 预定义的提示词模板。可以后续添加：

| Prompt 名 | 用途 |
|-----------|------|
| `new_project` | 创建新 Xray 项目的引导提示词 |
| `add_concurrency` | 将同步代码改为并发版本的提示词 |
| `debug_error` | 调试编译错误的引导提示词 |

### 12.2 Sampling

MCP 的 Sampling 能力允许 server 反向请求 AI 生成内容。可用于：
- 自动生成测试用例
- 根据错误自动建议修复方案

### 12.3 包生态集成

当 Xray 包管理器成熟后：
- `xray_pkg_search` — 搜索 pkg.xray-lang.org
- `xray_pkg_add` — 添加依赖
- `xray_pkg_info` — 查看包信息

---

## 附录 A：完整 Tool 优先级矩阵

| Tool | Phase | 实现复杂度 | 用户价值 | 依赖 |
|------|-------|-----------|---------|------|
| `xray_check` | 1 | 低（复用现有） | ⭐⭐⭐⭐⭐ | XrayIsolate + parser |
| `xray_syntax_lookup` | 1 | 低（解析 md） | ⭐⭐⭐⭐⭐ | language-spec.md |
| `xray_stdlib_search` | 1 | 中（解析 C 源码） | ⭐⭐⭐⭐ | stdlib/ 目录 |
| `xray_run_code` | 2 | 中（沙箱） | ⭐⭐⭐⭐ | XrayIsolate |
| `xray_format_code` | 2 | 低（复用 xfmt） | ⭐⭐⭐ | xfmt |
| `xray_analyze_project` | 2 | 低（复用 check） | ⭐⭐⭐⭐ | xcmd_check |
| `xray_run_tests` | 2 | 中（结果捕获） | ⭐⭐⭐ | xcmd_test |
| `xray_example_search` | 2 | 低（文件索引） | ⭐⭐⭐ | demos/ |
| `xray_explain_error` | 3 | 高（错误库） | ⭐⭐⭐ | 错误消息数据库 |
