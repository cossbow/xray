# Frontend 模块重构实施文档（`src/frontend`）

> 日期：2026-04-25
> 状态：Draft
> 范围：`src/frontend/lexer`、`parser`、`format`、`analyzer`、`codegen`、`xdiag_fmt.h`

---

## 1. 文档定位

本文把当前对 `src/frontend` 的审计结论收敛为一份**可执行的实施方案**，用于后续按阶段推进前端正确性、架构边界、模块职责和测试覆盖的收敛。

### 1.1 审计范围与置信度说明

本文基于两类信息：

1. **已完成的深审**
   - `src/frontend/lexer` 已做过一轮较细的实现审计。

2. **已完成的结构性扫描**
   - `parser / format / analyzer / codegen` 已完成入口、关键依赖、超大文件、显式 TODO、头文件耦合和若干 correctness spot check。

因此，本文中的结论分两类：

- **已确认问题**：已有直接源码证据，可直接纳入实施计划。
- **待二次深审项**：目前已看到风险信号，但还未逐文件逐函数验证；这些项只进入较后阶段，不作为 Phase 0/1 的强依赖。

### 1.2 开发原则

遵循现有工程风格，本文采用以下原则：

- **正确性优先于局部性能**
- **不保留错误契约**，名字、注释、实现必须一致
- **优先修正分层错误**，避免继续在错误边界上叠功能
- **每个 Phase 独立可合并、独立可回滚**
- **不把“结构性债务”伪装成“后续优化”**

---

## 2. 当前模块地图

### 2.1 当前目录结构

```text
src/frontend/
├── lexer/
│   ├── xlex.c
│   └── xlex.h
├── parser/
│   ├── xast*.{c,h}
│   ├── xparse*.{c,h}
│   └── ...
├── format/
│   ├── xfmt.c
│   └── xfmt.h
├── analyzer/
│   ├── xanalyzer*.{c,h}
│   └── ...
├── codegen/
│   ├── xcompiler*.{c,h}
│   ├── xexpr*.{c,h}
│   ├── xstmt*.{c,h}
│   ├── xoop*.{c,h}
│   └── ...
└── xdiag_fmt.h
```

### 2.2 期望的内部依赖方向

`src/frontend` 虽然整体同属 L4，但内部仍应有明确的“低到高”顺序：

```text
lexer
  → parser/ast
    → format
    → analyzer
      → codegen
```

补充约束：

- `parser` **不应**反向依赖 `analyzer`
- `format` **不应**依赖 L7 public API 头
- `codegen` **不应**直接 include `include/xray.h`
- `xdiag_fmt` 应保持轻量公共辅助角色，不承载高层语义逻辑

### 2.3 当前基线（按文件规模）

已确认的若干关键文件规模如下：

| 模块 | 文件 | 当前规模 | 备注 |
|------|------|----------|------|
| lexer | `lexer/xlex.c` | 1041 行 | 中等偏大，仍可维护 |
| parser | `parser/xparse.c` | 1556 行 | Pratt 主入口 + 初始化 + 错误恢复 |
| parser | `parser/xparse_decl.c` | 1572 行 | 声明解析过重 |
| parser | `parser/xparse_oop.c` | 1737 行 | OOP 语法解析偏重 |
| parser | `parser/xast.c` | 1897 行 | AST 工厂 + 工具函数较多 |
| parser | `parser/xast_nodes.h` | 807 行 | **已超过 `.h <= 800` 项目硬线** |
| format | `format/xfmt.c` | 1737 行 | 单文件承载全部格式化逻辑 |
| analyzer | `analyzer/xanalyzer_visitor.c` | 2533 行 | 核心 visitor 已非常接近维护上限 |
| analyzer | `analyzer/xanalyzer_visitor_expr.c` | 1942 行 | 表达式语义过于集中 |
| analyzer | `analyzer/xanalyzer_mono.c` | 1427 行 | AST mono clone 较重 |
| codegen | `codegen/xcompiler.c` | 1728 行 | 编译主入口过重 |
| codegen | `codegen/xexpr_call.c` | 1882 行 | 调用语义与 lowering 高耦合 |
| codegen | `codegen/xstmt_simple.c` | 1418 行 | 简单语句路径已经不简单 |
| codegen | `codegen/xexpr_binary.c` | 1353 行 | 二元表达式 lowering 集中 |

结论：

