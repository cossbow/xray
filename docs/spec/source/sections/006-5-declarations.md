---
id: spec.5_declarations
order: 006
---

<!-- xr-spec:cn -->
---

## 5. 声明 (Declarations)

> 真值源：`src/frontend/parser/xparse_decl.c`、`src/frontend/parser/xast_nodes_decl.h`、`src/frontend/analyzer/xanalyzer_visitor.c`。

### 5.1 `let` / `const` / `shared`

```ebnf
VarDecl ::= ('let' | 'const' | 'shared' ('const' | 'let')) Binding (',' Binding)*
Binding ::= Pattern (':' Type)? ('=' Expression)?
Pattern ::= Identifier
         | '[' BindingPattern (',' BindingPattern)* ','? ']'    // array destructure
         | '(' BindingPattern (',' BindingPattern)+ ','? ')'    // tuple destructure
         | '{' Identifier (',' Identifier)* ','? '}'            // object destructure
```

#### 5.1.1 `let` — 可变绑定

```xray
let x = 1                         // 类型推断为 int
let name: string = "Alice"        // 显式类型
let count: int                    // 仅声明无初值：使用零值
```

- 可重新赋值。
- 必须有初值**或**类型标注；否则编译错误 `E0303`。
- 无初值的情况下，初始值为类型的零值（`int` → `0`、`string` → `""`、`bool` → `false`、`T?` → `null`）。

#### 5.1.2 `const` — 不可变绑定

```xray
const PI = 3.14159
const MAX_LEN: int = 1024
```

- **必须**有初值。
- 不能重新赋值（编译错误 `E0303`）。
- 类型可推断或显式标注。

#### 5.1.3 `shared const` — 跨协程不可变共享

```xray
shared const CONFIG = { host: "localhost", port: 8080 }
shared const PRIMES = [2, 3, 5, 7, 11]
```

- 存储在**全局堆**，refcount 管理。
- 跨协程**零拷贝**只读访问。
- 是 `go` 闭包**唯一**能合法捕获的可变作用域之外的变量种类（其他必须走参数传递或 `move`）。

#### 5.1.4 `shared let` — 跨协程可变独占

```xray
shared let buffer = new Bytes(1024)
```

- **Move 语义**：必须用 `move` 显式转移所有权。
- 不能被 `go` 闭包捕获（必须 `move`）。
- `move` 之后访问 → 编译错误。

