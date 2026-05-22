---
id: variables
title: Variables
spec: #51-let-const-shared
aliases: [let, const, var, shared, declaration]
---
## Variables

```xray
let x = 1              // mutable
const PI = 3.14        // immutable
let name: string = "Alice"  // with type annotation
let age: int? = null   // nullable
```

### Modifiers
- `let` — mutable, can be reassigned
- `const` — immutable, cannot be reassigned
- `shared const` — immutable, readable across coroutines (zero-copy)
- `shared let` — mutable, uses move semantics for ownership transfer

### Multiple assignment
```xray
let (a, b) = (1, 2)
let (q, r) = divmod(17, 5)  // from function returning multiple values
```

### Destructuring
```xray
let [a, b, c] = [10, 20, 30]    // array destructuring
let { name, age } = obj          // object destructuring (no renaming)
```
