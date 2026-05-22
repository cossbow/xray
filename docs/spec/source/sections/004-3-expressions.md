---
id: spec.3_expressions
order: 004
---

<!-- xr-spec:cn -->
---

## 3. 表达式 (Expressions)

> 真值源：`src/frontend/parser/xparse_expr.c`、`src/frontend/parser/xast_types.h` 的 `AST_BINARY_*` / `AST_UNARY_*` / `AST_TERNARY` / `AST_*` 等节点。

### 3.1 优先级与结合性

完整优先级表（自**高至低**；同级运算符按结合性分组）：

| 级 | 运算符 | 结合性 | 说明 |
|--|--|--|--|
| 17 | `(...)` `[...]` `.x` `?.x` `f()` `e!` | 左 | 后缀：分组、索引、成员、可选链、调用、强制解包 |
| 16 | 前缀 `-` `+` `!` `~` `new` `move` `await` `go` | 右 | 一元前缀 + 协程操作（`++` / `--` 仅后缀） |
| 15 | `as` `is` | 左 | 类型转换 / 检查（`as T?` 安全形式靠目标类型可空，非独立 `as?` 运算符） |
| 14 | `*` `/` `%` | 左 | 乘除取模 |
| 13 | `+` `-` | 左 | 加减 |
| 12 | `<<` `>>` | 左 | 移位 |
| 11 | `<` `<=` `>` `>=` | 左 | 关系比较 |
| 10 | `==` `!=` `===` `!==` | 左 | 相等比较 |
| 9 | `&` | 左 | 位与 |
| 8 | `^` | 左 | 位异或 |
| 7 | `\|` | 左 | 位或（亦用于 union 类型） |
| 6 | `&&` | 左 | 逻辑与（短路） |
| 5 | `\|\|` | 左 | 逻辑或（短路） |
| 4 | `??` | 左 | 空值合并 |
| 3 | `..` | 左 | 范围 |
| 2 | `? :` | 右 | 三元 |
| 1 | `=` `+=` `-=` `*=` `/=` `%=` `&=` `\|=` `^=` `<<=` `>>=` | 右 | 赋值与复合赋值 |
| 0 | `,`（仅 `match` 多值、参数列表等特定位置）| — | 不是真正运算符 |

实现：`src/frontend/parser/xparse_expr.c` 的 Pratt-parser 风格。

### 3.2 一元表达式

```ebnf
UnaryExpr ::= ('-' | '+' | '!' | '~') UnaryExpr
            | 'new' Identifier TypeArgs? '(' ArgList? ')'
            | 'move' UnaryExpr
            | 'await' ('all' | 'any')? UnaryExpr
            | 'go' (Block | PostfixExpr)
            | 'try?' UnaryExpr
            | 'try!' UnaryExpr
            | PostfixExpr
```

| 运算符 | 适用类型 | 结果类型 | 备注 |
|--|--|--|--|
| `-x` | 数值 | 同 | 负号；浮点 NaN 保留 |
| `+x` | 数值 | 同 | 标识，几乎无用 |
| `!x` | `bool` | `bool` | 逻辑非；**不接受非 bool**（不像 JS） |
| `~x` | 整数 | 同 | 按位取反 |
| `x++` `x--` | 整数 | 同 | 后缀自增/减；返回**修改后**的值（与 C/Java 不同，本质上是 `x = x + 1` 的赋值结果） |

**自增/减语义**：
- xray **只提供后缀** `x++` / `x--`，不支持前缀 `++x` / `--x`（编译错误："prefix ++/-- not supported, use postfix form"）。
- 等价于赋值 `x = x + 1` / `x = x - 1`；表达式值即赋值后的新值。
- 不能内联在二元表达式中（如 `a + x++`、`f(x++)`、`x++ + 1`）；解析器会报"++/-- must be standalone statement"。允许 `let y = x++`（赋值左侧紧邻），此时 `y` 取到的是修改后的值。
- 仅作用于左值（变量、字段、索引）。
- 浮点数**不支持** `++`/`--`（编译错误）。

### 3.3 二元表达式

```ebnf
BinaryExpr ::= UnaryExpr (BinOp UnaryExpr)*
BinOp ::= '+' | '-' | '*' | '/' | '%'
       | '&' | '|' | '^' | '<<' | '>>'
       | '<' | '<=' | '>' | '>='
       | '==' | '!=' | '===' | '!=='
       | '&&' | '||'
       | '??'
```

#### 3.3.1 算术运算符

