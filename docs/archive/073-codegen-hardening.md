# 073 — codegen 工程化加固

> **作者**: Cascade (pair-programming with Xinglei Xu)
> **日期**: 2026-05-12
> **状态**: planned
> **配套架构 spec**: [`../engineering/codegen_architecture.md`](../engineering/codegen_architecture.md)
> **直接动机**: commit `8b06997` 修复的 x64 RT-opcode 漏 emit bug（Win11 上 `STATUS_HEAP_CORRUPTION`）
> **路线**: Dart 风格工程化（手写 assembler + 自写 disassembler + 强测试），不引入 codegen DSL
> **路线选型**: 曾考虑过 codegen DSL 方案，评估后未被采纳（详见配套 architecture spec §1.4 / §1.5）

> 本文档只回答"怎么做"和"何时做"。架构决策、driver / assembler 契约、disassembler 接口、测试形式在配套 spec 中定义，不在此重复。

---

## 0. 总览

```text
fail-fast 契约 → x64 disassembler → 测试框架 → byte-level coverage
                → llvm-mc differential → runtime parity → fuzz
```

| 阶段 | 标题 | 输出 | 估算 |
|---|---|---|---|
| P0 | codegen fail-fast 契约 | 所有 `XmOp` switch 穷尽 + emit 字节数自检 | 1 周 |
| P1 | x64 disassembler | `xm_x64_disasm.h`（与 `xm_arm64_disasm.h` 对齐） | 1-2 周 |
| P2 | ASSEMBLER_TEST 框架 | `tests/unit/jit/assembler_test.h` macro pair + ctx | 1 周 |
| P3 | x64 / arm64 byte-level unit test 覆盖 | 每个 emit fn ≥ 3 case，覆盖普通 / 扩展 / 特殊寄存器 | 3-4 周 |
| P4 | `llvm-mc` differential CI | random program + 双反汇编对比 | 1 周 |
| P5 | runtime-sensitive parity | call / deopt / OSR / safepoint / suspend 各自 DoD | 4-6 周 |
| P6 | codegen fuzz nightly | assembler / codegen / driver-invariant 三层 fuzz | 1 周 |

每阶段独立可回滚。任何阶段发现 bug，按 root cause 修复并补回归，不能掩盖。

> P0 完成即可堵死 commit `8b06997` 同类 bug，不需要等其他阶段。

---

## 1. 目标 / 非目标

### 1.1 目标

| 目标 | 验证方法 |
|---|---|
| commit `8b06997` 类 bug 在 build-time 捕获 | P0 + P3 后回归 ASAN + Win11 stress |
| 手写 assembler 行为 100% 可反汇编 | P1 + P3 round-trip test |
| 跨 backend 一致性可机械验证 | P4 `llvm-mc` differential + P5 parity test |
| Driver 沉默 fallback 不再可能 | P0 强制 `default: XR_UNREACHABLE` + 字节数自检 |
| 不引入 codegen DSL / generator | 不新增 `tools/xisagen/` 等目录，不引入 `.isa` schema |
| 不破坏现有 JIT | 每次代码接入后 ctest + regression 不退行 |

### 1.2 非目标

整体非目标见配套 spec §2。本 task 额外排除：

| 非目标 | 理由 |
|---|---|
| 引入 codegen DSL（`.isa` / generator） | xray 规模未达需要 DSL 的阈值；Dart 5 backend / 74K 行实证手写可控 |
| 嵌入 capstone 第三方 disassembler | 已有 `xm_arm64_disasm.h` 自写先例，跨 backend 一致性优先；二进制大小约束 |
| 引入 instruction scheduler | xray JIT 优化深度未到 |
| 引入 multi-instruction pattern fusion | 同上 |
| RISC-V backend bring-up | 与本 task 正交，待 user demand 触发后单独立项 |
| Win64 `.pdata` / `.xdata` unwind 信息 | 与本 task 正交，独立 follow-up |

---

## 2. 基线

> 代码量和测试数字取自 2026-05-12（本文初稿）；P0 启动时重采一次并在本节补录。

### 2.1 已有资产

| 资产 | 路径 | 规模 |
|---|---|---|
| x64 byte-stream assembler | `src/jit/xm_x64.{h,c}` | ~20 KB |
| arm64 fixed-width assembler | `src/jit/xm_arm64.{h,c}` | ~22 KB |
| arm64 disassembler | `src/jit/xm_arm64_disasm.h` | ~18 KB header-only |
| x64 codegen driver | `src/jit/xm_codegen_x64*.c` (7 文件) | ~5151 行 |
| arm64 codegen driver | `src/jit/xm_codegen*.c` (含 mem/call/stub/ins) | ~5814 行 |
| JIT E2E test | `tests/unit/jit/test_jit_e2e.c` | 单文件 |
| code allocator | `src/jit/xm_code_alloc.{h,c}` | 已有 |

