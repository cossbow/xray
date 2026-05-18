# JIT 稳定化实施计划（2026-04-23 · rev 2）

> Status: Draft → **Active**
> Scope: `src/jit` 正确性、已确认 bug 修复、死代码清理与发布边界收敛
> Related: `jit_refactoring_plan.md` · `jit_next_phase.md` · `jit_known_limitations.md` · `jit_vm_boundary.md`

---

## 0. 开发原则

> **⚡ 不考虑向后兼容性！**

Xray 没有外部用户，代码量可控，每个阶段都选择最佳设计。

- **直接采用最佳设计**：发现错误的抽象或接口就直接改掉，不做兼容层
- **删除死代码**：`if (false && ...)` 之类直接删除，需要时重新实现
- **统一到单一语义**：同一概念不允许两种实现，直接统一
- **消灭启发式**：能在编译期确定的信息不依赖运行时猜测
- **不保留"将来可能有用"的代码**：未验证的代码是负债，不是资产

判断标准：`能删就删 > 能简化就简化 > 能统一就统一 > 实在不行再加复杂度`

---

## 1. 背景与目标

`src/jit` 已完整接入 CMake / VM 热点检测 / OSR / worker 协程 / CLI 开关 / bg compile queue。

现阶段的主要问题：

- **两个 P0 SIGSEGV**（对象链、closure 多实例）
- **4 个已确认的代码级 bug**（offset 乘 4、deopt 启发式、release fence、spill 过期）
- **死代码增加维护负担**（`if(false && ...)`、空函数桩）
- **x86-64 处于早期实验阶段但 CMake 默认开启**

目标：

1. 消灭 P0 SIGSEGV
2. 修复已定位的代码级 bug
3. 清理死代码，降低维护负担
4. 统一安装语义为单一路径
5. 建立可重复的 JIT 回归矩阵
6. 明确 ARM64 / x86-64 可用边界

---

## 2. 当前基线（源码审计 + 实测）

### 2.1 源码规模

| 维度 | 数值 |
|------|------|
| 总文件数 | 63 (.c + .h) |
| 总代码行 | **41,686** |
| `.c` 最大 | `xir_builder.c` 2708 行（均 ≤ 3000 行规范） |
| `.h` 最大 | `xir.h` 679 行（均 ≤ 800 行规范） |

### 2.2 子系统结构

| 子系统 | 行数 | 职责 |
|--------|------|------|
| XIR IR | ~1.5k | SSA IR、vreg、block、指令 |
| Builder (4 files) | ~6.5k | Bytecode → XIR |
| 优化管线 | ~6.4k | CSE/GVN/LICM/DCE/SCCP/if-conv/peephole |
| 寄存器分配 | ~3.6k | 线性扫描 RA + coalesce |
| ARM64 codegen (3 files) | ~4.8k | XIR → ARM64 |
| x64 codegen (2 files) | ~3.4k | XIR → x86-64（实验性） |
| JIT 入口/桥 | ~1.3k | 编译触发、调用桥、deopt recovery |
| Runtime helpers | ~2.8k | 100+ C helper |
| 后台编译 | ~0.4k | MPSC 队列 + CAS |
| 其他 | ~11k | TFA、domtree、looptree、alias、debug 等 |

### 2.3 已具备的能力

**单元测试验证通过（10/10）：**

- XIR IR 构建、def-use、bset、fold、liveness、pass 管线
- ARM64 / x86-64 指令编码 + code allocator

**已集成但端到端不稳定：**

- 热函数检测 + Tier 1/Tier 2 分层编译
- 后台编译队列（N worker + MPSC ring buffer + CAS）
- deopt 表 + OSR 表 + GC stack map
- JIT suspend/resume（channel block → 挂起 → worker 恢复）
- code cache 驱逐（LRU + epoch-based reclaim）
- ~120 个 opcode 标记为 `JIT_OP_SUPPORTED`

### 2.4 已验证运行结果

| 用例 | 结果 | 结论 |
|------|------|------|
| JIT 单元测试 (10 tests) | **10/10 通过** | 基础层可用 |
| `1600_nested_jit_deopt.xr` + `--jit-force` | **通过** | deopt 路径可用 |
| `0661_jit_cross_boundary.xr` + `--jit-force` | **SIGSEGV** | 对象/属性链 P0 |
| `0661_jit_cross_boundary.xr` + `--no-jit` | **通过** | 问题属于 JIT |
| `0521_cell_upval.xr` + `--jit-force` | **SIGSEGV** | closure/cell P0 |
| `test_jit_e2e` / `test_aot_e2e` | **被注释掉** | `tests/unit/CMakeLists.txt:104-105` |

