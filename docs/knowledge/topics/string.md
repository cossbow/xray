---
id: string
title: String
aliases: [strings, string_methods, interpolation, template]
---
## Strings

### String interpolation
```xray
let name = "World"
print("Hello, ${name}!")     // template literal
```
Note: cannot use quotes inside `${}`. Use a variable instead.

### Methods
```xray
s.length; s.toLowerCase(); s.toUpperCase()
s.startsWith("He"); s.endsWith("!")
s.indexOf("sub"); s.includes("sub")
s.trim(); s.split(",")
s.replace("old", "new")
s[0:5]                        // substring slice
```

### Single-quoted strings
```xray
let raw = 'no ${interpolation} here'
```
