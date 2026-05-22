---
id: spec.1_lexical_structure
order: 002
---

<!-- xr-spec:cn -->
---

## 1. 词法结构 (Lexical Structure)

> 真值源：`src/frontend/lexer/xlex.h`（token 枚举）、`src/frontend/lexer/xkeywords.def`（关键字表，63 条）、`src/frontend/lexer/xlex.c`（扫描器实现）。

### 1.1 字符编码

xray 源文件**必须**是 UTF-8 编码。所有源码处理（包括字符串字面量、标识符、注释）按 UTF-8 字节序列进行；非 ASCII 字符仅在字符串字面量、注释、原始字符串内部允许（标识符暂只支持 ASCII，见 §1.4）。

文件可选 UTF-8 BOM（`EF BB BF`）；扫描器跳过开头的 BOM。

### 1.2 行结尾与空白

行结尾识别 `\n`（Unix）与 `\r\n`（Windows）。`\r` 单独出现视为非法字符。

**空白字符**：空格 (`U+0020`)、水平制表符 (`U+0009`)、行结尾。空白用于分隔 token，不传递语义（**异常**：泛型语境下连续 `>>` 的拆分依赖空白上下文）。

### 1.3 注释

xray 支持两种注释，**均不嵌套**：

```xray
// 行注释，从 // 到行尾
/* 块注释，
   可跨行；
   不支持嵌套：内部出现 /* 不开启新层级 */
```

注释可出现在任何空白能出现的地方。注释会被收集为 **trivia**，供 formatter 与 LSP 使用（见 `src/frontend/parser/xtrivia.*`），但不参与语法分析。

文档注释（与普通注释无语法差异）：约定以 `///` 或 `/** */` 开头，用于工具识别。当前编译器不强制此约定。

### 1.4 标识符

```ebnf
Identifier ::= IdentStart IdentCont*
IdentStart ::= 'a'..'z' | 'A'..'Z' | '_'
IdentCont  ::= IdentStart | '0'..'9'
```

仅 ASCII。最大长度受编译器限制（约 255 字节）。

**保留约束**：标识符不能与保留关键字相同（见 §1.5）；可与**上下文敏感关键字**相同（如 `from`、`to`、`default`、`ref`、`move`、`linked`、`supervisor`、`after` 可作为普通标识符）。

字符 `_` 是**专用通配符 token**，不是普通标识符：

- 在 `match` 模式中表示**通配符**（见 §6.7）。
- 在 `for-in` 中可用于忽略键或值：`for (_, v in m) { ... }`。
- 在解构绑定中可用于忽略位置：`let (a, _) = (1, 2)`。
- **不能**作为 `let _ = expr`、函数参数名或被引用的变量名；编译器会报"expected variable name"。
- 多下划线名（如 `__tmp`）是普通标识符。

### 1.5 关键字

xray 共 **63 个保留关键字**，源码真值表见 `src/frontend/lexer/xkeywords.def`。关键字按用途分组：

#### 1.5.1 声明与流程控制

| 关键字 | 用途 |
|--|--|
| `let` | 可变变量声明 |
| `const` | 不可变变量声明 |
| `shared` | 跨协程共享修饰符（与 `const`/`let` 组合） |
| `fn` | 函数声明 |
| `return` | 函数返回 |
| `yield` | 协程让出（语句形式）|
| `if` `else` | 条件分支 |
| `while` | 循环 |
| `for` `in` | 循环（C 风格 + for-in） |
| `break` `continue` | 循环控制 |
| `match` | 模式匹配 |

#### 1.5.2 面向对象与类型

| 关键字 | 用途 |
|--|--|
| `class` `struct` | 类/结构体声明 |
| `extends` | 类继承 |
| `interface` `implements` | 接口声明/实现 |
| `enum` | 枚举声明 |
| `type` | 类型别名 |
| `new` | 实例化 |
| `this` `super` | 自我/父类引用 |
| `constructor` | 构造器 |
| `static` `private` `public` | 可见性修饰符（`public` 是**默认**，几乎从不显式写出） |
| `abstract` `final` `override` | 类/方法修饰符（`override` 是**可选**——重写父类方法不要求显式标注） |
| `operator` | 运算符重载 |
| `is` `as` | 运行时类型检查 / 转换 |

#### 1.5.3 异常处理

`try` `catch` `finally` `throw`

#### 1.5.4 模块系统

`import` `export`

#### 1.5.5 协程与并发

`go` `await` `select` `defer` `scope`

#### 1.5.6 类型名（保留）

`int` `int8` `int16` `int32` `int64` `uint8` `uint16` `uint32` `uint64`
`float` `float32` `float64` `bool` `string` `unknown`