### 2.5 已确认的代码级 bug

以下通过源码审计确认，无需再排查"是否存在"，直接进入修复。

#### Bug #1：x86-64 offset 乘 4

**位置**：`xir_jit.c:510`、`xjit_compile_queue.c:130`、`xir_jit_debug.c:80`

```c
proto->jit_fast_entry = (char *)res.code + res.fast_entry_offset * 4;
proto->jit_resume_entry = res.resume_entry_offset
    ? (char *)res.code + res.resume_entry_offset * 4 : NULL;
```

`* 4` 是 ARM64 语义（指令索引 → 字节），但 x64 codegen 的 `resume_entry_offset` 已是字节偏移（`ctx->buf.pos`）。x64 含 suspend 点时恢复必崩。`fast_entry_offset = 0` 碰巧无害。

**修复**：统一所有 offset 为**字节偏移**。ARM64 codegen 端乘 4 后再赋值给 `result.fast_entry_offset`，安装端不再乘。3 处同改。

#### Bug #2：deopt 重建地址启发式

**位置**：`xir_jit_internal.h:148`、`xir_jit.c:1006`

```c
if (raw != 0 && (raw & 0x7) == 0 && (uint64_t)raw > 0x10000) {
    v.tag = XR_TAG_PTR;
    v.heap_type = (uint16_t)((XrGCHeader *)(intptr_t)raw)->type;  // 可能非法解引用
}
```

`xr_tag == UNKNOWN` 时用地址范围启发式判断 raw 是否为指针。整数恰好满足对齐条件即误判，解引用 GC header → SIGSEGV。出现在 `deopt_reconstruct()` 和 `xir_jit_read_multi_ret()` 两处。

**修复**：**删除启发式**。要求编译期携带明确 `xr_tag`；无法确定时默认 `XR_TAG_I64`（安全默认值），由解释器恢复后再做正确的类型检查。

#### Bug #3：同步编译无 release fence

**位置**：`xir_jit.c:509-516`

同步路径用普通写发布 `jit_entry` + metadata，后台路径正确使用 `atomic_store_explicit(..., memory_order_release)`。ARM64 弱内存序下其他线程可能看到不一致状态。

**修复**：**删除两套路径**，统一为单一 `jit_install_result()` helper（工作流 C）。

#### Bug #4：DEOPT_LOC_SPILL 读取过期栈帧

**位置**：`xir_jit.c:1047-1056`

源码注释承认：`stack frame is destroyed after epilogue; spill data may be stale`。deopt stub 在 epilogue 之后触发 recovery，栈帧已销毁，spill slot 数据不可靠。

**修复**：deopt stub 在栈帧销毁**之前**保存 spill 数据到 `jit_ctx->deopt_spill_save[]`，或在 deopt codegen 中将 spill 值先 load 到寄存器再触发 deopt。

### 2.6 死代码与实验残留

| ID | 位置 | 内容 | 处理 |
|----|------|------|------|
| Dead #1 | `xir_builder_call.c:1945` | `if (false && ...) { CALL_KNOWN for CALL_KEEP }` ~30 行 | **删除** |
| Dead #2 | `xjit_compile_queue.c:324-329` | `xjit_install_pending()` 空函数 | **删除** |

### 2.7 Eligibility 限制（记录，本轮不改）

`xir_eligibility.c` 当前准入门槛：

- ≤ 8 参数、≤ 16 upvalue、不支持 vararg
- 必须有类型信息（`param_types` 或 `type_feedback`）
- 14 个 opcode 为 `BAIL_OUT`（`OP_YIELD`、`OP_SLEEP`、`OP_DEFER` 等）
- deopt ≥ 20 次永久禁用

这些是有意的设计边界，本轮不扩展，但在文档中明确记录。

### 2.8 状态总结

| 维度 | 状态 |
|------|------|
| IR + 优化管线 + 单元测试 | ✅ 可用 |
| ARM64 E2E 基本路径 | ⚠️ 部分可用 |
| ARM64 并发/suspend/resume | ⚠️ 缺 E2E 覆盖 |
| x86-64 | ❌ 实验阶段 |
| 默认开启 | ❌ 不建议 |

---

## 3. 收敛目标与退出标准

### 3.1 第一层：ARM64 稳定化

