---
id: cheatsheet
---
# Xray Language Cheatsheet

## Basics
```xray
let x = 1                         // type inferred as int
let name: string = "Alice"        // explicit type
let count: int                    // no initializer: zero value used
```
```xray
const PI = 3.14159
const MAX_LEN: int = 1024
```
```xray
fn add(a: int, b: int) -> int {
    return a + b
}

fn greet(name: string) -> () {         // explicit Unit
    print("Hi ${name}")
}

fn echo(x: int) {                       // omitted return type = ()
    print(x)
}
```
```xray
// ── Bare lambda: most concise, restricted to call-argument position ──
arr.map(x -> x * 2)
arr.filter(x -> x % 2 == 0)

// ── Arrow lambda: any position, supports multi-param and type annotation ──
let sum = arr.reduce((acc, x) -> acc + x, 0)    // no type
let double = (x: int) -> x * 2                   // typed
let add = (a: int, b: int) -> a + b              // multi-param

// ── fn expression: multi-statement body, return-type annotation, generics ──
let inc = fn(x: int) -> int {
    let y = x + 1
    return y
}
let identity = fn<T>(x: T) -> T { return x }     // generic
```

## Types
`int`, `float`, `string`, `bool`, `()` | `Array<T>`, `Map<K,V>`, `Set<T>`

`T?` for nullable values | `A | B` for unions | `Json`, `Bytes`, `BigInt`, `Channel<T>`

## Control flow
```xray
if (x > 0) {
    print("positive")
} else if (x == 0) {
    print("zero")
} else {
    print("negative")
}
```
```xray
for (item in [1, 2, 3]) { print(item) }
for (i in 0..n) { print(i) }                  // range iteration (half-open)
for (ch in "hello") { print(ch) }             // string characters (by codepoint)
for (key in someMap) { print(key) }           // single variable over Map → key
for (key in someJson) { print(key) }          // single variable over Json → key
for (day in Color) { print(day.name) }        // enum iteration (declaration order)
for (_ in 0..n) { count++ }                   // discard with placeholder
```
```xray
match (x) { 1 -> print("one"), _ -> print("other") }

match (action) {
    "start" -> {
        log.info("starting")
        start_engine()
    }
    "stop" -> stop_engine()
    _ -> log.warn("unknown")
}
```

## OOP and data declarations
```xray
class Dog extends Animal {
    constructor(name: string) {
        super(name)                    // **must** be the first statement (derived classes only)
    }

    override speak() -> string {         // override is optional but recommended
        return "woof"
    }
}
```
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
```xray
interface Shape {
    area() -> float
    perimeter() -> float
}

// Interface method return types may be omitted (default ())
interface Greeter {
    greet(name: string)             // same as greet(name: string) -> ()
    log()                           // no parameters, no return value
}

class Circle implements Shape {
    radius: float
    constructor(r: float) { this.radius = r }
    area() -> float { return 3.14 * this.radius * this.radius }
    perimeter() -> float { return 6.28 * this.radius }
}

// Implement multiple interfaces
class Logger implements Shape, Greeter {
    radius: float
    constructor(r: float) { this.radius = r }
    area() -> float { return 3.14 * this.radius * this.radius }
    perimeter() -> float { return 6.28 * this.radius }
    greet(name: string) { print("hello,", name) }
    log() { print("logging") }
}

fn describe(s: Shape) -> string {
    return "area=${s.area()}, perimeter=${s.perimeter()}"
}
```
```xray
enum Color { Red, Green, Blue }
Color.Red.value     // 0
Color.Blue.value    // 2

enum HttpStatus {
    OK = 200,
    NotFound = 404,
    InternalError = 500,
}

enum Direction { North = "N", South = "S", East = "E", West = "W" }
enum Flag      { On = true, Off = false }
enum Pi        { Approximate = 3.14, Better = 3.14159 }
```

## Collections
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
// Object/Json literal: identifier or string key + colon ':'
let data: Json = { name: "Alice", tags: ["a", "b"], age: 30 }
let user = { name: "Bob", age: 25 }       // default type is Json
data.name              // type: Json (field access returns Json)
data["name"]           // equivalent

