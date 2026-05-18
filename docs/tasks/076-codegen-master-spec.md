# 076 — JIT codegen 最终 master 规约

> **作者**: Cascade (pair-programming with Xinglei Xu)
> **日期**: 2026-05-12
> **状态**: planned
> **定位**: 取代 073 / 074 / 075，作为后续 codegen / xisa / multi-backend 工作的**唯一**实施依据
> **核心原则**: Xray 没有外部用户，不做向后兼容，不保留旧接口，不做兼容层；每个阶段直接采用最佳设计
> **来源依据**:
>   - `docs/archive/073-codegen-hardening.md`（fail-fast / disasm / differential / parity / fuzz / effect / pointer 纪律）
>   - `docs/archive/074-codegen-final-plan.md`（XmOp contract / helper registry / decode priority / external reference / 删除清单）
>   - `docs/archive/075-codegen-final-spec.md`（原迁移纪律 / git tag / grep 反向不变量 / 时间估算 / 字节自检独立阶段；本文收紧为 test-only oracle + 单 PR 删除）
>   - `docs/archive/073-xisa-implementation.md`（C generator / S-expression DSL / encoding-first）
>   - `docs/archive/xisa_design.md`（DSL 语法 / 三 arch encoding 元件）
> **直接动机**:
>   - commit `8b06997` — x64 RT-opcode 漏 emit → Win11 `STATUS_HEAP_CORRUPTION`
>   - commit `4da96e3` — 跨边界 effect flag 漏标 → 内层 VM 异常 unwind 越外层 catch
>   - commit `d94c4a8` — 保守扫描接受错位指针 → 7 步 corruption chain
>   - commit `50a187d` (Bug C) — 多 Pass invariant 漂移 → scope 父链不一致

---

## 0. 总览与立论

### 0.1 最终决策

```text
L1: xisa/xm/ops.def                ← XmOp 契约表（enum/arity/effect/lowering/backend-coverage）
L2: xisa/xm/helpers.def            ← helper registry（id/ABI/effect/pointer-trust/post-call）
L3: xisa/arch/<arch>/isa.isa       ← machine instruction schema（operands/encoding/golden/disasm）
       ↓
   tools/xisagen/   ← C host generator（CMake build-time）
       ↓
   build/generated/xisa/
   ├── xm_ops.h / xm_op_table.c / xm_op_verify.c     (L1 生成物)
   ├── xm_helpers.h / xm_helper_table.c              (L2 生成物)
   ├── xisa_emit_<arch>.h / xisa_mcattr_<arch>.h     (L3 生成物)
   ├── xisa_disasm_<arch>.h                          (L3 反向生成)
   ├── xm_dispatch_<arch>.c                          (lowering 生成)
   └── xisa_golden_<arch>.c                          (golden 生成)
       ↓
   src/jit/xm_codegen_<arch>*.c   ← driver（frame/ABI/patch/OSR/deopt/suspend only）
       ↓
   code buffer + stack maps + deopt metadata + OSR metadata
```

### 0.2 核心立论

**xisa 是 JIT 机器编码、disassembler、XmOp 元表、helper 元表的唯一真相源**。所有手写 byte 拼装、手写 disassembler、手写 XmOp switch、手写 helper effect 推断 **全部删除**。runtime-sensitive 路径（call/deopt/OSR/safepoint/suspend）保留 driver 形态，但 driver **不允许出现裸 byte 拼装、不允许漏标 effect / pointer trust、不允许 silent fallback**。codegen-hardening 的全部纪律作为 xisa 的外围验证矩阵强制叠加。

### 0.3 双层防御

helper effect 与 pointer trust **同时在两层声明**，互为兜底：

- **L2 helper registry**：声明默认 `:effects` 与 `:pointer-trust`，作为真相源
- **L3 mcinsn schema**：通过 `:helper` 引用 L2，自动继承；想 override 必须显式 `:effect-override` 且配 audit 测试

任何漏标 / 不一致 → generator typecheck fail。运行期仍保留 `EMIT_BEGIN/END` 字节自检与 alignment fast-skip 作为运行时兜底。

### 0.4 不兼容原则

| 规则 | 含义 |
|---|---|
| 不保留旧接口 | 被 generated helper 替换的手写 emit / disasm / metadata 必须删除 |
| 不做兼容层 | 不保留 `old_emit → new_emit` 或反向包装 |
| dual-emit 不进主线 | 仅 test target / 短分支作为 oracle；替换合并前必须删除对应手写路径 |
| 不允许 silent fallback | unhandled op / unknown helper / unknown mcinsn 必须 hard fail |
| 不允许 codegen fallback 掩盖缺口 | Xi → Xm 可拒绝 JIT；Xm → machine 一旦开始必须 hard fail，不能回 VM 掩盖 backend bug |
| 不允许 raw helper pointer 作为语义来源 | 必须用 helper id + L2 registry |
| 不允许手写裸字节 | xisa 覆盖的指令不得在 driver 中手写 byte / bitfield |
| 不允许生成物进 git | generated 只存在于 build/ |
| 不为旧测试让路 | 旧测试依赖错误行为 → 修测试 + 修 root cause |
| 不允许伪 coverage | backend 不支持的 op 由 eligibility 拒绝，不用 `:unimplemented` 伪装 lowering |

---

## 1. 三层真相源

### 1.1 源码目录布局

`.def` / `.isa` 是 codegen 真相源源码，必须进 git；generated 只进入 build tree。

```text
xisa/
├── xm/
│   ├── ops.def           ← L1: XmOp 契约表
│   └── helpers.def       ← L2: helper registry
├── arch/
│   ├── x64/
│   │   ├── isa.isa       ← L3: x64 machine instruction schema
│   │   └── abi.def       ← 可选：register class / ABI / calling convention
│   ├── arm64/
│   │   ├── isa.isa
│   │   └── abi.def
│   └── riscv64/
│       ├── isa.isa
│       └── abi.def
└── subsets/
    ├── scalar.isa        ← differential interpreter / scalar subset
    └── memory.isa
```

放置规则：
- `xisa/` 是 repo 顶层目录，与 `src/` / `include/` / `tools/` 平级。
- `tools/xisagen/` 只放 generator 实现，不放 `.def` / `.isa` 真相源。
- `build/generated/xisa/` 只放生成物，不进 git。
- 不放 `src/jit/`，避免把 build-time spec 误当成 JIT 私有实现。
- 不放 `docs/`，因为 `.def` / `.isa` 参与构建，是源码，不是说明文档。

### 1.2 L1 — `xisa/xm/ops.def`（XmOp 契约表）

每 `XmOp` 必须声明：

