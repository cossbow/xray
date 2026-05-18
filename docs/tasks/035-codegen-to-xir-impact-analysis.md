# 035 — xcodegen → XIR 迁移影响分析

> **状态**：active
> **前置依赖**：034 XIR 设计文档
> **目标受众**：编译器核心开发者
> **目的**：逐项分析现有 AST→bytecode codegen 的特有机制，评估迁移到 AST→XIR(SSA)→bytecode 的影响

---

## 1. 现有 codegen 概况

`src/frontend/codegen/` 共 53 个文件，~21,700 行 .c + ~2,600 行 .h = **~24,300 行**。

### 1.1 核心子系统

| 子系统 | 主要文件 | 行数 | 职责 |
|--------|----------|------|------|
| **XrExprDesc 延迟求值** | `xexpr_desc.{c,h}` | ~780 | 表达式不立即发射指令，记录状态后按消费上下文选择最优路径 |
| **RELOC 指令回填** | `xexpr_desc.c` discharge2reg | ~100 | 指令先用 A=0 发射，在目标寄存器确定后回填 A 字段，避免 MOVE |
| **寄存器分配** | `xregalloc.c` | ~410 | LIFO 栈式分配，O(1)，protection stack 保护连续参数区 |
| **Peephole 优化** | `xemit.c`, `xpeephole.c`, `xemit_optimize.c` | ~1,160 | 实时窗口（redundant MOVE 消除）+ 后置 pass（跳转链、BOX+UNBOX 对消、NOP 压缩、尾调用检测） |
| **常量折叠** | `xconst_fold.c`, `xoptimize.c` | ~570 | 编译时常量求值（含变量查找 + 字符串拼接 + 范围传播） |
| **常量传播** | `xexpr_variable.c`, `xexpr_binary.c` | ~200 | const 变量内联为字面量，const 变量替换 AST 节点后触发下游 ADDI/MULK |
| **指令融合** | `xfusion.c` | ~320 | 后置 pass：LOADK→LOADI、LOADK+ADD→ADDI、LOADK+LT→LTI |
| **Inline 分析** | `xinline.c` | ~345 | 标记 inline 候选（无循环、无递归、指令数 ≤ 阈值），写入 proto->inline_hint |
| **类型感知发射** | `xexpr_binary.c`, `xstmt_typed.c` | ~2,180 | 根据 compile_type 选择 STRBUF/STR_REPEAT/BOX/UNBOX/ADDI/ADDK/LTI 等特化指令 |
| **Short-circuit** | `xexpr_binary.c` compile_and/or | ~100 | && / \|\| 通过 TESTSET+JMP 跳转链实现，延迟求值 |
| **BCE (Bounds Check Elimination)** | `xexpr_collection.c`, `xcompiler.h` | ~30 | for 循环中检测 index==loop_var 时发射 ARRAY_GET_NOCHECK |
| **GC Stackmap** | `xcompiler.c`, `xbc_stackmap.h` | ~80 | 在分配指令处记录 safepoint bitmap [0, freereg)，写入 proto->bc_stackmap |
| **inst_types per-PC 类型标注** | `xexpr_desc.c`, `xcompiler.c` | ~60 | 每个 discharge 点记录 XrType* → proto->inst_types[pc]，JIT/AOT 消费 |
| **三阶段编译** | `xcompiler.c` AST_BLOCK/PROGRAM | ~250 | Phase 1 hoist names, Phase 2 declarations, Phase 3 statements |
| **prescan_captured** | `xstmt_function.c` | ~260 | 编译函数体前扫描所有嵌套函数捕获的变量名，提前标记 captured |
| **for-in 特化** | `xstmt_forin.c` | ~910 | 10 种 for-in 变体：range(const/dynamic), array, enum, channel, Map/Set lazy iterator, custom iterator, Range object |
| **协程语句** | `xstmt_coroutine.c` | ~1,020 | go/spawn_cont/select/defer/await/channel_new/scope_block |
| **OOP 编译** | `xoop_class.c`, `xoop_enum.c`, `xoop_interface.c`, `xoop_class_descriptor_builder.c` | ~1,570 | class/struct/enum/interface → descriptor + vtable |

