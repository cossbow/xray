---
id: modules
title: Modules & Imports
spec: #11-模块系统-modules
aliases: [import, export, module, require, package]
---
## Modules & Imports

```xray
import http
import json
import time

fn handler() {}
let text = "{\"ok\":true}"

// Use module functions
http.route("GET", "/", handler)
let data = json.parse(text)
time.sleep(100)

// Export from your module
export fn helper() { return }
export class MyClass {
    value: int
    constructor() { this.value = 1 }
}
```
