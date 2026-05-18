# 068 — 编译器管线优化方案

> `src/frontend`、`src/ir`、`src/jit` 三个目录的综合改进实施方案。
> 基于两份独立静态审计报告综合而成（2026 年 5 月）。

## 开发原则

**不考虑向后兼容。不妥协。只选最优设计。**

Xray 没有外部用户、没有已发布的 ABI、代码量小。
本方案中每一项都直接瞄准最终正确设计——
不做临时 guard、不做两步迁移、不加兼容层。
如果一个改动需要动 50 个文件，就在一个 commit 里完成。

## 范围

编译器管线三个目录，合计约 85,000 行：

| 目录 | 行数 | 文件数 | TODO/FIXME 标记 |
|------|------|--------|----------------|
| `src/frontend` | ~33,857 | ~50 | 30（14 个文件） |
| `src/ir` | ~12,237 | ~30 | 11（6 个文件） |
| `src/jit` | ~38,905 | ~40 | 75（20 个文件） |

## 组织方式

工作分为 **5 个 Track**，可独立调度：

- **Track A** — 正确性与线程安全（P0）
- **Track B** — 架构与拆分重设计（P0–P1）
- **Track C** — 验证器与健壮性加固（P1）
- **Track D** — 后端对等与调试基础设施（P1）
- **Track E** — 性能与开发体验（P1–P2）

---

# Track A — 正确性与线程安全

## A1. `is_jit_eligible()` 在后台线程写 proto

**风险**：高 — `proto->return_type_info` 存在数据竞争。

**问题**：`xm_eligibility.c:is_jit_eligible()` 在后台编译 worker 中作为
只读谓词被调用，但当反馈稳定时会有条件地写入 `proto->return_type_info`。
这违反了"后台 worker 只读 proto、不写"的线程安全契约。

**涉及文件**：`src/jit/xm_eligibility.c`、`src/jit/xjit_compile_queue.c`

**修复**：
1. 将 `return_type_info` 提升逻辑从 `is_jit_eligible()` 中提取出来。
2. 将提升操作移到主线程的入队路径（`xjit_enqueue()` 或快照构建阶段），
   该阶段允许写 proto。
3. 使 `is_jit_eligible()` 成为纯谓词（无副作用）。
4. 在后台编译入口处添加 `XR_DCHECK` 确认没有发生 proto 写入。

**验证**：
- `ctest --output-on-failure` — 无回归。
- 人工审查：grep 后台编译路径中的 `proto->` 写入。

---

## A2. Inline pass 多返回值误编译

**风险**：中等 — 对多返回值函数的静默误编译。

**问题**：`xi_opt_inline.c` 内联被调用者时仅使用第一个返回值。在
`XI_CALL` 支持多返回值且 `XI_EXTRACT` 提取元组元素的情况下，
会产生不正确的代码。

**涉及文件**：`src/ir/xi_opt_inline.c`

**修复** — 直接实现完整的多返回值 inline（不加临时 guard）：
1. 内联展开时，扫描调用者中对被内联 `XI_CALL` 的
   `XI_EXTRACT(call_result, index)` 使用点。
2. 将每个 `XI_EXTRACT` 映射到内联体中 `XI_RETURN_MULTI` 终结器的对应返回值。
3. 如果被调用者有多个带 `XI_RETURN_MULTI` 的 return block，通过 phi 节点
   合并（与单返回值 inline 相同做法）。
4. 替换所有 `XI_EXTRACT` 使用点，然后删除原始 `XI_CALL`。

**测试**：添加 `let a, b = pair()` 测试，其中 `pair` 足够小可被内联。
验证 VM 和 JIT 均产生正确值。

**验证**：`ctest` + `scripts/run_regression_tests.sh`

---

## A3. 不支持的 lowering 必须是硬编译错误

**风险**：中等 — 未 lower 的 AST 节点导致运行时意外行为。

**问题**：`xi_lower.c` 中有两处默认回退：
- `xi_lower.c:2163` — 不支持的表达式 → 发射 null 占位符。
- `xi_lower.c:2661` — 不支持的语句 → 静默跳过。

两者都静默产生不正确的运行时行为，而不是编译错误。

