# X64 JIT Hardening 设计说明

> Version: 1.2 | Date: 2026-04-25 (initial), 2026-04-26 (status reconcile + tests landed)
> Related:
> - `docs/tasks/009-jit-x64-parity.md`
> - `docs/tasks/006-jit-stabilization.md`
> - `docs/engineering/jit_known_limitations.md`
>
> 本文不是新的“大而全 JIT 规划”，而是对 2026-04 这轮 x64 release-only JIT 问题的归因总结与加固清单。
>
> v1.2 修订（2026-04-26 晚）：§10.1–§10.6 描述的 7 个 e2e 回归测试已全部落入 `tests/unit/jit/test_jit_e2e.c`，并把该文件从被注释状态恢复到 ctest 默认运行集合。本机 ARM64 host 上 87/87 ctest + 288/288 回归全过；其中 `test_call_c_fp_live_across_call` 因 ARM64 FP regalloc 不支持 spill 而 SKIP，在 x64 host 上才会真验证 §10.1。详见 §10.8。

---

## 一、背景

本轮排查的直接症状包括：

- `test_call_c` 返回值错误
- `test_fp_basic` 结果错误
- `test_inline_basic` 结果错误
- `test_osr_entry` normal path 结果错误
- release x64 下 `test_jit_e2e` 存在错误结果与卡住现象

这些症状表面上分别落在：

- `CALL_C` 返回值恢复
- 浮点 codegen
- inline 后表达式求值
- OSR/loop normal path
- release-only backend correctness

但实际修复完成后可以确认：

**主要问题并不是“某个高层 opcode 语义没实现”，而是 x64 backend 在 value transport、ABI lowering、regalloc 协议落地上存在多处薄弱点。**

换句话说，当前 x64 最大风险不是“少几个 opcode”，而是：

**值是否在入口、边界、调用、返回、spill/reload 之间被正确地搬运到目标位置。**

---

## 二、本轮已经确认并修复的根因

### 2.1 返回 ABI 错误：`RET` payload 未进入 `RAX`

已确认问题：

- `XIR_JMP_RET` 在 x64 路径中只处理了 tag
- 未将返回 payload 移入 System V ABI 要求的 `RAX`

影响：

- JIT 生成代码返回旧寄存器值或未定义值
- 会表现为“函数逻辑大致正常，但最终返回值错误”

结论：

- `RET` lowering 必须被视为 backend contract 的核心部分，而不是 terminator 的附属逻辑

### 2.2 Edge copy 不完整：跨块值没有被正确搬运

已确认问题：

- x64 边复制发射不完整
- 尤其缺少或遗漏 `reg -> spill` 这类边界同步

影响：

- Phi 值错误
- loop-carried value 错误
- successor block 读到 stale spill 值
- release 下可能表现为循环不收敛、结果错误甚至卡住

结论：

- block edge 不是“跳转本身”，而是“跳转 + value relocation”的组合语义
- `reg -> reg`、`spill -> reg`、`reg -> spill` 三类都必须完整覆盖

### 2.3 Spill-only 参数未初始化

已确认问题：

- `x64_emit_prologue()` 与 `x64_emit_fast_prologue()` 中
- 当参数 vreg 为 spill-only 时，代码直接跳过
- 结果是参数既没有进入寄存器，也没有写入 spill slot

影响：

- 后续 reload 读到未初始化 spill
- 直接影响 `CALL_C`、FP 参数、跨块参数使用

结论：

- 参数入口初始化不能假设“参数一定先进入物理寄存器”
- 对 spill-only vreg，入口路径也必须显式 materialize 到 spill slot

### 2.4 Two-address clobber：`dst` 提前覆盖 `arg1`

已确认问题：

- x64 上 `ADD/MUL/AND/OR/XOR/FADD/FMUL` 等属于 destructive/two-address 形式
- 原先 emission 先将 `arg0` 搬到 `dst`，再读取 `arg1`
- 如果 `dst == arg1`，则 `arg1` 原值会先被覆盖

影响：

- `x * 2` 变成 `x * x`
- `a + b` 可能变成 `a + a`
- 表面上像 inline/FP/普通算术坏了，实际是寄存器别名踩踏

结论：

- x64 backend 不能沿用纯 SSA 三地址心智模型
- destructive destination 必须用统一 helper 处理 alias hazard

---

## 三、这轮修复带来的核心认识

### 3.1 真实主线是 value transport correctness

