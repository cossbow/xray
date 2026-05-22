---
id: functions
title: Functions
spec: #52-fn-函数声明
aliases: [fn, function, arrow, closure, lambda, params]
---
## Functions

### Named functions (parameters MUST have type annotations)
```xray
fn add(a: int, b: int) -> int {
    return a + b
}
```

### Arrow functions (MUST have type annotations)
```xray
let double = (x: int) -> x * 2
let clamp = fn(x: int, lo: int, hi: int) -> int {
    if (x < lo) { return lo }
    if (x > hi) { return hi }
    return x
}
```

### Default parameters
```xray
fn greet(name: string, greeting: string = "Hello") -> string {
    return "${greeting}, ${name}!"
}
```

### Multiple return values
```xray
fn divmod(a: int, b: int) -> (int, int) {
    return (a / b, a % b)
}
let (q, r) = divmod(17, 5)
```

### Rest parameters (no type annotation on rest param)
```xray
fn sum(...nums: int) -> int {
    let total = 0
    for (let i = 0; i < nums.length; i++) { total = total + nums[i] }
    return total
}
```

### Higher-order functions
```xray
fn apply(f: (int) -> int, x: int) -> int { return f(x) }
```
