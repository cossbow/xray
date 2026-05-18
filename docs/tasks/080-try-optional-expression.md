# 080 — `try?` 表达式（异常折叠为 null）

| 字段 | 值 |
|------|-----|
| 状态 | planned |
| 提案人 | 用户 |
| 设计来源 | 早期 Swift 风格语法分析（场景 3 / 场景 5 / 场景 6 / 场景 7） |
| 影响范围 | frontend/parser、frontend/analyzer、ir/xi_lower_expr、新增回归测试 |
| 后端是否改动 | **无**（复用现有 XI_TRY/XI_CATCH/XI_END_TRY） |
| 工作量估算 | 200~300 行核心代码 + 4~8 个回归测试 |

---

## 1. 目标

引入一个表达式级语法 `try? expr`，将异常折叠为 `null`，与 xray 现有的 `?.`、`??`、`!`、optional 类型构成完整的容错语义。

```xray
// 当前写法：13 行
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

// `try?` 之后：1 行
let dbHost = (try? Json.parse(readFile("config.json")))?.database?.host ?? "localhost"
```

---

## 2. 与现有语法的关系（**重要：不要重复造轮子**）

xray 已经实现的"配料"清单：

| 语法 | AST 节点 | 解析位置 |
|------|----------|----------|
| `expr ?? default` | `AST_NULLISH_COALESCE` | `src/frontend/parser/xparse_expr.c:881` |
| `obj?.prop / obj?.[i] / obj?.method()` | `AST_OPTIONAL_CHAIN` | `src/frontend/parser/xparse_expr.c:917` |
| `expr!` | `AST_FORCE_UNWRAP` | `src/frontend/parser/xparse_expr.c:891` |
| `expr as T?` (safe cast) | `AST_AS_EXPR` (is_safe=true) | `src/frontend/parser/xparse_expr.c:898` |
| `try { ... } catch (e) { ... } finally { ... }` 语句 | `AST_TRY_CATCH` | `src/frontend/parser/xparse_decl.c:1426` |
| Xi IR 异常机制 | `XI_TRY / XI_CATCH / XI_END_TRY / XI_THROW` | `src/ir/xi_lower_stmt.c:881-1085` |

**唯一缺失**：`try? expr`（表达式级，把异常折叠成 null）。

---

## 3. 语义定义

### 3.1 基本语义

```
try? expr   ≡   try { expr } catch (_) { null }
```

如果 `expr` 求值过程中抛出任何异常，`try? expr` 的求值结果为 `null`；否则为 `expr` 的值。

### 3.2 类型规则

| 操作数类型 | 结果类型 |
|-----------|---------|
| `expr: T`（非 nullable，T ≠ null） | `T?` |
| `expr: T?`（已 nullable） | `T?`（null 域被合并） |
| `expr: null`（仅 null 字面量） | `null`（无意义但合法） |

形式化：`type(try? e) = make_nullable(type(e))`。

### 3.3 与其他后缀算子的组合

下面的优先级保证表达式直觉与 PPT 示例一致：

```xray
try? a.b.c         // == try? (a.b.c)            高于成员访问
try? f(x)          // == try? (f(x))             高于调用
try? a?.b ?? c     // == (try? (a?.b)) ?? c      高于 ??
try? -x            // == try? (-x)               与 unary 同级
!(try? x)          // 必须显式括号
```

### 3.4 与函数体异常的关系

xray **没有 checked exception**，所有函数都隐式可能抛。`try?` 在 xray 里 **纯粹是糖**，不引入"必须标注"的强制语义。这与 Swift 不同，文档需明确说清。

### 3.5 求值副作用

`try? e` 不抑制 `e` 内部的副作用（已发生的写入、IO、go、channel send 都不会回滚），仅把"异常控制流"折叠为 null。这与 `try-catch` 一致。

---

## 4. 解析（Parser）

### 4.1 词法层面

`try?` 由两个独立 token 组成：`TK_TRY` + `TK_QUESTION`（中间允许空白）。**不需要新增 token**。

### 4.2 语法位置

xray 当前 `TK_TRY` 仅在语句起点出现（`xr_parse_statement` 显式分支）。需要让 `TK_TRY` 在表达式上下文也合法。

### 4.3 优先级

`try?` 作为 **prefix unary**，优先级 = `PREC_UNARY`（与 `!` `-` `~` 同级）。
右操作数用 `xr_parse_precedence(parser, PREC_UNARY)` 解析，自然吃掉所有更高优先级的 `.`、`?.`、`()`、`[]`。

### 4.4 在 parser 表中的注册

