---
id: enum
title: Enum
aliases: [enums, enumeration]
---
## Enums

```xray
enum Color { Red, Green, Blue }
print(Color.Red.name)       // "Red"
print(Color.Red.value)      // 0
print(Color.memberCount)    // 3

// Custom values
enum HttpStatus {
    OK = 200,
    NotFound = 404,
    ServerError = 500
}

// Iteration
for (c in Color) { print(c.name) }

// Match
match (status) {
    HttpStatus.OK -> "Success",
    HttpStatus.NotFound -> "Not Found",
    _ -> "Unknown"
}
```