---

## 2. 问题一：现有 codegen 的特有优势，迁移到 XIR 后的影响

### 2.1 ✅ 可平移或在 XIR 中做得更好的（~60% 代码量）

#### 2.1.1 常量折叠 — **XIR 更好**

**现状**：`xconst_fold.c` + `xoptimize.c` (~570 行) 在 AST 层做折叠，只能折叠字面量和 const 变量。

**XIR 方案**：SSA 中的常量折叠天然覆盖全部情况：
- SSA value 的 def 是 CONST → 传播到所有 use
- 不需要特殊的"AST 节点替换"hack（现有 `xexpr_binary.c:500-531` 直接修改 AST 节点类型）
- 可做**全局常量传播** (GCP/SCCP)，现有方案做不到

**复杂度**：**下降**。SSA constfold 是标准 pass (~200 行)，替换 ~770 行 ad-hoc 代码。

#### 2.1.2 指令融合 (ADDI/MULK/LTI) — **XIR 更好**

**现状**：`xfusion.c` (~320 行) 做后置字节码 pattern matching（LOADK+ADD→ADDI）。`xexpr_binary.c` (~200 行) 在编译时检测 literal 直接发射 ADDI。两套重复逻辑。

**XIR 方案**：在 XIR→bytecode lowering 阶段，统一做 instruction selection：
- `XIR: add %1, %2` + `%2 = const(5)` → `emit(OP_ADDI, dst, %1, 5)`
- 只需一套逻辑，在 lowering 时统一处理

**复杂度**：**下降**。一套 lowering pass 替代两套分散的融合逻辑。

#### 2.1.3 Peephole 优化 — **大部分可消除**

**现状**：`xpeephole.c` (~510 行) 做 6 种后置优化：跳转链消除、冗余指令删除、BOX+UNBOX 对消、MOVE 消除、NOP 压缩、尾调用检测。

**XIR 方案**：
- **跳转链**：SSA 的 CFG 结构天然没有跳转链
- **冗余指令/MOVE**：SSA 的 φ-node + copy coalescing 天然消除
- **BOX+UNBOX 对消**：XIR typed value 用 rep 标注，lowering 时直接知道是否需要 BOX
- **NOP 压缩**：XIR→bytecode 是全新发射，不产生 NOP
- **尾调用检测**：XIR 在 CFG 的 return block 检查是否 call+return，更可靠

**仅需保留**：`xemit.c` 中的实时 peephole（同 dest MOVE 消除 ~10 行），用于最终字节码发射时的微优化。

**复杂度**：**显著下降**。~510 行 peephole → ~30 行。

#### 2.1.4 Inline 分析 — **XIR 更好**

**现状**：`xinline.c` (~345 行) 在**字节码层**分析（数指令、检测 backward jump、查 OP_CLOSURE），非常粗糙。

**XIR 方案**：在 SSA 上做 inline 决策：
- 直接看 CFG 有无回边（loop detection）
- 精确的调用图（不需要 pattern-match OP_CLOSURE → OP_CALL）
- 可做真正的 inline expansion（XIR 层函数内联），现有方案只标记 hint

**复杂度**：**更强大且更简单**。

#### 2.1.5 inst_types per-PC 类型标注 — **被 XIR 替代**

**现状**：codegen 在每个 discharge 点记录 XrType* → `proto->inst_types[pc]`，JIT builder 消费。

**XIR 方案**：XIR value 自带 `XrType *type` + `XrRep rep`，不需要 side-table。JIT/AOT 直接读 XIR value 的类型。

**复杂度**：**消除**。

#### 2.1.6 GC Stackmap — **XIR 更好**

**现状**：在 emit 时插桩记录 bitmap，只在分配指令处有精确 map，其他 PC 是保守扫描。

**XIR 方案**：SSA 的 liveness analysis 可精确计算每个 safepoint 的 live value set + 它们的 rep (PTR/I64/F64)，生成完美的 stackmap。

**复杂度**：**上升~100行**（需要 liveness pass），但精度大幅提高。

---

