# Frontend 模块最终重构方案（Final）

> 日期：2026-04-25
> 状态：决策版
> 范围：`src/frontend/{lexer, parser, format, analyzer, codegen}` + `xdiag_fmt.h`
> 关联：合并自 `frontend_refactor_plan.md`（用户原版）+ `frontend_refactor_plan_cascade.md`（深审版）

---

## 0. 工程原则（强约束）

本文档执行 Xray 项目的核心工程纪律：

> **不考虑向后兼容性 — 直接采用最佳设计，避免技术债**

具体到本次重构：

- **不留过渡形态**：每个问题只给"终态"，不再有"先用 A 顶住、后改 B"
- **死代码立删**：跨层死 include、无调用面的"备用 API"、注释/命名/实现漂移的"假能力"
- **弱抽象立重写**：手写 trie、稀疏表、复制粘贴 helper — 一律换成单一数据源
- **失败语义立明确**：要么真做（incremental 真增量、formatter 真 round-trip），要么真删
- **单一真相源**：关键字表、TokenType 名字、AST 节点字段、依赖图边都只能有一份定义

非目标（明确不做）：
- 不为外部用户保留任何 API
- 不为"未来可能用到的"功能保留无调用面代码
- 不做"标注为 deprecated 但保留"的中间态

---

## 1. 现状基线

### 1.1 目录结构

```text
src/frontend/
├── lexer/    (2 文件: xlex.{c,h})
├── parser/   (18 文件: xast*, xparse*)
├── format/   (2 文件: xfmt.{c,h})
├── analyzer/ (30 文件: xanalyzer_*)
├── codegen/  (57 文件: xcompiler/xexpr/xstmt/xoop/...)
└── xdiag_fmt.h
```

### 1.2 关键文件规模（已逼近或越过硬线）

| 文件 | 行数 | 状态 |
|------|------|------|
| `parser/xast_nodes.h` | 807 | ❌ 已越 `.h ≤ 800` 硬线 |
| `analyzer/xanalyzer_visitor.c` | 2533 | ⚠️ 接近 `.c ≤ 3000` 硬线 |
| `analyzer/xanalyzer_visitor_expr.c` | 1942 | ⚠️ |
| `parser/xast.c` | 1897 | ⚠️ |
| `codegen/xexpr_call.c` | 1882 | ⚠️ |
| `format/xfmt.c` | 1737 | ⚠️ |
| `parser/xparse_oop.c` | 1737 | ⚠️ |
| `codegen/xcompiler.c` | 1728 | ⚠️ |

无单文件越 3000 行硬线，但第二梯队已多个 ≥ 1700 行。

### 1.3 目标依赖方向

```text
F0  base/xdiag_fmt           ←  唯一允许跨模块复用的诊断 helper
F1  lexer
F2  parser + ast              ←  含 parser-owned type scope（不借 analyzer 符号表）
F3  format                    ←  只依赖 ast，不碰 public API 头
F4  analyzer
F5  codegen
```

**绝对禁止**（需在 CI 中静态守护）：

- `parser/` 不得 include `analyzer/`
- `frontend/**` 不得 include `include/xray.h` / `include/xray_isolate.h` 等 L7 public API 头
- `format/` 不得 include `analyzer/`
- `lexer/` 不得 include `runtime/`、`vm/`、`jit/` 等任何 L1+ 模块

---

## 2. 问题清单与终态决策

合并 FE-01~FE-15（用户原版）+ CL-LEX-01~08（深审版）+ 新发现项。**每条问题给出最终形态决策，不留中间状态。**

### 2.1 词法层（lexer）

#### 🔴 L-01 `TK_IF` 缺长度守卫（真实 bug）

`@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/lexer/xlex.c:361` 直接 `return TK_IF`，未检 `==2`。
`iffy`/`ifElse` 会被误识别为 `if`。

**终态**：与 X-macro 关键字表合并实施（见 L-08），新表统一处理长度匹配，单点修复。

