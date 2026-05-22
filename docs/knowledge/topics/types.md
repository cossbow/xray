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

### Truthy / falsy
```xray
let x: int? = 41
if (x) {                  // truthy context: enters when x is neither null nor 0
    print(x + 1)          // x is narrowed to int in this branch
}

let s: string = ""
if (s) {
    print("non-empty")
} else {
    print("empty")             // falsy: enters else
}

let m: Map<string, int> = #{}
if (m) {
    print("non-empty map")
} else {
    print("empty map")         // falsy: empty Map
}

let a: int? = null
let b = a ?? 0                  // null coalescing: b = 0
```

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

### Type aliases
```xray
type Result = int | string
type Mapper = (int) -> int
type Point = { x: float, y: float }
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

### Explicit casts
```xray
let n = x as int        // throws TypeError on failure
let n = x as int?       // returns null on failure (safe cast)
```

### Construction expressions
```xray
let p = new Point(1.0, 2.0)
let arr = new Array<int>()
let ch = new Channel<int>(10)
let m = new Map<string, int>()
```
```xray
let a = [1, 2, 3]              // equivalent to new Array<int>() + push
let m = #{}                    // equivalent to new Map<...>()
let p = Point{x: 1, y: 2}      // struct literal
```
