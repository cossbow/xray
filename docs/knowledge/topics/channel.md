---
id: channel
title: Channel
spec: #105-channel
aliases: [channels, chan, Channel, send, recv, buffered]
---
## Channel

Channels are the primary inter-coroutine communication mechanism.

### Declaration (MUST be const)
```xray
shared const ch  = new Channel<int>(10)    // buffered, capacity = 10
shared const ch0 = new Channel<int>(0)     // unbuffered (synchronous handshake)
shared const cha = new Channel(3)          // element type inferred from the first send
```
`let ch = new Channel<int>(10)` is a **compile error** for coroutine sharing.

### Operations
```xray
shared const ch = new Channel<int>(10)
ch.send(42)                             // blocking send
let v = ch.recv()                       // blocking receive (null after close + drain)

let sent = ch.trySend(99)               // non-blocking send: true / false
let (next, ok) = ch.tryRecv()           // non-blocking receive: value + ok flag
if (ok) { print(next) }

ch.close()
```

### Function parameter type
```xray
fn producer(ch: Channel<int>) {
    ch.send(42)
}
```

### Concurrency rules
- Channels can be captured by `go` closures (exception to shared rule)
- Values sent through channel are deep-copied (pointer types)
- Channel is reference-counted on system heap
