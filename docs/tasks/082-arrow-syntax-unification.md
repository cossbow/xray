# 082 — 箭头符号统一改造（=> 全面替换为 ->）

> 决策时间：2026-05-18
> 状态：已拍板，待实施
> 依赖：无（独立改造，必须在 081 错误处理之前完成）
> 预估工期：5-7 天

---

## 目标

将 xray 函数相关语法从混合 `:` / `=>` 风格改造为完全统一的 `->` 风格，使函数返回类型、函数类型、闭包、`match`、`select` 全部使用单一箭头符号，并把 `=>` 从语言中彻底移除。同时调整 Map 字面量改为 `#{ k: v }` 前缀消歧义形式，让 `:` 完全用于 KV 标注（Json + Map），`->` 完全用于函数/分支箭头。

## 决策记录

经过完整业界调研、4 种方案视觉对比，最终拍板：

- **函数返回类型**：`fn f(): T { ... }` → `fn f() -> T { ... }`
- **函数类型注解**：`fn(T): U` → `(T) -> U`
- **箭头闭包**：`(x) => x + 1` → `(x) -> x + 1`
- **匿名 fn 主推**：`fn(x: int): int { ... }` → `fn(x: int) -> int { ... }`
- **箭头闭包不允许显式标返回类型**：`(x: int): int => { ... }` 形式废弃；要标返回类型 → 改用 `fn` 形式或在 `let: T = ...` 上标
- **`match` 分支**：`pat => body` → `pat -> body`
- **`select` 分支**：`val from ch => body` → `val from ch -> body`（含 `to`、`after`、`_`）
- **Map 字面量**：`{ k => v }` 与 `#{ k => v }` → 统一为 `#{ k: v }`；无前缀 `{ ... }` 永远是 Json
- **`=>` 从 xray 完全消失**

最终符号语义：

| 符号 | 用途 |
|------|------|
| `:`  | 类型/键值标注（let、参数、Json `{k:v}`、Map `#{k:v}`、命名参数） |
| `->` | 函数/分支箭头（fn 返回、函数类型、闭包、match、select） |
| `=>` | **不再使用** |

## 业界对照

- **全 `->` 路线**：Haskell、OCaml、F#、Kotlin（match 用 `->`）、Java 17 switch expr
- **全 `=>` 路线**：TypeScript、Scala
- **双轨路线**：Rust（被吐槽不一致）、Swift（独特 `in`）

xray 选择 Kotlin / Haskell 路线，函数式表达力最强，符号语义最清晰。

## 实施 Checklist

### Phase 1：lexer 改造

- [ ] `src/frontend/lexer/xlex.c`：
  - `case '-':` 增加 `if (match(scanner, '>')) return TK_ARROW;` 分支
  - `case '=':` 删除 `else if (match(scanner, '>')) return TK_ARROW;` 分支
  - `[TK_ARROW]` 字符串字面量从 `"=>"` 改为 `"->"`
- [ ] `src/frontend/lexer/xlex.h`：
  - `TK_ARROW` 注释从 `=> (arrow function)` 改为 `-> (arrow / function type)`
- [ ] 验证：`build && ctest` 中 lexer 单元测试预期会大批失败（`tests/lex/*.in` 和 `*.expected`）

### Phase 2：parser 改造

- [ ] `src/frontend/parser/xparse_expr.c`：
  - 删除 `:` 返回类型识别 + `->` 误用提示分支（行 643-656）
  - 改为：`)` 后期望 `TK_ARROW`，再读类型
  - 删除箭头闭包显式返回类型分支（如果存在）：`(n: int): int => { ... }` 解析路径
- [ ] `src/frontend/parser/xparse_type.c`：
  - 函数类型从 `fn(T1, T2): R` 改为 `(T1, T2) -> R`
  - `fn` 关键字在类型上下文废弃（仍可用于声明 `fn name`）
- [ ] `src/frontend/parser/xparse_match.c`：
  - match 分支符号确认仍是 `TK_ARROW`（无需改动，符号已变）
- [ ] `src/frontend/parser/xparse_coroutine.c`：
  - select 分支符号确认仍是 `TK_ARROW`（无需改动，符号已变）
