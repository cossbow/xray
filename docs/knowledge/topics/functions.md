---
id: functions
title: Functions
spec: #52-fn-函数声明
aliases: [fn, function, arrow, closure, lambda, params]
---
## Functions

### Named functions
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

### Default parameters
```xray
fn connect(host: string, port: int = 8080, tls: bool = false) {
    print(host, port, tls)
}

connect("localhost")              // port=8080, tls=false
connect("localhost", 443)         // tls=false
connect("localhost", 443, true)
```

### Multiple return values
```xray
fn divmod(a: int, b: int) -> (int, int) {
    return (a / b, a % b)
}

let (q, r) = divmod(17, 5)
let result = divmod(10, 3)        // result has type (int, int)
```

### Rest parameters
```xray
fn sum(...nums: int) -> int {
    let total = 0
    for (n in nums) { total += n }
    return total
}

sum(1, 2, 3)        // total = 6
```

### Lambda forms
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

### Higher-order functions
- Function types use `fn(T1, T2) -> R` in type position
- Function values can be passed to callbacks such as `map`, `filter`, and `reduce`
- Closure capture rules are described in the scoping and concurrency sections