### 2.2 ⚠️ 需要仔细设计的（复杂度基本持平或略升）

#### 2.2.1 XrExprDesc 延迟求值 + RELOC 回填 — **SSA 天然实现**

**现状**：`XrExprDesc` 是 xray codegen 最精巧的机制。表达式编译不立即发射指令，而是返回一个描述符（kind=RELOC/LOCAL/TEMP/CONST/...），消费者根据上下文决定目标寄存器。对于 `var x = a + b`：
- `compile_binary` 发射 `ADD R[0], Rb, Rc`（A=0 占位）
- 返回 `XEXPR_RELOC, pc=当前pc`
- `compile_var_decl` 调用 `xexpr_to_specific_reg(e, local_reg)` → 回填 A 为 local_reg
- 结果：**零 MOVE 指令**

**XIR 方案**：SSA 中这个问题被**自动解决**：
- SSA value 是虚拟的（`%5 = add %3, %4`），没有物理寄存器
- Register allocation 在 XIR→bytecode lowering 时才做
- Copy coalescing 自然消除冗余 MOVE

**结论**：XrExprDesc 的核心价值（避免 MOVE）在 SSA 中被 regalloc 天然覆盖。但需要一个**好的 SSA→bytecode register allocator**，否则反而会产生更多 MOVE。

**推荐**：使用 linear scan regalloc（Go SSA 做法），~300-400 行。不需要达到 LLVM 精度，但不能比现有 LIFO 差。

**风险等级**：🟡 中。regalloc 质量决定字节码质量。

#### 2.2.2 三阶段编译 (Hoist → Declarations → Statements) — **需要 XIR 层等价**

**现状**：`xcompiler.c` 中的 `AST_BLOCK` / `AST_PROGRAM` 处理：
1. Phase 1：扫描所有声明名，预分配 local slot / shared slot，为嵌套函数提供前向引用
2. Phase 2：编译所有声明（class/enum/function/import）
3. Phase 3：编译剩余语句

这实现了 JavaScript 风格的**声明提升 (hoisting)**——函数可在定义前调用。

**XIR 方案**：XIR builder（AST→XIR）仍然需要这个三阶段逻辑，因为：
- 这是**语言语义**，不是优化
- SSA 构建需要知道所有变量的名字和作用域才能正确生成 φ-node

**复杂度**：**持平**。代码会从 xcompiler.c 移到 XIR builder，结构类似。

#### 2.2.3 prescan_captured (闭包变量预扫描) — **需要 XIR 层等价**

**现状**：`xstmt_function.c` 编译函数体前，递归扫描所有嵌套函数引用的外层变量名，提前标记 `is_captured`。这影响：
- 变量是否需要 cellify（存入 heap cell 而非 register）
- upvalue 深度计算

**XIR 方案**：仍需预扫描，但可以更优雅：
- XIR builder 先做 name resolution pass（类似 analyzer），标记每个 variable 的 scope + captured
- SSA 构建时，captured variable 直接建模为 `cell_alloc + cell_get/cell_set`
- 比现在的"prescan 填表 → define_local 查表"更结构化

**复杂度**：**持平**。逻辑不变，实现更干净。

#### 2.2.4 Short-circuit Evaluation (&&, ||) — **SSA 标准做法**

**现状**：`compile_and/or` 生成 `TESTSET + JMP` 序列，用跳转链管理延迟求值。

**XIR 方案**：在 SSA 中，short-circuit 变成 **条件分支 + φ-node**：
```
  %cond_l = eval(left)
  br %cond_l, bb_right, bb_false
bb_right:
  %cond_r = eval(right)
  br bb_merge
bb_false:
  br bb_merge
bb_merge:
  %result = phi [%cond_l, bb_false], [%cond_r, bb_right]
```
这是 SSA 编译器的标准做法。

**复杂度**：**持平**。代码量类似，但结构更清晰。

#### 2.2.5 for-in 10 种变体 — **需要逐个迁移**