```lisp
(define-xm-op xm.add
  :operands   ((reg in) (reg in))
  :results    ((reg out))
  :effects    ()                              ; pure
  :class      pure                            ; pure | branch | call | runtime | deopt | osr | safepoint | suspend
  :lowering   (:x64     (x64.add.rr $r $0 $1))
              (:arm64   (arm64.add.rr $r $0 $1))
              (:riscv64 (rv64.add.rr $r $0 $1)))

(define-xm-op xm.call.indirect
  :operands   ((reg in) (varargs in))
  :results    ((reg out))
  :effects    (may-throw may-gc may-suspend)
  :class      call
  :lowering   (:x64     :driver-lowered)      ; runtime-sensitive 显式
              (:arm64   :driver-lowered)
              (:riscv64 :driver-lowered))
```

generator 强制：
- 每 `XmOp` × 每 backend 必须有 lowering 或 `:driver-lowered`，缺一个 → build fail
- `:driver-lowered` 必须有配套 parity test，否则 verifier fail
- enum / name / arity / effect / class 全部由 generator 输出

**生成物**：
```text
build/generated/xisa/xm_ops.h          ← XmOp enum + 元数据访问宏
build/generated/xisa/xm_op_table.c     ← 元数据查表
build/generated/xisa/xm_op_verify.c    ← backend coverage / handler 检查
build/generated/xisa/xm_dispatch_<arch>.c  ← lowering dispatch
```

**删除**：手写 `XmOp` enum、name table、arity table、默认 effect。

`driver-lowered` 不是兼容逃生口。每个 `:driver-lowered` op 必须额外声明：
- required metadata：deopt / stackmap / safepoint / OSR / suspend 等。
- allowed helpers：只能引用 L2 helper id。
- allowed mcinsns：底层机器指令仍必须走 L3 generated helper。
- post-call protocol：必须能被 verifier 检查。

示例：

```lisp
(define-xm-op xm.call.indirect
  :operands   ((reg in) (varargs in))
  :results    ((reg out))
  :effects    (may-throw may-gc may-suspend)
  :class      call
  :lowering   (:x64   :driver-lowered)
              (:arm64 :driver-lowered)
  :driver-contract
              (:metadata (stackmap deopt safepoint)
               :helpers  (xrt_vm_call_indirect)
               :mcinsns  (x64.mov.rr x64.call.helper x64.cmp.ri x64.jcc.rel32)
               :post-call (vm-pending-exception safepoint-poll)))
```

### 1.3 L2 — `xisa/xm/helpers.def`（helper registry）

每 C helper / runtime bridge 必须声明：

```lisp
(define-helper xrt_vm_call_indirect
  :c-symbol       "xrt_vm_call_indirect"
  :abi            (callee-saves rsp rbp ...)
  :return         reg
  :params         (reg reg reg)
  :effects        (may-throw may-gc may-suspend)
  :pointer-trust  external
  :post-call      (vm-pending-exception safepoint-poll))

(define-helper xrt_arith_add_i64
  :c-symbol       "xrt_arith_add_i64"
  :abi            (sysv-amd64)
  :return         reg
  :params         (reg reg)
  :effects        ()                          ; pure，可省 default
  :pointer-trust  none
  :post-call      ())
```

generator 强制：
- unknown helper 不允许进入 codegen（typecheck）
- 默认 effect 是 conservative：`(may-throw may-gc may-suspend)`
- 默认 `:pointer-trust` 是 `external`
- 去 flag 必须显式 `:effect-override (purity-asserted)` + audit 测试
- may-enter-vm / may-run-user-code / may-reenter-jit / needs-stackmap / exception-protocol / suspend-protocol 这类 runtime 协议必须显式声明，不能由 call site 推断

**生成物**：
```text
build/generated/xisa/xm_helpers.h        ← helper id enum + ABI 元数据
build/generated/xisa/xm_helper_table.c   ← id → C symbol / ABI / effect 查表
build/generated/xisa/xm_helper_verify.c  ← call site effect / trust 检查
```

**删除**：raw fn pointer effect 推断、手写 helper effect 默认表。

### 1.4 L3 — `xisa/arch/<arch>/isa.isa`（machine instruction schema）

每 mcinsn 必须声明：

```lisp
(define-mcinsn x64.add.rr
  :operands       (($dst reg:gpr64) ($src reg:gpr64))
  :constraints    ((same $dst $src))
  :encoding       (rex.w $src $dst 0x01 (modrm 0b11 $src $dst))
  :flags          ()
  :min-bytes      3
  :max-bytes      3
  :golden-bytes   (((rax rbx) "48 01 d8")
                   ((r12 r13) "4d 01 ec")
                   ((rsp rbp) "48 01 ec"))
  :golden-asm     (((rax rbx) "add rax, rbx")
                   ((r12 r13) "add r12, r13")
                   ((rsp rbp) "add rsp, rbp"))
  :decode-priority high                     ; 解 x64 编码别名时优先级
  :external-reference llvm-mc)              ; golden 来源标记

(define-mcinsn x64.call.helper
  :operands       ((($helper helper-id)))
  :encoding       (rex.w 0xff (modrm 0b11 2 $helper))
  :flags          (call patchable)
  :helper         $helper                   ; ← 从 L2 继承 effects / trust
  :effect-override ()                       ; 想去 flag 必须显式 + audit
  :post-call-check (vm-pending-exception)
  :min-bytes      3
  :golden-bytes   ...)

(define-mcinsn x64.gc.scan.conservative
  :operands       ((($obj reg:gpr64)))
  :encoding       ...
  :tier-input     conservative-scan         ; ← L3 层 pointer trust override
  :alignment-skip alignof-XrGcHeader        ; ← 自动生成 fast-skip
  ...)
```

generator 强制：
- 凡 call 类 mcinsn 必须有 `:helper` 引用 L2，effects 自动继承
- 凡接受 cross-tier pointer 的 mcinsn 必须显式 `:tier-input` + `:alignment-skip`，否则 typecheck fail
- 任何 `:effect-override` 必须有配套 audit 测试（generator 自动生成框架）

**新增字段（相对 archive xisa_design.md）**：
- `:helper` — 引用 L2 helper id
- `:effect-override` — 强制声明去 flag
- `:tier-input` — pointer trust 显式声明
- `:alignment-skip` — 自动生成 fast-skip
- `:golden-asm` — disassembler round-trip 用
- `:decode-priority` — 解编码别名
- `:external-reference` — golden 来源标

