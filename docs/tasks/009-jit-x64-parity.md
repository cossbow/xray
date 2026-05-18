# JIT x64/ARM64 对齐实施方案

> 日期：2026-04-24
> 状态：Active
> 范围：`src/jit`
> 替代：`jit_x64_arm64_parity_gpt_plan.md`（战略）+ `jit_x64_parity_impl.md`（战术）
> 原则：**不考虑向后兼容性，直接采用最佳设计**

---

## 1. 问题定义

当前主要矛盾不是"x64 缺几个指令"，而是：

**共享 IR 之后的 backend contract 还没有被收敛成单一语义。**

具体表现：
- x64 自标记 `Phase F.4.2 — integer arithmetic + control flow`
- 16 个 JIT opcode 缺失，3 个已知 bug
- x64 无 execute-level 测试，E2E 测试硬编码 ARM64 且被注释掉
- call/deopt/resume/safepoint 路径在两个 backend 中各自定义语义

真正危险的是"共享前半段 + 漂移后半段"——前半段越共享，越容易以为两个 backend 一致，而真正的 bug 都躲在最后几层。

---

## 2. 目标与非目标

**目标**：correctness-first，先做对再做快
1. 消灭 x64 已知 bug 和 crash 路径
2. 补齐全部缺失 opcode（~510 行）
3. x64 拥有 execute-level E2E 测试
4. call/deopt/resume/safepoint 两端语义一致
5. CI 双平台验证

**非目标**（本轮不做）：
- ~~Windows JIT~~
- ~~x64 性能追平 ARM64~~
- ~~统一 codegen 抽象层~~ — 2 个 backend 1 个开发者，强行抽象是负债
- ~~XIR 解释器~~ — parity test suite 给 80% 的价值
- ~~5 个新 contract 头文件~~ — contract 用 `XR_DCHECK` + 注释即可

---

## 3. 基线数据

| 层级 | 行数 |
|------|------|
| 共享层（builder + passes + regalloc + jit + runtime） | ~20,800 |
| ARM64 后端（codegen×3 + encoding + target） | ~6,000 |
| x64 后端（codegen×2 + encoding + target） | ~4,850 |
| **总计** | **~42,600** |

---

## 4. Opcode 覆盖矩阵

图例：✅ 已实现、❌ 缺失、⚠️ 部分实现（有 bug）、🔄 CALL_C 间接

### 4.1 x64 缺失 opcode（16 个，~510 行）

| 优先级 | Opcode | 影响 | 行数 |
|--------|--------|------|------|
| **P0** | `XIR_SAFEPOINT` | 长时间运行无法 GC/preempt | ~15 |
| **P0** | `XIR_BOX_I64/F64` | tagged value 路径不可用 | ~20 |
| **P0** | `XIR_UNBOX_I64/F64` | tagged value 路径不可用 | ~30 |
| **P0** | `XIR_TAG_CHECK` | 类型守卫不可用 | ~20 |
| **P0** | `XIR_TAG_LOAD` | tag 读取不可用 | ~10 |
| **P1** | `XIR_RT_ADD/SUB/MUL/DIV/MOD` | 混合类型运算 | ~200 |
| **P1** | `XIR_RT_UNM` | 混合类型取负 | ~30 |
| **P1** | `XIR_RT_LT/LE/EQ` | 混合类型比较 | ~80 |
| **P2** | `XIR_RT_PRINT` | 打印（可 CALL_C 代理） | ~30 |
| **P2** | `XIR_GUARD_CLASS` | Json shape guard | ~25 |
| **P2** | `XIR_GUARD_KLASS` | class instance guard | ~30 |
| **P3** | `XIR_CALL` (generic) | 泛型调用 | ~20 |

### 4.2 x64 有 bug 的 opcode（3 个）

