# Frontend 重构计划（`src/frontend/`）

**开发原则**：
- ⚡ **不考虑向后兼容**，Xray 无外部用户，直接采用最佳设计
- ✅ 避免临时 workaround 与兼容层，每一步落到"长期最优"
- ✅ 每个阶段结束必须 **`scripts/run_regression_tests.sh` 全绿** 才能合并
- ✅ 每阶段 commit 粒度控制在 1~3 次（功能性子步骤可拆 commit，但不留半成品状态）

**当前基线**（2026-04-19）：
- 总规模：108 文件，~51,900 行
- 超限文件：7 个 `.c` 超过 1600 行（最大 `xanalyzer_visitor.c` 2538 行）
- 超限头文件：`.h` 导出超过 25 的有 10 个（最多 `xast_api.h` 105 个）
- 明确漏洞：Parser 数组/字符串绕过 arena、`xast.c` 11 处 `exit(1)`、`break_jumps[256]` 静默截断、builtin 双份注册

---

## 阶段总览

| # | 阶段 | 文件范围 | 风险 | 预估工时 | 关键收益 |
|---|------|---------|------|---------|---------|
| P1 | Parser arena 统一化 | `parser/xast.c` + 所有 `xparse_*.c` | 中 | 1.5 d | 消除内存不一致、AST 整段 drop |
| P2 | 死代码与错误退出清理 | `xstmt.c`、`xast.c`、`xexpr.c` | 低 | 0.5 d | 删 ~180 行 dead code，LSP 稳定性 |
| P3 | Parser/Analyzer Builtin 单一真相源 | `analyzer/`、`codegen/xexpr_call.c` | 中 | 1 d | 消除双份注册漂移 |
| P4 | 大文件拆分 | `xast.c`、`xanalyzer_visitor.c`、`xcompiler.c` | 低 | 1.5 d | 每文件 ≤ 目标 2000 行 |
| P5 | 头文件收敛 + 内部结构封装 | `xcompiler.h`、`xast_api.h`、`xparse.h` | 中 | 1.5 d | `.h` 导出 ≤ 25 / 单元 |
| P6 | 固定上限数组消除 | `xcompiler.h`、`xparse_internal.h` | 低 | 0.5 d | 修复 256 截断、realloc 泄漏 |
| P7 | Analyzer 字符串 dispatch 替换为 intern 指针 | `xanalyzer_visitor_expr.c` 等 | 低 | 0.5 d | 每 keystroke 热路径提速 |
| P8 | 真正的函数级增量分析 | `analyzer/xanalyzer_incremental.c` + visitor | 高 | 2 d | LSP 大项目体验 |
| P9 | `xdiag_fmt` 拆为 `.c` | `frontend/xdiag_fmt.h` | 低 | 0.25 d | 二进制瘦身 |

**总计**：9 天有效开发时间。P1→P2→P3→P4→P5 为"骨架重构"链（串行），P6/P7/P9 可任意顺序，P8 单独推进。

---

## P1：Parser Arena 统一化

### 动机
目前 arena 只覆盖 AST 节点本身（`alloc_node`），节点内数组/字符串仍走 `xr_malloc` / `strdup`：

```@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/parser/xast.c:475-509
    size_t len = strlen(item_name);
    char *name_copy = (char*)xr_malloc(len + 1);
    ...
    char *key_copy = (char*)xr_malloc(key_len + 1);
```

导致：
- `xr_ast_free` 必须存在并深度 walk（`xast.c` 1700 行中超过一半是 free 分支）
- `default` 分支裸泄漏：`@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/parser/xast.c:1708-1712`
- arena 的 "一次释放 O(1)" 优势丧失

### 目标设计

**规则**：parse 阶段**所有分配走 arena**。`xr_ast_free` 彻底删除。

```c
// src/base/xarena.h (已存在)
XR_FUNC void *xr_arena_alloc(XrArena *arena, size_t size);
XR_FUNC char *xr_arena_strdup(XrArena *arena, const char *s);

// 新增 helper（注入 xast.c 顶部，替换所有 xr_malloc）
static void *ast_alloc(XrayIsolate *X, size_t size);
static void *ast_alloc_array(XrayIsolate *X, size_t elem_size, int count);
// ast_strdup 已存在

// 约束：get_arena(X) 返回非 NULL 时走 arena；否则 panic（不再 fallback）
```