> **注意**：以下名字**不是**词法关键字，而是 `prelude` 自动引入的内置类型符号：
> `Array` · `BigInt` · `Bytes` · `Channel` · `DateTime` · `Exception` · `Json` · `Logger` · `Map` · `NetConn` · `NetListener` · `Range` · `Regex` · `Set` · `StringBuilder`。
> `Result<T, E>` 是当前错误处理与 `catch!` 路径使用的内置 ADT enum，也可直接使用。
> 它们可被用户类同名覆盖（局部 shadow），但通常无须 import 即可使用。

#### 1.5.7 字面量关键字

`true` `false` `null`

#### 1.5.8 上下文敏感关键字

不在 lexer 关键字表中，由 parser 按位置识别。**可以**作为普通标识符使用：

| Token | 出现位置 |
|--|--|
| `from` | `select` 的接收分支 (`x from ch`)；也用于命名 import / re-export (`import { x } from "module"`) |
| `to` | `select` 的发送分支 (`value to ch`) |
| `default` | 保留，当前未启用 |
| `cancelled` | `cancelled()` 协程取消检查（实际上是 builtin 函数）|
| `ref` | 函数参数修饰符 (`fn f(p: ref T)`) |
| `move` | 所有权转移 (`move x`) |
| `linked` | `linked go` / `linked scope` 修饰符 |
| `supervisor` | `supervisor scope` 修饰符 |
| `after` | `select` 的超时分支 (`after 1000 -> ...`) |

### 1.6 字面量

#### 1.6.1 整数字面量

```ebnf
IntLiteral ::= DecLit | HexLit | OctLit | BinLit
DecLit ::= Digit (Digit | '_')*
HexLit ::= '0x' HexDigit (HexDigit | '_')*
OctLit ::= '0o' OctDigit (OctDigit | '_')*
BinLit ::= '0b' BinDigit (BinDigit | '_')*
```

- 千位分隔符 `_` 仅用于可读性，可出现在数字之间任意位置。
- 字面量默认类型为 `int`（= `int64`）。后缀 `n` 转为 `BigInt`（见 §1.6.3）。
- 范围：`int64` 表示范围 `[-(2^63), 2^63 - 1]`；溢出在编译期检测。

```xray
42
0xFF
0b1010
0o77
1_000_000      // 一百万
```

#### 1.6.2 浮点字面量

```ebnf
FloatLiteral ::= Digit+ '.' Digit* Exp?
              | Digit+ Exp
              | '.' Digit+ Exp?
Exp ::= ('e' | 'E') ('+' | '-')? Digit+
```

字面量类型为 `float`（= `float64`，IEEE-754 双精度）。

```xray
3.14
1.0e10
2.5E-3
.5             // 等价 0.5
```

#### 1.6.3 BigInt 字面量

```ebnf
BigIntLiteral ::= (DecLit | HexLit | OctLit | BinLit) 'n'
```

```xray
123n
0xFFn
0b1010n
```

任意精度整数，运算永不溢出。类型见 §14.8。

#### 1.6.4 布尔与 null 字面量

```xray
true
false
null
```

- `true` / `false`：类型 `bool`。
- `null`：类型 `null`（语义上是所有可空类型 `T?` 的零值）。

#### 1.6.5 字符串字面量

xray 支持两类字符串字面量：**带转义** 和 **原始字符串**。两者均使用双引号或单引号，且均支持 `${...}` 插值。反引号字符串不属于当前语法——lexer 直接报错。

##### 普通字符串（双引号 / 单引号）

```ebnf
StringLiteral ::= '"' StrChar* '"' | "'" StrChar* "'"
StrChar ::= 任何非引号、非反斜杠、非换行符
          | EscapeSeq
          | Interpolation
EscapeSeq ::= '\' ('"' | "'" | '\\' | 'n' | 't' | 'r' | '0'
                  | 'x' HexDigit{2}
                  | 'u' HexDigit{4}
                  | 'u{' HexDigit{1,6} '}')
Interpolation ::= '${' Expression '}'
```

- 双引号 / 单引号**完全等价**——都支持转义、`${...}` 插值。
- 字符串可跨行；行结尾包含在字符串中。
- 包含插值的字面量在 lexer 内部产出 `TK_TEMPLATE_STRING`；不含插值的产出 `TK_LITERAL_STRING`。

```xray
"hello"
'world'
"Hello, ${name}! ${1 + 2}"
'tab\there\nnewline'
"\u4F60\u597D"        // "你好"
"\u{1F600}"            // emoji
```

**插值表达式内禁止再嵌套未转义的引号字符**（lexer 限制）。

