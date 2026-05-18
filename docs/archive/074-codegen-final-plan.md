# 074 — JIT codegen 最终方案：xisa 主导的生成式编码与运行时契约加固

> **作者**: Cascade (pair-programming with Xinglei Xu)
> **日期**: 2026-05-12
> **状态**: planned
> **定位**: 取代 073 的两条分叉路线，作为后续 codegen / xisa / multi-backend 工作的唯一实施依据
> **核心原则**: Xray 没有外部用户，不做向后兼容，不保留旧接口，不做兼容层；每个阶段直接采用最佳设计

---

## 0. 最终决策

### 0.1 选择

最终方案采用：

```text
Xm contract table
    ↓
xisa machine instruction schema
    ↓
C host generator
    ↓
generated emit helpers + generated mcattr + generated disassembler + generated golden tests
    ↓
architecture-specific runtime driver
    ↓
code buffer + stack maps + deopt metadata + OSR metadata
```

简写为：

```text
xisa as machine-encoding truth source
+ generated verification surface
+ handwritten runtime-sensitive driver
+ hard fail-fast contracts
+ differential / parity / fuzz validation
```

### 0.2 为什么不是单纯 hardening

单纯 hardening 能把手写路线做得可靠，但仍然保留多个手写真相源：

- x64 手写 encoding
- arm64 手写 encoding
- 未来 riscv64 手写 encoding
- 手写 disassembler
- 手写 opcode coverage
- 手写 effect flag 约定

这会持续制造同步成本。理想状态下，Xray 应该减少重复真相源，而不是只给重复真相源补测试。

### 0.3 为什么不是 archive 里的 xisa 原方案

archive 里的 xisa 方案方向正确，但仍然偏保守：

- 不生成 `XmOp` 契约表
- 不把 disassembler 纳入 generator 输出
- 对旧手写 helper 迁移过于温和
- runtime-sensitive hardening 覆盖不完整
- effect flag / pointer hygiene 没有成为核心验收项

本方案保留 xisa 的最佳核心：**C host generator + S-expression DSL + encoding-first + driver-preserving**，同时吸收 hardening 方案的全部工程纪律，并按 Xray 当前原则移除兼容层。

### 0.4 不兼容原则

本方案执行以下硬规则：

| 规则 | 含义 |
|---|---|
| 不保留旧接口 | 被 generated helper 替换的手写 emit helper 必须删除 |
| 不做兼容层 | 不保留 `old_emit -> new_emit` 或 `new_emit -> old_emit` 包装 |
| 不长期 dual-emit | dual-emit 只允许作为测试或迁移探针，阶段结束前必须删除生产路径中的旧分支 |
| 不允许 silent fallback | 未处理 opcode / helper / mcinsn 必须 hard fail |
| 不允许 raw helper pointer 作为语义来源 | helper 调用必须使用稳定 helper id 和显式 effect contract |
| 不允许手写裸字节 | xisa 覆盖的机器指令不得在 driver 中手写 byte / bitfield |
| 不允许生成物进 git | generated headers / tests 只存在于 build 目录 |
| 不为旧测试让路 | 发现旧测试依赖错误行为时修测试和 root cause，不让新设计适配错误 |

---

## 1. 架构目标

### 1.1 长期目标

| 目标 | 最终状态 |
|---|---|
| 机器编码真相源 | `.isa` 是唯一真相源 |
| IR opcode 契约 | `XmOp` 由单一契约表生成 enum / metadata / verifier |
| emit helper | x64 / arm64 / riscv64 均由 generator 输出 typed helper |
| disassembler | 由 xisa schema 生成 Xray emit 子集的 disassembler |
| runtime driver | 仍手写，但只负责 frame / ABI / metadata / patch / runtime protocol |
| helper effects | helper registry 是唯一 effect source |
| backend parity | call / deopt / OSR / safepoint / suspend 全部有双后端对等测试 |
| fuzz | generated encoding / driver invariant / runtime pointer hygiene 长期 fuzz |

### 1.2 分层边界

