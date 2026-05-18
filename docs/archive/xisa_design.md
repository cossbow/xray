# xisa: xray ISA description language — 设计 spec

> **xisa** 是 xray JIT 后端使用的机器指令描述语言。
>
> **v1 定位**：xisa 首先解决 **machine encoding + build-time golden**，用声明式 `.isa` 文件生成类型化 emit helper，消除手写字节编码、漏 emit、寄存器特例遗漏这类 codegen bug。
>
> **非目标**：xisa v1 不替代 `Xm` 数据结构、regalloc、codegen driver、patch、OSR、deopt、stack map、AOT C backend，也不要求一开始生成完整 lowering / interpreter / linemap。

---

## 1. 背景与动机

### 1.1 当前 codegen 的真实规模

| 后端 | 行数 |
|---|---|
| x64 (`xm_codegen_x64*.c` + `xm_x64.c` + `xm_target_x64.c`) | 5,894 行 C |
| arm64 (`xm_codegen*.c` 非 x64 + `xm_arm64.c` + `xm_target_arm64.c`) | 5,814 行 C |
| **每后端平均** | **~5.8k 行** |

RISC-V 后端若沿用手写模式，会新增另一份几千行 machine emit / call ABI / patch / runtime protocol 代码。三后端长期并存时，最危险的不是单个后端代码量，而是**没有共同 schema**：同一类指令编码、patch 规则、fallback 路径、最小 emit 字节数只能靠人工同步。

### 1.2 直接动机

commit `8b06997` 暴露了一个典型后端漂移问题：x64 RT-opcode fallback 漏 emit 任何字节，而 arm64 同类路径 emit 了一个守护 NOP。结果是 Win64 上出现 `STATUS_HEAP_CORRUPTION`。

根因不是单次手误，而是当前架构把“每个 codegen 分支必须明确 emit 或 fail-fast”的契约写在注释和 review 习惯里。两个后端各自维护手写 emit，缺乏一个 build-time 可检查的共同描述。

同类风险还包括：

- **编码字节错误**：REX、ModR/M、SIB、arm64 bitfield、RISC-V immediate split。
- **特殊寄存器遗漏**：x64 `rsp/rbp/r12/r13`，arm64 `sp/xzr`，RISC-V `x0`。
- **emit 长度不一致**：fallback、NOP、patch placeholder 长度漂移。
- **reloc/patch 错误**：PC-relative offset、branch reach、call target patch。
- **后端行为发散**：同一 `Xm` op 在不同架构上的 runtime fallback 不一致。

### 1.3 设计窗口

第三个 native 后端是引入 xisa 的最佳窗口：

- **现有 oracle**：两个现有后端提供 byte-equal / behavior-equal 对照。
- **新后端验证**：RISC-V 可以验证 declarative backend 是否真正降低新后端成本。
- **独立价值**：先做 encoding/golden 可以独立交付价值，不需要一次性替换整套 codegen。

---

## 2. 核心决策

### 2.1 generator 用 C 写

**决策**：xisa generator 使用 C 实现，作为 CMake host tool 编译运行。

原因：

- **bootstrap 最稳**：构建 JIT emit helper 不能依赖已经构建完成的 xray 编译器或 JIT。
- **cross compile 清晰**：generator 是 host tool，生成 target C/header；不会混淆 host/target xray binary。
- **CI 和发行简单**：只依赖项目已有 C toolchain，不引入 Python/Lua/Rust，也不引入 xray 自举环。
- **调试成本低**：generator bug 可用普通 C 调试器、ASAN、ctest 覆盖。

约束：

- generator 不依赖 `xray_core`，避免构建环。
- generator 可复用必要的 `src/base` 小模块或独立 host support，但仍遵守项目 C 规则：分配走 `xr_malloc` / `xr_free` 风格封装，不直接裸用 `malloc/free`。
- 未来如果需要 dogfood，可在 xray 工具链稳定后把 generator 迁移为 xray 程序；这不是 xisa 成功的前提。

### 2.2 encoding-first

**决策**：xisa v1 只承诺生成 machine instruction emit helper 与 golden tests。

不在 v1 承诺：