- [ ] `0661` 与 `0521` 在 `--jit-force` 下通过
- [ ] Bug #2 #3 #4 修复
- [ ] Dead #1 #2 清理
- [ ] `run_jit_diff_tests.sh` ARM64 CRASH 清零
- [ ] JIT 单元测试全绿（含恢复 `test_jit_e2e`）
- [ ] `ctest` + `run_regression_tests.sh` 通过

### 3.2 第二层：x86-64 准入

- [ ] Bug #1（offset * 4）修复，统一为字节偏移
- [ ] x86-64 至少一组 E2E smoke
- [ ] `--jit-force` 下基础算术/控制流/调用/deopt 通过

达到之前，x86-64 通过编译开关降级为**显式 opt-in**。

---

## 4. 实施总顺序

| 工作流 | 优先级 | 目标 | 预估 |
|--------|--------|------|------|
| A | P0 | 修复对象/属性链崩溃 | 1-2 d |
| B | P0 | 修复 closure/cell/upvalue 崩溃 | 1-2 d |
| C | P1 | 统一安装语义 + release fence（Bug #3） | 0.5 d |
| D | P1 | 修复 Bug #1 #2 #4 + 清理 Dead #1 #2 | 1 d |
| E | P1 | x86-64 边界治理 | 1 d |
| F | P1 | 测试矩阵与文档同步 | 0.5 d |

```text
A -> B -> C -> D -> E -> F
```

---

## 5. 工作流 A：对象/属性链正确性（P0）

### 5.1 问题

- `0661_jit_cross_boundary.xr` 中 `test_object_field_chain` 在 `--jit-force` 下 SIGSEGV
- JIT 生成代码把整数当对象指针写字段

### 5.2 怀疑点

1. GETPROP/SETPROP fast path 对象值类型丢失
2. shape guard 通过后 slot 残留过时的 `shape_hint` / `slot_tag` / `slot_rep`
3. `XIR_STORE_FIELD` 对 obj/value 的 tag 契约与 codegen 不一致
4. 链式对象访问中上一层返回值被错误当作下一层对象基址

### 5.3 修改文件

- `src/jit/xir_builder_object.c` — field access builder
- `src/jit/xir_builder_misc.c` — slot 元数据管理
- `src/jit/xir_codegen.c` — ARM64 field access 发射
- `src/jit/xir_jit_runtime.c` — `xr_jit_getprop()` / `xr_jit_setprop()` / `xr_jit_getfield_ic()`

### 5.4 步骤

**A-1 最小复现 + 断言**

- 从 `0661` 抽取最小复现
- 加 `XR_DCHECK`：`XIR_STORE_FIELD` obj rep 必须为 `XR_REP_PTR`、`XIR_LOAD_FIELD` 返回值 tag/rep 同步

**A-2 梳理对象值流**

逐步确认 `root → current → current.child → node → node.child` 在 builder 中的 `slot_map` / `slot_rep` / `slot_tag` / `shape_hint` 传播。排除：

- `builder_set_slot()` 后语义信息未清理
- `VTAG_TAGGED` + `XR_REP_I64` 在对象路径上失真
- shape 命中后对象本身未被 guard 成 PTR

**A-3 修正 fast path**

原则：**不闭合的 fast path 直接删除，退回 `CALL_C` helper。**

- 对象类值：只有 `PTR` / `nullable PTR` 才进 `LOAD_FIELD/STORE_FIELD`
- 未知 tagged 值：必须经 guard 消歧
- 链式结果：每步清理 `shape_hint`

**A-4 回归固化**

新增：对象字段链写入/读取、mixed-type field overwrite、interpreter ↔ JIT 边界对象传递

### 5.5 验收

- [ ] `0661` + `--jit-force` 通过
- [ ] `--no-jit` 与 `--jit-force` 输出一致
- [ ] 最小复现加入回归集

### 5.6 提交

- A1: 最小复现 + 断言
- A2: 修复 builder/object fast path
- A3: 回归测试

---

## 6. 工作流 B：closure / cell / upvalue 正确性（P0）

### 6.1 问题

- `0521_cell_upval.xr` 中 `test_many_instances` 在 `--jit-force` 下 SIGSEGV
- 多实例、多次调用、共享/独立 cell 混合路径崩溃

### 6.2 怀疑点