#### 🔴 L-02 多行字符串 token 位置不可靠（用户 FE-01）

`make_token()` 用扫描结束的 `(line, column)`，不是 token 起始；`string_with_quote()` 跨行时只更新 `line`，不更新 `line_start`。

**终态**：

- `Scanner` 增加 `start_line` / `start_line_start` 字段，由 `xr_scanner_scan()` 入口快照
- `make_token()` 强制使用快照值，删除"扫描结束坐标"路径
- `string_with_quote()` / `raw_string_with_quote()` 跨行时同步更新 `line_start`

#### 🟡 L-03 `Token.start` 在错误时复用为 message（用户 FE-12）

错误 token 的 `start` 指向 `const char*` 错误消息字符串，正常 token 指向源码 — 同一字段两种语义。

**终态**：`Token` 增加 `error_message` 字段；正常 token 该字段为 `NULL`。`Token.start/length` **永远**指向源码片段，不变义。

```c
typedef struct Token {
    TokenType type;
    const char *start;
    int length;
    int line;
    int column;
    bool has_leading_space;
    XrTrivia *leading_trivia;
    XrTrivia *trailing_trivia;
    const char *error_message;  // NEW: only set when type == TK_ERROR
} Token;
```

#### 🟡 L-04 `xr_trivia_new` 缺 NULL 检查

违反 `xr_malloc 后必须查 NULL` 红线。

**终态**：函数返回 NULL；`trivia_append()` 容忍 NULL 并跳过；OOM 不再传播解引用。

#### 🟡 L-05 跨层死 include `runtime/value/xtype_names.h`

`xlex.c:20` 引入但未使用任何符号。

**终态**：删除该 include。同步检查 `<stdlib.h>` / `<ctype.h>` 是否同样未使用 — 是则一并删除。

#### 🔴 L-06 trailing trivia 未闭环（用户 FE-04）

`Token.trailing_trivia` 字段存在但永远为 `NULL`；parser 只消费 leading；formatter 只输出 leading。

**终态**：**真做**（不删字段）：

- lexer：在 token 之后、下一次换行/EOF 之前出现的注释 → 当前 token 的 `trailing_trivia`
- parser：将 trailing trivia 透传到对应 AST 节点（在 `AstNode` 增加 `trailing_comments` 字段）
- formatter：节点输出时同步输出 leading（前置）+ trailing（行尾）注释

#### 🟡 L-07 `TRIVIA_BLOCK_COMMENT` 注释错字、`is_hex_digit` wrapper、`token_names[]` 稀疏表

杂项卫生问题。

**终态**：与 L-08 一并清理 — 注释改为 `/* */`；删除 `is_hex_digit`，调用处直接用 `XR_IS_HEX`；`token_names[]` 重构为两段表（ASCII 段 + multi-char 段）。

#### 🟡 L-08 关键字识别从手写 trie 改为 X-macro 单一数据源（用户 FE-13）

270 行手写 switch 嵌套，关键字 ~70 个。L-01 bug 正是这种结构产物。

**终态**：单一 `keyword_table.def` X-macro 文件，同时生成：

```c
// src/frontend/lexer/xkeywords.def
//          spelling   token         length
XR_KW_DEF(  "let",     TK_LET,       3 )
XR_KW_DEF(  "const",   TK_CONST,     5 )
XR_KW_DEF(  "if",      TK_IF,        2 )
XR_KW_DEF(  "fn",      TK_FN,        2 )
/* ... */
```

由 `def` 文件展开生成：

- `TokenType` 枚举值
- `token_names[]` 数组
- 排序后的二分查找表 / perfect hash（从 spelling 到 TokenType）

`identifier_type()` 只剩一次哈希查找，270 行函数消失。L-01 类型 bug 在结构上不再可能存在。

### 2.2 语法层（parser）

#### 🔴 P-01 parser 反向依赖 analyzer（用户 FE-06）

