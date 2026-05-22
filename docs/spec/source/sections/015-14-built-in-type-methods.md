---
id: spec.14_built_in_type_methods
order: 015
---

<!-- xr-spec:cn -->
---

## 14. 内置类型方法 (Built-in Type Methods)

> 真值源：prelude / analyzer / runtime 中的内置类型注册与方法定义。
> MCP knowledge 只消费生成后的 analyzer metadata，不独立维护内置类型方法签名。

本节给出每种类型的**方法索引**（按主题分组）。具体签名、参数说明、行为细节以实现代码为准。

### 14.1 `int` 方法

| 方法 | 签名 | 说明 |
|--|--|--|
| `abs()` | `() -> int` | 绝对值 |
| `toString()` | `() -> string` | 十进制字符串 |
| `toBigInt()` | `() -> BigInt` | 转 BigInt |
| `toFloat()` | `() -> float` | 转 float |
| `toHex()` | `() -> string` | 十六进制字符串 |
| `max(other)` / `min(other)` | `(int) -> int` | 双值最值 |
| `floor()` / `ceil()` / `round()` | `() -> int` | 对 int 返回自身 |
| `sqrt()` | `() -> float` | 平方根 |
| `pow(exp)` | `(float) -> float` | 幂运算 |

### 14.2 `float` 方法

| 方法 | 签名 | 说明 |
|--|--|--|
| `abs()` | `() -> float` | 绝对值 |
| `toString()` | `() -> string` | 字符串化 |
| `toFixed(decimals?)` | `(int?) -> string` | 固定位数小数字符串 |
| `toInt()` | `() -> int` | 转 int |
| `floor()` / `ceil()` / `round()` | `() -> int` | 取整 |
| `sqrt()` | `() -> float` | 平方根 |
| `pow(exp)` | `(float) -> float` | 幂运算 |

### 14.3 `BigInt` 方法

| 方法 | 签名 | 说明 |
|--|--|--|
| `abs()` | `() -> BigInt` | 绝对值 |
| `toString()` | `() -> string` | 字符串化 |
| `sign()` | `() -> int` | -1 / 0 / 1 |
| `isZero()` / `isNegative()` / `isPositive()` | `() -> bool` | 符号判断 |
| `toInt()` | `() -> int?` | 无法表示时返回 null |
| `toFloat()` | `() -> float` | 转 float |

### 14.4 `bool` 方法

| 方法 | 签名 | 说明 |
|--|--|--|
| `toString()` | `() -> string` | 返回 `"true"` 或 `"false"` |

### 14.5 `string` 方法

| 成员 | 类型 / 说明 |
|--|--|
| `length` | 字符串长度属性 |
| `charAt(i)` | 返回指定位置字符 |
| `charCodeAt(i)` | 返回码点 |
| `concat(...others)` | 拼接字符串 |
| `includes(s)` | 是否包含子串 |
| `indexOf(s)` / `lastIndexOf(s)` | 查找子串 |
| `slice(start, end?)` / `substring(start, end?)` / `substr(start, len?)` | 子串 |
| `toLowerCase()` / `toUpperCase()` | 大小写转换 |
| `trim()` / `trimStart()` / `trimEnd()` | 去空白 |
| `split(sep, limit?)` | 分割为 `Array<string>` |
| `replace(from, to)` / `replaceAll(from, to)` | 替换 |
| `repeat(n)` | 重复 |
| `startsWith(s)` / `endsWith(s)` | 前缀/后缀判断 |
| `padStart(len, pad?)` / `padEnd(len, pad?)` | 填充 |
| `match(pattern)` | 正则匹配 |
| `iterator()` / `entriesIterator()` / `entries()` | 迭代协议 |

### 14.6 `Bytes`

`Bytes` 是 prelude 类型，构造由 `Bytes(n)` / `Bytes(n, fill)` 等内置路径处理。字符串转换和编码类操作优先使用 `encoding` / `base64` 模块。当前没有单独的 `stdlib/types/bytes.xr` 声明；工具不要假设存在完整 Array 同构 API。

### 14.7 `Array<T>` 方法

| 成员 | 类型/说明 |
|--|--|
| `length` | `int` 属性 |
| `arr[i]` / `arr[i] = v` | 下标读写 |
| `push(x)` / `pop()` | 尾部增删 |
| `shift()` / `unshift(x)` | 头部增删 |
| `slice(start?, end?)` | 切片 |
| `splice(start, deleteCount, ...items)` | 原地增删 |
| `concat(...arrays)` | 拼接 |
| `indexOf(x)` / `includes(x)` | 查找 |
| `join(sep?)` | 拼接为字符串 |
| `reverse()` / `sort(cmp?)` | 原地重排 |
| `map(fn)` / `filter(fn)` / `reduce(fn, init)` | 函数式处理 |
| `forEach(fn)` / `find(fn)` / `findIndex(fn)` / `every(fn)` / `some(fn)` | 遍历与谓词 |
| `flat(depth?)` / `fill(v, start?, end?)` / `copyWithin(target, start, end?)` | 数组工具 |
| `iterator()` / `entriesIterator()` / `entries()` | 迭代协议 |

