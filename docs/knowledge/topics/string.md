---
id: string
title: String
spec: #315-字符串插值
aliases: [strings, string_methods, interpolation, template]
---
## String

### String interpolation
```xray
"Hello, ${name}! Age: ${user.age + 1}"
```
Note: avoid unescaped quotes inside `${...}`; use a variable or escape the inner string.

### Slicing
```xray
arr[1:4]                // elements [1, 4)
arr[:3]                 // first 3
arr[2:]                 // from index 2 to the end
arr[:]                  // full slice (shallow copy)
str[0:5]                // string slice
```

### Common methods
- `s.length`
- `s.toLowerCase()` / `s.toUpperCase()`
- `s.startsWith(prefix)` / `s.endsWith(suffix)`
- `s.indexOf(sub)` / `s.includes(sub)`
- `s.trim()` / `s.split(sep)` / `s.replace(old, new)`