**废止 `xr_ast_free`**：单文件删除，AST 生命周期绑定 arena。调用者（parser/LSP）只负责 arena lifetime。

### 实施步骤

1. **把 arena 设为强制路径**
   - 修改 `@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/parser/xast.c:33-61` 的 `alloc_node` / `ast_strdup`：去掉 `if (arena)` 分支，arena 为 NULL 时 `XR_CHECK(arena != NULL, "parser requires arena")`
   - 所有 parser 入口（`xr_parse*`）保证 arena 已设置（检查 `xr_parser_init` 当前已走 arena 传参，只需让调用方填充）
2. **替换所有 `xr_malloc` 为 `ast_alloc`**
   - `@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/parser/xast.c` 共 ~30 处 `xr_malloc` 全部迁移
   - `xparse_decl.c` (109 处)、`xparse_oop.c` (92 处)、`xparse_import.c` (34 处) 等同步清理
   - 把 `strdup` 全部换成 `ast_strdup`
3. **废止 `XR_PARSE_PUSH` 宏的 realloc 赋值**
   - 见 P6 节；P1 阶段先改为 arena 版本：`arr = xr_arena_realloc(arena, arr, old, new)`，或更换为 arena grow 语义
4. **删除 `xr_ast_free`**
   - 删除 `@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/parser/xast.c:1511-1716`（整个 free 家族）
   - 删除 `@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/parser/xast_api.h:245` 的声明
   - 全仓 grep `xr_ast_free` 删除调用，替换为 arena drop
   - `xr_param_node_free` / `xr_pattern_free` 同步删除
5. **修正 `exit(1)`**
   - `@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/parser/xast.c` 11 处 `exit(1)` 改为 `XR_CHECK` 或 panic（因为 arena alloc 失败已由 arena 自己 longjmp）

### 验证
- `ctest --output-on-failure` 全绿
- ASAN 构建跑一遍：`/build-asan` workflow
- 对比前后 `valgrind --tool=massif` 峰值内存：大型 `.xr` 文件 parse 阶段应显著下降

### 风险与应对
- **LSP 复用 parser 多次**：每次 parse 前新建 arena；parse 失败时 arena 仍然正常 drop（arena 无泄漏）
- **增量分析需要 AST 存活**：analyzer 仍持有 AST 指针，arena lifetime 要覆盖 analyzer pass。现状已如此，无变更
- **Mono pass 生成新节点**：`xa_mono_pass` 生成的节点走同一 arena，确保 arena 在整个 compile 结束才 drop

---

## P2：死代码与错误退出清理

### 动机

1. **`compile_statement` 是死代码**
   ```@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/codegen/xstmt.c:24-166
   void compile_statement(XrCompilerContext *ctx, XrCompiler *c, AstNode *node) { ... }
   ```
   全仓 0 处调用。真实 dispatcher 是 `xr_compile_statement`（`xcompiler.c:885`）。

2. **`fprintf(stderr, ...)` 绕过诊断系统**
   前端 13 处裸 stderr 输出，LSP 场景完全看不到。

3. **`exit(1)` 违反错误处理规范**
   `xast.c` 11 处（P1 已清理），分析器 0 处，codegen 需复查。

### 目标设计

- 删除 `xstmt.c::compile_statement` 及关联声明
- `fprintf(stderr)` 统一替换为 `xr_compiler_error` / `xa_analyzer_add_diagnostic` / `xr_diag_print`
- `assert` 使用 `XR_DCHECK` / `XR_CHECK`

### 实施步骤

1. **删除 compile_statement 死代码**
   - 删除 `@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/codegen/xstmt.c:24-166`（整个函数）
   - 删除 `@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/codegen/xstmt.h` 中对应声明
   - 保留 `xstmt.c` 文件（因为 stmt helper 还在）但只留共享 helper（若最终为空则删整个文件）