### 14.8 `Map<K, V>` 方法

| 成员 | 类型/说明 |
|--|--|
| `length` | `int` 属性 |
| `m[k]` / `m[k] = v` | 下标读写 |
| `get(k)` / `set(k, v)` | 读取/写入 |
| `has(k)` / `delete(k)` / `clear()` | 查询与删除 |
| `keys()` / `values()` / `entries()` | 返回键、值、键值对 |
| `forEach(fn)` | 遍历 |
| `iterator()` / `entriesIterator()` | 迭代协议 |

**Map 字面量**：`#{"k1": v1, "k2": v2}` 或 `#{}`；使用 `:`，靠 `#` 前缀区别于 Object/Json 字面量。

### 14.9 `Set<T>` 方法

| 成员 | 类型/说明 |
|--|--|
| `length` | `int` 属性 |
| `add(x)` / `has(x)` / `delete(x)` | 插入、查询、删除 |
| `clear()` | 清空 |
| `values()` | 返回 `Array<T>` |
| `forEach(fn)` | 遍历 |
| `iterator()` | 迭代协议 |

**Set 字面量**：`#[1, 2, 3]` 或 `#[]`。

### 14.10 `Channel<T>` 方法

| 成员 | 类型/说明 |
|--|--|
| `send(v)` | 阻塞发送；channel 已关闭时抛异常 |
| `recv()` | 阻塞接收；关闭且缓冲为空时返回 `null` |
| `trySend(v)` | 非阻塞发送，返回 bool |
| `tryRecv()` | 非阻塞接收，返回 `(T, bool)` |
| `sendTimeout(v, ms)` | 带超时发送，超时或关闭返回 false |
| `recvTimeout(ms)` | 带超时接收，返回 `(T, bool)` |
| `close()` | 关闭 channel |
| `isClosed` / `isClosed()` | 关闭状态；运行时属性和方法均支持 |

> `stdlib/types/channel.xr` 仍声明 `closed` 属性，但运行时符号表和 VM 分派使用 `isClosed`；该声明漂移已记录为已知问题。

### 14.11 `Json`

`Json` 是动态结构化数据类型。普通字段访问使用 `j.field` / `j["field"]`；通用查询和编解码通过 `Json` 静态函数完成，避免与用户字段名冲突。

| 静态函数 | 说明 |
|--|--|
| `Json.keys(obj)` / `Json.values(obj)` / `Json.entries(obj)` | Object 字段枚举 |
| `Json.has(obj, key)` | 字段存在性 |
| `Json.get(obj, key, default?)` | 字段读取，不存在返回 default 或 null |
| `Json.size(obj)` | 字段数量 |
| `Json.isEmpty(obj)` | 是否为空 |
| `Json.parse(s)` / `Json.tryParse(s)` / `Json.isValid(s)` | JSON 解析与校验 |
| `Json.stringify(value, indent?)` | 序列化 |

**字面量**：`{ name: "alice", age: 30 }`，动态类型为 `Json`。如需 sealed 对象，用 `type T = { name: string, age: int }` 标注。

### 14.12 `Range`

`a..b` 是半开区间 `[a, b)`，用于表达式和 `for-in`。常见成员为 `start`、`end`、`length`、`includes(x)`、`toArray()`、`toString()`。

### 14.13 `DateTime`

通过 `import datetime` 获得工厂函数：`now`、`utc`、`create`、`createUTC`、`fromTimestamp`、`fromTimestampMs`、`parse`、`offset`。`DateTime` 实例由 prelude 注册，无需 import 类型名。

| 成员 | 类型/说明 |
|--|--|
| `year` / `month` / `day` | 日期分量属性 |
| `hour` / `minute` / `second` / `millisecond` | 时间分量属性 |
| `weekday` / `yearday` / `timestamp` | 派生属性 |
| `toString()` / `format(pattern?)` / `toISOString()` | 格式化 |
| `add(amount, unit)` / `diff(other, unit?)` | 日期运算 |
| `toUTC()` / `toLocal()` | 时区转换 |
| `isBefore(other)` / `isAfter(other)` / `equals(other)` | 比较 |
| `isLeapYear()` / `daysInMonth()` | 日历查询 |

### 14.14 `Regex`

| 方法 | 说明 |
|--|--|
| `test(s)` | 是否匹配 |
| `find(s)` | 首个匹配 |
| `findAll(s)` | 所有匹配 |
| `replace(s, replacement)` | 替换 |
| `split(s)` | 分割 |

### 14.15 `StringBuilder`