| 运算符 | int×int | float×float | int×float | string | 其他 |
|--|--|--|--|--|--|
| `+` | int | float | float（int→float 提升） | 字符串拼接 | ❌ |
| `-` | int | float | float | ❌ | ❌ |
| `*` | int | float | float | ❌ | ❌ |
| `/` | int（整除） | float | float | ❌ | ❌ |
| `%` | int | float（IEEE remainder） | float | ❌ | ❌ |

**特殊语义**：
- `int / 0` → 运行时抛 `XR_ERR_DIV_BY_ZERO` (E0420)。
- `int % 0` → 运行时抛 `XR_ERR_MOD_BY_ZERO` (E0421)。
- `float / 0.0` → `+inf` / `-inf` / `NaN`（IEEE-754 语义，**不抛**异常）。
- 整数溢出：见 §2.3.1。
- 字符串 `+ string` 是 O(n) 拼接；密集拼接请用 `StringBuilder`。

#### 3.3.2 位运算

`&` `|` `^` `~` `<<` `>>`

- 仅作用于整数类型。
- 移位计数取模 64（与 C 不同：xray 总是定义的）。
- `>>` 是**算术右移**（保留符号位）。无符号类型用对应的 `uintN`。
- bool 不参与位运算（用 `&&` `||`）。

#### 3.3.3 比较运算符

| 运算符 | 语义 |
|--|--|
| `==` | 值相等。`int` 与 `float` 可比较（int→float 隐式）。字符串按内容比较。class/struct 使用 `==` 重载或默认 identity。 |
| `!=` | `==` 的反 |
| `===` | **严格**相等：类型与值都必须相等；无任何隐式转换。 |
| `!==` | `===` 的反 |
| `<` `<=` `>` `>=` | 数值与字符串支持；其他类型默认不支持（可通过 `operator<` 重载启用）。 |

**与 JS 的区别**：xray 的 `==` **不**做 string↔number 转换；只做数值之间的 int↔float 提升。

#### 3.3.4 逻辑运算符

`&&` `||`：

- 两操作数**必须**是 `bool` 类型（编译期检查）。
- 短路求值：`false && X` 不求值 `X`；`true || X` 不求值 `X`。
- 结果类型 `bool`（不像 JS 返回操作数）。

#### 3.3.5 空值合并 `??`

```xray
let v = nullable_expr ?? default_value
```

- 当 `nullable_expr` 为 `null` 时返回 `default_value`，否则返回 `nullable_expr` 本身。
- **短路**：`default_value` 只在前者为 null 时求值。
- 类型推导：若 `nullable_expr: T?` 且 `default_value: T`，结果类型 `T`（非空）。
- 仅作用于可空类型；对 `T`（非空）使用 `??` 是编译警告/错误。

### 3.4 赋值与复合赋值

```ebnf
AssignExpr ::= LValue AssignOp Expression
LValue ::= Identifier | MemberAccess | IndexAccess
AssignOp ::= '=' | '+=' | '-=' | '*=' | '/=' | '%='
           | '&=' | '|=' | '^=' | '<<=' | '>>='
```

**语义**：
- 赋值是**表达式**，结果是赋值后的值（可链式：`a = b = 0`）。
- `x op= y` 等价于 `x = x op y`，但 `x` 只求值一次（重要：`obj.f += 1` 不会调用 `f` 的 getter 两次）。
- 不能赋值给 `const`（编译错误 `E0303`）。
- 不能赋值给 `shared const`（同上）。

**特殊**：
- 函数参数中的 `in T` 修饰符使参数变只读，对其赋值是编译错误。
- 数组/Map 字面量字段：`a[i] = v` 调用 `operator[]=` 或内置 setter。

### 3.5 三元 `? :`

```ebnf
TernaryExpr ::= LogicOrExpr ('?' Expression ':' Expression)?
```

```xray
let max = a > b ? a : b
```

- **右结合**：`a ? b : c ? d : e` = `a ? b : (c ? d : e)`。
- 条件必须是 `bool`。
- 两分支类型统一：取共同超类型（或 union）。

### 3.6 空合并 `??` 与可选链 `?.`

详见 §3.3.5（`??`）与下方（`?.`）。

#### 可选链 `?.`

```ebnf
OptionalChain ::= Primary ('?.' (Identifier | '[' Expr ']'))+
```

```xray
let len = name?.length          // null 时返回 null
let item = arr?.[0]             // 可选索引
```

