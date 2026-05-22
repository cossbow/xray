---
id: spec.8_error_handling
order: 009
---

<!-- xr-spec:cn -->
---

## 8. 错误处理 (Error Handling)

> 真值源：`src/runtime/error/xerror.c`、`src/vm/xvm_exception.c`、`stdlib/types/exception.xr`、`stdlib/types/result.xr`。

### 8.0 设计哲学：双轨制

Xray 同时提供两套互补的错误处理机制：

| 机制 | 适用场景 | 失败可见性 |
|--|--|--|
| **异常**（`throw` / `try` / `catch`） | 真正罕见、跨多层传播；致命错误；框架/顶层兜底 | 隐式（不污染中间层签名） |
| **Result**（`Result<T, E>` enum） | 库 API 的明确失败模式；调用方必须穷举处理；可序列化错误 | 显式（编译期可见） |

两套机制**职责互补、无重叠**：

- 函数签名**不**标 `throws`（不引入 Java/Swift 风格的"必须声明可抛异常"语义）。需要让调用方编译期看到失败可能时，请用 `Result<T, E>` 作为返回类型。
- `Exception` 是异常轨的统一基类（见 §8.1.4）；`Result<T, E>` 是 prelude enum（见 §8.2）。
- 三个糖关键字 `try!` / `try?` / `catch!` 负责两轨之间的桥接（见 §8.3）。

### 8.1 异常机制

#### 8.1.1 `throw` 表达式

`throw expr` 抛出异常。**`expr` 的静态类型必须是 `Exception` 或其派生类**。其它类型在编译期被拒绝（错误码 `E0370` `XR_ERR_ANALYZE_THROW_NON_EXCEPTION`）：

```xray
throw new Exception("oops")                 // ✅
throw new HttpError(404, "not found")       // ✅ 自定义 Exception 子类
throw "oops"                                // ❌ E0370: throw 必须是 Exception 派生
throw 42                                    // ❌ E0370
throw { code: 500 }                         // ❌ E0370
throw null                                  // ❌ E0370（静态类型为 null）
```

> **设计说明**：xray 早期允许 `throw` 任意值（字符串、整数、对象等）；自当前版本起，参考 Python 3 / Java / Swift 收紧为"必须 Exception 派生"。这与 xray "无 `any` 类型"的设计原则自洽——`catch (e)` 中 `e` 的静态类型恒为 `Exception`，工具链可稳定提供补全和类型分析。

抛出后行为：

```
抛出点 → 调用栈向上展开 → 沿途执行 defer / finally → catch 处理 → 否则继续向上 → 协程终止
```

未捕获的顶层异常会终止当前协程：

- 子协程：默认打印堆栈到 stderr，协程终止；**不**自动通知父协程（异常**不跨协程传播**，见 §8.1.6）
- 主协程：进程退出码 = 1

#### 8.1.2 `try` / `catch` / `finally`

```xray
try {
    risky()
} catch (e) {
    log.error(e.message)
} finally {
    cleanup()
}
```

**执行顺序**：

1. 执行 `try` 块
2. 若有异常逃出，按声明顺序逐一尝试匹配 `catch` 子句；首个匹配者执行其块体
3. 无论是否异常，都执行 `finally` 块（保证执行）
4. 若没有 `catch`，异常在 `finally` 之后继续向上传播

**类型化 catch 与多 catch 子句**：

`catch` 变量可带类型注解 `catch (e: T)`，运行时用 `is T` 判断是否匹配。支持多个 `catch` 子句，按声明顺序匹配，首个匹配者执行：

```xray
try {
    riskyIO()
} catch (e: HttpError) {
    log.error("HTTP:", e.statusCode)
} catch (e: DbError) {
    log.error("DB:", e.query)
} catch (e) {
    log.error("unexpected:", e.message)
}
```

**规则**：
- 无类型注解的 `catch (e)` 是"catch-all"，匹配所有异常；`e` 静态类型为 `Exception`。
- 有类型注解的 `catch (e: T)` 仅当异常值 `is T` 为真时匹配；`e` 静态类型为 `T`。
- 多个 `catch` 子句按声明顺序匹配，首个匹配者执行。
- 若所有类型化子句均不匹配且没有 catch-all，异常被自动重抛。
- 一个 `try` **必须**至少跟随 `catch` 或 `finally` 之一。