1. `call_closure` 在 direct/fallback/resume 路径上不一致
2. `xr_jit_closure_new()` 对 `UPVAL_SRC_UPVAL` / `UPVAL_SRC_REG` 初始化不完整
3. `xr_jit_closure_set_upval()` 的 `call_args[0..1]` 在嵌套调用中被污染
4. 多 closure 实例的 `closure->upvals[]` 被错误共享或覆盖
5. JIT→JIT / JIT→VM 切换时 closure 上下文泄漏

### 6.3 修改文件

- `src/jit/xir_builder_call.c` — call/closure builder
- `src/jit/xir_codegen_call.c` — ARM64 call codegen
- `src/jit/xir_jit_runtime.c` — `xr_jit_closure_new()` / `xr_jit_closure_set_upval()` / `xr_jit_upval_get()`

### 6.4 步骤

**B-1 closure 路径矩阵**

按机制拆分 `0521` 场景：`CELL_NEW/GET/SET`、`UPVAL_GET`、transitive capture、const capture、shared cell、many instances。标明每类走哪条调用路径。

**B-2 固化 `call_closure` 契约**

统一要求：

- callee 所见 `jit_ctx->call_closure` 必须是当前 callee 的 closure
- 任何 direct/known/slow/resume path 满足同一契约
- 调用返回后必须**显式恢复** caller 上下文

**原则：难以证明正确的 fast path 直接退回慢路径。**

**B-3 审计 closure materialization**

- upvalue 数量边界
- 多实例是否共享错误的 `closure->upvals[]`
- `UPVAL_SRC_UPVAL` / `UPVAL_SRC_REG` 填充顺序
- 嵌套 closure 构造时 `call_args` 是否被后续调用覆盖

**B-4 回归固化**

新增：many instances、nested closure mutation、shared cell、closure call after deopt

### 6.5 验收

- [ ] `0521` + `--jit-force` 通过
- [ ] 关键子场景进入回归集
- [ ] closure 相关 JIT diff CRASH 清零

### 6.6 提交

- B1: closure 路径矩阵 + 断言
- B2: 修复 `call_closure` / upvalue
- B3: 回归测试

---

## 7. 工作流 C：统一安装语义（P1 · Bug #3）

### 7.1 问题

当前存在**两套**编译结果安装路径，语义不一致：

| 路径 | 位置 | 内存序 |
|------|------|--------|
| 同步编译 | `xir_jit.c:509-516` | **普通写**，无 fence |
| 后台安装 | `xjit_compile_queue.c:160` | `atomic_store_explicit(..., memory_order_release)` |

两套路径长期容易漂移，且同步路径在 ARM64 弱内存序下有并发风险。

### 7.2 修复方案

**原则：删除两套路径，统一为单一 helper。**

新增 `static void jit_install_result(XrProto *proto, XirCodegenResult *res, uint8_t opt)`：

```
1. 写 stack_map, deopt_table, osr_entries
2. 写 jit_fast_entry, jit_resume_entry, jit_opt_level
3. atomic_thread_fence(memory_order_release)
4. 写 jit_entry  ← 最后发布
```

同步编译和后台安装都调用此 helper，禁止再有独立的写入序列。

同时定义统一的失效 helper `jit_invalidate_result(XrProto *proto)`：

- 先 `jit_entry = NULL`（acquire fence）
- 再清理 `jit_fast_entry`、`jit_resume_entry`、`stack_map` 等
- 明确哪些字段必须同步清空，哪些为活跃协程恢复保留

### 7.3 审计 VM 读取点

检查所有读取 `proto->jit_entry` 的位置（`OP_CALL`、OSR trigger、worker fast path、resume path），要求只依赖通过发布契约保证可见的数据。

### 7.4 验收

- [ ] 同步和后台不再有两套发布顺序
- [ ] 不存在"先写 `jit_entry` 后写 metadata"的路径
- [ ] 所有失效逻辑统一走 helper

### 7.5 提交

- C1: 新增 `jit_install_result()` + `jit_invalidate_result()`，替换所有直接写入

---

## 8. 工作流 D：已确认 bug 修复 + 死代码清理（P1）

### 8.1 概述

本工作流处理 §2.5 和 §2.6 中**已确认、无需再排查**的问题，直接执行修复。

### 8.2 D-1：修复 Bug #2 — 删除 deopt 地址启发式

**修改文件**：`xir_jit_internal.h`、`xir_jit.c`

**操作**：

1. 删除 `deopt_reconstruct()` 中的 `if (raw != 0 && (raw & 0x7) == 0 && ...)` 分支
2. 删除 `xir_jit_read_multi_ret()` 中的同类分支
3. `xr_tag == UNKNOWN` 时统一默认为 `XR_TAG_I64`
4. 在 builder 端确保 deopt slot 尽可能携带编译期 tag（`XirRtDeoptSlot.xr_tag`）

