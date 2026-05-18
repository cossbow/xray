# 075 — JIT codegen 最终规约（独立方案）

> **作者**: Cascade (pair-programming with Xinglei Xu)
> **日期**: 2026-05-12
> **状态**: planned
> **直接动机**: 整合近期 codegen 类 bug 的结构性根除路径
>   - commit `8b06997` — x64 RT-opcode 漏 emit → Win11 `STATUS_HEAP_CORRUPTION`
>   - commit `4da96e3` — 跨边界 effect flag 漏标 → 内层 VM 异常 unwind 越外层 catch
>   - commit `d94c4a8` — 保守扫描接受错位指针 → 7 步 corruption chain
>   - commit `50a187d` (Bug C) — 多 Pass invariant 漂移 → scope 父链不一致
> **来源依据**:
>   - `docs/tasks/073-codegen-hardening.md`
>   - `docs/archive/073-xisa-implementation.md`
>   - `docs/archive/xisa_design.md`
> **替代关系**: 与 `074-codegen-final-plan.md` 并行，作为独立第二方案备选；不依赖、不引用 074
> **开发原则**: ❌ 不考虑向后兼容；❌ 不保留旧接口；❌ 不做兼容层；✅ 每阶段最佳设计

---

## 0. 总览

```text
xisa DSL (machine encoding 真相源)
    ↓
C host generator (tools/xisagen/)
    ↓
generated emit / disasm / mcattr / dispatch / golden
    ↓
driver (frame/ABI/patch/OSR/deopt/suspend) + codegen-hardening 纪律
    ↓
strict 测试矩阵 (build-time + unit + differential + parity + fuzz)
```

**核心立论**：xisa 是 JIT 机器编码、disassembler、XmOp dispatch、effect/tier-input 元数据的**唯一真相源**。所有手写 byte 拼装、手写 disassembler、手写 XmOp switch dispatch **全部删除**。runtime-sensitive 路径（call/deopt/OSR/safepoint/suspend）保留 driver 形态，但 driver 不允许出现裸 byte 拼装、不允许漏标 effect/tier-input flag。codegen-hardening 的全部纪律作为 xisa 的外围验证矩阵强制叠加。

**不做**：dual-emit 长期共存、legacy fallback path、generator 与 driver 双源数据、emit 与 disasm 不同真相源。

---

## 1. 目标 / 非目标

### 1.1 目标

| 目标 | 验证方法 |
|---|---|
| `8b06997` 类 bug 被 build-time 拒（漏 emit / lowering 缺 case） | XmOp 真相表 + lowering 穷尽 typecheck |
| `4da96e3` 类 bug 被 build-time 拒（effect flag 漏标） | `:effect` 必填 + conservative-by-default typecheck |
| `d94c4a8` 类 bug 被 build-time 拒（cross-tier pointer 漏校验） | `:tier-input` 必填 + alignment fast-skip 自动插入 |
| `50a187d` 类 bug 在 mechanical 验证中暴露（多阶段漂移） | 三层 differential（emit / disasm / interpreter）+ fuzz |
| 多后端 marginal cost 显著降低 | x64 / arm64 / riscv64 共享 generator、subset、differential |
| 不存在两套 emit 并存 | grep 检查 driver 中无裸 `*buf++ = 0x` 模式 |
| ctest / regression / Win11 / nightly fuzz 全过 | CI 矩阵强制 |

### 1.2 非目标

| 非目标 | 理由 |
|---|---|
| AOT C backend 走 xisa | AOT 走 `src/aot/xi_cgen.c`，与 xisa 正交 |
| VM bytecode emitter 走 xisa | VM bytecode 是 ISA-independent 表示，与 machine encoding 无关 |
| `Xm` (历史名 XIR) 数据结构由 generator 输出 | xisa 只描述 mcinsn 和 lowering，不动 IR ABI |
| regalloc / frame layout / patch record / stack map | runtime contract，driver 主导 |
| 自动 branch relaxation | driver 决策 short/long branch；xisa 只算 reach |
| Win64 `.pdata` / `.xdata` unwind 信息 | 与 encoding 正交，独立 follow-up |
| SIMD / VEX / EVEX | v1 无需求 |
| 用 xray 自举 generator | bootstrap 死循环，禁止 |
| Python / Rust / Go / Lua / C++ 写 generator | 引入 build dep，违反单 toolchain 原则 |

---

## 2. 基线（2026-05-12，启动时重采）

### 2.1 codegen 代码量

