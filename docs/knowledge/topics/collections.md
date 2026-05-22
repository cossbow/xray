---
id: collections
title: Collections
spec: #24-复合类型
aliases: [array, map, set, list, dict, dictionary, json, bytes]
---
## Collections

### Array
```xray
let a: Array<int> = [1, 2, 3]
let b = [1, 2, 3]                // inferred as Array<int>
let c: Array<string> = []         // explicit empty array
```

### Map
```xray
let m: Map<string, int> = #{"a": 1, "b": 2}
let m2 = #{"a": 1, "b": 2}
let empty = #{}                                     // empty Map

m["c"] = 3                                          // insert / update
let v = m["a"]                                      // lookup; returns null if absent
```

### Set
```xray
let s: Set<int> = #[1, 2, 3]
```

### Json object
```xray
// Object/Json literal: identifier or string key + colon ':'
let data: Json = { name: "Alice", tags: ["a", "b"], age: 30 }
let user = { name: "Bob", age: 25 }       // default type is Json
data.name              // type: Json (field access returns Json)
data["name"]           // equivalent

// Field shorthand: when a field name matches a variable name
let name = "Alice"
let age = 30
let user = { name, age }                  // equivalent to { name: name, age: age }

// Map literal: `#{}` prefix + `:`
let m = #{"k1": 1, "k2": 2}           // type: Map<string, int>
```

### Other collection-like types
- `Bytes` is a typed byte buffer backed by contiguous memory
- `Channel<T>` is the coroutine communication container
- `WeakMap` / `WeakSet` hold weak references to heap objects
