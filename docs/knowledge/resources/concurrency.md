---
id: concurrency
---
# Xray Concurrency Model

## Golden Rule: If it compiles, it's concurrency-safe.

## Three ways to share data across coroutines:

### 1. Channel (communication)
```xray
shared const ch = new Channel<int>(10)
ch.send(value)           // deep copies pointer types
let val = ch.recv()      // null when closed
```

### 2. shared const (immutable sharing)
```xray
shared const CONFIG = { host: "localhost", port: 8080 }
// Zero-copy reads from any coroutine. Cannot modify.
```

### 3. Function parameters (deep copy)
```xray
go processData(myArray)  // myArray is deep-copied
```

## What you CANNOT do (compile errors):
```xray
let x = [1,2,3]
go fn() { print(x) }()  // COMPILE ERROR: cannot capture 'x'
// Fix: pass as parameter
```

## Move semantics:
```xray
shared let data = [1,2,3]
go fn() { let local = move data }()
print(data)  // COMPILE ERROR: 'data' already moved
```

## Structured concurrency:
```xray
scope {
    go taskA()
    go taskB()
}  // waits for all goroutines to complete
```