#### 8.1.3 重抛

`catch` 块内可重抛原异常或新异常：

```xray
try {
    fetch(url)
} catch (e) {
    log.error("network failed:", e.message)
    throw new ServiceError("upstream unavailable", e)  // 用 cause 链传递原因
}
```

#### 8.1.4 `Exception` 类

`Exception` 是 prelude 的内置类（声明：`stdlib/types/exception.xr`），可直接 `new`：

```xray
@native
class Exception {
    message: string             // 人类可读消息
    stack: Array<string>        // 自动 capture 的调用栈，每帧一行格式化字符串
    cause: Exception?           // 链式 cause
    code: int                   // 错误码（从 "E0xxx: ..." 前缀自动解析，默认 0）
    data: Json?                 // throw 非异常值时原始值被包装在此

    constructor(message: string = "", cause: Exception? = null)
    fn toString() -> string
}
```

字段语义：

- `message`：人类可读的错误消息（构造时传入）
- `stack`：`Array<string>`。构造时为空数组；`throw` 控制流经过其他帧时逐步 push 一行格式化帧描述（如 `"at f (line N)"`）。用户可遍历、join、取长度；可读可写但运行时不依赖用户修改。
- `cause`：可选的原因异常，用于错误链
- `code`：整形错误码。VM 报错时在 message 前缀 `"E0xxx: ..."` ，primitive ctor 解析该前缀并写入 code；用户手动 throw 时 code=0
- `data`：`Json?`。`throw <非Exception值>` 被运行时包装为 Exception 时，原始值放在此处；catch 可从 `e.data` 取回

构造方式：

```xray
throw new Exception("connection refused")
throw new Exception("upstream failed", originalErr)
```

#### 8.1.5 自定义 `Exception` 子类

定义业务错误推荐继承 `Exception`：

```xray
class HttpError extends Exception {
    statusCode: int
    constructor(statusCode: int, message: string, cause: Exception? = null) {
        super(message, cause)
        this.statusCode = statusCode
    }
}

class DbError extends Exception {
    table: string
    constructor(table: string, message: string) {
        super(message)
        this.table = table
    }
}

throw new HttpError(404, "not found")

try {
    fetchUser(id)
} catch (e: HttpError) {
    log.error("http", e.statusCode, e.message)
}
```

继承自 `Exception` 的子类自动获得 `message` / `stack` / `cause` 与 `toString()`，并可附加任意业务字段。

#### 8.1.6 异常的协程边界

异常**不跨协程传播**。子协程内未捕获的异常：

- 子协程立即终止
- 默认行为：将异常的 `toString()` + `stack` 打印到 stderr
- 父协程**不**自动感知

需要传递子协程错误时，用 Channel 显式传递：

```xray
const err_ch = Channel<Exception?>(1)

go {
    try {
        risky()
        err_ch.send(null)
    } catch (e) {
        err_ch.send(e)
    }
}

let err = err_ch.recv()
if (err != null) { log.error(err.message) }
```

或使用结构化并发的 `scope` 块（见 §10.5），让 scope 自动传播子协程异常给父协程。

#### 8.1.7 `defer`

`defer` 是资源清理语句，在所属作用域退出时**保证执行**（不论是否有异常）。语法见 §4.9。与 `try/finally` 关系：

- 二者可以混用
- 单个作用域多个 `defer` 按 **LIFO** 顺序执行
- 必发顺序（栈展开时）：
  1. 内层 `try` 的 `finally` 依次执行
  2. 当前作用域的 `defer` 按 LIFO 执行
  3. 控制权返回调用者（或异常继续向上传播）

```xray
fn fetch(url: string) -> string {
    let conn = open(url)
    defer conn.close()                       // 无论后续如何，conn 一定关闭

    try {
        return conn.read()
    } catch (e) {
        log.error(e.message)
        throw e                              // 重抛；defer 仍会执行
    }
}
```

### 8.2 `Result<T, E>`

#### 8.2.1 类型与构造

`Result<T, E>` 是 prelude 的内置 ADT enum（声明：`stdlib/types/result.xr`）：

```xray
@native
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

构造与解构：

```xray
let r1: Result<int, ParseError> = Result.Ok(42)
let r2: Result<int, ParseError> = Result.Err(ParseError.Empty)