- 当前 **没有 `.c > 3000`** 的硬性越线文件。
- 但已有多个“**第二梯队大文件**”，再继续增长就会逼近硬线。
- `xast_nodes.h` 已经实质性越线，应尽早拆分或压缩暴露面。

---

## 3. 已确认问题总览

下面只列**已有源码证据**的问题。

### 3.1 P0：correctness / 输出合法性问题

### FE-01 多行字符串 token 的位置契约不可靠

**模块**：`lexer`

已确认现象：

- `make_token()` 使用扫描结束时的 `scanner->line`
- `string_with_quote()` / `raw_string_with_quote()` 在遇到换行时只更新 `line`
- 同一过程中未同步更新 `line_start`

后果：

- 多行字符串 token 的 `(line, column)` 更接近“结束位置”而不是“起始位置”
- 后续 token 的列号计算可能漂移
- parser / analyzer / LSP 的错误定位会受污染

这属于前端最底层的 correctness 问题，优先级最高。

### FE-02 formatter 会输出**当前 lexer 不接受**的模板字符串语法

**模块**：`format`

已确认现象：

- lexer 侧已将反引号字符串视为 deprecated / error
- 但 `xfmt.c` 中 `fmt_template_string()` 仍直接输出 `` `...${}...` ``
- 当前 parser 的模板串入口来自普通引号字符串中的 `${}` 插值，而不是反引号

后果：

- formatter 对模板字符串的输出**不具备 round-trip 保证**
- 一次格式化后，源码可能变为当前前端无法再次接受的语法

这是一个明确的 P0 输出合法性问题。

### FE-03 formatter 对字符串字面量的序列化不保语义

**模块**：`parser` + `format`

已确认现象：

- parser 在 `xr_parse_literal()` / `xr_parse_template_string()` 中把字符串字面量解码为语义值
- `xfmt.c` 在输出 `AST_LITERAL_STRING` 时，直接把 `raw_value.string_val` 原样写回双引号中
- 没有重新做转义，也没有保留原始 raw-string / quote style / escape form

后果：

- 包含换行、反斜杠、引号、控制字符的字符串，格式化后可能变成非法源码或语义变化源码
- raw string 的“原样文本”属性被丢失

这同样属于 P0，因为它直接影响 formatter 的可用性和可信度。

### FE-04 inline comment / trailing trivia 没有闭环

**模块**：`lexer` + `parser` + `format`

已确认现象：

- `Token` 有 `trailing_trivia` 字段，但当前始终为 `NULL`
- parser 只消费 `leading_trivia`
- formatter 只输出 `leading_comments`

后果：

- 行尾注释缺少稳定归属模型
- 一旦源码中存在 inline comment，格式化保真度不可信
- trivia API 与实际能力存在漂移

### FE-05 analyzer 依赖图释放路径存在真实内存泄漏

**模块**：`analyzer`

已确认现象：

- `xa_dep_add()` 会分别分配 `forward` 边和 `reverse` 边节点
- `dep_graph_free()` 只释放了 `forward` 链表
- 注释写的是“reverse edges point to same nodes”，但实现并非如此

后果：

- analyzer incremental context 销毁时会泄漏全部 reverse 边
- 注释与实现已明显漂移

这不是纯优化项，而是确定的内存管理错误。

---

### 3.2 P1：契约漂移 / 架构边界问题

### FE-06 parser 反向依赖 analyzer，破坏 frontend 内部层次

**模块**：`parser`

已确认现象：

- `xparse.c` / `xparse_expr.c` / `xparse_decl.c` / `xparse_type.c` / `xparse_oop.c`
  直接 include `../analyzer/xanalyzer_symbol.h`
- parser 目前用 `XaScope` 存放 type alias / generic type param 解析作用域

后果：

- `parser -> analyzer` 形成内部反向依赖
- analyzer 的符号作用域抽象泄漏到 parser
- 后续如果要独立演进 parser（例如更轻量的 recoverable parse / AST-only parse），会被 analyzer 结构绑住

这不是总架构层级错误，但属于 `src/frontend` 内部的明显边界设计错误。

### FE-07 `xfmt.h` 依赖 public API 头，`codegen` 直接 include `include/xray.h`

**模块**：`format` + `codegen`

已确认现象：

- `format/xfmt.h` 直接 include `include/xray_isolate.h`
- `codegen/xexpr_higher_order.c` 直接 include `../../include/xray.h`

后果：