**现状**：`xstmt_forin.c` (~910 行) 有 10 种 for-in 编译路径：
1. `compile_for_in_range` — const range (0..N)
2. `compile_for_in_range_dynamic` — dynamic range (a..b)
3. `compile_for_in_range_object` — Range object → FORPREP/FORLOOP
4. `compile_for_in_array_single` — array index-based
5. `compile_for_in_enum_single` — enum members
6. `compile_for_in_keyvalue` — Map key-value
7. `compile_for_in_lazy_iterator` — Map/Set lazy iterator
8. `compile_for_in_custom_iterator` — custom iterator protocol
9. `compile_for_in_channel` — channel receive loop
10. 默认 array 路径

**XIR 方案**：每种变体在 XIR 中都需要对应的 lowering pattern。但优势是：
- Range loop → 直接生成 `%i = phi [start, entry], [%i_next, body]` + `%i_next = add %i, 1`
- Array/Map/Set → 可做 iterator devirtualization
- Channel → 生成标准 `chan_recv + branch` CFG

**复杂度**：**持平**。~910 行 → ~800-900 行，结构从 switch 变为 XIR lowering patterns。

---

### 2.3 🔴 复杂度真正上升的地方

#### 2.3.1 寄存器分配：LIFO → SSA regalloc

**现状**：LIFO 分配器 (`xregalloc.c` ~410 行) 极其简单：freereg++ 分配，freereg-- 回收。局部变量固定在 register 0..N-1，临时值在 N 之后栈式增长。O(1) 且零开销。

**XIR 方案**：SSA→bytecode 需要真正的 register allocator：
- SSA 有 unbounded 虚拟寄存器，需要映射到 255 个 bytecode register
- 需要 liveness analysis + interference graph 或 linear scan
- 需要 spill handling（如果 live value 超过 255）
- 需要 phi-node resolution（parallel copy）

**新增代码**：**~500-700 行**（linear scan + phi resolution + spill）

**这是最大的新增复杂度**。但 Go、Lua 5.x、CPython 都走过这条路，有成熟算法。

**推荐**：先实现最简单的 greedy coloring（~300 行），后续可升级到 linear scan。

#### 2.3.2 SSA 构建本身

**新增代码**：
- **Braun 算法**（on-the-fly SSA construction）：~200-300 行
- **CFG 构建**（basic block 划分）：~200 行
- **XIR node pool / value pool**：~150 行
- **Dominator tree**（用于 GCM 等高级优化）：~150 行（可延后）

总计：**~700-1,000 行**（核心框架）

#### 2.3.3 类型感知指令选择变复杂

**现状**：`xexpr_binary.c` 在编译时直接检查 `get_expr_type()` 选择 STRBUF/ADDI/STR_REPEAT：
```c
if (left_ct && XR_TYPE_IS_STRING(left_ct) && right_ct && XR_TYPE_IS_STRING(right_ct)) {
    // 直接发射 STRBUF_NEW + STRBUF_APPEND + STRBUF_FINISH
}
```
这种 AST walk + type check + emit 是"一遍完成"的。

**XIR 方案**：需要分两步：
1. AST→XIR：`%r = call @string_concat(%a, %b)` 或 `%r = add %a, %b` (unresolved)
2. XIR lowering：检查 `%a.type == string` → 选择 STRBUF 序列

**复杂度**：**略升**。原来是 inline 判断+发射（~50 行/pattern），现在需要在 lowering pass 中统一处理（~80 行/pattern）。但好处是：
- 类型检查集中在一个 pass，不散布在 20 个文件中
- XIR 优化 pass 可以跨语句推理（e.g., 多个连续 string concat 合并为一个 STRBUF 序列）

#### 2.3.4 协程语句的 SSA 表示

**现状**：`xstmt_coroutine.c` (~1,020 行) 直接发射特殊操作码：OP_GO, OP_SPAWN_CONT, OP_YIELD, OP_CHAN_SEND, OP_CHAN_RECV, OP_SELECT 等。

**XIR 方案**：这些操作码需要在 XIR 中建模为 intrinsic call + 特殊 CFG 边：
- `go expr` → `call @xir_go(closure)` — 简单
- `select` → 多个 `chan_try_recv` + 条件分支 — 需要仔细建模
- `scope { ... }` (continuation stealing) → OP_SPAWN_CONT 本质是一个 scope marker + 隐式 continuation

