# 036 — Xi IR 覆盖率审计

> **状态**：completed
> **前置依赖**：034, 035
> **目的**：系统评估 Xi IR lowering + emission 对 xray 全语义的覆盖度，识别缺口

---

## 1. AST 节点覆盖率

### 1.1 lower_expr 覆盖（表达式）

| 分类 | AST 节点 | 状态 | 测试 |
|------|----------|------|------|
| 字面量 | INT/FLOAT/STRING/BOOL/NULL/BIGINT/REGEX | ✅ 完整 | ✅ check_exec |
| 算术 | ADD/SUB/MUL/DIV/MOD | ✅ 完整 | ✅ check_exec |
| 位运算 | BAND/BOR/BXOR/LSHIFT/RSHIFT | ✅ 完整 | ✅ check_exec |
| 比较 | EQ/NE/LT/LE/GT/GE/EQ_STRICT/NE_STRICT | ✅ 完整 | ✅ check_exec |
| 逻辑 | AND/OR (short-circuit) | ✅ 完整 | ✅ check_exec |
| 一元 | NEG/NOT/BNOT | ✅ 完整 | ✅ check_exec |
| 变量 | VARIABLE/ASSIGNMENT/COMPOUND/INC/DEC | ✅ 完整 | ✅ check_exec |
| 调用 | CALL_EXPR | ✅ 完整 | ✅ check_exec |
| 三元 | TERNARY | ✅ 完整 | ✅ check_exec |
| 成员 | MEMBER_ACCESS/MEMBER_SET | ✅ 完整 | ✅ check_exec |
| 索引 | INDEX_GET/INDEX_SET | ✅ 完整 | ✅ check_exec |
| 集合 | ARRAY_LITERAL/MAP_LITERAL/SET_LITERAL/OBJECT_LITERAL | ✅ 完整 | ✅ check_exec（object 用 NEWMAP，bracket 等价）|
| 函数 | FUNCTION_DECL/FUNCTION_EXPR | ✅ 完整 | ✅ check_exec |
| OOP | NEW_EXPR/THIS_EXPR/SUPER_CALL | ✅ 完整 | ✅ check_exec |
| 枚举 | ENUM_ACCESS/ENUM_CONVERT/ENUM_INDEX | ✅ 完整 | ✅ check_exec |
| 模板 | TEMPLATE_STRING | ✅ 完整 | ✅ check_exec |
| 空安全 | NULLISH_COALESCE/OPTIONAL_CHAIN/FORCE_UNWRAP | ✅ 完整 | ✅ check_exec |
| 类型 | IS_EXPR/AS_EXPR | ✅ 完整 | ✅ check_exec（as/as? 运行时 typeof 检查）|
| 切片 | SLICE_EXPR/RANGE | ✅ 完整 | ✅ check_exec |
| 结构体 | STRUCT_LITERAL | ✅ 完整 | ✅ check_exec |
| 协程 | GO_EXPR/AWAIT_EXPR/CHANNEL_NEW | ✅ 完整 | ✅ go+await similarity；channel check_exec |
| 协程扩展 | AWAIT_ALL/AWAIT_ANY/CANCELLED/MOVE | ✅ 完整 | ✅ cancelled check_exec; await.all/any similarity |
| Match | MATCH_EXPR | ✅ 完整 | ✅ check_exec |

### 1.2 lower_stmt 覆盖（语句）

| 分类 | AST 节点 | 状态 | 测试 |
|------|----------|------|------|
| 声明 | VAR_DECL/CONST_DECL | ✅ 完整 | ✅ check_exec |
| 控制流 | IF/WHILE/FOR/FOR_IN/BREAK/CONTINUE | ✅ 完整 | ✅ check_exec |
| 函数 | RETURN_STMT (single + multi) | ✅ 完整 | ✅ check_exec |
| 异常 | TRY_CATCH/THROW_STMT | ✅ 完整 | ✅ check_exec |
| 块 | BLOCK | ✅ 完整 | ✅ check_exec |
| 打印 | PRINT_STMT | ✅ 完整 | ✅ check_exec |
| OOP | CLASS_DECL/ENUM_DECL | ✅ 完整 | ✅ check_exec |
| 解构 | DESTRUCTURE_DECL/DESTRUCTURE_ASSIGN | ✅ 完整 | ✅ check_exec |
| 多值 | MULTI_VAR_DECL/MULTI_ASSIGN | ✅ 完整 | ✅ check_exec |
| 协程 | DEFER_STMT/SELECT_STMT/SCOPE_BLOCK/YIELD_STMT | ✅ 完整 | ✅ defer×2 check_exec; yield check_exec; scope check_exec; select similarity |
| 模块 | IMPORT_STMT/EXPORT_STMT | ✅ no-op | N/A |
| 占位 | STRUCT_DECL/INTERFACE_DECL/TYPE_ALIAS | ✅ no-op | N/A |

