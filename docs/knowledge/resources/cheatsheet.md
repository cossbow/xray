---
id: cheatsheet
---
# Xray Language Cheatsheet

## Basics
```xray
let x = 1; const PI = 3.14; let name: string = "hello"
fn add(a: int, b: int) -> int { return a + b }
let double = (x: int) -> x * 2
```

## Types
int, float, string, bool, void | Array<T>, Map<K,V>, Set<T>
int?, string? (nullable) | int | string (union) | Json, Bytes, BigInt, Channel<T>

## Control flow
```xray
if (cond) {} else {}
for (i in 0..10) {} | for (item in arr) {}
match (x) { 1 -> "one", _ -> "other" }
```

## OOP
```xray
class Dog extends Animal { constructor(n: string) { super(n) } }
struct Point { x: float; y: float }
interface Shape { area() -> float }
enum Color { Red, Green, Blue }
```

## Collections
```xray
[1,2,3]  arr.map(fn).filter(fn)  arr[1:3]
#{"key": val}  m.get(k)  m.set(k,v)
#[1,2,3]  s.add(v)  s.has(v)
```

## Concurrency (core differentiator)
```xray
let t = go someFunc(args)       // spawn coroutine
let result = await t             // wait for result
shared const ch = new Channel<int>(10)
ch.send(val); ch.recv()          // send/receive
select { msg from ch -> handle(msg); after 1000 -> timeout(); _ -> idle() }
shared const CFG = {...}         // immutable cross-coroutine sharing
```

## Safety: compile pass = concurrency safe
- shared const: zero-copy read across coroutines
- Channel: deep copy on send
- Function params to go: deep copy
- Regular let/const: cannot be captured by go closures

## Modules
```xray
import http; import json; import time
export fn helper() {}
```

## Testing
```xray
@test fn test_add() { assert_eq(1+1, 2) }
```

## Important notes
- `print()` outputs with newline (no `println`)
- Arrow function params MUST have type annotations
- new Channel<T>(n) at construction, Channel<T> in type position
- `this.x++` not allowed, use `this.x = this.x + 1`
- No quotes inside `${}` interpolation — use a variable