`xparse.c` 等多文件 include `../analyzer/xanalyzer_symbol.h`，借 `XaScope` 存 type alias / generic param。

**终态**：parser 自有 `XrTypeScope`（不复用 analyzer）

新增 `@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/parser/xtype_scope.{c,h}`：

```c
typedef struct XrTypeScope XrTypeScope;

XR_FUNC XrTypeScope *xr_type_scope_new(XrTypeScope *parent);
XR_FUNC void xr_type_scope_free(XrTypeScope *scope);
XR_FUNC bool xr_type_scope_define(XrTypeScope *scope, const char *name, XrType *type);
XR_FUNC XrType *xr_type_scope_lookup(XrTypeScope *scope, const char *name);
```

- 全部 `xparse_*.c` 中的 `XaScope` 用法替换为 `XrTypeScope`
- **删除** `parser/*.{c,h}` 中所有 `#include "../analyzer/..."`（在 CI 中加 grep 守护）

#### 🟡 P-02 `xparse.h` 暴露面过大（用户 FE-08）

公私 API 混杂，下游误用 parse helper。

**终态**：`xparse.h` 只保留 5 类公共 API：

1. `xr_parse_program(...)` 入口
2. `xr_parse_recoverable(...)` 容错入口
3. `xr_parser_init/free`
4. AST 节点构造器（如已暴露给外部）
5. `XrParserResult` 结构

其余全部移入 `xparse_internal.h`。

#### 🟡 P-03 `xast_nodes.h` 越 800 行硬线（用户 FE-08）

807 行，已实质越线。

**终态**：拆为 4 个主题头 + 1 个聚合：

```text
parser/
├── xast_nodes_common.h     // AstKind, AstNode 基类, 位置/comment 字段
├── xast_nodes_expr.h        // 表达式节点
├── xast_nodes_stmt.h        // 语句节点
├── xast_nodes_decl.h        // 声明节点 (fn/class/struct/enum/...)
└── xast_nodes.h             // 聚合 #include 上面 4 个
```

下游不变（继续 include `xast_nodes.h`）；单头 ≤ 350 行。

#### 🟡 P-04 删除"compatibility"/"reserved"模糊状态

`xr_parse_set_literal()` 注释为 compatibility 保留；`yield` 标为 reserved for future。

**终态**：

- compatibility 路径：**直接删除**（无外部用户）
- reserved 但未实现的关键字（`yield`）：要么实现，要么删除 token 类型；不留模糊态

#### 🟡 P-05 codegen 通过 parser 借类型打印 helper（用户 FE-11）

`xoop_class.c` include `xparse.h` 只为 `xr_compile_type_to_string()`。

**终态**：将 `xr_compile_type_to_string()` 下沉到 `runtime/value/xtype_format.{c,h}` 或 `base/xtype_print.{c,h}`，parser/codegen 都从该处取。`xparse.h` 移除该函数声明。

### 2.3 格式化层（format）

#### 🔴 F-01 formatter 输出反引号模板串，但 lexer 已 deprecate（用户 FE-02）

会生成当前 parser **拒绝**的源码。

**终态**：完全删除反引号输出路径。所有模板串统一输出为 `"..." with ${expr}` 形式。

#### 🔴 F-02 字符串字面量序列化不保语义（用户 FE-03）

parser 把字符串解码为语义值，formatter 直接原样写双引号。包含换行/反斜杠/引号/控制字符的字符串格式化后是非法源码或语义改变。

**终态**：新增 `format/xfmt_literal.{c,h}`，统一处理：

```c
XR_FUNC void xfmt_emit_string(XrtBuffer *out, const char *value, int len);
XR_FUNC void xfmt_emit_template_string(XrtBuffer *out, const XrAstNode *node);
XR_FUNC void xfmt_emit_raw_string(XrtBuffer *out, const char *value, int len);
```

要求：

