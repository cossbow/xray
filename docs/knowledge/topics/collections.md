---
id: collections
title: Collections
aliases: [array, map, set, list, dict, dictionary, json, bytes]
---
## Collections

### Array
```xray
let arr = [1, 2, 3, 4, 5]
arr.push(6); arr.pop()
arr[1:3]                       // slice [2, 3]
arr.map((x: int) -> x * 2)
arr.filter((x: int) -> x > 2)
arr.reduce((acc: int, x: int) -> acc + x, 0)
arr.find((x: int) -> x > 3)
arr.sort(); arr.reverse()
arr.indexOf(3); arr.includes(5)
arr.join(", ")
```

### Map
```xray
let m = #{"a": 1, "b": 2}
m.get("a"); m.set("c", 3); m.delete("b")
m.has("a"); m.length
m.keys(); m.values()
```

### Set
```xray
let s = #[1, 2, 3]
s.add(4); s.delete(1)
s.has(3); s.length
```

### Json (dynamic object)
```xray
import json
let obj: Json = { name: "Alice", age: 30 }
obj.name                      // field access
obj.missing                   // returns null
json.keys(obj)                // ["name", "age"]
json.stringify(obj)           // JSON string
json.parse('{"x":1}')       // parse string to Json
```
