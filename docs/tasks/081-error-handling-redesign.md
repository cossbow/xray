# Error Handling Redesign — 错误处理重大重设计

> Spec 真相源：`docs/rules/language-spec.md` §8（已重写）；本任务把 spec 落到实现。
> 涉及：lexer / parser / analyzer / IR / VM / GC / stdlib types / prelude / LSP / MCP 知识库。

## 目标

把 xray 错误处理升级为**双轨制**：

1. **异常轨**（`throw` / `try-catch`）：罕见路径、跨多层、致命错误；操作数收紧到 `Exception` 派生
2. **Result 轨**（`Result<T, E>` ADT enum）：可枚举失败、调用方必须穷举处理
3. **桥接糖**：`try!` / `try?` / `catch!` 三个关键字覆盖两轨之间所有桥接

## 设计决策（已敲定）

| ID | 决策 | 选择 |
|----|------|------|
| D1 | `throw` 操作数限制 | **必须**是 `Exception` 派生（`E0820`）；不允许字符串、数字、Json、数组 |
| D2 | `Exception` 类 | 可 `new`、可继承；构造器自动 capture stack；新增 `cause` 字段 |
| D3 | ADT enum | 引入 variant payload（位置或具名字段） |
| D4 | `Result<T, E>` | prelude ADT enum，含完整方法集 |
| D5 | `try!` 语义 | 操作数限定 `Result<T,E>` 或 `T?`；按当前函数返回类型 dispatch |
| D6 | `catch! { ... }` | 块凝结为 `Result<T, Exception>`；不支持类型过滤 |
| D7 | `throws` 函数签名 | **不**引入（避免 Java checked exception 失败教训） |
| D8 | tuple `(T, E?)` 风格 | 不强制迁移；按 API 性质选 |
| D9 | Result Err 自动转换 | **不**自动；必须显式 `.mapErr(...)` |
| D10 | `T?` 底层表示 | 保持 union（`T | null`），不改 ADT |
| D11 | Result API | 链式（`map` / `mapErr` / `andThen` / ...）+ `match` 双支持 |

## 范围与阶段

### Phase 1：stdlib 类型升级（独立、最小改动）

工作内容：

- **`stdlib/types/exception.xr`** 加构造器 + cause 字段：
  ```xray
  @native
  class Exception {
      message: string
      stack: string
      cause: Exception?
      constructor(message: string = "", cause: Exception? = null)
      fn toString() -> string
  }
  ```
- **C 侧 native 实现**：
  - `xr_exception_constructor(...)`：分配 Exception 实例，写入 `message` / `cause`，调用 VM `xvm_capture_stack()` 写 `stack`
  - 注册到 prelude 类符号表（`stdlib/prelude/prelude_types.def`），让 `Exception` 名字在用户作用域可见
- **新增 `stdlib/types/result.xr`**（enum 体内定义方法，见 spec §5.6.7；不引入 `impl` 关键字）：
  ```xray
  @native
  enum Result<T, E> {
      Ok(T),
      Err(E)

      fn isOk() -> bool
      fn isErr() -> bool
      fn ok() -> T?
      fn err() -> E?
      fn unwrap() -> T
      fn unwrapOr(default: T) -> T
      fn unwrapOrElse(handler: (E) -> T) -> T
      fn map<U>(transform: (T) -> U) -> Result<U, E>
      fn mapErr<F>(transform: (E) -> F) -> Result<T, F>
      fn andThen<U>(transform: (T) -> Result<U, E>) -> Result<U, E>
  }
  ```
  > **依赖 Phase 2 的 ADT enum + enum 方法支持**——Phase 1 只能完成 .xr 声明文件；C 侧 lower 与 prelude 注册放 Phase 2。

- 更新 `scripts/gen_stdlib_types.py` 让它能处理 ADT variant（生成 MCP 知识库）

**验收**：
- `./build/xray -e 'let e = new Exception("hi"); print(e.message)'` 输出 `hi`
- `Exception` 可作为类型注解：`fn f() -> Exception?`
- `try { ... } catch (e: Exception)` 可静态检查

### Phase 2：parser / analyzer / IR — ADT enum payload

工作内容：

**Parser** (`src/frontend/parser/`)：

- `xparse_decl.c::xr_parse_enum_decl`：
  - 变体语法：扩展为支持 `Identifier '(' VariantField (',' VariantField)* ')'`，其中 `VariantField ::= (Identifier ':')? Type`（位置或具名）
  - 泛型：支持 `enum Name<T, U> { ... }`
  - **方法**：变体列表后允许跟方法定义（`'fn' ...`），与 class 内方法语法一致。变体之间逗号分隔；最后一个变体后逗号可选；方法之间无逗号（与 class 内方法一致）
  - 实现：parser 状态机里读完变体（识别 `,` 或下一个 `fn`/`}`）后切换到方法解析模式
- `xparse_match.c::xr_parse_pattern`：扩展 `EnumPattern` 支持 payload 解构 `Result.Ok(v)` / `NetEvent.Error(code, _)` / 嵌套
- `xast_types.h`：新增 AST 节点 `AST_ENUM_VARIANT_PAYLOAD` / `AST_ENUM_PATTERN_PAYLOAD`；enum decl 节点扩展 `methods` 字段（复用 `AST_FN_DECL`）