**语义**：
- 若 `?.` 左侧为 `null`，整个表达式短路返回 `null`。
- **传播**：`a?.b.c.d` 中，若 `a` 为 null，整个链返回 null；中间 `.` 不重新检查。
- 结果类型：原类型加 `?`（若已经 `?` 则保持）。
- 当前 `?.` 支持属性访问与索引访问；函数可选调用 `func?.()` 不属于当前语法。

### 3.7 强制解包 `!` / `try?` / `try!` / `catch!`

> 完整错误处理语义见 §8。本节只列表达式语法与简要语义。

#### 强制解包 `expr!`

```xray
let v: int = nullable_int!      // null 时运行时抛 NullThrowError (E0410)
```

仅当编译期可确定 `expr` 是可空类型 (`T?`) 时合法；对非空类型 `T` 使用 `!` 是编译错误。

#### `try?` 表达式 —— 失败转 null

```ebnf
TryOptional ::= 'try?' Expression
```

`try? e` 把任何失败折叠为 `null`，丢弃错因：

| `e` 的静态类型 | `try? e` 的类型 | 行为 |
|--|--|--|
| `Result<T, E>` | `T?` | `Err` → `null`；`Ok(v)` → `v` |
| `T`（普通类型，调用可能抛异常） | `T?` | 抛异常 → `null`；正常 → `v` |
| `T?` | `T?` | 透传 |

```xray
let n: int? = try? parseInt(input)         // Result.Err → null
let v: Json? = try? http.get(url).json()   // 抛异常 → null
```

#### `try!` 表达式 —— 早退或跨轨升级

```ebnf
TryForce ::= 'try!' Expression
```

`try!` 后面的表达式静态类型**必须**是 `Result<T, E>` 或 `T?`（其它类型编译报错 `E0821`）。行为按表达式类型 + 当前函数返回类型双重 dispatch：

| `e` 类型 | 当前函数返回 | 失败行为 | 成功值 |
|--|--|--|--|
| `Result<T, E>` | `Result<_, E>` | 早退 `return Result.Err(e)` | `v` |
| `Result<T, E>` | 其它 | `throw e`（要求 `E` 是 `Exception` 派生） | `v` |
| `T?` | `_?` | 早退 `return null` | `v` |
| `T?` | 其它 | `throw new NullThrowError(...)` | `v` |

```xray
fn pipeline(s: string) -> Result<int, ParseError> {
    let n = try! parseInt(s)      // Err 早退 Err
    return Result.Ok(n + 1)
}
```

完整说明见 §8.3.1。**`try!` 不是异常调用的强制仪式**——xray 不要求会抛异常的调用前必须写 `try!`。

#### `catch!` 块表达式 —— 异常块凝结成 Result

```ebnf
CatchBlock ::= 'catch!' Block
```

`catch! { ... }` 把代码块中可能抛出的任何异常包装为 `Result<T, Exception>`：

```xray
let r: Result<int, Exception> = catch! {
    let x = riskyOp()
    let y = another(x)
    return y + 1                   // 块内的 return 是"从 catch! 块返回"
}
```

- 块内 `return v` 是从 `catch!` 块返回（不影响外层函数）
- 块的最后一个表达式作为 `Ok(v)` 的值
- 块内**逃出的任何异常**包装为 `Err(e)`，`e` 静态类型恒为 `Exception`
- 不支持类型过滤——需要时用手写 `try/catch`

完整说明见 §8.3.3。

### 3.8 `as` / `is`

#### `is` 运行时类型检查

```ebnf
IsExpr ::= UnaryExpr 'is' Type
```

```xray
if (v is User) {
    // 此分支内 v 窄化为 User
    print(v.name)
}
```

- 结果类型 `bool`。
- **类型守卫**：分析器在分支内窄化 `v` 的静态类型。
- 适用于 union、可空、class 层级、`Json` 结构匹配。

#### `as` 类型转换

```ebnf
AsExpr ::= UnaryExpr 'as' Type
        |  UnaryExpr 'as' Type '?'
```

```xray
let n = v as int           // 失败抛 TypeError
let n = v as int?          // 失败返回 null（"as nullable" 安全形式）
```

| 形式 | 失败行为 | 用途 |
|--|--|--|
| `expr as T` | 抛 `XR_ERR_TYPE_MISMATCH` (E0404) | 必须成功的转换 |
| `expr as T?` | 返回 `null` | 不确定能否转换的尝试 |

**支持的转换**：
- 数值之间（`int → float` 不丢失，`float → int` 截断）。
- `Json → T`（运行时按 T 结构检查）。
- 父类 → 子类（运行时 instanceof）。
- Union 成员 → 具体成员。