本轮已修复问题虽然分散在：

- prologue
- edge copy
- return lowering
- arithmetic emission

但本质上都属于同一条主线：

- param → reg/spill
- block_end → successor_start
- vreg → ABI return register
- arg0/arg1 → destructive dst

因此，x64 hardening 的主轴应该从“补 opcode”升级为：

**建立并验证 value transport contract。**

### 3.2 release-only 问题应优先怀疑 RA / ABI / edge 语义

对于这类特征明显的问题：

- debug 不明显
- release 才暴露
- 错法包括错值、偶发 crash、卡住

应优先排查：

1. return ABI
2. entry param materialization
3. edge copy
4. spill/reload
5. destructive dst alias
6. call clobber / save-restore

而不应首先怀疑高层 builder 语义本身。

### 3.3 ARM64 是 x64 最有效的语义对照组

本轮最有效的定位手段之一，就是直接对比 ARM64 与 x64 的：

- prologue
- fast prologue
- call / return path
- spill slot 初始化语义

结论：

- 后续所有 x64 hardening 工作都应默认带一项：
  **与 ARM64 同路径逐点对照**

---

## 四、x64 backend 当前最需要加固的协议面

下面不是“所有未来工作”，而是从本轮根因倒推出的高风险协议面。

### 4.1 入口协议：参数 materialization contract

必须明确的语义：

- 每个参数在 function entry 时必须有确定落点
- 落点可能是：
  - physical register
  - spill slot
  - register + spill 双写
- 不能因为 `first_reg < 0` 就跳过参数初始化

建议约束：

- normal prologue 和 fast prologue 必须共享同一语义说明
- 入口参数初始化后，任何后续 reload 都不允许读未初始化 spill

### 4.2 边界协议：edge relocation contract

必须明确的语义：

- block terminator 完成前，所有 successor 所需 live value 都已经被搬到 successor 期望位置
- 需要统一覆盖：
  - phi copies
  - split transition copies
  - `reg -> reg`
  - `spill -> reg`
  - `reg -> spill`
  - GP / FP 两套

建议约束：

- edge copy resolver 必须被当作独立子系统维护
- 不允许在若干 opcode 内部“顺手修补”跨块值流

### 4.3 调用协议：call boundary contract

本轮确认 `CALL_C` 问题已经被 spill-only 参数初始化修复，但调用路径仍然是 x64 高风险区域。

必须明确的语义：

- call 前 live 值保存规则
- call 后 payload/tag 恢复规则
- helper/stub 与 direct call 的 ABI 一致性
- FP/GP 返回路径一致性
- call-result tag 与 payload 的写回时机

建议约束：

- 所有 call family 共用一份 x64 ABI checklist
- 不能把 return payload 与 return tag 的恢复逻辑分散成若干“局部惯例”

### 4.4 返回协议：return lowering contract

必须明确的语义：

- i64 / ptr / boxed payload 的返回寄存器
- tag 返回寄存器或 side-channel 存放位置
- `RET` lowering 是 ABI contract，不是普通 terminator emission

建议约束：

- return path 应单独列 checklist
- 凡是涉及 `CALL_C`、`RET`、deopt 返回桥的地方，都必须按 ABI 清单逐项核对

### 4.5 破坏性指令协议：destructive-dst contract

必须明确的语义：

- 哪些 x64 op 是 two-address / destructive destination
- 当 `dst` 与任一源操作数别名时，如何避免 clobber
- commutative 与 non-commutative 场景分别如何处理

建议约束：

- 不再允许各 opcode 分散手写 `ensure_dst` + `get_operand`
- 应统一走专门 helper

---

## 五、推荐的 hardening 顺序

## Phase H1：补协议，不先补新 opcode

目标：避免再出现“实现了一条语义，但值没搬对”的情况。

### H1.1 梳理 x64 value transport checklist

为以下路径建立统一 checklist：

- normal prologue
- fast prologue
- edge copies
- `CALL_C` / `CALL_DIRECT` / `CALL_SELF_DIRECT`
- `RET`
- deopt recovery
- OSR entry
- safepoint live-value 保存

每个路径至少回答：

- value 来自哪里
- 当前位于哪里
- 目标应位于哪里
- 谁负责搬运
- 搬运后谁拥有最终语义

### H1.2 审计所有 destructive/two-address op

优先审计：

