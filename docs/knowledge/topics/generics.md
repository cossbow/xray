---
id: generics
title: Generics
spec: #9-泛型-generics
aliases: [generic, type_parameter, template]
---
## Generics

### Generic declarations
```xray
// Generic function
fn identity<T>(x: T) -> T {
    return x
}

let a = identity<int>(42)
let b = identity("hello")               // T inferred as string

// Generic class
class Box<T> {
    value: T
    constructor(v: T) { this.value = v }
    get() -> T { return this.value }
}

let b1 = new Box<int>(42)
let b2 = new Box<string>("hi")

// Multi-parameter generic
class Pair<K, V> {
    key: K
    value: V
    constructor(k: K, v: V) {
        this.key = k; this.value = v
    }
}

// Generic interface
interface Comparable<T> {
    compareTo(other: T) -> int
}
```

### Constraints
```xray
// Single constraint
fn first<T: Comparable>(a: T, b: T) -> T {
    return a
}

// Multiple constraints (intersection) — T must satisfy Comparable, Hashable, and Stringable
fn passThrough<T: Comparable & Hashable & Stringable>(x: T) -> T {
    return x
}

// Multiple type parameters, each independently constrained
fn pickValue<K: Hashable, V>(k: K, v: V) -> V {
    return v
}
```

### Type inference
```xray
identity(42)                    // T inferred as int
new Box("hello")                // T inferred as string
new Pair("key", 100)            // K=string, V=int
```

### Explicit instantiation
```xray
let empty = new Array<int>()              // no element to infer from
let m = new Map<string, int>()
let result = identity<float>(0)            // 0 defaults to int; force float
```

### Runtime reflection
```xray
class Container<T> {
    items: Array<T>
}
let c = new Container<int>()
print(Reflect.typeOf(c))       // "Container<int>"
```
