---
id: concurrency
---
# Xray Concurrency Model

## Golden rule
If it compiles, it is concurrency-safe.

## Channels: communication
```xray
shared const ch  = new Channel<int>(10)    // buffered, capacity = 10
shared const ch0 = new Channel<int>(0)     // unbuffered (synchronous handshake)
shared const cha = new Channel(3)          // element type inferred from the first send
```
```xray
shared const ch = new Channel<int>(10)
ch.send(42)                             // blocking send
let v = ch.recv()                       // blocking receive (null after close + drain)

let sent = ch.trySend(99)               // non-blocking send: true / false
let (next, ok) = ch.tryRecv()           // non-blocking receive: value + ok flag
if (ok) { print(next) }

ch.close()
```

## Immutable sharing
```xray
shared const CONFIG = { host: "localhost", port: 8080 }
shared const PRIMES = [2, 3, 5, 7, 11]
```

## Deep-copy and explicit move
```xray
// Form 1: call an existing function
let t1 = go worker(0, channel)

// Form 2: call a lambda literal (inline logic + captured arguments)
let t2 = go fn(d: Json) -> int {
    return d.value * 2
}(payload)

// Form 3: block form (implicitly wrapped as a zero-argument lambda)
let t3 = go {
    return compute()
}
```
```xray
shared let data = { value: 10 }
let task = go fn(d: Json) -> int {
    return d.value + 1
}(move data)        // transfer data ownership to the coroutine; data is unusable afterwards
```
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

## Structured concurrency
```xray
// lexical scope use
let x = 1
scope {
    let x = 10            // shadow the outer x; in effect inside the block
    print(x)              // 10
}
print(x)                  // 1

// structured concurrency use (with go)
scope {
    go worker_a()
    go worker_b()
    // before the block exits, both a/b are awaited; an exception in either does not affect siblings
}
```
```xray
// linked scope: failure propagation
try {
    linked scope {
        go ok_worker()
        go failing_worker()         // throws
    }
} catch (e) {
    print("caught:", e)              // hits this branch
}

// supervisor scope: collect errors
let errors = supervisor scope {
    go failing("error1")
    go failing("error2")
    go ok()
}
print(errors.length)                 // 2 (only the failures are counted)
```

## Select
```xray
shared const ch1 = new Channel<int>(2)
shared const ch2 = new Channel<int>(2)

select {
    msg from ch1 -> { print("got from ch1:", msg) }      // receive arm
    msg from ch2 -> { print("got from ch2:", msg) }      // receive arm
    100  to   ch1 -> { print("sent 100 to ch1") }        // send arm
    _ -> { print("no channel ready") }                   // default arm (non-blocking)
}
```