### 3.9 范围 `..` 与展开 `...`

#### 范围 `a..b`

```ebnf
RangeExpr ::= AddExpr ('..' AddExpr)?
```

```xray
0..10                  // 0..10，左闭右开（包含 0，不包含 10）
let r = 1..100
for (i in 0..n) { ... }
```

- 类型 `Range`（仅 int 范围）。
- 半开区间 `[a, b)`：`a` 包含、`b` 不包含。`for-in`、`Range.includes`、`Range.length`、`Range.toArray()`、`match` 中的 `a..b` 模式全部遵循同一语义。
- 主要用途：`for-in` 循环、模式匹配中的范围判定。
- 当前不提供闭区间语法（`a..=b`）；如需"包含 b"，写 `a..(b+1)`。

#### 展开 `...`

仅在以下位置使用：
- **函数 rest 参数声明**：`fn f(...args: int)`
- **函数调用展开**：`f(...args)`，展开源必须是静态 arity 已知的 tuple。
- **tuple 字面量展开**：`(head, ...tail)`，展开源必须是静态 arity 已知的 tuple。

### 3.10 字面量构造

#### Array `[...]`

```ebnf
ArrayLit ::= '[' (Expr (',' Expr)* ','?)? ']'
```

```xray
let a = [1, 2, 3]
let empty: Array<int> = []
let mixed = [1, "hello"]    // 类型 Array<int | string>
```

#### Map `#{k: v, ...}` 与 `#{}`

```ebnf
MapLit   ::= '#{' (MapEntry (',' MapEntry)* ','?)? '}'
MapEntry ::= Expression ':' Expression
EmptyMap ::= '#{' '}'    // 注意：'#{' 是单个 token
```

```xray
let m = #{"a": 1, "b": 2}
let empty = #{}                           // 空 Map
```

**关键区别**：`{}` 始终是**Json / Object**；`#{}` 始终是 **Map**。两者都用 `:` 作键值分隔，靠 `#` 前缀区分。

#### Set `#[...]`

```ebnf
SetLit ::= '#[' (Expr (',' Expr)* ','?)? ']'
```

```xray
let s = #[1, 2, 3]
let empty = #[]
```

#### Object（结构化对象）`{ field: value, ... }`

```ebnf
ObjectLit  ::= '{' ObjectField (',' ObjectField)* ','? '}'
ObjectField ::= Identifier ':' Expr
              | Identifier            // shorthand: `{ x }` 等价 `{ x: x }`
```

```xray
let p = { name: "Alice", age: 30 }
let users = "Bob"
let obj = { users }              // shorthand
```

- 默认推断为**可扩展**的结构化对象类型（见 §2.4.6 / §2.10 Json 行为）。
- 用 `type` 别名固化结构：`let u: User = {...}`（编译期检查字段集，密封）。

#### Bytes `new Bytes(...)`

详见 §2.4.5 与 §14.5。

#### Channel `new Channel<T>(buf?)`

```xray
const ch: Channel<int> = new Channel<int>(10)
```

详见 §10.5。

### 3.11 调用 / 成员访问 / 索引 / 切片

#### 函数调用

```ebnf
CallExpr ::= Primary '(' ArgList? ')'
ArgList ::= Expr (',' Expr)* ','?
```

- 参数按位置传递；不支持命名参数。
- rest 参数收集多余参数到数组。
- 参数计数不匹配 → 编译错误 `E0307` / `E0450`。

#### 成员访问

```ebnf
MemberAccess ::= Primary '.' Identifier
```

```xray
obj.field
obj.method(args)
ClassName.staticMethod()
EnumName.MemberName
```

- 字段访问：编译期检查类型有此字段。
- 方法调用：解析为 invoke（带 IC 缓存优化）。
- 模块成员：`module.export_name`。
- 枚举成员：`Color.Red`。

#### 索引访问

```ebnf
IndexAccess ::= Primary '[' Expr ']'
```

```xray
arr[0]
arr[0] = 10
map["key"]
str[i]                  // 返回单字符字符串
```

- `Array` 索引：`int`，越界抛 `E0430`。
- `Map` 索引：键类型；找不到键 → `E0431`。
- `string` 索引：返回长度为 1 的字符串（**不是** char/int）。
- 自定义类：通过 `operator[]` 重载。

#### 切片

```ebnf
Slice ::= Primary '[' Expr? ':' Expr? ']'
```