```text
x64 backend:
  xm_codegen_x64.c            ~945
  xm_codegen_x64_call.c       ~922
  xm_codegen_x64_ins.c       ~1631
  xm_codegen_x64_mem.c        ~714
  xm_codegen_x64_osr.c        ~392
  xm_codegen_x64_patch.c      ~180
  xm_codegen_x64_stub.c       ~367
  xm_x64.c                    ~622
  xm_target_x64.c             ~121
  total                      ~5894 行

arm64 backend:
  xm_codegen.c               ~1621
  xm_codegen_call.c          ~1003
  xm_codegen_ins.c            ~434
  xm_codegen_mem.c           ~1301
  xm_codegen_stub.c           ~784
  xm_arm64.c                  ~605
  xm_target_arm64.c            ~66
  total                      ~5814 行
```

### 2.2 测试基线

| 平台 | 测试 | 通过数 |
|---|---|---|
| macOS Debug | ctest | 113/113 |
| macOS Debug | regression | 293/293 |
| Win11 Release | ctest | 105/105 |
| Win11 Release | regression | 291/293（已知 2 个非 codegen bug） |

阶段退出时不得降低基线。任何阶段暴露真实 bug 必须按 root cause 修复纪律处理。

---

## 3. 三层真相源

### 3.1 三层结构

| 层 | 文件 | 内容 | 谁消费 |
|---|---|---|---|
| machine encoding | `xisa/<arch>/isa.def` | mcinsn 编码 / operand / constraint / `:flags` / `:effect` / `:tier-input` / `:min-bytes` / `:max-bytes` / `:golden` | generator → emit/disasm/mcattr/golden |
| XmOp lowering | `xisa/lowering/<arch>.def` | `XmOp` → `mcinsn` 映射 + `:when` predicate；runtime-sensitive op 标 `:driver-lowered` | generator → `xm_dispatch_<arch>.c` |
| XmOp 元表 | `src/jit/xm_op.def` (X-macro) | `XmOp` enum + 属性（输入数 / 输出数 / 是否 patchable / 是否 may-deopt） | generator typecheck（强制 lowering 穷尽）+ driver 共享 |

### 3.2 对 archive xisa_design.md 的两条结构性强化

**强化 A：effect flag 进 schema**

```lisp
(define-mcinsn x64.call.indirect
  :operands ((($target reg:gpr64)))
  :encoding (rex.w 0xff (modrm 0b11 2 $target))
  :min-bytes 3
  :flags    (call patchable)
  :effect   (may-throw may-gc may-suspend)         ; ★ 必填
  :post-call-check (vm-pending-exception)          ; ★ 必填（may-throw 时）
  :golden   ...)
```

generator 看到 `:effect may-throw` → 自动在 driver-side helper 内插 post-call exception check。任何 `XM_CALL_*` mcinsn 的 `:effect` 缺省 → typecheck fail。

> **commit `4da96e3` 类 bug 在 build-time 直接拒**。

**强化 B：tier-input flag**

```lisp
(define-mcinsn x64.gc.scan.conservative
  :operands ((($obj reg:gpr64)))
  :tier-input conservative-scan                    ; ★ 必填
  :alignment-skip alignof-XrGcHeader               ; ★ 必填
  ...)
```

generator 看到 `:tier-input conservative-scan` → 在 emit helper 入口自动插 alignment fast-skip。任何接收 cross-tier pointer 的 mcinsn 缺省 `:tier-input` → typecheck fail。

> **commit `d94c4a8` 类 bug 在 build-time 直接拒**，并在运行期保留 alignment fast-skip 兜底。

### 3.3 generator 用纯 C

| 决策 | 内容 |
|---|---|
| 实现语言 | C99/C11，host tool |
| DSL 语法 | S-expression（仿 archive xisa_design §4） |
| bootstrap | CMake 先 build `xisagen`，再生成 header，再 build `xray_core` |
| 依赖 | host C toolchain only；不链接 `xray_core` |
| 复用 | `src/base/xarena` / `xmap` / `xstring` / `xerror` 通过 CMake `OBJECT` library 共享源（非链接 `xray_core`） |
| 三平台 | macOS / Linux / Windows MSVC clean build |

---

## 4. 实施阶段（P0 - P9）

每阶段独立可回滚。任何阶段发现 bug，按 root cause 修复并补回归。

### P0 — xisagen 底座 + .isa schema

**目标**：generator 可用，schema 三大强化（lowering / effect / tier-input）落定。

