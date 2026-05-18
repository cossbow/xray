# xray 语言语法规范

## 1. 字面量

```xray
42; 0xFF; 0b1010; 0o77; 1_000_000     // 整数
3.14; 1.0e10; 2.5E-3                   // 浮点
123n                                    // BigInt
"hello"; 'world'                        // 字符串
"Hello, ${name}! ${1 + 2}"             // 插值（双引号/单引号均可）
r"C:\path\to\file"                      // 原始字符串（不处理转义）
/pattern/flags                          // 正则 (flags: g,i,m,s)
true; false; null                       // 布尔/空
```

## 2. 变量声明

```xray
let x = 1                    // 可变（类型推断）
const PI = 3.14159           // 常量（不可重新赋值）
let count: int = 0           // 带类型注解
let name: string             // 有注解无初值 → 零值 ""
shared const CFG = {...}     // 跨协程不可变共享（全局堆，零拷贝读）
shared let data = [...]      // 跨协程可变独占（必须配合 move 传递所有权）

// 数组解构
let [a, b, c] = [1, 2, 3]
let [first, , third] = [10, 20, 30]   // 跳过元素

// 对象解构
let { name, age } = { name: "Alice", age: 30 }

// 元组解构（多返回值）
let (result, ok) = divide(10, 2)
```

## 3. 类型

```xray
// 基本: int, float, string, bool, void
// 精确: int8..int64, uint8..uint64, float32, float64
// 容器: Array<T>, Map<K,V>, Set<T>, Channel<T>
// 特殊: Json, JsonValue, BigInt, Range, DateTime, Bytes, StringBuilder, WeakMap, WeakSet
// 可空: int?, string?, MyClass?
// Union: int | string, int | string | bool  (最多 6 个成员；超限为编译错误)
// 注意: `any` 类型已移除；动态值请用 Json 或具体 Union
// 说明: `unknown` 是编译器内部的未解析/不精确类型占位，不作为用户层动态类型使用
// 函数: fn(int, int): int
// 元组返回: fn divide(a: int, b: int): (int, bool)
// 别名: type Result = int | string
//        type Point = { x: float, y: float }
//        type Mapper = fn(int): int

// JsonValue（内置 union，Json 字段动态类型）
// JsonValue = (bool | int | float | string | Json | Array)?
let val: JsonValue = data.get("key")
```

类型兼容: `int→float`(隐式), `T→T?`(隐式), `int|null→int?`(规范化), `Json→具体类型`(显式转换或运行时检查)

typeof 返回 `Type` 枚举: `Type.int`, `Type.float`, `Type.string`, `Type.bool`,
`Type.null`, `Type.Array`, `Type.Map`, `Type.Set`, `Type.Json`, `Type.function`

## 4. 运算符

```xray
// 算术: + - * / %  比较: == != === !== < <= > >=  逻辑: && || !
// 位运算: & | ^ ~ << >>  赋值: = += -= *= /= %= &= |= ^= <<= >>=
// 自增自减: ++ --  三元: cond ? a : b
// 空值合并: a ?? b  可选链: obj?.prop, obj?.method()
// 范围: 0..10  展开: ...arr
// 类型检查: x is T  类型转换: x as T  安全转换: x as T?
// 强制解包: expr!
```

`==` 值相等，`===` 严格相等（类型+值），`is` 运行时类型检查
`as T` 失败抛异常，`as T?` 安全转换（失败返回 null）

## 5. 控制流

```xray
if (x > 0) { } else if (x == 0) { } else { }
while (cond) { }
for (let i = 0; i < 10; i++) { }
for (item in arr) { }
for (i in 0..100) { }
for (key in map) { }
for (day in MyEnum) { }           // 枚举迭代
break; continue

let result = match x {
    1 => "one",
    2, 3, 4 => "few",             // 多值匹配
    10..20 => "teen",              // 范围匹配
    n if (n > 100) => "big",      // 守卫条件
    Color.Red => "red",           // 枚举匹配
    _ => "default"                // 通配符
}
```

## 6. 函数

