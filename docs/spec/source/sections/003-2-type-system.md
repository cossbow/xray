---
id: spec.2_type_system
order: 003
---

<!-- xr-spec:cn -->
---

## 2. 类型系统 (Type System)

> 真值源：`src/runtime/value/xtype.h`（XrType 定义）、`src/runtime/value/xtype.c`、`src/frontend/parser/xparse_type.c`（语法）、`src/frontend/analyzer/xtype_ref_resolve.c`（解析）、`stdlib/prelude/prelude_types.def`（内置类型表）。

### 2.1 概述

Xray 是静态类型语言；每个表达式在编译期有确定类型。类型系统的核心特性：

1. **类型推断**：变量声明几乎不用写类型；分析器从初始值/上下文推导。
2. **Nullable 分离**：`T` 永不为 `null`；`T?` 是 `T | null` 的语法糖。
3. **Union 类型**：`A | B | ...`（最多 6 个成员）。
4. **Generic reified**：泛型类型参数运行时可反射。
5. **Structural Json + Nominal class**：Json 对象按字段结构兼容（duck typing），class 按名义兼容。
6. **运行时反射**：`typeof` / `Reflect.*` API。

### 2.2 类型分类

| 类别 | 示例 |
|--|--|
| Primitive | `int`、`float`、`bool`、`string`、`()`（Unit，无返回值） |
| 精确整数 | `int8`、`int16`、`int32`、`int64`、`uint8`..`uint64` |
| 精确浮点 | `float32`、`float64` |
| 容器 | `Array<T>`、`Map<K,V>`、`Set<T>`、`Channel<T>`、`Bytes`（即 `Array<uint8>`） |
| 特殊 | `Json`、`BigInt`、`Range`、`DateTime`、`Regex`、`StringBuilder`、`Logger`、`NetConn`、`NetListener` |
| 错误处理 prelude | `Exception`、`Result<T, E>`（见 §8） |
| 弱引用容器 | `WeakMap`、`WeakSet` |
| Nullable | `T?` |
| Union | `A \| B \| ...` |
| Tuple | `(T1, T2, ...)` |
| Function | `fn(T1, T2) -> R` |
| Class / Struct / Interface | 用户定义（nominal） |
| Enum | 用户定义（含 ADT enum，见 §5.6） |
| Type alias | `type Name = SomeType` |

### 2.3 基本类型

#### 2.3.1 整数类型

| 类型 | 范围 | 别名 |
|--|--|--|
| `int8` | `[-128, 127]` | — |
| `int16` | `[-32768, 32767]` | — |
| `int32` | `[-2³¹, 2³¹-1]` | — |
| `int64` | `[-2⁶³, 2⁶³-1]` | `int`（默认整数类型）|
| `uint8`..`uint64` | 无符号对应 | — |

- 字面量默认 `int`；可被上下文窄化（如赋给 `int32` 变量）。
- 算术：二补码环绕语义（wrap on overflow），不区分 debug / release 构建。

#### 2.3.2 浮点类型

| 类型 | 标准 |
|--|--|
| `float32` | IEEE-754 单精度 |
| `float64` | IEEE-754 双精度；`float` 的别名 |

字面量默认 `float`。

#### 2.3.3 `bool`

`true` / `false`，独立类型，与数值类型**不可隐式互转**（不能 `let x: int = true`，也不能 `let b: bool = 1`）。

**truthy / falsy 上下文**（仅作用于 `if` / `while` / `?:` / `??` / `&&` / `||` 等控制流位置，**不**改变变量类型）：

| 值 | 视作 |
|---|---|
| `false`、`null`、`0`、`0.0`、`""`、`Bytes(0)`、空数组 / 空 Map | **falsy** |
| 其他一切（包括 `0.0001`、非空字符串/集合、对象引用） | **truthy** |

```xray
let x: int? = maybe_int()
if (x) {                  // truthy 上下文：x 既不是 null 也不是 0 时进入
    print(x + 1)          // 此分支中 x 被窄化为 int
}

let s: string = ""
if (s) { ... } else { ... }    // falsy：进入 else

let m: Map<string, int> = #{}
if (m) { ... }                  // falsy：空 Map

let a: int? = null
let b = a ?? 0                  // null 合并：b = 0
```

**注意**：`x is T` 和 `x != null` 等显式比较是首选，truthy/falsy 主要用于简洁的"存在性"判断（如 `if (user)`）。

#### 2.3.4 `string`

不可变 UTF-8 字符串。支持 `length`、索引、切片、丰富方法集（见 §14.2）。

底层使用引用计数（ARC）+ 字符串驻留（interning）优化。

#### 2.3.5 Unit `()`（无返回值）