**涉及文件**：`src/ir/xi_lower.c`

**修复** — 硬错误，无回退：
1. 将两个 default 分支替换为 `XR_UNREACHABLE("unsupported AST
   node kind %d", node->type)`。Debug 构建下立即 abort。
2. Release 构建下设置 `ctx->error = true` 并返回 `XI_NOP`。
3. `xi_pipeline` 已有 lowering 错误检查——只需确保标志正确传播。
   无需新的 pipeline 错误基础设施。

分析器接受的每个 AST 节点都必须可 lower。如果不能 lower，
那是编译器 bug，不是用户错误。

**测试**：验证所有现有测试语料库编译时不触发新断言。
不需要"不支持节点的负面测试"——不应存在合法的不支持节点。

**验证**：`ctest` + `scripts/run_regression_tests.sh`

---

## A4. Deopt 快照：基于 liveness 并使用哈希索引

**风险**：中等 — guard deopt 可能恢复不正确的值。

**问题**：`xi_to_xm.c::record_deopt()` 通过扫描整个 `slot_map` 构建
deopt 快照，而非使用 guard 点处的实际 liveness。其他问题：
- `XM_MAX_DEOPT_SLOTS` = 32，溢出被静默截断。
- `slot_map_bc_pc()` 每次调用都是线性扫描。
- 未知 TAGGED rep 的处理过于粗糙。

**涉及文件**：`src/jit/xi_to_xm.c`

**修复** — 一次性解决三个问题：
1. **哈希索引**：在 `xi_to_xm_lower()` 初始化时构建
   `value_id → slot_map_entry` 哈希表。每个 guard 点 O(1) 查找。
2. **溢出 = 强制 deopt**：当 deopt slots 超过 `XM_MAX_DEOPT_SLOTS` 时，
   发射 `XM_GUARD_ALWAYS_FAIL`（放弃优化路径）。不静默截断。
3. **基于 liveness 的快照**：Xm SSA 构建完成后，将 slot map 与每个
   guard 点的 SSA liveness 集合求交集。只包含在 guard 处实际存活的值，
   而非整个 slot map。

三个修改小且紧密耦合，没有理由分步实施。

**验证**：带 deopt 触发的 JIT E2E 测试 + GC 压力测试。

---

# Track B — 架构与拆分重设计

## B1. 重新设计 `xi_emit.c`（2290 行，`emit_value()` 1129 行）

**优先级**：P0 — 项目中最大的函数。

该文件同时处理 VM 寄存器分配、liveness、RPO 线性化、跳转 patching、
try/finally、闭包 proto 发射、class 描述符、string/常量、IC value_pc、
slot map、模块导出和指令融合。

**设计变更**：用**表驱动分发**替换 1129 行的 `emit_value()` switch：

```c
typedef void (*XiEmitHandler)(EmitCtx *ctx, XiValue *v);
static const XiEmitHandler emit_handlers[XI_OP_COUNT] = {
    [XI_ADD]         = emit_arith,
    [XI_CALL]        = emit_call,
    [XI_CLASS_CREATE]= emit_class_create,
    ...
};
```

每个 handler 是位于领域专属文件中的小型 static 函数。
分发变为 `emit_handlers[v->op](ctx, v)`。

**目标拆分**：

| 新文件 | 职责 | 预估行数 |
|--------|------|----------|
| `xi_emit.c` | 驱动器、EmitCtx、表分发、`xi_emit()` 入口 | ~300 |
| `xi_emit_reg.c` | 寄存器分配、liveness、free-list | ~300 |
| `xi_emit_cf.c` | Block 线性化、分支、phi move、patch | ~300 |
| `xi_emit_arith.c` | 算术、比较、逻辑、位运算 | ~300 |
| `xi_emit_call.c` | 调用/方法/内置/多返回值发射 | ~250 |
| `xi_emit_object.c` | Class/enum/容器/字段描述符 | ~250 |
| `xi_emit_eh.c` | Try/catch/finally/defer | ~200 |
| `xi_emit_slotmap.c` | value_pc、bc_slot、IC/deopt 元数据 | ~150 |