- `frontend` 内部模块对 L7 public API 形成不必要依赖
- 破坏“低层不依赖高层”的方向性
- 增大编译扇出，模糊真正依赖面

### FE-08 parser public header 暴露面过大，内外 API 混杂

**模块**：`parser`

已确认现象：

- `xparse.h` 暴露了大量仅供 parser 子文件互调的内部函数
- `xparse_internal.h` 已存在，但公共/内部边界没有真正收紧
- `xast_nodes.h` 已超过项目头文件 800 行硬线

后果：

- parser 内部拆分难度变大
- 任何调用者都能误用内部 parse helper
- review 和静态审计成本偏高

### FE-09 analyzer 的“incremental”命名已经超出真实能力

**模块**：`analyzer`

已确认现象：

- `xanalyzer_incremental.h` / `xanalyzer.h` 对外暴露了较完整的 incremental API
- 但 `xa_incremental_update()` 自己明确写着 `TODO: Implement true incremental parsing`
- `xa_analyzer_update_incremental()` 当前仍是“移除整文件旧符号 + 整文件重分析 + dirty propagation”

后果：

- 名字让维护者误以为已经是精细增量分析
- 实际上目前更接近“带 hash / dependency bookkeeping 的 full-file update”
- 容易在 LSP / IDE 侧被过度信任

### FE-10 AST 同时承载 parser / formatter / analyzer / codegen 的多类状态

**模块**：`parser` + `format` + `analyzer` + `codegen`

已确认现象：

`AstNode` 目前同时携带：

- 语法位置信息：`line/column/end_line/end_column`
- formatter 注释：`leading_comments`
- 语义缓存：`compile_type`

后果：

- AST 变成跨阶段共享可变结构
- formatter / syntax-only 路径与 analyzer/codegen 路径耦合
- 未来若要做 parser-only 或 formatter-only 的轻量路径，边界会非常难收紧

### FE-11 codegen 依赖 parser 的“类型打印辅助”，边界过粗

**模块**：`codegen`

已确认现象：

- `xoop_class.c` 通过 include `xparse.h` 使用 `xr_compile_type_to_string()`

后果：

- codegen 被迫依赖完整 parser 公共头
- 一个调试/辅助函数把模块边界拉粗了

---

### 3.3 P2：维护性 / 可演进性问题

### FE-12 lexer 的 `Token` 契约不纯

**模块**：`lexer`

已确认现象：

- 正常 token 的 `start` 指向源码片段
- `TK_ERROR` 的 `start` 却被复用为错误消息字符串

后果：

- `Token.start` 的语义不再统一
- 任何复用 token 的通用逻辑都必须额外知道 `TK_ERROR` 特例

### FE-13 lexer 关键字识别逻辑已进入高维护成本区

**模块**：`lexer`

已确认现象：

- `identifier_type()` 是一个非常长的手写分支树
- 关键字表与 `token_names[]` 不是同一份数据源

后果：

- 新增关键字时容易引入分支覆盖错误
- 关键字表与名字表有双重维护成本

### FE-14 analyzer 的依赖图/缓存删除路径没有闭环

**模块**：`analyzer`

已确认现象：

- `xa_dep_remove_symbol()` 已实现，但当前无调用面
- 文件删除/符号删除路径里，没有看到依赖图边同步删除
- `XaFileCache` / dependency graph / symbol table 三者不是一条统一的删除链路

后果：

- stale edge 会持续积累
- dirty propagation 的长期行为不可预测
- incremental 统计数据会逐渐失真

### FE-15 formatter 仍然是单文件巨型实现

**模块**：`format`

已确认现象：

- `xfmt.c` 1737 行，承载 literal / stmt / block / type / comment 输出全部路径

后果：

- 后续一旦补齐 string/trivia/range-format 行为，文件还会继续增长
- formatter 的 correctness hotfix 与风格调整难以分开 review

---

## 4. 重构目标

### 4.1 最终目标

1. **前端输出合法且可 round-trip**
   - lexer 提供可靠位置
   - formatter 输出当前 parser 真正接受的语法
   - string/template/comment 不再破坏源码语义

2. **frontend 内部层次清晰**
   - `parser` 不再依赖 `analyzer`
   - `format` / `codegen` 不再依赖 public API 头
   - 共享抽象只保留真正底层、语义稳定的部分

3. **incremental / dependency / cache 契约真实可信**
   - 名字、注释、实现一致
   - 删除路径闭环
   - 内存/缓存不泄漏、不漂移

