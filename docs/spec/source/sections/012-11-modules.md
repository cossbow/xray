---
id: spec.11_modules
order: 012
---

<!-- xr-spec:cn -->
---

## 11. 模块系统 (Modules)

> 真值源：`src/module/xmodule.c`、`src/module/xmodule_resolve.c`、`src/frontend/parser/xparse_import.c`。

### 11.1 模块定义

- 每个 `.xr` 文件是一个模块。
- 模块名 = 文件名（去除 `.xr` 后缀）。
- 模块路径反映目录结构：`src/utils/string.xr` → `utils.string`。

### 11.2 项目结构

```
my_project/
├── xray.toml              # 包清单（包名、依赖、入口）
├── src/
│   ├── main.xr            # 入口
│   ├── utils.xr
│   └── lib/
│       └── helper.xr
├── tests/
│   └── test_utils.xr
└── docs/
```

`xray.toml` 示例：

```toml
[package]
name = "my_project"
version = "0.1.0"
entry = "src/main.xr"

[dependencies]
http = "1.0"
json = "0.2"

[dev-dependencies]
test = "1.0"
```

### 11.3 `import` 语法

```ebnf
ImportStmt ::= 'import' ImportMembers 'from' ImportModule
            |  'import' ImportModule ('as' Identifier)?
ImportMembers ::= '{' ImportMember (',' ImportMember)* ','? '}'
ImportMember  ::= Identifier ('as' Identifier)?
ImportModule  ::= StringLiteral | ModuleName
ModuleName    ::= Identifier ('/' Identifier)?
```

```xray
// 1. stdlib：裸标识符；没有 `as` 时别名等于模块名
import time
import datetime
import http as httpClient

// 2. 第三方包：owner/name 形式
import alice/utils
import bob/http_client as httpClient

// 3. 文件路径或目录路径：字符串字面量，可显式 alias，也可从路径尾段推导
import "./modules/mod_a.xr" as a
import "../utils/string_utils.xr" as utils
import "models/user" as user

// 4. 命名 import：成员可重命名；`from` 后可接字符串路径或裸模块名
import { readFile, writeFile as write } from io
import { publicFn } from "./modules/mod_a.xr"
```

**不支持** JavaScript 默认导入 `import name from "module"`。在 Xray 中使用 `import "module" as name`、`import module` 或 `import { name } from module`。

**解析算法**（按优先级）：
1. **stdlib 命名解析**：裸标识符 `import time` → 内置 stdlib 模块表。
2. **相对路径**：`"./xxx.xr"` 与 `"../xxx.xr"` 相对当前文件解析。
3. **项目根目录路径**：不以 `./` 或 `../` 开头的 quoted path 作为项目目录 import。
4. **第三方包**：`owner/name` 由 `xray.toml` 的 `[dependencies]` 解析。

**真值源**：`xparse_import.c` 与 `xmodule_resolve_path()`。

### 11.4 `export` 与可见性

xray 支持三种 export 形式：

```ebnf
ExportStmt ::= 'export' Declaration                              // 直接 export 声明
            |  'export' Identifier                               // export 已声明的标识符
            |  'export' '{' ExportSpec (',' ExportSpec)* '}' 'from' StringLiteral
            |  'export' '*' 'from' StringLiteral
ExportSpec ::= Identifier ('as' Identifier)?
```

```xray
// 1. 直接 export 声明
export fn publicFn() -> string { return "hi" }
export class PublicClass { ... }
export const VERSION = "1.0"

// 2. export 已声明的标识符（用于内部先声明、最后统一暴露）
fn _helper() -> string { return "..." }
fn publicFn() -> string { return _helper() }
export publicFn

// 3. 重导出（带可选重命名）
export { getUser, getUserAge as getAge } from "./user"

// 4. 通配重导出（把另一个模块的全部 export 转出）
export * from "./product"
```

- 未标 `export` 的声明仅模块内可见（**私有**）。
- 模块的内部状态（`let _x`, `const _VERSION`, `fn _helper`）在不同模块中互不冲突，即使同名。
- 重导出与通配重导出常用于 `index.xr` 聚合子模块的公开 API。

### 11.5 命名约定

- 模块名 `snake_case`：`http_client.xr` / `string_utils.xr`。
- 公开符号 `camelCase` 或 `PascalCase`（类/接口）。
- 内部符号约定前缀 `_`：`_internal_helper`。

### 11.6 循环依赖

xray **禁止**循环依赖。模块加载时建立 DAG；检测到循环 → 编译错误。

### 11.7 native 模块

C 层暴露的模块（如 `time`、`http`、`os`）通过 native ABI 注册：

```c
// C 端
XRAY_API void register_time_module(xray_vm_t* vm) {
    xray_module_t* m = xray_module_create(vm, "time");
    xray_module_add_fn(m, "now", time_now);
    xray_module_add_fn(m, "sleep", time_sleep);
    xray_module_register(vm, m);
}
```

xray 端用法相同：

```xray
import time
let t = time.now()
time.sleep(100)
```

详见 `docs/rules/architecture.md` 的"native 模块注册"章节。
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 11. Modules

### 11.1 Module Identity

Each `.xr` file is a module. Module paths follow directory structure. A module exports only declarations explicitly marked with `export`.

### 11.2 Import Syntax

```ebnf
ImportStmt ::= 'import' ImportMembers 'from' ImportModule
            |  'import' ImportModule ('as' Identifier)?
ImportMembers ::= '{' ImportMember (',' ImportMember)* ','? '}'
ImportMember  ::= Identifier ('as' Identifier)?
ImportModule  ::= StringLiteral | ModuleName
ModuleName    ::= Identifier ('/' Identifier)?
```

Examples:

```xray
import time
import http as h
import alice/utils
import "./helper.xr" as helper
import "models/user" as user
import { readFile, writeFile as write } from io
```

### 11.3 Export Syntax

```xray
export fn f() {}
export const VERSION = "1.0"
export f, VERSION
export { a, b as c } from "./m"
export * from "./m"
```

### 11.4 Resolution

Resolution distinguishes:

1. Standard library modules.
2. Relative quoted paths (`./`, `../`).
3. Project-root quoted paths.
4. Third-party `owner/name` package names.

JavaScript default import syntax is rejected.
<!-- /xr-spec:en -->
