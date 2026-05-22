---
id: concurrency_rules
title: Concurrency Safety Rules
aliases: [shared_rules, sharing, move, safety, concurrent]
---
## Concurrency Safety Rules

**Golden rule: If it compiles, it's concurrency-safe.**

### Three ways to share data across coroutines:

| Mechanism | Syntax | How it works |
|-----------|--------|-------------|
| shared const | `shared const CFG = {...}` | Zero-copy reads, immutable |
| Channel | `shared const ch = new Channel<T>(n)` | Deep-copy on send |
| Function params | `go fn(data)` | Deep-copied to child |

### What you CANNOT do (compiler rejects these):
```xray
let x = [1,2,3]
go fn() { print(x) }()    // ERROR: cannot capture let variable

let ch = new Channel<int>(1) // ERROR: Channel must be shared const for coroutine sharing

shared let data = [1,2,3]
go fn() { data.push(4) }() // ERROR: cannot capture shared let
```

### Move semantics
```xray
shared let data = [1,2,3]
go fn() { let local = move data }()  // ownership transferred
print(data)  // ERROR: data already moved
```