- 普通字符串：重新转义所有需转义字符
- raw string：**统一规范化为普通双引号字符串**（不保留 raw 风格 — 项目原则下选最佳设计，canonical form 优先于 lexeme preservation）
- template string：转义引号/反斜杠 + 防止意外拼出 `${`

#### 🟡 F-03 `xfmt.c` 1737 行单文件巨型（用户 FE-15）

**终态**：随 F-01/F-02 同步拆分：

```text
format/
├── xfmt.{c,h}             // 入口、配置、主调度
├── xfmt_expr.c            // 表达式输出
├── xfmt_stmt.c            // 语句输出
├── xfmt_decl.c            // 声明输出
├── xfmt_literal.{c,h}     // 字面量与字符串序列化（NEW）
├── xfmt_type.c            // 类型注解输出
└── xfmt_trivia.c          // 注释附着 / leading / trailing
```

#### 🟡 F-04 `xfmt.h` 依赖 public API 头（用户 FE-07）

include `include/xray_isolate.h`。

**终态**：用 forward declaration 替代；`xfmt.h` 不 include 任何 `include/*.h`。

### 2.4 语义分析层（analyzer）

#### 🔴 A-01 dependency graph reverse-edge 内存泄漏（用户 FE-05）

`dep_graph_free()` 只释放 forward 链，reverse 链节点泄漏。

**终态**：`dep_graph_free()` 显式释放 reverse 链，注释同步修正。新增 ASan 单测验证。

#### 🔴 A-02 incremental 名称与实际能力漂移（用户 FE-09）

`xa_incremental_update()` 自带 `TODO: Implement true incremental parsing`；`xa_analyzer_update_incremental()` 实质是"full-file 重分析 + dirty propagation"。

**终态决策**：在"无兼容"原则下选**真做**。

不再保留"标注为 full-file bookkeeping 但叫 incremental"的混淆名。新设计：

- 现有"full-file 重分析"重命名为 `xa_analyzer_refresh_file()`（不掩饰能力）
- 新增真正的 block-level incremental API：`xa_analyzer_invalidate_range(file, start_line, end_line)`，仅在 LSP 增量编辑路径调用
- 短期实现策略：block-level 失效 → 触发 full-file refresh，但 API 名字不再骗人；后续替换为真增量时调用方零改动

#### 🔴 A-03 文件删除 / 符号删除路径未闭环（用户 FE-14）

`xa_dep_remove_symbol()` 已实现但无调用面；删除路径未同步清理依赖图边。

**终态**：

- 统一删除入口 `xa_analyzer_remove_file(analyzer, path)`：删 `XaFileCache` + 反查所有依赖该文件符号的边并删除 + 触发 dirty propagation
- 删除 `xa_dep_remove_symbol()` 的孤立 API，并入统一入口
- 加 `XR_DCHECK` 断言：删除后 `files_map.size`、`dep_graph.edge_count`、`symbol_table.size` 数量自洽

#### 🟡 A-04 visitor 大文件拆分

`xanalyzer_visitor.c` 2533 行 + `xanalyzer_visitor_expr.c` 1942 行已逼近硬线。

**终态**：按"语义阶段"拆，不按行数：

```text
analyzer/
├── xanalyzer_visitor.{c,h}        // 主入口与调度
├── xanalyzer_visitor_decl.c        // class/struct/enum/fn/let
├── xanalyzer_visitor_stmt.c        // 控制流 / try / scope
├── xanalyzer_visitor_expr.c        // 拆分后只剩二元/调用/字面量
├── xanalyzer_visitor_call.c        // 调用与 method dispatch（NEW）
├── xanalyzer_visitor_pattern.c     // match / 解构（NEW）
└── xanalyzer_visitor_internal.h
```

每文件 ≤ 1500 行。

### 2.5 代码生成层（codegen）

#### 🔴 C-01 `xexpr_higher_order.c` 直接 include `include/xray.h`（用户 FE-07）

frontend 内部模块依赖 L7 public API。