### 2.2 缺口

| 缺口 | 影响 |
|---|---|
| `xm_x64_disasm.h` | dump / debug / test golden 都依赖；目前完全没有 |
| `tests/unit/jit/assembler_test.h` | 没有 Dart 风格 macro pair |
| `tests/unit/jit/test_x64_assembler.c` | 没有 byte-level emit unit test |
| `tests/unit/jit/test_arm64_assembler.c` | 同上 |
| `tests/unit/jit/test_<arch>_disasm.c` | 没有 disassembler round-trip test |
| codegen `default: XR_UNREACHABLE` | `xm_codegen_x64*.c` 全文 `grep` 结果 0 — commit `8b06997` 根因 |
| 字节数自检 | driver 没有 `XR_DCHECK(emitted > 0)` |
| `llvm-mc` differential | 没有 |
| codegen fuzz | 没有 |

### 2.3 测试基线（2026-05-11）

| 平台 | 测试 | 通过数 |
|---|---|---|
| macOS Debug | ctest | 113/113 |
| macOS Debug | regression | 293/293（含 1207_gc_stress） |
| Win11 Release | ctest | 105/105 |
| Win11 Release | regression | 291/293（已知 2 个非 codegen bug） |

阶段退出时不得降低基线。

### 2.4 历史 bug 教训（直接动机的扩展）

本 task 的直接动机是 commit `8b06997`，但近期 bug 修复历史还揭示了三类同样需要根除的抽象模式。**它们都不需要 codegen DSL 才能解决，但需要纳入本 task 的实施纪律**。

| 模式 | 代表 commit | 现象 | 抽象根因 | 在哪个阶段处理 |
|---|---|---|---|---|
| **A. 沉默 fallback / 漏 emit** | `8b06997` | x64 codegen RT-opcode 漏 emit 任何字节 → Win11 `STATUS_HEAP_CORRUPTION` | switch 没穷尽 + 没 emit 字节自检 | **P0**（switch 穷尽 + `EMIT_XMOP_BEGIN/END`） |
| **B. 跨边界 effect 漏标** | `4da96e3` | generic `XM_CALL_DIRECT` 桥没标 `XM_FLAG_MAY_THROW` → 内层 VM 异常 unwind 越过外层 catch → `AFTER_TRY` / `END` 重复打印 | codegen 调用 runtime / VM 桥时丢失副作用声明，post-call check 缺失 | **P5.6** effect flag completeness audit |
| **C. 保守抽象接受错位输入 → 下游不校验** | `d94c4a8` | 保守扫描接受 `obj + 8` 错位指针 → 7 步 corruption chain → `type` 字段从 30 减到 28 → sweep 送伪 Logger 到 destructor → crash | 任何"接受任意输入"的抽象，下游解引用前必须 alignment / range / tag 校验 | **P5.7** cross-tier pointer hygiene audit |
| **D. 多 Pass / 多 stage invariant 漂移** | `50a187d` (Bug C) | analyzer Pass 1 对 `AST_SCOPE_BLOCK` 不创建 scope，Pass 2 创建 BLOCK scope → 两 Pass 父链不一致 → 循环变量 Pass 1 注册但 Pass 2 找不到 | 多 Pass 编译器 / 多 stage codegen，对同一节点的"作用域开关 / 状态契约"必须结构对称 | **P3 + P4**（disassembler round-trip + `llvm-mc` differential 是同款 mechanical 验证） |

> 模式 A 已是 P0 主线。模式 B / C 在原 P5 的 5 子项里没有对应位置 — 本 task 把它们补成 **P5.6 / P5.7**，让 5 子项扩到 7 子项。模式 D 不是新增子项，但在 P3 / P4 退出标准中体现：driver / assembler / disassembler 三层 mechanical round-trip 即可捕获 stage 间漂移。

> **不在本 task scope** 但相关：
> - 测试设计纪律（race / `@test` + 显式调用重复 / shared const + push 矛盾）— 与 codegen 正交，建议归 `070-regression-bug-triage.md` 或独立 task
> - 诊断方法学（lldb watchpoint × `XR_GC_TRIPWIRE_HOOK` × 批量重跑）— 来自 commit `d94c4a8` 调查过程，建议独立成 `docs/engineering/runtime_debug_workflow.md`，被 P5.7 / P6 引用而非内嵌