**生成物**：
```text
build/generated/xisa/xisa_emit_<arch>.h     ← typed emit helper
build/generated/xisa/xisa_disasm_<arch>.h   ← 反向生成 disassembler
build/generated/xisa/xisa_mcattr_<arch>.h   ← min/max-bytes/flags 编译期常量
build/generated/xisa/xisa_golden_<arch>.c   ← build-time golden test
```

**删除**：`xm_x64.c` / `xm_arm64.c` 中所有手写 byte 拼装；所有手写 emitted-subset disassembler。

### 1.5 subset 拆分

```text
xisa/subsets/scalar.isa      ← pure arithmetic / compare / bitwise / select
xisa/subsets/memory.isa      ← mock load/store / addressing
```

用于 differential interpreter；runtime-sensitive op 不进 subset。

---

## 2. Generator 设计

### 2.1 实现语言：纯 C

| 决策 | 内容 |
|---|---|
| 实现语言 | C99/C11 |
| 形态 | CMake host tool（`tools/xisagen/`） |
| DSL 语法 | S-expression |
| bootstrap | CMake 先 build `xisagen`，再生成 header，再 build `xray_core` |
| 依赖 | host C toolchain only；不链接 `xray_core` / runtime / VM / JIT |
| 复用 | `src/base/xarena` / `xmap` / `xstring` / `xerror` 通过 CMake `OBJECT` library 共享源 |
| 三平台 | macOS / Linux / Windows MSVC clean build |
| 内存 | 复用 `xr_malloc` / `xr_arena`，不直接 `malloc/free` |

**不用**：Python / Rust / Go / Lua / Zig / C++ / xray 自举（理由见 `docs/archive/074-codegen-final-plan.md §10` + `Comparing Codegen Hardening Plans.md`）。

### 2.2 模块拆分

```text
tools/xisagen/
├── xisa_main.c
├── xisa_lex.c              (S-expression lexer，仿 src/base/xtoml.c)
├── xisa_parse.c            (递归下降 parser)
├── xisa_ast.c              (AST + arena)
├── xisa_typecheck.c        (含 L1/L2/L3 三层一致性检查)
├── xisa_encode.c           (encoding 元件求值)
├── xisa_decode.c           (反向 → disassembler 生成)
├── xisa_emit_c.c           (输出 emit helper)
├── xisa_emit_disasm.c      (输出 disassembler)
├── xisa_emit_ops.c         (输出 XmOp 元表 + dispatch)
├── xisa_emit_helpers.c     (输出 helper registry)
├── xisa_golden.c           (输出 golden test + 跑 build-time golden)
├── xisa_diag.c             (诊断输出：行+列+mcinsn+operands+expected/actual)
└── xisa_host.c             (CLI / file IO)
```

### 2.3 生成物约束

- emit helper 是 `static inline` 或内部 C 函数，不分配内存
- emit helper 返回实际写入字节数；`cap < min_bytes` 返回 0，调用方 hard fail
- disassembler 是 header-only 纯函数，不接触 `XmFunc` / `XmIns` / `XrIsolate`
- 未识别字节优雅输出 `<unknown bytes: XX XX XX>`，不 crash
- generated 文件全部在 `build/generated/xisa/`，不进 git
- 每个 generated 文件头部带 `// AUTO-GENERATED FROM xisa/*.def — DO NOT EDIT` 警告

### 2.4 generator 自身测试

`tests/unit/xisagen/`：
- lex / parse / typecheck / encode / decode 各 ≥ 5 错误路径
- L1/L2/L3 一致性 ≥ 5 错误路径（如 mcinsn 引用 unknown helper、effect 不一致、lowering 缺 backend）
- 跑 ASAN

---

## 3. Driver 最终形态

### 3.1 driver 只做 runtime-sensitive

| driver 职责 | 是否保留 |
|---|---|
| block layout | 保留 |
| regalloc 结果消费 | 保留 |
| frame setup / teardown | 保留 |
| ABI move / call sequence | 保留 |
| stack map emission | 保留 |
| deopt snapshot metadata | 保留 |
| OSR entry / resume entry | 保留 |
| branch target resolution | 保留 |
| **raw byte encoding** | **删除** |
| **instruction bitfield 拼装** | **删除** |
| **special register encoding hack** | **删除** |
| **silent fallback** | **删除** |
| **手写 XmOp switch** | **删除**（改用 `xm_dispatch_<arch>` generated） |
| **手写 helper effect 推断** | **删除**（改用 L2 registry） |

### 3.2 driver 调用模式

```text
backend driver chooses mcinsn + operands
  ↓
generated helper validates operands and writes bytes
  ↓
driver records metadata / patch / stack map
```

driver 不允许出现：
- 手写 REX / ModR/M / SIB
- 手写 ARM64 bitfield
- 手写 RISC-V immediate split
- 对 covered instruction 直接写 hex byte

### 3.3 branch / patch / metadata 契约（独立验证层）

| 契约 | 验证方法 |
|---|---|
| branch reach | driver 选 short/long form；xisa helper 检查 immediate range |
| patch record | 记录 `(mcinsn-id, offset, width, signedness)` |
| patch 重新验证 | patch 阶段调 generator 输出的 range checker |
| stack map | PC range × emitted bytes 一致性（verifier） |
| deopt metadata | spill/register location verifier |
| OSR metadata | entry live set verifier |
| call post-call protocol | L2 `:post-call` 与 driver emit 序列一致（verifier） |

任何越界 / 不一致 → hard fail，不允许截断。

---

## 4. 验证体系

### 4.1 build-time hard fail

| 检查 | 失败后果 |
|---|---|
| L1/L2/L3 parse | build fail |
| L1/L2/L3 typecheck | build fail |
| `XmOp` × backend lowering 缺失 | build fail |
| helper effect 缺失或不一致 | build fail |
| helper `:pointer-trust` 缺失 | build fail |
| mcinsn 引用 unknown helper | build fail |
| mcinsn `:effect-override` 无 audit 测试 | build fail |
| mcinsn cross-tier 入口无 `:tier-input` | build fail |
| encoding 宽度错误 | build fail |
| golden-bytes 不匹配 | build fail |
| golden-asm 不匹配 | build fail |
| min-bytes = 0 | build fail |
| backend coverage 缺失 | build fail |

### 4.2 compile-warn 强制

- `-Wswitch-enum -Werror=switch-enum`（即便 dispatch 已 generated，driver 中其他 enum switch 也强制穷尽）
- `-Werror=missing-field-initializers`

### 4.3 runtime hard fail

`EMIT_BEGIN(ctx) / EMIT_END(ctx, mcinsn, min_bytes)` 自动包裹每个 emit helper：
- `XR_CHECK(emitted >= min_bytes, ...)`（**Release 也保留**）

`:tier-input` mcinsn 自动插 alignment fast-skip（运行期兜底）。