```xray
arr[1:4]                // 元素 [1,4)
arr[:3]                 // 前 3 个
arr[2:]                 // 从索引 2 到末尾
arr[:]                  // 全切片（浅拷贝）
str[0:5]                // 字符串切片
```

- 半开区间 `[start, end)`。
- `string` 切片支持负索引，负数从末尾计数；`Array` 切片中负 `start` 夹到 `0`，负 `end` 视为数组长度。
- 切片返回新对象，不修改原数组。

### 3.12 匿名函数与 Lambda

xray 有三种匿名函数语法，全部编译为相同的 `AST_FUNCTION_EXPR` 节点，语义完全等价，仅在简洁度和适用场景上有区别。

```ebnf
AnonFunction ::= BareLambda | ArrowLambda | FnExpression
BareLambda   ::= Identifier '->' (Expression | Block)
ArrowLambda  ::= '(' ArrowParams? ')' '->' (Expression | Block)
ArrowParams  ::= ArrowParam (',' ArrowParam)*
ArrowParam   ::= Identifier (':' Type)?      // 类型可省略，由上下文推断
FnExpression ::= 'fn' GenericParams? '(' Params ')' ('->' Type)? Block
```

```xray
// ── 裸 lambda：最简洁，仅限调用参数位置 ──
arr.map(x -> x * 2)
arr.filter(x -> x % 2 == 0)

// ── 箭头 lambda：任意位置，支持多参数和类型注解 ──
let sum = arr.reduce((acc, x) -> acc + x, 0)    // 无类型
let double = (x: int) -> x * 2                   // 有类型
let add = (a: int, b: int) -> a + b              // 多参数

// ── fn 表达式：多语句体、返回类型注解、泛型参数 ──
let inc = fn(x: int) -> int {
    let y = x + 1
    return y
}
let identity = fn<T>(x: T) -> T { return x }     // 泛型
```

**三种形式的选择指南**：

| 形式 | 语法 | 适用场景 |
|------|------|----------|
| 裸 lambda | `x -> expr` | 单参数回调，最简洁 |
| 箭头 lambda | `(x, y) -> expr` | 多参数、需类型注解、或非调用参数位置 |
| fn 表达式 | `fn(x: T) -> R { ... }` | 多语句体、返回类型注解、泛型参数 |

**关键规则**：
- **裸 lambda**（`x -> expr`）：仅限**调用参数位置**，单参数无括号。参数类型由被调函数签名或容器元素类型推断。
- **箭头 lambda**（`(x) -> expr`、`(x, y) -> expr`）：任意位置可用。参数类型可省略，由上下文推断；推断失败时报 E0365。
- **fn 表达式**（`fn(x: T) { ... }`）：任意位置可用。支持泛型参数 `fn<T>(...)`、返回类型注解 `-> T`、多语句体。
- 单表达式形式 `-> expr` 自动 `return`。
- 块形式 `-> { ... }` 或 `{ ... }` 用显式 `return`。
- 捕获规则：见 §7.4 闭包捕获。**`go` 协程闭包对 `let` 变量的捕获是编译错误**——必须显式 `shared const`、`move`、或参数传递。

### 3.13 `match` 表达式

```ebnf
MatchExpr ::= 'match' Expr '{' MatchArm (',' MatchArm)* ','? '}'
MatchArm ::= Pattern ('if' Expr)? '->' Expression
```

```xray
let result = match x {
    1 -> "one",
    2, 3, 4 -> "few",                 // 多值
    10..20 -> "teen",                 // 范围
    n if (n > 100) -> "big",          // 守卫
    Color.Red -> "red",               // 枚举
    is User -> "a user",              // 类型模式
    _ -> "default"                    // 通配符
}
```