```text
Xi IR
  ↓
Xm IR
  - generated enum / name / arity / effect metadata
  - verifier checks all op contracts
  ↓
backend driver
  - lowering decisions
  - frame layout
  - register ABI moves
  - stack maps
  - deopt snapshots
  - OSR entry
  - suspend / resume
  - patch records
  ↓
xisa generated helpers
  - machine instruction encoding
  - operand constraints
  - immediate range checks
  - min/max bytes
  - generated disassembly
  ↓
code buffer
```

### 1.3 xisa 接管什么

| 领域 | 接管内容 |
|---|---|
| encoding | byte / bitfield / prefix / ModR/M / SIB / immediate split |
| constraints | register class、特殊寄存器、立即数范围、alignment |
| attributes | min bytes、max bytes、fixed width、branch、call、patchable |
| golden | 每条 mcinsn 的 expected bytes 和 expected asm |
| disasm | Xray emit 子集的 machine bytes → asm text |
| diagnostics | `.isa` 路径、行列、mcinsn、operand、expected / actual bytes |

### 1.4 xisa 不接管什么

| 领域 | 保留位置 |
|---|---|
| regalloc | 现有 JIT regalloc |
| frame layout | backend driver |
| ABI shuffle | backend driver |
| call protocol | backend driver + helper registry |
| deopt reconstruction | backend driver + metadata verifier |
| OSR | backend driver |
| safepoint / stack map | backend driver + verifier |
| coroutine suspend / resume | backend driver + runtime tests |
| AOT C backend | 不受 xisa 影响 |

---

## 2. 新的单一真相源

### 2.1 `XmOp` contract table

新增一个低层契约表，作为 `XmOp` 的唯一来源。它不描述机器编码，只描述 low-level IR 的语义壳：

```text
xisa/xm_ops.def
```

每个 op 至少声明：

| 字段 | 用途 |
|---|---|
| name | enum / dump / diagnostics |
| operand arity | verifier 检查 |
| result arity | verifier / codegen 检查 |
| base effects | may_throw / may_gc / may_suspend / writes_memory / reads_memory |
| lowering class | pure / branch / call / runtime / deopt / osr / safepoint / suspend |
| backend coverage | x64 / arm64 / riscv64 handler 必填矩阵 |

生成物：

```text
build/generated/xisa/xm_ops.h
build/generated/xisa/xm_op_table.c
build/generated/xisa/xm_op_verify.c
```

最终要求：

- 手写 `XmOp` enum 删除。
- 手写 op name table 删除。
- 手写 arity table 删除。
- 手写 effect 默认值删除。
- backend handler coverage 由 generated verifier 检查。

### 2.2 helper registry

所有 C helper / runtime bridge 必须通过 registry 声明：

```text
xisa/xm_helpers.def
```

每个 helper 至少声明：

| 字段 | 用途 |
|---|---|
| helper id | stable id，IR 中引用 |
| C symbol | codegen 解析为地址 |
| ABI signature | argument / return convention |
| effects | may_throw / may_gc / may_suspend |
| post-call checks | exception / safepoint / suspend handling |
| pointer trust level | trusted / conservative / external |

最终要求：

- `Xm` IR 不再靠 raw function pointer 推断 helper 语义。
- unknown helper 不允许进入 codegen。
- 默认 effect 不做乐观假设。
- 去掉 effect 必须有测试证明。

### 2.3 machine instruction schema

机器指令继续使用 S-expression `.isa`：

```text
xisa/backends/x64/isa.def
xisa/backends/arm64/isa.def
xisa/backends/riscv64/isa.def
xisa/subsets/scalar.isa
xisa/subsets/memory.isa
```

每条 mcinsn 必须声明：

- operands
- constraints
- flags
- encoding
- min bytes
- max bytes
- golden bytes
- golden asm

与旧设计相比，新增强制项：

| 新强制项 | 原因 |
|---|---|
| golden asm | 支持 generated disasm round-trip |
| external reference tag | 标记 golden 来自 llvm-mc / GNU as / vendor manual 哪一种 |
| decode priority | 解决 x64 编码别名时的 disasm 选择 |
| synthetic flag | NOP / padding / patch island 也纳入 schema |