- `ADD`
- `MUL`
- `AND`
- `OR`
- `XOR`
- `FADD`
- `FMUL`
- `SUB`
- `FSUB`
- `DIV/MOD`
- `SHL/SHR`

说明：

- 本轮已经修了 commutative 的一部分
- 但 destructive-dst 问题不应只靠“修几个当前失败的 case”结束

### H1.3 建立 ARM64 对位审计表

建议将以下内容表格化：

- 入口参数初始化
- spill-only 处理
- edge copy 触发点
- call stub 保存/恢复
- return ABI
- OSR/deopt 入口协议
- safepoint 相关状态保存

目标不是抽象共享 codegen，而是：

**确保两端 backend 在最后一层 contract 上不漂移。**

---

## Phase H2：补 x64 execute-level 定点回归

目标：从“能编码”提升为“能稳定执行”。

### H2.1 强化 `test_jit_e2e` 的 x64 定点覆盖

本轮已证明下列测试对 x64 特别有价值：

- `test_call_c`
- `test_fp_basic`
- `test_inline_basic`
- `test_osr_entry`

建议继续补的测试族：

#### A. entry / spill-only

- spill-only int param
- spill-only fp param
- fast prologue spill-only param
- 参数进入后立即跨块 reload

#### B. edge copies

- `reg -> reg`
- `spill -> reg`
- `reg -> spill`
- loop back-edge
- false edge / exit edge
- GP / FP 混合

#### C. destructive-dst

- `dst == arg1`
- `dst == arg0`
- `dst == scratch`
- const + vreg
- spill + reg
- FP / GP 各一套

#### D. call family

- `CALL_C`
- `CALL_DIRECT`
- `CALL_SELF_DIRECT`
- call 后结果立刻跨块流动
- call + safepoint
- call + FP return

#### E. ABI return

- i64 return
- ptr return
- FP return
- tagged return payload/tag 联动

### H2.2 增加更细粒度 execute-level smoke tests

除了 `test_jit_e2e`，建议增加小型 execute-level test：

- 专门测试 one-op + one-branch
- 专门测试 edge copy
- 专门测试 self-call / recursive call
- 专门测试 return ABI

目的：

- 缩短定位路径
- 避免每次都在大测试里猜哪个子系统坏了

---

## Phase H3：增强诊断与失败可见性

目标：避免 backend failure 被 VM fallback 掩盖。

### H3.1 提高 JIT 失败日志粒度

目前过于粗糙的失败信息容易导致：

- 看起来只是“JIT 没提速”
- 实际是 x64 codegen 在某个点失败

建议最少打印：

- 函数名
- block id / block label
- instruction index
- opcode 名
- dst / args
- 当前 `ra_pos`
- 当前 backend 名称
- 是否 fallback 到 VM

### H3.2 为关键 value transport 路径补 debug 断言

建议在 debug 构建下强化：

- spill-only param 是否已 materialize
- edge copy 前后的 live destination 是否存在
- return lowering 前 value 是否在预期位置
- destructive-dst helper 是否遇到 alias hazard

### H3.3 将 timeout / 卡住与 backend failure 区分开

远端测试或 playground 场景中，以下现象经常被混淆：

- JIT 编译失败后 fallback 到 VM，导致性能异常
- 测试进程卡住
- 运行超时
- 真正 crash

建议后续把这些状态分层报告，而不是全部落成“无输出”或“跑很慢”。

---

## 六、当前仍应重点复验的 x64 路径

本轮 `test_jit_e2e` 已恢复通过，但以下路径仍然值得作为下一轮重点回归对象：

### 6.1 `CALL_SELF_DIRECT` / 递归热点路径

原因：

- 历史上 `fib` 一类路径暴露过 x64 JIT failure / fallback 现象
- 这类路径天然会同时覆盖：
  - 调用约定
  - 返回值
  - 寄存器压力
  - loop / recursion

建议专项复验：

- 递归 `fib`
- 非递归 self-call
- 带 spill 参数的 self-call
- FP / mixed self-call

### 6.2 真正命中 OSR entry 的路径

本轮恢复了 `test_osr_entry` normal path correctness，但仍建议专项确认：

- 真正命中 OSR entry
- OSR 入口 live value 是否完整恢复
- OSR 后再退出 / deopt 是否正确

### 6.3 deopt / safepoint / spill 联合路径

这类路径最容易把“平时不出问题”的 value transport 漏洞放大。

建议专项覆盖：