**语义**：
- 自上而下匹配第一个成功的分支。
- 所有分支表达式必须返回相同类型（或 union）。
- **穷举性**：对 enum 变量（ADT 与简单枚举）编译器强制穷举。对其他表达式不强制，运行时无匹配抛 `Exception(E0442)`。
- 模式详见 [§6](#6-模式-patterns)。

### 3.14 `new`

```ebnf
NewExpr ::= 'new' Identifier TypeArgs? '(' ArgList? ')'
```

```xray
let p = new Point(1.0, 2.0)
let arr = new Array<int>()
let ch = new Channel<int>(10)
let m = new Map<string, int>()
```

**用于**：
- 类与 struct 实例化。
- 容器内置类型构造（`Array`/`Map`/`Set`/`Channel`/`Bytes`/`StringBuilder` 等）。

**与字面量的关系**：
```xray
let a = [1, 2, 3]              // 等价 new Array<int>() + push
let m = #{}                    // 等价 new Map<...>()
let p = Point{x: 1, y: 2}      // struct literal
```

### 3.15 字符串插值

详见 §1.6.5。简要：

```xray
"Hello, ${name}! Age: ${user.age + 1}"
```

- `${...}` 内任意表达式（含函数调用、对象访问、算术）。
- 嵌套字符串字面量需转义引号或换用单引号外层。
- 表达式类型必须可转为字符串（实现 `toString()` 或为基本类型）。

### 3.16 `yield` 语句

```xray
yield                       // 让出执行权
```

**当前实现**：仅支持无值语句形式，让协程让出 CPU（类似 Go 的 `runtime.Gosched()`）。

详见 §10.10。
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 3. Expressions

### 3.1 Precedence

From low to high:

1. Assignment and compound assignment.
2. Ternary `? :`.
3. Nullish coalescing and logical operators.
4. Bitwise operators.
5. Equality and comparison.
6. Type checks/casts (`is`, `as`).
7. Shifts.
8. Addition/subtraction.
9. Multiplication/division/modulo.
10. Unary operators.
11. Range.
12. Calls, member access, indexing, slicing, optional chaining, force unwrap.

### 3.2 Literals

```xray
let a = [1, 2, 3]
let m = #{"a": 1, "b": 2}
let s = #[1, 2, 3]
let obj = { name: "alice", age: 30 }
let tup = (1, "x")
```

Array literals use `[ ... ]`. Map literals use `#{ ... }` with `:` key/value separators. Set literals use `#[ ... ]`. Object/Json literals use `{ ... }`.

### 3.3 Variables and Assignment

```xray
let x = 1
x = 2
x += 3
```

Assignment requires an assignable lvalue. `const` bindings and readonly fields cannot be assigned after initialization.

### 3.4 Indexing and Slicing

```xray
arr[0]
arr[1] = 10
arr[start:end]
arr[:end]
arr[start:]
arr[:]
```

The parser distinguishes indexing from slicing by the presence of `:` inside brackets.

### 3.5 Member Access

```xray
obj.field
obj.method(arg)
tuple.0
json.class
```

After `.`, identifiers, keywords, and integer literals are accepted as member names. Integer member names are used for tuple fields.

### 3.6 Calls

```xray
f()
f(1, 2)
obj.method(x)
```

Calls evaluate the callee and arguments, then dispatch according to function, closure, native function, class constructor, static method, or instance method semantics.

### 3.7 Closures

```xray
let inc = (x: int) -> x + 1
let add = (a: int, b: int) -> {
    return a + b
}
```

Arrow closures cannot declare an explicit return type. Annotate the binding or use a named `fn` if an explicit return type is required.

### 3.8 Type Operators

```xray
if (x is string) {
    print(x)
}
let y = x as int?
```

`is` checks runtime type and can narrow static type. `as` performs a cast; nullable target types are used for safe failure paths.

### 3.9 Nullable Operators

```xray
user?.name
value!
left ?? fallback
```

Optional chaining returns `null` if the receiver is null. Force unwrap throws or fails if the operand is null. Nullish coalescing returns its right operand only when the left operand is null.

### 3.10 `new`

```xray
let p = new Point(1, 2)
let ch = new Channel<int>(10)
```

`new` constructs class, struct, and supported native/prelude values. Constructor dispatch follows nominal type semantics.

### 3.11 `move`

```xray
ch.send(move payload)
```

`move` transfers ownership across boundaries, most importantly across channels and coroutine boundaries.

### 3.12 Range Expressions

```xray
let r = 1..10
for (i in 1..10) { print(i) }
```

Ranges are half-open in normal iteration: `[start, end)`.

### 3.13 `match` Expressions

```xray
let label = match (x) {
    0 -> "zero"
    1..10 -> "small"
    _ -> "other"
}
```

The scrutinee must be parenthesized: `match (x)`. Arms use `->`. Guards use `if (condition)` after the pattern.

### 3.14 Try Expressions

```xray
let maybe = try? mayThrow()
let value = try! mayThrow()
let result = catch! { mayThrow() }
```

`try?` converts exceptions to `null`. `try!` rethrows. `catch!` wraps success/error paths into a Result-style value.

### 3.15 Throw Expressions

```xray
throw new Exception("error")
```

The operand must be statically typed as `Exception` or an `Exception` subclass.
<!-- /xr-spec:en -->