---

## 3. Generator 设计

### 3.1 实现语言

generator 使用 C 实现，作为 CMake host tool：

```text
tools/xisagen/
```

原因：

- clean build 不依赖已构建的 xray binary。
- cross compile 时 host / target 边界清晰。
- 不引入 Python / Rust / Zig / Go 工具链。
- 可以直接接入 ctest / ASAN。
- 输出 C header 与当前 JIT codegen 自然衔接。

约束：

- 不链接 runtime / VM / JIT。
- 只允许依赖 L0 base 中必要的 allocator / arena / string / diagnostic 工具。
- 不直接调用 `malloc` / `free` / `calloc` / `realloc`。
- 生成物只写入 build 目录。

### 3.2 模块拆分

```text
tools/xisagen/
├── xisa_main.c
├── xisa_lex.c
├── xisa_parse.c
├── xisa_ast.c
├── xisa_typecheck.c
├── xisa_encode.c
├── xisa_decode.c
├── xisa_emit_c.c
├── xisa_emit_disasm.c
├── xisa_emit_ops.c
├── xisa_emit_helpers.c
├── xisa_golden.c
├── xisa_diag.c
└── xisa_host.c
```

### 3.3 输出

```text
build/generated/xisa/
├── xisa_emit_x64.h
├── xisa_emit_arm64.h
├── xisa_emit_riscv64.h
├── xisa_mcattr_x64.h
├── xisa_mcattr_arm64.h
├── xisa_mcattr_riscv64.h
├── xisa_disasm_x64.h
├── xisa_disasm_arm64.h
├── xisa_disasm_riscv64.h
├── xm_ops.h
├── xm_op_table.c
├── xm_op_verify.c
├── xm_helpers.h
├── xm_helper_table.c
├── xisa_linemap.h
└── xisa_golden_tests.c
```

### 3.4 生成物约束

- emit helper 是 `static inline` 或内部 C 函数，不分配内存。
- emit helper 返回实际写入字节数。
- helper 返回 0 表示 operand / cap 不合法，调用方必须 hard fail。
- generated disassembler 只承诺覆盖 Xray schema 中声明的 emitted subset。
- unknown bytes 必须优雅输出 `<unknown>`，debug dump 不允许崩溃。

---

## 4. Driver 最终形态

### 4.1 driver 只做运行时敏感工作

backend driver 保留，但职责收窄：

| driver 职责 | 是否保留 |
|---|---|
| block layout | 保留 |
| register allocation 结果消费 | 保留 |
| frame setup / teardown | 保留 |
| ABI move / call sequence | 保留 |
| stack map emission | 保留 |
| deopt snapshot metadata | 保留 |
| OSR entry / resume entry | 保留 |
| branch target resolution | 保留 |
| raw byte encoding | 删除 |
| instruction bitfield 拼装 | 删除 |
| special register encoding hack | 删除 |
| silent fallback | 删除 |

### 4.2 driver 调用模式

最终 driver 只能通过 generated helper 发出机器指令：

```text
backend driver chooses mcinsn + operands
  ↓
generated helper validates operands and writes bytes
  ↓
driver records metadata / patch / stack map
```

不得出现：

- driver 手写 REX / ModR/M / SIB。
- driver 手写 ARM64 bitfield。
- driver 手写 RISC-V immediate split。
- driver 对 covered instruction 直接写 hex byte。

### 4.3 branch / patch

branch 仍由 driver 决定 layout 和 target offset，但编码由 xisa helper 完成。

最终规则：

- driver 选择 short / long form。
- xisa helper 检查 reach 和 immediate encoding。
- patch record 必须记录 mcinsn id、offset、width、signedness。
- patch 阶段必须重新用 generated range checker 验证 offset。
- patch 失败 hard fail，不允许截断。

### 4.4 call / runtime helper

call path 不允许隐式 effect：

```text
XmCall(helper_id)
  ↓
helper registry says MAY_THROW / MAY_GC / MAY_SUSPEND
  ↓
driver emits call sequence
  ↓
generated post-call protocol checker verifies required checks are present
```