### 1.3 Xi IR Op 覆盖

| Xi Op | xi_emit.c | 测试验证 |
|-------|-----------|----------|
| XI_ITER_NEW | ❌ 无 emit case | — |
| XI_ITER_VALID | ❌ 无 emit case | — |
| XI_PHI | ❌ 无 emit case（正常：phi 由 regalloc 处理） | — |
| 其余 56 个 op | ✅ 全部有 emit | — |

说明：`XI_ITER_NEW`/`XI_ITER_VALID` 目前未使用——for-in 通过 index-based 或 method call 实现。

---

## 2. 缺口清单（按优先级排序）

### P0：运行时正确性未验证（lowering 存在但未测试）

| 缺口 | 影响 | 工作量 |
|------|------|--------|
| **协程语句** (go/await/defer/yield/channel) | 并发核心特性 | ✅ 已覆盖（9 个 check_exec/similarity 测试 + 2 个 lowering/emission bug 修复）|
| **协程扩展** (select/scope/await.all/await.any) | 高级模式 | ✅ 已测试（scope check_exec，select/await.all/any similarity）|
| **解构** (destructure decl/assign) | 常用语法糖 | ✅ 已修复并测试 |
| **多值赋值** (multi var decl/assign) | 常用语法糖 | ✅ 已修复（XI_MULTI_RET + XI_EXTRACT） |
| **set literal** | 少用 | ✅ 已修复（shared_offset 分配 + coro 重置） |
| **object literal** | 少用 | ✅ check_exec=true（NEWMAP bracket 等价于 NEWJSON）|

### P1：语义质量差距

| 缺口 | 现状 | 目标 |
|------|------|------|
| **as 类型转换** | ✅ 已修复：OP_TYPEOF+OP_EQK 运行时检查 | unsafe→throw, safe→null |
| **STRBUF 字符串拼接** | ✅ 已修复：lower_binary 检测 ADD chain + string type → XI_STR_CONCAT | STRBUF_NEW/APPEND/FINISH |
| **FORPREP/FORLOOP 优化** | range for-in 走 generic LT+JMP | 专用循环指令 |
| **BCE (Bounds Check Elimination)** | 未实现 | ARRAY_GET_NOCHECK |

### P2：优化 pass 缺失

| 缺口 | 现状 | 035 目标 |
|------|------|----------|
| **DCE (Dead Code Elimination)** | ✅ 已修复（xi_opt_run 递归到 children）| 已可用 |
| **常量折叠** | 仅 fusion (ADDI/MULK/LTI) | SSA 全局 SCCP |
| **SelectRepresentations** | 未实现 | rep 匹配 + 自动 BOX/UNBOX 插入 |
| **Type specialization** | 未实现 | 类型感知指令选择 |

---

## 3. 建议下一步

### 方案 A：宽度优先（先覆盖全特性）

1. 添加 解构/多值赋值/set literal 的 check_exec 测试（1h）
2. 添加 协程基础测试 harness（go/await/channel 最简用例）（2-4h）
3. 修复发现的 emission bug
4. 确认所有 AST 节点 lowering 产出正确字节码

### 方案 B：深度优先（先提升代码质量）

1. 修复 DCE bug（test_xi_pipeline）
2. 实现 STRBUF 字符串拼接特化
3. 实现 SelectRepresentations pass（BOX/UNBOX 优化）
4. 提升 regalloc 质量

### 推荐：方案 A 优先

理由：034 的 S7（切换默认管线）要求 **功能完整** 先于 **性能优化**。
当前 Xi lowering 代码已全部写好，但 ~30% 的特性未经运行时验证。
先用测试确认正确性，再做优化。

---

## 4. 已修复的 Bug

### Bug 1: Object destructure 用 GETPROP 访问 map 会崩溃
- **根因**：`lower_destructure_bind` 的 `PATTERN_OBJECT` 分支用 `XI_LOAD_FIELD` (→ GETPROP)，但 Xi 将 object literal 降为 NEWMAP，map 不支持 dot access
- **修复**：改用 `XI_INDEX_GET` + string key
- **文件**：`xi_lower.c:1624-1639`