2. **`fprintf(stderr)` 替换**
   - `@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/codegen/xexpr.c:541` → `xr_compiler_error(ctx, c, "unknown expression type: %d", node->type)`
   - 13 处逐一整改，见 `grep -n "fprintf(stderr" src/frontend/**/*.c`
3. **`exit(1)` 清查**（如果 P1 没清完）
   - `grep -n "exit(" src/frontend/` 应只剩 0 个

### 验证
- `ctest` 绿
- LSP stdout-only 模式下（`app/lsp`）跑一遍包含错误的测试用例，确认错误走 JSON-RPC 而非 stderr

---

## P3：Parser/Analyzer Builtin 单一真相源

### 动机

静态分析器与代码生成器各自维护 builtin 列表：

- 分析器：`@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/analyzer/xanalyzer.c:77-167`（~100 行手写 `register_builtin_func`）
- Codegen：`@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/codegen/xexpr_call.c:449-472`（21 条 `BuiltinEntry`）

两边任意一边漏加/类型签名错配都会导致"分析器放行但 codegen 报错"或反之。目前已经有 `xanalyzer_builtins_generated.h`（33 KB），可以扩展模式覆盖更多。

### 目标设计

单一 X-macro 作为唯一真相源：

```c
// src/frontend/analyzer/xbuiltin_table.def
// 格式：XR_BUILTIN(name, kind, sig_desc, codegen_fn_or_NULL)
// kind: FUNC | MODULE | VAR
// sig_desc: 类型签名 DSL（用字符串描述，由统一 parser 构建 XrType）
// codegen_fn: 若 codegen 有快路径特化，填函数名；否则 NULL（走 OP_CALL）

XR_BUILTIN(assert,      FUNC,  "(any, string?) -> void",       compile_builtin_assert)
XR_BUILTIN(assert_eq,   FUNC,  "(any, any, string?) -> void",  compile_builtin_assert_eq)
XR_BUILTIN(int,         FUNC,  "(any) -> int",                 compile_builtin_int)
XR_BUILTIN(Array,       FUNC,  "(...any) -> Array<any>",       compile_builtin_array)
XR_BUILTIN(typeof,      FUNC,  "(any) -> int",                 NULL)
XR_BUILTIN(Json,        MODULE,NULL,                           NULL)
// ...
```

两侧各自 include 同一 `.def`：
- `xanalyzer.c` 通过展开调用 `register_builtin_func(name, parse_sig(sig_desc))`
- `xexpr_call.c` 通过展开生成排序好的 `BuiltinEntry builtin_functions[]`

### 实施步骤

1. **起草 `xbuiltin_table.def`**
   - 把 `xanalyzer.c:77-167` 和 `xexpr_call.c:449-472` 全部迁入
   - 设计"签名 DSL"的 parser（已有 `xr_type_new_function` API 可用，写个极简字符串 parser ~80 行）
2. **分析器侧 X-macro 展开**
   - 删除 `xa_register_codegen_builtins` 手写代码
   - 改为 `#define XR_BUILTIN(...) register_by_def(__VA_ARGS__) ; #include "xbuiltin_table.def" ; #undef XR_BUILTIN`
3. **Codegen 侧 X-macro 展开**
   - 把 `builtin_functions[]` 数组改为从 `.def` 展开（`NULL` codegen_fn 不生成 entry）
   - 保持二分查找；`.def` 顺序可以随意（构建期 qsort 一次，或让 `.def` 保持字典序）
4. **加一个编译期 sanity check**
   - 启动时遍历 builtin table，确认每个 name 的 analyzer / codegen 视图一致（例如 arity 一致）
5. **单元测试**
   - 新增 `tests/unit/test_builtin_table.c`：枚举 builtin，验证双方注册

### 验证
- 回归测试全绿
- `assert_throws`、`assert_ne` 等只在一边注册的情况被 compile-time 检测出来

### 风险
- **签名 DSL 复杂度蔓延**：约束签名只支持 `()` / `<>` / `,` / `?` / `...`，见现有 XrType 能力即可，不加 union/intersection
- **mono/generic builtin**：目前 `Array`/`Map`/`Set` 用 `any`，P3 不扩展，保持现状

---

## P4：大文件拆分