验收：

- 去掉 `MAY_THROW`，异常回归必须失败。
- 去掉 `MAY_GC`，GC stress 必须失败或 verifier 必须拒绝。
- 去掉 `MAY_SUSPEND`，coroutine suspend 回归必须失败或 verifier 必须拒绝。

---

## 5. 验证体系

### 5.1 build-time hard fail

| 检查 | 失败后果 |
|---|---|
| `.isa` parse | build fail |
| `.isa` typecheck | build fail |
| constraint 不满足 | build fail |
| encoding 宽度错误 | build fail |
| golden bytes 不匹配 | build fail |
| golden asm 不匹配 | build fail |
| min bytes 为 0 | build fail |
| `XmOp` handler coverage 缺失 | build fail |
| helper effect 缺失 | build fail |
| backend 未声明 runtime-sensitive handling | build fail |

### 5.2 disasm round-trip

每条 generated emit helper 必须满足：

```text
operands → emit bytes → generated disasm → expected asm
```

每条 golden case 必须至少覆盖：

- 普通寄存器
- 扩展寄存器
- 特殊寄存器
- immediate 边界
- memory addressing 边界
- branch 正负 offset 边界

### 5.3 external differential

使用 external assembler / disassembler 作为第三方 reference：

| arch | reference |
|---|---|
| x64 | llvm-mc |
| arm64 | llvm-mc |
| riscv64 | llvm-mc 或 GNU binutils |

流程：

```text
random mcinsn operands
  ↓
xisa emit bytes
  ↓
generated disasm text
  ↓
external disasm text
  ↓
normalized compare
```

CI：

- PR：固定 seed + 1k cases。
- nightly：随机 seed + 1M cases。
- 失败输出 seed、mcinsn、operands、bytes、两边 disasm。

### 5.4 driver invariant tests

每个 `XmOp` 必须满足：

- 被 backend coverage matrix 覆盖。
- emit path 写入至少 1 字节，或显式标记为 metadata-only op。
- runtime-sensitive op 必须生成相应 metadata。
- branch op 必须生成 patch record 或 final target。
- call op 必须生成 post-call protocol。

### 5.5 runtime parity

必须覆盖以下路径：

| 子项 | 验证 |
|---|---|
| call ABI | argument / return / live-across-call 双后端一致 |
| helper call | effect flags 与 post-call checks 完整 |
| safepoint | stack map slot / spill offset / PC range 一致 |
| deopt | reconstructed VM register state byte-equal |
| OSR | 多轮 reentry 不依赖 frame 残留 |
| suspend / resume | coroutine yield 点、GC 根、channel 状态一致 |
| patch site | IC / deopt / branch patch 不越界 |

### 5.6 pointer hygiene

所有 cross-tier pointer 入口必须有统一校验：

- alignment check
- heap range check
- header type range check
- trust level check
- conservative input fast-skip

覆盖入口：

- conservative stack scan
- safepoint stack scan
- write barrier
- deopt slot reconstruction
- shared refs flow
- GC mark / sweep / decref

验收：

- ASAN + `--jit-force` + 高频 GC stress。
- 故意注入错位 pointer，fuzz 必须触发或 verifier 必须拒绝。
- sweep / mark / decref 不允许把非法 header 送入 destructor。

### 5.7 fuzz

四层 fuzz：

| 层 | 内容 |
|---|---|
| mcinsn fuzz | random operands → emit → disasm → external diff |
| Xm fuzz | random small Xm program → JIT → interpreter compare |
| driver invariant fuzz | random op sequence → coverage / bytes / metadata verifier |
| corruption-chain fuzz | JIT force + ASAN + GC stress + pointer validation |

必须带 shrinker：

- emit bug 缩到单条 mcinsn。
- driver bug 缩到最小 Xm sequence。
- runtime corruption 缩到最短 cross-tier chain。

---

## 6. 实施阶段

### S0 — 契约冻结与旧路径清点

目标：把最终设计变成可执行边界，先禁止继续扩大旧手写路线。

任务：