match (r1) {
    Result.Ok(v)  -> print("got:", v),
    Result.Err(e) -> print("failed:", e),
}
```

#### 8.2.2 错误类型 `E` 的选择

`E` 可以是任意类型；推荐风格：

| 错误形态 | E 类型 | 例子 |
|--|--|--|
| 多种可枚举的失败原因 | 用户定义的 ADT enum | `enum ParseError { Empty, NotNumber(s: string), Overflow }` |
| 单一字符串错因 | `string` | `Result<int, string>` |
| 异常对象（与异常轨桥接） | `Exception` 派生 | `Result<T, Exception>` |

**强烈推荐用 ADT enum**——可让 `match` 在编译期检查错因穷举性。

#### 8.2.3 `Result` 上的方法

`Result<T, E>` 是带方法的 enum（见 §5.6.7）。完整签名（声明：`stdlib/types/result.xr`）：

```xray
@native
enum Result<T, E> {
    Ok(T),
    Err(E)

    fn isOk() -> bool
    fn isErr() -> bool

    fn ok() -> T?                                          // Ok(v) -> v；Err -> null
    fn err() -> E?                                         // Err(e) -> e；Ok -> null

    fn unwrap() -> T                                       // Err 时 throw（要求 E 是 Exception 派生）
    fn unwrapOr(default: T) -> T                           // Err 时返回 default
    fn unwrapOrElse(handler: (E) -> T) -> T                // Err 时由 handler 计算

    fn map<U>(transform: (T) -> U) -> Result<U, E>         // 转换 Ok 内的值
    fn mapErr<F>(transform: (E) -> F) -> Result<T, F>      // 转换 Err 内的值
    fn andThen<U>(transform: (T) -> Result<U, E>) -> Result<U, E>  // 链式（flatMap）
}
```

`map` / `mapErr` 是"不破壳换内里"的快捷方式，避免每次 `match` 套话：

```xray
// 不用 map：5 行
let r2: Result<float, ParseError>
match (parseInt(s)) {
    Result.Ok(n)  -> r2 = Result.Ok(n.toFloat()),
    Result.Err(e) -> r2 = Result.Err(e),
}

// 用 map：1 行
let r2 = parseInt(s).map(n -> n.toFloat())
```

#### 8.2.4 错误类型转换：必须显式 `mapErr`

跨层 `Result` 组合时，**不**自动转换 Err 类型。调用方必须显式 `.mapErr(...)`：

```xray
fn loadConfig(text: string) -> Result<Config, ConfigError> {
    let json = try! parseJson(text).mapErr(e -> ConfigError.BadJson(e))
    //                              ^^^^^^^^ 必须显式：ParseError → ConfigError
    let port = try! json["port"].toInt().mapErr(e -> ConfigError.BadField("port", e))
    return Result.Ok(Config(port: port))
}
```

每个 `.mapErr(...)` 是一行**人类可读的错误升级路径**——优于 Rust `From::from` 的隐式转换。

### 8.3 桥接：`try!` / `try?` / `catch!`

三个糖关键字覆盖所有两轨之间的桥接，同时也是同轨道内的早退糖。

#### 8.3.1 `try! e` —— 早退或跨轨升级

`try!` 后面的表达式静态类型**必须**是 `Result<T, E>` 或 `T?`。其它类型编译报错（`E0821 XR_ERR_TRY_BANG_BAD_OPERAND`）。

行为按 `e` 类型 + 当前函数返回类型双重 dispatch：

| `e` 类型 | 当前函数返回 | 行为 |
|--|--|--|
| `Result<T, E>` | `Result<_, E>` | `Err(e)` → `return Result.Err(e)`；`Ok(v)` → `v` |
| `Result<T, E>` | 其它 | `Err(e)` → `throw e`（要求 `E` 是 `Exception` 派生，否则 `E0822`）；`Ok(v)` → `v` |
| `T?` | `_?` | `null` → `return null`；`v` → `v` |
| `T?` | 其它 | `null` → `throw new NullThrowError("try! on null")`；`v` → `v` |

例子：

```xray
// 同轨道早退
fn parsePair(s: string) -> Result<(int, int), ParseError> {
    let parts = s.split(",")
    if (parts.length != 2) return Result.Err(ParseError.Empty)
    let a = try! parseInt(parts[0])              // Err 早退 Err
    let b = try! parseInt(parts[1])
    return Result.Ok((a, b))
}