---

## 3. 实施阶段

### P0 — codegen fail-fast 契约

**目标**：根除 commit `8b06997` 类"沉默 fallback"bug，不靠新测试，**只靠强契约 + 现有 regression**。

**任务**：

- [ ] 全文扫描 `src/jit/xm_codegen*.c`，找出所有对 `XmOp` 的 switch
- [ ] 每个 switch 末尾加 `default: XR_UNREACHABLE("unhandled XmOp %d in %s", op, __func__);`
- [ ] 启用 `-Wswitch-enum`，让漏 case 在编译期 warning（CI 升级为 error）
- [ ] 在 `src/jit/xm_codegen_internal.h` 添加 `EMIT_XMOP_BEGIN(ctx, ins) / EMIT_XMOP_END(ctx, ins)` macro pair
  - `EMIT_XMOP_BEGIN`：记录 `pos_before = buf_offset(...)`
  - `EMIT_XMOP_END`：`XR_CHECK(buf_offset(...) > pos_before, "XmOp %d emitted 0 bytes", ins->op)`
  - 注意：用 `XR_CHECK` 而非 `XR_DCHECK`，**Release 也保留检查**
- [ ] 把每个 IR op 的 emit 入口（`emit_xm_<opname>` 风格）包到 `EMIT_XMOP_BEGIN/END` 之间
- [ ] 现有 regression 全过

**退出标准**：

- `grep -rn 'switch.*XmOp\|switch.*ins->op' src/jit/xm_codegen*.c` 找到的 switch 100% 有 `default: XR_UNREACHABLE`
- 故意 comment 掉一个 case，build 阶段 `-Werror=switch-enum` 直接拒
- 故意改一个 IR op 的 emit handler 让其漏 emit，runtime 在第一次执行时 `XR_CHECK` fail（不再静默通过）
- ctest + regression 全过
- Win11 Release 重跑 commit `8b06997` 复现路径，确认仍正常

**回滚锚点**：`codegen-p0-fail-fast`

---

### P1 — x64 disassembler

**目标**：补齐 `xm_x64_disasm.h`，与 `xm_arm64_disasm.h` 风格对齐。

**任务**：

- [ ] 创建 `src/jit/xm_x64_disasm.h`（header-only pure func）
- [ ] 接口：

  ```c
  size_t x64_disasm_one(const uint8_t *code, size_t code_size,
                         size_t offset, char *buf, size_t buf_size);
  void   x64_disasm_region(const uint8_t *code, size_t size,
                            char *buf, size_t buf_size);
  ```
- [ ] 覆盖 xray x64 assembler 实际 emit 的指令集合，包括：
  - REX prefix decode + 寄存器名映射
  - ModR/M / SIB decode
  - mov reg/reg / mov imm / add / sub / and / or / xor / cmp / test / lea
  - load / store（含 rsp/rbp/r12/r13 special form）
  - jcc / jmp / call / ret
  - push / pop
  - xray 实际用到的所有 opcode（用 P3 测试反向倒逼覆盖率）
- [ ] 输出格式：Intel asm 风格（`add rax, rcx` / `mov [rsp+8], rax`），与 `llvm-mc -x86-asm-syntax=intel` 对齐
- [ ] 未识别指令：返回 0，输出 `"<unknown bytes: XX XX XX>"`（dump 退化优雅）
- [ ] 在 `tests/unit/jit/test_x64_disasm.c` 写 round-trip：`x64_assembler emit → x64_disasm_one → expected string`，覆盖每条 assembler emit fn

**退出标准**：

- `xm_x64_disasm.h` 不需要修改 `xm_x64.c` 或链接任何 .c，纯 header-only
- 100 个手写 round-trip case 全过
- 输出格式与 `llvm-mc -x86-asm-syntax=intel -disassemble` normalize 后一致（normalize：去多余空格、统一立即数 `0x` 前缀）
- 未识别字节优雅降级，不 abort
- ctest + regression 不退行

**回滚锚点**：`codegen-p1-x64-disasm`

---

### P2 — ASSEMBLER_TEST 框架

**目标**：建立 Dart 风格 `ASSEMBLER_TEST_GENERATE` / `ASSEMBLER_TEST_RUN` macro pair。

**任务**：