- [ ] 冻结本方案为 codegen 唯一执行入口。
- [ ] 清点所有手写 raw byte / bitfield / special register encoding。
- [ ] 清点所有 `XmOp` switch 和 backend handler。
- [ ] 清点所有 runtime helper call site。
- [ ] 清点所有 cross-tier pointer 入口。
- [ ] 在当前 driver 中先补 hard fail：unhandled op、0-byte emit、unknown helper。

退出标准：

- raw encoding inventory 完整。
- helper inventory 完整。
- pointer boundary inventory 完整。
- 当前 regression 不退行。
- 新增 codegen 不允许继续走未登记手写 encoding。

### S1 — `XmOp` contract table 与 helper registry

目标：先解决 IR opcode / helper effect 的真相源漂移。

任务：

- [ ] 新建 `xisa/xm_ops.def`。
- [ ] 生成 `XmOp` enum、name table、arity table、effect metadata。
- [ ] 生成 verifier，检查 block 内 op arity / effect / backend coverage。
- [ ] 新建 `xisa/xm_helpers.def`。
- [ ] 把 raw helper pointer 迁移为 helper id。
- [ ] 删除旧的手写 op metadata table。

退出标准：

- unknown op 无法进入 backend driver。
- unknown helper 无法进入 codegen。
- effect 缺失 build/test fail。
- ctest + regression 通过。

### S2 — xisa generator 核心

目标：实现 C host generator 和最小 x64 / arm64 / riscv64 schema。

任务：

- [ ] 实现 lexer / parser / AST / typechecker。
- [ ] 实现 encoding evaluator。
- [ ] 生成 emit helpers。
- [ ] 生成 mcattr tables。
- [ ] 生成 golden tests。
- [ ] generator 自身单测覆盖 parse / typecheck / encode 错误路径。

退出标准：

- macOS / Linux / Windows MSVC clean build 都不依赖 xray binary。
- 每个 arch 至少 20 条 mcinsn fixture。
- 每条 mcinsn 至少 3 个 golden bytes。
- 故意改错 encoding 时 build/test fail，诊断包含路径、行列、mcinsn、operands、expected / actual。

### S3 — generated disassembler 与 external diff

目标：不再手写 disassembler 作为第二真相源。

任务：

- [ ] xisa schema 增加 golden asm 和 decode priority。
- [ ] 生成 x64 / arm64 / riscv64 emitted-subset disassembler。
- [ ] 接入 round-trip tests。
- [ ] 接入 llvm-mc / binutils differential。
- [ ] 建立 normalize 层。

退出标准：

- generated emit + generated disasm round-trip 全过。
- external diff 1k seed 全过。
- unknown bytes 优雅输出，不 crash。
- 新增 mcinsn 没有 golden asm 时 build fail。

### S4 — x64 encoding 替换并删除旧 helper

目标：x64 scalar / memory / control machine encoding 改为 generated helper，删除对应旧手写 helper。

任务：

- [ ] 迁移 scalar integer / float emit。
- [ ] 迁移 mov / load / store / lea。
- [ ] 迁移 cmp / test / set / branch encoding。
- [ ] 迁移 patchable forms。
- [ ] 删除被替换的 `xm_x64` raw encoding helper。
- [ ] driver 只保留 runtime-sensitive logic。

退出标准：

- x64 covered instruction 不存在手写 byte / ModR/M / SIB 拼装。
- x64 regression + JIT force 通过。
- x64 external diff nightly 启动。
- Win11 Release 重点协程回归无 heap corruption。

### S5 — arm64 encoding 替换并删除旧 helper

目标：arm64 scalar / memory / control machine encoding 改为 generated helper，删除对应旧手写 helper。

任务：

- [ ] 迁移 fixed-width scalar emit。
- [ ] 迁移 load / store addressing forms。
- [ ] 迁移 branch immediate forms。
- [ ] 迁移 patchable forms。
- [ ] 删除被替换的 `xm_arm64` raw bitfield helper。

退出标准：

- arm64 covered instruction 不存在手写 bitfield 拼装。
- arm64 ctest + regression 通过。
- x64 / arm64 scalar differential 0 分歧。
- arm64 external diff nightly 启动。