**复杂度**：**略升**。select 语句的 CFG 表示比线性字节码更复杂。但这也意味着 XIR 优化 pass 可以分析 select 的分支结构。

---

## 3. 问题二：现有字节码的类型信息现状

### 3.1 已有的类型信息

现有字节码实际上**已经携带了不少类型信息**，分为 4 个层次：

| 层次 | 存储位置 | 内容 | 消费者 |
|------|----------|------|--------|
| **① param_types** | `proto->param_types[i]` = `XrType*` | 每个参数的编译时类型 | JIT entry guard, AOT codegen |
| **② inst_types** | `proto->inst_types[pc]` = `XrType*` | 每条指令结果的编译时类型 | JIT builder, AOT struct inference |
| **③ return_type_info** | `proto->return_type_info` = `XrType*` | 函数返回类型 | JIT caller specialization |
| **④ upval type_info** | `UpvalInfo.type_info` = `XrType*` | 每个 upvalue 的编译时类型 | JIT upvalue access |
| **⑤ bc_stackmap** | `proto->bc_stackmap` | 分配指令处的 live slot bitmap | GC precise scanning |
| **⑥ inline_hint** | `proto->inline_hint` | 是否适合内联 | JIT inline decision |
| **⑦ loop_headers / bb_leaders** | `proto->loop_headers`, `proto->bb_leaders` | 循环头 / 基本块入口 bitmap | JIT OSR, CFG construction |
| **⑧ 专用操作码** | OP_BOX_I64, OP_UNBOX_F64, OP_ADDI, OP_LTI, OP_ARRAY_GET_NOCHECK, OP_STRBUF_* | 指令本身隐含类型 | VM fast path |

### 3.2 架构限制导致无法带全部类型信息

**是的，迫于架构设计，类型信息不完整**。具体限制：

#### 3.2.1 inst_types 是稀疏的

`inst_types[pc]` 只在有 `XrExprDesc.compile_type` 的 discharge 点被填充。大量指令没有类型标注：
- 来自 untyped 变量的操作
- 控制流指令（JMP, TEST）
- 容器操作（NEWARRAY, NEWMAP）的元素类型丢失
- 方法调用返回值类型经常是 NULL（analyzer 推不出来）

**根因**：codegen 是 one-pass 发射，analyzer 的类型推断和 codegen 是解耦的。Analyzer 只标注 AST 节点（`XaNodeTable`），codegen 在发射时尝试从 AST 节点拿类型，但：
- 有些表达式没有对应 AST 节点（中间临时值）
- analyzer 的推断不总是精确的（XR_KIND_UNKNOWN 回退）

#### 3.2.2 类型信息无法跨基本块流动

当前 codegen 是线性发射，没有 CFG 概念。类型信息无法跨基本块传播：
```
if (x is int) {
    // 这里 codegen 知道 x: int
    let y = x + 1  // → ADDI (typed path)
} else {
    // 这里 codegen 不知道 x 的类型
    let y = x + 1  // → ADD (generic path)
}
// 合并后，y 的类型丢失
```

在 SSA 中，flow-sensitive type narrowing 可以通过 φ-node 精确传播：
```
bb_if:
  %x_int = assert_type %x, int
  %y1 = addi %x_int, 1       ; typed
bb_else:
  %y2 = add %x, const(1)     ; generic
bb_merge:
  %y = phi [%y1, bb_if], [%y2, bb_else]  ; type = int|unknown
```

#### 3.2.3 BOX/UNBOX 决策是局部的

`XrExprDesc.is_raw` 标志在单个表达式范围内跟踪"寄存器里是 raw i64/f64 还是 tagged XrValue"。但跨语句时：
- 每条语句结束后 `freereg` 重置到 `local_end`
- local 变量永远是 tagged（`is_raw` 只在临时值上有效）
- 所以 `let x: int = compute()` → x 存的是 **tagged** XrValue，每次用 x 都需要 runtime tag check