4. **头文件与大型文件收敛**
   - `xast_nodes.h` 回到硬线以内
   - parser सार्वजनिक API 缩减
   - formatter / analyzer / codegen 的超大热点文件进一步拆分

5. **测试回归网覆盖关键路径**
   - 词法位置
   - formatter round-trip
   - parser 类型作用域
   - analyzer 增量删除/dirty propagation
   - architecture include 边界

### 4.2 非目标

本文不要求一步做到以下事项：

- 真正 AST 级别的 fine-grained incremental parsing
- 一次性重写整个 analyzer / codegen
- 立刻把所有 AST 语义缓存都迁出 AST
- 立刻完成 formatter 的 range formatting / minimal diff formatting

策略是：

- **先修 correctness 与边界问题**
- **再收敛契约与抽象**
- **最后处理大规模结构整理**

---

## 5. 目标架构

### 5.1 Frontend 内部层次

建议把 `src/frontend` 视为下面这条明确链路：

```text
F0  frontend/common-diag
    └── xdiag_fmt.h（后续可视需要下沉/重命名）

F1  lexer
    └── xlex.[ch]

F2  parser + ast
    ├── xast*.{c,h}
    ├── xparse*.{c,h}
    └── parser-owned type scope / soft-keyword helpers

F3  format
    └── xfmt.[ch]

F4  analyzer
    └── xanalyzer*.{c,h}

F5  codegen
    └── xcompiler / xexpr / xstmt / xoop / optimize / regalloc
```

关键约束：

- `F2 parser` 只能依赖 `F1 lexer` 与 AST 本身
- `F3 format` 只能依赖 `F2 parser/ast` + 必要的低层类型打印 helper
- `F4 analyzer` 依赖 `F2 parser/ast`
- `F5 codegen` 依赖 `F2 parser/ast` 和 `F4 analyzer`

### 5.2 AST 的边界原则

短期内不强行重写 AST，但应明确：

- **语法位置信息**属于 AST 稳定契约
- **formatter trivia 附着**可以暂时留在 AST，但必须有清晰闭环
- **语义缓存**（如 `compile_type`）属于后续可迁移对象，不能再继续任意扩张

### 5.3 formatter 的合法输出原则

formatter 不必保留原始字面 lexeme，但必须满足：

- 输出源码能被当前 parser 接受
- string / raw string / template string 的语义不变
- comment 至少不丢失，不乱绑定

换言之，短期可接受“规范化重写”，但不可接受“生成无效源码”或“语义被改写”。

---

## 6. 分阶段实施计划

### 阶段总览

| Phase | 目标 | 优先级 | 产出 |
|------|------|--------|------|
| 0 | 修复 lexer/formatter correctness hotfix | P0 | 位置正确、formatter 不再输出非法源码 |
| 1 | 收敛 trivia / string / template 输出契约 | P0/P1 | comment 与字面量模型闭环 |
| 2 | 修复 parser 内部层次与公共 API 暴露 | P1 | parser 不再依赖 analyzer，头文件收敛 |
| 3 | 修复 analyzer incremental/delete/caching correctness | P1 | 依赖图删除闭环、命名真实 |
| 4 | 收敛 frontend 内部共享抽象 | P1/P2 | AST 职责更清晰，公共 helper 下沉 |
| 5 | 清理 codegen 边界与热点大文件 | P2 | codegen 依赖面变窄，文件体量回落 |
| 6 | 测试与架构回归补齐 | P1/P2 | 有最小可信回归网 |

---

### 6.1 Phase 0 — lexer / formatter correctness 热修

### 目标

1. lexer 提供可靠 token 位置
2. formatter 不再输出当前 parser 不接受的源码
3. 字符串输出保证语义正确

### 主要改动

#### 0.1 修复多行字符串位置跟踪

涉及文件：

- `src/frontend/lexer/xlex.c`
- `src/frontend/lexer/xlex.h`

建议方案：

- 在 scanner 内部显式记录 token 起始 `line/column` 或 `start_line/start_line_start`
- `string_with_quote()` / `raw_string_with_quote()` 中遇到换行时同步更新 `line_start`
- `make_token()` 统一使用 token 起点坐标，而不是扫描结束时状态

#### 0.2 修复 formatter 的模板串输出语法

涉及文件：

- `src/frontend/format/xfmt.c`

要求：