```xray
fn add(a: int, b: int): int { return a + b }
fn connect(host: string, port: int = 8080): void { }  // 默认参数
fn divmod(a: int, b: int): (int, int) { return a / b, a % b }  // 多返回值

// 参数传递模式（仅用于 struct 值类型参数）
fn length_sq(v: in Vec2): float { }        // in: 只读引用（不拷贝，不可修改）
fn translate(v: ref Vec2, dx: float): void { }  // ref: 可变引用（修改对调用方可见）
fn process(...items: int): void { }        // rest: 可变参数

// 箭头函数
let double = (x) => x * 2
let process = (x) => { let a = x * 2; return a + 10 }

// 柯里化
let add = (a) => (b) => (c) => a + b + c

// 高阶函数
arr.map((x) => x * 2).filter((x) => x > 5).reduce((a, b) => a + b, 0)

// 函数提升（fn 声明可先用后定义）
// 尾递归优化（accumulator 风格自动转循环）
```

## 7. 集合

```xray
// Array
let arr = [1, 2, 3]
arr.push(4); arr.pop(); arr.length
arr[0]; arr[0] = 10                    // 索引
arr[1:4]; arr[:3]; arr[2:]; arr[:]     // 切片
arr.map(fn); arr.filter(fn); arr.reduce(fn, init); arr.forEach(fn)

// Map（动态键值对）
let m = { "key" => value }            // Map 字面量
let empty = #{}                        // 空 Map
m.get(key); m.set(key, val); m.delete(key); m.has(key)
m.length; m.keys(); m.values()

// Set
let s = #[1, 2, 3]
s.add(4); s.has(2); s.delete(1); s.length

// Object 字面量（静态结构）
let obj = { name: "Alice", age: 30 }

// Json 类型（动态）
let data: Json = { id: 1, tags: ["a", "b"] }
data.id; data["tags"]

// Bytes（类型化字节数组 Array<uint8>）
let b = new Bytes()              // 空
let b = new Bytes(10)            // 指定长度（填充 0）
let b = new Bytes(3, 255)        // 指定长度+填充值
let b = new Bytes([72, 101, 108])// 从数组创建
b[0]; b[0] = 65; b.push(66); b.length
b[1:4]; b[:3]; b[2:]             // 切片

// WeakMap / WeakSet（弱引用，key 必须是对象）
let wm = new WeakMap()
wm.set(key, value); wm.get(key); wm.has(key); wm.delete(key)
let ws = new WeakSet()
ws.add(obj); ws.has(obj); ws.delete(obj)
```

## 8. 字符串

```xray
str.length; str[0]
str.charAt(i); str.charCodeAt(i)           // 字符 / Unicode code point
str.ord()                                   // 首字符 code point
str.toLowerCase(); str.toUpperCase()
str.trim(); str.trimStart(); str.trimEnd()
str.split(","); str.split(",", limit)
str.includes("x"); str.startsWith("a"); str.endsWith("z")
str.indexOf("x"); str.lastIndexOf("x")
str.replace("old", "new"); str.replaceAll("old", "new")
str.slice(start, end); str.substring(start, end); str.substr(start, len)
str.repeat(3); str.concat("a", "b")
str.padStart(10, "0"); str.padEnd(10, " ")
str.match(/pattern/); str.isWhitespace()

// StringBuilder（高性能拼接）
let sb = new StringBuilder()
sb.append("hello").append(" world")        // 支持链式调用
sb.length; sb.toString(); sb.clear()
```

## 9. 面向对象

### class（引用语义）

```xray
class Animal {
    name: string
    private _age: int
    constructor(name: string) { this.name = name }
    speak(): string { return "..." }
    static create(): Animal { }
}
class Dog extends Animal {
    constructor(name) { super(name) }
    override speak(): string { return "woof" }  // 重写
}
final class Config { }                          // 禁止继承
interface Shape { area(): float; perimeter(): float }
class Circle implements Shape { ... }
```

### struct（值语义）

```xray
struct Point {
    x: float
    y: float
    magnitude_sq(): float { return this.x * this.x + this.y * this.y }
}

let p = new Point()               // 默认构造
let p = Point{x: 3.0, y: 4.0}    // 字面量构造
p.x = 5.0                         // 字段赋值

// struct 支持: 方法、static、getter/setter、private 字段
// struct 支持: implements interface、泛型、运算符重载
// struct 不支持: 继承（extends）
```

访问修饰符: `private`, `public`(默认), `static`, `final`, `abstract`, `override`

运算符重载: `operator+`, `operator-`, `operator*`, `operator/`, `operator%`,
`operator&`, `operator|`, `operator^`,
`operator==`, `operator!=`, `operator<`, `operator<=`, `operator>`, `operator>=`,
`operator[]`, `operator[]=`, 一元 `operator-`, `operator!`, `operator~`

