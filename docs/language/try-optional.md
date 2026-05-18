# try? 与 try! — 异常处理与可空类型的统一

Xray 提供了 `try?` 和 `try!` 两个表达式，将异常处理和可空类型（`T?`）无缝衔接。它们不引入新能力——所有功能用 `try-catch` 都能实现——但大幅减少了样板代码，让错误处理和空值处理共用同一套操作符家族（`?.`、`??`、`match`）。

## 快速参考

| 语法 | 含义 | 结果类型 |
|------|------|----------|
| `try? expr` | 执行表达式；抛异常时返回 `null` | `T?` |
| `try! expr` | 执行表达式；抛异常时重抛原异常（**不**是 abort） | `T` |

## 与已有操作符的配合

```xray
// try? 的结果是 T?，可以直接接入可空操作符链
let name = (try? readConfig())?.user?.name ?? "default"
//          ^^^^                ^^          ^^
//          异常→null          可选链       空值合并
```

这三个操作符形成完整的错误处理链：
- `try?` — 异常 → `null`
- `?.` — 空值传播
- `??` — 兜底默认值

---

## 对比 1：读取配置，失败用默认值

这是最常见的模式——尝试做一件事，失败了就用预设值。

**传统写法：**

```xray
let config: Json
try {
    config = Json.parse(readFile("config.json"))
} catch (e) {
    config = { port: 8080, host: "localhost" }
}
```

需要先声明变量，再在 try-catch 两个分支中分别赋值。变量的作用域被迫提升到 try 外部。

**使用 `try?`：**

```xray
let config = try? Json.parse(readFile("config.json")) ?? { port: 8080, host: "localhost" }
```

一行完成。变量声明和初始化在同一处，意图清晰。

---

## 对比 2：链式取值 — 读文件 → 解析 → 取嵌套字段

**传统写法：**

```xray
let dbHost = "localhost"
try {
    let content = readFile("config.json")
    try {
        let config = Json.parse(content)
        if (config.database != null) {
            if (config.database.host != null) {
                dbHost = config.database.host
            }
        }
    } catch (e) { }
} catch (e) { }
```

嵌套了两层 try-catch 加两层空检查，真正的业务意图被淹没在错误处理代码中。

**使用 `try?`：**

```xray
let dbHost = (try? Json.parse(readFile("config.json")))?.database?.host ?? "localhost"
```

`try?` 把异常折叠成 `null`，`?.` 沿路传播空值，`??` 在最后兜底。一行替代十几行。

---

## 对比 3：多个独立操作，各自可能失败

**传统写法：**

```xray
let name = "unknown"
let age = 0
let email = ""

try {
    name = fetchUserName(id)
} catch (e) { }

try {
    age = int(fetchUserAge(id))
} catch (e) { }

try {
    email = fetchUserEmail(id)
} catch (e) { }
```

三个空 `catch` 块，视觉噪音严重。每个变量都要提前声明默认值，再在 try 内覆盖。

**使用 `try?`：**

```xray
let name  = try? fetchUserName(id)     ?? "unknown"
let age   = try? int(fetchUserAge(id)) ?? 0
let email = try? fetchUserEmail(id)    ?? ""
```

三行对齐，默认值紧跟在同一行，一目了然。

---

## 对比 4：条件分支 — 能解析就用，否则跳过

**传统写法：**

```xray
let port: int? = null
try {
    let val = readEnv("PORT")
    port = int(val)
} catch (e) { }

if (port != null) {
    server.listen(port)
} else {
    server.listen(8080)
}
```

**使用 `try?`：**

```xray
let port = try? int(readEnv("PORT"))
server.listen(port ?? 8080)
```

---

## 对比 5：批量处理，跳过失败项

**传统写法：**

```xray
let results: Array<Json> = []
for (path in files) {
    try {
        let content = readFile(path)
        let parsed = Json.parse(content)
        results.push(parsed)
    } catch (e) {
        // skip
    }
}
```

空的 `catch(e) { }` 是一个常见的反模式——它隐藏了错误，但你确实就是想跳过。

**使用 `try?`：**

```xray
let results: Array<Json> = []
for (path in files) {
    let parsed = try? Json.parse(readFile(path))
    if (parsed != null) { results.push(parsed) }
}
```

不再有空 catch 块。"跳过失败项"的意图由 `try?` + `null` 检查自然表达。

---

## 对比 6：确信不会失败 — `try!`

