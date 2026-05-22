---
id: spec.appendix_e_differences_from_other_languages
order: 024
---

<!-- xr-spec:cn -->
---

## 附录 E. 与其他语言的差异速查

xray 在开发过程中借鉴了现有语言的许多优秀设计，但还是有显著差异。

### E.1 vs JavaScript / TypeScript

| 维度 | JS/TS | xray |
|--|--|--|
| 静态类型 | TS 可选 | **强制**（除 `Json` 是动态） |
| 数值 | 仅 `number`（双精度） | `int` `float` `BigInt` 严格区分 |
| 真值转换 | truthy / falsy | truthy / falsy（与 JS 相近）但 `bool` 类型本身不接受 int/null 隐式赋值 |
| `===` 与 `==` | `===` 强、`==` 弱（string↔number 自动转） | `==`/`!=` 为值相等（仅数值 int↔float 提升）；`===`/`!==` 为类型和值都严格相等 |
| 闭包捕获 | 引用 | 引用（默认）；`go` 闭包严格受限 |
| 对象 | 动态字段 | 默认动态；带 `type T = {...}` 注解后 sealed |
| import | ES Module | 自有 import 语法（含 stdlib 无引号形式） |
| 并发 | 异步/Promise | 协程 + Channel |

### E.2 vs Go

| 维度 | Go | xray |
|--|--|--|
| 类型系统 | 简单 + interface 隐式 | 较丰富 + 显式 `implements` |
| 错误处理 | 多返回值 + `err != nil` | 异常 + `Result<T,E>` + `try?`/`try!` |
| 协程 | `go func() {}`（语句） | `go expr`（表达式，返回 `Task<T>`） |
| 等待结果 | 无直接等价（通过 channel/WaitGroup） | `await t`、`await all [...]`、`await any [...]` |
| Channel | 内置 `chan T`，`<-` 操作符 | `Channel<T>` 类，方法 `send`/`recv`/`trySend`/`tryRecv` |
| select 分支 | `case x := <-ch:` / `case ch <- v:` / `default:` | `x from ch ->` / `v to ch ->` / `after ms ->` / `_ ->` |
| GC | 三色并发 | per-coroutine Mark-Sweep / Immix |
| 类与继承 | 无（仅 struct + interface） | class 支持继承 |
| 泛型 | 1.18+ 有 | 有，monomorphization + 运行时 reified |

### E.3 vs Rust

| 维度 | Rust | xray |
|--|--|--|
| 内存安全 | borrow checker 全面 | 仅跨协程用 `move`；其他用 GC |
| 错误 | `Result<T, E>` | 异常 + `Result<T,E>` |
| 类型推断 | Hindley-Milner 强 | 双向推断 |
| trait | 完整 | 类似 `interface`，少功能 |
| 性能 | 接近 C | VM/JIT，热路径接近 native |
| 编译期 | macro / const | 简单常量折叠 |

### E.4 vs Python

| 维度 | Python | xray |
|--|--|--|
| 类型 | 动态（可选 hint） | 静态 |
| GIL | 有 | 无（M:N 协程） |
| 字符串 | unicode str | utf-8 string |
| 缩进 | 强制 | 自由（用 `{}`） |
| 类 | 动态属性 | 静态字段 |
| 性能 | CPython 慢 | JIT 后接近 V8/JVM |

### E.5 vs Swift

| 维度 | Swift | xray |
|--|--|--|
| 可空 `?` | 有 | 有 |
| `!` 解包 | 有 | 有 |
| `try?` / `try!` 语义 | `try?` 折叠为 nil；`try!` 失败 abort | `try?` 折叠为 null；`try!` **重抛原异常**（不 abort） |
| struct vs class | 值/引用 | 值/引用 |
| 协议 | 有强 | `interface` 较弱 |
| 并发 | actor + async/await | 协程 + Channel + `go`/`await all`/`scope` |
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## Appendix E. Differences from Other Languages

### JavaScript / TypeScript

- Xray is statically typed; there is no source-level `any`.
- `Map` literals use `#{...}`, not object syntax.
- Imports are Xray-specific and do not support JS default import syntax.
- Concurrency is coroutine/channel based, not Promise-based.

### Go

- Xray has classes, structs, interfaces, enums, exceptions, and nullable types.
- `go` is an expression returning `Task<T>`.
- Channels are `Channel<T>` objects with methods.
- `select` uses `x from ch ->`, `v to ch ->`, `after ms ->`, and `_ ->` arms.

### Rust

- Xray uses GC and explicit cross-coroutine sharing instead of a full borrow checker.
- ADT enums and Result-style values are supported, but exceptions are also part of the language.

### Swift

- Xray has `T?`, `try?`, and `try!`, but `try!` rethrows rather than aborting.
- Concurrency is coroutine/channel based.
<!-- /xr-spec:en -->