**共享头文件**：`xi_emit_internal.h` — `EmitCtx` 结构体、寄存器辅助函数、
常量/字符串池辅助函数、handler 类型定义。所有子文件 include 此头文件
并注册各自的 handler。

**验证**：每个子步骤后 `ctest` + `scripts/run_regression_tests.sh`
（纯重构，零语义变更）。

---

## B2. 重新设计 `xm_codegen_x64.c`（3582 行，`x64_emit_xm_ins()` 1395 行）

**优先级**：P0 — 项目中第二大函数。

与 B1 相同思路：对 `x64_emit_xm_ins()` 使用**表驱动分发**。

**目标拆分**：

| 新文件 | 职责 | 预估行数 |
|--------|------|----------|
| `xm_codegen_x64.c` | 入口、prologue/epilogue、frame patch | ~500 |
| `xm_codegen_x64_ins.c` | 表分发 + 算术/逻辑/move | ~500 |
| `xm_codegen_x64_call.c` | （已有）调用/deopt bridge | ~850 |
| `xm_codegen_x64_mem.c` | 内联 alloc、写屏障、GC | ~400 |
| `xm_codegen_x64_stub.c` | call_c_stub、barrier stubs、deopt stub | ~400 |
| `xm_codegen_x64_osr.c` | OSR stubs、resume entry | ~400 |
| `xm_codegen_x64_patch.c` | 分支 patching、block 偏移解析 | ~300 |

**共享头文件**：`xm_codegen_x64_internal.h` — `X64CodegenCtx`、编码器宏、
寄存器映射、patch 结构体。

**强制清理**（同一 commit）：
- 提取 `x64_ctx_cleanup()` 替换 4 处重复的 `xr_free()` 序列
  （bail jmp、regalloc error、alloc error、codegen error）。
- 通过 `goto cleanup` 或类 RAII 风格实现单一清理路径。

**验证**：`ctest` + x64 主机上的 JIT E2E 测试。

---

## B3. 拆分 `xi_lower.c`（2935 行，接近 3000 行上限）

已有 `xi_lower_stmt.c` 和 `xi_lower_class.inc.c`，但主文件仍承载过多职责。

**目标拆分**：

| 新文件 | 职责 |
|--------|------|
| `xi_lower.c` | Braun SSA 核心、作用域、函数入口、驱动器 |
| `xi_lower_stmt.c` | （已有）语句 lowering |
| `xi_lower_expr.c` | 二元/一元/调用/成员/索引/三元 |
| `xi_lower_closure.c` | 捕获/upvalue/共享变量 |
| `xi_lower_import.c` | 模块 import/export |
| `xi_lower_enum.c` | 枚举编译期构造 |
| `xi_lower_coro.c` | go/await/scope/select/channel |
| `xi_lower_eh.c` | try/catch/finally/defer/throw |

**共享头文件**：`xi_lower_internal.h`（已存在，按需扩展）。

**验证**：`ctest` + `127/127 Xi Compare` + 回归测试。

---

## B4. 拆分 `xm_jit.c`（1232 行，重度耦合）

该文件 include 了 20+ 个 runtime 头文件，混合了生命周期管理、反馈收集、
编译、安装和桥接等职责。

**目标拆分**：

| 新文件 | 职责 |
|--------|------|
| `xm_jit.c` | Init/destroy/stats、公开 API 表面 |
| `xm_jit_driver.c` | 编译决策、pipeline 编排 |
| `xm_jit_install.c` | 发布/安装代码、内存序 |
| `xm_jit_bridge.c` | 解释器 ↔ JIT 调用/deopt 桥接 |
| `xm_jit_feedback.c` | TFA/类型反馈/shape/IC 快照 |

**验证**：`ctest` + JIT E2E 测试。

---

## B5. 拆分 ARM64 `xm_codegen.c`（2779 行）

与 x64 采用相同模式以保持对等：

| 新文件 | 职责 |
|--------|------|
| `xm_codegen.c` | 入口、prologue/epilogue、frame patch |
| `xm_codegen_ins.c` | `emit_xm_ins()` opcode 分发 |
| `xm_codegen_call.c` | （已有）调用 ops |
| `xm_codegen_mem.c` | （已有）alloc/写屏障 |
| `xm_codegen_stub.c` | call_c_stub、barrier stubs、deopt stub |
| `xm_codegen_osr.c` | OSR stubs、resume entry |