**原则**：宁可 deopt 后解释器侧多做一次类型检查，也不在 JIT 侧做不可靠的猜测。

### 8.3 D-2：修复 Bug #4 — DEOPT_LOC_SPILL 过期栈帧

**修改文件**：`xir_codegen.c`（ARM64 deopt stub）、`xir_codegen_x64.c`（x64 deopt stub）、`xir_jit.c`

**方案**（二选一，选更简单的）：

- **方案 A**：deopt stub 在销毁栈帧前，先把 spill slot 值 load 到 `jit_ctx->deopt_spill_save[]` 数组
- **方案 B**：codegen 的 deopt 点生成代码中，将每个 spill slot 先 load 到临时寄存器，再存到 `deopt_regs[]`（把 spill 变成 reg，消除 `DEOPT_LOC_SPILL` 路径）

方案 B 更彻底——如果可行，直接删除 `DEOPT_LOC_SPILL` 枚举值和处理代码。

### 8.4 D-3：清理 Dead #1 — CALL_KEEP 的 CALL_KNOWN 死代码

**修改文件**：`xir_builder_call.c`

**操作**：删除 `if (false && callee_proto && !b->aot_mode)` 整个分支（约 30 行）。如果将来需要 CALL_KEEP 的 CALL_KNOWN 路径，从零开始实现，不保留未验证的残留代码。

### 8.5 D-4：清理 Dead #2 — xjit_install_pending 空函数

**修改文件**：`xjit_compile_queue.c`、`xjit_compile_queue.h`

**操作**：删除 `xjit_install_pending()` 函数定义和声明。如果有调用点，一并删除。

### 8.6 验收

- [ ] deopt 重建无地址启发式
- [ ] DEOPT_LOC_SPILL 路径修复或消除
- [ ] 无 `if (false && ...)` 死代码
- [ ] 无空函数桩
- [ ] `ctest` + `run_regression_tests.sh` 通过

### 8.7 提交

- D1: 删除 deopt 启发式 + 加强编译期 tag
- D2: 修复/消除 DEOPT_LOC_SPILL
- D3: 清理死代码（Dead #1 + Dead #2）

---

## 9. 工作流 E：x86-64 边界治理（P1 · Bug #1）

### 9.1 问题

- `CMakeLists.txt` 在 x86_64 上默认 `XRAY_HAS_JIT=1`，但后端标注 `Phase F.4.2`
- offset 语义 bug：安装端统一 `* 4`，但 x64 返回字节偏移
- 无 fast entry、无 OSR、无 CALL_KNOWN
- 无端到端测试

### 9.2 E-1：统一 offset 为字节偏移

**决策**：`XirCodegenResult` 中所有 offset 字段统一为**字节偏移**。

修改清单：

| 文件 | 修改 |
|------|------|
| `xir_codegen.c` (ARM64) | `result.fast_entry_offset = ctx.fast_entry_offset * 4` |
| `xir_codegen.c` (ARM64) | `result.resume_entry_offset = ctx.resume_entry_offset * 4` |
| `xir_codegen.h` | 注释更新：`byte offset from code start` |
| `xir_jit.c:510-512` | 删除 `* 4`：`proto->jit_fast_entry = (char *)res.code + res.fast_entry_offset` |
| `xjit_compile_queue.c:130-132` | 删除 `* 4` |
| `xir_jit_debug.c:80` | 删除 `* 4` |
| `xir_codegen_x64.c` | 无变化（已是字节偏移） |

不在安装端 `#if defined(__aarch64__)` 分支，而是在生成端统一语义。安装端永远不需要知道目标架构的指令宽度。

### 9.3 E-2：x86-64 降级为显式 opt-in

在 `CMakeLists.txt` 中：

- x86-64 上 `XRAY_HAS_JIT` 默认为 0
- 新增 `-DXRAY_JIT_X64_EXPERIMENTAL=ON` 开关显式启用
- 启用时在构建日志输出 `WARNING: x86-64 JIT is experimental`

### 9.4 E-3：x86-64 smoke 基线

至少新增：

- 1 个算术/控制流用例
- 1 个函数调用用例
- 1 个 deopt 用例

suspend/resume 若当前不支持则明确 skip。

### 9.5 验收

- [ ] offset 统一为字节偏移，安装端无 `* 4`
- [ ] x86-64 JIT 非默认开启
- [ ] 至少一组 x86-64 E2E smoke