| 方法 | 说明 |
|--|--|
| `length` | 当前长度属性 |
| `append(s)` | 追加并返回自身 |
| `toString()` | 输出字符串 |
| `clear()` | 清空并返回自身 |

### 14.16 `Exception`

内置 `Exception` 类包含 `message`、`stack`、`cause`、`code`、`data` 字段，构造函数 `constructor(message: string = "", cause: Exception? = null)`，以及 `toString()`。

### 14.17 `Task<T>` / `EnumValue` / `EnumType`

`Task<T>` 属性：`done`、`cancelled`、`result`、`error`；方法：`cancel()`。`EnumValue` 属性：`name`、`value`、`ordinal`，方法：`toString()`。`EnumType` 属性：`name`、`memberCount`，方法：`getMember(name)`。

### 14.18 其他 prelude 类型（`Logger` / `NetConn` / `NetListener`）

这些类型由 prelude 注册，实例由 `log` / `net` 等模块工厂函数构造。完整运行时能力以对应 stdlib 模块为准。
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 14. Built-in Type Methods

### 14.1 Numeric and Bool Methods

`int`: `abs`, `toString`, `toBigInt`, `toFloat`, `toHex`, `max`, `min`, `floor`, `ceil`, `round`, `sqrt`, `pow`.

`float`: `abs`, `toString`, `toFixed`, `toInt`, `floor`, `ceil`, `round`, `sqrt`, `pow`.

`BigInt`: `abs`, `toString`, `sign`, `isZero`, `isNegative`, `isPositive`, `toInt`, `toFloat`.

`bool`: `toString`.

### 14.2 `string`

Supported members include `length`, `charAt`, `charCodeAt`, `concat`, `includes`, `indexOf`, `lastIndexOf`, `slice`, `substring`, `substr`, `toLowerCase`, `toUpperCase`, `trim`, `trimStart`, `trimEnd`, `split`, `replace`, `replaceAll`, `repeat`, `startsWith`, `endsWith`, `padStart`, `padEnd`, `match`, `iterator`, `entriesIterator`, and `entries`.

### 14.3 `Bytes`

`Bytes` is a prelude byte-buffer type. Construction is handled by builtin paths such as `Bytes(n)` and `Bytes(n, fill)`. Encoding/decoding helpers live in modules such as `encoding` and `base64`.

### 14.4 `Array<T>`

Supported members include `length`, indexing, `push`, `pop`, `shift`, `unshift`, `slice`, `splice`, `concat`, `indexOf`, `includes`, `join`, `reverse`, `sort`, `map`, `filter`, `reduce`, `forEach`, `find`, `findIndex`, `every`, `some`, `flat`, `fill`, `copyWithin`, `iterator`, `entriesIterator`, and `entries`.

### 14.5 `Map<K, V>`

Supported members include `length`, indexing, `get`, `set`, `has`, `delete`, `clear`, `keys`, `values`, `entries`, `forEach`, `iterator`, and `entriesIterator`.

Map literals are written with a `#` prefix and colon entries:

```xray
let m = #{"k": 1}
```

### 14.6 `Set<T>`

Supported members include `length`, `add`, `has`, `delete`, `clear`, `values`, `forEach`, and `iterator`.

Set literals use `#[...]`.

### 14.7 `Json`

Json field data is accessed with normal field/index syntax. Generic utility functions are static methods on `Json`:

`keys`, `values`, `entries`, `has`, `get`, `size`, `isEmpty`, `parse`, `tryParse`, `isValid`, and `stringify`.

### 14.8 `Channel<T>`

Methods: `send`, `recv`, `trySend`, `tryRecv`, `sendTimeout`, `recvTimeout`, `close`, and `isClosed`/`isClosed()`.

### 14.9 `DateTime`

`DateTime` instances provide component properties (`year`, `month`, `day`, `hour`, `minute`, `second`, `millisecond`, `weekday`, `yearday`, `timestamp`) and methods (`toString`, `format`, `toISOString`, `add`, `diff`, `toUTC`, `toLocal`, `isBefore`, `isAfter`, `equals`, `isLeapYear`, `daysInMonth`).

The `datetime` module exports factory functions: `now`, `utc`, `create`, `createUTC`, `fromTimestamp`, `fromTimestampMs`, `parse`, and `offset`.

### 14.10 `Regex`

Methods: `test`, `find`, `findAll`, `replace`, and `split`.

### 14.11 `StringBuilder`

Members: `length`, `append`, `toString`, and `clear`.

### 14.12 `Exception`, `Task`, and Enum Runtime Types

`Exception` exposes `message`, `stack`, `cause`, `code`, `data`, and `toString()`.

`Task<T>` exposes `done`, `cancelled`, `result`, `error`, and `cancel()`.

`EnumValue` exposes `name`, `value`, `ordinal`, and `toString()`. `EnumType` exposes `name`, `memberCount`, and `getMember(name)`.
<!-- /xr-spec:en -->