- **完整 lowering**：不一开始生成 `XmOp -> mcinsn` switch。
- **全 op interpreter**：不把 runtime-sensitive op 都塞进自动解释器。
- **全机器字节 linemap**：先覆盖 generated mcinsn，driver synthetic region 后续再接。
- **Win64 unwind**：不在 v1 自动生成 `.pdata` / `.xdata`。
- **branch relaxation**：short/long branch 与 trampoline 单独设计。
- **整套 driver 删除**：driver 逻辑保留，逐步替换其中的 machine encoding。

这些能力可以在 v1 稳定后逐步加入，但不能阻塞第一批真实替换。

### 2.3 driver-preserving

**决策**：xisa 替换手写机器编码，不替换 codegen driver。

xisa 不接管：

- **IR 层**：`XmFunc` / `XmIns` / SSA 数据结构。
- **优化层**：CFG、liveness、DCE、type propagation、regalloc。
- **frame 层**：prologue / epilogue、frame layout、spill/reload。
- **runtime 层**：code cache、W^X、OSR、deopt、stack map、safepoint、suspend/resume。
- **patch 层**：patch record、branch layout、call relocation。

xisa 接管：

- **编码规则**：架构机器指令的 byte / bitfield encoding。
- **约束检查**：operand 类型、寄存器类、immediate range、特殊寄存器规则。
- **golden tests**：每条 machine instruction 的 build-time byte 验证。
- **emit helper**：generated inline/static C helper。
- **属性表**：可选的 mcinsn 属性表。

### 2.4 subset-first

**决策**：所有验证先从纯标量子集开始，再扩展到 runtime-sensitive 路径。

迁移顺序：

1. **纯 emit**：`mov/add/sub/cmp/load/store` 等无 patch、无 runtime 状态的指令。
2. **branch/patch**：明确 label、reloc、short/long branch 的 driver 契约。
3. **call/runtime**：call ABI、helper call、safepoint、deopt、OSR、suspend/resume。
4. **cross-arch differential**：先 pure subset，再 memory subset，再 runtime subset。

---

## 3. 整体架构

```text
xisa/*.isa
    ↓
tools/xisagen/          C host tool
    ↓
build/generated/xisa/
    ├── xisa_emit_x64.h
    ├── xisa_emit_arm64.h
    ├── xisa_emit_riscv64.h
    ├── xisa_mcattr_x64.h
    ├── xisa_mcattr_arm64.h
    ├── xisa_mcattr_riscv64.h
    └── xisa_golden_tests.c
```

`xisa_mcattr_<arch>.h` 见 §4.7：每条 mcinsn 的 flags / min-bytes / max-bytes 编译期常量表，driver 用它做 reach / patch slot 检查。

生成物由 CMake 在 build 目录产生，不进入 git。因为 generator 是 C host tool，clean build 不需要预先存在 xray binary。

### 3.1 目录建议

```text
xisa/
├── backends/
│   ├── x64/isa.def
│   ├── arm64/isa.def
│   └── riscv64/isa.def
└── subsets/
    └── scalar.isa

tools/xisagen/
├── xisa_main.c
├── xisa_lex.c
├── xisa_parse.c
├── xisa_ast.c
├── xisa_typecheck.c
├── xisa_encode.c
├── xisa_emit_c.c
├── xisa_golden.c
└── xisa_host.c
```

`subsets/scalar.isa` 用于 early differential / fixtures，不作为 v1 的完整 lowering 真相源。

### 3.2 CMake bootstrap

构建顺序：

```text
host cc
  ↓
xisagen host executable
  ↓
generated xisa headers / tests
  ↓
xray_core / xray executable
```

关键约束：

- **host tool**：`xisagen` 在 build machine 上运行。
- **无构建环**：`xisagen` 不链接 `xray_core`。
- **target-independent 输出**：cross compile 时仍由 host generator 输出 C/header。
- **生成物隔离**：generated headers 只被 JIT codegen include。

---

## 4. DSL 核心概念

