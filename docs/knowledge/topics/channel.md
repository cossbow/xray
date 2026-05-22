---
id: channel
title: Channel
aliases: [channels, chan, Channel, send, recv, buffered]
---
## Channel

Channels are the primary inter-coroutine communication mechanism.

### Declaration (MUST be const)
```xray
shared const ch = new Channel<int>(10)
```
`let ch = new Channel<int>(10)` is a **compile error** for coroutine sharing.

### Operations
```xray
shared const ch = new Channel<int>(10)
let value = 42
ch.send(value)                   // blocking send
let val = ch.recv()              // blocking receive (null on closed)
let (next, ok) = ch.tryRecv()    // non-blocking receive
ch.close()                       // close channel
```

### Function parameter type
```xray
fn producer(ch: Channel<int>) {  // use Channel<T> in type position
    ch.send(42)
}
```

### Concurrency rules
- Channels can be captured by `go` closures (exception to shared rule)
- Values sent through channel are deep-copied (pointer types)
- Channel is reference-counted on system heap