xray 用 **0-元组 `()`** 表示"无返回值"（Unit 类型）：

```xray
fn log(msg: string) -> () { print(msg) }   // 显式 Unit 返回
fn ping() { print("pong") }                  // 省略返回类型 = ()
let r: () = log("hi")                        // 允许；r 是 Unit 值
```

- 一个函数省略返回类型等同于 `-> ()`。
- `void` 不是类型名：写 `fn f() -> void` 会被拒绝（`E0804`）；无返回值使用 `-> ()` 或省略返回类型。

### 2.4 复合类型

#### 2.4.1 `Array<T>`

有序可变数组。详见 §14.1。

```xray
let a: Array<int> = [1, 2, 3]
let b = [1, 2, 3]                // 推断为 Array<int>
let c: Array<string> = []         // 显式空数组
```

`Array<T>` 的 `T` 必须能在编译期确定。空 `[]` 在无类型标注时是编译错误：`Empty array '[]' requires a type annotation`。

#### 2.4.2 `Map<K, V>`

哈希字典，**保持插入顺序**。详见 §14.7。

**Map 字面量**必须用 `#{ ... }` 前缀，分隔符用 `:`（与 Json 一致，靠 `#` 前缀消歧）：

```xray
let m: Map<string, int> = #{"a": 1, "b": 2}
let m2 = #{"a": 1, "b": 2}
let empty = #{}                                     // 空 Map

m["c"] = 3                                          // 添加/修改
let v = m["a"]                                      // 取值；不存在返回 null
```

| 字面量形式 | 类型 | 用途 |
|---|---|---|
| `{ key: value }`（无前缀） | `Json` / `Object`（结构化） | 见 §2.4.6 |
| `#{ "k": v }`（`#` 前缀 + `:`） | `Map<K, V>`（哈希字典） | 本节 |
| `#{}` | `Map<K, V>`（空） | 显式空 Map |
| `[]` | `Array<T>` | 数组 |
| `#[]` | `Set<T>` | 集合 |

`K` 必须实现 `Hashable`（详见 §14.14）：通常是 `int`、`string`、`bool`、`enum`、或自定义实现 `Hashable` 的类。

#### 2.4.3 `Set<T>`

去重集合。详见 §14.4。

```xray
let s: Set<int> = #[1, 2, 3]
```

#### 2.4.4 `Channel<T>`

协程间通信通道。**必须**用 `const` 声明（见 §10.5）。

```xray
const ch: Channel<int> = new Channel<int>(10)
```

#### 2.4.5 `Bytes`

类型化字节缓冲。语义等价 `Array<uint8>`，但底层是连续内存。

```xray
let buf = new Bytes(1024)
let init = new Bytes([72, 101, 108, 108, 111])
```

#### 2.4.6 `Json` 与对象字面量

`Json` 是 xray 的**动态结构化数据类型**——可以装载 JSON 等价的任意结构。详见 §14.10 与 §2.10。

**对象字面量** `{ field: value, ... }` 与 Map 字面量的关键区别：

```xray
// Object/Json 字面量：标识符或字符串 key + 冒号 ':'
let data: Json = { name: "Alice", tags: ["a", "b"], age: 30 }
let user = { name: "Bob", age: 25 }       // 默认类型为 Json
data.name              // 类型: Json（字段访问返回 Json）
data["name"]           // 等价

// 字段简写：当字段名与变量名相同
let name = "Alice"
let age = 30
let user = { name, age }                  // 等价 { name: name, age: age }

// Map 字面量：`#{}` 前缀 + `:`
let m = #{"k1": 1, "k2": 2}           // 类型: Map<string, int>
```

**对照表**：

| 写法 | 类型 | 备注 |
|---|---|---|
| `{ name: "x", age: 1 }` | `Json` / `Object` | 标识符或字符串 key 后跟 `:` |
| `{ x: y }`（`x` 是字段名，`y` 是变量名） | `Json` / `Object` | 字段简写 `{ x }` 等价 `{ x: x }`，仅裸 key |
| `#{"a": 1}` | `Map<K, V>` | `#` 前缀消歧，分隔符用 `:` |
| `Point{x: 1.0, y: 2.0}` | `Point`（struct） | 类型名 + `{...}` 字面量 |

**密封（sealed）对象类型**：通过 `type` 别名为对象类型起名后，类型成为 sealed——访问/赋值未声明字段是编译错误：

```xray
type User = { name: string, age: int }

let u: User = { name: "Alice", age: 30 }
print(u.name)         // OK
// u.extra = "x"      // 编译错误：sealed type User has no field 'extra'

// 不指定类型则为动态 Json
let u2 = { name: "Alice", age: 30 }      // Json（可动态扩展）
u2.extra = "x"        // OK（Json 是动态的）
```

