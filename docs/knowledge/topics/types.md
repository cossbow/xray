---
id: types
title: Types
spec: #2-类型系统-type-system
aliases: [int, float, string, bool, nullable, type, tuple, unit, union, Result]
---
## Types

### Primitive and built-in categories
- `int` — default signed integer
- `float` — default floating-point number
- `string` — UTF-8 string
- `bool` — `true` / `false`
- `()` — Unit, no return value
- `Array<T>`, `Map<K,V>`, `Set<T>`, `Channel<T>`, `Json`, `Bytes`, `BigInt`

### Nullable
```xray
let x: int? = null      // OK
let y: int? = 42        // OK
let z: int = null       // compile error: null is not int
```

### Collections
```xray
let a: Array<int> = [1, 2, 3]
let b = [1, 2, 3]                // inferred as Array<int>
let c: Array<string> = []         // explicit empty array
```
```xray
let m: Map<string, int> = #{"a": 1, "b": 2}
let m2 = #{"a": 1, "b": 2}
let empty = #{}                                     // empty Map

m["c"] = 3                                          // insert / update
let v = m["a"]                                      // lookup; returns null if absent
```
```xray
let s: Set<int> = #[1, 2, 3]
```
```xray
const ch: Channel<int> = new Channel<int>(10)
```

### Union types
```xray
let v: int | string = 42
v = "hello"             // OK
```

### Tuple types
```xray
// Literals
let t = (1, 2, 3)                 // type inferred as (int, int, int)
let h = (10, "hi", true)          // heterogeneous tuple
let single = (99,)                // single-element tuple: note trailing comma

// Type annotation
let p: (int, string) = (7, "ok")

// Field access: .N (N is a compile-time constant integer index)
let first = t.0                   // 1
let mid   = t.1                   // 2
let nest  = ((1, 2), (3, 4))
let a     = nest.0.0              // 1
let b     = nest.1.1              // 4

// Function return and destructuring
fn divmod(a: int, b: int) -> (int, int) { return (a / b, a % b) }
let (q, r) = divmod(17, 5)        // tuple destructure

// Generic
fn pair<A, B>(a: A, b: B) -> (A, B) { return (a, b) }
let p2 = pair(1, "x")             // (int, string)
```

### Type inference
```xray
let x = 1               // x: int
let y = 1.5             // y: float
let z = "hello"         // z: string
let a = [1, 2, 3]       // a: Array<int>
let m = #{"a": 1}    // m: Map<string, int>
let p = { name: "A" }   // p: { name: string } — structured object type
let f = (x: int) -> x   // f: (int) -> int — arrow parameters require annotation
```