| Opcode | Bug | 位置 |
|--------|-----|------|
| `XIR_DIV/MOD` | 无除零守卫，裸 IDIV → SIGFPE | `xir_codegen_x64.c:337` |
| `XIR_SHL/SHR` | RCX clobber：rd==RCX 或 rn==RCX 时覆写 | `xir_codegen_x64.c:396` |
| `XIR_CALL_SELF_DIRECT` | `/* TODO: frame smap ptr slot */` | `xir_codegen_x64_call.c:286` |

### 4.3 已对齐的 opcode（完整列表）

**基础算术/逻辑**：ADD/SUB/MUL/NEG/AND/OR/XOR/NOT/EQ/NE/LT/LE/GT/GE ✅
**浮点**：FADD/FSUB/FMUL/FDIV/FNEG/FEQ/FNE/FLT/FLE/I2F/F2I ✅
**常量**：CONST_I64/PTR/F64 ✅
**内存**：LOAD/STORE（全部变体 8/16/32/F32/FIELD/CORO） ✅
**Guard**：GUARD_BOUNDS/TAG/NONNULL/SHAPE/DEOPT ✅
**Call**：CALL_C/C_LEAF/SELF_DIRECT/DIRECT/KNOWN/KNOWN_REG ✅
**Coro/GC**：SUSPEND/ALLOC/CATCH/BARRIER_FWD/BACK ✅
**Misc**：MOV/REDEFINE/NOP/SELECT/SELECT_COND ✅
**Upvalue**：LOAD_UPVAL/STORE_UPVAL 🔄（两端均 CALL_C）

---

## 5. 已知 BUG 详情

### BUG-01: DIV/MOD 除零（`xir_codegen_x64.c:337-361`）

ARM64 `SDIV` 除零返回 0；x64 `IDIV` 除零触发 SIGFPE。

**修复**：IDIV 前加 `TEST divisor, divisor; JZ deopt`（~8 行）

### BUG-02: SHL/SHR 寄存器冲突（`xir_codegen_x64.c:396-412`）

x86-64 移位要求 count 在 CL。当 `rd==RCX` 时 `MOV RCX, rm` 覆写目标；当 `rn==RCX` 时覆写源。

**修复**：3-way swap 用 scratch 寄存器（~12 行 × 2）

### BUG-03: smap ptr slot TODO（`xir_codegen_x64_call.c:~286`）

自调用路径未正确保存/恢复 stack map 指针。ARM64 有对应的 save/restore 到固定帧槽。

**修复**：镜像 ARM64 的 smap slot save/restore（~20 行）

---

## 6. 测试覆盖差距

| 测试 | ARM64 | x64 |
|------|-------|-----|
| 指令编码 | encode + **execute** | encode only |
| E2E codegen | `test_jit_e2e.c`（硬编码 arm64，**被注释**） | ❌ 无 |
| Peephole | ✅ | ❌ |
| Pass/Builder/Liveness | ✅ 共享 | ✅ 共享 |

**关键问题**：x64 从未真正执行过 JIT 生成的代码来验证正确性。

---

## 7. 实施计划（3 个 Phase，~2 周）

### Phase 1: 修 bug + 建测试（3 天）

先修 bug，立刻启用测试——在测试保护下再补 opcode。

| # | 任务 | 文件 | 预估 |
|---|------|------|------|
| 1.1 | 修 DIV/MOD 除零守卫 | `xir_codegen_x64.c` | 0.5h |
| 1.2 | 修 SHL/SHR RCX clobber | `xir_codegen_x64.c` | 1h |
| 1.3 | 修 smap ptr slot TODO | `xir_codegen_x64_call.c` | 2h |
| 1.4 | `test_jit_e2e.c` → `#ifdef` 双后端 | `test_jit_e2e.c` | 0.5h |
| 1.5 | 取消 CMakeLists 注释启用 E2E | `tests/unit/CMakeLists.txt` | 5min |
| 1.6 | `test_x64_emit.c` 添加 execute 测试 | `test_x64_emit.c` | 2h |
| 1.7 | CI 添加 x64 runner | `.github/workflows/ci.yml` | 1h |