**任务**：
- [ ] 创建目录：`xisa/<arch>/`、`xisa/lowering/`、`xisa/subsets/`、`tools/xisagen/`、`tests/unit/xisagen/`
- [ ] generator 6 模块：`xisa_lex.c` / `xisa_parse.c` / `xisa_ast.c` / `xisa_typecheck.c` / `xisa_encode.c` / `xisa_emit_c.c` + `xisa_golden.c` / `xisa_host.c` / `xisa_main.c`
- [ ] schema 落定：`:operands` / `:constraints` / `:flags` / `:encoding` / `:min-bytes` / `:max-bytes` / `:golden` / **`:effect`** / **`:tier-input`** / `:post-call-check` / `:alignment-skip`
- [ ] fixture ≥ 20 条 mcinsn，覆盖：
  - x64 REX / ModR/M / SIB / `rsp/rbp/r12/r13`
  - arm64 fixed-width bitfield / `sp/xzr`
  - riscv64 R/I/S/B/U/J 基础格式
  - 至少一条 call mcinsn 带 `:effect`
  - 至少一条 GC scan mcinsn 带 `:tier-input`
- [ ] 每条 fixture ≥ 3 golden，必须用 **两种独立手段**交叉验证（手写 emit · `llvm-mc` · GNU `as` · `objdump` 任二）
- [ ] generator 单测：lex / parse / typecheck / encode 各 ≥ 5 错误路径

**退出条件**：
- 三平台 clean build
- 故意改错 fixture → build fail，诊断含 `.isa` 路径 + 行号 + 列号 + mcinsn 名 + 期望 vs 实际字节
- generator 单测 100% pass
- 不接入 JIT，仅独立 test target

**回滚锚点**：`xisa-p0`

---

### P1 — XmOp 真相表 + dispatch generation

**目标**：消除手写 `XmOp` switch；generator 强制 lowering 穷尽。

**任务**：
- [ ] 创建 `src/jit/xm_op.def`（X-macro），列出所有 `XmOp` + 属性
- [ ] 创建 `xisa/lowering/<arch>.def`，每个 `XmOp` 必须有：
  - `(define-lowering ... :backend ... :default ...)` 或
  - `(define-lowering ... :driver-lowered)`（标记 driver 自管，仍纳入元表）
- [ ] generator 输出 `build/generated/xisa/xm_dispatch_<arch>.c`：每 `XmOp` 一个 case，缺一个就 build fail
- [ ] driver 中所有 `XmOp` switch 替换为 `xm_dispatch_<arch>(ctx, ins)`
- [ ] 全文扫描：`grep -rn 'switch.*XmOp\|switch.*ins->op' src/jit/` 必须仅命中 generated 文件

**退出条件**：
- 故意从 `xisa/lowering/x64.def` 删一个 `XmOp` lowering → build fail
- 故意往 `xm_op.def` 加一个新 `XmOp` 但不补 lowering → build fail
- driver 中无手写 `switch (op)`
- ctest + regression 全过

**回滚锚点**：`xisa-p1`

---

### P2 — fail-fast 契约 + 字节自检

**目标**：补齐 P1 兜底，确保任何 emit 路径不可能 0 字节。

**任务**：
- [ ] generator 在每条 mcinsn 的 emit helper 内自动包裹 `EMIT_BEGIN(ctx) / EMIT_END(ctx, mcinsn_name, min_bytes)`
  - `EMIT_BEGIN`：记录 `pos_before = buf_offset(...)`
  - `EMIT_END`：`XR_CHECK(buf_offset(...) - pos_before >= min_bytes, "%s emitted < %d bytes", name, min_bytes)`
  - **`XR_CHECK`**（Release 也保留），不是 `XR_DCHECK`
- [ ] driver 中所有非 generated emit 入口（仍存在的 patch / stub helper）也强制走相同宏
- [ ] 编译选项：`-Wswitch-enum -Werror=switch-enum`（CI 强制）
- [ ] 兜底 `default: XR_UNREACHABLE("...")` 加到 generated 与 driver 中所有 enum switch（即便已 P1 穷尽）

**退出条件**：
- 故意改一个 mcinsn 让其漏 emit → 第一次 JIT 编译时 `XR_CHECK` fail
- 故意 comment 掉一个 `XmOp` case → build 阶段 `-Werror=switch-enum` 拒
- Win11 Release 重跑 commit `8b06997` 复现路径，确认仍正常
- ctest + regression 全过

**回滚锚点**：`xisa-p2`

---

### P3 — x64 machine encoding 全面迁移

**目标**：x64 全部 machine encoding 走 generated helper；删除手写 byte 拼装。

**范围**：
- integer arithmetic / mov / cmp / test / lea
- load / store（含 SIB / RIP-relative / `rsp/rbp/r12/r13` 特殊形）
- branch / jcc / jmp / call / ret
- push / pop
- 所有 xray 实际 emit 的 x64 opcode

