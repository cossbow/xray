---
id: control_flow
title: Control Flow
aliases: [if, else, while, for, loop, match, switch, try, catch, throw, error_handling]
---
## Control Flow

### if / else
```xray
let condition = true
let other = false
if (condition) { print("yes") } else if (other) { print("maybe") } else { print("no") }
```

### while
```xray
let i = 0
while (i < 3) { i++ }
```

### for (C-style)
```xray
for (let i = 0; i < 10; i++) { print(i) }
```

### for-in (range)
```xray
let array = [1, 2, 3]
for (i in 0..10) { print(i) }       // 0 to 9
for (item in array) { print(item) } // iterate array
```

### match (pattern matching)
```xray
let result = match (x) {
    1 -> "one",
    2, 3 -> "two or three",
    4..10 -> "four to ten",
    n if (n < 0) -> "negative",
    _ -> "other"
}
```

### try / catch / finally
```xray
try { throw "error message" } catch (e) { print(e) } finally { print("done") }
```