**终态**：删除 `include/xray.h` 引入，改为只 include 真正用到的 `runtime/*` / `vm/*` 内部头。

#### 🟡 C-02 第二梯队热点文件拆分

`xexpr_call.c` 1882、`xstmt_simple.c` 1418、`xexpr_binary.c` 1353、`xcompiler.c` 1728。

**终态**：按 lowering 主题拆：

```text
codegen/
├── xexpr_call.c           // 普通函数调用
├── xexpr_call_method.c    // 方法调用 / dispatch（NEW）
├── xexpr_call_ctor.c      // 构造调用（NEW）
├── xexpr_call_ho.c        // 高阶 / 闭包调用（NEW）
├── xstmt_simple.c         // 真·简单语句（拆掉带类型检查/桥接的）
├── xstmt_typed.c          // 带类型检查 / 运行时桥接的语句（NEW）
└── ...
```

不机械按行数拆，按 cohesion 拆。

### 2.6 跨阶段共享（AST / 公共 helper）

#### 🟡 X-01 AST 多阶段状态膨胀（用户 FE-10）

`AstNode` 同时挂位置信息 + formatter 注释 + 语义缓存（`compile_type`）。

**终态决策**：在"无兼容"原则下，**直接 side-table 化**，不保留过渡。

- 新增 `analyzer/xa_node_table.{c,h}`：`AstNode*` → semantic info 的哈希映射
- `compile_type` 从 `AstNode` 删除，改为 `xa_node_table_get_type(table, node)`
- formatter 路径不再触碰任何 semantic state（编译期检查：`format/` 不 include `analyzer/`）
- `AstNode` 只剩 `kind/start_line/start_col/end_line/end_col/leading_comments/trailing_comments` + 节点专属 union 字段

预期影响范围：analyzer、codegen 所有读 `node->compile_type` 的位置统一改为 table 查询。这是大改动，但符合"不留过渡"原则。

#### 🟡 X-02 `xdiag_fmt.h` 角色明确化

当前是 frontend 顶层公共 helper。

**终态**：保持位置不变（`src/frontend/xdiag_fmt.h`），但确认它**只做**诊断格式化字符串拼装，不承担任何高层语义。补 `XR_DCHECK` 守护。

---

## 3. 实施阶段（压缩为 3 个 Phase）

不留"过渡 phase"。

### Phase 1 — 词法 + 格式化层正确性 + parser 边界（约 3-4 天）

**单一 commit 不可分割**：

- L-01 ~ L-08（lexer 全套）
- F-01 ~ F-04（formatter 字符串/模板/拆分/去 public API 依赖）
- P-01（parser 自有 type scope，剪 analyzer 反向依赖）
- P-02、P-03（公共 API 收敛 + xast_nodes.h 拆分）
- P-04（删除 compatibility / reserved 模糊路径）

**验收**：

- 全部新增单测通过
- `tests/unit/frontend/test_frontend_architecture.c` 守护：
  - `parser/` 不 include `analyzer/`
  - `format/` 不 include `analyzer/` 或 `include/xray*.h`
  - `lexer/` 不 include `runtime/`
- formatter round-trip：随机一组源码格式化两次 = 同一输出
- `xast_nodes.h` ≤ 800 行

### Phase 2 — 语义层闭环 + AST side-table 化（约 4-5 天）

- A-01 ~ A-04（analyzer 全套）
- X-01（`compile_type` 移出 AST）
- C-01（codegen 去 public API 依赖）
- P-05（类型打印 helper 下沉）

**验收**：

- ASan 跑 analyzer 单测无泄漏
- 删除文件后 `dep_graph.edge_count` / `files_map.size` 自洽断言通过
- `format/` 子目录 grep 无 `analyzer/` 任何引用
- `frontend/**` grep 无 `include/xray.h` / `include/xray_isolate.h`

### Phase 3 — codegen 边界 + 测试网 + CI 守护（约 2-3 天）

- C-02（codegen 热点文件拆分）
- 完整测试覆盖（见 §4）
- CI 集成架构守护脚本