- spill 后触发 deopt
- call 前后 safepoint
- FP spill + deopt
- loop + safepoint + edge copy

---

## 七、与现有文档的关系

### 与 `jit_x64_parity_plan.md` 的关系

- parity plan 更偏“缺什么、补什么、如何分 phase 实施”
- 本文更偏“这次真实 bug 说明 x64 最脆弱的是哪些 contract，以及如何加固”

### 与 `jit_stabilization_plan.md` 的关系

- stabilization plan 是整个 JIT 子系统的稳定化路线
- 本文只聚焦 x64 backend，尤其是 value transport / ABI / RA contract

因此，本文适合作为：

- x64 bug 排查 checklist
- x64 correctness hardening 的专项说明
- 未来继续写 parity/stabilization 任务时的判定依据

---

## 八、建议的退出标准

x64 backend 不应只以“`test_jit_e2e` 这次过了”为退出条件。

建议最少达到：

1. `test_jit_e2e` 在 x64 release 稳定通过
2. 覆盖 spill-only / edge copy / destructive-dst / return ABI / call family 的 execute-level 定点测试存在
3. x64/ARM64 对位审计表完成
4. JIT failure 日志可以定位到函数 + block + ins + opcode
5. `CALL_SELF_DIRECT` / OSR / deopt / safepoint 至少各有一条 execute-level 回归

在达到以上条件前，应继续把 x64 定义为：

**已恢复基本正确性，但仍处于 hardening 阶段。**

---

## 九、结论

本轮修复最重要的结论不是“修了 4 个 bug”，而是明确了 x64 backend 的主要风险模型：

**风险中心在 value transport contract，而不是 opcode 数量本身。**

后续工作如果继续沿着“缺哪个 opcode 就补哪个”的方式推进，仍然可能在：

- 入口
- 边界
- 调用
- 返回
- spill/reload
- destructive destination

这些位置反复出现 release-only 问题。

因此，x64 hardening 的首要任务应是：

**把 value transport、ABI lowering、regalloc 落地协议做成可检查、可测试、可诊断的体系。**

---

## 十、已落地的 hardening 进展

### 10.1 第一批：FP scratch contract 收紧（codegen: 2026-04-25）

问题：

- `xmm15` 同时被当作 allocatable FP register 与 `X64_SCRATCH_XMM`
- `CALL_C` stub 仅保存 `xmm0–xmm14`
- 三者交叉触碰 call boundary / scratch / live FP 三个 contract

修复（已落地）：

- `X64_MAX_FP_REGS`：`16 → 15`
- `xir_target_x64.c` 暴露 `nfpr=15`、`nfpr_caller_save=15`，allocatable 仅 `xmm0–xmm14`
- `xir_codegen_x64.c` 的 `x64_alloc_fp_regs[]` 同步去掉 `xmm15`

回归测试（**已落地**）：

- `test_call_c_fp_live_across_call`：跨 `CALL_C` 让 16 个 f64 参数同时存活，期望结果 `136.0`。
- 本机 ARM64 host 上需 SKIP（ARM64 FP regalloc 不支持 spill，16 个 FP vreg 同时 live 会触发 `[regalloc-fp]` fail-loud）；x64 host 上为真验证路径。

### 10.2 第二批：非交换 destructive-dst 的 alias-aware emission（codegen: 2026-04-25）

问题：

- `XIR_SUB / FSUB / FDIV` 走的是简单的 `mov rd, rn; OP rd, rm` 模板
- 当 `rm == rd && rn != rd`（dst 与 arg1 共寄存器）时，`mov rd, rn` 会先把 arg1 覆盖掉
- 结果是 `dst = something OP itself` 之类的静默错算
- 同时 `x64_get_operand` 对 const / spill-only 操作数都会落到唯一的 GP/FP scratch
- 两个 args 都需要 scratch 时，第二次会无声覆盖第一次

修复：

- 新增 `x64_emit_noncommutative_gp` / `x64_emit_noncommutative_fp` helper
  - `rm == rd && rn != rd`：通过 scratch 中转
    - `mov scratch, rn`（若 `rn != scratch`）
    - `OP scratch, rm`
    - `mov rd, scratch`
  - 把 arg1 在 rd 中的存活时间拉到 OP 之后，避免静默覆盖
- `XIR_SUB / FSUB / FDIV` 全部改走新 helper
- 给 `x64_prepare_commutative_gp / fp` 与新 helper 都加 DCHECK：
  - `rn == scratch && rm == scratch && arg0 != arg1` 时直接 fail loud
  - 把"两个 args 都需 scratch"这条历史 silent miscompile 转成可检测错误