### 4.4 disasm round-trip

```text
operands → emit bytes → generated disasm → expected golden-asm
```

每条 mcinsn ≥ 3 case：普通 / 扩展 / 特殊寄存器 + immediate / mem / branch 边界。

### 4.5 external differential

| arch | reference |
|---|---|
| x64 | llvm-mc |
| arm64 | llvm-mc |
| riscv64 | llvm-mc / GNU binutils |

流程：random mcinsn operands → xisa emit → 双 disasm → normalize compare。

CI：
- PR：1k seed fixed
- nightly：1M seed random
- LLVM 版本 pin；normalize 层做版本兼容

### 4.6 driver invariant

每 `XmOp`：
- 被 backend coverage matrix 覆盖
- emit path ≥ 1 字节（除非显式 `:metadata-only`）
- runtime-sensitive op 必须生成对应 metadata
- branch op 必须生成 patch record 或 final target
- call op 必须生成 post-call protocol

### 4.7 runtime parity 7 子项

| 子项 | DoD |
|---|---|
| **call ABI parity** | 4 类 call × 2 backend = 8 子集 0 分歧 |
| **safepoint stack map** | slot ID / spill offset / PC range 一致；GC stress 30min 无 leak |
| **deopt reconstruction** | 同 deopt 点两 backend byte-equal；不读未初始化 spill |
| **OSR entry** | x64/arm64 重入 ≥ 4 轮；多参 / spill-pressure byte-equal |
| **suspend / resume** | Win11 Release `1115_cancel.xr` / `1109_await_any.xr` / `1127_coro_priority.xr` / `1128_yield.xr` 无 `STATUS_HEAP_CORRUPTION` |
| **effect flag completeness** | L2 audit clean；故意删 `may-throw` → 异常回归 fail；故意删 `may-gc` → GC stress fail |
| **cross-tier pointer hygiene** | 所有 conservative 入口经 `:tier-input` mcinsn；故意删 fast-skip → 30min fuzz 必复现；`obj->type` 越界检查覆盖 sweep/mark/decref |

### 4.8 fuzz 4 层

| 层 | 内容 | nightly seed |
|---|---|---|
| **L1 mcinsn fuzz** | random operands → emit → disasm → external diff | 1M |
| **L2 Xm program fuzz** | random Xm program → JIT → interpreter compare | 1M |
| **L3 driver invariant fuzz** | random op sequence → coverage / bytes / metadata verifier | 1M |
| **L4 corruption-chain fuzz** | JIT force + ASAN + GC stress + pointer validation 30min | matrix |

shrinker：emit bug 缩到单条 mcinsn / 30s；driver bug 缩到 ≤ 5 节点 Xm；corruption 缩到最短 cross-tier chain。

### 4.9 JIT fallback 边界

fallback 必须按层定义，避免“测试通过但实际没跑 JIT”。

| 边界 | 是否允许 fallback | 规则 |
|---|---|---|
| Xi → Xm eligibility | 允许拒绝 JIT | 不支持的语言语义可以返回“不编译”，交给 VM 执行 |
| Xm verify 之前 | 允许拒绝 JIT | 类型 / CFG / effect contract 不满足时拒绝进入 backend |
| Xm → machine code | 不允许 | 一旦进入 backend，unhandled op / unknown helper / emit 0 byte / patch 越界都是 compiler bug，必须 hard fail |
| helper metadata | 不允许 | raw pointer / unknown helper 不得被当成 pure helper 或普通 C call |
| backend coverage | 不允许伪装 | unsupported op 由 eligibility 拒绝，不写 `:unimplemented` lowering |

---

## 5. 实施阶段（S0 - S10）

每阶段独立 git tag：`xisa-s0` ... `xisa-s10`。任何阶段发现 bug，按 root cause 修复并补回归；不让 generator 适配错误。

### S0 — 契约冻结 + old codegen quarantine + 当前 driver hard fail

**目标**：把最终设计变成可执行边界；先禁止扩大旧手写路线。

**任务**：
- [ ] 冻结本方案为 codegen 唯一执行入口（其他 codegen task 全部归档）
- [ ] 旧 codegen API 进入 read-only quarantine：只能删除 / 替换，不能新增公开 emit API
- [ ] 清点：所有手写 raw byte / bitfield / special register encoding
- [ ] 清点：所有 `XmOp` switch 和 backend handler
- [ ] 清点：所有 runtime helper call site
- [ ] 清点：所有 cross-tier pointer 入口
- [ ] 当前 driver 补 hard fail：unhandled op / 0-byte emit / unknown helper / silent default
- [ ] CI 增加 denylist：新增 `a64_*` / `x64_*` 手写 encoding helper、raw byte、raw helper pointer 语义判断直接 fail

**退出**：
- 三份 inventory 完整（raw encoding / helper / pointer boundary）
- 当前 regression 不退行
- 新增 codegen 不允许扩大手写 encoding / 手写 `XmOp` switch / raw helper pointer 语义判断

**回滚锚点**：`xisa-s0`

---

### S1 — `xisa/xm/ops.def` 契约表 + `xisa/xm/helpers.def` registry

**目标**：消灭 IR opcode / helper effect 真相源漂移。

**任务**：
- [ ] 创建 `xisa/xm/ops.def`，列出全部 `XmOp` + 元数据
- [ ] 创建 `xisa/xm/helpers.def`，列出全部 C helper
- [ ] generator 输出 `xm_ops.h` / `xm_op_table.c` / `xm_op_verify.c` / `xm_helpers.h` / `xm_helper_table.c` / `xm_helper_verify.c`
- [ ] 删除手写 `XmOp` enum、name/arity/effect 表
- [ ] 把 raw helper pointer 替换为 helper id（codegen 中所有 `xrt_*` 调用都改为 `XM_HELPER_*` id）
- [ ] 未知 helper 进入 codegen → typecheck fail
- [ ] effect 缺失 → typecheck fail

**退出**：
- 故意从 `xisa/xm/ops.def` 删一个 op → build fail
- 故意 codegen 用 unknown helper → build fail
- 故意 helper 缺 `:effects` → build fail
- ctest + regression 通过

**回滚锚点**：`xisa-s1`

---

### S2 — xisagen 核心 + S-expression schema + fixture

**目标**：generator 可用；三 arch 第一批 mcinsn fixture 落地。