**Analyzer** (`src/frontend/analyzer/`)：

- 类型推断：`enum Result<T, E>` 的泛型实例化、`Result.Ok(v)` 的类型推回、`match` payload binding 类型
- **enum 方法解析**：方法体内 `this` 类型 = 当前 enum 的实例化类型；`match (this)` 时按变体推 payload 字段
- **穷举性检查** (`xanalyzer_match_exhaustive.c` 新文件)：ADT enum 的 match 必须覆盖所有变体或带 `_`；缺失时报 `E0823`
- 错误码：新增 `XR_ERR_ANALYZE_ENUM_PAYLOAD_MISMATCH`、`XR_ERR_MATCH_NOT_EXHAUSTIVE`

**IR / Lower** (`src/ir/`)：

- `xi_lower_expr.c`：lower `Result.Ok(v)` 为 tag(=0) + payload box；lower `match` 为 tag-test + 顺序比较
- `xi_types.c`：enum payload 的运行时表示决定（建议起步用 boxing：每变体堆分配 `{ tag, payload }`，后期再做 inline / niche optimization）

**VM** (`src/vm/`)：

- `xvm_value.h`：新增 `XR_TAG_ENUM_PAYLOAD` 值类型
- `xvm_op_enum.c` 新文件：constructor / extract / tag-test 三类指令实现
- GC 写屏障：payload 含引用时正确扫描

**验收**：
- 简单 ADT：`enum E { A(int), B(string) }` 可定义、构造、match
- 泛型 ADT：`Result<int, string>` 可定义、构造、match
- enum 含方法：`enum Shape { Circle(float), Rect(float,float)  fn area() -> float { match(this){...} } }` 可定义、调用 `s.area()`
- 穷举性：漏分支编译报错 `E0823`，提示缺失变体名
- 嵌套：`Result.Ok(NetEvent.DataReceived(b))` 可解构
- 性能：成功路径无堆分配（仅 Ok/Err 包装本身）

### Phase 3：throw 收紧 + try!/try?/catch! 语义

工作内容：

**Lexer** (`src/frontend/lexer/`)：

- 已有 `try` 关键字 + `!` / `?` 后缀；需要让 lexer 识别 `try!` / `try?` 为单 token？或保持两个 token 由 parser 组合？
- 新增：`catch!` 复合 token（或保持 `catch` + `!` 由 parser 识别）

**Parser**：

- `xparse_expr.c`：
  - `xr_parse_throw`：保留语法不变（已有）；类型检查放 analyzer
  - `xr_parse_try_expr`：保持 `try?` / `try!` 表达式语法
  - `xr_parse_catch_block`：新增 `catch! { ... }` 语法解析为 `AST_CATCH_BLOCK`
- 优先级：`catch! { }` 是 primary expression（同 match expression 级别）

**Analyzer**：

- `throw expr`：检查 `expr` 静态类型必须是 `Exception` 或派生（含 union 中所有分支）；否则报 `E0820`
- `try! expr`：
  - 检查 `expr` 类型必须是 `Result<T, E>` 或 `T?`（否则 `E0821`）
  - 推断当前函数返回类型，确定是早退还是抛异常
  - 跨轨升级时若 `E` 不是 `Exception` 派生，报 `E0822`
- `try? expr`：扩展原有逻辑，对 `Result<T, E>` 也接受
- `catch! { block }`：推断块的成功值类型 `T`，结果类型 `Result<T, Exception>`
- `Result.unwrap()`：当 `E` 不是 `Exception` 派生时报 `E0824`

**IR / Lower**：

- `xi_lower_expr.c::lower_try_expr`：扩展支持 Result/Optional 的早退
- 新增 `lower_catch_block`：把块包装成 try-catch + Result 构造
- `lower_throw`：编译期检查通过后照常 lower（无运行时变化）

**VM**：

- 异常路径已有，无需改动
- `try!` Result 早退：lower 为 if-tag-test + return；无新指令
- `catch!` 块：lower 为 try-table + catch handler 包装成 `Result.Err`

**错误码** (`src/runtime/error/xerror_codes.h`)：

```c
XR_ERR_THROW_NOT_EXCEPTION       0x0820
XR_ERR_TRY_BANG_BAD_OPERAND      0x0821
XR_ERR_TRY_BANG_NON_EXCEPTION_ERR 0x0822
XR_ERR_MATCH_NOT_EXHAUSTIVE      0x0823
XR_ERR_UNWRAP_NON_EXCEPTION_ERR  0x0824
```

**验收**：
- `throw "msg"` → 编译报 `E0820`
- `throw new Exception("msg")` → 运行时正常抛
- `try! 42` → 编译报 `E0821`
- `Result<int, ParseError>` 在返回 `int` 的函数里 `try!` → 跨轨升级失败（`E0822`，因为 `ParseError` 不是 `Exception`）
- `catch! { throw new Exception("x") }` → 得到 `Result.Err` 包装

### Phase 4：回归测试 + 文档示例迁移