### S6 — branch / patch / metadata 契约统一

目标：把 branch reach、patch width、metadata emission 变成机器可检查契约。

任务：

- [ ] patch record 记录 mcinsn id、offset、width、signedness。
- [ ] patch 阶段复用 generated range checker。
- [ ] stack map verifier 检查 PC range 与 emitted bytes 对齐。
- [ ] deopt metadata verifier 检查 spill/register location。
- [ ] OSR metadata verifier 检查 entry live set。

退出标准：

- 故意 patch 越界必须 fail-fast。
- 故意 stack map PC range 错误必须 fail-fast。
- 故意 deopt spill offset 错误必须 fail-fast。
- call / deopt / OSR / safepoint parity tests 通过。

### S7 — runtime-sensitive hardening

目标：补齐 machine encoding 之外的 JIT 高危路径。

任务：

- [ ] call ABI parity。
- [ ] helper effect completeness。
- [ ] exception post-call checks。
- [ ] safepoint GC root correctness。
- [ ] deopt reconstruction correctness。
- [ ] OSR spill-only vreg materialization。
- [ ] suspend / resume parity。
- [ ] cross-tier pointer hygiene。

退出标准：

- effect flag 故意漏标时测试失败。
- pointer alignment fast-skip 覆盖所有 conservative 入口。
- ASAN + JIT force + GC stress 30 分钟无 corruption。
- Win11 Release 协程重点回归无 heap corruption。

### S8 — RISC-V 后端从零使用 xisa

目标：第三后端不复制旧手写路线。

任务：

- [ ] `xisa/backends/riscv64/isa.def` 覆盖 RV64 scalar / mem / branch / call subset。
- [ ] 写 riscv64 driver skeleton。
- [ ] QEMU 跑 scalar JIT unit tests。
- [ ] scalar differential 加入 riscv64。
- [ ] 不允许新增 riscv64 手写 raw encoding helper。

退出标准：

- riscv64 subset 在 QEMU 通过。
- x64 / arm64 / riscv64 scalar differential 0 分歧。
- riscv64 generated disasm + external diff 通过。

### S9 — fuzz、linemap 与长期 CI

目标：把 rare codegen bug 移到 nightly 自动发现。

任务：

- [ ] generated mcinsn linemap。
- [ ] driver synthetic region linemap。
- [ ] mcinsn fuzz。
- [ ] Xm program fuzz。
- [ ] driver invariant fuzz。
- [ ] corruption-chain fuzz。
- [ ] shrinker。
- [ ] CI 指标 dashboard。

退出标准：

- nightly 1M mcinsn seed 连续 4 周 0 分歧。
- ASAN + JIT force + GC stress nightly 稳定。
- 任一 fuzz failure 可用 seed 单命令复现。
- shrinker 能把 emit bug 缩到单条 mcinsn。

---

## 7. 删除清单

最终必须删除或替换：

| 类别 | 处理 |
|---|---|
| 手写 `XmOp` enum | 由 contract table 生成 |
| 手写 op name / arity / effect table | 由 contract table 生成 |
| raw helper pointer 语义判断 | 改为 helper id + registry |
| x64 raw byte emit helper | covered forms 删除 |
| arm64 raw bitfield emit helper | covered forms 删除 |
| 手写 x64 emitted-subset disassembler | 改为 generated disassembler |
| 手写 arm64 emitted-subset disassembler | 改为 generated disassembler |
| silent fallback | 删除，改 hard fail |
| permanent dual-emit | 禁止 |
| legacy wrapper | 禁止 |

允许保留：

| 类别 | 原因 |
|---|---|
| backend driver | runtime-sensitive protocol 必须手写控制 |
| code allocator | 与 xisa 无关 |
| regalloc | 与 xisa 无关 |
| frame layout logic | driver 职责 |
| deopt / OSR / safepoint metadata writer | driver 职责，但必须有 verifier |

---

## 8. 完成判定

### 8.1 v1 完成

