---
id: variables
title: Variables
spec: #51-let-const-shared
aliases: [let, const, var, shared, declaration]
---
## Variables

### let
```xray
let x = 1                         // type inferred as int
let name: string = "Alice"        // explicit type
let count: int                    // no initializer: zero value used
```

### const
```xray
const PI = 3.14159
const MAX_LEN: int = 1024
```

### shared const
```xray
shared const CONFIG = { host: "localhost", port: 8080 }
shared const PRIMES = [2, 3, 5, 7, 11]
```

### shared let
```xray
shared let buffer = new Bytes(1024)
```

### Destructuring
```xray
// array destructuring
let [a, b, c] = [1, 2, 3]
let [first, , third] = [10, 20, 30]         // skip elements

// tuple destructuring (multi-return)
let (q, r) = divmod(17, 5)

// object destructuring (extract by name; **no** rename syntax)
let { name, age } = { name: "Alice", age: 30 }
```