> 本章规定的语法和元件清单是 S0 编写 fixture 的契约。所有词法约定：
>
> - S-expression 语法，`;` 起行内注释。
> - 字面量：十进制整数 `0`-`-1`、十六进制 `0xNN`、二进制 `0b...`、双引号字符串 `"..."`。
> - 标识符 `[A-Za-z_][A-Za-z0-9_.-]*`；点号用于命名空间（如 `x64.add.rr`）。
> - operand 以 `$` 前缀引用。
> - keyword 以 `:` 前缀（如 `:operands`），它的位置只能出现在顶层 form 内。
> - 所有 keyword 是无序的，generator 解析时按 keyword 名称索引。

### 4.1 顶层 grammar

```bnf
file        ::= form*
form        ::= mcinsn-form
              | lowering-form
              | subset-form
              | enum-form

mcinsn-form ::= '(' 'define-mcinsn' ident
                     ':operands'    '(' operand-decl* ')'
                    [':constraints' '(' constraint*   ')']
                    [':flags'       '(' flag*         ')']
                     ':encoding'    enc-expr
                     ':min-bytes'   integer
                    [':max-bytes'   integer]              ; default = min-bytes
                     ':golden'      '(' golden-case+ ')'
                ')'

lowering-form ::= '(' 'define-lowering' xm-op
                       ':backend' arch
                       ':subset'  subset-name
                      [':when'    '(' predicate* ')']
                       lowering-body
                  ')'

subset-form ::= '(' 'define-subset' subset-name
                     ':include' '(' subset-name* ')'?
                     ':ops'     '(' xm-op*       ')'?
                ')'

enum-form   ::= '(' 'define-enum' enum-name '(' (ident integer)+ ')' ')'
arch        ::= 'x64' | 'arm64' | 'riscv64'
```

`define-subset` 只描述哪些 op 进 reference interpreter 与 differential。`define-enum` 用来命名 condition codes、size codes 等，让 mcinsn 字段可读。

### 4.2 operand 类型

```bnf
operand-decl ::= '(' '$' ident operand-kind ')'

operand-kind ::= 'reg' ':' reg-class
              | 'imm' ':' imm-type
              | 'label'                  ; branch / call target placeholder
              | 'mem' ':' mem-form       ; addressing form
              | 'cc'                     ; condition code (per-arch enum)
```

reg class（v1 必须支持）：

| arch | class | 备注 |
|---|---|---|
| x64 | `gpr32`, `gpr64`, `xmm` | rsp/rbp/r12/r13 等需要触发 golden |
| arm64 | `gpr32`, `gpr64`, `vreg` | `wsp/sp` / `wzr/xzr` 单独处理 |
| riscv64 | `gpr`, `freg` | `x0/zero` 单独处理 |

imm type：`i8`、`u8`、`i16`、`u16`、`i32`、`u32`、`i64`、`u64`。arch-specific 立即数编码（arm64 `imm12<<shift`、`logical-imm`，RISC-V `B/J/U/S-imm`）通过专门 encoding 元件表达，operand 仍记为 `imm:iN`。

mem form（v1 仅 listed forms，未列出的 mode 暂归 driver 手写）：

| form | 语义 | 适用 |
|---|---|---|
| `(base)` | `[base]` | 三 arch |
| `(base disp)` | `[base + disp]`，disp 是 imm | 三 arch |
| `(base index scale)` | x64 SIB | x64 |
| `(base index scale disp)` | x64 SIB + disp | x64 |
| `(pre-index base disp)` | arm64 `[base, #disp]!` | arm64 |
| `(post-index base disp)` | arm64 `[base], #disp` | arm64 |

### 4.3 mcinsn 字段详解

```text
(define-mcinsn x64.add.rr
  :operands     (($dst reg:gpr64) ($src reg:gpr64))
  :constraints  ((same $dst $src))     ; destructive dst
  :encoding     (rex.w $src $dst 0x01 (modrm 0b11 $src $dst))
  :min-bytes    3
  :golden       (((rax rbx) "48 01 d8")
                 ((r12 r13) "4d 01 ec")
                 ((rsp rbp) "48 01 ec")))
```