**验证**：`ctest` + ARM64 主机上的 JIT E2E 测试。

---

# Track C — 验证器与健壮性加固

## C1. 将 `xi_verify.c` 升级为语义验证器（无条件启用）

**现状**：仅做结构检查（函数/block/value/phi 结构、CFG 一致性、
type 非 NULL、op 范围）。

**缺失的验证项**：

| 检查 | 描述 |
|------|------|
| 支配关系 | 每个 use 必须被其 def 支配（phi 参数由前驱块支配） |
| 操作数数量 | 每个 `XiOp` 有已知参数数量，与 `narg` 做校验 |
| 类型契约 | 比较 → bool、`XI_SELECT` 两臂类型兼容、`XI_BOX`/`XI_UNBOX` rep 合法、`XI_CALL` 多返回值 ↔ `XI_EXTRACT` 范围 |
| 副作用标志 | call/throw/store/alloc/safepoint 必须有 `XI_FLAG_SIDE_EFFECT` |
| 后端契约 | VM 可发射 op 子集、JIT 可 lower 的 op 子集、AOT 可 cgen 的 op 子集 |

**涉及文件**：`src/ir/xi_verify.c`、`src/ir/xi_verify.h`

**实现**：
1. 添加 `xi_verify_dominance()` — 遍历 RPO，检查每个 value 的参数。
2. 添加 `xi_verify_op_arity()` — 静态表 `uint8_t expected_narg[XI_OP_COUNT]`。
3. 添加 `xi_verify_types()` — 逐 op 类型契约检查。
4. 添加 `xi_verify_flags()` — 副作用标志一致性。

**不设门控标志**。所有检查无条件运行。如果现有 IR 生成器产生不合法 IR，
修复生成器——这正是本项的价值所在。没有外部用户，可以立即破坏并修复。

**验证**：修复新检查暴露的任何 IR 生成器 bug 后，所有现有测试必须通过。
为每个新检查类别添加针对性的非法 IR 测试。

---

## C2. Xi pipeline 编译预算与统计

**问题**：Xm pass 驱动器有 `XmCompileBudget`，但 Xi pipeline 只有
max-rounds。无计时、无逐 pass 统计、无法诊断编译时间回归。

**涉及文件**：`src/ir/xi_opt.c`、`src/ir/xi_pass.h`、`src/ir/xi_pipeline.c`

**实现**：
1. 添加 `XiPassStats` 结构体：pass 名称、调用次数、values
   增删数量、耗时（微秒）。
2. 添加 `XiPipelineStats`：逐 pass 数组、总轮次、总时间。
3. 在 `xi_opt_run_pipeline()` 中填充统计数据。
4. JIT Tier 1 使用紧凑预算（如 `XI_OPT_LIGHT` + 5ms 上限）。
5. AOT 无预算限制（完整优化）。
6. 可选 dump：`XRAY_XI_STATS=1` 环境变量打印逐函数统计。

**验证**：`ctest`（无行为变更，统计信息仅供参考）。

---

# Track D — 后端对等与调试基础设施

## D1. 将 JIT 调试基础设施移植到 x64

**问题**：`xm_jit_debug.c` 整体被 `#ifdef __aarch64__` 门控，
x64 只有空 stub。但 crash handler 内部已有 `__x86_64__` 分支
（位于 `#ifdef __aarch64__` 块内，属于死代码）。

**涉及文件**：`src/jit/xm_jit_debug.c`、`src/jit/xm_jit_debug.h`

**修复**：
1. 移除外层 `#ifdef __aarch64__` 门控。
2. 使代码区域注册表（`g_regions[]`、register/lookup/dump）平台无关。
3. guard-page safepoint 保持在 `#ifdef __aarch64__` 下（使用 x28 寄存器，
   ARM64 专有机制）。
4. 信号处理器：已有 `__x86_64__` fault_pc 提取——只需使其可达。
5. 反汇编辅助：条件调用 ARM64 或 x64 disasm。
6. 将头文件中的 `#else` stub 替换为实际声明。