##### 原始字符串（`r` 前缀）

```ebnf
RawString ::= 'r' ('"' RawChar* '"' | "'" RawChar* "'")
RawChar ::= 任何非引号字符（包括 `\`，不做转义处理）
```

- **不**处理任何转义（`\n`、`\t` 等保持原样）。
- 仍然支持 `${...}` 插值。
- 标识符 `r` 单独使用时仍为普通标识符（`TK_NAME`），仅当后紧接引号才识别为原始字符串前缀。

```xray
r"C:\path\to\file"          // 字面量包含两个反斜杠
r'C:\Users\${USER}'         // 反斜杠不转义，但 ${USER} 仍插值
```

##### 反引号字符串（非法）

源码 lexer 显式拒绝反引号字符串。如需模板，使用普通双 / 单引号 + `${...}`。

#### 1.6.6 正则字面量

```ebnf
RegexLiteral ::= '/' RegexBody '/' RegexFlag*
RegexFlag ::= 'g' | 'i' | 'm' | 's'
```

```xray
/[a-z]+/i
/\d+\.\d+/g
```

- flags：`g`（全局）、`i`（忽略大小写）、`m`（多行）、`s`（dot 匹配换行）。
- 实现：见 `stdlib/regex`。
- **歧义消解**：当 `/` 出现在能接受一元 `/` 的位置（如紧跟 `=`、`,`、`(`、操作符），扫描器识别为正则；其他位置识别为除法。

### 1.7 操作符与 Token

完整 token 表（按类别）：

#### 1.7.1 标点

| Token | 用途 |
|--|--|
| `(` `)` | 分组、调用、参数列表 |
| `{` `}` | 块、对象字面量 |
| `[` `]` | 数组字面量、索引 |
| `,` | 分隔符 |
| `.` | 成员访问 |
| `:` | 类型注解、map kv、ternary |
| `;` | for 循环分隔（其他位置可选） |
| `?` | 可空类型、ternary |
| `@` | 注解标记 (`@test`) |

#### 1.7.2 算术

`+` `-` `*` `/` `%`

#### 1.7.3 位运算

`&` `|` `^` `~` `<<` `>>`

#### 1.7.4 比较

`==` `!=` `===` `!==` `<` `<=` `>` `>=`

- `==` `!=`：值相等（隐式数值转换：int→float）
- `===` `!==`：严格相等（类型+值；无转换）
- `<` 等：数字、字符串支持；其他类型不支持

#### 1.7.5 逻辑

`&&` `||` `!`

短路求值。

#### 1.7.6 赋值

`=` `+=` `-=` `*=` `/=` `%=` `&=` `|=` `^=` `<<=` `>>=`

#### 1.7.7 自增自减

`++` `--`

仅支持**后缀**形式 `x++` / `x--`；前缀 `++x` / `--x` 编译报错。详见 §3.2。

#### 1.7.8 类型相关

| Token | 用途 |
|--|--|
| `?` | 可空类型 (`T?`)、ternary、可选链前缀 |
| `?.` | 可选链 (`obj?.prop`) |
| `??` | 空值合并 (`a ?? b`) |
| `!` | 强制解包（后缀，`expr!`）/ 逻辑非（前缀） |
| `\|` | union 类型 (`int \| string`) / 位或 |
| `->` | 统一箭头：函数返回类型、函数类型、闭包、`match` / `select` 分支 |
| `...` | rest / spread |
| `..` | 范围 (`0..10`) |
| `is` | 运行时类型检查 |
| `as` | 类型转换 |

`!` 的歧义在 parser 阶段消解：紧跟表达式末尾且无空白时识别为强制解包；前缀位置识别为逻辑非。

#### 1.7.9 集合字面量起始符

| Token | 用途 |
|--|--|
| `#{` | 空 Map 字面量 |
| `#[` | Set 字面量起始 |

例：

```xray
let empty_map = #{}
let primes = #[2, 3, 5, 7]
```

#### 1.7.10 模式

| Token | 用途 |
|--|--|
| `_` | `match` 通配符 |

#### 1.7.11 操作符优先级