有些操作在逻辑上不可能失败（例如读取程序自带的资源文件）。但传统写法无法在代码中表达这种信心。

**传统写法：**

```xray
// 这个文件是程序自带的，不可能缺失
let schema = Json.parse(readFile("/app/schema.json"))
// 但如果真出了问题，异常会默默传播到上层，难以定位
```

异常会沿调用栈向上传播，可能在完全无关的地方被捕获或导致崩溃，错误信息远离出错点。

**使用 `try!`：**

```xray
let schema = try! Json.parse(readFile("/app/schema.json"))
// 语义明确：如果失败就立即重抛原异常，不会静默吞下
// 读代码的人一眼知道："作者认为这里不应该失败"
```

`try!` 是一种文档化的断言，它告诉读者和编译器：这里的失败是 bug，不是预期的业务错误。

> **与 Swift 的区别**：这里的 `try!` **重抛原异常**，不会 `abort`/`panic`；什么都不包装。效果等价于 "不写 try? ，允许异常照常传播"——作为一种**意图标记**存在。

---

## 对比 7：函数返回可空值

很多函数的语义就是"能查到就返回，查不到就返回 null"。传统写法需要手动把异常转成 null。

**传统写法：**

```xray
fn findUser(id: int): Json? {
    try {
        return db.query("SELECT * FROM users WHERE id = ${id}")
    } catch (e) {
        return null
    }
}
```

**使用 `try?`：**

```xray
fn findUser(id: int): Json? {
    return try? db.query("SELECT * FROM users WHERE id = ${id}")
}
```

函数体缩减为一行，返回类型 `Json?` 与 `try?` 的语义完全对应。

---

## 对比 8：与 `match` 结合

**传统写法：**

```xray
let result: int? = null
try {
    result = int(input)
} catch (e) { }

match (result) {
    null => print("not a number"),
    n => print("got: ${n}")
}
```

**使用 `try?`：**

```xray
match (try? int(input)) {
    null => print("not a number"),
    n => print("got: ${n}")
}
```

`try?` 的结果直接作为 `match` 的输入，不需要中间变量。

---

## 对比 9：多层嵌套 — 真实场景的极端案例

**需求：** 读 XML 配置 → 解析 → 取 server 节点 → 取 port 属性 → 转整数 → 不在合法范围就用默认值。

**传统写法：**

```xray
fn getServerPort(path: string): int {
    let port = 8080
    try {
        let content = readFile(path)
        try {
            let doc = Xml.parse(content)
            let server = doc.find("server")
            if (server != null) {
                let portStr = server.getAttribute("port")
                if (portStr != null) {
                    try {
                        let p = int(portStr)
                        if (p >= 1 && p <= 65535) {
                            port = p
                        }
                    } catch (e) { }
                }
            }
        } catch (e) { }
    } catch (e) { }
    return port
}
```

18 行，3 层 try-catch 嵌套，2 层 null 检查。真正的业务逻辑（取 port、校验范围）被深埋在缩进中。

**使用 `try?`：**

```xray
fn getServerPort(path: string): int {
    let p = try? int(
        (try? Xml.parse(readFile(path)))?.find("server")?.getAttribute("port") ?? ""
    )
    return (p != null && p >= 1 && p <= 65535) ? p : 8080
}
```

3 行，零嵌套。每一步的失败都被 `try?` 和 `?.` 自然吸收，最终汇聚到一个简单的范围检查。

---

## 总结

| 模式 | 传统写法 | 使用 `try?` / `try!` |
|------|----------|---------------------|
| 异常 → 默认值 | `try { x = expr } catch { x = default }` | `x = try? expr ?? default` |
| 异常 → null | `try { return expr } catch { return null }` | `return try? expr` |
| 链式可能失败 | 嵌套 try-catch + if 检查 | `(try? a)?.b?.c ?? d` |
| 确信不失败 | 无标注，异常默默传播 | `try!`（重抛原异常，作为意图标记） |
| 空 catch 块 | `catch(e) { }` 散落各处 | 完全消失 |
| 代码量 | 3–10 倍 | 基准 |

### 核心理念

`try?` 的价值不在于它能做什么新事情——它做的一切 `try-catch` 都能做。它的价值在于**消除了"异常"和"可空"之间的语法断裂**：

- 异常是"操作可能失败"
- 可空是"值可能不存在"

这两者在业务语义上常常是同一件事。`try?` 让它们共用 `?.`、`??`、`match` 这套已有的工具链，而不是强迫开发者在两套语法之间反复切换。