自定义迭代器: 实现 `iterator()` 返回 `{ hasNext(): bool, next(): T? }`

## 10. 枚举

```xray
enum Color { Red, Green, Blue }
enum HttpStatus { OK = 200, NotFound = 404 }
enum Direction { North = "N", South = "S" }    // 字符串枚举

Color.Red.name      // "Red"
Color.Red.value     // 0
Color.Red.ordinal   // 0
Color.memberCount   // 3
for (day in Day) { print(day.name) }           // 枚举迭代
```

## 11. 泛型

```xray
fn identity<T>(x: T): T { return x }
class Box<T> { value: T; get(): T { return this.value } }
class Pair<K, V> { key: K; value: V }
struct Stack<T> { ... }                        // struct 泛型

// 实例化（reified，运行时可反射）
let b = new Box<int>(42)
let arr: Array<int> = [1, 2, 3]
let ch: Channel<string> = new Channel<string>(10)
```

## 12. 异常

```xray
try { riskyOp() } catch (e) { handle(e) } finally { cleanup() }
throw "message"                   // throw 任意类型
throw { code: 500, msg: "err" }   // throw 对象
// 异常跨函数自动传播
```

## 13. 模块

```xray
import time; import math; import json
export fn helper() { }
export class Utils { }
```

标准库: base64, cluster, compress, crypto, csv, datetime, encoding,
gc, http, io, json, log, math, net, os, path, regex, time, toml, url, ws, xml, yaml

## 14. 协程与并发

### 基本

```xray
let task = go someFunction(args)          // 启动协程
let task = go { return 42 }               // 匿名块协程
let task = go(name: "worker-1") fn() {}() // 命名协程
let result = await task                    // 等待
let results = await all [t1, t2, t3]      // 等待全部
let first = await any [t1, t2, t3]        // 等待任一
let result = await(timeout: 1000) task     // 超时等待
```

### Task API

```xray
task.done           // bool: 是否已完成
task.cancelled      // bool: 是否已取消
task.result         // T?: 结果（未完成为 null）
task.error          // JsonValue?: 错误信息
task.cancel()       // 取消协程
task.monitor()      // 返回 Channel，任务完成时发送结果/错误
task.link(other)    // 双向关联：一方失败则另一方被取消
task.unlink(other)  // 解除关联
```

### Channel

```xray
const ch = new Channel(10)                   // 缓冲 Channel（必须 const，系统堆 + refcount）
ch.send(value)                               // 阻塞发送
let val = ch.recv()                          // 阻塞接收
let ok = ch.trySend(value)                   // 非阻塞发送
let (val, ok) = ch.tryRecv()                 // 非阻塞接收（返回 (T, bool) 元组）
let ok = ch.sendTimeout(value, 1000)         // 超时发送（毫秒）
let (val, ok) = ch.recvTimeout(1000)         // 超时接收（返回 (T, bool) 元组）
ch.close(); ch.closed                        // 关闭 / 检查是否已关闭
ch.isClosed()                                // 方法形式检查
```

### select / defer / scope

```xray
// select 多路复用
select {
    msg from ch1 => { print(msg) }     // 接收分支
    100 to ch => { print("sent") }     // 发送分支
    after 1000 => { print("timeout") } // 超时分支
    _ => { print("default") }          // default
}

// defer（LIFO 延迟执行）
defer cleanup()
defer { print("block") }              // defer 块

// scope（结构化并发，自动等待内部所有 go）
scope { go task1(); go task2() }

// linked scope（子协程错误传播给父级）
linked scope { go task1(); go task2() }

// supervisor scope（父级监管子协程，子协程异常不影响父级）
supervisor scope { go task1(); go task2() }

// linked go（单个协程关联到父级）
let t = linked go someFunc()

// move（显式所有权转移）
let data = [1, 2, 3]
go fn() { let local = move data }()        // data 移动到协程内
```

### Coro API