**验收**：

- 所有 `.c` ≤ 3000 行、`.h` ≤ 800 行
- `scripts/check_frontend_arch.sh` 接入 CI，违反即失败
- 单测数量 ≥ Phase 0 基线 + 30%

---

## 4. 测试覆盖矩阵

新建测试文件清单（按 phase 顺序）：

```text
tests/unit/frontend/
├── test_lexer.c                   # 现有，扩展
├── test_lexer_positions.c         # NEW Phase 1: 多行 token 坐标
├── test_lexer_keywords.c          # NEW Phase 1: 前缀标识符 + 全关键字
├── test_lexer_trivia.c            # NEW Phase 1: leading/trailing 归属
├── test_formatter_strings.c       # NEW Phase 1: 字符串/raw/template round-trip
├── test_formatter_comments.c      # NEW Phase 1: comment 保真
├── test_parser_type_scope.c       # NEW Phase 1: parser 自有 scope
├── test_parser_recoverable.c      # NEW Phase 1: 错误恢复稳定性
├── test_analyzer_incremental.c    # NEW Phase 2: dep / cache / dirty 闭环
├── test_analyzer_remove_file.c    # NEW Phase 2: 删除路径自洽
├── test_ast_side_table.c          # NEW Phase 2: compile_type side-table
└── test_frontend_architecture.c   # NEW Phase 3: include 边界 lint
```

**关键覆盖点**：

| 测试 | 验证 | 关联问题 |
|------|------|----------|
| `test_lexer_keywords::keyword_prefix_identifiers` | `iffy/letter/inflate` → TK_NAME | L-01, L-08 |
| `test_lexer_positions::multiline_string_position` | 多行字符串后续 token 列号正确 | L-02 |
| `test_lexer_trivia::trailing_comment_attached` | 行尾注释挂 `trailing_trivia` | L-06 |
| `test_formatter_strings::roundtrip_random` | 随机字符串两次 format = 同一输出 | F-01, F-02 |
| `test_parser_type_scope::generic_param_shadow` | 泛型参数作用域正确 | P-01 |
| `test_analyzer_remove_file::dep_edge_cleaned` | 删除文件后无悬挂边 | A-01, A-03 |
| `test_ast_side_table::no_compile_type_field` | compile_type 不在 AstNode | X-01 |
| `test_frontend_architecture::parser_no_analyzer` | grep 守护 include 边界 | P-01, F-04, C-01 |

---

## 5. 文件级变更清单

### 5.1 新增

| 文件 | Phase | 用途 |
|------|-------|------|
| `src/frontend/lexer/xkeywords.def` | 1 | X-macro 关键字单一数据源 |
| `src/frontend/parser/xtype_scope.{c,h}` | 1 | parser 自有类型作用域 |
| `src/frontend/parser/xast_nodes_common.h` | 1 | 拆分 xast_nodes |
| `src/frontend/parser/xast_nodes_expr.h` | 1 | 同上 |
| `src/frontend/parser/xast_nodes_stmt.h` | 1 | 同上 |
| `src/frontend/parser/xast_nodes_decl.h` | 1 | 同上 |
| `src/frontend/format/xfmt_literal.{c,h}` | 1 | 字符串/模板序列化 |
| `src/frontend/format/xfmt_expr.c` | 1 | xfmt.c 拆分 |
| `src/frontend/format/xfmt_stmt.c` | 1 | 同上 |
| `src/frontend/format/xfmt_decl.c` | 1 | 同上 |
| `src/frontend/format/xfmt_type.c` | 1 | 同上 |
| `src/frontend/format/xfmt_trivia.c` | 1 | 同上 |
| `src/frontend/analyzer/xa_node_table.{c,h}` | 2 | AST → semantic info side-table |
| `src/frontend/analyzer/xanalyzer_visitor_call.c` | 2 | visitor 拆分 |
| `src/frontend/analyzer/xanalyzer_visitor_pattern.c` | 2 | 同上 |
| `src/frontend/codegen/xexpr_call_method.c` | 3 | codegen 拆分 |
| `src/frontend/codegen/xexpr_call_ctor.c` | 3 | 同上 |
| `src/frontend/codegen/xexpr_call_ho.c` | 3 | 同上 |
| `src/frontend/codegen/xstmt_typed.c` | 3 | 同上 |
| `src/base/xtype_print.{c,h}` 或 `runtime/value/xtype_format.*` 扩展 | 2 | 类型打印 helper 下沉 |
| `scripts/check_frontend_arch.sh` | 3 | include 边界 lint |
| `tests/unit/frontend/test_*` | 1-3 | 见 §4 |