**线程安全**（强制，同一 commit）：
- 使 `g_regions[]` 访问原子化或使用简单自旋锁，因为后台编译线程会
  并发调用 `jit_debug_register()`。
- 添加溢出检查：`XR_DCHECK(g_nregions < JIT_DEBUG_MAX_REGIONS)`。

**验证**：ARM64 和 x64 上均运行 `ctest`。手动测试：在 x64 上触发
JIT crash，验证 crash handler 打印代码区域 + 反汇编。

---

## D2. 解决 x64 frame smap ptr TODO

**问题**：`xm_codegen_x64_call.c:305` 有 `/* TODO: frame smap ptr slot */`。
这影响 x64 上 JIT→JIT 调用的 GC stack map 正确性。

**涉及文件**：`src/jit/xm_codegen_x64_call.c`

**修复**：
1. 研究 ARM64 实现中 `FRAME_SMAP_PTR_OFFSET` 的处理方式。
2. 在 x64 deopt return 路径中实现等价逻辑。
3. 与 ARM64 对齐：frame smap ptr 存储、调用后 active stack map 恢复、
   嵌套 JIT 调用的 deopt 传播。

**需添加的测试**：
- JIT→JIT 调用后接 GC safepoint。
- JIT→JIT 调用后接 deopt。
- spill + call + deopt 组合场景。

**验证**：`ctest` + x64 上带 GC 压力的 JIT E2E 测试。

---

## D3. 后端契约测试

**问题**：ARM64 和 x64 后端独立实现相同契约（入口约定、返回协议、
stack map、deopt、OSR），没有共享规范的强制执行。漂移不可避免。

**实现**：
定义并通过双后端 E2E 测试验证以下契约：

| 契约 | 描述 |
|------|------|
| Entry / fast_entry / resume_entry | 参数传递、栈帧建立 |
| Return payload/tag | 返回值和类型标签的存放位置 |
| Stack map active 指针 | 跨调用如何保存/恢复 active smap |
| Deopt spill save | 为 deopt 重建保存哪些寄存器 |
| OSR materialization | 解释器状态如何映射到 JIT 寄存器 |
| JIT→JIT call smap restore | 跨 JIT 调用的 stack map 指针管理 |

**涉及文件**：`tests/unit/jit/test_jit_backend_contract.c`（新建）

每个测试用两个后端编译一个小函数（若可交叉编译），至少验证宿主后端。
测试检查生成代码的结构属性（入口偏移、smap 元数据、deopt info），
而非仅检查执行输出。

**验证**：`ctest`（新测试套件）。

---

# Track E — 性能与开发体验

## E1. 编译期字符串池

**问题**：`xast.c:105` — `// TODO: Implement compile-time string pool`。
相同的字符串字面量被分别分配，大源文件中浪费内存。

**涉及文件**：`src/frontend/parser/xast.c`

**实现**：
1. 在 `XrayIsolate` 中添加哈希表或专用 `XrStringPool` 结构体，
   挂载到编译上下文。
2. `xr_ast_string_literal()` 先查池；命中返回已有指针，未命中则插入。
3. 池生命周期 = 编译单元生命周期（编译结束时释放）。

**影响**：减少大文件中重复字符串字面量的内存占用。
Parser 和 analyzer 对相同字符串看到相同指针，
从而支持更廉价的指针比较。

**验证**：`ctest` + 对大 .xr 文件做内存分析。

---

## E2. Analyzer visitor 拆分

**问题**：`xanalyzer_visitor.c`（1634 行）、`xanalyzer_visitor_expr.c`
（1576 行）、`xanalyzer_visitor_decl.c`（1347 行）混合了多个语义领域，
维护困难。

**按语义领域目标拆分**：

| 新文件 | 领域 |
|--------|------|
| `xanalyzer_visitor_class.c` | Class/interface/struct 解析 |
| `xanalyzer_visitor_enum.c` | 枚举类型和值解析 |
| `xanalyzer_visitor_nullable.c` | Nullable/optional chain 窄化 |
| `xanalyzer_visitor_collection.c` | 集合/迭代器类型推导 |
| `xanalyzer_visitor_builtin.c` | 内置/成员解析 |