```xray
yield                            // 让出执行权
cancelled()                      // 检查当前协程是否被取消
Coro.self()                      // 获取当前协程名（string?）
Coro.setLocal(key, value)        // 协程本地存储
Coro.getLocal(key)               // 获取本地存储
Coro.setPriority(task, level)    // 设置优先级 (0=low, 1=normal, 2=high)
Coro.stats()                     // 协程统计（返回 Json）
Coro.list(limit?, state?)        // 列出协程（返回 Array<Json>）
Coro.dump(limit?)                // 打印协程信息
Coro.stalled(timeout_ms?)        // 获取停滞协程
Coro.deadlocks()                 // 检测死锁协程
Coro.top(n, metric?)             // Top N 协程（按指标排序）
Coro.groupBy(field)              // 按字段分组（返回 Json）
Coro.monitor(name)               // 监控命名协程（返回 Channel）
Coro.demonitor(ch)               // 取消监控
Coro.whereis(name)               // 检查命名协程是否存在
Coro.kill(name, reason?)         // 终止命名协程
Coro.lockThread()                // 锁定当前线程
Coro.unlockThread()              // 解锁当前线程
```

### 并发共享三条规则

xray 采用**显式共享**模型：跨协程可见的值必须通过以下三种机制之一。**没有第四种**，编译器不做隐式升级。

1. **`Channel`** — 跨协程通信（唯一免写 `shared` 的例外）
   - 必须 `const ch = new Channel(n)`；禁止 `let ch = ...`
   - 系统堆 + refcount，传参时 incref

2. **`shared const`** — 跨协程不可变共享（只读）
   - 全局堆 + refcount，零拷贝并发读
   - 字面量也须写 `shared const`（编译器仍做常量内联优化）

3. **函数参数** — 跨协程传值（深拷贝，零标注）
   - `go f(x)` / `ch.send(x)`：指针类型深拷贝到子协程堆

### `shared let` 的特殊用法

可变独占 + Move 语义（用于 pipeline 零拷贝所有权转移）：

- 禁止作为闭包 upvalue（闭包捕获 = 共享可变）
- 跨协程传递必须 `move`：`go task(move x)` 或 `ch.send(move x)`
- `move` 后再访问 → 编译错误

### 普通 `let`/`const` 的限制

- 在当前协程内：正常使用（协程本地堆）
- **被 `go` 闭包捕获 → 编译错误**（提示改用 `shared const` 或通过参数传递）
- 传给 `go f(x)` / `ch.send(x)` → 深拷贝，无需标注

## 15. 测试

```xray
@test fn test_example(): void { assert_eq(1 + 1, 2) }
@test(skip) fn test_skip(): void { }
@test(timeout: 5000) fn test_slow(): void { }
@before_each fn setup(): void { }
@after_each fn teardown(): void { }
@before_all fn global_setup(): void { }
@after_all fn global_teardown(): void { }

// 断言函数
assert(condition)                // 真值断言
assert_eq(actual, expected)      // 等值
assert_ne(actual, expected)      // 不等
assert_true(expr)                // 严格 true
assert_false(expr)               // 严格 false
assert_throws(fn)                // 预期抛异常
```

## 16. 内置函数

```xray
// I/O
print(value)                  // 输出（不换行）
println(value)                // 输出（换行）
dump(value)                   // 调试输出（含类型信息）

// 类型转换
int(value); float(value); string(value); bool(value)
chr(codepoint)                // Unicode code point → 字符串

// 类型检查
typeof(value)                 // 返回 Type 枚举值（int）
typename(value)               // 返回类型名（string）

// 工具
copy(value)                   // 深拷贝
sleep(ms)                     // 休眠（毫秒，协程内非阻塞）
spawn(fn)                     // 启动协程（等同 go）

// 容器构造（所有堆对象必须用 new）
new Array(); new Map(); new Set(); new Bytes()
new WeakMap(); new WeakSet()
new Channel(bufferSize?); new StringBuilder()
```

## 17. 全局变量

```xray
__file__                      // 当前脚本文件路径（string）
__dir__                       // 当前脚本所在目录（string）
process                       // 进程信息对象
```

## 18. 内置模块对象

```xray
// Reflect（运行时反射）
Reflect.getType(obj)          // 获取类型信息（Json）
Reflect.typeOf(obj)           // 获取类型名（string）
Reflect.isInstance(obj, cls)  // 是否为某类实例
Reflect.fieldCount(obj)       // 字段数量
Reflect.getAllTypes()          // 所有已注册类型

// CoroPool（协程池）
let pool = CoroPool(size)
pool.submit(fn)               // 提交任务
pool.close()                  // 关闭池

// cluster（分布式）
cluster.start(config); cluster.join(host, port)
cluster.channel(name); cluster.serve(name, handler)
cluster.call(node, service, data); cluster.monitor(node)
```

## 19. 关键字