### 动机
单文件 > 2000 行 的有 4 个（P1 完成后 `xast.c` 会因 free 代码删除降到 ~1400 行，但 visitor/call 仍超）。

### 目标布局

```
src/frontend/parser/
├── xast_ctor_literal.c     // literals, binary, unary, grouping
├── xast_ctor_stmt.c        // stmt: if/while/for/return/try
├── xast_ctor_decl.c        // var/fn/class/struct/interface/enum
├── xast_ctor_expr.c        // call/member/index/array/object/map/set
├── xast_ctor_oop.c         // new/this/super/method/field
├── xast_ctor_coro.c        // go/await/channel/select/defer/scope
├── xast_debug.c            // xr_ast_print, xr_ast_typename
└── xast.h (unchanged)

src/frontend/analyzer/
├── xanalyzer_visitor.c                    // 主入口 (< 400 行)
├── xanalyzer_visitor_narrowing.c          // Type narrowing / flow-sensitive
├── xanalyzer_visitor_generic.c            // resolve_class_to_type_param 等
├── xanalyzer_visitor_class_link.c         // class inheritance linking
└── xanalyzer_visitor_internal.h

src/frontend/codegen/
├── xcompiler.c                    // 只保留 xr_compile + dispatcher (< 500 行)
├── xcompiler_block.c              // AST_BLOCK / AST_PROGRAM 两阶段 hoist
├── xcompiler_diag.c               // diagnostics 聚合
└── ...
```

### 实施步骤

1. **xast.c 拆分**（P1 前置完成，free 代码已删）
   - 按 AST 节点类别分组，每组到独立 `.c`
   - 保持 `xast_api.h` 不变（单一入口），所有新 `.c` include 它
   - 公共 helper（`alloc_node` / `ast_strdup` / `ast_alloc_array`）挪到 `xast_internal.h`
2. **xanalyzer_visitor.c 拆分**
   - 目前已有 `xanalyzer_visitor_expr.c` / `xanalyzer_visitor_stmt.c`，但主 `.c` 仍 2538 行
   - 抽出：`resolve_class_to_type_param` 家族 → `xanalyzer_visitor_generic.c`；narrowing 相关 → `xanalyzer_visitor_narrowing.c`；class link → `xanalyzer_visitor_class_link.c`
3. **xcompiler.c 拆分**
   - `xr_compile_statement` 2 个大 case（`AST_BLOCK` / `AST_PROGRAM`）抽取到 `xcompiler_block.c`
   - `xr_compile` 里的 analyzer + diagnostics 流水线抽到 `xcompiler_diag.c`
4. **CMakeLists 更新**
   - 每阶段单独 commit，`git log --stat` 可审
5. **目标验证**
   - `find src/frontend -name "*.c" | xargs wc -l | awk '$1 > 2000'` 为空

### 风险
- **拆分产生的额外 include 导致构建时间上升**：每个 `.c` 只 include 自己需要的头；头文件本身不变
- **static helper 需要变为 XR_FUNC**：可能违反"static 函数比例 ≥ 90%"。做法是把 helper 放 `xast_internal.h` 或用 `XR_FUNC` 标注（internal 模块间函数是允许的 `XR_FUNC`）

---

## P5：头文件收敛 + 内部结构封装

### 动机
- `xast_api.h` 105 个导出；`xparse.h` 77 个；`xcompiler.h` 39 个
- `XrCompiler` 结构体完全暴露，改一个字段触发 codegen 整模块重编
- 头文件互相 include 增加了耦合

### 目标设计

#### 5.1 `xast_api.h` 拆分为分类头
```
xast_api.h                 (仅 include 下面这些)
├── xast_api_literal.h     // 字面量工厂
├── xast_api_binary.h      // 二元表达式
├── xast_api_decl.h        // 声明
├── xast_api_stmt.h        // 语句
├── xast_api_oop.h         // OOP 节点
├── xast_api_coro.h        // 协程节点
└── xast_api_pattern.h     // 析构模式
```
每个子头 ≤ 25 个导出；用户仍只 include `xast_api.h`。

