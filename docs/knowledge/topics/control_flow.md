---
id: control_flow
title: Control Flow
spec: #4-语句-statements
aliases: [if, else, while, for, loop, match, switch, try, catch, throw, error_handling]
---
## Control Flow

### if / else
```xray
if (x > 0) {
    print("positive")
} else if (x == 0) {
    print("zero")
} else {
    print("negative")
}
```

### while
```xray
let i = 0
while (i < 10) {
    print(i)
    i++
}
```

### for / for-in
```xray
for (let i = 0; i < 10; i++) {
    print(i)
}
for (let j = 100; j > 90; j--) {
    print(j)
}
```
```xray
for (item in [1, 2, 3]) { print(item) }
for (i in 0..n) { print(i) }                  // range iteration (half-open)
for (ch in "hello") { print(ch) }             // string characters (by codepoint)
for (key in someMap) { print(key) }           // single variable over Map → key
for (key in someJson) { print(key) }          // single variable over Json → key
for (day in Color) { print(day.name) }        // enum iteration (declaration order)
for (_ in 0..n) { count++ }                   // discard with placeholder
```
```xray
// Form A: two bare identifiers (more common)
for (k, v in someMap) { print("${k}=${v}") }     // Map → (key, value)
for (i, e in someArray) { print("${i}: ${e}") }  // Array → (index, element)
for (i, c in "hello") { print("${i}:${c}") }     // string → (index, char)

// Form B: tuple-parenthesized (pairs well with .entries())
for ((i, e) in someArray.entries()) { print("${i}=${e}") }
for ((i, c) in "hi".entries()) { print("${i}-${c}") }
```

### match
```xray
match (x) { 1 -> print("one"), _ -> print("other") }

match (action) {
    "start" -> {
        log.info("starting")
        start_engine()
    }
    "stop" -> stop_engine()
    _ -> log.warn("unknown")
}
```

### try / catch / finally / throw
```xray
try { throw new Exception("inline") } catch (e) { print(e.message) }

try {
    risky()
} catch (e) {
    log.error("failed:", e.message)
} finally {
    cleanup()
}

throw new Exception("error message")        // ✅ Exception-derived
throw new HttpError(500, "internal")        // ✅ user Exception subclass
// throw "msg"                              // ❌ E0370: must be Exception-derived
```

### defer
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