**任务**：
- [ ] 实现 6 模块：lex / parse / ast / typecheck / encode / emit + 配套 golden / diag / host
- [ ] schema 三层强化（§1.4 新增字段）落定
- [ ] 三 arch 每 ≥ 20 mcinsn fixture，含三类寄存器（普通/扩展/特殊）+ 一条 call + 一条 conservative scan
- [ ] 每条 fixture ≥ 3 golden-bytes + 3 golden-asm
- [ ] golden 必须用两种独立 reference 交叉验证（手写 emit / llvm-mc / GNU as / objdump 任二）
- [ ] generator 自身单测（§2.4）

**退出**：
- 三平台 clean build 不依赖 `xray_core`
- 故意改错 fixture → build fail；诊断含 `.isa` 路径 + 行 + 列 + mcinsn + operands + expected/actual
- generator 单测 100% pass
- 不接入 JIT；仅独立 test target

**回滚锚点**：`xisa-s2`

---

### S3 — XmOp dispatch generation + 字节自检契约

**目标**：消除手写 XmOp switch + 兜死 `8b06997` 类 bug。

**任务**：
- [ ] generator 把 L1 `:lowering` 编为 `xm_dispatch_<arch>.c`
- [ ] driver 中所有 `XmOp` switch 替换为 `xm_dispatch_<arch>(ctx, ins)`
- [ ] generator 自动包裹每个 emit helper 为 `EMIT_BEGIN(ctx) / EMIT_END(ctx, mcinsn, min_bytes)`，使用 `XR_CHECK`（Release 保留）
- [ ] driver 中其他 enum switch 强制 `default: XR_UNREACHABLE`
- [ ] 编译选项：`-Wswitch-enum -Werror=switch-enum`（CI 强制）

**退出**：
- 全文 `grep 'switch.*XmOp\|switch.*ins->op' src/jit/` 仅命中 generated 文件
- 故意删一个 `:lowering` → build fail
- 故意让一个 mcinsn 漏 emit → JIT 编译时 `XR_CHECK` fail（Release 也 fail）
- Win11 Release 重跑 commit `8b06997` 复现路径，确认仍正常
- ctest + regression 通过

**回滚锚点**：`xisa-s3`

---

### S4 — generated disassembler + external differential

**目标**：disassembler 不再是第二真相源；llvm-mc 作为第三方 reference。

**任务**：
- [ ] xisa schema 加 `:golden-asm` / `:decode-priority` / `:external-reference`
- [ ] generator 实现 `xisa_decode.c` 模块，反向生成 `xisa_disasm_<arch>.h`
- [ ] 接入 round-trip：emit → disasm → expected golden-asm
- [ ] 接入 `tests/unit/jit/test_assembler_diff.c`：xray emit → 双 disasm → normalize → diff
- [ ] CI：macOS + Linux runner 跑（Windows 不要求 llvm-mc）
- [ ] LLVM 版本 pin；normalize 层做版本兼容
- [ ] `scripts/codegen_diff.sh` 本地复现

**退出**：
- generated emit + generated disasm round-trip 全过
- macOS / Linux PR 1k seed 0 分歧
- 新增 mcinsn 缺 `:golden-asm` → build fail
- 未识别字节优雅输出 `<unknown>`，不 crash

**回滚锚点**：`xisa-s4`

---

### S5 — x64 全面替换 + 删除旧 helper

**目标**：x64 covered instruction 全部走 generated helper；删除对应手写。

**替换流程（每条 mcinsn，单 PR 闭环）**：

```text
1. 把 mcinsn X 加入 xisa/arch/x64/isa.isa + L1 :lowering
2. test-only oracle 对比旧手写 encoder 与 generated helper
3. 差异为旧 bug → 修 root cause + 补 golden；差异为 schema bug → 修 schema
4. 删除 X 对应的手写 emit + 所有 call site
5. driver 只调用 generated helper
6. 合并前主线不能保留 dual-emit runtime path
```

**任务**：
- [ ] 完整覆盖 `xm_x64.h` 暴露的所有 emit 入口
- [ ] 每个 covered mcinsn 替换 PR 合并前必须删除对应手写
- [ ] `grep` `*buf++ = 0x\|*buf++ = .*(uint8` 在 `xm_x64.c` 命中数 = 0
- [ ] test-only oracle 不进入 release/runtime path

**退出**：
- x64 covered instruction 不存在手写 byte / ModR/M / SIB 拼装
- ctest + regression + Win11 不退行
- x64 stress 24h 无新 bug
- x64 external diff nightly 启动
- 替换期发现的手写 bug 已修 root cause 并补 golden

**回滚锚点**：`xisa-s5`

---

### S6 — arm64 全面替换 + 删除旧 helper

**目标**：同 S5，覆盖 arm64。

**任务**：
- [ ] 完整覆盖 `xm_arm64.h` 暴露的所有 emit 入口
- [ ] bitfield / logical imm / shifted imm 单独 golden 集
- [ ] 替换流程同 S5：单 PR 闭环，合并前删除对应手写路径
- [ ] `xm_arm64.c` 中"手写 bitfield 拼装"行数清零

**退出**：
- 同 S5 标准
- arm64 JIT stress 24h 无新 bug
- arm64 external diff nightly 启动
- x64 / arm64 scalar differential 0 分歧

**回滚锚点**：`xisa-s6`

---

### S7 — branch / patch / metadata 契约统一

**目标**：把 branch reach、patch width、metadata emission 变成机器可检查契约。

**任务**：
- [ ] patch record 记录 `(mcinsn-id, offset, width, signedness)`
- [ ] patch 阶段复用 generator 输出的 range checker
- [ ] stack map verifier：检查 PC range 与 emitted bytes 对齐
- [ ] deopt metadata verifier：检查 spill / register location
- [ ] OSR metadata verifier：检查 entry live set
- [ ] call post-call protocol verifier：driver emit 序列与 L2 `:post-call` 一致

**退出**：
- 故意 patch 越界 → fail-fast
- 故意 stack map PC range 错误 → fail-fast
- 故意 deopt spill offset 错误 → fail-fast
- 故意 OSR live set 不一致 → fail-fast
- 故意 call 缺 post-call check → fail-fast
- ctest + regression 通过

**回滚锚点**：`xisa-s7`

---

### S8 — runtime parity 7 子项

**目标**：补齐 machine encoding 之外的 JIT 高危路径。7 子项独立 DoD（详见 §4.7）。

**任务**：
- [ ] **S8.1 call ABI parity**
- [ ] **S8.2 safepoint stack map**
- [ ] **S8.3 deopt reconstruction**
- [ ] **S8.4 OSR entry**
- [ ] **S8.5 suspend / resume**
- [ ] **S8.6 effect flag completeness audit**（因 L2 自动 build-time 检查，本子项主要 audit 已存量 + 补回归）
- [ ] **S8.7 cross-tier pointer hygiene audit**（因 L3 `:tier-input` 自动 build-time 检查，本子项主要 audit 已存量入口 + 写 fuzz）