#### 5.2 `XrCompiler` 内部化
```c
// xcompiler.h（对外）
typedef struct XrCompiler XrCompiler;  // opaque
XR_FUNC XrCompiler *xr_compiler_new(XrCompilerContext *ctx, XrFunctionType type);
XR_FUNC void xr_compiler_free(XrCompiler *);
XR_FUNC XrProto *xr_compiler_end(...);
XR_FUNC void xr_compile_statement(...);
// ... 只保留真正的"公共 API"

// xcompiler_internal.h（codegen 内部）
struct XrCompiler { ... 所有字段 ... };
XR_FUNC XRegAlloc *xr_compiler_get_regalloc(XrCompiler *c);
XR_FUNC XrEmitter *xr_compiler_get_emitter(XrCompiler *c);
// ... 足够的 accessor
```

stmt/expr 子模块改为 include `xcompiler_internal.h`。

#### 5.3 `xparse.h` 拆分
- 纯 API（`xr_parse` / `xr_parser_init` 等，~10 个）留在 `xparse.h`
- 其余 parse 函数（`xr_parse_binary` / `xr_parse_class_declaration` 等）全部迁入 `xparse_internal.h`（本来就有）

### 实施步骤

1. 拆 `xast_api.h`，让每个 `.h` ≤ 25 导出
2. 新建 `xcompiler_internal.h`，所有 codegen `.c` 统一 include 它
3. 把 `XrCompiler` 结构体从 `xcompiler.h` 移到 internal
4. 外层 `xcompiler.h` 保留 API 标注为"public"
5. 全仓 `grep -l "struct XrCompiler {"` 应只在 internal 头出现

### 验证
- `ctest` 绿
- 对比前后 `build.ninja -t compdb` 的依赖深度

### 风险
- **access helper 过多**：只暴露真正需要跨文件访问的（regalloc、emitter、proto），属于状态而非字段的直接读写

---

## P6：固定上限数组消除

### 动机

```@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/codegen/xstmt.c:95-98
int jump_pos = emit_jump(c->emitter, OP_JMP);
if (c->break_count < 256) {
    c->break_jumps[c->break_count++] = jump_pos;
}
```
第 257 个 break 静默丢失，生成错误字节码。同类问题：
- `break_jumps[256]` / `continue_jumps[256]`（`xcompiler.h:145-148`）
- `upvalues[UINT8_MAX]`（合理，VM 受限，但超出应报错而非 UB）
- `prescan_captured[256]`（同上）
- `block_stack[64]` → 已有 `XR_MAX_BLOCK_DEPTH` 宏，越界会越界写

此外 `XR_PARSE_PUSH` 宏违反"xr_realloc 必须中转"规则：

```@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/parser/xparse_internal.h:27-33
#define XR_PARSE_PUSH(arr, count, cap, item) do { \
    if ((count) >= (cap)) { \
        (cap) = (cap) == 0 ? 4 : (cap) * 2; \
        (arr) = xr_realloc((arr), sizeof(*(arr)) * (cap)); \
    } \
    ...
```

### 目标设计

- `break_jumps` / `continue_jumps` 改为 arena-growable 数组（`XrArenaVec<int>`）
- `prescan_captured` 改为 arena vec
- `XR_PARSE_PUSH` 宏改为调用 arena grow helper，不再 realloc
- `upvalues` 保持静态（VM 硬限制），但超限时 `xr_compiler_error`

### 实施步骤

1. 新增 `src/base/xarena_vec.h`：`XrArenaVec` 泛型（宏）
   ```c
   #define XR_AVEC_DECL(T) struct { T *data; int count; int cap; }
   #define XR_AVEC_PUSH(arena, vec, item) ...
   ```
2. `XrCompiler.break_jumps/continue_jumps` 改为 `XR_AVEC_DECL(int)`
3. `XR_PARSE_PUSH` 改写用 arena
4. 加 `XR_CHECK(c->local_count < UINT8_MAX, "function too complex")` 等显式 bound

### 验证
- 新增 stress 测试：一个 loop 有 500+ break（用 macro 生成）
- `scripts/static_check.py -c memory` 跑过

---

## P7：Analyzer 字符串 dispatch 替换

### 动机