| 字段 | 必填 | 含义 |
|---|---|---|
| `:operands` | 是 | 名称 + kind，顺序即 emit helper 形参顺序 |
| `:constraints` | 否 | 见 §4.5 |
| `:flags` | 否 | 见下表 |
| `:encoding` | 是 | 见 §4.4 |
| `:min-bytes` | 是 | 该 mcinsn emit 的最小字节数；运行时小于该值即 fail-fast |
| `:max-bytes` | 否 | 默认等于 `:min-bytes`，仅当 emit 长度依赖 operand 时给出；v1 不支持 emit 阶段做 branch relaxation |
| `:golden` | 是 | 至少 3 case，见 §4.8 |

flags（v1 必须识别）：

| flag | 含义 |
|---|---|
| `branch` | 该 mcinsn 含 pc-relative offset，需要 patch |
| `call` | 该 mcinsn 转移控制流到外部目标 |
| `patchable` | 该 mcinsn 可能被运行时 patch（IC、deopt、stub head） |
| `fixed-width` | 长度恒定（典型为 ARM64 / RISC-V，4 字节） |
| `synthetic` | 该 mcinsn 不来自 IR lowering，由 driver 直接 emit（如 NOP、padding） |
| `prefix-only` | 该 form 仅输出 prefix（66H、F2/F3、ARM64 size override），供其它 mcinsn 引用 |

### 4.4 encoding expression

`enc-expr` 是 0 或多个**字段**和**字节**拼接，generator 把它按声明顺序 emit。

#### 4.4.1 通用元件

| 元件 | 含义 |
|---|---|
| `0xNN` | 直接输出 1 字节 |
| `(byte expr)` | 输出 1 字节，`expr` 求值后必须 fit in `u8` |
| `(imm8 $x)` / `(imm16 $x)` / `(imm32 $x)` / `(imm64 $x)` | 输出对应位宽小端立即数 |
| `(disp8 $x)` / `(disp32 $x)` | 输出小端 pc-relative 位移；driver 后期 patch |
| `(field width value)` | 把 `value` 编为 `width` bit；只能出现在 `bitsN` 内 |
| `(concat field*)` | 字段按声明从高位到低位拼接 |
| `(bits16 (concat ...))` / `(bits32 (concat ...))` | 输出 16/32 bit 固定宽，小端写出 |
| `(reg-id $r)` | 取寄存器号的低 N 位（具体 N 由父字段宽度决定）|
| `(reg-hi $r)` | 取寄存器号的扩展位（x64 REX.X/B、ARM64 高位用） |
| `(if pred then else)` | 静态条件输出（pred 必须可在 codegen 时求值） |

#### 4.4.2 x64 元件

| 元件 | 含义 |
|---|---|
| `(rex.w args...)` | REX 前缀，自动根据 args 设置 `W/R/X/B` |
| `(rex args...)` | 同上但 `W=0` |
| `(modrm mod reg rm)` | 1 字节 ModR/M |
| `(sib scale index base)` | 1 字节 SIB |
| `(addr16-prefix)` | `0x67` |
| `(opsz-prefix)` | `0x66` |
| `(rep-prefix)` / `(repne-prefix)` | `0xF3` / `0xF2` |
| `(opcode-w op)` | 把 op 末位 OR 上 1 表示 64-bit 变体（如 `MOV r/m64, imm32`）|

ModR/M `mod` 取值：`0b00`（无 disp）、`0b01`（disp8）、`0b10`（disp32）、`0b11`（reg-direct）。SIB / RIP-relative / disp 选择遵循 Intel SDM，generator typecheck 必须拒绝非法组合。

#### 4.4.3 arm64 元件

ARM64 所有 mcinsn 都是 32-bit fixed width，统一用 `(bits32 (concat ...))` 包裹：

| 元件 | 含义 |
|---|---|
| `(sf-bit $reg)` | 1 bit size override：`gpr32 → 0`，`gpr64 → 1` |
| `(reg5 $r)` | 5 bit 寄存器号 |
| `(imm12 $x [:shift n])` | 12 bit 立即数，`:shift` 默认 0，可为 0 或 12 |
| `(imm16 $x [:lsl n])` | 16 bit 立即数 + 移位（MOVK/MOVN/MOVZ）|
| `(imm-bitmask $x)` | logical immediate 编码（N + immr + imms），generator 在 build-time 算出，无法编码即 fail |
| `(branch-imm26 $label)` | unconditional branch 偏移（4 字节对齐） |
| `(branch-imm19 $label)` | conditional branch 偏移 |
| `(cond $cc)` | 4 bit condition code（引用 `define-enum arm64.cc`） |