- 禁止再输出反引号模板串
- 改为输出当前语言实际支持的引号模板语法
- 对模板段中的字面文本执行必要转义，确保重新 parse 成功

#### 0.3 增加统一的字符串序列化 helper

涉及文件：

- `src/frontend/format/xfmt.c`
- 必要时新增 `src/frontend/format/xfmt_string.c` / `xfmt_string.h`

要求：

- 普通字符串：重新转义 `"`、`\\`、控制字符、换行、制表符等
- raw string：短期可统一规范化为普通双引号字符串，只要语义不变
- template literal segment：既要逃逸引号/反斜杠，也要防止错误拼出 `${` 边界

#### 0.4 修复 trivia 分配失败路径

涉及文件：

- `src/frontend/lexer/xlex.c`

要求：

- `xr_trivia_new()` 检查 `xr_malloc()` 返回值
- 注释收集失败不能直接解引用空指针

### 验收标准

- 多行字符串后的后续 token 列号稳定
- formatter 处理模板串后，重新 parse 不报语法错误
- 带换行/引号/反斜杠的字符串格式化后语义不变
- lexer 注释收集路径无空指针崩溃风险

### 建议测试

- `tests/unit/frontend/test_lexer_positions.c`
  - 多行字符串起始坐标
  - multiline raw string / template string 坐标
- `tests/unit/frontend/test_formatter_strings.c`
  - 普通字符串 round-trip
  - raw string 规范化后 round-trip
  - template string round-trip

---

### 6.2 Phase 1 — trivia / comment / literal 契约闭环

### 目标

1. trivia 模型不再半实现
2. inline comment 不再静默丢失
3. formatter 的字面量与注释输出模型清晰稳定

### 主要改动

#### 1.1 明确 trailing comment 设计

涉及文件：

- `src/frontend/lexer/xlex.h`
- `src/frontend/lexer/xlex.c`
- `src/frontend/parser/xast_nodes.h`
- `src/frontend/parser/xparse.c`
- `src/frontend/format/xfmt.c`

二选一，建议直接选**闭环实现**：

- lexer 真正填充 `trailing_trivia`
- parser 为可挂注释的节点增加 inline comment 附着位，或定义“上一语句 trailing comment 归属规则”
- formatter 按节点位置输出 trailing comment

不要继续保留“字段存在但无实现”的状态。

#### 1.2 统一字面量 canonical formatting 规则

要求：

- formatter 可统一使用双引号作为规范输出
- 若暂不保留原始 quote/raw 风格，则文档与注释必须明确“formatter 做 canonical rewrite，不做 lexeme preservation”
- 对 template string 的 canonical form 做一致定义

#### 1.3 清理 lexer trivia API 漂移

要求：

- 若 trailing trivia 最终不做，删除对应字段和注释
- 若保留，则所有相关路径必须真正接通

### 验收标准

- inline comment / leading comment 均不再被静默丢失
- formatter 对字符串、模板串、注释的行为是可预测且文档化的
- trivia 相关 public struct 字段与真实实现一致

---

### 6.3 Phase 2 — parser 层次修复与公共 API 收敛

### 目标

1. `parser` 不再依赖 `analyzer`
2. `xparse.h` 只暴露真正的 parser public API
3. `xast_nodes.h` 回到体量红线以内

### 主要改动

#### 2.1 提取 parser 自有 type scope

涉及文件：

- `src/frontend/parser/xparse.c`
- `src/frontend/parser/xparse_decl.c`
- `src/frontend/parser/xparse_type.c`
- `src/frontend/parser/xparse_oop.c`
- 新增建议：`src/frontend/parser/xtype_scope.[ch]`

建议设计：

```c
typedef struct XrTypeScope XrTypeScope;

XR_FUNC XrTypeScope *xr_type_scope_new(XrTypeScope *parent);
XR_FUNC void xr_type_scope_free(XrTypeScope *scope);
XR_FUNC bool xr_type_scope_define_alias(XrTypeScope *scope, const char *name, XrType *type);
XR_FUNC XrType *xr_type_scope_resolve_alias(XrTypeScope *scope, const char *name);
```

目标：

- parser 用自己的轻量 alias scope
- 移除 `../analyzer/xanalyzer_symbol.h` 对 parser 的反向渗透

#### 2.2 缩减 `xparse.h` 暴露面

要求：