详见 [§10.11](#1011-并发安全模型)。

#### 5.1.5 解构绑定

```xray
// 数组解构
let [a, b, c] = [1, 2, 3]
let [first, , third] = [10, 20, 30]         // 跳过元素

// 元组解构（多返回值）
let (q, r) = divmod(17, 5)

// 对象解构（仅按名提取，**不**支持重命名）
let { name, age } = { name: "Alice", age: 30 }
```

约束：
- 解构变量数必须匹配（除 rest 模式外）。
- 对象解构只接受 `Identifier` 列表，**不支持** `{ name: localName }` 风格的重命名。

### 5.2 `fn` 函数声明

```ebnf
FnDecl ::= AttrList? Modifier* 'fn' Identifier TypeParams? '(' ParamList? ')' ReturnType? Block
ParamList ::= Param (',' Param)*
Param     ::= Modifier* Identifier ':' Type ('=' DefaultValue)?
            | '...' Identifier ':' Type
Modifier  ::= 'in' | 'ref'
ReturnType ::= ':' Type
            |  ':' '(' Type (',' Type)+ ')'   // 元组返回
TypeParams ::= '<' Identifier (',' Identifier)* '>'
AttrList ::= ('@' Identifier ('(' AttrArgList? ')')?)*
```

#### 5.2.1 基本形式

```xray
fn add(a: int, b: int) -> int {
    return a + b
}

fn greet(name: string) -> () {         // 显式 Unit
    print("Hi ${name}")
}

fn echo(x: int) {                       // 省略返回类型 = ()
    print(x)
}
```

**关键**：
- 参数**必须**带类型标注（与箭头函数一致）。
- 返回类型省略 = `()`（Unit）；推荐显式标注以增强可读性。
- 函数体必须是块。

#### 5.2.2 默认参数值

```xray
fn connect(host: string, port: int = 8080, tls: bool = false) -> () { ... }

connect("localhost")              // port=8080, tls=false
connect("localhost", 443)         // tls=false
connect("localhost", 443, true)
```

- 默认值在被调函数入口求值；调用方省略参数时传入 `null`，函数入口用默认表达式替换该 `null`。
- 有默认值的参数必须在尾部连续出现。

#### 5.2.3 多返回值

```xray
fn divmod(a: int, b: int): (int, int) {
    return (a / b, a % b)
}

let (q, r) = divmod(17, 5)
let result = divmod(10, 3)        // result 类型 (int, int)
```

**约束**：
- 返回类型用括号包裹元组：`(int, bool)`。
- 单返回值不写括号：`: int`。
- `return (a, b)` 必须带括号；裸逗号 `return a, b` 是编译错误（`E0801`）。

#### 5.2.4 参数修饰符

仅适用于 **`struct` 值类型参数**。

```xray
fn length_sq(v: in Vec2) -> float {
    // v 是只读引用（不拷贝，不可修改）
    return v.x * v.x + v.y * v.y
}

fn translate(v: ref Vec2, dx: float, dy: float) -> () {
    // v 是可变引用（修改对调用方可见）
    v.x += dx
    v.y += dy
}
```

| 修饰符 | 语义 |
|--|--|
| 无 | 按值传递（struct 拷贝） |
| `in` | 按只读引用传递（不拷贝、不可写） |
| `ref` | 按可变引用传递（不拷贝、可写、修改可见） |

#### 5.2.5 rest 参数

```xray
fn sum(...nums: int) -> int {
    let total = 0
    for (n in nums) { total += n }
    return total
}

sum(1, 2, 3)        // total = 6
```

- rest 参数在参数列表**最后**。
- 类型 `...T` 内部实际是 `Array<T>`。
- 只能有一个 rest 参数。

#### 5.2.6 函数提升

```xray
main()                       // OK：函数声明被提升

fn main() { ... }
```

- 顶层 `fn` 声明被提升到当前作用域顶部。
- `let f = (x: int) -> x`（赋值给变量的箭头函数）**不**提升。

#### 5.2.7 尾递归优化

编译器自动识别 accumulator 风格的尾递归并转为循环（避免栈溢出）。详见 [§17](#17-编译流水线-compilation-pipeline)。

```xray
fn factorial(n: int, acc: int = 1) -> int {
    if (n <= 1) { return acc }
    return factorial(n - 1, acc * n)     // 尾调用：自动优化为循环
}
```

#### 5.2.8 程序入口

xray **没有隐式 `main` 入口**：脚本/模块从顶层开始顺序执行，遇到 `fn` 声明被提升注册，遇到表达式或语句被立即执行。

```xray
// hello.xr
print("loading")          // 顶层语句，立即执行
fn greet() { print("hi") }
greet()                   // 必须显式调用
```

- `fn main()` 没有任何特殊含义；如需手动调用，写 `main()`。
- 顶层不允许 `return`（编译错误 `E0306`）。
- 多文件项目的入口由 `xray.toml` 的 `entry` 字段指定，对应文件按上述脚本规则执行。

### 5.3 `class` 声明

```ebnf
ClassDecl ::= Modifier* 'class' Identifier TypeParams?
              ('extends' Identifier TypeArgs?)?
              ('implements' Identifier TypeArgs? (',' Identifier TypeArgs?)*)?
              '{' ClassMember* '}'
ClassMember ::= FieldDecl | MethodDecl | ConstructorDecl | StaticBlock
FieldDecl ::= Modifier* Identifier ':' Type ('=' Expression)?
MethodDecl ::= Modifier* Identifier '(' ParamList? ')' ReturnType? Block
            |  Modifier* 'operator' OpToken '(' ParamList? ')' ReturnType? Block
ConstructorDecl ::= 'constructor' '(' ParamList? ')' Block          // 参数类型可省
Modifier ::= 'private' | 'public' | 'static' | 'final' | 'abstract' | 'override'
```

> **关于 `public` 和 `override`**：这两个修饰符**在词法层是合法关键字**，但实际编码风格中**几乎从不使用**：
>
> - `public` 是**默认可见性**——所有未带 `private` 的字段/方法都是公开的，因此显式写 `public` 是冗余的。
> - `override` 是**可选**——重写父类方法只要同名同参就自动覆盖，不要求显式 `override` 标注。
>
> 标准库和回归测试一致采用"省略默认修饰符"风格。

#### 5.3.1 基本类

```xray
class Animal {
    name: string                       // 字段
    private _age: int = 0              // 私有字段，可有默认值

    constructor(name: string) {
        this.name = name
    }

    speak() -> string {
        return "..."
    }

    static create(name: string) -> Animal {
        return new Animal(name)
    }
}

let a = new Animal("Rex")
print(a.speak())
print(Animal.create("Bob").name)
```

#### 5.3.2 继承

```xray
class Dog extends Animal {
    constructor(name: string) {
        super(name)                    // **必须**首语句（仅限派生类）
    }

    speak() -> string {                  // 重写：不需要关键字
        return "woof"
    }
}
```

**约束**：
- 派生类构造器**第一行**必须是 `super(...)`（除非未声明构造器）；否则编译错误。
- 不能在 `super(...)` 之前访问 `this`。
- **重写父类方法不需要任何关键字**——只要子类出现同名同参的方法即自动重写（`override` 修饰符存在但**可选**）。
- 父类标 `final class` 则不可继承。
- 父类方法标 `final` 则不可重写。
- 父类方法标 `abstract` 则子类**必须**实现（除非子类也是 `abstract`）。
- `super.method()` 可在重写的方法体内调用被屏蔽的父类方法。

#### 5.3.3 修饰符

| 修饰符 | 适用 | 语义 |
|--|--|--|
| （无） | 字段/方法 | 默认 public——公开可见 |
| `public` | 字段/方法 | **冗余**——与默认相同；实际从不写出 |
| `private` | 字段/方法 | 仅类内部可访问；子类不能直接访问，但可通过父类公开方法间接访问 |
| `static` | 字段/方法 | 类级别，不属于实例；调用为 `ClassName.method()` |
| `final` | 类/方法/字段 | 类：禁止继承；方法：禁止重写；字段：初始化后不可修改 |
| `abstract` | 类/方法 | 不可实例化 / 必须由子类实现 |
| `override` | 方法 | **可选**——重写不要求显式标注；写了仅作文档作用 |

**修饰符可组合**：`private final secret: string = "key123"`、`static final pi() -> float`、`private static counter: int = 0`。

xray **没有** `protected` 修饰符——子类通过父类公开方法间接访问私有字段即可。

#### 5.3.4 构造器

```xray
class Point {
    x: float
    y: float
    constructor(x: float, y: float) {
        this.x = x
        this.y = y
    }
}

// 参数类型可省（从同名字段推断）
class Vector2 {
    x: float
    y: float
    constructor(x, y) {         // 等价于显式写 (x: float, y: float)
        this.x = x
        this.y = y
    }
}
```

- 关键字 `constructor`（不是 `init` 也不是与类同名）。
- 一个类**只有一个构造器**（不支持构造器重载）；要多种创建方式用 `static` 工厂方法。
- 构造器参数**类型可省**——若参数名与字段同名，从字段类型自动推断；其他情况推断为调用位点的实参类型。
- 构造器隐式返回 `this`（编译期注入）。
- 派生类构造器必须首行调 `super(...)`。
- struct 可以**没有**构造器（`new Point()` 创建隐式零值实例，后续手动赋值；详见 §5.4）。

#### 5.3.5 运算符重载

```xray
class Vec2 {
    x: float
    y: float

    constructor(x: float, y: float) {
        this.x = x; this.y = y
    }

    operator+(other: Vec2) -> Vec2 {
        return new Vec2(this.x + other.x, this.y + other.y)
    }

    operator==(other: Vec2) -> bool {
        return this.x == other.x && this.y == other.y
    }

    operator[](index: int) -> float {
        if (index == 0) { return this.x }
        return this.y
    }
}
```

**可重载的运算符**（完整列表，源自 `xparse_oop.c`）：

| 类别 | 运算符 | 参数数 | 备注 |
|--|--|--|--|
| 二元算术 | `+` `-` `*` `/` `%` | 1 | `-` 单参数视为一元负号 |
| 位运算 | `&` `\|` `^` `<<` `>>` | 1 | |
| 比较 | `==` `!=` `<` `<=` `>` `>=` | 1 | 一般成对实现 `==`/`!=`、`<`/`<=`/`>`/`>=` |
| 下标 | `[]`（getter）`[]=`（setter） | 1 / 2 | setter 是 `(index, value)` |
| 一元 | `!` `~` `++` `--` | 0 | |
| 复合赋值 | `+=` `-=` `*=` `/=` `%=` `&=` `\|=` `^=` `<<=` `>>=` | 1 | |

```xray
class Counter {
    n: int = 0
    operator++() -> Counter { this.n = this.n + 1; return this }
    operator+=(other: int) -> Counter { this.n = this.n + other; return this }
    operator[](i: int) -> int { return this.n + i }
    operator[]=(i: int, v: int) { this.n = v - i }
}
```

**不能**重载：`&&` `\|\|` `=` `?.` `?:` `??` `,` `.`

#### 5.3.6 自定义迭代器

实现 `iterator()` 返回带 `hasNext() -> bool` 和 `next() -> T?` 的对象即可启用 `for-in`。详见 §14.15。

### 5.4 `struct` 声明

```ebnf
StructDecl ::= 'struct' Identifier TypeParams?
               ('implements' Identifier (',' Identifier)*)?
               '{' StructMember* '}'
```

```xray
struct Point {
    x: float
    y: float

    magnitude_sq() -> float {
        return this.x * this.x + this.y * this.y
    }
}

// 两种创建方式
let p = new Point()                  // 默认构造（字段为零值）后逐个赋值
p.x = 3.0
p.y = 4.0

let q = Point{x: 3.0, y: 4.0}        // struct 字面量：类型名 + { field: value }
let pt = Point{x: 1.0, y: 2.0}

// 值语义：赋值与传参都是拷贝
let b = q                            // b 是 q 的独立拷贝
b.x = 99.0
// q.x 仍为 3.0
```

**与 `class` 的差异**：

| 维度 | `class` | `struct` |
|--|--|--|
| 内存语义 | 引用类型（堆） | 值类型（栈或内联） |
| 赋值/传参 | 共享引用 | **拷贝**（`let b = a` 生产独立副本） |
| 继承 | 支持 `extends` | **不支持**继承 |
| `implements` | ✅ | ✅ |
| 泛型 | ✅ | ✅ |
| `static` / `private` / `final` | ✅ | ✅ |
| 运算符重载 | ✅ | ✅ |
| 构造器 | `constructor(...)` | **可省略**：`new Point()` 生成零值实例 |
| 字面量 | 无 | `TypeName{field: value, ...}` |

**适用场景**：
- 数学类型（Vec2/Vec3/Quat/Color）
- 短生命周期值（迭代器状态、临时元组替代）
- 性能敏感、希望避免堆分配的数据

### 5.5 `interface` 与 `implements`

xray 接口实现是**显式声明的**（与 Go 的隐式实现不同）：类 / struct 必须用 `implements` 列出实现的接口。

```ebnf
InterfaceDecl ::= 'interface' Identifier TypeParams?
                  ('extends' NamedType (',' NamedType)*)?
                  '{' InterfaceMember* '}'
InterfaceMember ::= Identifier '(' ParamList? ')' ReturnType?       // 方法签名
                 |  ('const')? Identifier ':' Type                   // 属性签名（可加 const 表示只读）
```

```xray
interface Shape {
    area() -> float
    perimeter() -> float
}

// 接口方法返回类型可省略（默认 ()）
interface Greeter {
    greet(name: string)             // 等价于 greet(name: string) -> ()
    log()                           // 无参无返回
}

class Circle implements Shape {
    radius: float
    constructor(r: float) { this.radius = r }
    area() -> float { return 3.14 * this.radius * this.radius }
    perimeter() -> float { return 6.28 * this.radius }
}

// 实现多个接口
class Logger implements Shape, Greeter {
    radius: float
    constructor(r: float) { this.radius = r }
    area() -> float { return 3.14 * this.radius * this.radius }
    perimeter() -> float { return 6.28 * this.radius }
    greet(name: string) { print("hello,", name) }
    log() { print("logging") }
}

fn describe(s: Shape) -> string {
    return "area=${s.area()}, perimeter=${s.perimeter()}"
}
```

**约束**：

- 接口可继承其他接口（`extends`）；支持泛型 `interface Container<T>` 与受约束 `interface Stats<T: Numeric>`。
- 类 / struct 用 `implements I1, I2, ...` 声明实现一个或多个接口（**显式**，不存在隐式实现）。
- 实现类**必须**提供所有接口成员（方法同名同参同返回；属性同名同类型）。
- 接口方法声明中的**返回类型可省略**（默认 `()`）。
- 接口方法默认 `abstract`（无方法体）。
- 接口可声明**属性签名**（`length: int`、`const id: int`）；实现类必须有相应字段。
- 实现类可以提供额外的方法（接口仅定义最小集）。

```xray
// 属性签名 + 接口继承
interface HasLength {
    length: int
}
interface SizedCollection<T> extends HasLength {
    first() -> T
}

class Buffer implements SizedCollection<int> {
    length: int                       // 实现属性签名
    private data: Array<int>
    constructor(n: int) {
        this.length = n
        this.data = []
    }
    first() -> int { return this.data[0] }
}
```

### 5.6 `enum` 声明

xray 的 `enum` 是**代数数据类型 (Algebraic Data Type)**：每个变体可以是无 payload 的简单标签（C 风格枚举），也可以**携带类型化的 payload 数据**（ADT 风格）。两者可在同一个 enum 中混用。

```ebnf
EnumDecl       ::= 'enum' Identifier TypeParams?
                   ('implements' NamedType (',' NamedType)*)?
                   '{' EnumVariant (',' EnumVariant)* ','? EnumMethod* '}'
EnumVariant    ::= Identifier VariantPayload?
                |  Identifier '=' BackingValue                // 简单枚举的显式 backing value
EnumMethod     ::= 'fn' Identifier TypeParams? '(' ParamList? ')' ReturnType? Block
VariantPayload ::= '(' VariantField (',' VariantField)* ')'
VariantField   ::= (Identifier ':')? Type
BackingValue   ::= IntLiteral | FloatLiteral | StringLiteral | BoolLiteral
```

> 变体声明必须排在前面（逗号分隔），方法声明排在所有变体之后（无逗号，靠块边界分隔，与 `class` 内方法一致）。详见 §5.6.7。

#### 5.6.1 简单枚举（无 payload）

```xray
enum Color { Red, Green, Blue }
Color.Red.value     // 0
Color.Blue.value    // 2

enum HttpStatus {
    OK = 200,
    NotFound = 404,
    InternalError = 500,
}

enum Direction { North = "N", South = "S", East = "E", West = "W" }
enum Flag      { On = true, Off = false }
enum Pi        { Approximate = 3.14, Better = 3.14159 }
```

简单枚举的所有成员必须使用相同 backing type（全 int / 全 float / 全 string / 全 bool）；混合类型编译错误 `XR_ERR_ANALYZE_ENUM_MIXED_TYPE`。

#### 5.6.2 ADT 枚举（带 payload）

变体名后跟括号声明 payload 字段（位置参数或具名字段）：

```xray
// 位置 payload
enum Result<T, E> {
    Ok(T),
    Err(E),
}

// 具名字段 payload（推荐：可读性更好）
enum NetEvent {
    Connected,
    Disconnected(reason: string),
    DataReceived(bytes: Bytes),
    Error(code: int, message: string),
}

// 状态机
enum ConnState {
    Idle,
    Connecting(retry: int),
    Connected(peer: string, since: int),
    Failed(reason: string),
}

// AST 节点
enum Expr {
    Number(int),
    Binary(op: string, left: Expr, right: Expr),
    Call(name: string, args: Array<Expr>),
}
```

**ADT 与简单枚举的区别**：

| 特性 | 简单枚举 | ADT 枚举 |
|------|--|--|
| 携带数据 | ❌ | ✅ 每变体独立的字段集 |
| `.value` / `.ordinal` | ✅ | 仅对无 payload 的变体可用 |
| backing value (`= 200`) | ✅ | ❌ 不能与 payload 混用 |
| 泛型 | ❌ | ✅ `enum Result<T, E> { ... }` |
| match 解构 | 仅按值 | 按变体 + 解构 payload |
| `for-in` 遍历 | ✅ 按声明顺序 | ❌ 含 payload 时无意义 |
| 内存表示 | 整数/字符串值 | tag + payload |

混合：一个 enum 可以同时含有"无 payload"和"带 payload"的变体（见上面的 `NetEvent` / `ConnState`）。

#### 5.6.3 构造与解构

构造：

```xray
let c = Color.Red                                   // 简单
let r1 = Result.Ok(42)                              // 位置 payload
let e1 = NetEvent.DataReceived(bytes: b)            // 具名 payload，可写字段名
let e2 = NetEvent.Error(404, "not found")           // 也可省略字段名按位置传
let e3 = NetEvent.Connected                         // 无 payload 变体不写括号
```

解构（match）：

```xray
match (event) {
    NetEvent.Connected            -> print("connected"),
    NetEvent.Disconnected(reason) -> print("by:", reason),
    NetEvent.DataReceived(b)      -> process(b),
    NetEvent.Error(code, msg)     -> log.error(code, msg),
}
```

详见 §6.3。

#### 5.6.4 简单枚举的 Member API

仅适用于**无 payload** 的变体（含 ADT 中的"纯标签"变体）。

实例属性（作用在枚举值上）：

```xray
Color.Red.name        // "Red"          变体名 (string)
Color.Red.value       // 0              backing value
Color.Red.ordinal     // 0              声明顺序索引 (int，从 0)
Color.Red.toString()  // "Color.Red"    "<EnumName>.<VariantName>" 格式
```

类静态属性/方法：

```xray
Color.memberCount     // 3              简单变体总数 (int)
Color.getMember(0)    // Color.Red      按 ordinal 取
```

含 payload 的 ADT 变体**不**支持 `.value` / `.ordinal` / `getMember`，但仍可调用 `.name` 与 `toString()`（后者会带 payload 摘要，如 `Result.Ok(42)`）。

#### 5.6.5 遍历

简单枚举可被 `for-in` 按声明顺序遍历：

```xray
for (c in Color) { print(c.name) }        // "Red" "Green" "Blue"
```

含 payload 的 ADT enum **不**支持直接 `for-in`——遍历"所有可能值"无意义（`Result<int, string>` 有无穷多个）。

#### 5.6.6 反查（从值到成员）

简单整数枚举编译器优化反查（Tier 1/2 contiguous/sparse；其他类型走线性扫描）。ADT 变体不支持反查。

#### 5.6.7 enum 实例方法

`enum` 体内可定义实例方法，语法与 `class` 内的方法完全一致（不引入 `impl` 关键字）。方法在所有变体上可调用；方法体内通过 `match (this)` 区分变体行为：

```xray
enum Shape {
    Circle(radius: float),
    Rect(w: float, h: float),
    Triangle(a: float, b: float, c: float)

    fn area() -> float {
        return match (this) {
            Shape.Circle(r)     -> 3.14159 * r * r,
            Shape.Rect(w, h)    -> w * h,
            Shape.Triangle(a, b, c) -> {
                let s = (a + b + c) / 2.0
                return (s * (s-a) * (s-b) * (s-c)).sqrt()
            },
        }
    }

    fn isRound() -> bool {
        return match (this) {
            Shape.Circle(_) -> true,
            _               -> false,
        }
    }
}

let s = Shape.Circle(radius: 1.0)
print(s.area())          // 3.14159
print(s.isRound())       // true
```

> 注意 `Triangle(...)` 后没有逗号——最后一个变体与方法块之间用空白分隔（trailing comma 允许但不强制）。

**规则**：

- 方法语法与 `class` 内方法一致：`fn name(params) -> ReturnType { body }`
- 方法体内 `this` 的静态类型是 enum 自身（如 `Result<T, E>`），需要 `match (this)` 才能取出变体 payload
- **不**支持 `constructor`（变体语法本身就是构造器）
- **不**支持继承（`enum E extends ...` 是非法）；如需共享行为，用接口实现（`enum E implements Iface`）或顶层函数
- 简单枚举（无 payload）也可定义方法，但方法体内 `this` 是该 enum 的值，可用 `==` 直接比较：
  ```xray
  enum Color {
      Red, Green, Blue

      fn isWarm() -> bool { return this == Color.Red }
  }
  ```
- 方法**不能**和变体名同名
- 静态方法目前**不支持**（如需"工厂方法"请用顶层函数）

> 此设计与 Java enum / Swift enum / Kotlin sealed class 一致。Rust 的 `impl` 块在 xray 中**不**引入——xray 的方法定义统一在类型体内。

### 5.7 `type` 别名

```ebnf
TypeAliasDecl ::= 'type' Identifier TypeParams? '=' Type
```

```xray
type Outcome = int | string                          // union 别名（不要与 prelude 的 Result 重名）
type Mapper = fn(int) -> int                            // 函数类型别名
type Point = { x: float, y: float }                  // 结构化对象别名（sealed）
```

**语义**：
- 别名是**纯语法**替换，不产生新名义类型。
- `type Point = {...}` 的对象类型在使用此别名标注时**密封**：未声明的字段访问/赋值是编译错误。
- `type T = Json` 等于 `Json`（不密封）。
- 别名可前向引用，但**禁止循环别名**。
- 当前 `type` 别名不带类型参数；泛型抽象使用泛型函数、泛型 class / struct / enum / interface。

详见 [§2.4.6](#246-json) 与 [§2.8](#28-类型别名)。

### 5.8 `import` / `export`

详见 [§11](#11-模块系统-modules)。语法要点：

```xray
// stdlib / 第三方包：裸标识符，可自动生成别名
import time
import http
import alice/utils as utils

// 文件路径或目录路径：字符串，可显式 `as`，否则从路径尾段推导别名
import "./modules/mod_a.xr" as a
import "../utils/string_utils.xr" as utils
import "models/user" as user

// 命名 import：支持 quoted path 或裸模块名，成员可用 `as` 重命名
import { readFile, writeFile as write } from io
import { publicFn } from "./modules/mod_a.xr"

// 导出
export fn publicFn() -> string { return "hi" }
export const VERSION = "1.0"
export publicFn, VERSION                    // 后置 export 已声明标识符列表
export { name1, name2 as alias } from "./other"
export * from "./other"
```

**xray 不支持** JavaScript 默认导入 `import name from "module"`。使用 `import "module" as name`、`import module` 或 `import { name } from module`。

完整规则、路径解析、可见性细则见 [§11 模块系统](#11-模块系统-modules)。
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 5. Declarations

### 5.1 Variables

```xray
let x = 1
let y: int
const PI = 3.14159
shared const CONFIG = { host: "localhost" }
shared let state = 0
```

`let` creates a mutable binding. `const` creates an immutable binding and must be initialized. A declaration without an initializer needs a type annotation and receives a zero value.

`shared const` stores immutable data in a shared/global region. `shared let` is explicit mutable shared state and is subject to move/concurrency restrictions.

### 5.2 Functions

```xray
fn add(a: int, b: int) -> int {
    return a + b
}

fn log(msg: string) {
    print(msg)
}
```

A no-return function may omit `-> ()`. Function parameters use `name: Type`. Rest parameters use `...name: Type`.

### 5.3 Classes

```xray
class Animal {
    name: string

    constructor(name: string) {
        this.name = name
    }

    fn speak() -> string {
        return "..."
    }
}
```

Classes support fields, methods, constructors, inheritance, interfaces, visibility/modifier syntax, static members where implemented, and operator overload declarations where supported by the runtime/analyzer.

### 5.4 Structs

```xray
struct Point {
    x: float
    y: float
}
```

Struct declarations use class-like member syntax. They are value-oriented declarations and can implement interfaces.

### 5.5 Interfaces

```xray
interface Drawable {
    draw() -> ()
}
```

Interfaces describe method contracts. A class/struct/enum may declare `implements InterfaceName`.

### 5.6 Enums

#### Simple Backed Enums

```xray
enum Color {
    Red = "red",
    Green = "green",
    Blue = "blue"
}
```

Backed enum values may be `int`, `float`, `string`, or `bool`. All members in one backed enum must use the same backing type. Mixed backing value types are a compile-time analyzer error.

#### ADT Enums

```xray
enum Result<T, E> {
    Ok(T)
    Err(E)
}
```

ADT variants may carry payloads and are destructured with `match`.

### 5.7 Type Aliases

```xray
type Mapper = (int) -> int
type User = { name: string, age: int }
```

Aliases do not introduce new nominal types. Object aliases are sealed when used for annotations.

### 5.8 Import and Export

```xray
import time
import http as httpClient
import alice/utils as utils
import "./helper.xr" as helper
import "models/user" as user
import { readFile, writeFile as write } from io
import { publicFn } from "./helper.xr"

export fn publicFn() -> string { return "hi" }
export const VERSION = "1.0"
export publicFn, VERSION
export { publicFn as fnAlias } from "./other"
export * from "./other"
```

JavaScript default imports (`import name from "module"`) are not supported.
<!-- /xr-spec:en -->