### 5.2 删除

| 文件 / 符号 | Phase | 原因 |
|-------------|-------|------|
| `xlex.c` 中 `is_hex_digit` 函数 | 1 | trivial wrapper |
| `xlex.c` `#include "../../runtime/value/xtype_names.h"` | 1 | 跨层死 include |
| `xlex.c` `<stdlib.h>` / `<ctype.h>`（验证后） | 1 | 未使用 |
| `xparse_*.c` 全部 `#include "../analyzer/xanalyzer_symbol.h"` | 1 | 反向依赖 |
| `xa_dep_remove_symbol()` 独立 API | 2 | 并入统一删除入口 |
| `AstNode::compile_type` 字段 | 2 | side-table 化 |
| `xexpr_higher_order.c` `#include "../../include/xray.h"` | 2 | 跨层依赖 |
| `xparse.h` 中 `xr_compile_type_to_string()` 声明 | 2 | 下沉 |
| `xfmt.c` 反引号模板输出代码 | 1 | 输出非法源码 |
| `xr_parse_set_literal()` compatibility 路径 | 1 | 无外部用户 |

### 5.3 修改（非平凡）

| 文件 | Phase | 主要改动 |
|------|-------|----------|
| `lexer/xlex.{c,h}` | 1 | L-01~L-08 全套 + Token 增 `error_message` 字段 |
| `parser/xparse_*.c` | 1 | P-01 type scope 替换 |
| `parser/xparse.h` | 1 | P-02 公共 API 收敛 |
| `parser/xast_nodes.h` | 1 | P-03 改为聚合头 |
| `format/xfmt.c` | 1 | F-03 内容拆出至各子文件 |
| `format/xfmt.h` | 1 | F-04 forward decl 替代 public API |
| `analyzer/xanalyzer_incremental.c` | 2 | A-01 reverse-edge 释放 + A-02 命名 |
| `analyzer/xanalyzer.c` | 2 | A-03 统一删除入口 |
| `codegen/*` | 2-3 | X-01 `compile_type` → side-table 查询 |

---

## 6. 风险与回滚

| 风险 | 触发场景 | 缓解 |
|------|----------|------|
| AST side-table 改造影响 codegen 全路径 | X-01 改动面大 | Phase 2 单独分支，全量 ctest + smoke 通过后合并；side-table 用 hash map，O(1) 查询，性能影响可控 |
| X-macro 关键字表 bug 导致词法回归 | L-08 实现错误 | 先补 `test_lexer_keywords` 全关键字单测，再做替换；对每个关键字加 prefix 测试 |
| formatter canonical form 影响现有 `.xr` 文件外观 | F-02 raw string → 普通字符串规范化 | 项目原则下接受 canonical rewrite；提供 `xray fmt --check` 让用户主动选择是否改写仓库 |
| analyzer 删除路径改动引入 dirty propagation 回归 | A-03 | 加 `XR_DCHECK` 断言文件/符号/边数量自洽；ASan 跑全量 LSP 测试 |
| Phase 1 单 commit 过大难 review | 改动跨 4 个子模块 | Phase 1 内部按子模块开 sub-PR，但上游分支测试一次性合并 |