- [ ] `src/frontend/parser/xparse_decl.c`：
  - `xr_parse_object_or_map_literal`（行 700-730）：删除 `=>` 分隔符识别，改为 `{...}` 永远是 Json，遇到 `=>` 报错
  - `xr_parse_empty_map_literal`（行 760-810）：把 `xr_parser_consume(parser, TK_ARROW, ...)` 改为 `xr_parser_consume(parser, TK_COLON, ...)`，错误信息更新

### Phase 3：所有 .xr 测试和 stdlib 迁移

需要执行的批量替换（**按语境严格**）：

| 替换规则 | 范围 | 工具 |
|---------|------|------|
| `): T {` → `) -> T {` (函数返回类型) | 所有 .xr 文件 | regex |
| `: fn(...): T` → `: (...) -> T` (函数类型) | 所有 .xr 文件 | regex |
| `=>` 在闭包/match/select → `->` | 所有 .xr 文件 | regex |
| `(p: T): U =>` 闭包 → 改写为 `fn(p: T) -> U` | 个别测试 | 手动 |
| `{ k => v }` Map → `#{ k: v }` | 个别测试 | 手动 |

**目录范围**：
- [ ] `stdlib/types/*.xr`（核心类型定义）
- [ ] `stdlib/native/*.xr`（原生模块声明）
- [ ] `tests/regression/**/*.xr`（260+ 文件）
- [ ] `tests/aot/**/*.xr`
- [ ] `tests/jit/**/*.xr`
- [ ] `tests/compile_errors/**/*.xr`
- [ ] `examples/**/*.xr`
- [ ] `demos/**/*.xr`

**编写迁移脚本** `scripts/migrate_arrow_syntax.py`（或类似），分阶段执行：
1. 干 run，统计每个规则会改多少行
2. 按规则逐个 apply，每次跑一遍 ctest 验收
3. 个别复杂场景（闭包带返回类型 / Map 字面量）由人工 review

### Phase 4：formatter 与 LSP

- [ ] `src/frontend/format/*.c`：所有输出 `=>` 的分支改为 `->`
- [ ] `src/frontend/format/*.c`：所有输出 `: T {` 函数返回的分支改为 `-> T {`
- [ ] `src/app/lsp/*.c`：补全 / hover / signature help 中的箭头符号同步更新

### Phase 5：文档与 spec

- [ ] `docs/rules/language-spec.md`：
  - 全文 `=>` → `->`（按语境）
  - §2.10 函数类型注解：`fn(T): R` → `(T) -> R`
  - §3.7 表达式：箭头闭包语法
  - §4 语句：函数声明返回类型
  - §5.2 fn 声明 / §5.6 enum 内方法签名
  - §6 模式匹配
  - §11 协程 select 分支
  - 附录 A EBNF：所有相关 production 同步
  - 附录 G 增加 v0.8.0 之前再加 v0.7.x 条目记录"箭头符号统一"
- [ ] `docs/rules/architecture.md`：示例代码同步
- [ ] `docs/tasks/081-error-handling-redesign.md`：所有签名示例同步
- [ ] `docs/tasks/080-try-optional-expression.md`：检查并同步
- [ ] `README.md`、`CHANGELOG.md`：发版说明
- [ ] xray-website：所有教程 / 示例代码同步（独立 repo，跟随发版）

### Phase 6：错误诊断完善

- [ ] parser 在遇到 `=>` 时给出明确迁移提示：
  - 闭包 `(x) =>` → `提示：使用 (x) -> ...`
  - match `pat =>` → `提示：使用 pat -> ...`
  - Map `{ k =>` → `提示：使用 #{ k: ... }`
- [ ] parser 在遇到老式 `): T` 函数返回 → 提示用 `-> T`
- [ ] 这些诊断只在过渡期（一两个版本）保留，之后删除

### Phase 7：验证

- [ ] `cmake --build build -j 8` 全绿
- [ ] `ctest --output-on-failure` 全绿
- [ ] `scripts/run_regression_tests.sh` 全绿
- [ ] `scripts/run_compile_error_tests.sh` 全绿
- [ ] LSP smoke test 全绿
- [ ] formatter round-trip 测试（格式化后再解析等价）