#### 4.4.4 RISC-V 元件

RV64 base 也是 32-bit fixed width；高扩展（如 C extension）超出 v1 范围。

| 元件 | 含义 |
|---|---|
| `(opcode-7 $x)` | bit \[6:0\] 主 opcode |
| `(funct3 $x)` / `(funct7 $x)` | funct field |
| `(reg5 $r)` | 5 bit 寄存器号 |
| `(imm-i $x)` | I-type 12 bit immediate |
| `(imm-s $x)` | S-type 12 bit immediate（带位拆分） |
| `(imm-b $label)` | B-type 13 bit branch immediate（位重排） |
| `(imm-u $x)` | U-type 20 bit immediate |
| `(imm-j $label)` | J-type 21 bit jump immediate（位重排） |

B/J/U-type 位重排细节由 generator 实现，golden 必须覆盖跨字节边界的样本（如 `+0x1000`、`-0x1000`、`+0xFFFFE`）。

#### 4.4.5 静态可求值

`:encoding` 内的所有表达式必须在 emit 时仅依赖 operand value，不能引用 frame state、runtime flag、isolate state。任何 runtime-dependent emit 需要由 driver 在调用 generated helper 前完成 lookup，再把结果作为 operand 传入。

### 4.5 constraint

```bnf
constraint ::= '(' 'same'        '$' ident '$' ident ')'
            | '(' 'not-same'    '$' ident '$' ident ')'
            | '(' 'align'       '$' ident integer    ')'   ; operand value 对齐
            | '(' 'imm-range'   '$' ident integer integer ')'  ; [lo, hi] 闭区间
            | '(' 'reg-class'   '$' ident class-name  ')'  ; 收紧寄存器类
            | '(' 'forbid-regs' '$' ident '(' reg-name+ ')' ')'  ; 如 x64 禁 rsp
```

generator 在 build-time 检查 constraint 与 encoding 是否相容（如 `same $dst $src` 不应出现两个独立的 `(reg-id ...)` field）。

### 4.6 predicate（用于 `define-lowering :when`）

只在 lowering rule 出现，emit helper 内部不会执行 predicate。

```bnf
predicate ::= '(' 'arg-kind'     '$' ident kind     ')'   ; imm | reg | mem | label
           | '(' 'fits-i8'      '$' ident          ')'
           | '(' 'fits-i16'     '$' ident          ')'
           | '(' 'fits-i32'     '$' ident          ')'
           | '(' 'fits-imm12'   '$' ident          ')'   ; arm64
           | '(' 'fits-bitmask' '$' ident          ')'   ; arm64 logical
           | '(' 'reg-class-is' '$' ident class    ')'
           | '(' 'equal'        '$' ident integer  ')'
           | '(' 'in'           '$' ident '(' int+ ')' ')'
```

predicate 集合在 v1 固定；后续增加新 predicate 需要更新本节，否则 generator typecheck 失败。

### 4.7 generated emit helper C 接口

每条 mcinsn 都生成一个 inline static helper。命名规则：`xisa_emit_<arch>_<mcinsn-name-with-dots-as-underscores>`。

```c
// 例：x64.add.rr → xisa_emit_x64_add_rr
static inline size_t xisa_emit_x64_add_rr(
    uint8_t *buf, size_t cap,
    uint8_t dst, uint8_t src);
```

约定：

- 返回值是实际写入字节数；若 `cap < min_bytes` 直接返回 0，由调用方决定 fail-fast。
- 不分配内存，不调 `xr_malloc`。
- 不接触 `XmFunc` / `XmIns` / `XrIsolate` / `XrVMContext`，保持与 IR/runtime 解耦。
- branch mcinsn 的 disp operand 接受 `int32_t` placeholder，由 driver 后期 patch。
- generator 同步生成可选 `xisa_mcattr_<arch>.h`，把每条 mcinsn 的 flags / min-bytes / max-bytes 暴露为编译期常量，让 driver 做 reach 检查。