```@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/analyzer/xanalyzer_visitor_expr.c:1217-1232
if (strcmp(name, "int") == 0)      return XR_KIND_INT;
if (strcmp(name, "float") == 0)    return XR_KIND_FLOAT;
```
每次 member access 走 O(n) `strcmp`。已有 `xr_compile_time_intern` 基础设施。

### 目标设计

- 启动时 intern 所有"特殊名字"（Type.int, Coro.LOW, 关键字等）为单例 `XrString*`
- 热路径比较指针而非 `strcmp`
- 或改为 perfect hash（`gperf` 生成），但 xray 倾向内建实现

### 实施步骤

1. 新增 `src/frontend/analyzer/xanalyzer_names.h`：声明所有 intern 后的 `XrString *g_name_int; g_name_float; ...`
2. 在 `xa_analyzer_new` 中初始化所有单例
3. `xanalyzer_visitor_expr.c` 中所有 `strcmp(name, "xxx")` 改为指针比较

### 验证
- 微基准：10 万次 member access 的 parse+analyze 时间
- 单元测试验证 intern 命中率

---

## P8：真正的函数级增量分析

### 动机

```@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/analyzer/xanalyzer_incremental.c:583-609
    // TODO: Implement true incremental parsing
    ...
    // For now, do full re-analysis but track statistics
    xa_analyzer_update(analyzer, file, (XrAstNode*)ast);
```
依赖图 / 缓存 / 脏传播全建好了，最终仍调全量。

### 目标设计

Function-level 重分析：
1. 每个 `FunctionDeclNode` / `MethodDeclNode` 编译时计算 body hash
2. 改动后，对比新旧 AST 的 body hash：
   - 函数签名未变 + body hash 未变 → **跳过**
   - 函数签名未变 + body hash 变 → 只重跑该函数的 infer pass（symbol 表不动）
   - 签名变 → 标记依赖者 dirty，递归传播
3. Symbol 表支持"增量替换单个函数"：`xa_scope_replace_symbol(scope, name, new_sym)`

### 实施步骤

1. **AST 节点 hash**
   - 给 `FunctionDeclNode` / `MethodDeclNode` 增加 `uint64_t body_hash` 字段
   - 在 parser 生成后计算（或 analyzer collect 时计算）
2. **Diff 算法**
   - `xa_detect_changes` 目前为空 stub（`@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/analyzer/xanalyzer_incremental.c`）
   - 实现：遍历新 AST，按 `(kind, name)` 匹配旧 symbol，hash 不同标记 modified
3. **Symbol 局部替换**
   - `xa_scope_replace_symbol` 实现
   - 清除该 symbol 的依赖边，重建
4. **Visitor 单函数模式**
   - 新增 `xa_visit_infer_function_only(ctx, fn_node)`，独立跑该函数的 infer
   - 跳过全局 Pass 1 collect
5. **LSP 集成**
   - `app/lsp` 的 incremental path 调用 `xa_incremental_update`
   - 统计日志：`skipped_functions` / `incremental_updates` 对比 `full_analyses`
6. **基准**
   - 10k 行 xray 源码 + 修改单个函数 body
   - 目标：re-analyze 时间 < 10 ms（当前全量 ~100 ms）

### 风险
- **跨函数类型推导**：如果 callee 签名变，所有 caller 都要重跑。通过依赖图实现
- **闭包捕获**：upvalue 类型依赖外层作用域。依赖图边用 `XA_DEP_TYPE_USE` 覆盖

---

## P9：`xdiag_fmt.h` 拆为 `.c`

### 动机

`@/Users/xuxinglei/workspace/xray-lang/xray/src/frontend/xdiag_fmt.h:88-182` 整个 `xr_diag_print`（~100 行 + `fprintf` stdio 依赖）是 `static inline`，被 10+ 个 TU include。每个 TU 都实例化一份，浪费二进制。

### 目标设计

```
src/frontend/
├── xdiag_fmt.h   // 仅函数声明 + ANSI 宏（<30 行）
└── xdiag_fmt.c   // 定义
```

### 实施步骤

1. 把 `static inline` 函数体迁到 `xdiag_fmt.c`
2. 头文件只保留：
   - ANSI 色码宏
   - `XrDiagLevel` enum
   - `xr_diag_print` / `xr_diag_print_summary` 声明（`XR_FUNC`）