**migration 流程**（每条 mcinsn）：

```text
PR-N   : 把 mcinsn X 加入 xisa/x64/isa.def
         开 dual-emit (XR_ENABLE_XISA_DUAL_EMIT=1)
         CI macOS+Linux Debug+ASAN 跑 ctest + regression，0 分歧
PR-N+1 : 删除 X 对应的手写 emit 函数 + 所有 call site
         driver 直接调 generated helper
         关闭对应 dual-emit 路径
```

**禁止**：
- 长期保留两套 emit 并存
- "如果 generated 出问题就 fallback 到手写"的兜底

**任务**：
- [ ] 完整覆盖 `xm_x64.h` 暴露的所有 emit 入口
- [ ] dual-emit 0 分歧后立即删除手写
- [ ] `xm_x64.c` 中"手写 byte 拼装"行数清零（仅保留 patch helper / branch reach / driver-only utility）
- [ ] 全文 `grep -rn '\*buf++ = 0x\|\*buf\+\+ = .*\(uint8' src/jit/xm_x64.c` 命中数 = 0

**退出条件**：
- migration 期 dual-emit 0 分歧
- 删除手写后 ctest + regression + Win11 不退行
- x64 stress 24h 无新 bug；ASAN 跑一轮 regression 无新 leak / heap-buffer-overflow
- migration 期发现的手写 bug 已修 root cause 并补 golden

**回滚锚点**：`xisa-p3`

---

### P4 — arm64 machine encoding 全面迁移

**目标**：同 P3，覆盖 arm64。bitfield 与 logical immediate 单独 golden。

**任务**：
- [ ] 完整覆盖 `xm_arm64.h` 暴露的所有 emit 入口
- [ ] bitfield / logical imm / shifted imm 单独 golden 集
- [ ] dual-emit 流程同 P3
- [ ] `xm_arm64.c` 中"手写 byte 拼装"行数清零

**退出条件**：
- 同 P3 标准
- arm64 JIT stress 24h 无新 bug
- scalar `.isa` interpreter（P6 引入）与两后端联动 0 分歧的前置准备到位

**回滚锚点**：`xisa-p4`

---

### P5 — disassembler & assembler test framework

**目标**：generator 输出 disassembler；Dart 风格 macro pair 落地；byte-level 覆盖每条 mcinsn。

**任务**：
- [ ] generator 新增 `xisa_emit_disasm.c` 模块，从 `.isa` 反向生成 `build/generated/xisa/xisa_disasm_<arch>.h`（header-only，纯函数）
- [ ] disassembler 接口：
  ```c
  size_t xisa_disasm_one_<arch>(const uint8_t *code, size_t code_size,
                                size_t offset, char *buf, size_t buf_size);
  void   xisa_disasm_region_<arch>(const uint8_t *code, size_t size,
                                   char *buf, size_t buf_size);
  ```
- [ ] 输出 Intel asm 风格（x64）/ AArch64 standard 风格（arm64），与 `llvm-mc -disassemble` normalize 后对齐
- [ ] 未识别字节优雅降级 `<unknown bytes: XX XX XX>`，不 abort
- [ ] 创建 `tests/unit/jit/assembler_test.h`：`ASSEMBLER_TEST_GENERATE_<arch>` / `ASSEMBLER_TEST_RUN_<arch>` / `EXPECT_DISASSEMBLY_<arch>`
- [ ] 每条 mcinsn ≥ 3 case：普通寄存器 / 扩展寄存器 / 特殊寄存器（`rsp/rbp/r12/r13` / `sp/xzr` / `x0/zero`）
- [ ] mem form 单独覆盖：`[base]` / `[base+disp8]` / `[base+disp32]` / `[base+index*scale]` / `[rsp+...]`
- [ ] 立即数边界：i8 / i32 / i64 各形式

**退出条件**：
- 100% mcinsn 反汇编 round-trip 通过（emit → disasm → expected string）
- 故意改错某 mcinsn 1 字节 → 对应 test fail，诊断含 mismatch line + offset
- 三平台 build pass
- ctest + regression 不退行

**回滚锚点**：`xisa-p5`

---

### P6 — `llvm-mc` differential + scalar interpreter

**目标**：第三方 reference + 自家 scalar interpreter 三方对比。

**任务**：
- [ ] 创建 `tests/unit/jit/test_assembler_diff.c`：
  - random program generator：每 PR 1k seed，nightly 1M seed
  - 每个 seed → emit via xray → bytes → 双 disasm（xray + `llvm-mc`）→ normalize → diff
  - 不一致 → fail-fast，dump bytes + both texts + 复现 seed