- [ ] 创建 `tests/unit/jit/assembler_test.h`（macro 定义）和 `assembler_test.c`（context 实现）
- [ ] 接口：

  ```c
  /* 共享 ctx，承载 buf + code memory + disasm scratch */
  typedef struct {
      X64Buf  x64_buf;       /* 仅 x64 测试用 */
      A64Buf  arm64_buf;     /* 仅 arm64 测试用 */
      uint8_t code_mem[4096];
      char    disasm_buf[4096];
  } AssemblerTestCtx;

  void assembler_test_ctx_init_x64(AssemblerTestCtx *ctx);
  void assembler_test_ctx_init_arm64(AssemblerTestCtx *ctx);
  void assembler_test_disassemble_x64(AssemblerTestCtx *ctx, char *out, size_t out_size);
  void assembler_test_disassemble_arm64(AssemblerTestCtx *ctx, char *out, size_t out_size);
  ```
- [ ] Macro pair（per-arch 包装）：

  ```c
  ASSEMBLER_TEST_GENERATE_X64(name, asm_var) { ... }
  ASSEMBLER_TEST_RUN_X64(name, ctx) {
      EXPECT_DISASSEMBLY_X64("expected line 1\nexpected line 2\n");
  }
  ```
- [ ] `EXPECT_DISASSEMBLY_*` 失败时输出：actual / expected / 第一行差异位置 / 字节 hex dump
- [ ] 集成到现有 ctest，新增 target `test_assembler_framework`

**退出标准**：

- 框架本身有 ≥ 5 个 self-test（包括故意制造 disassembly mismatch 验证错误信息可读）
- macro pair 在 macOS / Linux / Windows MSVC 三平台 build pass
- ctest + regression 不退行

**回滚锚点**：`codegen-p2-test-framework`

---

### P3 — x64 / arm64 byte-level unit test 覆盖

**目标**：用 P2 框架为两个 backend 的所有 assembler emit fn 写 byte-level test。

**任务**：

- [ ] `tests/unit/jit/test_x64_assembler.c`：覆盖 `xm_x64.h` 所有顶层 emit fn
  - 每个 emit fn ≥ 3 个 case：普通寄存器、扩展寄存器（r8-r15）、特殊寄存器（rsp / rbp / r12 / r13）
  - mem form 单独覆盖：`[base]` / `[base+disp8]` / `[base+disp32]` / `[base+index*scale]` / `[rsp+...]`
  - 立即数边界：i8 / i32 / i64 各形式
- [ ] `tests/unit/jit/test_arm64_assembler.c`：覆盖 `xm_arm64.h` 所有顶层 emit fn
  - 每个 emit fn ≥ 3 case
  - bitfield imm 边界：12-bit / 16-bit / shifted forms
  - 特殊寄存器：sp / xzr context
- [ ] `tests/unit/jit/test_x64_modrm.c`：`x64_modrm_mem` 各种 base/disp 组合穷尽
- [ ] `tests/unit/jit/test_arm64_disasm.c`：现有 `xm_arm64_disasm.h` 的 round-trip 覆盖（之前没写过单独的 disasm test）
- [ ] 把 `EXPECT_DISASSEMBLY_*` 的 expected string 用 `llvm-mc` 离线生成 + 人工 review 后 commit

**退出标准**：

- 每个 `xm_x64.h` 顶层 emit fn 至少 3 个 byte-level test，**含特殊寄存器**
- 每个 `xm_arm64.h` 顶层 emit fn 同上
- `tests/unit/jit/` 新增测试数 ≥ 200 case
- 故意改错某个 emit fn 的 1 字节，对应 test fail 且诊断信息直接指出 mismatch line + offset
- ctest + regression 不退行

**回滚锚点**：`codegen-p3-byte-coverage`

---

### P4 — `llvm-mc` differential CI

**目标**：用 LLVM 的 `llvm-mc` 作为第三方 reference，对 xray emit 的字节做 cross-validation。

**任务**：

- [ ] 创建 `tests/unit/jit/test_assembler_diff.c`：
  - random program generator：每 PR 1k 种子，nightly 1M 种子
  - 每个种子 → emit via xray assembler → bytes
  - bytes → xray `xm_<arch>_disasm` → text_xray
  - bytes → `llvm-mc -disassemble` → text_llvm
  - normalize（去空格 / 统一立即数前缀 / mnemonic 大小写）
  - 不一致 → fail-fast，dump bytes / both texts / 复现 seed
