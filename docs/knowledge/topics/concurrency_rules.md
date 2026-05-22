---
id: concurrency_rules
title: Concurrency Safety Rules
spec: #1011-并发安全模型
aliases: [shared_rules, sharing, move, safety, concurrent]
---
## Concurrency Safety Rules

### Sharing model
- `shared const` is immutable and can be shared across coroutines without copying
- `shared let` represents exclusive mutable ownership and must be transferred with `move`
- ordinary local variables cannot be captured by `go` closures; pass data explicitly

### Move into coroutine
```xray
shared let data = { value: 10 }
let task = go fn(d: Json) -> int {
    return d.value + 1
}(move data)        // transfer data ownership to the coroutine; data is unusable afterwards
```

### Move ownership
```xray
shared let buf = new Bytes(1024 * 1024)

// hand off to a coroutine
let t = go fn(b: Bytes) -> int {
    return process(b)
}(move buf)
// compile error: buf has been moved
// print(buf.length)

// hand off to a channel
shared const ch = new Channel<Bytes>(1)
shared let payload = new Bytes(4096)
ch.send(move payload)
// compile error: payload has been moved
```

### Channels
```xray
shared const ch = new Channel<int>(10)
ch.send(42)                             // blocking send
let v = ch.recv()                       // blocking receive (null after close + drain)

let sent = ch.trySend(99)               // non-blocking send: true / false
let (next, ok) = ch.tryRecv()           // non-blocking receive: value + ok flag
if (ok) { print(next) }

ch.close()
```