- [ ] 创建 `xisa/subsets/scalar.isa`：纯 arithmetic / compare / bitwise / select / mock load-store
- [ ] generator 输出 `xisa_scalar_interp.c`（最小 interpreter）
- [ ] random scalar program generator：x64 / arm64 / interpreter 三方对比
- [ ] CI：macOS + Linux runner 跑（Windows MSVC 不要求 `llvm-mc`，本地 dev 用 `scripts/codegen_diff.sh`）
- [ ] LLVM 版本 pin；normalize 层做版本兼容

**退出条件**：
- macOS / Linux PR 1k seed 0 分歧
- nightly 1M seed 4 周 0 分歧
- 故意改错某 mcinsn 1 bit → CI fail 且 seed 可独立复现
- runtime-sensitive op 显式标 non-interpretable，不进 scalar diff

**回滚锚点**：`xisa-p6`

---

### P7 — runtime parity 7 子项

**目标**：runtime-sensitive 路径单独 audit + parity test。每子项独立 DoD。

#### P7.1 call ABI parity

- [ ] 4 类 call (`call` / `call-known` / `call-c-helper` / `call-self-direct`) × 2 backend = 8 子集 parity
- [ ] argument / return / live-across-call 行为对等
- **DoD**：8 子集 0 分歧

#### P7.2 safepoint stack map

- [ ] metadata verifier 比对每个 JIT 导出的 stack map 的 slot ID / spill offset / PC range
- [ ] GC stress 下 safepoint poll 不丢根 / 不误根
- **DoD**：`scripts/run_regression_tests.sh` + GC stress 30min 无新 leak / heap-buffer-overflow / use-after-free

#### P7.3 deopt spill / register reconstruction

- [ ] 同一 deopt 点在 x64 / arm64 上重建后的 VM register state byte-equal
- [ ] 覆盖 spill-register live 混合、同函数多点 deopt、deopt + post-call resume
- **DoD**：两后端 observable state 一致；不读未初始化 spill slot

#### P7.4 OSR entry

- [ ] 复用 `test_osr_entry_pressure` 框架拓展多点 / 多参 / spill-pressure 场景
- **DoD**：x64 / arm64 OSR 重入 ≥ 4 轮不退行；多参 / spill-pressure byte-equal observable state

#### P7.5 suspend / resume

- [ ] 协程 suspend/resume parity test：调度轮序 / yield 点 / GC 根 / channel 状态在两后端一致
- [ ] Win11 Release `1115_cancel.xr` / `1109_await_any.xr` / `1127_coro_priority.xr` / `1128_yield.xr` 无 `STATUS_HEAP_CORRUPTION`
- **DoD**：以上四用例两后端 Debug + Release 0 复现 heap 错误

#### P7.6 effect flag completeness audit

> 因 §3.2 强化 A，此项**自动 build-time 检查**，本子项主要做"audit 已存量代码 + 补回归测试"。

- [ ] 全文扫描 `XM_CALL_*` 发出点，确认每个都通过 generated helper（`:effect` 已声明）
- [ ] unknown C bridge 默认 conservative：`(may-throw may-gc may-suspend)` 三 flag 全置；显式声明"纯计算 helper"才能去 flag
- [ ] 在 `:effect` 元件里加 `:purity-asserted` 子句证明无副作用
- [ ] 补 commit `4da96e3` 的回归测试（嵌套异常 + JIT call bridge）
- **DoD**：故意从某 mcinsn 删 `:effect may-throw` → typecheck fail；commit `4da96e3` 回归测试在 macOS Debug 通过；x64 / arm64 effect flag 设置在 generated 文件中 byte-equal

#### P7.7 cross-tier pointer hygiene audit

> 因 §3.2 强化 B，此项**自动 build-time 检查 + 运行期 fast-skip**，本子项主要做"audit 已存量入口 + 写 fuzz"。

- [ ] 全文扫描 cross-tier pointer 入口：保守扫描（`mark_coro_roots`）/ write-barrier / safepoint stack scan / deopt slot 重建 / `shared_refs[]` 流转
- [ ] 每个入口必须经过带 `:tier-input` 的 mcinsn；不存在直接 deref 的兜底路径
- [ ] `obj->type` 越界检查覆盖 sweep / mark / decref 三入口
- [ ] 周期性 `gc.collect()` × ASAN × `--jit-force` × 50 次跑 `1206_gc_enhanced.xr`
- **DoD**：故意删一个 fast-skip → 30min 内 ASAN fuzz 必复现；`1206_gc_enhanced.xr` × `--jit-force` × ASAN × 50 次 0 失败