回归测试（**已落地**）：

- `test_int_sub_chain`：`f(x, y) = x - (x - y) = y` 双 SUB 链。
- `test_fp_noncommutative_chain`：`f(x, y) = (x - y) / y` FSUB+FDIV 链。
- 两者覆盖"arg1 在后续指令仍 live、可能与 dst 共寄存器"的非交换场景，在 ARM64 host 上也能跳过所有 PASS。

### 10.3 第三批：binop 双 args 都需 scratch 的真正修复（codegen: 2026-04-25）

问题：

- 第二批已经把这条路径转成 DCHECK fail-loud
- 但只要某条 binop 的 `arg0 != arg1` 且两者都是 const / spill-only，JIT 会直接 abort
- 实际场景下，spill-only + spill-only 在高 RA 压力时是会发生的

修复：

- 新增轻量探测函数：`x64_probe_phys_reg`、`x64_arg_needs_scratch_gp`、`x64_arg_needs_scratch_fp`
- 让 binop helper 在第一次 `x64_get_operand` 之后探测 arg1 是否也会落到 scratch
- 命中时根据是 commutative 还是 non-commutative 走不同 fallback：
  - **commutative**：先把 arg0 从 scratch 移到 rd 释放 scratch，再加载 arg1 到 scratch；caller 仍按 `OP rd, rm` 模板 emit
  - **non-commutative GP**：`push scratch`（stash arg0）→ 加载 arg1 到 scratch → `pop rd`（rd = arg0）→ `OP rd, scratch`
  - **non-commutative FP**：用 `sub rsp, 16; movsd [rsp], xmm15` 与 `movsd fd, [rsp]; add rsp, 16` 实现 16 字节栈滑窗的 stash / pop
- 替换原 DCHECK：剩余的 DCHECK 仅守 `dst == scratch` 的三向冲突，留作进一步 hardening 用

回归测试（**已落地**）：

- `test_binop_pressure_chain`：12 个 i64 参数的链式 ADD/SUB（覆盖 commutative + non-commutative 两条路径）。
- 12 个参数同时存活会迫使 RA 在 x64（仅 11 个 GP）下产生 spill；ARM64 可跳过但仍验证 IR 正确性。

### 10.4 第四批：deopt + spill 与 CALL_SELF_DIRECT 的 execute-level 回归（codegen: 2026-04-25）

问题：

- `test_deopt_guard` 仅覆盖低 vreg 压力的 guard fail，没有触碰 deopt stub 的 spill copy 路径
- `XIR_CALL_SELF_DIRECT` 在 x64 已经实现（含 fast prologue / safepoint id 写入 / frame stack-map 恢复），但 e2e 测试中**完全没有覆盖**
- 审计中还发现：**x64 codegen 完全没有 `emit_osr_stubs` 的实现**（ARM64 有），见 §10.5

修复（已落地）：

- `x64_emit_deopt_stub` 中加了 spill copy 循环：`mov R11, [RBP-spill]; mov [R14+DEOPT_SPILL_SAVE+s*8], R11`，把所有活的 spill 槽写到 `jit_ctx->deopt_spill_save[]`，让 deopt 后 VM 端能恢复

回归测试（**已落地**）：

- **`test_deopt_with_spill_pressure`**：12 个 i64 参数 + 中段 `GUARD_NONNULL`
  - 强制 RA 进入 spill 区
  - x64 上走 `mov [rbp-spill], r11; mov [r14+save_offset], r11` 的 spill copy 循环
  - 验证 success path（`sum=78`）与 deopt path（`DEOPT_MARKER`）都正确。
- **`test_call_self_direct`**：递归 `f(n) = (n==0) ? 0 : n + f(n-1)`
  - 走 register-passing self-call → fast_entry → `X64_PATCH_CALL_SELF_FAST` / ARM64 `PATCH_CALL_SELF_FAST`
  - 验证 `f(0)=0`、`f(1)=1`、`f(5)=15`、`f(10)=55`
  - 同时锁住 safepoint id 写入与 frame stack-map 恢复路径。

### 10.5 第五批：x64 OSR stub 实现（codegen: 2026-04-25）

问题：