**退出**：
- 7 子项 DoD 全过（§4.7）
- ASAN + `--jit-force` + GC stress 30min 无 corruption
- commit `4da96e3` 回归测试在 macOS Debug 通过
- commit `d94c4a8` 回归（`1206_gc_enhanced.xr` × `--jit-force` × ASAN × 50 次）0 失败
- Win11 Release 协程 4 用例无 `STATUS_HEAP_CORRUPTION`

**回滚锚点**：`xisa-s8`

---

### S9 — fuzz 4 层 + linemap + nightly CI

**目标**：rare codegen bug 移到 nightly 自动发现。

**任务**：
- [ ] L1 mcinsn fuzz
- [ ] L2 Xm program fuzz
- [ ] L3 driver invariant fuzz
- [ ] L4 corruption-chain fuzz（`--jit-force` × ASAN × GC stress 30min）
- [ ] shrinker（emit bug 缩到单 mcinsn / driver bug 缩到 ≤ 5 节点 Xm / corruption 缩到最短 chain）
- [ ] generated mcinsn linemap：bytes → mcinsn → `.isa` 行
- [ ] driver synthetic region linemap：prologue / stub / patch island 标 synthetic
- [ ] CI 指标 dashboard：seed 数 / fail 数 / 中位缩小 IR / chain depth

**退出**：
- 4 层 nightly 1M seed 4 周 0 分歧
- ASAN + JIT force + GC stress nightly 稳定
- 任一 failure 可用 seed 单命令复现
- 故意 inject emit bug → shrinker 30s 缩到单 mcinsn
- 故意 inject 错位指针 → 30min corruption fuzz 触发

**回滚锚点**：`xisa-s9`

---

### S10 — RISC-V subset bring-up

**目标**：第三后端从零使用 xisa，不复制旧手写路线。

**范围（v1）**：integer arithmetic / load-store / branch-jump / simple call-return / basic prologue-epilogue。

**不要求**：OSR / deopt / safepoint full protocol / coroutine / compressed extension。

**任务**：
- [ ] `xisa/arch/riscv64/isa.isa` 覆盖 RV64 base scalar subset
- [ ] L1 `xisa/xm/ops.def` 只为 riscv64 声明已支持 subset 的 backend coverage
- [ ] unsupported `XmOp` 由 riscv64 JIT eligibility 拒绝，不写 `:driver-lowered :unimplemented`
- [ ] 写 riscv64 driver skeleton（仅 v1 范围）
- [ ] QEMU 跑 scalar JIT unit
- [ ] scalar differential 加入 riscv64
- [ ] **不允许新增 riscv64 手写 raw encoding helper**

**退出**：
- QEMU RISC-V 上 JIT scalar unit 全过
- x64 / arm64 / riscv64 / interpreter 三方 scalar diff 0 分歧
- riscv64 generated disasm + external diff 通过
- riscv64 unsupported runtime-sensitive op 不进入 codegen；eligibility 拒绝路径有测试覆盖

**回滚锚点**：`xisa-s10`

---

## 6. 关键 milestone

| Milestone | 何时 | 评估 | 决策 |
|---|---|---|---|
| M0 inventory 完整 | S0 | 三份 inventory 是否覆盖全部入口？ | 不全 → 继续清点 |
| M1 真相源生效 | S1 | 删 op / unknown helper / 缺 effect 是否 build fail？ | 否 → 修 generator |
| M2 schema 可表达 | S2 | 20 mcinsn 三 arch 是否能表达？ | 不能 → 修 schema |
| M3 commit `8b06997` 根除 | S3 | 字节自检覆盖所有路径？dispatch 是否 generated？ | 不全 → 补 |
| M4 第三方一致 | S4 | llvm-mc 1k seed 0 分歧？ | 有 → 修 root cause |
| M5 x64 手写归零 | S5 | 手写 byte 拼装行数 = 0？test-only oracle 是否未进入 runtime？ | 否 → 继续替换 |
| M6 arm64 手写归零 | S6 | 同上 | 同上 |
| M7 patch/metadata verifier | S7 | 故意越界/错位是否全 fail-fast？ | 否 → 补 verifier |
| M8 runtime parity | S8 | 7 子项 DoD 全过？commit `4da96e3` / `d94c4a8` 不复现？ | 缺一不可 |
| M9 fuzz 稳定 | S9 | 4 层 nightly 4 周 0 分歧？shrinker 可用？ | 有分歧 → triage |
| M10 RISC-V subset | S10 | QEMU JIT scalar 全过？三后端 diff 0 分歧？ | 失败 → 复盘 |

---

## 7. 反向不变量（12 条 grep + CI 强制）

| # | 不变量 | 检查方法 | 来源 |
|---|---|---|---|
| 1 | 手写 `XmOp` enum / name / arity / effect 表 = 0 | `grep` `enum XmOp` 仅在 generated 文件命中 | 074 |
| 2 | 手写 helper effect / pointer-trust 推断 = 0 | `grep` raw fn pointer effect 判断 = 0 | 074 |
| 3 | driver 中手写 byte 拼装 = 0 | `grep` `*buf\+\+ = 0x\|*buf\+\+ = .*(uint8` 在 `src/jit/` 命中 = 0 | 075 |
| 4 | 手写 emitted-subset disassembler = 0 | `xm_arm64_disasm.h` 等手写文件全部删除，仅 generated 存在 | 074 |
| 5 | silent fallback (`default: break;`) = 0 | `grep` `default:\s*break;\?\s*$` 在 codegen = 0 | 075 |
| 6 | runtime dual-emit = 0 | `XR_ENABLE_XISA_DUAL_EMIT` 不得进入 release/runtime path；只允许 test-only oracle | 074 + 075 |
| 7 | legacy wrapper / 兼容层 = 0 | `grep` `old_emit\|legacy_helper\|XR_COMPAT` = 0 | 074 |
| 8 | unknown helper 进入 codegen = 0 | L2 typecheck = 100% pass | 074 |
| 9 | unknown opcode 进入 backend = 0 | L1 typecheck = 100% pass | 074 |
| 10 | cross-tier pointer 无 `:tier-input` 入口 = 0 | L3 typecheck = 100% pass | 075 |
| 11 | generated 文件进 git = 0 | `.gitignore` 覆盖 `build/generated/xisa/`；CI 检查 | 074 |
| 12 | 跳过 failing regression = 0 | known_failures 必须 link issue + 30 天过期 alert | 074 + 075 |

每条不变量都对应 CI script，**违反即阻塞 merge**。

---

## 8. 完成判定