`src/frontend/parser/xparse.c` 新增：

```c
[TK_TRY] = { xr_parse_try_expr_prefix, NULL, PREC_NONE },
```

注意：infix 留 NULL，因为 `try` 不能出现在中缀位置。

### 4.5 prefix handler 设计

```c
// xparse_expr.c
AstNode *xr_parse_try_expr_prefix(Parser *parser) {
    int line = parser->current.line;
    xr_parser_advance(parser); // consume 'try'

    // 必须紧跟 ? 才是表达式 try?
    if (parser->current.type != TK_QUESTION) {
        xr_parser_error_at_current(parser,
            "'try' in expression position must be followed by '?'; "
            "use 'try { ... } catch ...' as a statement");
        return NULL;
    }
    xr_parser_advance(parser); // consume '?'

    AstNode *operand = xr_parse_precedence(parser, PREC_UNARY);
    if (!operand) return NULL;

    return xr_ast_try_expr(parser->X, operand, AST_TRY_OPTIONAL, line);
}
```

### 4.6 与语句 `try` 的歧义消除

statement-level 的 `xr_parse_statement` **优先**捕获 `TK_TRY`（已经在 `xparse.c:774`），所以：
- 块开头 `try { ... }` → 走语句路径，不会进 prefix handler
- 表达式上下文（`let x = try?`、`return try?`、参数中、`?? try?` 右操作数）→ 走 prefix handler

**没有解析歧义**。

### 4.7 错误处理

| 错误情形 | 错误信息 |
|---------|---------|
| `let x = try { ... }` | "block-form 'try' is a statement; use 'try? expr' for expression form" |
| `let x = try expr`（漏 `?`） | "'try' in expression position must be followed by '?'" |
| `let x = try?`（漏操作数） | "expected expression after 'try?'" |

---

## 5. AST 变更

新增一个节点（与未来可能的 `try!`、bare `try` 共用）：

```c
// xast_nodes_expr.h（或合适位置）
typedef enum {
    AST_TRY_OPTIONAL = 0,  // try? expr
    // 未来扩展：
    // AST_TRY_FORCE = 1,  // try! expr
    // AST_TRY_BARE  = 2,  // try expr （仅标记，不改语义）
} TryExprMode;

typedef struct {
    AstNode *operand;
    TryExprMode mode;
} TryExprNode;

// 在 AstNode 联合中新增：
case AST_TRY_EXPR:
    TryExprNode try_expr;
```

`xr_ast_try_expr()` 工厂方法放到 `xast.c`，对齐现有 `xr_ast_optional_chain` 等的风格。

---

## 6. Analyzer

新增 `xa_visit_try_expr`：

```c
XrType *xa_visit_try_expr(XaInferContext *ctx, AstNode *node) {
    if (!ctx || !node) return xr_type_new_unknown(NULL);
    XrType *inner = xa_visit_infer_expr(ctx, node->as.try_expr.operand);
    if (!inner) return xr_type_new_unknown(NULL);
    return xr_type_make_nullable(ctx->analyzer->isolate, inner);
}
```

挂到 `xa_visit_infer_expr` 的 dispatch 即可。

---

## 7. Xi IR Lowering

### 7.1 模板

参考 `lower_nullish_coalesce`（`src/ir/xi_lower_expr.c:1460-1505`）的"双块 + phi"骨架，结合 `lower_try_catch_impl`（`src/ir/xi_lower_stmt.c:986-1074`）的 try/catch 控制流。

### 7.2 实现草图