- 之前审计发现 **x64 codegen 完全没有 `emit_osr_stubs` 对应实现**
- ARM64 已经有完整版本（`xir_codegen.c::emit_osr_stubs`）
- x64 backend 上 OSR 完全不可用，`test_osr_entry` 在 x64 host 跑会走 `(no OSR entries)` 分支静默 skip

修复：

- **`X64CodegenCtx`**：新增 `osr_snaps[XIR_MAX_OSR_ENTRIES]` 与 `nosr_snap`；`frame_patch_sub` 容量从 8 扩到 16 以容纳 8 个 OSR stub
- **`x64_emit_block`**：在 loop_header 块开头按 ARM64 同样的方式收集 snapshot（含 `has_coro_deopt` 跳过保护）
- **`x64_osr_should_skip_vreg`**：跳过本块定义的 vreg 与已 coalesce 的 phi 输入（行为对位 ARM64 helper）
- **`x64_osr_materialize_const`**：把 `XIR_CONST_I64/PTR/F64` 直接 materialize 到目标 phys reg；FP 路径用 R11 中转 `MOVQ`
- **`x64_emit_osr_stubs`**：每个 snapshot 发一个 stub
  - 标准 prologue（push rbp / sub rsp / push callee-saved / load coro+jit_ctx / smap 写入）
  - `mov R11, RSI` 把 values_ptr 救到 scratch（避免 RSI 被 vreg 加载覆盖）
  - **Pass 1**：所有有 `bc_slot` 的 phys reg vreg 从 `[R11 + slot*8]` 加载（GP `mov`、FP `movsd`），同时填 `entry->slots[]`
  - **Pass 2**：无 bc_slot 但有 `XIR_CONST_*` def 的 vreg 直接 materialize（此时 R11 已自由可作 const 中转）
  - `JMP rel32` → loop header 块（`block_offsets` 已知，displacement 直接计算无需 patch）
- **主流程**：在 `x64_emit_resume_entry` 之前调用 `x64_emit_osr_stubs`，使其与 deopt / call_c / barrier stubs 处于同一阶段

设计差异（vs ARM64）：

- ARM64 一次循环里穿插处理 bc_slot loads 与 const materialize（用 X16 装 values_ptr，X17 装 const 中转）
- x64 只有一个 GP scratch（R11）。改为 **两遍 pass**：先做所有 bc_slot loads（R11 = values_ptr 不动），再统一做 const materialize（这时 R11 可自由复用）
- 这样无需第二个 scratch，也不打栈，逻辑直观

回归测试（现状）：

- `test_osr_entry` 现在在默认 ctest 内跑过（OSR 运行环境修复后，见 §10.8）。
- 本机 ARM64 host 上因没有 Blueprint、`emit_osr_stubs` graceful skip（`nosr=0`），只验证 normal entry 路径不被破坏。
- 在 x64 host 上跑同一个测试应从 `(no OSR entries)` 静默路径切换到 `osr_ok=1 osr=37` 真验证路径（待 x64 host 复跑确认）。

### 10.6 第六批：OSR 多重入 + loop-invariant 压力回归（设计: 2026-04-25）

回归测试（**已落地**）：

- **`test_osr_entry_pressure`**：`f(count, p1, p2, p3, p4) = count * (p1+p2+p3+p4)`
  - 2 个 phi vreg（count, sum）+ 4 个 loop-invariant 参数 + 链式 ADD 中间结果，让 loop header 入口同时活跃 8+ 个 vreg
  - 覆盖 x64 OSR stub 的：
    - **Pass 1**：多个 bc_slot vreg 从 `values[]` 加载（`mov dst, [r11+slot*8]`）
    - **Pass 2**：内联的 `0`/`1` const vreg 在没有 bc_slot 时直接 materialize
    - **可重入性**：用同一 `res.code` 跑 4 个 round（不同的 `count/p1..p4`），验证没有依赖前次 frame 残余。
- ARM64 host 上 `nosr=0`，只跑 normal entry；x64 host 同一代码上会额外验证 OSR stub 两遭 pass + 可重入性。

附带修复（已落地，不在 hardening 范围）：

- `src/vm/xvm_api.c` 中 3 处调用 `xr_vm_get_current_ctx`（不存在的函数名）改为 `xr_vm_current_ctx`
  - 这是与本批工作无关的独立 regression，已落库；属于解阻塞 build 的副产物

### 10.7 已知尚未处理项

