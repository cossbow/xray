---
id: coroutine
title: Coroutines & Concurrency
spec: #10-并发与协程-concurrency
aliases: [go, goroutine, goroutines, await, task, async, concurrency, scope, select, defer]
---
## Coroutines & Concurrency

### go — spawn a coroutine
```xray
let task = go compute(42)      // returns a Task
let result = await task         // wait for result
```

### await all / await any
```xray
let results = await all [t1, t2, t3]   // wait for all
let first = await any [t1, t2]         // first to finish
```

### scope — structured concurrency
```xray
scope {
    go taskA()
    go taskB()
}  // waits for ALL goroutines to finish
```

### select — multiplex channels
```xray
select {
    msg from ch1 -> { handle(msg) }
    msg from ch2 -> { handle(msg) }
    after 1000 -> { print("timeout") }
}
```

### defer — LIFO cleanup
```xray
fn process() {
    defer { cleanup() }   // runs when function exits
    doWork()
}
```
