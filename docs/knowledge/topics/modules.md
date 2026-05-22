---
id: modules
title: Modules & Imports
spec: #11-模块系统-modules
aliases: [import, export, module, require, package]
---
## Modules & Imports

### Imports
```xray
// 1. stdlib: bare identifier; without `as`, the alias equals the module name
import time
import datetime
import http as httpClient

// 2. third-party packages: owner/name form
import alice/utils
import bob/http_client as httpClient

// 3. file-path or directory-path: string literal, optional explicit alias, otherwise inferred from the trailing path segment
import "./modules/mod_a.xr" as a
import "../utils/string_utils.xr" as utils
import "models/user" as user

// 4. named imports: members may be renamed; the `from` operand may be a quoted path or a bare module name
import { readFile, writeFile as write } from io
import { publicFn } from "./modules/mod_a.xr"
```

### Use module symbols
```xray
import time
let t = time.now()
time.sleep(100)
```

### Exports
```xray
// 1. export a declaration directly
export fn helper() { return }
export class MyClass {
    value: int
    constructor() { this.value = 1 }
}
export const VERSION = "1.0"

// 2. export an already-declared identifier (declare internally first, expose at the end)
fn _helper() -> string { return "..." }
fn publicFn() -> string { return _helper() }
export publicFn

// 3. re-export (with optional renaming)
export { getUser, getUserAge as getAge } from "./user"

// 4. wildcard re-export (forward all exports of another module)
export * from "./product"
```