- **OSR 入口 spill-only vreg 初始化** ✅ codegen 落地 / 真验证待 x64 host
  - ARM64：`emit_osr_stubs` 加 Pass 3。Pass 2 之后 `SCRATCH_REG (X16)` 仍是 values_ptr，用 `SCRATCH_REG2 (X17)` 作中转：对 `ri<0 && spill>=0 && bc_slot>=0` 的 vreg，`ldr X17, [X16+bc_slot*8]; str X17, [SP+SPILL_BASE+spill*8]`。`osr_should_skip_vreg(... ri=-1)` 跳过 osr_blk 内被定义的 vreg（含 phi dst）。
  - x64：`x64_emit_osr_stubs` 加 Pass 0（必须在 Pass 1 之前；Pass 1 会饱和 alloc_regs[]）。`R11` 装 values_ptr，借 `alloc_regs[0]=RAX` 作中转（Pass 1 会立即覆写 RAX，但 spill slot 已落盘）。
  - 本机 ARM64 上 nosr=0（无 Blueprint），新增字节流随 OSR stub emit 但不被执行；现有 ctest 87/87 + 288/288 不退化。x64 host 上 high-pressure loop header 是真验证路径。
- **LSRA spill-slot overlap + 同源 normal-entry prologue spill-only init** ✅ 已落地
  - 现象：24-param 的 `f(p0..p23) = sum` 在修前得到 258 / 302（应 300）。
  - 双根因（必须一起修）：
    1. **LSRA `assign_spill_slot` fresh 路径用 `range_end(r)` 设 `slot_end[s]`**。`r` 是当前 LsRange sibling（如 head [0, 44)），但同 vreg 的 tail [44, 50) 通过 bundle 路径会再用同一 slot。第二个不相关的 vreg `rstart=44` 来 reuse 检查 `slot_end[0]=44 ≤ 44 → true`，复用 slot 0；之后 vreg=22 tail bundle 路径强制也用 slot 0，于是两个在 [44, 50) live 的 vreg 共享 slot 0。
       - 修法：`assign_spill_slot` fresh 时用**完整 vreg 的所有 sibling 中最大 `range_end`** 设 `slot_end[s]`，预占整个 vreg 生命周期。bundle reuse 仍然合法（phi-connected siblings 按构造 disjoint）。
    2. **`emit_prologue` 用 `xra_vreg_first_reg(i)` 决定 param 装哪**。该 helper 返回第一个有 reg 的 segment——但若 leading segment 是 spill，它指向后面的 reg；prologue 装错地方且 spill slot 不被初始化。
       - 修法：用 `xra_reg_at_pos(xra, i, 0)` + `xra_vreg_live_at(xra, i, 0)`。leading reg 走原路径；leading spill 用 `SCRATCH_REG (X16)` 中转把 `args[i]` 装到 spill slot。
  - 单独修任一个都不够：只修 LSRA → prologue 不装 spill-only param；只修 prologue → 装入正确位置但被另一个 vreg 复用覆写。
  - 触发：`num_params >= 23`（ARM64 22 allocatable）或 `>= 12`（x64 11 allocatable）。现有测试集无该 case，所以长期未暴露。
  - 测试：`test_spill_only_param_init` 在 `test_jit_e2e.c`，3 个 input vector（ones / 1..24 / 交替正负），每个都验证 spill-only param 能正确读到 args 值。
  - 验证：本机 ARM64 上 ctest 87/87 + regression 288/288 全过。
- **OSR 入口 + 后续 deopt 联合回归**（中）
  - 现在 OSR 路径已通，可以在 x64 host 上构造一个"OSR 进入后某次迭代 guard fail 触发 deopt"的回归
- **`dst == scratch` 与 binop 双 args 都需 scratch 同时发生**（低）
  - dst 是 spill-only 整数 / FP vreg 时的三向冲突
  - 当前仍是 DCHECK fail-loud

### 10.8 状态（更新于 2026-04-26 晚）

Codegen 落地情况：

- §10.1–§10.5 的 codegen 改动已经全部进入主线
  - FP scratch contract、binop alias-aware emission、scratch 双占 fallback、deopt+spill copy、x64 OSR stub 实现
- §10.6 的附带修复（`xvm_api.c` 的 `xr_vm_current_ctx` typo）已经进入主线

`tests/unit/jit/test_jit_e2e.c` 恢复运行环境（本轮新增）：