### 8.1 v1 完成（S0 - S4）

| 指标 | 目标 |
|---|---|
| L1 `xisa/xm/ops.def` 真相源 | 生效；删 op / 缺 lowering build fail |
| L2 `xisa/xm/helpers.def` 真相源 | 生效；unknown helper / 缺 effect build fail |
| xisagen clean build | 三平台 pass |
| 三 arch 每 ≥ 20 mcinsn fixture | golden 全过；double-reference 交叉验证 |
| XmOp dispatch generated | driver 无手写 switch |
| 字节自检契约 | Release 保留；commit `8b06997` 路径 0 复现 |
| generated disassembler | 100% mcinsn round-trip |
| llvm-mc differential | PR 1k seed 0 分歧 |
| ctest / regression / Win11 | 三平台不退行 |

### 8.1.1 076-A Foundation Complete（优先启动闭环）

076-A 是本方案的最小可交付闭环，用于尽快冻结旧路线并让新真相源接管核心路径。

| 指标 | 目标 |
|---|---|
| old codegen quarantine | 旧 emit API 只能删除 / 替换；禁止新增手写 encoding helper |
| `.def` / `.isa` 源码落位 | `xisa/xm/ops.def`、`xisa/xm/helpers.def`、`xisa/arch/*/isa.isa` 进 git |
| generated 输出隔离 | 只输出到 `build/generated/xisa/`，不进 git |
| L1/L2 metadata generated | `XmOp` / helper enum、arity、effect、trust、ABI 由 generator 输出 |
| 核心 mcinsn fixture | x64 / arm64 scalar core ≥ 20 条；每条 ≥ 3 golden |
| dispatch + byte self-check | generated dispatch 生效；Release 保留 emit byte check |
| external diff 基础版 | macOS / Linux PR 1k seed 0 分歧 |

### 8.2 完整完成（S0 - S10）

| 指标 | 目标 |
|---|---|
| x64 / arm64 手写 byte 拼装 | 0 行 |
| runtime dual-emit | 不存在；test-only oracle 不进入 release/runtime path |
| branch/patch/metadata verifier | 全部 hard fail |
| runtime parity 7 子项 | 全 DoD 满足 |
| commit `4da96e3` 类 bug | build-time 拒（L2 typecheck） |
| commit `d94c4a8` 类 bug | build-time 拒（L3 `:tier-input` typecheck）+ 运行期 fast-skip 双层 |
| fuzz 4 层 nightly | 1M seed 4 周 0 分歧 |
| corruption-chain fuzz × ASAN × `--jit-force` × GC stress | nightly 30min 0 复现 |
| Win11 Release 协程 4 用例 | 0 `STATUS_HEAP_CORRUPTION` |
| RISC-V subset | QEMU JIT scalar 全过；三后端 + interpreter diff 0 分歧 |
| generated mcinsn linemap | 100% generated 区域可反查 `.isa` 行 |

### 8.3 不强求

- SIMD / VEX / EVEX：非目标
- instruction scheduler / pattern fusion：非目标
- LLVM TableGen 级别通用框架：非目标
- AOT C backend：与 xisa 正交（走 `src/aot/xi_cgen.c`）
- VM bytecode emitter：与 xisa 正交（走 `src/ir/xi_emit.c`）
- Win64 `.pdata` / `.xdata` unwind：与 xisa 正交，独立 follow-up
- 完整 runtime-sensitive lowering 自动化：driver 只编排 runtime protocol；底层 encoding 必须 generated

---

## 9. 删除清单

最终必须删除或替换：

| 类别 | 处理 |
|---|---|
| 手写 `XmOp` enum / name / arity / effect table | L1 generated 替换 |
| raw helper pointer 语义判断 | L2 helper id + registry 替换 |
| x64 raw byte emit helper（covered forms） | L3 generated helper 替换，原文件删除 |
| arm64 raw bitfield emit helper（covered forms） | 同上 |
| 手写 x64 emitted-subset disassembler | L3 generated disassembler 替换 |
| 手写 arm64 emitted-subset disassembler | 同上 |
| 手写 XmOp switch（driver 中） | generated `xm_dispatch_<arch>` 替换 |
| silent fallback (`default: break`) | hard fail (`XR_UNREACHABLE`) |
| runtime dual-emit | 禁止存在；test-only oracle 不进入 release/runtime path |
| legacy wrapper / 兼容层 | 禁止存在 |
| `XR_ENABLE_XISA_DUAL_EMIT` flag | 不允许作为 runtime/release 开关；如需对比只能放 test-only target |

**允许保留**：

| 类别 | 原因 |
|---|---|
| backend driver | runtime-sensitive protocol 必须手写 |
| code allocator (`xm_code_alloc.c`) | 与 xisa 无关 |
| regalloc | 与 xisa 无关 |
| frame layout logic | driver 职责 |
| deopt / OSR / safepoint metadata writer | driver 职责，但必须有 verifier |
| patch helper（branch reach / target resolution） | driver 职责 |

---

## 10. 风险登记册

| ID | 风险 | 概率 | 影响 | 应对 |
|---|---|---|---|---|
| R1 | generator 自身 bug 把 schema 错误固化为 generated 错误 | 中 | 高 | test-only oracle 保护；generator ≥ 20 错误路径单测；任何 oracle 分歧先修 root cause |
| R2 | L1/L2/L3 schema 表达力不够 | 中 | 中 | S0/S1/S2 fixture 用困难 op 提前暴露；schema 改动必须同步 fixture |
| R3 | DSL 表达力不够覆盖 runtime-sensitive op | 高 | 中 | `:driver-lowered` 显式标记；driver 仍写 lowering 序列；底层 mcinsn 仍 generator 输出 |
| R4 | Windows MSVC 上 generator / generated header 行为飘移 | 中 | 高 | S2 退出强制三平台 build；CMake 不用 unix-only flag |
| R5 | `llvm-mc` 版本飘移导致 differential 假阳性 | 低 | 中 | pin LLVM 版本；normalize 层版本兼容；CI 报版本号 |
| R6 | S5/S6 oracle 暴露大量历史手写 bug | 高 | 中 | 先修 root cause + 加 golden，再删手写；不让 generator 适配错误 |
| R7 | test-only oracle 误入 runtime path | 中 | 高 | CI grep 阻断 release/runtime 中的 `XR_ENABLE_XISA_DUAL_EMIT` / dual-emit path |
| R8 | S8 runtime parity 暴露非平凡 bug | 高 | 高 | 7 子项独立 DoD；按 root cause 修复；不允许 known_failures 长期屏蔽 |
| R9 | L2 helper registry audit 暴露大量历史漏标 | 高 | 中 | S1 默认 conservative（unknown bridge → `may-throw + may-gc + may-suspend`）；去 flag 必须配 audit 测试 |
| R10 | L3 `:tier-input` audit 工作面广 | 高 | 高 | S8.7 拆按"指针来源"分批；每批独立 DoD |
| R11 | fuzz × ASAN × `--jit-force` × GC stress 暴露 bug 超过修复带宽 | 中 | 高 | shrinker 输出按 root cause 分组；写 `docs/known_bugs.md`；按 bug 修复纪律 |
| R12 | RISC-V 暴露 schema 不通用 | 中 | 中 | S10 失败仅 RISC-V 暂停；schema 改动同步 fixture |
| R13 | 团队仍倾向静默 fallback / 手写 byte 拼装 | 中 | 高 | 反向不变量 12 条 CI 强制；PR template / code review checklist 固化 |
| R14 | DSL 错误诊断质量差 | 中 | 中 | S2 退出含行 + 列 + mcinsn + operands + expected/actual 四要素；后续加 suggested fix |