- `xparse.h` 只保留：parse entry、recoverable parse、极少量确需对外的 helper
- Pratt rule、具体子语法函数、内部 parse helper 全部转入 `xparse_internal.h`
- 下游模块不得再 include 过多内部 parse 细节

#### 2.3 拆分 `xast_nodes.h`

建议拆法：

```text
parser/
├── xast_nodes_common.h
├── xast_nodes_expr.h
├── xast_nodes_stmt.h
├── xast_nodes_decl.h
└── xast_nodes.h          # 薄聚合头
```

要求：

- 总接口保持不变
- 聚合头可继续存在，但主定义文件不再单头超过 800 行

#### 2.4 处理 parser 中的 compatibility / reserved 路径

已看到的典型信号：

- `xr_parse_set_literal()` 仍以“compatibility”注释保留
- `yield` 为 reserved for future

要求：

- 把“为了兼容暂留”的接口与“语言保留字但未实现”的接口分别标注清楚
- 不保留模糊状态

### 验收标准

- parser 子模块编译不再需要 analyzer 头文件
- `xparse.h` 显著变薄
- `xast_nodes.h` 回到项目头文件硬线以内
- parser 的对外/对内 API 边界清晰可解释

### 建议测试

- `tests/unit/frontend/test_parser_type_scope.c`
  - generic type param scope
  - type alias shadowing
- `scripts/check_architecture.sh`
  - 重点关注 `parser -> analyzer` 反向 include 消失

---

### 6.4 Phase 3 — analyzer incremental / dependency / cache 正确性收敛

### 目标

1. incremental 相关注释、命名、实现一致
2. 删除路径闭环
3. dependency graph 不泄漏、不残留明显 stale edge

### 主要改动

#### 3.1 修复 dependency graph reverse-edge 释放错误

涉及文件：

- `src/frontend/analyzer/xanalyzer_incremental.c`

要求：

- `dep_graph_free()` 必须显式释放 reverse 链表节点
- 注释同步修正，禁止继续保留错误说明

#### 3.2 建立文件删除时的依赖图清理路径

涉及文件：

- `src/frontend/analyzer/xanalyzer.c`
- `src/frontend/analyzer/xanalyzer_incremental.c`

要求：

- 当符号从某文件删除时，依赖边同步清理
- `xa_dep_remove_symbol()` 要么真正接入，要么删除并以更合适的批量清理 API 取代
- `xa_cache_invalidate_file()` / file removal / symbol removal 三条路径合并为统一 helper

#### 3.3 收敛 incremental 命名与契约

二选一：

- **要么**明确标注当前能力为 full-file incremental bookkeeping
- **要么**逐步实现更细粒度的 block/function 级更新

在当前阶段建议先做前者：

- 修正文档/注释/API 描述
- 不再让调用者误以为已经具备真正精细增量分析

#### 3.4 补齐 analyzer 删除/更新路径统计与断言

要求：

- 文件删除后，`files_map`、dependency graph、file cache、diagnostics 的数量变化可断言
- dirty propagation 只处理当前仍存在的 symbol id
- 对长期累计数据增加最小健康检查

### 验收标准

- analyzer free 不再泄漏 reverse edge
- 文件删除/替换后 dependency graph 不持续膨胀
- incremental 路径的名字、注释、实现一致
- stale edge 不再成为长期内存/行为风险

### 建议测试

- `tests/unit/frontend/test_analyzer_incremental.c`
  - dependency add/remove
  - file delete invalidates cache
  - dirty propagation on symbol removal
- 若已有 ASan 工作流，可在该阶段重点跑 analyzer 单测

---

### 6.5 Phase 4 — 前端共享抽象与边界再收敛

### 目标

1. AST 上的共享状态不再无序扩张
2. frontend 公共 helper 放到正确位置
3. formatter / analyzer / codegen 的共享依赖面变窄

### 主要改动

#### 4.1 评估 `compile_type` 的长期归属

涉及文件：

- `src/frontend/parser/xast_nodes.h`
- `src/frontend/analyzer/*`
- `src/frontend/codegen/*`

短期策略：

- 保留 `compile_type`，不做破坏性迁移
- 明确禁止继续向 `AstNode` 追加新的 analyzer/codegen 专用可变字段

中期候选：

- `compile_type` 迁到 side table / node metadata map
- formatter / parser-only 路径不再碰 semantic state

#### 4.2 提取公共 helper，消除粗粒度 include

优先处理：

