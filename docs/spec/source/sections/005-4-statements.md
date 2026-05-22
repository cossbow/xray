---
id: spec.4_statements
order: 005
---

<!-- xr-spec:cn -->
---

## 4. 语句 (Statements)

> 真值源：`src/frontend/parser/xparse_stmt.c`、`src/frontend/parser/xast_nodes_stmt.h`。

Xray 语句以 `\n` 或 `;` 分隔；语句末尾的 `;` 在大多数位置可省略，仅 `for` 循环的初始化/条件/步进三段必须用 `;` 分隔。

### 4.1 表达式语句与块

```ebnf
ExprStmt ::= Expression (';' | LineBreak)
Block    ::= '{' Statement* '}'
```

```xray
foo()                  // 表达式语句
x = 1                  // 赋值表达式作为语句
{                      // 块
    let y = 2
    y + 1              // 表达式但结果被丢弃
}
```

**注**：块**不是表达式**——它没有值。如果需要从块求值，用 `match` 或包装成立即调用函数。

### 4.2 `if` / `else`

```ebnf
IfStmt ::= 'if' '(' Expression ')' Block ElseIfChain? ElseClause?
ElseIfChain ::= ('else' 'if' '(' Expression ')' Block)+
ElseClause  ::= 'else' Block
```

```xray
if (x > 0) {
    print("positive")
} else if (x == 0) {
    print("zero")
} else {
    print("negative")
}
```

**约束**：
- 条件**必须**用括号包裹（与 Go/Rust 不同）。
- 条件按 truthy/falsy 上下文求值（见 §2.3.3）；推荐使用显式 `bool` 表达式或 `x != null` / `x is T` 等比较以提高可读性。
- 分支体必须是块 `{...}`，**不允许**单语句省略括号。
- `if` 不是表达式；要表达式形式用三元 `? :` 或 `match`。

### 4.3 `while`

```ebnf
WhileStmt ::= 'while' '(' Expression ')' Block
```

```xray
let i = 0
while (i < 10) {
    print(i)
    i++
}
```

无 `do-while` 形式。

### 4.4 `for`（C 风格）与 `for-in`

#### C 风格 `for`

```ebnf
ForStmt ::= 'for' '(' ForInit? ';' Expression? ';' ForStep? ')' Block
ForInit ::= VarDecl | ExprStmt
ForStep ::= Expression (',' Expression)*
```

```xray
for (let i = 0; i < 10; i++) {
    print(i)
}
for (let i = 0, j = 100; i < j; i++, j--) { ... }
```

- `ForInit` 中声明的变量作用域限于循环体。
- 三个部分都可省略：`for (;;)` 是无限循环。

#### `for-in` 单变量

```ebnf
ForInStmt ::= 'for' '(' Identifier 'in' Expression ')' Block
```

```xray
for (item in [1, 2, 3]) { print(item) }
for (i in 0..n) { print(i) }                  // 范围迭代（半开区间）
for (ch in "hello") { print(ch) }             // 字符串字符（按 codepoint）
for (key in someMap) { print(key) }           // Map 单变量 → key
for (key in someJson) { print(key) }          // Json 单变量 → key
for (day in Color) { print(day.name) }        // 枚举迭代（按声明顺序）
for (_ in 0..n) { count++ }                   // 占位符忽略
```

#### `for-in` 双变量解构

xray 支持两种等价的双变量形式：

```ebnf
ForInPairStmt ::= 'for' '(' Identifier ',' Identifier 'in' Expression ')' Block
              |  'for' '(' '(' Identifier ',' Identifier ')' 'in' Expression ')' Block
```

```xray
// 形式 A：直接两标识符（更常见）
for (k, v in someMap) { print("${k}=${v}") }     // Map → (key, value)
for (i, e in someArray) { print("${i}: ${e}") }  // Array → (index, element)
for (i, c in "hello") { print("${i}:${c}") }     // string → (index, char)

// 形式 B：元组括号包裹（与 .entries() 配合）
for ((i, e) in someArray.entries()) { print("${i}=${e}") }
for ((i, c) in "hi".entries()) { print("${i}-${c}") }
```

迭代来源与产出对应关系：

| 集合类型 | 单变量产出 | 双变量产出 |
|---|---|---|
| `Array<T>` / `T[]` | element | (index, element) |
| `Map<K, V>` | key | (key, value) |
| `Json` | key (string) | (key, value) |
| `string` | char (1-codepoint string) | (index, char) |
| `Range`（`a..b`） | int | — |
| Enum 类型 | EnumValue | — |
| 自定义 `Iterator<T>` | T | — |

#### 自定义迭代器

实现 `iterator()` 方法返回 `Iterator<T>` 协议对象（含 `hasNext()` 和 `next()`）即可在 `for-in` 中使用。详见 §14.15。

### 4.5 `match` 语句

```ebnf
MatchStmt ::= 'match' '(' Expression ')' '{' MatchArm (','? MatchArm)* ','? '}'
MatchArm  ::= Pattern ('if' '(' Expression ')')? '->' (Expression | Block)
```

**关键语法**：
- 被匹配的表达式**必须**用括号包裹：`match (x) {...}`。
- 分支之间的逗号**可选**——同一个 match 中可以混用（不写更常见）。
- 守卫条件 `if` 后的表达式必须用括号：`n if (n > 0)`。

```xray
match (action) {
    "start" -> {
        log.info("starting")
        start_engine()
    }
    "stop" -> stop_engine()
    _ -> log.warn("unknown")
}
```

`match` 既可作语句也可作表达式（详见 §3.13）；当作表达式时分支体必须是单一表达式或块的最后一个表达式。