完整优先级表见 [§3.1](#31-优先级与结合性)。
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 1. Lexical Structure

### 1.1 Encoding

Xray source files are UTF-8. The lexer treats strings, comments, and raw string bodies as UTF-8 byte sequences. Identifiers are currently ASCII-only.

A UTF-8 BOM at the beginning of a file may be skipped by the scanner.

### 1.2 Line Endings and Whitespace

Line endings are `\n` or `\r\n`. A bare `\r` is not a normal line terminator.

Whitespace characters are space, horizontal tab, and line terminators. Whitespace separates tokens. In most locations it does not carry semantic meaning, but the parser may use token adjacency for a few disambiguations such as generic closing brackets.

### 1.3 Comments

Xray supports line comments and block comments:

```xray
// line comment

/* block comment
   spanning multiple lines */
```

Comments do not nest. Comments are collected as trivia for formatters/LSP but are not part of the AST semantics. Documentation comments such as `///` or `/** ... */` are conventions only.

### 1.4 Identifiers

```ebnf
Identifier ::= IdentStart IdentCont*
IdentStart ::= 'a'..'z' | 'A'..'Z' | '_'
IdentCont  ::= IdentStart | '0'..'9'
```

Identifiers are ASCII. A reserved keyword cannot be used as an identifier, except where the grammar explicitly treats it as a member name after `.`.

A single `_` is special:

- Wildcard in patterns.
- Discard marker in destructuring and selected loop positions.
- Not a normal binding name.

Names such as `__tmp` are ordinary identifiers.

### 1.5 Keywords

The lexer keyword table contains 63 reserved keywords. They include:

| Group | Keywords |
|--|--|
| Declarations and flow | `let`, `const`, `shared`, `fn`, `return`, `if`, `else`, `while`, `for`, `in`, `break`, `continue`, `match` |
| OOP and types | `class`, `struct`, `interface`, `enum`, `type`, `extends`, `implements`, `constructor`, `this`, `super`, `new`, `static`, `final`, `abstract`, `override`, `operator`, `is`, `as` |
| Error handling | `try`, `catch`, `finally`, `throw` |
| Modules | `import`, `export` |
| Concurrency | `go`, `await`, `select`, `defer`, `scope` |
| Primitive type names | `int`, `int8`, `int16`, `int32`, `int64`, `uint8`, `uint16`, `uint32`, `uint64`, `float`, `float32`, `float64`, `bool`, `string`, `unknown` |
| Literal keywords | `true`, `false`, `null` |

Context-sensitive words are parsed by position and may otherwise be used as ordinary identifiers:

| Word | Context |
|--|--|
| `from` | named import/re-export; `select` receive arm |
| `to` | `select` send arm |
| `after` | `select` timeout arm |
| `move` | ownership transfer expression |
| `ref` | parameter modifier |
| `linked` | linked coroutine/scope modifier |
| `supervisor` | supervisor scope modifier |
| `cancelled` | cancellation check call |

Prelude type names are not lexer keywords. They are automatically available type symbols: `Array`, `BigInt`, `Bytes`, `Channel`, `DateTime`, `Exception`, `Json`, `Logger`, `Map`, `NetConn`, `NetListener`, `Range`, `Regex`, `Set`, and `StringBuilder`. The built-in `Result<T, E>` ADT is used by error-handling paths such as `catch!`.

### 1.6 Literals

#### Integer Literals

```xray
0
42
1_000_000
0xff
0o755
0b1010
```

Underscores may separate digits. Supported bases are decimal, hexadecimal, octal, and binary.

#### Floating-Point Literals

```xray
3.14
.5
1e9
6.02e23
```

Floating literals have type `float`.

#### BigInt Literals

```xray
123n
0xffn
0b1010n
```

A trailing `n` creates an arbitrary-precision integer value.

#### Boolean and Null Literals

```xray
true
false
null
```

`true` and `false` have type `bool`. `null` is the singleton null value and is assignable to nullable types.

#### Strings

```xray
"hello"
'hello'
"hello ${name}"
r"C:\Users\${USER}"
r'raw string'
```

Single- and double-quoted strings are supported. Raw strings use an `r` prefix and still support interpolation. Backtick strings are not supported.

#### Regex Literals

```xray
/[a-z]+/i
/^xray$/
```

Regex literals are lexed contextually to avoid confusion with division. Runtime behavior is implemented by the regex support in the standard library/runtime.

### 1.7 Tokens and Operators

Important punctuation:

| Token | Meaning |
|--|--|
| `(` `)` | grouping, calls, parameter lists |
| `{` `}` | blocks, object/Json literals, match/select bodies |
| `[` `]` | arrays, indexing, slicing |
| `#{` | Map literal start |
| `#[` | Set literal start |
| `.` `?.` | member access and optional chaining |
| `:` | type annotations, object/Map entry separator |
| `;` | optional statement separator |
| `@` | attribute marker |
| `?` | nullable types and ternary operator |

Operators include arithmetic, bitwise, comparison, logical, assignment, increment/decrement, range, optional chaining, force unwrap, and nullish coalescing operators.

The arrow token is `->`. Xray does not use `=>` for match arms, select arms, or Map literals.
<!-- /xr-spec:en -->