**总阶段退出标准**：
- P7.1 - P7.7 全部 DoD 满足
- 期间发现的真实 bug 已修 root cause 并补回归

**回滚锚点**：`xisa-p7`

---

### P8 — fuzz 矩阵 4 层

**目标**：长期 nightly 防止 rare encoding bug 漂移；捕获 cross-tier corruption chain。

**4 层**：

```text
Layer 1: assembler fuzz
  random (reg, imm, mem-form) → emit → disasm → llvm-mc 对比

Layer 2: codegen fuzz
  random small Xm IR program → JIT 编译 → 真机执行 → interpreter 对比

Layer 3: driver invariant fuzz
  random IR op sequence → 验证每条 IR op emit ≥ 1 字节 + effect flag 一致

Layer 4: corruption chain fuzz
  random IR × --jit-force × ASAN × 周期性 gc.collect() × 30min
  扫堆验证所有 GC header `type` 字段在合法枚举范围
```

**任务**：
- [ ] 4 层独立 CI 任务，nightly 1M seed
- [ ] shrinker：失败 seed 自动缩小到 ≤ 5 节点 IR / 30 秒
- [ ] CI 指标：每层失败 seed 数 / 总 seed / 缩小后 IR 节点中位数 / corruption chain depth
- [ ] chain depth ≥ 3 步标 high-priority

**退出条件**：
- 4 层各 nightly 1M seed 4 周 0 分歧
- 故意 inject 1 emit bug → shrinker 30 秒缩小到 ≤ 5 节点
- 故意 inject 1 错位指针 → corruption chain fuzz 30min 内必触发
- ctest + regression 不退行

**回滚锚点**：`xisa-p8`

---

### P9 — RISC-V subset bring-up

**目标**：用 xisa 写 RV64 subset backend，QEMU 跑通最小 JIT。

**范围**（仅 v1）：
- integer arithmetic
- load / store
- branch / jump
- simple call / return
- basic prologue / epilogue

**不要求**：
- OSR / deopt / safepoint full protocol
- coroutine suspend/resume
- full regression parity
- compressed instruction extension

**任务**：
- [ ] `xisa/riscv64/isa.def` 覆盖 RV64 base scalar subset
- [ ] `xisa/lowering/riscv64.def` 覆盖 v1 范围 op；其余标 `:driver-lowered` + `unimplemented`
- [ ] 写 RISC-V driver skeleton（仅 v1 范围）
- [ ] QEMU 上跑 JIT scalar unit subset
- [ ] scalar differential 加入 RISC-V

**退出条件**：
- QEMU RISC-V 上 JIT scalar unit tests 全过
- scalar diff x64 / arm64 / rv64 / interpreter 0 分歧
- runtime parity 缺口写入 `docs/known_bugs.md`，不阻塞 P9 退出

**回滚锚点**：`xisa-p9`

---

## 5. 关键 milestone

| Milestone | 何时 | 评估什么 | 决策点 |
|---|---|---|---|
| M0 schema 表达力 | P0 | 20 条 mcinsn 三 arch 三类寄存器是否能表达？ | 不能 → 修 schema |
| M1 lowering 穷尽 | P1 | 删一个 lowering 是否 build fail？ | 否 → 修 generator |
| M2 commit `8b06997` 根除 | P2 | 字节自检是否覆盖所有路径？ | 不全 → 补 |
| M3 x64 手写归零 | P3 | 手写 byte 拼装行数 = 0？ | 否 → 继续迁移 |
| M4 arm64 手写归零 | P4 | 同上 | 同上 |
| M5 disassembler 可用 | P5 | 100% mcinsn round-trip？ | 不全 → 修 |
| M6 第三方一致 | P6 | `llvm-mc` 1k seed 0 分歧？ | 有分歧 → 修 root cause |
| M7 runtime parity | P7 | 7 子项 DoD 全过？ | 缺一不可 |
| M7.1 commit `4da96e3` 根除 | P7.6 | typecheck 是否拦 missing `:effect`？ | 否 → 修 generator |
| M7.2 commit `d94c4a8` 根除 | P7.7 | typecheck 是否拦 missing `:tier-input`？ | 否 → 修 generator |
| M8 fuzz 稳定 | P8 | 4 层 nightly 4 周 0 分歧？ | 有 → triage |
| M9 RISC-V subset | P9 | QEMU JIT scalar 全过？ | 失败 → 复盘 |

---

## 6. 风险登记册