### 9.6 提交

- E1: 统一 offset 语义
- E2: CMake 降级 + smoke 测试

---

## 10. 工作流 F：测试矩阵与文档同步（P1）

### 10.1 测试分层

#### F-1 基础层（保持常绿）

`test_xir` · `test_xir_builder` · `test_xir_pass` · `test_code_alloc` · `test_arm64_emit` · `test_x64_emit`

#### F-2 恢复 E2E 测试

取消 `tests/unit/CMakeLists.txt:104-105` 的注释，恢复 `test_jit_e2e` 和 `test_aot_e2e`。如果不能通过，修复或标记 `XFAIL`，不继续注释掉。

**原则：注释掉测试 = 隐藏问题。要么修复，要么显式标记失败，不允许静默跳过。**

#### F-3 smoke 回归层

固定 smoke 集合：

- `1600_nested_jit_deopt.xr`
- `0661_jit_cross_boundary.xr`
- `0521_cell_upval.xr`
- 1 个算术/控制流热循环
- 1 个 suspend/resume

#### F-4 全量差分层

`scripts/run_jit_diff_tests.sh`：

- crash / mismatch 分开统计
- known failures 清单有 owner 和更新时间

### 10.2 文档同步

检查并更新：

- `docs/engineering/jit_known_limitations.md`
- `docs/engineering/jit_verifier_framework.md`

要求：

- 过时结论标为 `[OUTDATED]`
- 当前失败项必须可在文档中找到
- 文档与源码/测试现状一致

### 10.3 验收

- [ ] `test_jit_e2e` 恢复到构建
- [ ] JIT smoke 可单独运行
- [ ] 文档与现状不冲突

---

## 11. 每个工作流的统一验证步骤

```bash
# 1. 构建
cmake --build build -j8

# 2. 快验
ctest --output-on-failure --test-dir build

# 3. JIT smoke
./build/xray test --quiet --jit-force tests/regression/16_jit/1600_nested_jit_deopt.xr
./build/xray test --quiet --jit-force tests/regression/06_collections/0661_jit_cross_boundary.xr
./build/xray test --quiet --jit-force tests/regression/05_functions/0521_cell_upval.xr

# 4. 对照
./build/xray test --quiet --no-jit tests/regression/06_collections/0661_jit_cross_boundary.xr
./build/xray test --quiet --no-jit tests/regression/05_functions/0521_cell_upval.xr

# 5. 完整回归
scripts/run_regression_tests.sh
```

涉及 `src/jit` / `src/vm` / `src/coro` 的修改不得省略任何步骤。

---

## 12. 推荐提交切分

| 组 | 内容 | 对应工作流 |
|----|------|------------|
| 1 | 最小复现 + 断言 + smoke 脚本 | A-1, B-1 |
| 2 | 对象/属性链修复 | A-2, A-3 |
| 3 | closure/cell/upvalue 修复 | B-2, B-3 |
| 4 | 统一安装 helper | C |
| 5 | bug 修复 + 死代码清理 | D |
| 6 | x86-64 offset + 降级 + smoke | E |
| 7 | 恢复 E2E 测试 + 文档同步 | F |

禁止混合不同工作流到一次提交。

---

## 13. 明确不做的事

- ❌ 新增大批 opcode 支持
- ❌ 引入新 IR 形态或 Sea of Nodes 重写
- ❌ 大规模 pipeline 重构
- ❌ 在 x86-64 无 smoke 基线前扩 feature

但与旧版计划不同，本轮**允许并鼓励**：

- ✅ 删除死代码和空函数桩
- ✅ 删除不可靠的启发式，替换为更简单的安全默认值
- ✅ 统一两套路径为一套
- ✅ 修改 codegen 输出的 offset 语义（不需要"兼容旧行为"）

---

## 14. 完成标志

当以下条件**全部**成立时，JIT 从"实验性"提升为"稳定可用子集"：

- [ ] `0661` 与 `0521` 修复
- [ ] Bug #1 #2 #3 #4 全部修复
- [ ] Dead #1 #2 清理
- [ ] ARM64 JIT diff 无 crash
- [ ] 安装语义统一为单一路径
- [ ] `test_jit_e2e` 恢复到构建且通过
- [ ] smoke / regression / diff 流程可重复执行
- [ ] x86-64 标注为 experimental 或达到独立准入标准

在此之前，JIT 不作为默认 correctness 承诺后端。