3. CMakeLists 加入 `xdiag_fmt.c`

### 验证
- `size build/libxray_core.a` 对比前后 text 段
- 预期减少 ~20-40 KB（仅估算）

---

## 落地顺序与合并策略

### 推荐顺序

```
P1 (Arena 化) ─── 前置基础
  ├─ P2 (死代码) ─── 独立
  ├─ P6 (固定数组) ─── 依赖 P1 的 arena vec
  │
  ├─ P3 (Builtin 合并) ─── 独立
  │
  ├─ P4 (大文件拆分) ─── P1 完成后文件已变小，再拆
  │   └─ P5 (头文件收敛) ─── 依赖 P4 的结构
  │
  └─ P7 (字符串 intern) ─── 独立
  └─ P9 (xdiag_fmt) ─── 独立

P8 (增量分析) ─── 最后做，前面降低噪声
```

### 每阶段 checklist 模板

- [ ] 提前写测试（能覆盖到被改动代码的现有测试清单）
- [ ] 实施 + 本地 `ctest`
- [ ] `scripts/check_architecture.sh` 不增加错误
- [ ] `scripts/static_check.py -c memory` 通过
- [ ] `/build-asan` workflow 跑 `tests/asan_smoke.sh`
- [ ] commit 信息遵守 `git` 规则（英文、用 `-F /tmp/xray_commit_msg.txt`）
- [ ] 更新本文档"进度"表格

### 进度表（自行勾选）

| 阶段 | 状态 | 完成日期 | 相关 commit |
|-----|------|---------|------------|
| P1  | ☐ 未开始 | - | - |
| P2  | ✓ 完成 | 2026-04-19 | （本次改动） |
| P3  | ☐ 未开始 | - | - |
| P4  | ☐ 未开始 | - | - |
| P5  | ☐ 未开始 | - | - |
| P6  | ☐ 未开始 | - | - |
| P7  | ☐ 未开始 | - | - |
| P8  | ☐ 未开始 | - | - |
| P9  | ☐ 未开始 | - | - |

### P2 落地细节

- 删除 `src/frontend/codegen/xstmt.c`（168 行死代码 `compile_statement`，无调用方）
- 替换 13 处 `fprintf(stderr, ...)` → 诊断系统
  - `xanalyzer_mono.c` × 2 → `xr_log_debug`（保留原 env gate）
  - `xparse.c` × 2 → `xr_log_warning`（LSP 早期失败路径）
  - `xanalyzer_visitor.c` × 1 → 删除整个 `#if 0` 调试块
  - `xcompiler_context.c` × 1 → `xr_diag_print(XR_DIAG_ERROR)` + `ctx->had_error = true`
  - `xexpr.c` × 1 → `xr_compiler_error`
  - `xexpr_binary.c` × 1 → `xr_diag_print(XR_DIAG_WARNING)`
  - `xexpr_literal.c` × 1 → 合并到旁边的 `xr_compiler_error`
  - `xexpr_unary.c` × 1 → 合并到旁边的 `xr_compiler_error`
  - `xoop_class.c` × 1 → `xr_compiler_error`
  - `xstmt_assignment.c` × 1 → `xr_compiler_error`（附 line 临时切换）
  - `xstmt_forin.c` × 1 → `xr_diag_print(XR_DIAG_WARNING)`
- `exit(1)` 残留仅在 `parser/xast.c` 11 处（全部在 `xr_malloc` 失败路径）—— 留到 P1 随 arena 统一化一起消除
- 验证：
  - `ctest` 61/61 通过（Debug + ASAN 双构建）
  - `scripts/run_regression_tests.sh` 274/274 通过
  - `scripts/check_architecture.sh` 7 errors（全部在 VM/JIT/CORO，与 P2 无关），warnings 45 → 43

---

## 参考

- 原始分析：本次会话上文对 `src/frontend` 的问题排查
- 模块依赖规则：`docs/rules/architecture.md`
- 内存规则：`docs/rules/gc-memory.md`
- 开发工作流：`docs/rules/dev-workflow.md`
- 已有重构模板：`docs/module_restructure_plan.md`