| ID | 风险 | 概率 | 影响 | 应对 |
|---|---|---|---|---|
| R1 | generator 自身 bug 把 schema 错误固化为 generated 错误 | 中 | 高 | dual-emit migration 期保护；generator ≥ 20 错误路径单测；任何 dual-emit 分歧先修 root cause |
| R2 | DSL 表达力不够覆盖 runtime-sensitive op | 高 | 中 | `:driver-lowered` 显式标记；driver 仍写 lowering 序列；底层 mcinsn 仍由 generator 输出 |
| R3 | Windows MSVC 上 generator / generated header 行为飘移 | 中 | 高 | P0 退出强制三平台 build；CMake 不用 unix-only flag |
| R4 | `llvm-mc` 版本飘移导致 differential 假阳性 | 低 | 中 | pin LLVM 版本；normalize 层版本兼容；CI 报版本号 |
| R5 | P3/P4 dual-emit 暴露大量历史手写 bug | 高 | 中 | 先修 root cause，加 golden，再删手写；不让 generator 适配错误 |
| R6 | P7 runtime parity 暴露非平凡 bug | 高 | 高 | 7 子项独立 DoD；按 root cause 修复；不允许 known_failures 长期屏蔽 |
| R7 | fuzz × ASAN × `--jit-force` × GC stress 暴露 bug 超过修复带宽 | 中 | 高 | shrinker 输出按 root cause 分组；写 `docs/known_bugs.md`；按 bug 修复纪律逐个处理 |
| R8 | xisa lowering 强制穷尽导致大量 `:driver-lowered` stub | 中 | 低 | 接受；`:driver-lowered` 是显式契约不是 escape hatch；每个必须有 parity test |
| R9 | RISC-V 暴露 schema 不通用 | 中 | 中 | P9 失败仅 RISC-V 暂停，不影响主流程；schema 改动必须同步 fixture |
| R10 | DSL 错误诊断质量差，写 `.isa` 时调试成本超过手写 | 中 | 中 | P0 退出条件含行号 + 列号 + mcinsn 名 + 期望/实际字节四要素；后续可加 suggested fix |
| R11 | 团队仍倾向静默 fallback / 手写 byte 拼装 | 中 | 高 | P3/P4 退出条件含 grep 反向不变量；PR template / code review checklist 固化反向不变量 |
| R12 | generator 与 driver 共享的 `xm_op.def` 形态选择 | 低 | 中 | X-macro 方式；generator 与 driver 都 include 同一文件；任何修改必须同时更新 lowering |

---

## 7. 完成判定

### 7.1 v1 完成（P0 - P6）

| 指标 | 目标 |
|---|---|
| `xm_x64.c` / `xm_arm64.c` 中"手写 byte 拼装" | 0 行（grep 反向不变量） |
| XmOp dispatch | 100% generated；driver 中无手写 `switch (op)` |
| `:effect` / `:tier-input` | 必填，typecheck pass = 100% audit clean |
| commit `8b06997` 类 bug | build-time 拒（lowering 穷尽 + min-bytes typecheck） |
| commit `4da96e3` 类 bug | build-time 拒（`:effect` 漏标 typecheck fail） |
| commit `d94c4a8` 类 bug | build-time 拒 + 运行期 alignment fast-skip 双重防御 |
| dual-emit | migration 完毕后整子模块删除（不存在长期共存） |
| ctest / regression / Win11 | 三平台 100%，不退行 |
| `llvm-mc` differential | 1k seed 0 分歧 |
| disassembler | 100% mcinsn round-trip |

### 7.2 完整完成（P0 - P9）

| 指标 | 目标 |
|---|---|
| runtime parity 7 子项 | 全 DoD 满足 |
| fuzz 4 层 nightly | 1M seed 4 周 0 分歧 |
| corruption chain fuzz × ASAN × `--jit-force` × GC stress | nightly 30min 0 复现 |
| Win11 Release 协程 4 用例 | 0 `STATUS_HEAP_CORRUPTION` |
| RISC-V subset | QEMU JIT scalar 全过；三后端 + interpreter 0 分歧 |

### 7.3 反向不变量（任何阶段都不能违反）

每条都需 CI script + grep 强制：

| # | 不变量 | 检查方法 |
|---|---|---|
| 1 | 不存在两套 emit 并存（migration 期之外） | `grep` 手写 emit fn 在 P3/P4 之后行数 = 0 |
| 2 | 不存在 `default:` 静默 fallback | `grep` codegen 中 `default:\s*break` 命中 = 0 |
| 3 | 不存在 unknown C bridge 无 conservative effect flag | typecheck pass = 100% |
| 4 | 不存在 cross-tier pointer 解引用前无 alignment fast-skip | typecheck pass = 100% |
| 5 | 不存在 disassembler 与 emit 用不同真相源 | disassembler 由 generator 输出 |
| 6 | driver 不直接写 byte | `grep` `*buf++ = 0x` 在 driver 中命中 = 0 |
| 7 | XmOp 真相唯一 | `xm_op.def` 是唯一 enum 真相，generator + driver 共享 |