### 4.8 golden encoding

每条 mcinsn 至少 3 个 golden case，必须覆盖：

- **普通**：常见低编号寄存器。
- **扩展**：x64 `r8`-`r15`、arm64 `x16`-`x30`、RISC-V `x16`-`x31`、立即数边界。
- **特殊**：x64 `rsp/rbp/r12/r13`、arm64 `sp/xzr`、RISC-V `x0/zero`、负向 / 跨字节 immediate。

执行链路：

```text
.isa → generated emit helper → byte buffer → expected hex
```

任何不一致 build fail，错误信息至少包含：`.isa` 路径、行号、mcinsn 名称、操作数取值、实际字节、期望字节。

---

## 5. lowering 的位置

v1 不生成完整 lowering switch。原因：

- **regalloc 耦合**：spill/reload、gap moves、edge copies 仍由现有 driver 消化。
- **runtime-sensitive op 多**：`XmOp` 中有 call/deopt/safepoint/OSR/suspend 等不能简单描述成 mcinsn 序列。
- **最小价值已足够**：先替换 machine encoding helper 已能消除最常见字节级 bug。

后续可增加 `define-lowering`，但必须满足：

- 先覆盖 pure scalar subset。
- 不改变 `XmOp` enum 值和现有 IR ABI。
- 不生成 `xm_ops.h`，除非整个 JIT 已准备迁移到 generated opcode 属性表。
- runtime-sensitive op 允许显式标记为 `driver-lowered`。

示例（`arg-kind`、`fits-i32` 见 §4.6；`$src1` / `$src2` 是 `xm.add` 的两个 IR 输入 operand，`$dst` 是其结果 vreg）：

```text
(define-lowering xm.add :backend x64 :subset scalar
  :when ((arg-kind $src2 imm) (fits-i32 $src2))
    (x64.add.ri32 $dst $src1 $src2)
  :default
    (x64.add.rr $dst $src1 $src2))
```

---

## 6. reference interpreter 与 differential

reference interpreter 不是 v1 的全局 ground truth。

### 6.1 可解释子集

只对以下子集自动解释：

- integer / float arithmetic
- comparison
- bitwise
- simple select
- simple load/store mock memory

### 6.2 不自动解释的 op

这些 op 初期不放入 generated interpreter：

- call / helper call
- allocation
- write barrier
- safepoint
- deopt
- OSR
- suspend/resume
- exception
- IC guard / patch site

它们通过专门的 runtime tests、metadata verifier 和 backend parity tests 覆盖。

### 6.3 differential 分层

| 层级 | 覆盖 |
|---|---|
| scalar differential | pure arithmetic / compare / bitwise |
| memory differential | load/store mock memory |
| runtime parity | call/deopt/OSR/safepoint 专项测试 |
| full regression | `ctest` + regression scripts |

---

## 7. branch、patch 与 runtime protocol

### 7.1 branch / patch

driver 负责：

- block layout
- branch target offset 计算
- short/long branch 选择
- relocation patch
- trampoline / island

xisa 负责：

- 生成具体 branch encoding helper。
- 检查 immediate range。
- 提供 `min-bytes` / `fixed-bytes` 属性。

branch relaxation 需要单独设计和测试，不作为 v1 encoding pilot 的前提。

### 7.2 call / deopt / OSR / safepoint

这些路径由 driver 保持主导：

- prologue / epilogue
- ABI register move
- stack map record
- deopt spill copy
- resume entry
- OSR entry
- guard page poll

xisa 只提供底层 machine instruction helper。driver 仍然决定何时 emit、如何记录 metadata、如何 patch。

---

## 8. linemap

linemap 分两层：

| 层级 | 目标 |
|---|---|
| generated mcinsn linemap | 每条 generated emit helper 可反查 `.isa` 行 |
| driver synthetic linemap | prologue、stub、patch island、alignment NOP 标为 synthetic region |