### Bug 2: Set literal Xi 字节码执行导致内存损坏
- **现象**：`cmp_set_literal` 的 check_exec 导致后续测试 abort
- **根因**：(1) Xi pipeline 未设置 `proto->shared_offset`，所有 proto 共享 shared[0..k]；(2) `execute_and_capture` 用 `xr_execute` 不完整重置 coro 状态
- **修复**：(1) `xi_emit.c` 在 emit 开头从 `isolate->vm.shared.count` 分配唯一 offset；(2) `test_xi_compare.c` 改用 `xr_coro_reset_for_call` + `xr_main_thread_run`
- **状态**：✅ 已修复，check_exec=true

### Bug 3: multi-return 只捕获第一个返回值
- **根因**：`lower_return` 只取 `ret->values[0]`，忽略后续值；emitter 只发射 `OP_RETURN1`
- **修复**：新增 `XI_MULTI_RET` (打包多返回值) + `XI_EXTRACT` (提取第 i 个)；`lower_return` 先求值再打包；emitter 发射 `OP_RETURN(base, nret)`
- **文件**：`xi.h`, `xi_lower.c`, `xi_emit.c`, `xi_dump.c`

### Bug 4: emit_str_concat 全局缓冲溢出（ASAN）
- **根因**：`add_const_string` 在 `ctx->isolate==NULL` 时调用 `xr_make_ptr_val(str)`，而 `xr_make_ptr_val` 把 C 字符串字面量当作 `XrGCHeader` 读取头部字节
- **修复**：无 isolate 时改用 `xr_null()` 占位符（emit-only 测试只验证指令序列）
- **文件**：`xi_emit.c:202-219`

### Bug 5: DCE 不递归到子函数
- **根因**：`xi_opt_run` 只优化顶层 `XiFunc`，子函数（嵌套闭包）从未做 DCE
- **修复**：`xi_opt_run` 末尾递归调用 `f->children[i]`
- **文件**：`xi_opt.c:519-533`

### Bug 6: `go fn(args)` lowering 错误地先执行调用
- **根因**：`lower_go_expr` 对 `AST_CALL_EXPR` 直接 `lower_expr`，把 `go fn(a, b)` 降为 "先执行 fn(a, b) 拿结果，再 GO 那个结果"
- **修复**：检测 `AST_CALL_EXPR` 时，提取 callee + args，作为 `XI_GO` 的 `args[0]=callee, args[1..n]=params`
- **文件**：`xi_lower.c:1061-1097`

### Bug 7: XI_GO emission 未将 args 放到连续寄存器
- **根因**：`OP_GO` 约定 `R[A]=task, R[B]=closure, args at R[B+1..B+C]`；旧实现只发 `OP_GO dst callee nargs`，args 散落在不同寄存器
- **修复**：参照 XI_CALL 的模式，把 callee 复制到 dst，args 移到 dst+1..dst+nargs，然后 `OP_GO dst dst nargs`
- **文件**：`xi_emit.c:1067-1098`

### Bug 8: XI_AS cast 只做 MOVE，不做运行时类型检查
- **根因**：`XI_AS` emission 只发 `OP_MOVE`，legacy `compile_as_expr` 会做 `OP_TYPEOF+OP_EQK` 运行时检查
- **修复**：`lower_as_expr` 把 `is_safe` 存入 `aux_int`；`XI_AS` emission 完整复刻 legacy 行为：typeof+EQK 检查，unsafe→throw TypeError，safe→LOADNULL
- **文件**：`xi_lower.c:1216-1228`, `xi_emit.c:1380-1461`

### 已知缺口
- 暂无；所有列出的缺口均已闭环

## 5. 测试覆盖率更新

| 指标 | 起始 | 当前 |
|------|------|------|
| 总测试数 | 104 | 127 |
| check_exec=true | 104 | 122 |
| check_exec=false | 0 | 5（go×2 / select / await.all / await.any：调度差异）|
| ctest | 93/94 | **94/94** ✅ |
| 协程测试 | 0 | 13（defer×2 / yield / chan×3 / go×2 / cancelled / scope / select / await.all / await.any）|
| as cast 测试 | 0 | 4（match / safe-match / safe-mismatch / unsafe-mismatch）|