## 已识别的"硬"场景（人工 review 必经之路）

### 场景 A：闭包显式标返回类型（`tests/regression/05_functions/0530_arrow.xr:29`）

```xray
// 旧
let factorial = (n: int): int => {
    if (n <= 1) { return 1 }
    return n * factorial(n - 1)
}

// 新（迁移到匿名 fn）
let factorial = fn(n: int) -> int {
    if (n <= 1) { return 1 }
    return n * factorial(n - 1)
}
```

### 场景 B：函数类型别名（`tests/regression/12_type_checking/1231_type_alias.xr`）

```xray
// 旧
type Mapper = fn(int): int
type Predicate = fn(int): bool
type Action = fn(int)

// 新
type Mapper = (int) -> int
type Predicate = (int) -> bool
type Action = (int) -> ()    // 或用 unit 类型
```

⚠️ 注意：无返回值的函数类型 `fn(int)` 在新语法下需要明确返回类型，可能用 `(int) -> ()` 或 `(int) -> void` —— 需要决策。建议用 `()`（unit）保持一致性。

### 场景 C：Map 字面量（`tests/regression/06_collections/*.xr` 部分）

需要扫描所有 Map 用法，确认：
- `{ "k" => v }` → `#{ "k": v }`
- `#{ "k" => v }` → `#{ "k": v }`

### 场景 D：select 分支带块体

```xray
// 旧
select {
    msg from ch1 => { print(msg) }
    after 1000 => { print("timeout") }
    _ => { print("default") }
}

// 新
select {
    msg from ch1 -> { print(msg) }
    after 1000 -> { print("timeout") }
    _ -> { print("default") }
}
```

### 场景 E：匿名 fn 在变量绑定（`1234_function_binding_assignment.xr`）

```xray
// 旧
let f = fn(x: int): int { return x + 1 }
f = fn(y: int): int { return y * 2 }

// 新
let f = fn(x: int) -> int { return x + 1 }
f = fn(y: int) -> int { return y * 2 }
```

## 不在本次 task 范围

- 错误处理重构（task 081 单独执行，本 task 完成后启动）
- ADT enum 实现（task 081 一部分）
- `try!` / `try?` / `catch!` 实现（task 081 一部分）
- 任何运行时 / VM / IR 改动（本 task 纯前端语法层）

## 风险与回滚

- 风险：批量正则替换可能误伤注释、字符串字面量内的 `=>`
  - 缓解：替换脚本必须 token-aware，跳过字符串和注释
- 风险：lexer 改了 `->` 后，老式 `): T` 仍能解析但 type 后必须接 `{`，可能与表达式上下文冲突
  - 缓解：parser 的迁移期诊断要做透，不靠静默接受
- 回滚：通过 git revert 回到改造前的 commit。所有迁移文件应在单一 commit 中聚合。

## 实施顺序建议

1. 起手：lexer 改造（小改动，先让 `->` 成为合法 token）
2. parser 改造（中等改动）
3. 写迁移脚本，干 run 看冲击面
4. 批量迁移 + 跑测试
5. 修补人工场景（A-E）
6. formatter / LSP / spec / 文档同步
7. 全量验收
8. 单 commit 提交，message：

```
Unify arrow syntax: replace => with -> across all contexts

- Function return type: fn f(): T -> fn f() -> T
- Function type: fn(T): U -> (T) -> U
- Closure body: (x) => expr -> (x) -> expr
- match/select branch: pat => body -> pat -> body
- Map literal: { k => v } -> #{ k: v }
- Forbid explicit return type on arrow closures (use fn form instead)
- => token removed from the language

Single source of symbol semantics:
  : for type / key-value annotation
  -> for function / branch arrow
```

## 后续衔接

本 task 完成后，task 081 错误处理重构启动，所有错误处理新语法（`Result<T, E>`、ADT enum 内方法、`try!` / `try?` / `catch!`）天然按本 task 的统一箭头规则书写，无需再做语法适配。
