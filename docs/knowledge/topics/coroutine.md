---
id: coroutine
title: Coroutines & Concurrency
spec: #10-并发与协程-concurrency
aliases: [go, goroutine, goroutines, await, task, async, concurrency, scope, select, defer]
---
## Coroutines & Concurrency

### go — spawn a coroutine
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

### await
```xray
// single task
let task = go fetch("https://example.com")
let result = await task                    // yields the current coroutine until task completes

// await all: wait for all, returns the result array (in input order)
let t1 = go compute(2)
let t2 = go compute(3)
let t3 = go compute(4)
let results: Array<int> = await all [t1, t2, t3]
// also works on a variable directly, no brackets needed
let tasks = [t1, t2, t3]
let results2: Array<int> = await all tasks

// await any: wait for the first to complete, return its result; the others keep running
let first = await any [t1, t2, t3]

// await anySuccess: skip failing tasks; wait for the first successful one
let firstOk = await anySuccess [t1, t2, t3]
```

### Task handle
```xray
let t = go fetch(url)
if (!t.done) { /* still running */ }
let r = await t

// read properties directly (no await required)
print(t.done, t.cancelled, t.result, t.error)
```

### scope — structured concurrency
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

### select — multiplex channels
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

### yield — cooperative safepoint
```xray
for (i in 0..1000) {
    do_chunk(i)
    yield                       // explicit safepoint, lets other coroutines run
}
```
```xray
yield                       // yield execution
```

### defer — LIFO cleanup
```xray
fn read_file(path: string) -> string {
    let f = open(path)
    defer f.close()                  // always runs before the function returns
    return f.readAll()
}

fn process() {
    defer {                          // block form
        log.info("done")
        cleanup()
    }
    do_work()
}
```