- 该文件之前长期失修：
  - `xir_pass_const_prop` 已被 `xir_pass_sccp` 替换，编译都不过
  - 旧的 `jit_callN` helper 一直传 `coro=0`，跑到 prologue 第一行 `LDR JIT_CTX_REG, [CORO_REG, ...]` 立刻 SIGSEGV
  - 多个 if/else/loop 测试 `xir_block_set_br` 后没有 `xir_block_add_pred`，pipeline 里 `xir_verify_cfg` 直接 DCHECK fail
  - `test_load_field` / `test_store_load_field` / `test_call_c_const_arg` 写入 fake_obj 时没考虑 `XIR_XRVALUE_PAYLOAD_OFFSET=8`
  - `test_alloc_inline` 的 fake_coro 没设置 `coro->jit_ctx`，且把 `XIR_GC_CURRENTWHITE_OFFSET` 写成了过时的 `101`（实际 `109`）
  - 因此 `tests/unit/CMakeLists.txt` 长期把它注释掉，没在默认 ctest 内
- 修复方式：
  - 用一个最小 `XrCoroutine` + `XrJitScratch` + `mmap` 的 safepoint guard page 构造 fake JIT 运行环境，所有 `jit_callN` 共用这个 env
  - `safe_codegen` 在跑 pipeline 前显式 `xir_rebuild_preds`，避免测试漏写 `add_pred` 时 verify 失败
  - 修正 LOAD_FIELD / STORE_FIELD 的 payload offset、`XIR_GC_CURRENTWHITE_OFFSET` 等过时常数
  - `emit_osr_stubs` 在 `func->proto == NULL` 时 graceful skip（之前会段错误，是 unit test 路径下的 NULL deref，与正常 JIT 路径无关）
  - 把 `add_xray_unit_test(test_jit_e2e ...)` 重新启用

回归测试落地情况：

- §10.1–§10.6 在本文档中描述的 7 个 e2e 测试**全部已落入 `tests/unit/jit/test_jit_e2e.c`** 并进入默认 ctest：
  - `test_int_sub_chain` ✓
  - `test_fp_noncommutative_chain` ✓
  - `test_call_c_fp_live_across_call` ✓ (本机 ARM64 SKIP；x64 host 真验证)
  - `test_binop_pressure_chain` ✓
  - `test_deopt_with_spill_pressure` ✓
  - `test_call_self_direct` ✓
  - `test_osr_entry_pressure` ✓
- 本机 ARM64 host 当前结果：
  - `ctest`：87/87 pass（含 `test_jit_e2e`）
  - `scripts/run_regression_tests.sh`：288/288 pass

x64 host 复跑建议：

- 本机非 x86_64，x64 codegen 改动不会被本机真机执行。x64 真机回归仍建议在 x64 host 上跑一次完整 `test_jit_e2e`，重点关注：
  - `test_call_c_fp_live_across_call` 在 x64 上不再 SKIP，应得到 `136.0`（§10.1）
  - `test_osr_entry`、`test_osr_entry_pressure` 应从 `(no OSR entries)` 切换到 `nosr=1` + 真验证路径（§10.5–§10.6）
  - `test_deopt_with_spill_pressure`、`test_call_self_direct`、`test_binop_pressure_chain` 等其余 4 个新测试已是 backend-agnostic IR，正确性在 x64 上同等验证

本轮额外落地（§10.7 头两项）：

- OSR 入口 spill-only vreg 初始化（ARM64 Pass 3 + x64 Pass 0），见 §10.7 第一项
- LSRA spill-slot overlap + normal-entry prologue spill-only param init（双根因），见 §10.7 第二项

下一步建议（按优先级）：

1. **x64 host 完整 release 回归**（贯穿 §10.1 / §10.5 / §10.6 / §10.7 #1）：
   - `test_call_c_fp_live_across_call` 在 x64 上不再 SKIP，应得到 `136.0`
   - `test_osr_entry` / `test_osr_entry_pressure` 应切换到 `nosr=1` + 真验证路径，含 OSR Pass 0 spill-only 初始化
   - 其他 backend-agnostic 新增 case 同样作 x64 codegen 路径独立确认
2. **OSR 入口 + 后续 deopt 联合回归**（中，§10.7 第三项）：x64 host 上构造"OSR 进入后某次迭代 guard fail 触发 deopt"的回归
3. **`dst == scratch` 与 binop 双 args 都需 scratch 同时发生**（低，§10.7 第四项）：当前 DCHECK fail-loud；可加 push/pop stash dst 的 fallback path