### 7.4 不强求

- SIMD / VEX / EVEX：非目标
- instruction scheduler / pattern fusion：非目标
- AOT C backend：与 xisa 正交
- VM bytecode：与 xisa 正交
- Win64 unwind 信息：独立 follow-up

---

## 8. 与其他 task 的关系

| 关联 task | 关系 |
|---|---|
| `008-jit-multi-backend.md` | xisa 是其 machine encoding 层的实施载体 |
| `009-jit-x64-parity.md` | parity 进入 P7.1-P7.5；effect/pointer hygiene 进入 P7.6/P7.7 |
| `068-compiler-pipeline-optimization.md` | Xi → Xm 不在 xisa 范围；xisa 只覆盖 Xm → machine code |
| `070-regression-bug-triage.md` | codegen 类 bug 可转成 golden / differential / unit test 固定样例 |
| `073-codegen-hardening.md` | 本 task 完整继承其纪律（fail-fast / disasm / differential / parity / fuzz） |
| `074-codegen-final-plan.md` | 平行独立第二方案；不互引、不互依赖 |
| `archive/073-xisa-implementation.md` | 本 task 继承其 xisa DSL 思想；schema 在此基础上扩两条强化 |
| `archive/xisa_design.md` | 本 task 是其设计的实施延续；扩 effect / tier-input / lowering 三大强化 |

---

## 9. 文件清单（最终落地后）

```text
xisa/                                    ← schema 真相源
├── x64/isa.def
├── arm64/isa.def
├── riscv64/isa.def
├── lowering/
│   ├── x64.def
│   ├── arm64.def
│   └── riscv64.def
└── subsets/scalar.isa

tools/xisagen/                           ← 纯 C host generator
├── xisa_main.c
├── xisa_lex.c                           (仿 src/base/xtoml.c)
├── xisa_parse.c
├── xisa_ast.c
├── xisa_typecheck.c                     (含 effect / tier-input 强制)
├── xisa_encode.c
├── xisa_emit_c.c                        (输出 emit / disasm / mcattr / dispatch)
├── xisa_golden.c
└── xisa_host.c

build/generated/xisa/                    ← .gitignore；CMake 产出
├── xisa_emit_<arch>.h
├── xisa_disasm_<arch>.h
├── xisa_mcattr_<arch>.h
├── xm_dispatch_<arch>.c
└── xisa_golden_<arch>.c

src/jit/                                 ← driver-only
├── xm_op.def                            (XmOp X-macro 真相)
├── xm_codegen_<arch>*.c                 (frame / ABI / patch / OSR / deopt / suspend)
└── (xm_x64.c / xm_arm64.c 中"手写 byte 拼装"全部删除)

tests/unit/xisagen/                      ← generator 自身单测
tests/unit/jit/                          ← assembler / runtime parity / disasm
tests/fuzz/jit/                          ← 4 层 fuzz
```

---

## 10. 资源与时间估算

| 阶段 | 估算 | 备注 |
|---|---|---|
| P0 | 2-3 周 | C generator + schema 强化 + fixture |
| P1 | 1 周 | XmOp 真相表 + dispatch generation |
| P2 | 1 周 | fail-fast + 字节自检 |
| P3 | 2-3 周 | x64 全面迁移（dual-emit 流水） |
| P4 | 2-3 周 | arm64 全面迁移 |
| P5 | 2 周 | disassembler + assembler test framework |
| P6 | 1-2 周 | `llvm-mc` differential + scalar interpreter |
| P7 | 4-6 周 | runtime parity 7 子项 |
| P8 | 1-2 周 | fuzz 4 层 nightly |
| P9 | 2-3 周 | RISC-V subset bring-up |

**总计**：18-26 周。

> P0-P6 12-16 周后已拿到完整 codegen build-time 健壮性 + 第三方 reference 一致性。
> P7-P9 与其他 task 可并行推进。

---

## 11. 修订历史

| 日期 | 改动 | 作者 |
|---|---|---|
| 2026-05-12 | 初稿；独立第二方案；继承 073-codegen-hardening 全部纪律；扩 xisa schema 两条强化（`:effect` / `:tier-input`）；XmOp 真相表 + lowering 穷尽 build-time 检查；不留 dual-emit 长期共存 / 不留 fallback / 不做兼容层 | Cascade + xingleixu |
