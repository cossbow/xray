---
id: types
title: Types
spec: #2-类型系统-type-system
aliases: [int, float, string, bool, nullable, type]
---
## Types

### Primitive types
- `int` — 64-bit integer
- `float` — 64-bit double
- `string` — UTF-8 string
- `bool` — true / false
- `void` — no value

### Nullable
```xray
let x: int? = null     // nullable int
let v = x ?? 42        // null coalescing
let y = obj?.field      // optional chaining
```

### Collection types
- `Array<T>` — ordered list
- `Map<K,V>` — hash map (`#{"key": val}`)
- `Set<T>` — unique elements (`#[1, 2, 3]`)
- `Json` — dynamic JSON object
- `Bytes` — byte buffer
- `BigInt` — arbitrary precision integer
- `Channel<T>` — inter-coroutine communication

### Union types
```xray
let value: int | string = 42   // can hold int or string
let result: int | null = null  // equivalent to int?
```
Note: `T?` is shorthand for `T | null`.