```c
// xi_lower_expr.c
static XiValue *lower_try_expr(XiLower *l, AstNode *node) {
    /* CFG:
     *   cur ──XI_TRY──► try_blk ──(success)──► merge
     *                     │
     *                     └──(throw)──► catch_blk ──► merge
     */
    XiBlock *try_blk = xi_block_new(l->func);
    XiBlock *catch_blk = xi_block_new(l->func);
    XiBlock *merge = xi_block_new(l->func);

    XiValue *try_op = xi_value_new(l->func, l->cur_block, XI_TRY, l->type_unit, 0);
    if (try_op) {
        try_op->aux = (void *) catch_blk;
        try_op->aux_int = -1;            // no separate finally
        try_op->flags |= XI_FLAG_SIDE_EFFECT;
        try_op->line = (uint32_t) node->line;
    }

    xi_block_set_jump(l->cur_block, try_blk);
    xi_lower_braun_seal(l, try_blk);

    /* try block: evaluate operand */
    l->cur_block = try_blk;
    l->dead_after_throw = false;
    l->try_depth++;
    XiValue *value = xi_lower_expr(l, node->as.try_expr.operand);
    l->try_depth--;
    XiBlock *try_exit = l->cur_block;
    if (try_exit)
        xi_block_set_jump(try_exit, merge);

    /* catch block: bind exception (discard), produce null */
    XiBlock *catch_pred = try_exit ? try_exit : try_blk;
    xi_block_add_pred(catch_blk, catch_pred);
    xi_lower_braun_seal(l, catch_blk);
    l->cur_block = catch_blk;
    l->dead_after_throw = false;

    XiValue *catch_op = xi_value_new(l->func, l->cur_block, XI_CATCH, l->type_any, 0);
    if (catch_op) {
        catch_op->flags |= XI_FLAG_SIDE_EFFECT;
        catch_op->line = (uint32_t) node->line;
    }

    XiValue *null_val = xi_const_null(l->func, l->cur_block, l->type_any);
    xi_block_set_jump(l->cur_block, merge);

    /* merge: phi(value, null) */
    xi_lower_braun_seal(l, merge);
    l->cur_block = merge;

    XrType *result_type = xi_lower_node_type(l, node);
    XiPhi *phi = xi_phi_new(l->func, merge, result_type, merge->npreds);
    if (phi) {
        for (uint16_t i = 0; i < merge->npreds; i++) {
            if (merge->preds[i] == try_exit)
                phi->value.args[i] = value ? value : null_val;
            else
                phi->value.args[i] = null_val;
        }
    }

    /* end_try marker */
    XiValue *end_op = xi_value_new(l->func, l->cur_block, XI_END_TRY, l->type_unit, 0);
    if (end_op) {
        end_op->flags |= XI_FLAG_SIDE_EFFECT;
        end_op->line = (uint32_t) node->line;
    }

    return phi ? &phi->value : null_val;
}
```

### 7.3 dispatch 接入

`xi_lower_expr` 主分发表（`src/ir/xi_lower_expr.c:2853` 附近）增加：

```c
case AST_TRY_EXPR:
    return lower_try_expr(l, node);
```

### 7.4 SSA / Braun 细节

- `try_depth++/--` 包住操作数 lowering：保证操作数内部 `throw` 能正确保留 dead block 用于 phi 输入（这是 `lower_try_catch_impl` 已经验证过的机制）
- `xi_block_add_pred` + `xi_lower_braun_seal` 顺序与 try-catch 语句一致
- `dead_after_throw` 标志在每个新块开始时清零

### 7.5 优化器交互

- **DCE**：`XI_TRY/XI_CATCH/XI_END_TRY` 已带 `XI_FLAG_SIDE_EFFECT`，不会被消除
- **GCM/LICM**：side-effect 边界保持不变
- **constant folding**：如果操作数是纯字面量（无副作用、不可能抛），可以折叠为 `expr` 本身。**首版不做此优化**，保持语义清晰
- **inlining**：与现有 try-catch 一致，不破坏

---

## 8. 后端

**零改动**。VM、JIT、AOT 的 OP_TRY/OP_CATCH/OP_END_TRY 路径已经稳定（见 `xworker_sysmon.c` 的主线程错误传播、xi_emit OP_TRY patching、xcgen exception 路径）。

---

## 9. Formatter / 工具链

| 模块 | 改动 |
|------|------|
| `src/app/cli/xfmt_expr.c` | 输出 `try? <operand>`，operand 周围视情况加括号（与 unary 同规则） |
| `xray-vscode` syntax | TextMate grammar 增加 `\btry\?` 高亮 |
| LSP hover | 对 `AST_TRY_EXPR` 显示 `T → T?` 并附说明 |
| `docs/spec/language-spec.md` | 新增章节"Optional Try Expression" |

---

## 10. 测试计划

### 10.1 单元测试

`tests/unit/ir/test_xi_lower.c`：
- `lower_try_expr_basic`：`try? f()` 产生 XI_TRY/XI_CATCH/XI_END_TRY + phi
- `lower_try_expr_chains_with_optional_chain`：`try? a()?.b ?? c` 整条链路 SSA 合法
- `lower_try_expr_nested`：`try? (try? f())` 不破坏外层 try_depth

### 10.2 回归测试（对应 PPT 4 个场景）

放到 `tests/regression/0950_exception/0958_try_optional_*.xr`：