// 跨轨道升级（Result → 异常）
fn dangerous(s: string) -> int {
    let n = try! parseInt(s)                     // Err 抛异常
    return n * 2
}

// Optional 早退
fn lookupTwo(m: Map<string, int>, k1: string, k2: string) -> int? {
    let v1 = try! m.get(k1)                      // null 早退 null
    let v2 = try! m.get(k2)
    return v1 + v2
}
```

> **`try!` 不是异常调用的强制仪式**——xray **不**像 Swift 那样要求每个会抛的调用前都写 `try`。`try!` 只用于 `Result` / `Optional` 的早退或升级。

#### 8.3.2 `try? e` —— 失败转 null

`try?` 把任何失败统一转为 `null`，丢弃错因。

| `e` 类型 | `try? e` 类型 | 行为 |
|--|--|--|
| `Result<T, E>` | `T?` | `Err` → `null`（错因丢弃）；`Ok(v)` → `v` |
| 普通可抛调用，类型 `T` | `T?` | 抛异常 → `null`；正常 → `v` |
| `T?` | `T?` | 透传 |

```xray
let n: int? = try? parseInt(s)              // Err 转 null
let v: Json? = try? http.get(url).json()    // 抛异常转 null
```

`try?` 适合"调用方不关心错因，只想要默认值"的场景，常与 `??` 搭配：

```xray
let port = (try? parseConfig(text).map(c -> c.port)) ?? 8080
```

#### 8.3.3 `catch! { ... }` —— 异常块凝结成 Result

`catch!` 把一段可能抛异常的代码块凝结成 `Result<T, Exception>`：

```xray
let r: Result<Server, Exception> = catch! {
    let cfg = loadConfig(path)
    let conn = openDb(cfg.dbUrl)
    return startServer(cfg, conn)
}

match (r) {
    Result.Ok(s)  -> s.run(),
    Result.Err(e) -> log.error("startup failed:", e.message),
}
```

规则：

- 块的最后一个表达式或 `return v` 作为 `Ok(v)` 的值
- 块内**逃出的任何异常**被捕获并包装为 `Err(e)`
- `e` 的静态类型为 `Exception`（即结果恒为 `Result<T, Exception>`）
- 块内 `return v` 是"从 `catch!` 块返回"，**不是**从外层函数返回
- `defer` 在块内仍按规则执行
- 不支持类型过滤——需要过滤特定异常请用手写 `try` / `catch`

#### 8.3.4 `Result.unwrap()` —— Result 升级为异常

`unwrap()` 是反方向桥接：把 `Err(e)` 变成 `throw e`：

```xray
let cfg = loadConfig(text).unwrap()         // Err 时抛异常（要求 E 是 Exception 派生）
```

若 `E` 不是 `Exception` 派生，编译期报错（建议使用 `unwrapOr(...)` 或 `match`）。

#### 8.3.5 桥接矩阵

```
                   ┌──────────────────────────────────────────┐
                   │  ↓ 异常轨 (throw / catch)                 │
                   │                                          │
       catch! { } ─┤◄── 异常 → Result<T, Exception>          │
                   │                                          │
       try? expr  ─┤◄── 异常 → T? (丢弃错因)                 │
                   │                                          │
       try { } catch (e) { ... }    完整异常控制流          │
                   └──────────────────────────────────────────┘
                                    ▲
                                    │  unwrap / unwrapOr / try!
                                    │
                   ┌──────────────────────────────────────────┐
                   │  ↑ Result 轨 (Result<T, E>)              │
                   │                                          │
       try! result ─┤── 同轨道早退 / 跨轨抛异常              │
                   │                                          │
       try? result ─┤── Err → null (丢弃错因)                │
                   │                                          │
       result.ok()  ─── Result<T,E> → T?                     │
                   └──────────────────────────────────────────┘