- `xr_compile_type_to_string()` 从 parser 公共头中抽离
- 若需要共享类型打印，迁到更合适的低层 helper（例如 frontend/type-print 或 runtime/value 打印辅助）

#### 4.3 清理 frontend 对 public API 头的依赖

要求：

- `xfmt.h` 用 forward declaration 代替 `include/xray_isolate.h`
- `codegen/xexpr_higher_order.c` 不再直接 include `include/xray.h`
- 只 include 真正需要的窄接口头

### 验收标准

- frontend 内部源文件基本不再依赖 `include/xray.h` / `include/xray_isolate.h`
- parser 不再是“类型打印 helper”的承载方
- AST 字段职责更稳定，新增需求不会继续往 `AstNode` 塞状态

---

### 6.6 Phase 5 — codegen 热点文件与模块边界清理

### 目标

1. 收窄 codegen 依赖面
2. 将几处热点大文件拆到更好维护的粒度
3. 避免 codegen 因共享 helper 继续把 parser/analyzer 头整包拉进来

### 主要改动

#### 5.1 清理粗粒度 include

已确认的优先项：

- `xoop_class.c` 不再通过 `xparse.h` 获取类型打印 helper
- `xexpr_higher_order.c` 不再直接 include `include/xray.h`

#### 5.2 第二梯队热点文件拆分

建议优先观察并视改动量拆分：

- `codegen/xexpr_call.c`
- `codegen/xstmt_simple.c`
- `codegen/xexpr_binary.c`
- `codegen/xcompiler.c`

拆分原则：

- 按 lowering 主题拆，而不是机械按行数拆
- 例如：普通函数调用 / method call / constructor call / higher-order call 分开
- 简单语句与带类型检查/运行时桥接的路径分开

#### 5.3 收敛 analyzer → codegen 数据交互面

当前 codegen 多处包含 analyzer 头。短期可接受，但中期应考虑：

- 通过更窄的 query 接口使用 analyzer 结果
- 避免把大而全的 analyzer 头长期暴露给每个 lowering 子文件

### 验收标准

- codegen 不再依赖 parser 的粗粒度公共头做辅助工作
- 热点文件的职责更单一
- 新功能不再默认堆进 `xcompiler.c` / `xexpr_call.c` / `xstmt_simple.c`

---

### 6.7 Phase 6 — 测试与回归体系补齐

### 目标

建立 `src/frontend` 的最小可信回归网，而不是继续依赖手工验证。

### 建议测试布局

```text
tests/unit/frontend/
├── test_lexer_positions.c
├── test_lexer_trivia.c
├── test_formatter_strings.c
├── test_formatter_comments.c
├── test_parser_type_scope.c
├── test_parser_recoverable.c
├── test_analyzer_incremental.c
└── test_frontend_architecture.c
```

### 建议覆盖点

#### 6.1 lexer

- 多行字符串起始/结束坐标
- raw/template token 坐标
- trivia leading/trailing 归属

#### 6.2 formatter

- string / raw string / template string round-trip
- leading / trailing comment 保真
- formatter 输出可被 parser 重新接受

#### 6.3 parser

- type alias scope
- generic param scope
- recoverable parse 在语法错误下的稳定行为

#### 6.4 analyzer

- dependency graph add/remove/free
- file cache invalidation
- full-file incremental bookkeeping 行为与注释一致

#### 6.5 architecture

至少检查：

- `parser` 不再 include `analyzer`
- `frontend` 不再直接 include `include/xray.h`
- `xfmt.h` 不再依赖 `include/xray_isolate.h`

### 验收标准

- `src/frontend` 的关键 correctness 路径均有单测覆盖
- 至少有一组 formatter round-trip 回归
- 架构边界有脚本或单测级守护

---

## 7. 建议文件变更总览

### 7.1 新增文件（建议）

| 文件 | Phase | 说明 |
|------|-------|------|
| `docs/engineering/frontend_refactor_plan.md` | now | 前端实施文档 |
| `src/frontend/parser/xtype_scope.h` | 2 | parser 自有 type alias scope |
| `src/frontend/parser/xtype_scope.c` | 2 | parser 自有 type alias scope 实现 |
| `tests/unit/frontend/test_lexer_positions.c` | 0 | token 位置回归 |
| `tests/unit/frontend/test_formatter_strings.c` | 0 | string/template round-trip |
| `tests/unit/frontend/test_formatter_comments.c` | 1 | trivia/comment 输出 |
| `tests/unit/frontend/test_parser_type_scope.c` | 2 | parser type scope |
| `tests/unit/frontend/test_analyzer_incremental.c` | 3 | analyzer incremental / cache / deps |

