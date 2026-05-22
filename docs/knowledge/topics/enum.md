---
id: enum
title: Enum
spec: #56-enum-声明
aliases: [enums, enumeration, adt, payload, variant, Result, Ok, Err]
---
## Enum

### Simple enum
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

### Properties
```xray
Color.Red.name        // "Red"          variant name (string)
Color.Red.value       // 0              backing value
Color.Red.ordinal     // 0              declaration index (int, zero-based)
Color.Red.toString()  // "Color.Red"    "<EnumName>.<VariantName>" format
```

### Iteration
```xray
for (c in Color) { print(c.name) }        // "Red" "Green" "Blue"
```

### Pattern matching
```xray
match (event) {
    NetEvent.Connected            -> print("connected"),
    NetEvent.Disconnected(reason) -> print("by:", reason),
    NetEvent.DataReceived(b)      -> process(b),
    NetEvent.Error(code, msg)     -> log.error(code, msg),
}
```

### ADT-style payload enum
```xray
// positional payload
enum Result<T, E> {
    Ok(T),
    Err(E),
}

// named-field payload (recommended for readability)
enum NetEvent {
    Connected,
    Disconnected(reason: string),
    DataReceived(bytes: Bytes),
    Error(code: int, message: string),
}

// state machine
enum ConnState {
    Idle,
    Connecting(retry: int),
    Connected(peer: string, since: int),
    Failed(reason: string),
}

// AST nodes
enum Expr {
    Number(int),
    Binary(op: string, left: Expr, right: Expr),
    Call(name: string, args: Array<Expr>),
}
```