// Field shorthand: when a field name matches a variable name
let name = "Alice"
let age = 30
let user = { name, age }                  // equivalent to { name: name, age: age }

// Map literal: `#{}` prefix + `:`
let m = #{"k1": 1, "k2": 2}           // type: Map<string, int>
```

## Concurrency
```xray
// Form 1: call an existing function
let t1 = go worker(0, channel)

// Form 2: call a lambda literal (inline logic + captured arguments)
let t2 = go fn(d: Json) -> int {
    return d.value * 2
}(payload)

// Form 3: block form (implicitly wrapped as a zero-argument lambda)
let t3 = go {
    return compute()
}
```
```xray
// single task
let task = go fetch("https://example.com")
let result = await task                    // yields the current coroutine until task completes

// await all: wait for all, returns the result array (in input order)
let t1 = go compute(2)
let t2 = go compute(3)
let t3 = go compute(4)
let results: Array<int> = await all [t1, t2, t3]
// also works on a variable directly, no brackets needed
let tasks = [t1, t2, t3]
let results2: Array<int> = await all tasks

// await any: wait for the first to complete, return its result; the others keep running
let first = await any [t1, t2, t3]

// await anySuccess: skip failing tasks; wait for the first successful one
let firstOk = await anySuccess [t1, t2, t3]
```
```xray
shared const ch = new Channel<int>(10)
ch.send(42)                             // blocking send
let v = ch.recv()                       // blocking receive (null after close + drain)

let sent = ch.trySend(99)               // non-blocking send: true / false
let (next, ok) = ch.tryRecv()           // non-blocking receive: value + ok flag
if (ok) { print(next) }

ch.close()
```
```xray
shared const ch1 = new Channel<int>(2)
shared const ch2 = new Channel<int>(2)

select {
    msg from ch1 -> { print("got from ch1:", msg) }      // receive arm
    msg from ch2 -> { print("got from ch2:", msg) }      // receive arm
    100  to   ch1 -> { print("sent 100 to ch1") }        // send arm
    _ -> { print("no channel ready") }                   // default arm (non-blocking)
}
```

## Safety: compile pass = concurrency safe
- `shared const`: zero-copy read across coroutines
- `Channel<T>`: communication with well-defined ownership semantics
- `move`: explicit ownership transfer for `shared let`
- ordinary `let` variables cannot be captured by `go` closures

## Modules
```xray
// 1. stdlib: bare identifier; without `as`, the alias equals the module name
import time
import datetime
import http as httpClient

// 2. third-party packages: owner/name form
import alice/utils
import bob/http_client as httpClient

// 3. file-path or directory-path: string literal, optional explicit alias, otherwise inferred from the trailing path segment
import "./modules/mod_a.xr" as a
import "../utils/string_utils.xr" as utils
import "models/user" as user

// 4. named imports: members may be renamed; the `from` operand may be a quoted path or a bare module name
import { readFile, writeFile as write } from io
import { publicFn } from "./modules/mod_a.xr"
```
```xray
// 1. export a declaration directly
export fn helper() { return }
export class MyClass {
    value: int
    constructor() { this.value = 1 }
}
export const VERSION = "1.0"

// 2. export an already-declared identifier (declare internally first, expose at the end)
fn _helper() -> string { return "..." }
fn publicFn() -> string { return _helper() }
export publicFn

// 3. re-export (with optional renaming)
export { getUser, getUserAge as getAge } from "./user"

// 4. wildcard re-export (forward all exports of another module)
export * from "./product"
```

## Testing
```xray
@test
fn test_addition() {
    assert_eq(1 + 1, 2)
}

@test
fn test_with_assertions() {
    let result = compute()
    assert_eq(result, 42)
    assert(result > 0)
}
```

## Important notes
- `print()` outputs with newline; there is no `println`
- arrow function parameter types may be inferred only when the context is known
- `new Channel<T>(n)` constructs a channel; `Channel<T>` is used in type position
- field increment such as `this.x++` is not supported; assign explicitly
- avoid unescaped quotes inside `${...}` interpolation