| 文件 | 场景 |
|------|------|
| `0958_try_optional_basic.xr` | 场景 7：函数返回 nullable，`try? db.query(...)` 直接折叠 |
| `0959_try_optional_default.xr` | 场景 3：`try? expr ?? default` 多变量批量 |
| `0960_try_optional_chain.xr` | 场景 5：`(try? Json.parse(...))?.field ?? "x"` |
| `0961_try_optional_object.xr` | 场景 6：`try? Json.parse(...) ?? { default: ... }` |

每个文件包含 `@test` 函数，覆盖：
- 正常路径返回值正确
- 抛异常路径返回 null
- 副作用（IO、`print`、channel send）在异常前已发生时不被回滚
- 类型推导：`typeof(try? f())` 应为 nullable

### 10.3 类型测试

`tests/compile_errors/type/`：
- 把 `T?` 当 `T` 用应报错（除非走 `?.`、`??`、`!`）
- `try? expr` 后赋给非 nullable 字段应报错

### 10.4 兼容性回归

- `XRAY_SKIP_BUILD=1 scripts/run_regression_tests.sh`：现有 `try-catch` 测试全部仍通过
- ASAN 全量：异常路径无泄漏
- JIT diff：VM/JIT 输出一致

---

## 11. 风险与缓解

| 风险 | 严重度 | 缓解 |
|------|-------|------|
| 静默吞异常导致调试困难 | 高 | 加环境变量 `XRAY_TRY_TRACE=1`，被吞异常打印到 stderr 含栈 |
| 用户写 `try?` 而非 `try-catch` 处理本应 recover 的异常 | 中 | 文档明示设计意图；lint 检查 `let _ = try? ...`（结果被丢弃）→ warning |
| 类型膨胀（链式 `T?`） | 中 | `??` 已能收敛；演示文档说明配套使用 |
| 错误定位变难（链式吞异常） | 中 | 与第一项同方案 |
| 解析新增前缀 token 可能与未来语法冲突 | 低 | `try` 已是关键字，无冲突 |

---

## 12. 实施顺序（推荐拆 4 个 commit）

1. **commit 1**: AST 节点 + factory + parser 前缀处理 + 单元测试（编译期）
2. **commit 2**: Analyzer 类型推导 + xi_lower_expr lowering + xi_compare 测试
3. **commit 3**: 4 个回归测试 + 全量 ctest/regression 验证（VM/JIT/AOT 三路 diff）
4. **commit 4**: formatter + spec 文档 + xray-vscode grammar 同步发布

每个 commit 独立可回滚，先合并 1+2 即可让 `try?` 在主仓可用。

---

## 13. 验收标准

- 所有 4 个 PPT 场景的"✅ 后"代码可在 xray 中编译通过并运行
- `tests/regression` 全部通过（296+ 用例）
- `xi_compare` 全部通过
- ASAN clean
- VM/JIT/AOT 输出 diff = 0
- `docs/spec/language-spec.md` 更新章节通过 review

---

## 14. 不在本任务范围

- `try!`（异常时 panic）— 留给后续任务
- bare `try` 表达式（仅标记，不改行为，Swift 风格）— 留给后续任务
- `Result<T, E>` / `Either` 类型 — xray 走 nullable 路线，不引入双轨容错
- 异常类型缩窄 / typed catch — 与本任务正交

---

## 15. 参考实现位置速查

| 角色 | 文件 | 关键符号 |
|------|------|---------|
| 词法 token | `src/frontend/lexer/xtoken.h` | `TK_TRY`、`TK_QUESTION`（已有） |
| 解析表 | `src/frontend/parser/xparse.c` | parsing rules table |
| 解析 helper | `src/frontend/parser/xparse_expr.c` | 新增 `xr_parse_try_expr_prefix` |
| AST 工厂 | `src/frontend/parser/xast.c` | 新增 `xr_ast_try_expr` |
| AST 节点声明 | `src/frontend/parser/xast_nodes_expr.h` | 新增 `TryExprNode` / `AST_TRY_EXPR` |
| Analyzer | `src/frontend/analyzer/xanalyzer_visitor_expr.c` | 新增 `xa_visit_try_expr` |
| Lowerer | `src/ir/xi_lower_expr.c` | 新增 `lower_try_expr`，dispatch `case AST_TRY_EXPR` |
| 模板参考 | `src/ir/xi_lower_expr.c:1460` | `lower_nullish_coalesce`（双路 phi 模板） |
| 模板参考 | `src/ir/xi_lower_stmt.c:986` | `lower_try_catch_impl`（try-catch CFG 模板） |
| Formatter | `src/app/cli/xfmt_expr.c` | 增加 `AST_TRY_EXPR` 输出 |
| 语言规范 | `docs/spec/language-spec.md` | 增章节"Optional Try Expression" |
