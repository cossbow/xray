---
id: struct
title: Struct
spec: #54-struct-声明
aliases: [structs, value_type]
---
## Struct

Value types with no inheritance. Struct literals use `StructName{field: value}` syntax.

### Struct declaration and literals
```xray
struct Point {
    x: float
    y: float

    magnitude_sq() -> float {
        return this.x * this.x + this.y * this.y
    }
}

// Two creation styles
let p = new Point()                  // default-construct (zero-valued fields), then assign
p.x = 3.0
p.y = 4.0

let q = Point{x: 3.0, y: 4.0}        // struct literal: TypeName + { field: value }
let pt = Point{x: 1.0, y: 2.0}

// Value semantics: assignment and parameter passing copy
let b = q                            // b is an independent copy of q
b.x = 99.0
// q.x is still 3.0
```
