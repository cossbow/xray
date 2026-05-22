---
id: builtin_functions
title: Built-in Functions
spec: #13-内置函数-built-in-functions
aliases: [builtin, print, dump, typeof, builtins]
---
## Built-in Functions

### Core built-ins
- `print(value)` — print with newline
- `dump(value)` — debug print with type info
- `typeof(value)` — return the runtime type name
- `x is T` — runtime type check with analyzer narrowing
- `int(value)` / `float(value)` / `string(value)` / `bool(value)` — explicit conversions
- `assert(condition)` and `assert_*` helpers — testing assertions

### Type inspection
```xray
let x = 42
print(typeof(x))                // "int"
print(x is int)                 // true
print(typeof(x) == "int")       // true
```