在 XIR 中，`%x` 的 `rep = XR_REP_I64` 可以贯穿整个函数，lowering 到 bytecode 时可以：
- 为 int-typed local 分配专用 register，在寄存器中保持 raw i64
- 只在函数边界（call/return）和 GC safepoint 做 BOX

#### 3.2.4 字节码指令集没有为全类型设计

当前指令集是"通用操作码 + 少量特化操作码"混合：
- `OP_ADD/SUB/MUL/DIV` 是 **untyped**——VM 内部 runtime dispatch
- `OP_ADDI/SUBI/MULI` 是 **半特化**——右操作数是 immediate，但左操作数仍是 tagged
- `OP_BOX_I64/UNBOX_F64` 是 **转换指令**——在 typed/untyped 边界插入

没有 `OP_ADD_I64`、`OP_ADD_F64` 等全特化指令。这意味着：
- VM 永远需要 runtime type dispatch（~5-10 ns/op overhead）
- JIT 通过 `inst_types` 做 speculative specialization，但需要 deopt guard

**如果引入 XIR**：可以在 XIR→bytecode lowering 时，根据 type 选择：
- 全类型已知 → 发射特化指令（需要扩展指令集）或 inline 展开
- 部分类型 → 发射 guard + fast path
- 未知类型 → 发射通用指令

---

## 4. 综合评估

### 4.1 代码量变化估算

| 分类 | 现有行数 | XIR 后行数 | 变化 |
|------|----------|-----------|------|
| 常量折叠/传播 | ~770 | ~200 (SSA pass) | **-570** |
| 指令融合 | ~520 | 0 (merged into lowering) | **-520** |
| Peephole | ~510 | ~30 | **-480** |
| Inline 分析 | ~345 | ~200 (SSA inline) | **-145** |
| inst_types bookkeeping | ~60 | 0 (XIR 自带类型) | **-60** |
| XrExprDesc 系统 | ~780 | 0 (被 SSA regalloc 替代) | **-780** |
| LIFO regalloc | ~410 | 0 (被 SSA regalloc 替代) | **-410** |
| 类型感知发射 | ~2,180 | ~1,800 (移到 lowering) | **-380** |
| **新增：SSA core** | 0 | ~800 (Braun + CFG + pool) | **+800** |
| **新增：SSA regalloc** | 0 | ~500 (linear scan) | **+500** |
| **新增：XIR→bytecode lowering** | 0 | ~600 (instruction selection) | **+600** |
| **新增：Liveness + GC stackmap** | 0 | ~200 | **+200** |
| 其他（forin/coro/oop/scope）| ~5,680 | ~5,400 (结构调整) | **-280** |
| **净变化** | ~11,255 | ~9,730 | **-1,525** |

### 4.2 关键结论

1. **没有"在 SSA 中无法实现"的功能**——所有现有特性都可以在 XIR 中表达
2. **绝大多数优化在 SSA 中更简单、更通用**（constfold, DCE, peephole, inline）
3. **主要新增复杂度是 SSA regalloc**（~500-700 行）——这是唯一真正新增的工程量
4. **类型信息在 XIR 中天然更完整**——现有 inst_types 的稀疏性和跨 BB 断裂问题被彻底解决
5. **风险集中在 regalloc 质量**——如果 SSA regalloc 做得差，字节码质量反而下降
6. **迁移可以增量进行**——先实现 XIR 框架 + lowering 基本指令，逐步迁移 for-in/coro/oop 等

### 4.3 建议实施顺序

1. **S1**：XIR core (value/inst/block/func) + Braun SSA constructor + 基本 CFG
2. **S2**：XIR→bytecode lowering (基础指令: load/store/arith/cmp/branch/call/return)
3. **S3**：Linear scan regalloc + phi resolution
4. **S4**：类型感知 lowering (ADDI/STRBUF/BOX/UNBOX 指令选择)
5. **S5**：for-in/coro/oop lowering patterns
6. **S6**：SSA optimization passes (constfold, DCE, type specialization)
7. **S7**：切换默认管线到 XIR，删除 xcodegen

每一步都可以独立测试（XIR→bytecode 对比现有 codegen 的输出差异）。