- [ ] CI 集成（macOS + Linux runner 上启用，Windows 暂不要求 — `llvm-mc` 在 MSVC 环境配置成本高）
- [ ] `scripts/codegen_diff.sh`：本地复现命令，传入 seed 直接复现
- [ ] 不引入 `llvm-mc` 到 xray runtime 依赖；仅 CI / 本地 dev 时使用

**退出标准**：

- macOS / Linux CI 上 `test_assembler_diff` 1k 种子 0 分歧
- 故意改错某条 assembler emit fn 1 个 bit，CI fail 且 seed 可独立复现
- nightly 1M 种子 0 分歧（首次启动后观察 1 周）

**回滚锚点**：`codegen-p4-llvm-mc-diff`

---

### P5 — runtime-sensitive parity

**目标**：逐项处理 call / deopt / OSR / safepoint / suspend-resume，加 effect flag 完整性 + cross-tier pointer hygiene，不混入 P0-P4。

**7 个独立子项**（任意顺序，各自 DoD）：

#### P5.1 call ABI parity

- [ ] x64 / arm64 各自补 `call`、`call-known`、`call-c-helper`、`call-self-direct` parity test
- [ ] 验证 argument / return / live-across-call 在两 backend 行为一致
- **DoD**：4 类 call × 2 backend 共 8 个子集 0 分歧

#### P5.2 safepoint stack map

- [ ] 新增 metadata verifier：每个 JIT 导出的 stack map 验证 slot ID / spill offset / PC range 与 driver 出口一致
- [ ] 补 GC stress 下 safepoint poll 不丢根 / 不误根的回归
- **DoD**：`scripts/run_regression_tests.sh` + GC stress 30 分钟无新 leak / heap-buffer-overflow / use-after-free

#### P5.3 deopt spill / register reconstruction

- [ ] 补 deopt parity test：同一 deopt 点在 x64 / arm64 上重建后的 VM register state byte-equal
- [ ] 覆盖 spill 与 register live 混合、同函数多点 deopt、deopt + post-call resume
- **DoD**：两 backend 同输入 deopt observable state 一致；不出现读未初始化 spill slot 的回归

#### P5.4 OSR entry

- [ ] 复用 `test_osr_entry_pressure` 框架拓展多点 / 多参 / spill-pressure 场景
- [ ] 验证 OSR 重入不依赖 frame 残留、不读未初始化 spill-only vreg
- **DoD**：x64 与 arm64 OSR 重入 ≥ 4 轮不退行，多参 / spill-pressure byte-equal observable state

#### P5.5 suspend / resume

- [ ] 补 coroutine suspend / resume parity test：同一协程在两 backend 上调度轮序 / yield 点 / GC 根 / channel 状态一致
- [ ] 验证 `xchannel` / `xcoro_suspend` resume 在 x64 上不产生 Win11 `STATUS_HEAP_CORRUPTION`（commit `8b06997` 同类 bug 重现验证）
- **DoD**：协程用例在两 backend Debug + Release 不退行；Win11 Release 上 `1115_cancel.xr` / `1109_await_any.xr` / `1127_coro_priority.xr` / `1128_yield.xr` 无 `STATUS_HEAP_CORRUPTION` 及类似 heap 错误

#### P5.6 effect flag completeness audit

> **动机**：commit `4da96e3` 修复的 nested-VM 异常泄漏 bug — generic `XM_CALL_DIRECT` 桥未传 `XM_FLAG_MAY_THROW`，codegen 没插 post-call exception check，导致内层 VM 异常 unwind 越过外层 catch。这是"effect 漏标 → 沉默丢失副作用"的抽象模式。

- [ ] 全文扫描所有 codegen 中 emit call 的位置（grep `xi_to_xm.c` / `xm_codegen*.c` 中的 `XM_CALL_*` 入口）
- [ ] 每个 call site 必须显式标 effect flag 三元组：`MAY_THROW` / `MAY_SUSPEND` / `MAY_GC`
- [ ] 在 `xi_to_xm.c` / `xm_codegen.c` 中**默认 conservative**：unknown C bridge → 三个 flag 全部置位；只有显式声明"纯计算 helper"的才能去 flag
- [ ] 新增 unit test：每条 effect flag 故意去掉一个 → codegen post-call check 应缺失 → 对应 regression test 应 fail
- [ ] 在 `src/jit/xm.h` 文档化每个 effect flag 的语义和必填条件
- [ ] 全 audit clean 后补一个 commit `4da96e3` 的回归测试（嵌套异常 + JIT call bridge）

