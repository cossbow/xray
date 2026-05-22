---
id: generics
title: Generics
aliases: [generic, type_parameter, template]
---
## Generics

```xray
class Box<T> {
    value: T
    constructor(v: T) { this.value = v }
    get() -> T { return this.value }
    map(f: (T) -> T) -> Box<T> { return new Box(f(this.value)) }
}

let intBox = new Box(42)      // T inferred as int
let strBox = new Box("hello") // T inferred as string
```

### Multi-parameter generics
```xray
class Pair<A, B> {
    first: A
    second: B
    constructor(a: A, b: B) { this.first = a; this.second = b }
}
```