```

### 8.4 Optional 与错误处理

`T?` 是 `T | null` 的语法糖，用于"二态：有值/无值"场景。详见 §2.5。与错误处理的关系：

- **不带错因的失败**：用 `T?` 比 `Result<T, ()>` 更简洁
- 与 `try!` / `try?` 协同（见 §8.3.1 / §8.3.2）
- 与 `??`（默认值）/ `?.`（可选链）/ `e!`（强制解包）配合
- 不要把 `T?` 当通用错误返回——需要错因时用 `Result<T, E>`

### 8.5 决策树：选择哪种机制

按"**调用方对失败的处理需求**"选择：

```
失败需要被调用方处理吗？
│
├─ 不需要（致命、不可恢复、跨多层兜底）
│   ↓
│   throw / try-catch
│
├─ 需要，且失败有结构化错因，调用方应穷举处理
│   ↓
│   Result<T, E>，E 用 ADT enum
│
├─ 需要，但失败只表示"没值"，无错因意义
│   ↓
│   T? + ?? / ?. / try?
│
├─ 需要，且函数有 ≥3 种正常状态（不只是"成功/失败"）
│   ↓
│   用户 ADT enum 直接作为返回类型
│
└─ 需要返回多个对等值（不是"成功/失败"二元）
    ↓
    tuple (a, b, ...)
```

完整对照：

| 场景 | 推荐 | 例 |
|--|--|--|
| 解析、解码、状态转换 | `Result<T, E>` | `parseInt(s) -> Result<int, ParseError>` |
| 字典查找、可选字段 | `T?` | `map.get(k) -> Value?` |
| IO、网络、不可恢复 | `throw` + 顶层 `catch` | `readFile(p) -> Bytes`（可能抛 IOError） |
| 多分支结果 | enum | `nextEvent() -> NetEvent` |
| 主结果 + 元数据 | tuple | `parse(s) -> (Ast, int)` // ast + 已读字节数 |

### 8.6 常用模式

#### 模式 1：库 API 用 Result，业务边界用异常

```xray
// 库层：Result
fn parseConfig(text: string) -> Result<Config, ConfigError> {
    let json = try! parseJson(text).mapErr(e -> ConfigError.BadJson(e))
    return Result.Ok(Config(port: json["port"].toInt().unwrap()))
}

// 业务层：组合 Result
fn loadConfig(path: string) -> Result<Config, ConfigError> {
    let text = readFile(path)                    // 可能抛 IOError（异常）
    return parseConfig(text)
}

// 顶层：异常 catch 兜底（必须显式调用，没有自动 main）
fn main() {
    try {
        let cfg = loadConfig("app.toml").unwrap()
        startServer(cfg)
    } catch (e: IOError) {
        log.error("config missing:", e.message)
        exit(1)
    } catch (e: ConfigError) {
        log.error("bad config:", e)
        exit(2)
    }
}

main()                                     // 显式调用入口
```

#### 模式 2：异常块凝结成 Result（用于 RPC / 序列化）

```xray
fn handleRequest(req: Request) -> Response {
    let r: Result<Json, Exception> = catch! {
        let user = db.query(req.userId)         // 可能抛 DbError
        return user.toJson()                    // 可能抛 SerializeError
    }
    match (r) {
        Result.Ok(json) -> Response.success(json),
        Result.Err(e)   -> Response.error(500, e.message),
    }
}
```

#### 模式 3：`try?` + `??` 提供默认值

```xray
let port = (try? parseConfig(text).map(c -> c.port)) ?? 8080
let user = (try? db.findUser(id)) ?? guestUser
```
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 8. Error Handling

### 8.1 Exceptions

```xray
throw new Exception("message")
```

Only `Exception` and subclasses may be thrown. Throwing a string, number, Json value, or null is a compile-time error.

`Exception` fields:

| Field | Meaning |
|--|--|
| `message` | human-readable message |
| `stack` | stack trace lines |
| `cause` | optional chained cause |
| `code` | numeric error code |
| `data` | optional structured payload |

### 8.2 Try/Catch/Finally

```xray
try {
    risky()
} catch (e: HttpError) {
    handle(e)
} catch (e) {
    fallback(e)
} finally {
    cleanup()
}
```

`finally` executes during both normal and exceptional exits.

### 8.3 `try?` and `try!`

`try? expr` converts an exception into `null`. `try! expr` rethrows the original exception.

### 8.4 `catch!`

`catch! { ... }` evaluates a block and wraps success/failure into a Result-style value when the built-in Result enum is available.

### 8.5 `defer`

`defer` is a resource-management tool that runs at function exit and is compatible with exceptional exits.
<!-- /xr-spec:en -->