v1 只要求 generated mcinsn linemap。全机器字节 100% 反查必须等 driver synthetic region 也接入后再作为目标。

---

## 9. 与现有 JIT / AOT 的边界

### 9.1 JIT

当前 JIT 管线仍保持：

```text
Xi IR → JIT low-level IR → optimization → regalloc → codegen driver → machine code
```

xisa 只进入最后的 machine encoding 层。

### 9.2 AOT

AOT 仍保持：

```text
Xi IR → C codegen → system C compiler → native binary
```

xisa 不生成 AOT C，不改变 AOT runtime，不参与 `xi_cgen`。

### 9.3 VM

VM bytecode emitter 与 interpreter 不受 xisa 影响。

---

## 10. 验证策略

### 10.1 build-time

| 检查 | 失败后果 |
|---|---|
| DSL parse | build fail |
| DSL typecheck | build fail |
| operand constraint | build fail |
| encoding width / length | build fail |
| golden count | build fail |
| golden byte-equal | build fail |
| min-bytes > 0 | build fail |

### 10.2 integration

| 检查 | 用途 |
|---|---|
| dual-emit byte-equal | 迁移纯 emit helper 时对比现有后端 |
| generated helper unit tests | 覆盖特殊寄存器 / immediate / bitfield |
| scalar differential | 早期行为一致性 |
| backend parity tests | 覆盖 call/deopt/OSR/safepoint |
| full ctest / regression | 每次代码迁移后的总验证 |

如果 dual-emit 发现现有手写 codegen bug，必须先修 root cause、加回归或 golden，再继续迁移；不能让 generator 匹配已知错误。

---

## 11. 与业界系统的取舍

| 系统 | xisa 取舍 |
|---|---|
| DynASM | 借鉴“轻量 emit DSL”，但不用 runtime action list；xisa 生成 C helper |
| Cranelift ISLE | 暂不引入 rewrite system；xray 当前 lowering 深度不需要 |
| LLVM TableGen | 不做 asm parser / disassembler / sched model / multiclass 全栈生成 |

xisa 的目标不是成为通用 compiler backend framework，而是把 xray JIT 后端最容易出错的 machine encoding 部分收敛到一个可检查 schema。

---

## 12. 实施原则

任何具体阶段拆分、退出标准、时间估算见 [`../tasks/073-xisa-implementation.md`](../tasks/073-xisa-implementation.md)。本节只列必须长期遵守的设计原则：

- **不先生成 `xm_ops.h`**：先生成 helper，不动 `XmOp` ABI。
- **不先删除 driver**：先保留 driver，只替换其中的 machine encoding。
- **不先承诺全 interpreter**：先 pure scalar subset，runtime-sensitive op 单独走 backend parity test。
- **不先承诺 RISC-V full parity**：先 subset bring-up；OSR / deopt / safepoint 进 §8 后再追加。
- **dual-emit 发现的现有 bug 必须先修 root cause**：不能让 generator 匹配已知错误编码。

---

## 13. 长期判定标准

本节只描述 xisa 长期希望达成的稳定状态。每个 release 的具体退出条件由 [`../tasks/073-xisa-implementation.md` §7](../tasks/073-xisa-implementation.md#7-完成判定) 给出。

| 指标 | 目标 |
|---|---|
| x64 / arm64 handwritten encoding | 基本删除，只保留 driver logic |
| RISC-V bring-up | subset backend 可在 QEMU 跑 JIT unit tests |
| differential | scalar subset 持续 0 分歧 |
| runtime-sensitive parity | call / deopt / OSR / safepoint 专项 parity test 全过 |
| build-time 增量成本 | generator + golden ≤ 2 秒 |

---

## 14. 修订历史

| 日期 | 改动 | 作者 |
|---|---|---|
| 2026-05-11 | 初稿 | xingleixu |
| 2026-05-11 | 收窄为 C generator、encoding-first、driver-preserving、subset-first | Cascade + xingleixu |
| 2026-05-11 | DSL 章节补完整 BNF / 三 arch encoding 元件 / constraint / predicate / emit helper C 接口；压缩与 task 重复的阶段总览与 v1 指标 | Cascade + xingleixu |