模式细节见 [§6](#6-模式-patterns)。

### 4.6 `break` / `continue`

```xray
break                  // 跳出最内层循环
continue               // 进入下一次循环
```

**约束**：
- 必须在 `while` / `for` 内部；否则编译错误 `E0304` / `E0305`。
- `match` 内部的 `break` / `continue` **不**作用于 `match`，而是跳出包裹 `match` 的循环。
- **无标签** break/continue（不像 Java/Rust）。

### 4.7 `return`

```ebnf
ReturnStmt ::= 'return' ReturnValue? (';' | LineBreak)
ReturnValue ::= Expression | '(' Expression (',' Expression)+ ')'
```

```xray
return                 // 隐式返回 ()（Unit）
return 42
return (a, b)          // 多返回值，必须用括号包裹元组
```

> **注意**：多返回值必须用元组形式 `return (a, b)`；裸逗号 `return a, b` 是编译错误（`E0801`）。

**约束**：
- 只能在函数体内（含闭包）；顶层 return 是编译错误 `E0306`。
- 返回值类型必须与函数声明的返回类型兼容。

### 4.8 `throw` / `try` / `catch` / `finally`

```ebnf
ThrowStmt ::= 'throw' Expression

TryStmt       ::= 'try' Block CatchClause* FinallyClause?
CatchClause   ::= 'catch' '(' Identifier (':' Type)? ')' Block
FinallyClause ::= 'finally' Block
```

```xray
try {
    risky()
} catch (e) {
    log.error("failed:", e.message)
} finally {
    cleanup()
}

throw new Exception("error message")        // ✅ Exception 派生
throw new HttpError(500, "internal")        // ✅ 自定义 Exception 子类
// throw "msg"                              // ❌ E0370: 必须是 Exception 派生
```

**语义**：
- `try` 必须至少跟 `catch` 或 `finally` 之一。
- `catch (e)` 中 `e` 类型默认 `Exception`；可带类型注解 `catch (e: HttpError)` 实现类型过滤；多个 `catch` 子句按声明顺序匹配。
- `finally` **保证执行**，无论是否抛异常或 return。
- `throw` 操作数静态类型必须是 `Exception` 派生（`E0370`）。
- 完整异常语义见 [§8](#8-错误处理-error-handling)。

### 4.9 `defer`

```ebnf
DeferStmt ::= 'defer' (Expression | Block)
```

```xray
fn read_file(path: string) -> string {
    let f = open(path)
    defer f.close()                  // 函数返回前必执行
    return f.readAll()
}

fn process() {
    defer {                          // 块形式
        log.info("done")
        cleanup()
    }
    do_work()
}
```

**语义**：
- 在**函数作用域结束**时执行（不是块作用域，与 Swift 不同）。
- **LIFO**：多个 `defer` 按声明的逆序执行。
- **必执行**：即使函数因异常退出也执行（类似 `finally`）。
- 与 `finally` 的区别：`defer` 绑定函数作用域，`finally` 绑定 `try` 块。
- `defer` 中的异常会**取代**当前正在传播的异常（参考 Go 语义）。

### 4.10 内置打印函数

`print` / `dump` 是**内置全局函数**（非关键字，详见 §13.1），列于此处便于查阅：

```xray
print("hello")                 // 自动追加换行
print("a:", a, "b:", b)        // 多参用空格分隔
dump(some_obj)                 // 调试输出，含类型信息与结构布局
```

**行为说明**：
- 接受任意类型与任意数量参数（变长）；每个参数自动调用其 `toString()` 或内置格式化。
- 输出到 stdout；不参与异常机制。
- 多参时以单空格分隔。
- `print` 默认会追加换行（与 C/Python 不同，与回归测试一致）。
- `dump` 用于调试，输出格式包含类型标注与对象内部结构。
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 4. Statements

### 4.1 Blocks and Expression Statements

```xray
{
    let x = 1
    print(x)
}
```

A block introduces a lexical scope. Expression statements evaluate an expression for side effects.

### 4.2 `if`

```xray
if (cond) {
    a()
} else if (other) {
    b()
} else {
    c()
}
```

Conditions are parenthesized.

### 4.3 `while`

```xray
while (cond) {
    step()
}
```

### 4.4 `for` and `for-in`

```xray
for (let i = 0; i < 10; i++) { print(i) }
for (v in values) { print(v) }
for (k, v in map) { print(k, v) }
for ((i, v) in arr.entries()) { print(i, v) }
```

Single-variable iteration yields elements/keys depending on container type. Pair iteration yields `(index, element)` for arrays/strings and `(key, value)` for maps/Json objects.

### 4.5 `match` Statement

```xray
match (action) {
    "start" -> start()
    "stop" -> stop()
    _ -> print("unknown")
}
```

`match` can be used as a statement or expression.

### 4.6 `break` and `continue`

`break` and `continue` are only valid inside loops. They are not labeled.

### 4.7 `return`

```xray
return
return 42
return (a, b)
```

Multiple return values are represented as a tuple and must be parenthesized.

### 4.8 `throw`, `try`, `catch`, `finally`

```xray
try {
    risky()
} catch (e: Exception) {
    print(e.message)
} finally {
    cleanup()
}
```

A `try` statement must have at least one `catch` or a `finally` block.

### 4.9 `defer`

```xray
fn read(path: string) -> string {
    let f = open(path)
    defer f.close()
    return f.readAll()
}
```

`defer` runs at function exit in reverse declaration order. It is function-scoped, not block-scoped.

### 4.10 `yield`

```xray
yield
```

`yield` explicitly gives the scheduler a safepoint. It does not yield a value.
<!-- /xr-spec:en -->