每阶段独立回滚锚点 git tag；发现阶段不可达 → 回滚到上一 tag + root cause 复盘。

---

## 11. 资源与时间估算

| 阶段 | 周 | 累计 | 备注 |
|---|---|---|---|
| S0 | 1 | 1 | 契约冻结 + inventory + old codegen quarantine + hard fail |
| S1 | 2 | 3 | XmOp contract + helper registry |
| S2 | 2-3 | 5-6 | generator 核心 + schema + fixture |
| S3 | 1-2 | 6-8 | dispatch generation + 字节自检 |
| S4 | 2 | 8-10 | disassembler + external diff |
| S5 | 2-3 | 10-13 | x64 替换 + 删除手写 |
| S6 | 2-3 | 12-16 | arm64 替换 + 删除手写 |
| S7 | 1-2 | 13-18 | branch/patch/metadata verifier |
| S8 | 4-6 | 17-24 | runtime parity 7 子项 |
| S9 | 1-2 | 18-26 | fuzz 4 层 + linemap + nightly CI |
| S10 | 2-3 | 20-29 | RISC-V subset bring-up |

**总计 20-29 周**。

> **S0-S4 6-10 周后已拿到完整真相源 + build-time 健壮性 + 第三方 reference 一致性**。
> S5-S6 是替换 + 删除主体（4-6 周）。
> S7-S9 是 runtime + fuzz 加固（6-10 周）。
> S10 可与其他 task 并行推进。

---

## 12. 与其他 task 的关系

| 关联 task | 关系 |
|---|---|
| `008-jit-multi-backend.md` | 本方案是其 machine encoding 层的最终载体 |
| `009-jit-x64-parity.md` | parity 进入 S8.1-S8.5；effect/pointer hygiene 进入 S8.6/S8.7 |
| `068-compiler-pipeline-optimization.md` | Xi → Xm 不在本方案范围；本方案只覆盖 Xm → machine code |
| `070-regression-bug-triage.md` | codegen 类 bug 最终转为 golden / differential / parity / fuzz case |
| `archive/073-codegen-hardening.md` | 已归档；纪律全部继承到本方案 |
| `archive/074-codegen-final-plan.md` | 已归档；XmOp contract / helper registry / 删除清单继承到本方案 |
| `archive/075-codegen-final-spec.md` | 已归档；git tag / grep 反向不变量 / 时间估算继承到本方案；原 migration 流程在本文收紧为单 PR 替换 + 删除 |
| `archive/073-xisa-implementation.md` | 已归档；C generator / S-expression / encoding-first 继承到本方案 |
| `archive/xisa_design.md` | 已归档；DSL 语法 / 三 arch encoding 元件继承到本方案 |
| `aot/xi_cgen.c` (AOT) | 与本方案正交，不互引 |
| `ir/xi_emit.c` (VM bytecode) | 与本方案正交，不互引 |

---

## 13. 推荐启动顺序

```text
S0 hard fail + inventory
  ↓
S1 XmOp contract + helper registry         ← 立即消灭元表漂移
  ↓
S2 generator + schema + fixture            ← 建立新真相源
  ↓
S3 dispatch + 字节自检                     ← 兜死 8b06997 类
  ↓
S4 disassembler + external diff            ← 建立可观察性
  ↓
S5 x64 替换                                ← 真正降低技术债（从此起删除旧手写）
```

**理由**：
- S0 立即堵 silent fallback（不让旧路线扩大）
- S1 先消灭 effect / opcode 元表漂移（最高 ROI）
- S2 建立 generator 真相源
- S3 兜死最严重的 bug 模式（`8b06997`）
- S4 建立可观察性 + 第三方 reference
- S5 开始真正的旧手写删除（前面四步是地基）

S0-S4 一旦完成（6-10 周），项目已经拿到本方案 80% 的核心收益。剩余 S5-S10 是大规模替换删除 + runtime 加固，可按节奏推进。

---

## 14. 执行纪律

| 纪律 | 强制方法 |
|---|---|
| 每阶段发现真实 bug，立即修 root cause 并补测试 | bug 修复铁律（user_rules） |
| 不允许用 generator 适配已知错误编码 | test-only oracle 分歧必须修 root cause |
| 不允许为保留旧行为加兼容层 | 反向不变量 #7 grep 检查 |
| 不允许保留未使用旧 helper | S5/S6 退出条件含 `grep` 检查 |
| 不允许跳过 failing regression | known_failures 必须 link issue + 30 天过期 alert |
| 不允许 unknown helper 当 pure helper | L2 typecheck = 100% pass |
| 不允许 conservative pointer 未校验解引用 | L3 `:tier-input` typecheck = 100% pass |
| 每次代码修改后 ctest | user_rules 测试纪律 |
| 涉及 JIT runtime 必须跑 regression | user_rules 测试纪律 |
| 涉及 pointer / GC 必须跑 ASAN + GC stress | S8.7 / S9 L4 |
| test-only oracle 误入 runtime path 必须阻断 | CI grep 自动 fail |
| generated 文件不进 git | `.gitignore` 覆盖 + CI 检查 |

---

## 15. 修订历史

| 日期 | 改动 | 作者 |
|---|---|---|
| 2026-05-12 | 初稿；合并 073 / 074 / 075 / archive xisa 五份方案；确立三层真相源（L1 XmOp contract + L2 helper registry + L3 mcinsn schema）+ 双层 effect/trust 防御 + 单 PR 替换删除流程 + 阶段 git tag + 12 条反向不变量 + S0-S10 共 11 阶段；073/074/075 同步归档到 docs/archive/ | Cascade + xingleixu |