#### 2.4.7 `BigInt`

任意精度整数。见 §14.8。

#### 2.4.8 `Range`

由 `..` 运算符产生。见 §3.12。

#### 2.4.9 `DateTime` / `Regex` / `StringBuilder`

详见 §14。

#### 2.4.10 `WeakMap` / `WeakSet`

`WeakMap` 的键、`WeakSet` 的元素必须是堆对象；弱引用不阻止 GC 回收。弱集合不提供会长期持有元素的遍历回调。

### 2.5 可空类型

`T?` 是 `T | null` 的语法糖。

```xray
let x: int? = null      // OK
let y: int? = 42        // OK
let z: int = null       // 编译错误：null 不是 int
```

#### 解包

```xray
// 1. 空合并
let v = x ?? 0

// 2. 可选链
let len = name?.length    // 若 name 为 null，结果为 null

// 3. 强制解包
let v: int = x!           // 若 x 为 null，运行时抛 NullError

// 4. is 检查
if (x is int) {
    // 此分支内 x 类型窄化为 int
    print(x + 1)
}
```

### 2.6 Union 类型

```xray
let v: int | string = 42
v = "hello"             // OK
```

约束：
- 最多 **6 个成员**（编译期检查；超限 → 错误）。
- 成员互不为彼此的子类型（否则会被规范化）。
- 处理 union 值需用 `match` 或 `is` 窄化：

```xray
let v: int | string = ...
match v {
    is int    -> print("int: ${v}"),
    is string -> print("str: ${v}"),
}
```

**特殊化**：
- `int | null` 规范化为 `int?`。
- `T?` 出现在 union 时：`int? | string` 实际等价 `int | string | null`，规范化为 `(int | string)?`。

### 2.7 元组类型

xray 的元组**是头等公民**——可以作为任意值出现、作为字段保存、嵌套。

```xray
// 字面量
let t = (1, 2, 3)                 // 类型推断为 (int, int, int)
let h = (10, "hi", true)          // 异构元组
let single = (99,)                // 单元素元组：注意尾逗号

// 类型注解
let p: (int, string) = (7, "ok")

// 字段访问：.N（N 是编译期常量整数下标）
let first = t.0                   // 1
let mid   = t.1                   // 2
let nest  = ((1, 2), (3, 4))
let a     = nest.0.0              // 1
let b     = nest.1.1              // 4

// 函数返回与解构
fn divmod(a: int, b: int): (int, int) { return (a / b, a % b) }
let (q, r) = divmod(17, 5)        // tuple destructure

// 泛型
fn pair<A, B>(a: A, b: B): (A, B) { return (a, b) }
let p2 = pair(1, "x")             // (int, string)
```

**注意事项**：

- **单元素元组**必须用尾逗号 `(x,)`——不带逗号的 `(x)` 是分组括号（普通表达式）。
- 字段访问 `t.N` 中 N **必须是字面量整数**；用变量或字符串访问是编译错误 `XR_ERR_ANALYZE_TUPLE_FIELD_NAME` / `_RANGE`。
- 元组**不可变**：`t.0 = v` 是编译错误。修改必须重新构造。

### 2.8 类型别名

```xray
type Result = int | string
type Mapper = (int) -> int
type Point = { x: float, y: float }
```

别名是**纯语法**等价，不产生新类型。

### 2.9 类型推断

详见 §7.4。简述：

```xray
let x = 1               // x: int
let y = 1.5             // y: float
let z = "hello"         // z: string
let a = [1, 2, 3]       // a: Array<int>
let m = #{"a": 1}    // m: Map<string, int>
let p = { name: "A" }   // p: { name: string } —— 结构化对象类型
let f = (x: int) -> x   // f: (int) -> int —— 箭头参数必须标注
```

### 2.10 类型兼容性与转换

#### 2.10.1 隐式转换

| 源 | 目标 | 允许 |
|--|--|--|
| `int` | `float` | ✅ |
| `int8` | `int` (= `int64`) | ✅ |
| `T` | `T?` | ✅ |
| `T` | `Json`（如果 T 是 Json 兼容） | ✅ |
| `null` | `T?` | ✅ |
| Subtype | Supertype（class）| ✅ |
| 子集对象类型 | 超集对象类型 | ❌（结构化兼容是 superset → subset） |

> **结构化兼容方向**（duck typing）：字段更多的类型可赋给字段更少的类型。
> ```xray
> type User = { name: string }
> let full = { name: "A", age: 18 }
> let u: User = full       // OK：full 是 User 的超集
> ```