**验证**：`ctest`（纯重构，零语义变更）。

---

## E3. 内置 API 单一真相源

**问题**：内置类型签名分散在多处：
- `xanalyzer_builtins.c` + `xanalyzer_builtins_generated.h`（analyzer）
- 运行时方法表（各 stdlib `*.c` 文件）
- LSP 补全（消费 analyzer 数据）
- Formatter/类型打印器

这些可能产生漂移。已通过自动生成 `xanalyzer_builtins_generated.h`
部分解决，但运行时方法表仍然独立维护。

**实现**：
1. 定义规范的内置元数据格式（TOML/JSON 或 `.def` 文件）。
2. 从中生成：
   - `xanalyzer_builtins_generated.h`（已存在，标准化输入源）
   - 运行时分发表
   - LSP 补全数据
3. CI 检查：生成文件与源定义匹配。

**验证**：`ctest` + CI 生成器检查。

---

## E4. `XaJitMetadata` — 无消费者则删除

**问题**：`xanalyzer_jit.h` 定义了 `XaJitMetadata`，但不确定 JIT
是否真正消费此数据。JIT 主要使用：
- `proto->param_types`
- 运行时类型反馈 / IC 快照
- Xi slot map

**动作**：Grep 所有消费者。如果 JIT 编译路径中没有真正的消费者
→ 删除整个模块（`xanalyzer_jit.h`、`xanalyzer_jit.c`、所有
`xa_jit_*` 函数）。不做"留着以后用"——实际需要时可以从头重建。

**验证**：删除后 `ctest`。

---

## E5. 增量分析改进

**问题**：`xa_analyzer_refresh_file()` 执行全文件重建。
`xa_analyzer_invalidate_range()` 标记整个文件为脏。
这是 LSP 性能的核心瓶颈。

**路线图**（非单个 PR）：
1. **文件级依赖精度**：追踪每个文件 import 了哪些文件；
   只重分析传递性受影响的文件。
2. **函数/类级脏标记**：在文件内，仅重分析签名或函数体发生变化的函数。
3. **块级脏标记**（长期）：函数内语句级增量重分析。

每个层级独立有价值且可测试。

**涉及文件**：`src/frontend/analyzer/xanalyzer_incremental.c`、
`src/frontend/analyzer/xanalyzer.c`

---

## E6. Formatter 幂等性与 trivia roundtrip

**问题**：Formatter 基于 AST 再生成；注释/空白的 roundtrip 脆弱。

**需添加的测试**：
- **幂等性**：对所有测试语料库文件 `fmt(fmt(src)) == fmt(src)`。
- **注释保留**：行尾注释、同行块注释、多行块注释、
  class/function/member 前的 doc comment。
- **字符串字面量 roundtrip**：转义、raw string、模板字符串、unicode。
- **不输出弃用语法**：Formatter 不得生成旧语法。

**涉及文件**：`tests/unit/frontend/test_fmt_roundtrip.c`（新建）

---

## E7. JIT 编译预算 — 严格执行

**问题**：`xm_run_fixedpoint()` 支持 `XmCompileBudget`，但 JIT
主路径（`xm_jit.c`、`xjit_compile_queue.c`）调用
`xm_run_pipeline_ex()` 时未传递预算。

**修复**：
1. Tier 1 JIT（同步，主线程）：强制 10ms 预算。
2. 后台编译：每个函数强制 50ms 预算。
3. 预算超时 → 回退到解释器。不"保留部分优化代码"——
   要么完整优化，要么解释器。

**涉及文件**：`src/jit/xm_jit.c`、`src/jit/xjit_compile_queue.c`、
`src/jit/xm_pass.c`

**验证**：`ctest` + 大函数编译时间压力测试。

---

## E8. Parser `XrType*` 解耦

**问题**：Parser 在 AST 节点中直接存储 `XrType*` 指针。
这导致 parser（应为纯语法阶段）与运行时类型系统（语义关注点）耦合。

**涉及文件**：`src/frontend/parser/xparse_*.c`、`src/frontend/xast.h`

