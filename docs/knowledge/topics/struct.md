---
id: struct
title: Struct
spec: #54-struct-声明
aliases: [structs, value_type]
---
## Structs

Value types (no inheritance). Created with `StructName{field: value}` syntax.

```xray
struct Vec2 {
    x: float
    y: float
    length() -> float {
        return (this.x * this.x + this.y * this.y).sqrt()
    }
}
let p = Vec2{x: 3.0, y: 4.0}
```