**DoD**：
- 所有 `XM_CALL_*` 发出点已 audit；不允许 unknown bridge 不带 conservative flag
- 故意去掉 generic `XM_CALL_DIRECT` 的 `MAY_THROW`，回归测试在 macOS Debug fail
- 跨 backend：x64 与 arm64 的 IR op flag 设置 byte-equal

#### P5.7 cross-tier pointer hygiene audit

> **动机**：commit `d94c4a8` 修复的 7 步 corruption chain — 保守扫描接受 `obj + 8` 错位指针 → `markobject` 不校验对齐 → 错位指针进 `shared_refs` → 下次 GC `xr_shared_decref` 对真实 GC header 做 `atomic_fetch_sub` → `type` 字段从 `TBLOB(30)` 减到 `TLOGGER(28)` → sweep 把"伪 Logger"送 destroy → `EXC_BAD_ACCESS`。这是"保守抽象接受任意输入 → 下游解引用前不校验"的抽象模式。

- [ ] 全文扫描所有 cross-tier pointer 入口：保守扫描（`mark_coro_roots` 退化路径）/ write-barrier / safepoint stack scan / deopt slot 重建 / `shared_refs[]` 流转 / Conservative GC 误根
- [ ] 在每个解引用前加 alignment + range fast-skip：

  ```c
  /* 任何来源不可信的 GC pointer，解引用前必须先 skip 错位 */
  if (((uintptr_t) obj & alignof(XrGcHeader)) != 0)
      return;  /* 静默跳过非对齐输入；real GC obj 不可能错位 */
  ```
- [ ] 验证 `obj->type` 在 sweep 入口时仍属合法枚举范围；越界即 `XR_FATAL` 而非传给 destructor
- [ ] 新增 GC stress regression：周期性 `gc.collect()` × ASAN × `--jit-force`，检查 7 步 chain 不重现
- [ ] 在 `docs/engineering/codegen_architecture.md` 的 cross-tier pointer 章节固化"保守输入必须 alignment fast-skip"为 spec contract（如未列出，本 task 推动 spec 补一节）

**DoD**：
- 所有列出的入口已加 alignment fast-skip；故意删除任一 fast-skip → 对应 fuzz / regression 在 ASAN 下 30 分钟内必复现 corruption
- `obj->type` 越界检查覆盖 sweep / mark / decref 三个入口
- commit `d94c4a8` 修复的 `1206_gc_enhanced.xr` × `--jit-force` × ASAN × 50 次 0 失败

**总阶段退出标准**：

- P5.1 - P5.7 七个子项 DoD 全部满足
- 期间发现的真实 bug 已修 root cause 并补回归

**回滚锚点**：`codegen-p5-runtime-parity`

---

### P6 — codegen fuzz nightly

**目标**：长期运行的 fuzz 防止 rare encoding bug 漂移；额外捕获 cross-tier corruption chain（commit `d94c4a8` 同类）。

**任务**：

- [ ] **Assembler 层 fuzz**：random `(reg, imm, mem-form)` → emit → disasm → 与 `llvm-mc` 对齐
- [ ] **Codegen 层 fuzz**：random small `Xm` IR program → JIT 编译 → 真机执行 → 与 interpreter 结果对比
- [ ] **Driver invariant fuzz**：random IR op sequence → 验证每条 IR op emit ≥ 1 字节（commit `8b06997` 类回归捕获）
- [ ] **Corruption chain fuzz**（NEW）：random IR program × `--jit-force` × ASAN build × `gc.collect()` 高频调用 × 每分钟触发 GC → 检测保守扫描 → shared_decref → type 字段腐蚀链路（commit `d94c4a8` 同类）
  - 配置：CI matrix 加一组 `Debug + ASAN + --jit-force + GC stress 30min`
  - 触发条件：测试程序周期性 `gc.collect()`，每次 collect 后扫一遍堆，验证所有 GC header `type` 字段在合法枚举范围内
  - 失败现象：ASAN 报 heap-buffer-overflow / use-after-free / 自检发现 `type` 字段越界
- [ ] shrinker：失败种子自动缩小到最小复现
- [ ] CI 指标：nightly 失败种子数 / 总种子数 / 平均缩小后 IR 节点数 / 每条 corruption chain 的步数（chain depth ≥ 3 步标记为高优先级）

**退出标准**：

- 四层 fuzz 各自 nightly 1M 种子 0 分歧（连续 4 周观察）
- 故意 inject 一个 emit bug，shrinker 在 30 秒内缩小到 ≤ 5 节点 IR 复现
- 故意 inject 一个保守扫描错位指针 case，corruption chain fuzz 在 30 分钟内必触发
- ctest + regression 不退行