#### 2.10.2 显式 `as`

```xray
let n = x as int        // 失败抛 TypeError
let n = x as int?       // 失败返回 null（安全转换）
```

适用于：
- 数值之间（含 `Json → int`，运行时检查）。
- `Json → User`（结构化 narrowing）。
- 父类 → 子类（向下转）。

#### 2.10.3 `is` 检查

```xray
if (v is User) {
    // 编译器在此分支窄化 v 的类型为 User
}
```

仅作类型守卫；不改变值。

### 2.11 typeof / typename / Type 枚举

```xray
typeof(value)     // 返回 Type 枚举值（int 表示）
typename(value)   // 返回类型名字符串
```

`Type` 枚举成员：

`Type.int`、`Type.float`、`Type.string`、`Type.bool`、`Type.null`、
`Type.Array`、`Type.Map`、`Type.Set`、`Type.Channel`、`Type.Json`、
`Type.function`、`Type.class`、`Type.struct`、`Type.enum`、`Type.module`、`Type.bigint`、...

完整列表见 `stdlib/types/enum.xr` / `src/runtime/value/xtype.h`。

### 2.12 运行时反射

`Reflect` 模块（内置）：

```xray
Reflect.getType(obj)        // 获取类型信息（Json）
Reflect.typeOf(obj)         // 获取类型名（string）
Reflect.isInstance(obj, cls)// 是否某类实例
Reflect.fieldCount(obj)     // 字段数量
Reflect.getAllTypes()       // 所有已注册类型
```

详见 §13 与 §14。
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 2. Type System

### 2.1 Overview

Xray is statically typed. Local variables usually rely on inference, while function parameters, public APIs, fields, and complex values often use explicit annotations.

There is no source-level `any` type. When this document says “value” in an API signature, it means an arbitrary runtime value accepted by an intrinsic path, not a type you can write in source.

### 2.2 Primitive Types

| Type | Description |
|--|--|
| `int` | integer value |
| `float` | floating-point value |
| `bool` | boolean |
| `string` | UTF-8 string |
| `null` | singleton null value |
| `()` | unit type/value |

Width-specific numeric type names are recognized, but the exact lowering is implementation-defined for current releases.

### 2.3 Nullable Types

```xray
let s: string? = null
let n: int? = 42
```

`T?` means `T | null`. Nullable values can be checked against `null`, accessed with optional chaining, or force-unwrapped with `!`.

### 2.4 Union Types

```xray
let id: int | string = 42
id = "abc"
```

Union types are written with `|`. Type narrowing is performed in supported control-flow and `is` checks.

### 2.5 Function Types

```xray
type Mapper = (int) -> int
let pred: (string) -> bool
```

Function types use `->`. Parameter names are not part of a function type.

### 2.6 Tuple Types

```xray
let p: (int, string) = (1, "x")
print(p.0)
print(p.1)
```

Tuples are fixed-arity product values. Tuple fields can be accessed by numeric member names.

### 2.7 Object and Json Types

Object type aliases are sealed when used as annotations:

```xray
type User = { name: string, age: int }
let u: User = { name: "alice", age: 30 }
```

Access to undeclared fields on a sealed object is rejected. Unannotated object literals generally produce dynamic `Json`-style values.

`Json` is the dynamic structured data type for JSON-like values.

### 2.8 Nominal Types

`class`, `struct`, `interface`, and `enum` declarations create nominal types. Classes are reference-oriented. Structs are value-oriented declarations sharing class-like member syntax. Interfaces define contracts. Enums support both backed simple enums and ADT payload variants.

### 2.9 Built-in Container Types

| Type | Meaning |
|--|--|
| `Array<T>` | ordered mutable sequence |
| `Map<K, V>` | insertion-ordered key/value table |
| `Set<T>` | unique-value collection |
| `Channel<T>` | typed coroutine channel |
| `Range` | range created by `a..b` |
| `Bytes` | byte-buffer type |
| `StringBuilder` | mutable string builder |
| `Regex` | compiled regex value |
| `DateTime` | native date/time value |

### 2.10 Assignability

Values are assignable when their static type is equal to or safely compatible with the target type. Nullable assignment requires a nullable target. Union assignment requires the source to be assignable to at least one union arm.

Numeric implicit conversion is intentionally limited. Use explicit conversion functions such as `int(x)`, `float(x)`, and `string(x)` when needed.

### 2.11 Truthiness

Control-flow conditions accept truthy/falsy values. Common falsy values are `false`, `null`, numeric zero, empty string, and empty containers. Static assignment to `bool` remains type-checked and does not mean arbitrary truthy values are implicitly bools.
<!-- /xr-spec:en -->