- [ ] `XmOp` contract table 生效。
- [ ] helper registry 生效。
- [ ] C host generator clean build。
- [ ] x64 / arm64 scalar + memory + control covered forms 使用 generated helper。
- [ ] 对应旧手写 helper 已删除。
- [ ] generated disassembler 可用。
- [ ] golden bytes + golden asm 全过。
- [ ] external diff PR 1k seed 全过。
- [ ] ctest + regression 不退行。

### 8.2 完整完成

- [ ] x64 / arm64 covered machine encoding 无手写 byte / bitfield 拼装。
- [ ] riscv64 subset 从零使用 xisa 跑通。
- [ ] runtime-sensitive parity 全部通过。
- [ ] effect flag audit clean。
- [ ] pointer hygiene audit clean。
- [ ] fuzz + shrinker + ASAN GC stress nightly 稳定。
- [ ] generated linemap 覆盖 generated mcinsn。

---

## 9. 与现有文档关系

| 文档 | 关系 |
|---|---|
| `073-codegen-hardening.md` | 被本方案吸收：fail-fast、runtime parity、effect flag、pointer hygiene、fuzz 保留 |
| `073-xisa-implementation.md` | 被本方案吸收：C generator、S-expression DSL、encoding-first、driver-preserving 保留 |
| `xisa_design.md` | 被本方案升级：增加 `XmOp` contract、helper registry、generated disassembler、删除旧 helper 策略 |
| `008-jit-multi-backend.md` | 本方案成为多后端 codegen 的新基础 |
| `009-jit-x64-parity.md` | parity 经验纳入 S7 |
| `068-m4-jit-from-xir.md` | Xi→Xm 输入保持不变，本方案只处理 Xm 之后 |
| `070-regression-bug-triage.md` | codegen 类 bug 最终转为 golden / differential / parity / fuzz case |

---

## 10. 不做的事

| 不做 | 原因 |
|---|---|
| 用 Python / Rust / Zig 写 generator | 引入额外核心 build toolchain，不符合当前项目约束 |
| 用 xray 自举 generator | 会制造 bootstrap 环，等工具链稳定后再考虑 dogfood |
| 做 LLVM TableGen 级别通用框架 | Xray 只需要 emitted subset 的 machine encoding truth source |
| 生成完整 runtime-sensitive lowering | call / deopt / OSR / safepoint / suspend 仍由 driver 手写更清晰 |
| 保留旧手写接口 | 违反不兼容原则，会制造长期技术债 |
| 在 driver 中继续手写 covered instruction bytes | 破坏 xisa 单一真相源 |

---

## 11. 执行纪律

- 每个阶段发现真实 bug，立即修 root cause 并补测试。
- 不允许用 generator 适配已知错误编码。
- 不允许为了保留旧行为增加兼容层。
- 不允许保留未使用旧 helper。
- 不允许跳过 failing regression。
- 不允许把 unknown helper 当 pure helper。
- 不允许让 conservative pointer 未校验就进入解引用路径。
- 每次代码修改后运行 ctest；涉及 JIT runtime 的阶段必须跑 regression；涉及 pointer / GC 的阶段必须跑 ASAN + GC stress。

---

## 12. 推荐启动顺序

最小启动闭环：

```text
S0 hard fail + inventory
  ↓
S1 XmOp contract + helper registry
  ↓
S2 generator + golden
  ↓
S3 generated disasm + external diff
  ↓
S4 x64 replacement
```

原因：

- S0 立即堵住 silent fallback。
- S1 先消灭 effect / opcode metadata 漂移。
- S2 建立新真相源。
- S3 建立可观察性和第三方 reference。
- S4 开始删除旧手写 encoding，真正降低技术债。

---

## 13. 修订历史

| 日期 | 改动 | 作者 |
|---|---|---|
| 2026-05-12 | 初稿：合并 xisa 与 codegen hardening 两条路线，确立 xisa 为 machine encoding 真相源，同时引入 `XmOp` contract、helper registry、generated disassembler、runtime parity、effect flag、pointer hygiene、fuzz；明确不兼容原则与旧 helper 删除策略 | Cascade + Xinglei Xu |