**回滚锚点**：`codegen-p6-fuzz`

---

## 4. 关键 milestone

| Milestone | 何时 | 评估什么 | 决策点 |
|---|---|---|---|
| M1 commit `8b06997` 根除 | P0 | switch 穷尽 + 字节自检是否覆盖所有 codegen 入口 | 未覆盖 → 补 P0 |
| M2 disassembler 可用 | P1 | x64 disasm 是否能反汇编 xray 实际 emit 的所有指令 | 不够 → 扩 disasm |
| M3 测试框架成型 | P2 | macro pair 是否在三平台稳定 | Windows 失败 → 修 |
| M4 byte-level coverage | P3 | 每个 emit fn ≥ 3 case，特殊寄存器全覆盖 | 不足 → 补 case |
| M5 第三方 reference 一致 | P4 | `llvm-mc` differential 1k 种子 0 分歧 | 有分歧 → 修 root cause |
| M6 runtime parity | P5.1-P5.5 | 5 个子项 DoD 全过 | 缺一不可 |
| M6.5 effect flag audit clean | P5.6 | 所有 codegen call site 已 audit + conservative default | 漏标即视为 commit `4da96e3` 同类，立刻补 |
| M6.6 cross-tier pointer hygiene | P5.7 | 保守扫描 / write-barrier / deopt slot 重建路径全覆盖 alignment fast-skip | 任何路径漏校验即视为 commit `d94c4a8` 风险面 |
| M7 fuzz 稳定 | P6 | nightly 1M 种子 4 周 0 分歧 | 出现分歧 → triage |
| M7.5 fuzz × ASAN × `--jit-force` × GC stress | P6 | nightly 30 分钟无 leak / heap-buffer-overflow / use-after-free | 出现 corruption chain → 写 `docs/known_bugs.md` + root cause 修复 |

---

## 5. 风险

| ID | 风险 | 概率 | 影响 | 应对 |
|---|---|---|---|---|
| R1 | P0 全文加 `XR_UNREACHABLE` 后暴露多个隐藏路径 | 高 | 中 | 把每个新发现按 bug 修复纪律处理；不允许 silently 改回原样 |
| R2 | x64 disassembler 覆盖不全，P3 round-trip 失败 | 高 | 低 | 用 P3 测试驱动 disasm 覆盖率，缺什么补什么 |
| R3 | `EXPECT_DISASSEMBLY` 因 `llvm-mc` 与自家 disasm 格式不一致频繁误报 | 中 | 中 | normalize 层在 P2 早建立；格式差异写到 spec |
| R4 | Windows MSVC 上 macro 兼容性问题 | 中 | 高 | P2 退出标准强制三平台 build pass |
| R5 | P5 runtime-sensitive parity 暴露非平凡 bug | 高 | 高 | 各子项独立 DoD，不让单个子项阻塞整体；root cause 修复 |
| R6 | `llvm-mc` 在 CI runner 上版本飘移导致 normalize 不稳定 | 低 | 中 | pin LLVM 版本；normalize 层做版本兼容 |
| R7 | fuzz 暴露的 bug 数超出修复带宽 | 中 | 中 | shrinker 输出按 root cause 分组；P6 启动后专项 triage |
| R8 | 团队习惯仍倾向静默 fallback | 中 | 高 | P0 落地后在 PR template / code review checklist 中固化 `default: XR_UNREACHABLE` 规则 |
| R9 | 新人不熟悉 ASSEMBLER_TEST 风格 | 中 | 低 | spec §6 + 5 个 sample test 作为 onboarding 起点 |
| R10 | effect flag audit 暴露大量历史漏标（commit `4da96e3` 模式） | 高 | 中 | P5.6 默认按 conservative 策略（unknown bridge → `MAY_THROW + MAY_GC + MAY_SUSPEND`），每个去 conservative 标的 call site 必须配测试证明无副作用 |
| R11 | cross-tier pointer audit 工作面广（GC mark / shared_decref / safepoint stack scan / deopt slot 重建） | 高 | 高 | P5.7 拆按"指针来源"分批：保守扫描入口 → write-barrier 入口 → deopt slot 入口；每批独立 DoD |
| R12 | fuzz × `--jit-force` × ASAN × GC stress 组合下暴露的 corruption chain 超出修复带宽（commit `d94c4a8` 是 7 步 chain） | 中 | 高 | shrinker 按 root cause 分组；命中链路写到 `docs/known_bugs.md` 并按 bug 修复纪律逐个处理；不允许加 known_failures 长期屏蔽 |