```
// 声明
let const shared fn return yield

// 控制流
if else while for in break continue match

// OOP
class struct extends constructor this super new static
private public abstract override final interface implements operator

// 类型
type void bool int float string
int8 int16 int32 int64 uint8 uint16 uint32 uint64 float32 float64
Array Map Set Json BigInt Channel Range DateTime Bytes
WeakMap WeakSet Type StringBuilder JsonValue

// 异常
try catch finally throw

// 模块
import export as

// 协程
go await select defer scope

// 字面量
true false null enum is

// 模式
_ (wildcard)

// 上下文关键字（也可用作标识符）
from to default cancelled ref move linked supervisor after
```

## 附录 A. 内置类型方法速查

> 数据来源: `src/module/xbuiltin_method_defs.h`（X-macro 定义）+ VM 实现

### Array\<T>
| 属性/方法 | 签名 |
|-----------|------|
| length | : int |
| push/unshift | (value: T): void |
| pop/shift | (): T? |
| slice | (start?: int, end?: int): Array\<T> |
| splice | (start: int, deleteCount?: int, ...items: T): Array\<T> |
| concat | (...arrays: Array\<T>): Array\<T> |
| indexOf | (value: T): int |
| includes | (value: T): bool |
| join | (separator?: string): string |
| reverse | (): Array\<T> |
| sort | (compareFn?: fn(a: T, b: T): int): Array\<T> |
| map | (fn: fn(item: T, index: int): U): Array\<U> |
| filter | (fn: fn(item: T, index: int): bool): Array\<T> |
| reduce | (fn: fn(acc: U, item: T): U, initial: U): U |
| forEach | (fn: fn(item: T, index: int): void): void |
| find | (fn: fn(item: T): bool): T? |
| findIndex | (fn: fn(item: T): bool): int |
| every/some | (fn: fn(item: T): bool): bool |
| flat | (depth?: int): Array\<T> |
| fill | (value: T, start?: int, end?: int): Array\<T> |
| copyWithin | (target: int, start: int, end?: int): Array\<T> |

### string
| 属性/方法 | 签名 |
|-----------|------|
| length | : int |
| charAt | (index: int): string |
| charCodeAt | (index: int): int |
| ord | (): int |
| includes | (search: string): bool |
| indexOf/lastIndexOf | (search: string): int |
| slice/substring | (start: int, end?: int): string |
| substr | (start: int, length?: int): string |
| toLowerCase/toUpperCase | (): string |
| trim/trimStart/trimEnd | (): string |
| split | (separator: string, limit?: int): Array\<string> |
| replace/replaceAll | (search: string, replacement: string): string |
| repeat | (count: int): string |
| startsWith/endsWith | (search: string): bool |
| padStart/padEnd | (length: int, pad?: string): string |
| match | (pattern: string): Array\<string>? |
| concat | (...strings: string): string |
| isWhitespace | (): bool |

### Map\<K, V>
length, get(key): V?, set(key, value), has(key): bool, delete(key): bool,
clear(), keys(): Array\<K>, values(): Array\<V>, entries(), forEach(fn)

### Set\<T>
length, add(value), has(value): bool, delete(value): bool,
clear(), values(): Array\<T>, forEach(fn)

### Channel\<T>
send(value), recv(): T, trySend(value): bool, tryRecv(): (T, bool),
sendTimeout(value, ms): bool, recvTimeout(ms): T?, close(), closed: bool, isClosed(): bool

### int
abs(), toString(), toBigInt(), toFloat(), toHex(), max(other), min(other),
floor(), ceil(), round(), sqrt(): float, pow(exp): float

### float
abs(), toString(), toFixed(decimals?): string, toInt(),
floor(), ceil(), round(), sqrt(), pow(exp)

### BigInt
abs(), toString(), sign(): int, isZero(), isNegative(), isPositive(),
toInt(): int?, toFloat()

### Json
keys(), values(), entries(), has(key), get(key), isEmpty(), delete(key), clear(), toString()

### Regex
test(text): bool, find(text): string?, findAll(text): Array\<string>,
replace(text, replacement): string, split(text): Array\<string>

### StringBuilder
length, append(value): StringBuilder, toString(): string, clear(): StringBuilder

### Exception
message: string, stack: string, toString()

### Task
done: bool, cancelled: bool, result: any, error: string?,
cancel(), monitor(): Channel, link(other), unlink(other)

### EnumValue
name: string, value: any, ordinal: int, toString()

### EnumType
memberCount: int, getMember(index): EnumValue
