---
id: operators
title: Operators
spec: #17-操作符与-token
aliases: [operator, arithmetic, comparison, logical, bitwise]
---
## Operators

### Operator groups
### Arithmetic: `+`, `-`, `*`, `/`, `%`, `**` (power)
### Comparison: `==`, `!=`, `<`, `>`, `<=`, `>=`
### Logical: `&&`, `||`, `!`
### Bitwise: `&`, `|`, `^`, `~`, `<<`, `>>`
### Assignment: `=`, `+=`, `-=`, `*=`, `/=`, `%=`
### Increment/Decrement: `i++`, `i--` (variables only, not fields)
### Null coalescing: `x ?? default`
### Optional chaining: `obj?.field`
### Spread: `...arr`
### Type: `typeof(x)`, `x is Type`

### Range and null-safe operators
```xray
0..10                  // 0..10, left-closed right-open (includes 0, excludes 10)
let r = 1..100
let n = 10
for (i in 0..n) { print(i) }
```
```xray
let v = nullable_expr ?? default_value
```
```xray
let len = name?.length          // returns null when name is null
let item = arr?.[0]             // optional index
```