---

## 6. 资源与时间估算

| 阶段 | 工作量 | 备注 |
|---|---|---|
| P0 | 1 周 | switch 穷尽 + 字节自检 |
| P1 | 1-2 周 | x64 disassembler |
| P2 | 1 周 | ASSEMBLER_TEST 框架 |
| P3 | 3-4 周 | x64 / arm64 byte-level coverage |
| P4 | 1 周 | `llvm-mc` differential CI |
| P5 | 6-9 周 | runtime-sensitive parity（7 子项独立 DoD：call / safepoint / deopt / OSR / suspend / effect flag / cross-tier pointer hygiene） |
| P6 | 1-2 周 | fuzz nightly + ASAN × `--jit-force` × GC stress 组合 |

**总计**：14-20 周。

> **P0-P4 7-9 周后已拿到完整 codegen build-time 健壮性收益**；P5 与 P6 可与其他 task 并行推进。

---

## 7. 完成判定

### 7.1 v1 完成（P0-P4）

| 指标 | 目标 |
|---|---|
| commit `8b06997` 类 bug | 在编译期 / build-time test 阶段捕获，runtime 不再出现 |
| `xm_codegen*.c` switch 穷尽 | 100% 有 `default: XR_UNREACHABLE` |
| 字节数自检 | 100% IR op emit 路径包 `EMIT_XMOP_BEGIN/END` |
| x64 disassembler 覆盖 | xray 实际 emit 的所有指令 100% 反汇编正确 |
| ASSEMBLER_TEST 框架 | 三平台 build pass，self-test ≥ 5 |
| Byte-level coverage | x64 / arm64 每个 emit fn ≥ 3 case |
| `llvm-mc` differential | 1k 种子 0 分歧 |
| ctest + regression | 三平台不退行 |

### 7.2 完整完成（P0-P6）

| 指标 | 目标 |
|---|---|
| Runtime-sensitive parity | P5.1-P5.7 全部 DoD 满足 |
| commit `4da96e3` 类 bug（effect flag 漏标） | P5.6 audit clean，故意去掉一个 `MAY_THROW` 对应 regression test fail |
| commit `d94c4a8` 类 bug（保守扫描错位指针） | P5.7 audit clean，alignment fast-skip 覆盖所有 cross-tier pointer 解引用入口 |
| Codegen fuzz | 三层 fuzz nightly 1M 种子 4 周 0 分歧 |
| Codegen fuzz × `--jit-force` × ASAN × GC stress 组合 | P6 nightly 至少跑 1 组，30 分钟无 leak / heap-buffer-overflow / use-after-free |
| Win11 Release 协程 regression | 重点 4 个用例无 `STATUS_HEAP_CORRUPTION` |

### 7.3 不强求

- RISC-V backend：与本 task 正交，待 user demand 后单独立项
- SIMD / VEX / EVEX：非目标
- instruction scheduler / pattern fusion：非目标
- 自动 branch relaxation：driver 仍负责

---

## 8. 与其他 task 的关系

| 关联 task | 关系 |
|---|---|
| `008-jit-multi-backend.md` | codegen 在平台相关层内部演进，不引入统一 codegen abstraction |
| `009-jit-x64-parity.md` | x64 与 arm64 行为对等是本 task 的输入 baseline；P5 进一步固化 |
| `068-m4-jit-from-xir.md` | Xi IR 直接驱动 JIT 与本 task 正交：IR 层 vs codegen 层。源码里原 XIR 已 rename 为 `Xm`，仅 task 文件名保留历史称呼 |
| `070-regression-bug-triage.md` | codegen 类 bug 可转成 byte-level unit test 固定样例 |

---

## 9. 修订历史

| 日期 | 改动 | 作者 |
|---|---|---|
| 2026-05-12 | 初稿；选择 Dart 风格工程化路线（手写 assembler + 自写 disassembler + 强测试），未采纳曾评估的 codegen DSL 方案 | Cascade + xingleixu |
| 2026-05-12 | 基于近期 bug 修复历史（commit `4da96e3` / `d94c4a8` / `50a187d` / `8b06997`）抽出 4 类抽象模式，纳入 §2.4 历史 bug 教训；新增 P5.6 effect flag completeness audit；新增 P5.7 cross-tier pointer hygiene audit；P6 fuzz 加 ASAN + JIT-force + GC stress 组合；§5 加 R10 / R11 / R12 | Cascade + xingleixu |