**验收**：ctest 全通过，x64 有 execute-level 测试。

### Phase 2: 补齐全部缺失 opcode（5 天）

在测试保护下逐批实现。

| # | 任务 | 文件 | 预估 |
|---|------|------|------|
| 2.1 | 新建 `xir_codegen_x64_mem.c` 骨架 | 新文件 | 1h |
| 2.2 | BOX_I64/F64 + UNBOX_I64/F64 | `xir_codegen_x64.c` | 2h |
| 2.3 | TAG_LOAD + TAG_CHECK | `xir_codegen_x64_mem.c` | 1h |
| 2.4 | SAFEPOINT（guard page poll） | `xir_codegen_x64_mem.c` | 2h |
| 2.5 | RT_ADD/SUB/MUL/DIV/MOD（内联 i64↔f64） | `xir_codegen_x64_mem.c` | 6h |
| 2.6 | RT_UNM | `xir_codegen_x64_mem.c` | 1h |
| 2.7 | RT_LT/LE/EQ | `xir_codegen_x64_mem.c` | 3h |
| 2.8 | RT_PRINT（CALL_C 代理） | `xir_codegen_x64_mem.c` | 1h |
| 2.9 | GUARD_CLASS + GUARD_KLASS | `xir_codegen_x64_mem.c` | 2h |
| 2.10 | CALL generic（降级 CALL_C） | `xir_codegen_x64_call.c` | 1h |

**SAFEPOINT 设计决策**：从 jit_ctx 加载 guard page 地址（不占固定寄存器）。x64 只有 16 GP，寄存器压力大，不值得为 safepoint 预留。

**文件结构对齐 ARM64**：
```
xir_codegen_x64.c      — 算术/逻辑/比较/FP/转换/MOV/BR/BOX/UNBOX
xir_codegen_x64_call.c — 所有 CALL 类
xir_codegen_x64_mem.c  — RT_*/TAG_*/GUARD_CLASS/KLASS/SAFEPOINT（新建）
```

**验收**：所有 16 个缺失 opcode 实现，ctest + E2E 全通过。

### Phase 3: 语义对齐审计（2 天）

对高风险路径做一次系统审计，确保两端语义一致。

| # | 任务 | 审计范围 |
|---|------|---------|
| 3.1 | call result 语义 | 返回值 payload/tag 约定 |
| 3.2 | deopt 语义 | deopt id / marker / metadata 写入顺序 |
| 3.3 | suspend/resume | live state 保存范围、resume 入口恢复顺序 |
| 3.4 | safepoint/stack-map | active smap 更新时机、frame ptr 写入 |
| 3.5 | entry/fast_entry/resume_entry | 三种入口的语义差异 |

**落地方式**：不建新文件。在代码中用 `XR_DCHECK` 表达 contract，在关键路径添加断言。

**验收**：无 `/* TODO */` 残留在关键路径，所有 CALL 类 opcode 语义一致。

---

## 8. 工程纪律

### 8.1 先有测试，再开快路径

没有 parity case 的路径，不配拥有 fast path。

### 8.2 每个 bug 双落地

每修一个 bug，必须补两类回归：
- 语言级 `.xr` 脚本 case
- IR/backend 级最小 case

### 8.3 Backend 只决定"怎么发射"

Backend 可以不同的：物理寄存器、ABI 映射、指令编码、patch 方式。

Backend **不应自由发明**的：返回 tag 协议、deopt marker 语义、resume 恢复顺序。

### 8.4 必测场景清单

每个高风险 case 至少覆盖 `--no-jit` + `--jit-force` + ARM64 + x64：

- 基础整数/浮点运算、分支与循环
- closure / upvalue / object field
- CALL_C / CALL_KNOWN / CALL_SELF_DIRECT
- deopt recovery、suspend / resume
- GC safepoint、exception 边界