### 7.2 修改文件（核心）

| 文件 | Phase | 改动 |
|------|-------|------|
| `src/frontend/lexer/xlex.c` | 0/1 | 多行位置、trivia 闭环、错误 token 契约 |
| `src/frontend/lexer/xlex.h` | 1 | trivia/token contract 收敛 |
| `src/frontend/format/xfmt.c` | 0/1 | template/string 输出合法化，comment 闭环 |
| `src/frontend/format/xfmt.h` | 4 | forward declaration，收窄依赖 |
| `src/frontend/parser/xparse.c` | 2 | parser init/type_scope 去 analyzer 依赖 |
| `src/frontend/parser/xparse_decl.c` | 2 | generic/type alias 作用域迁移 |
| `src/frontend/parser/xparse_type.c` | 2 | type alias lookup 改用 parser-owned scope |
| `src/frontend/parser/xparse_oop.c` | 2 | class/struct generic scope 改造 |
| `src/frontend/parser/xparse.h` | 2/4 | public API 缩减，helper 抽离 |
| `src/frontend/parser/xast_nodes.h` | 2/4 | 拆分，回到硬线以内 |
| `src/frontend/analyzer/xanalyzer_incremental.c` | 3 | reverse-edge free、delete 闭环、契约修正 |
| `src/frontend/analyzer/xanalyzer.c` | 3 | file remove/update 删除链路统一 |
| `src/frontend/codegen/xoop_class.c` | 4/5 | 摆脱 parser helper 粗依赖 |
| `src/frontend/codegen/xexpr_higher_order.c` | 4/5 | 移除 `include/xray.h` 依赖 |
| `docs/engineering/README.md` | now | 增加文档入口 |

---

## 8. 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 修 formatter 过程中改变现有输出风格 | 回归噪音 | 先保 correctness，再在 Phase 1 明确 canonical style |
| parser 去 analyzer 依赖时影响 type alias/generic 解析 | 语法回归 | 先补 parser type scope 单测，再迁移 |
| analyzer delete path 改造引入 dirty propagation 回归 | LSP / compile 行为变化 | Phase 3 只先收敛 delete/free/caching，不一步上真正细粒度增量 |
| AST 边界调整过快影响 codegen | 编译路径震荡 | AST side-table 改造推迟到 Phase 4，先冻结新增字段 |
| codegen 热点文件拆分过早 | review 成本高 | 先做 Phase 0-4 正确性与边界修复，最后再拆热点文件 |

---

## 9. 推荐执行顺序

如果只按“性价比最高”的路径推进，建议顺序如下：

1. **Phase 0**：先把 lexer/formatter 的输出合法性修掉
2. **Phase 2**：马上切掉 `parser -> analyzer` 反向依赖
3. **Phase 3**：修 analyzer incremental/delete/caching 的真实 correctness 问题
4. **Phase 1**：补齐 trivia/comment/literal 契约
5. **Phase 4/5**：再做更大范围的共享抽象与 codegen 边界收敛
6. **Phase 6**：把回归网补齐并接入常规开发流程

如果资源只够做一个最小闭环，建议优先完成：

- FE-01
- FE-02
- FE-03
- FE-06
- FE-05

因为这五项分别覆盖：

- 词法位置信息
- formatter 输出合法性
- formatter 语义保持
- parser/analyzer 反向耦合
- analyzer 内存/删除正确性

---

## 10. 当前结论

`src/frontend` 的主要问题不是“某一个文件有几个 bug”，而是已经出现了典型的**多阶段共享结构膨胀 + 模块边界变粗 + 契约漂移**：

- lexer 的位置/注释模型没有完全闭环
- formatter 目前已经触碰到“输出非法源码”的硬 correctness 问题
- parser 对 analyzer 的反向依赖破坏了前端内部层次
- analyzer 的 incremental 名称与实际能力不一致，且删除路径没有闭环
- codegen 已开始出现对粗粒度 parser/public API 的不必要依赖

好消息是：

- 这些问题现在仍然**集中且可切分**
- 当前还没有单个 `.c` 文件越过 3000 行不可控红线
- 只要按本文阶段推进，`src/frontend` 仍然可以在不做一次性大爆破重写的前提下，逐步回到清晰、可信、可测试的状态