- 新增 `tests/regression/error-handling/`：覆盖每个错误码、桥接矩阵、决策树场景（约 40 个用例）
- 新增 `tests/regression/adt-enum/`：变体构造、解构、穷举、泛型、嵌套（约 20 个用例）
- 修订所有 `demos/` 中含 throw 的示例符合新语义
- 修订之前误改的 `panic(...) → throw new Exception(...)` 历史 docs（README、design docs、tasks）
- 更新 `docs/language/try-optional.md` 与 `prelude.md`
- 更新 MCP 知识库自动生成脚本
- 跑 `scripts/run_regression_tests.sh` 全绿

## 工作量估计

| Phase | 估计 | 关键风险 |
|-------|-----:|---------|
| 1 stdlib 类型 | 1 天 | 依赖 Phase 2 的 ADT；可分两步先做 Exception |
| 2 ADT enum | 5–8 天 | analyzer 穷举检查 + IR/VM 表示设计 |
| 3 throw/try!/catch! | 3–4 天 | dispatch 规则的 corner case |
| 4 测试 + 文档 | 2–3 天 | 大量 demo / README 改动 |

总计：约 **2 周**全栈工作量。

## 落地顺序建议

1. **先做 Phase 1（部分）**：让 `Exception` 可 `new`、可继承——独立改动，立即可用
2. **再做 Phase 2**：ADT enum 是最大块，独立可验证
3. **Phase 1 收尾**：在 Phase 2 完成后注册 `Result<T, E>` 到 prelude
4. **Phase 3**：在 Phase 1 + 2 之上加桥接糖
5. **Phase 4**：测试 + 文档清理

每个 phase 内部都要 build + ctest 全绿才进下一个。

## 与 spec 的对照

| spec 章节 | 本任务对应 phase |
|-----------|------|
| §1.5.6 prelude | Phase 1 |
| §2.2 类型分类 | Phase 1 |
| §3.7 try!/try?/catch! | Phase 3 |
| §5.6 ADT enum | Phase 2 |
| §6.3 enum 模式 + 穷举 | Phase 2 |
| §8.1 异常机制 | Phase 1 + 3 |
| §8.2 Result | Phase 1 + 2 |
| §8.3 桥接 | Phase 3 |
| §8.5 决策树 | 文档 only |
| §8.6 常用模式 | 文档 only |
| §15 / §16.6 / §16.7 | Phase 1 |
| §18.7 错误码 | Phase 3 |
| 附录 A EBNF | Phase 2 / 3 |

## 不在本任务范围

- **`throws` 函数签名**：D7 已决定不引入
- **enum payload 的 niche optimization**：起步用 boxing，性能优化留 JIT/AOT 阶段
- **Effect system**（algebraic effects）：与现有双轨制竞争，不在路线图

## 已知难点 / 待研究

- ADT enum 与 union types 的边界：`int | string` vs `enum E { I(int), S(string) }`，何时用谁？文档需要给经验法则。
- `match` 嵌套深度限制？目前 spec 没限，建议先用递归 lower，深度过深时报错。
- `Result<T, E>` 中 `E` 是泛型 union（如 `Exception | string`）的语义？建议禁止——E 必须是单一类型（class、enum、基本类型）。

## 附：典型场景验证清单

实现完成后必须能跑通以下代码：

```xray
// 1. Exception 可 new + 继承
class HttpError : Exception {
    code: int
    constructor(code: int, message: string) {
        super(message)
        this.code = code
    }
}
throw new HttpError(404, "not found")

// 2. ADT enum 构造与 match
enum NetEvent {
    Connected,
    Disconnected(reason: string),
    DataReceived(bytes: Bytes),
    Error(code: int, message: string),
}

let e = NetEvent.Disconnected(reason: "peer reset")
match (e) {
    NetEvent.Connected            -> print("ok"),
    NetEvent.Disconnected(r)      -> print("by:", r),
    NetEvent.DataReceived(b)      -> process(b),
    NetEvent.Error(code, msg)     -> log.error(code, msg),
}

// 3. Result + try!
fn parsePair(s: string) -> Result<(int, int), ParseError> {
    let parts = s.split(",")
    if (parts.length != 2) return Result.Err(ParseError.Empty)
    let a = try! parseInt(parts[0])
    let b = try! parseInt(parts[1])
    return Result.Ok((a, b))
}

// 4. 跨轨升级
fn dangerous(s: string) -> int {
    let n = try! parseInt(s)            // Err 抛异常
    return n * 2
}

// 5. catch! 块
let r: Result<int, Exception> = catch! {
    let cfg = loadConfig(text).unwrap()
    return startServer(cfg)
}

// 6. mapErr 错误转换
fn loadConfig(text: string) -> Result<Config, ConfigError> {
    let json = try! parseJson(text).mapErr(e -> ConfigError.BadJson(e))
    let port = try! json["port"].toInt().mapErr(e -> ConfigError.BadField("port", e))
    return Result.Ok(Config(port: port))
}

// 7. 编译期拒绝
throw "oops"                            // ❌ E0820
throw 42                                // ❌ E0820
let x = try! 42                         // ❌ E0821
```