**修复**：
1. 引入 `XrTypeRef` — 一种轻量 AST 级类型表示，记录类型语法
   （名称、泛型、nullable、union arms），不解析为运行时 `XrType*`。
2. 将 AST 节点中所有 `XrType*` 字段替换为 `XrTypeRef`。
3. `XrTypeRef` → `XrType*` 的解析在 analyzer 中进行，
   类型解析本就属于那里。

**影响**：Parser 成为纯语法阶段。Analyzer 拥有所有类型解析。
Parser 中不再有运行时依赖。

**验证**：`ctest` + `scripts/run_regression_tests.sh`

---

## E9. 死代码与未使用 IC 快照清理

**问题**：`XmTarget.ic_snapshot` 被传递但未充分利用。多个内置 IC 字段
看起来被设置但从未被 JIT 优化 pass 消费。

**动作**：审计每个 `ic_snapshot` 的写入和读取位置。删除在优化 pipeline
中有写入者但无读取者的任何字段/路径。如果功能仅部分实现，整体删除——
等到完整设计就绪时可以干净地重建。

**涉及文件**：`src/jit/xm_jit.c`、`src/jit/xi_to_xm.c`、`src/jit/xm_pass_type.c`

**验证**：`ctest` + JIT E2E 测试。

---

# 实施排期

## 建议顺序（考虑依赖关系）

```
第 1 周：Track A（正确性 — 风险最高，改动最小）
  A1 → A2 → A3 → A4

第 2-3 周：Track B（架构 — 工作量最大）
  B1 → B2 → B3 → B4 → B5
  （每个子拆分为独立 commit，每步之间测试）

第 3-4 周：Track C + D（加固 + 对等）
  C1 → D1 → D2 → D3 → C2

第 4-5 周：Track E（干净设计，不再低优先级）
  E4 → E9 → E8 → E1 → E7 → E2 → E3 → E5 → E6
```

排期相比保守方案已压缩：无兼容负担，每项都是干净重写，
不是小心翼翼的迁移。

## 横切规则

1. **每个 commit**：`cd build && ctest --output-on-failure` 必须通过。
2. **每个 Track B commit**：`scripts/run_regression_tests.sh` 必须通过。
3. **Track B**：结构性重设计，不只是机械拆分。
4. **Track A commit**：每个都包含针对性的回归测试。
5. **积极删除**：未使用代码、死特性、投机性基础设施。
6. **注释用英文，不引用 doc 路径，不使用阶段标签。**

---

# 风险矩阵

| 风险 | 影响 | 可能性 | 缓解措施 |
|------|------|--------|----------|
| B1/B2 表驱动分发遗漏边界情况 | 高 | 低 | 每个 opcode handler 可独立测试 |
| A1 修复改变 JIT 编译行为 | 中 | 中 | 只移动提升发生的位置，不改变是否提升 |
| D1 信号处理器在 x64 上故障 | 中 | 低 | 双平台测试；guard page 保持 ARM64 专有 |
| C1 新验证器暴露 IR 生成器 bug | 中 | **预期中** | 修复生成器——这正是本项的价值 |
| A4 基于 liveness 的 deopt 快照破坏 JIT | 高 | 中 | 完整 JIT E2E + GC 压力测试套件 |
| E8 XrTypeRef 涉及大量文件 | 中 | 低 | 机械性重构；parser 测试捕获所有回归 |

---

# 完成标准

- [ ] `is_jit_eligible()` 是纯谓词（零 proto 写入）
- [ ] 无函数超过 500 行
- [ ] 无 `.c` 文件超过 3000 行
- [ ] `xi_verify` 无条件运行所有检查，全部测试通过
- [ ] JIT 调试基础设施在 ARM64 和 x64 上均可工作
- [ ] x64 frame smap ptr 正确实现并有测试
- [ ] 所有现有测试绿灯：ctest + Xi Compare + 回归
- [ ] 编译预算在所有 JIT 路径中强制执行
- [ ] 无死代码模块残留（XaJitMetadata、未使用的 IC plumbing）
- [ ] Parser 中零 `XrType*` 引用（类型解析仅在 analyzer 中）
