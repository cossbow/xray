# 073 — xisa codegen DSL 实施计划

> **作者**: Cascade (pair-programming with Xinglei Xu)
> **日期**: 2026-05-11
> **状态**: planned
> **配套设计 spec**: [`../engineering/xisa_design.md`](../engineering/xisa_design.md)
> **直接动机**: commit `8b06997` 修复的 x64 RT-opcode 漏 emit bug（Win11 上 `STATUS_HEAP_CORRUPTION`）
> **generator 决策**: **C host tool**，不使用 xray 自举
> **目标后端**: x64 / arm64 / riscv64

> 本文档只回答"怎么做"和"何时做"。设计决策、DSL 语法、encoding 元件清单、emit helper C 接口在 spec 中定义，不在此重复。

---

## 0. 总览

本文档把 xisa 的落地路线收敛为：

```text
encoding-first → driver-preserving → subset-first → runtime-sensitive hardening
```

四条核心决策（generator 用 C / v1 只做 encoding + golden / 保留 codegen driver / subset-first）见 [`xisa_design.md` §2](../engineering/xisa_design.md#2-核心决策)。本文档以这些决策为前提，描述阶段、退出标准和时间估算。

### 0.1 阶段总览

| 阶段 | 标题 | 输出 | 估算 |
|---|---|---|---|
| S0 | schema fixture + baseline | 10-20 条 mcinsn 声明、代码量/测试/build 基线 | 3-5 天 |
| S1 | C generator + golden | C parser/typecheck/emit generator、build-time golden | 2-3 周 |
| S2 | x64 pure emit pilot | x64 ARITH/MOV/CMP dual-emit byte-equal | 1 周 |
| S3 | branch / patch contract | label/reloc/short-long branch 契约和测试 | 1 周 |
| S4 | x64 encoding coverage | x64 scalar/mem/control encoding helper 覆盖，driver 保留 | 2 周 |
| S5 | scalar interpreter + differential | pure subset interpreter 与 differential target | 1 周 |
| S6 | arm64 encoding coverage | arm64 scalar/mem/control encoding helper 覆盖 | 2 周 |
| S7 | RISC-V subset bring-up | RV64 scalar/mem/branch/call subset 可跑 JIT unit tests | 2-3 周 |
| S8 | runtime-sensitive hardening | call / deopt / OSR / safepoint / suspend-resume 各自独立 DoD | 4-6 周 |
| S9 | linemap + fuzz automation | generated mcinsn linemap、nightly fuzz、shrinker | 1-2 周 |

每个阶段都必须独立可回滚。任何阶段发现 bug，按 root cause 修复并补测试，不能把 generator 调整成匹配已知错误。

---

## 1. 目标 / 非目标

### 1.1 目标

| 目标 | 验证方法 |
|---|---|
| machine encoding bug 前移到 build-time | golden byte-equal |
| x64 / arm64 纯 emit helper 可由 `.isa` 生成 | dual-emit + ctest |
| branch/patch 契约明确 | patch unit tests |
| RISC-V 后端先完成可执行 subset | QEMU / target unit tests |
| 不引入构建环 | C host generator clean build |
| 不破坏现有 JIT | 每次代码接入后 ctest + regression |

### 1.2 非目标

整体非目标见 [`xisa_design.md` §0 / §2.2 / §2.3 / §2.4](../engineering/xisa_design.md#2-核心决策)（不替代 `Xm` 数据结构、不生成 `xm_ops.h`、不删 driver、不做全 op interpreter、不生成 AOT C backend 等）。本 task 额外补充的实施侧非目标：

| 非目标 | 理由 |
|---|---|
| 用 xray 写 generator | bootstrap / cross-compile 成本高，工程收益不足 |
| SIMD / VEX / EVEX | v1 无需求，后续扩展 |
| 自动 branch relaxation | 见 S3，driver 自己选择 short / long branch；generator 只算 reach |
| Win64 `.pdata` / `.xdata` unwind 信息 | 与 encoding 正交，单独 follow-up |

---

## 2. 基线

> 代码量和测试数字取自 2026-05-11（本文初稿）；S0 启动时重采一次并在本节补录。后续阶段只记“不可退行 baseline”，不逐阶段重复贴数字。

### 2.1 codegen 代码量（2026-05-11）

```text
x64 backend:
  xm_codegen_x64.c            945
  xm_codegen_x64_call.c       922
  xm_codegen_x64_ins.c       1631
  xm_codegen_x64_mem.c        714
  xm_codegen_x64_osr.c        392
  xm_codegen_x64_patch.c      180
  xm_codegen_x64_stub.c       367
  xm_x64.c                    622
  xm_target_x64.c             121
  total                      5894 行

arm64 backend:
  xm_codegen.c               1621
  xm_codegen_call.c          1003
  xm_codegen_ins.c            434
  xm_codegen_mem.c           1301
  xm_codegen_stub.c           784
  xm_arm64.c                  605
  xm_target_arm64.c            66
  total                      5814 行
```

### 2.2 测试基线（2026-05-11）

| 平台 | 测试 | 通过数 |
|---|---|---|
| macOS Debug | ctest | 113/113 |
| macOS Debug | regression | 293/293（含 1207_gc_stress） |
| Win11 Release | ctest | 105/105 |
| Win11 Release | regression | 291/293（已知 2 个非 xisa bug） |

阶段退出时不得降低基线。若发现新的真实 bug，必须按 bug 修复纪律处理（修 root cause，不让 generator 适配错误）。

### 2.3 build 时间基线

| 配置 | 时间 |
|---|---|
| macOS Debug 全量 build | TBD（S0 实测） |
| macOS Debug 增量 build | TBD（S0 实测） |

v1 目标：generator + golden 带来的增量 build 时间 ≤ 2 秒。

---

## 3. 实施阶段

### S0 — schema fixture + baseline

**目标**：先验证 DSL 表达力和当前基线，不写 generator 逻辑。

**任务**：

- [ ] 创建目录：`xisa/backends/{x64,arm64,riscv64}/`、`xisa/subsets/`、`tools/xisagen/`。
- [ ] 手写 10-20 条 mcinsn 声明，覆盖：
  - x64 REX / ModR/M / SIB / imm64 / `rsp/rbp/r12/r13`。
  - arm64 fixed-width bitfield / `sp/xzr`。
  - RISC-V R/I/S/B/U/J 基础格式。
- [ ] 每条 mcinsn 写至少 3 个 golden case。
- [ ] 不写完整 lowering；只写 `scalar.isa` fixtures。
- [ ] 实测并填入 §2.1 / §2.2 / §2.3 基线数字。

**退出标准**（全部满足才进 S1）：

- >= 12 条 mcinsn fixture 能用 spec §4 定义的 DSL 表达，同时覆盖三 arch 的“普通 / 扩展 / 特殊”三类寄存器。
- 每条 fixture 的 expected bytes 至少可用**两种互独立手段**交叉验证（现有手写 emit · LLVM `llvm-mc` · GNU `as` · binutils objdump 任二），不能只靠人脑读 spec。
- 发现任何 DSL 表达力缺口 → **先修 spec §4 并重跑 fixture**，才能推进 S1；spec 未同步不允许进 S1。
- 未覆盖的 mcinsn 列表写入本 task 附录（或 `docs/known_bugs.md` 未实现区），S1 不会那里取走。
- baseline 表填实。

**回滚锚点**：`xisa-s0-schema-fixture`

---

### S1 — C generator + golden

**目标**：用 C 写出最小可用 generator：parse/typecheck `.isa`，生成 emit helper、attribute header、golden tests。

**任务**：

- [ ] `tools/xisagen/xisa_lex.c`：S-expression lexer（行号 / 列号严格跟踪）。
- [ ] `tools/xisagen/xisa_parse.c`：parser，产生 AST 节点。
- [ ] `tools/xisagen/xisa_ast.c`：AST / symbol table / arena。
- [ ] `tools/xisagen/xisa_typecheck.c`：operand / constraint / encoding / golden 静态验证。
- [ ] `tools/xisagen/xisa_emit_c.c`：生成 `xisa_emit_<arch>.h` + `xisa_mcattr_<arch>.h`。
- [ ] `tools/xisagen/xisa_golden.c`：生成 / 执行 golden tests。
- [ ] `tools/xisagen/xisa_host.c`：host allocator / diagnostics / file IO（仅依赖 host C 标准库）。
- [ ] **generator 自身单测**：`tests/unit/xisagen/` 覆盖 lex 边界 / parse 错误恢复 / typecheck 拒绝路径 / encode 位拼接。
- [ ] CMake：先 build host `xisagen`，再生成 `build/generated/xisa/*`；Windows MSVC 与 Unix 同样运行。
- [ ] 生成物不接入 JIT driver，仅作为独立 test target。

**输出**：

```text
build/generated/xisa/
├── xisa_emit_x64.h
├── xisa_emit_arm64.h
├── xisa_emit_riscv64.h
├── xisa_mcattr_x64.h
├── xisa_mcattr_arm64.h
├── xisa_mcattr_riscv64.h
└── xisa_golden_tests.c
```

**退出标准**：

- clean build 在 macOS / Linux / Windows MSVC 三个环境不需要预先存在 xray binary。
- generator 单测 100% pass，lex/parse/typecheck/encode 四块各至少 5 个错误路径被验证会 fail-fast。
- 故意改错 fixture 中一条 encoding，build/test fail 且诊断信息同时包含：`.isa` 路径 · 行号 · 列号 · mcinsn 名 · 期望 vs 实际字节。
- golden 全过。
- 现有 JIT 未接入、ctest / regression 不退行（跟 §2 基线对齐）。

**回滚锚点**：`xisa-s1-c-generator-golden`

---

### S2 — x64 pure emit pilot

**目标**：在 x64 后端对纯 emit 指令接入 generated helper，使用 dual-emit 验证。

**范围**：

- integer add/sub/and/or/xor
- mov reg/reg、mov imm
- cmp/test
- simple load/store addressing forms

不碰：branch、call、deopt、OSR、safepoint、stub。

**dual-emit 实现**：

- 在 `xm_codegen_x64*.c` 里保留手写 emit 常规路径；新增 `xm_codegen_x64_dual.c`（仅 Debug + `XR_ENABLE_XISA_DUAL_EMIT=1` 时编译）提供 `x64_emit_dual()` 包装函数。
- 该包装函数同时调用手写 emit 与 generated emit，比较字节 buffer；不一致 → `XR_CHECK` fail 且在错误信息里输出：XmIns 上下文 · operand values · 手写 bytes · generated bytes · 差异 offset。
- 默认 Release 不启用，避免 performance impact；CI 在 macOS / Linux Debug + `XR_ENABLE_XISA_DUAL_EMIT=1` 上跑 ctest 作为回归。

**任务**：

- [ ] 为 x64 pure emit 指令补 `.isa` 和 golden。
- [ ] 实现上述 `x64_emit_dual()`。
- [ ] 将手写 emit call site 改为 `x64_emit_dual()`（仅 pure emit 范围，branch/call/stub 不动）。
- [ ] 覆盖特殊寄存器：`rsp / rbp / r12 / r13 / r8-r15`。
- [ ] 以 `XRAY_JIT_FORCE=1` 跑 `tests/unit/jit/test_jit_e2e` 与 regression 验证高压运行下 dual-emit 不产生分歧。

**退出标准**：

- macOS / Linux Debug + `XR_ENABLE_XISA_DUAL_EMIT=1` 下，ctest + regression 全过，且 dual-emit 0 分歧。
- pure emit 实际调用次数 ≥ 1e6。
- 发现的手写 bug 已修复 root cause，并补入 golden 防回归。
- Release 构建仍能 disable `XR_ENABLE_XISA_DUAL_EMIT`，不引入运行时开销。

**回滚锚点**：`xisa-s2-x64-pure-dualemit`

---

### S3 — branch / patch contract

**目标**：在迁移 control flow 前，先定义 branch/patch 的职责边界。

**任务**：

- [ ] 定义 branch mcinsn 的 `:flags branch`、reach、fixed/variable length 字段。
- [ ] 明确 driver 负责 block layout、target offset、short/long branch 选择。
- [ ] 生成 branch encoding helper，但不自动做 relaxation。
- [ ] 新增 patch unit tests：
  - x64 rel8 / rel32。
  - arm64 conditional / unconditional branch reach。
  - RISC-V B/J immediate split。
- [ ] 明确 trampoline / island 不在 S3 实现。

**退出标准**：

- branch encoding helper golden 全过。
- patch tests 覆盖 forward/backward、边界 reach、越界 fail-fast。
- driver 与 xisa 职责写入设计 spec。

**回滚锚点**：`xisa-s3-branch-patch-contract`

---

### S4 — x64 encoding coverage

**目标**：x64 scalar/mem/control 的机器编码由 xisa helper 覆盖，driver 仍保留。

**任务**：

- [ ] 迁移 x64 scalar/mem/control machine encoding helper。
- [ ] 删除被 generated helper 完全替代的局部 encoding helper。
- [ ] 保留 `xm_codegen_x64_*` driver 文件，driver 继续负责 ctx、patch、OSR、deopt。
- [ ] 不生成完整 lowering switch。
- [ ] 对每类迁移执行 dual-emit 验证。

**Coverage 度量定义**（需用脚本可重现）：

- 分母：在 S4 范围内（x64 scalar / mem / control）现有手写 emit 在 codegen 代码中的 **call site 总数** — 可用 `scripts/xisa_coverage_x64.sh` 扫 `xm_codegen_x64*.c` 中 emit helper 调用。
- 分子：已走 `x64_emit_dual()` 或已完全迁移到 generated helper 的 call site 数。
- coverage 需随 S4 commit 跟踪，CI 报表；仅以 line / function 覆盖不足以裁定。

**退出标准**：

- 上述定义下覆盖率 ≥ 80%，下限 ≥ 70% 且 mem / arith / cmp 三类各自 ≥ 70%（防止某一类全未迁移）。
- ctest + regression 通过。
- 24h x64 JIT stress 无新 bug；ASAN 跑一轮 regression 无新 leak / heap-buffer-overflow。
- 手写 machine byte 拼装明显减少，但不要求删除整个 driver。

**回滚锚点**：`xisa-s4-x64-encoding-coverage`

---

### S5 — scalar interpreter + differential

**目标**：只对 pure scalar subset 建 reference interpreter 和 differential test。

**任务**：

- [ ] 定义 `xisa/subsets/scalar.isa` 的可解释 op 集。
- [ ] generator 输出 scalar interpreter 或手写最小 interpreter。
- [ ] random scalar program generator：只生成 arithmetic/compare/bitwise/select。
- [ ] ctest 新增 `xisa_scalar_diff_test`。
- [ ] 每次 PR 跑固定 seed；nightly 跑更多 seed。

**退出标准**：

- scalar differential 1 万程序 0 分歧。
- 故意改错一个 pure emit helper，可被 golden 或 differential 捕获。
- runtime-sensitive op 明确标为 non-interpretable，不进入 scalar diff。

**回滚锚点**：`xisa-s5-scalar-diff`

---

### S6 — arm64 encoding coverage

**目标**：按 x64 已验证流程迁移 arm64 scalar/mem/control encoding。

**任务**：

- [ ] 为 arm64 scalar/mem/control 补 `.isa`。
- [ ] bitfield helper 单独 golden。
- [ ] dual-emit 对比现有 arm64 emit helper。
- [ ] 保留 arm64 driver、patch、OSR、deopt 逻辑。
- [ ] 与 S5 scalar differential 联动验证行为一致。

**退出标准**：

- arm64 scalar/mem/control generated helper 覆盖率 ≥ 80%。
- ctest + regression 通过。
- arm64 JIT stress 无新 bug。
- scalar differential x64/arm64/interpreter 0 分歧。

**回滚锚点**：`xisa-s6-arm64-encoding-coverage`

---

### S7 — RISC-V subset bring-up

**目标**：用 xisa 写 RISC-V subset backend，先跑通最小 JIT 子集。

**范围**：

- integer arithmetic
- load/store
- branch/jump
- simple call/return
- basic prologue/epilogue

暂不要求：

- OSR
- deopt
- coroutine suspend/resume
- full safepoint protocol
- full regression parity
- RISC-V compressed instruction extension

**任务**：

- [ ] `xisa/backends/riscv64/isa.def` 覆盖 RV64 base scalar subset。
- [ ] 写 RISC-V driver skeleton。
- [ ] QEMU 上跑 JIT unit subset。
- [ ] scalar differential 加入 RISC-V。

**退出标准**：

- QEMU RISC-V 上 JIT scalar unit tests 通过。
- scalar differential x64/arm64/riscv64/interpreter 0 分歧。
- 记录未覆盖 runtime-sensitive 能力，不把未实现功能记为 bug。

**回滚锚点**：`xisa-s7-riscv-subset`

---

### S8 — runtime-sensitive hardening

**目标**：逐项处理 call / deopt / OSR / safepoint / suspend-resume，不把它们混入 pure emit 迁移。

**5 个独立子项**（可以任意顺序交付，但每个都需独立 DoD）：

#### S8.1 call ABI parity

- [ ] x64 / arm64 各自补 `call`、`call-known`、`call-c-helper`、`call-self-direct` 三类 parity test。
- [ ] 验证 argument / return / live-across-call 在两 backend 上行为一致。
- **DoD**：4 类 call × 2 backend 全8 个子集 0 分歧。

#### S8.2 safepoint stack map

- [ ] 新增 metadata verifier：对每个 JIT 导出的 stack map，验证 slot ID、spill offset、PC range 与现有 driver 出口一致。
- [ ] 补 GC stress 下 safepoint poll 不丢根、不误该根的回归。
- **DoD**：`scripts/run_regression_tests.sh` + GC stress 运行 30 分钟无新 leak / heap-buffer-overflow / use-after-free。

#### S8.3 deopt spill/register reconstruction

- [ ] 补 deopt parity test：同一 deopt 点在 x64 / arm64 上重建的 VM 寄存器状态 byte-equal。
- [ ] 覆盖 spill 与 register live 混合场景、同一函数多点 deopt、deopt + post-call resume。
- **DoD**：两 backend 上同一输入的 deopt observable state 一致；不出现读未初始化 spill slot 的回归。

#### S8.4 OSR entry

- [ ] 复用 `test_osr_entry_pressure` 框架拓展多点 OSR 、多参 OSR 场景。
- [ ] 验证 OSR 重入不依赖 frame 残留、不读未初始化的 spill-only vreg。
- **DoD**：x64 与 arm64 OSR 重入 ≥ 4 轮不退行，多参 / spill-pressure 场景 byte-equal observable state。

#### S8.5 suspend / resume

- [ ] 补 coroutine suspend / resume parity test：同一协程在两 backend 上调度轮序 · yield 点 · GC 根 · channel 状态 一致。
- [ ] 验证 `xchannel` / `xcoro_suspend` resume 在 x64 上不产生 Win11 `STATUS_HEAP_CORRUPTION`（S0 引入动机同类 bug 的重现验证）。
- **DoD**：`scripts/run_regression_tests.sh` 覆盖的协程用例在两 backend Debug + Release 运行不退行；Win11 Release 上 `1115_cancel.xr` · `1109_await_any.xr` · `1127_coro_priority.xr` · `1128_yield.xr` 无 `STATUS_HEAP_CORRUPTION` 及类似 heap 错误。

**总阶段退出标准**：

- S8.1 - S8.5 五个子项 DoD 全部满足。
- RISC-V 明确记录 runtime parity 缺口与后续计划（不作为退出阻塞项）。
- 任何期间发现的真实 bug 都已修 root cause 并补回归。

**回滚锚点**：`xisa-s8-runtime-hardening`

---

### S9 — linemap + fuzz automation

**目标**：扩展调试与持续验证。

**任务**：

- [ ] generated mcinsn linemap：offset → `.isa` line。
- [ ] driver synthetic region：prologue/stub/alignment/patch island 标记为 synthetic。
- [ ] nightly fuzz：scalar + memory subset。
- [ ] shrinker：失败程序自动缩小。
- [ ] CI 指标：golden count、generated helper 覆盖率、diff pass count。

**退出标准**：

- generated mcinsn 100% 可反查 `.isa` 行。
- driver synthetic region 不误报为 `.isa` 行。
- nightly 连续 4 周 0 分歧。

**回滚锚点**：`xisa-s9-linemap-fuzz`

---

## 4. 关键 milestone

| Milestone | 何时 | 评估什么 | 决策点 |
|---|---|---|---|
| M1 schema 表达力 | S0 | 10-20 条 mcinsn 能否表达？ | 不能 → 修 spec |
| M2 C generator 可行 | S1 | clean build + golden 是否稳定？ | 不稳 → 缩小 DSL |
| M3 x64 pure emit | S2 | byte-equal 是否 0 分歧？ | 有分歧 → 修 root cause |
| M4 branch/patch 契约 | S3 | control flow 是否可安全接入？ | 不清晰 → 禁止迁移 branch |
| M5 双后端 scalar | S6 | x64/arm64 scalar 是否一致？ | 不一致 → 修 schema 或后端 |
| M6 RISC-V subset | S7 | declarative subset 是否能跑？ | 失败 → 复盘 xisa 价值 |
| M7 runtime parity | S8 | deopt/OSR/safepoint 是否安全？ | 不达标 → 禁止推广到 full JIT |

---

## 5. 风险登记册

| ID | 风险 | 概率 | 影响 | 应对 |
|---|---|---|---|---|
| R1 | DSL 表达力不足 | 中 | 中 | S0 用困难指令提前暴露 |
| R2 | generator C 代码膨胀 | 中 | 中 | v1 只做 encoding/golden，不做 full lowering |
| R3 | dual-emit 暴露手写 bug | 高 | 中 | 先修 root cause，加 golden，再迁移 |
| R4 | branch relaxation 复杂 | 高 | 高 | 单独 S3，不混入 ARITH 迁移 |
| R5 | runtime-sensitive path 不能声明式表达 | 高 | 中 | driver 保持主导，xisa 只生成底层 helper |
| R6 | RISC-V QEMU 与硬件差异 | 低 | 中 | subset bring-up 先以 QEMU 为 baseline |
| R7 | 构建生成物依赖顺序错误 | 中 | 高 | C host tool，不链接 xray_core |
| R8 | 全量目标膨胀 | 中 | 高 | 每阶段只交付一个可验证能力 |
| R9 | DSL 错误诊断质量差，写 `.isa` 时调试成本超过手写 emit | 中 | 中 | S1 退出标准要求行号 + 列号 + mcinsn 名 + 期望/实际字节四要素；后续可加 suggested fix |
| R10 | Windows MSVC 上 host tool 编译 / 路径 / 换行差异破坏 generator | 中 | 高 | S1 退出标准要求 macOS / Linux / Windows MSVC 三平台 clean build；CMake 不使用 unix-only flag |
| R11 | generator 自身回归（lex/parse/typecheck/encode）随 DSL 演进破坏 | 中 | 高 | S1 强制 `tests/unit/xisagen/` 单测，每块 ≥ 5 错误路径；DSL 改动必须同步加 fixture |

---

## 6. 资源与时间估算

| 阶段 | 工作量 | 备注 |
|---|---|---|
| S0 | 3-5 天 | schema fixture 与基线 |
| S1 | 2-3 周 | C generator + golden + generator 自身单测 |
| S2 | 1 周 | x64 pure emit pilot |
| S3 | 1 周 | branch/patch contract |
| S4 | 2 周 | x64 encoding coverage |
| S5 | 1 周 | scalar differential |
| S6 | 2 周 | arm64 encoding coverage |
| S7 | 2-3 周 | RISC-V subset |
| S8 | 4-6 周 | runtime-sensitive hardening（5 个子项独立 DoD） |
| S9 | 1-2 周 | linemap + fuzz |

总计不是线性承诺；S0-S2 已经有独立价值，S7 之后是扩展价值。

---

## 7. 完成判定

### 7.1 v1 完成

- [ ] C generator clean build 稳定。
- [ ] x64 / arm64 scalar/mem 基础指令由 generated helper 覆盖。
- [ ] golden 全过。
- [ ] x64 pure emit dual-emit 0 分歧。
- [ ] scalar differential 持续 0 分歧。
- [ ] ctest + regression 不退行。

### 7.2 长期完成

- [ ] 手写 machine byte encoding 基本从 x64 / arm64 后端移除。
- [ ] RISC-V subset backend 可在 QEMU 跑 JIT unit tests。
- [ ] runtime-sensitive parity tests 全过。
- [ ] generated mcinsn linemap 可用。
- [ ] nightly fuzz 自动化稳定。

---

## 8. 暂停 / 失败场景

- **S1 失败**：C generator 复杂度超过收益，保留 `.isa` fixture 与 golden 思路，回到手写 emit + 离线检查。
- **S2 失败**：不进入 wider migration；先修所有 byte-equal 分歧的 root cause。
- **S3 失败**：禁止迁移 branch/control flow，只保留 pure emit generated helper。
- **S7 失败**：RISC-V subset 未证明价值，暂停 RISC-V，继续把 xisa 用于现有双后端 encoding hardening。
- **S8 失败**：runtime-sensitive path 保持 driver 手写，不强行声明式化。

暂停不等于失败；只要 S1/S2 成立，xisa 已经提供 machine encoding golden 和纯 emit hardening 价值。

---

## 9. 与其他 task 的关系

| 关联 task | 关系 |
|---|---|
| [`008-jit-multi-backend.md`](008-jit-multi-backend.md) | xisa 在平台相关层内部演进；不引入统一 codegen abstraction |
| [`009-jit-x64-parity.md`](009-jit-x64-parity.md) | x64 与 arm64 行为对等是 xisa 的输入 baseline |
| [`068-m4-jit-from-xir.md`](068-m4-jit-from-xir.md) | Xi IR 直接驱动 JIT（取代原 `xir_builder` 从 bytecode 重建 SSA）与 xisa 正交：IR 层 vs machine encoding 层。源码里原 XIR 已 rename 为 `Xm`，仅 task 文件名保留历史称呼 |
| [`070-regression-bug-triage.md`](070-regression-bug-triage.md) | codegen 类 bug 可转成 golden / differential 固定样例 |

---

## 10. 修订历史

| 日期 | 改动 | 作者 |
|---|---|---|
| 2026-05-11 | 初稿（配套 xisa_design.md 同日发布） | Cascade + xingleixu |
| 2026-05-11 | 改为 C generator、encoding-first、driver-preserving、subset-first 实施路线 | Cascade + xingleixu |
| 2026-05-11 | 细化 S0/S1/S2/S4/S8 退出标准与 DoD；S1/S8 时间估算上调；补 R9/R10/R11 风险；§0/§1.2 决策列表收敛为 spec 引用；mcattr header 改为 per-arch | Cascade + xingleixu |