**回滚策略**：每个 Phase 独立分支；单 Phase 失败不阻塞其它 Phase。Phase 1/2/3 之间无强依赖（Phase 2 的 X-01 假设 Phase 1 已拆分 xast_nodes，但即使未拆 X-01 也可独立完成）。

---

## 7. 与原版文档的关系

| 原版条目 | 本文条目 | 决策变化 |
|----------|----------|----------|
| FE-01 多行 string 位置 | L-02 | 沿用 |
| FE-02 formatter 反引号 | F-01 | 沿用，更激进 |
| FE-03 string 序列化 | F-02 | **加强**：raw string 直接 canonical rewrite，不保留 lexeme |
| FE-04 trailing trivia 半实现 | L-06 | **加强**：原版"二选一"→ 直接选真做 |
| FE-05 reverse-edge 泄漏 | A-01 | 沿用 |
| FE-06 parser→analyzer 反向 | P-01 | 沿用 |
| FE-07 public API 头依赖 | F-04, C-01 | 沿用 |
| FE-08 xparse.h / xast_nodes.h | P-02, P-03 | 沿用 |
| FE-09 incremental 漂移 | A-02 | **加强**：原版"标注为 bookkeeping"→ 真做 + API 重命名 |
| FE-10 AST 多阶段状态 | X-01 | **加强**：原版"短期保留 compile_type"→ 直接 side-table 化 |
| FE-11 codegen 借 parser helper | P-05 | 沿用 |
| FE-12 `Token.start` 双义 | L-03 | 沿用 |
| FE-13 关键字 trie 维护 | L-08 | **加强**：原版"P2 后续"→ Phase 1 直接换 X-macro |
| FE-14 删除路径未闭环 | A-03 | 沿用，加强为统一入口 |
| FE-15 xfmt.c 单文件巨型 | F-03 | 沿用 |
| **CL-LEX-01** `TK_IF` bug | L-01 | 新增（深审版独有） |
| **CL-LEX-02** xtype_names 死 include | L-05 | 新增 |
| **CL-LEX-03** trivia NULL 检查 | L-04 | 新增 |
| **CL-LEX-04** TRIVIA 注释错字 | L-07 | 新增 |
| **CL-LEX-05~07** stdlib/ctype/wrapper/稀疏表 | L-05, L-07 | 新增 |
| **CL-LEX-08** 关键字 trie | L-08 | 与 FE-13 合并强化 |

**净增量**：相比原版，本文升级了 5 项（FE-03/04/09/10/13），合并 8 项 lexer 深审发现，整体压缩 7 个 Phase 为 3 个 Phase。

---

## 8. 待审计模块（Phase 之外）

本文已确认的问题集中在**已审计的 lexer + 已扫描的 parser/format/analyzer/codegen**。未做完逐文件深审的子目录留待后续：

- `parser/` 18 文件 — 已知超大文件清单，未逐函数审；P-04 中可能有更多 compatibility 路径待清理
- `analyzer/` 30 文件 — visitor 之外的 `xanalyzer_flow.c` / `xanalyzer_escape.c` / `xanalyzer_mono.c` / `xanalyzer_jit.c` 未深审
- `codegen/` 57 文件 — 体量最大，仅观察了热点文件

后续若深审发现新问题，按本文同样格式追加（不再开新文档）。

---

## 9. 决策摘要

```text
保留：lexer 整体架构 / parser Pratt 设计 / formatter 节点驱动 / analyzer visitor 模式
删除：跨层死 include · 反向依赖 · compatibility 路径 · 反引号模板 · trivial wrapper
重写：关键字 trie → X-macro · raw string lexeme 保留 → canonical rewrite · AST 内嵌 compile_type → side-table
拆分：xast_nodes.h · xfmt.c · xanalyzer_visitor*.c · xexpr_call.c
新增：xtype_scope · xa_node_table · xfmt_literal · 测试 + CI 守护
```

每一项都选最佳形态，没有过渡阶段。