---

## 9. 退出标准

- [x] 16 个缺失 opcode 全部实现 *(2026-04-25: 审计确认全部已实现)*
- [x] 3 个已知 BUG 修复并有回归测试 *(2026-04-25: 审计确认全部已修复)*
- [x] `test_jit_e2e.c` 在 x64 上通过 *(2026-04-25: Rosetta 2 — 69/69 ctest + 280/280 regression)*
- [x] `test_x64_emit.c` 包含 execute-level 测试 *(已由 test_jit_e2e dual-backend 覆盖)*
- [x] CI 在 x64 和 ARM64 两个平台运行 *(2026-04-25: 4 matrix entries + ctest step)*
- [x] 无 `/* TODO */` 残留在 x64 codegen 关键路径 *(2026-04-25: grep 确认无残留)*
- [x] 所有 CALL 类 opcode 在两端语义一致 *(2026-04-25: Phase 3 审计完成)*

### Phase 3 审计发现与修复 (2026-04-25)

| # | 问题 | 严重性 | 修复 |
|---|------|--------|------|
| 3.2a | x64 deopt stub 未复制 spill slots 到 `deopt_spill_save[]` | **HIGH** | 添加 spill copy 循环 (`xir_codegen_x64.c`) |
| 3.5a | x64 resume entry 未存 `smap_ptr` 到 frame | **HIGH** | 添加 smap_ptr store (`xir_codegen_x64.c`) |
| 3.T1 | `test_alloc_inline` heap_buf 未 16KB 对齐 → SIGBUS | **MEDIUM** | 改用 `posix_memalign` + cursor offset (`test_jit_e2e.c`) |

### 后续审计修复 (2026-04-25)

| # | 发现 | 严重性 | 修复 |
|---|------|--------|------|
| F1 | `xir_codegen_x64.c` = 2758 行逼近 3000 限，`xir_codegen_x64_mem.c` 未创建 | **HIGH** | 拆分 tag/guard/RT/safepoint 到 `xir_codegen_x64_mem.c` (431 行)，主文件降至 2402 行 |
| F2 | 文件头 `STATUS: Phase F.4.2` 过时 | **MEDIUM** | 更新为 `Complete — full opcode parity with ARM64 backend` |
| F3 | `test_mod` 无除零边界，无 SHL/SHR 专项测试 | **HIGH** | 新增 `test_div_edge_cases` + `test_shift` (SHL+SHR) |
| F5 | 断言密度 ~110 行/个，低于 50-80 要求 | **MEDIUM** | 补充 XR_DCHECK 至 72/39/84 行/个 |

---

## 附录 A: x64 与 ARM64 ISA 关键差异

| 特性 | ARM64 | x64 | codegen 影响 |
|------|-------|-----|-------------|
| 指令宽度 | 固定 4B | 变长 1-15B | branch patch 用 rel32 |
| 操作数 | 3-operand | 2-operand (dst=src1) | 需要 `ensure_dst` 前置 MOV |
| 除法 | SDIV（无异常） | IDIV RDX:RAX（SIGFPE） | x64 需除零守卫 |
| 移位 | LSL rd, rn, rm | SHL rd, CL | shift amt 必须在 CL |
| FP 取负 | FNEG | XORPD sign bit | 多一条常量 |
| 寄存器 | 31 GP + 32 FP | 16 GP + 16 XMM | x64 压力大 |
| 调用约定 | AAPCS64 (x0-x7) | SysV (rdi,rsi,rdx,rcx,r8,r9) | 参数寄存器不同 |

## 附录 B: 相关文档

| 文档 | 内容 |
|------|------|
| `cross_platform_jit.md` | 跨平台工程纪律（三层防御体系） |
| `jit_stabilization_plan.md` | JIT 整体稳定化 |
| `jit_known_limitations.md` | 已知限制与 workaround |
| `jit_vm_boundary.md` | JIT 与 VM 边界契约 |
