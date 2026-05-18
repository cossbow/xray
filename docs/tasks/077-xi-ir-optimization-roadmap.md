# 077 — Xi IR 优化 master roadmap

> Xi IR 层优化的权威规约。与 076 并列：076 是 machine code 层（Xm → bytes），077 是 high-level IR 层（AST → Xi → Xm 入口之前的 SSA 优化）。
> 两份 spec 各自独立可演化，但共享同一组纪律（fail-fast、单一真相源、反向不变量、回滚锚点）。

## 开发原则

**不考虑向后兼容。不妥协。只选最优设计。**

xray 没有外部用户、没有已发布的 ABI，Xi IR 的 enum / invariant / pass order 可以自由演化。
本方案中每一项直接瞄准最终正确设计——不做临时 guard、不做两步迁移、不加兼容层。
如果一个改动需要动 50 个文件，就在一个 commit 里完成。

## 阶段坐标禁入源码与 commit

- `Xi-0` / `Xi-1.4` / `Xi-3.2` 等是本文档的**实施坐标**，仅用于文档内交叉引用与计划跟踪
- **禁止**在源码注释、doc comment、git commit message 中出现 `Xi-N.M` / `Phase` / `Round` / `Step N` / "这一步" / "本次重构" 等阶段表述
- **禁止**在源码注释中引用 `docs/tasks/077-...md` 路径——文档会重命名 / 归档，引用必然失效
- 任何要长期保留的设计意图，写到对应函数 / 类型 / pass 的 doc comment 里
- 源码、commit、对话注释必须**自包含**：直接说事实和原因
- 反例：`// Xi-1.1 TBAA: mark mem_group` / `feat(xi): finish Xi-0.3 verifier`
- 正例：`// TBAA group id used by alias-aware LICM/GVN-PRE` / `Strengthen xi_verify: dominance + arity + side-effect contract`

## 启动前提

本 roadmap 不可独立启动，需要以下前置条件全部就绪：

| 前置 | 来源 | 必需性 |
|---|---|---|
| 068 A1-A4 已完成（pipeline / canon / lower / verify 债务） | 068 | 硬依赖：Xi-0 全部子项以此为入口 |
| 076 S0 quarantine 生效（old codegen 不再扩大） | 076 | 硬依赖：防止 Xi 加 pass 时旧 codegen 路径继续扩张 |
| 076 S1 `xisa/xm/helpers.def` 上线（helper effect 真相源） | 076 | 硬依赖：Xi-1.1 TBAA 的 effect 来源 |
| 076 S1 `xisa/xm/ops.def` 上线（XmOp 契约表） | 076 | 硬依赖：Xi 新增 op 需同步进 L1 ops.def |
| `xi_verify` 升级（Xi-0.3）完成 | 077 本身 | 软依赖：所有 Xi-1+ 新 pass 必须有 verifier 配合 |

未满足上述前置时不应启动 Xi-1+；Xi-0 可与 068 / 076 S0/S1 并行推进。

---

## 范围划分

| 层 | 负责 | 真相源 | 主规约 |
|---|---|---|---|
| **高层 IR（Xi）** | SSA 优化、分析、stage contract、pass pipeline | `src/ir/xi.h` + pass table | **本文（077）** |
| **低层 IR（Xm）+ codegen** | machine encoding、regalloc、frame、runtime 契约 | `xisa/` DSL | 076 |
| **前端 / analyzer** | lex / parse / typechecker / canon | `src/frontend/*` | 068 / 069 |
| **AOT C backend** | Xi → C | `src/aot/xi_cgen.c` | 066 |

077 **只管** Xi IR（高层 SSA）。Xi → Xm 的 lowering 边界属于 076 的 S2-S6。

---

## 当前基线（2026-05-12 现状）

基于 `src/ir/` 实际代码。**本文中的源码引用以函数名 / 类型名 / 文件名为准**；行号仅作为当前快照参考，代码变动后可能失效。

### 已实现的 SSA 优化 passes

`src/ir/xi_opt.c` 中的 `xi_pass_table[]`（当前快照）：

```c
/* Pass table: ordered by recommended execution sequence.
 * The driver runs all passes whose min_level <= requested level. */
static const XiPassDesc xi_pass_table[] = {
    /* name              fn                       min_level      flags               in_stage
       out_stage */
    {"constfold", xi_opt_const_fold, XI_OPT_LIGHT, XI_PASS_NONE, XI_STAGE_RAW, XI_STAGE_RAW},
    {"strength_reduce", xi_opt_strength_reduce, XI_OPT_LIGHT, XI_PASS_NONE, XI_STAGE_RAW,
     XI_STAGE_RAW},
    {"copy_prop", xi_opt_copy_prop, XI_OPT_LIGHT, XI_PASS_NONE, XI_STAGE_RAW, XI_STAGE_RAW},
    {"phi_simplify", xi_opt_phi_simplify, XI_OPT_LIGHT, XI_PASS_NONE, XI_STAGE_RAW, XI_STAGE_RAW},
    {"dce", xi_opt_dce, XI_OPT_LIGHT, XI_PASS_NONE, XI_STAGE_RAW, XI_STAGE_RAW},
    {"sccp", xi_opt_sccp, XI_OPT_FULL, XI_PASS_NONE, XI_STAGE_RAW, XI_STAGE_RAW},
    {"gvn", xi_opt_gvn, XI_OPT_FULL, XI_PASS_NEEDS_DOM, XI_STAGE_RAW, XI_STAGE_RAW},
    {"licm", xi_opt_licm, XI_OPT_FULL, XI_PASS_NEEDS_DOM, XI_STAGE_RAW, XI_STAGE_RAW},
    {"inline", xi_opt_inline, XI_OPT_FULL, XI_PASS_NONE, XI_STAGE_RAW, XI_STAGE_RAW},
    {"ifconv", xi_opt_ifconv, XI_OPT_FULL, XI_PASS_NEEDS_DOM, XI_STAGE_RAW, XI_STAGE_RAW},
};
```

### 已有分析模块

| 文件 | 内容 | 成熟度 |
|---|---|---|
| `xi_analysis.c` | Dominator tree (Cooper-Harvey-Kennedy) / RPO | ✅ 成熟 |
| `xi_defuse.c` | Def-use chain | ✅ 成熟 |
| `xi_loop.c` | Natural loop detection + nesting | ✅ 成熟 |
| `xi_escape.c` | 4-level escape lattice（NONE/ARG/HEAP/GLOBAL） | ✅ 成熟 |
| `xi_arc.c` | Escape-driven ARC retain/release 插入 | ✅ 成熟 |
| `xi_verify.c` | 结构 + stage-aware invariant | ⚠️ 068 C1 要求扩展 |
| `xi_pass_close.c` | Closure meta / env layout / cell index | ✅ 成熟 |
| `xi_backend_lower.c` | High-level → `XI_CALL_BUILTIN` 下沉 | ✅ 成熟 |
| `xi_pipeline.c` | AST → canon → lower → verify → opt → escape → rep → backend → arc → emit | ✅ 成熟 |

### Pass framework（工业级基础）

- **Stage contract**：`RAW → CANONICAL → CLOSED → BACKEND → OWNED` 单调递增，`XiPassDesc.input_stage` / `output_stage` 强制约束
- **Declarative ordering constraints**：`xi_pass_order_check()` 启动时校验
- **Fixed-point iteration**：`XI_OPT_MAX_ROUNDS = 8`
- **Per-pass stats**：`XiPassStats`（invocation / n_removed / n_added / elapsed_ns）
- **Budget control**：`budget_ns` 到期直接 `goto done`
- **Debug hooks**：
  - `XRAY_XI_SHUFFLE=1` — 随机打乱 blocks/values 顺序，主动暴露 implicit ordering bug
  - `XRAY_XI_CHECK=1` — per-pass verify（定位 invariant 破坏者）
  - `XRAY_XI_DUMP=func:pass` — 定点 IR dump
  - `XRAY_XI_PASS=gvn:dump=1,licm:enable=0` — 单 pass disable/dump
  - `XRAY_XI_STATS=1` — 全 pipeline 统计

### 特色能力（开源罕见）

| 能力 | 说明 |
|---|---|
| Escape-driven stack alloc | `NO_ESCAPE` heap alloc 自动改 `XI_STACK_ALLOC` |
| Escape-driven ARC | 基于 escape lattice 插 retain/release，省去全 GC 扫描 |
| 随机 pass shuffle | `XRAY_XI_SHUFFLE` 检测 implicit ordering assumption |
| Stage-aware verify | 每 stage 自动升级 invariant mask，verify 跟进 |

---

## 业界对标（诚实版）

| 能力 | xray 当前 | LLVM -O2 | HotSpot C2 | V8 TurboFan | Cranelift |
|---|---|---|---|---|---|
| SSA framework (Braun) | ✅ | ✅ | ✅ | ✅ | ✅ |
| Const fold / strength reduce | ✅ | ✅ | ✅ | ✅ | ✅ egraph |
| Copy prop / DCE / phi simp | ✅ | ✅ | ✅ | ✅ | ✅ |
| SCCP | ✅ 完整 | ✅ | ✅ | ✅ | 弱 |
| GVN | ✅ binary only | ✅ GVN-PRE | ✅ PRE-CSE | ✅ | ✅ egraph |
| LICM | ✅ innermost | ✅ alias-aware | ✅ | ✅ | ✅ |
| Inlining | ✅ cost-based | ✅ aggressive | ✅ speculative | ✅ speculative | 中等 |
| IfConv / select | ✅ | ✅ | ✅ | ✅ | ✅ |
| Escape + stack alloc | ✅ 4-level | ✅ capture | ✅ | 弱 | 弱 |
| **Alias analysis / TBAA** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Range / value range** | ❌ | ✅ SCEV | ✅ | ✅ | ✅ |
| **Bounds check elimination** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **PRE / GVN-PRE** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Jump threading** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Devirtualization (CHA)** | 弱 | ✅ | ✅ | ✅ | 中 |
| **Loop unroll / peel / split** | ❌ | ✅ | ✅ | 部分 | 部分 |
| **Induction variable opt** | ❌ | ✅ IndVars | ✅ | 部分 | ❌ |
| **Loop vectorization** | ❌ | ✅ LV + SLP | ✅ SuperWord | ❌ | 规划中 |
| **Speculative type narrow** | 弱 | ❌ | ✅ 核心 | ✅ 核心 | ❌ |
| **Profile-guided opt** | ❌ | ✅ | ✅ | ✅ | ❌ |
| **Partial evaluation** | ❌ | ❌ | ❌ | ❌ | ❌（Graal 有） |

### 当前定位

```text
xray Xi 当前 ≈ LuaJIT IR opt ~ HotSpot C1
              < Cranelift（有 egraph + alias）
              << LLVM -O2 / HotSpot C2 / V8 TurboFan
              <<< Graal（partial evaluation + polymorphic IC）
```

**开源小语言 landscape**：已在顶尖梯队（和 LuaJIT 同档），但**缺"工业级可预期优化"的 5 大基础件**（alias / range / BCE / PRE / speculative），补齐后可直接对标 Cranelift 成熟态 → V8 Maglev。

---

## 目标与非目标

### Goals

本 roadmap 完成后，xray Xi 应具备：

1. **Alias-aware load/store 优化**（LICM/GVN 跨 call 可 hoist 不变 load）
2. **Range-aware 边界检查消除**（数组/字符串访问在可证路径删 BCE）
3. **GVN-PRE 的部分冗余消除**（不仅消除完全冗余，覆盖 partial-redundant）
4. **Jump threading + block simplification**（扁平化 CFG）
5. **Class hierarchy 静态 devirtualization**（单子类时 call 直接化）
6. **循环基础 opt**（unroll / peel / induction var / rotation / split）
7. **Speculative type narrowing + IC 消费**（JIT 特化热路径）
8. **Profile-guided inlining + block layout**（hot path 紧凑）
9. **SLP + 简单 loop vectorization**（数据并行）
10. **Cross-module LTO + partial evaluation for comptime**

### Anti-goals（明确不做）

| 不做 | 理由 |
|---|---|
| **全重写 IR** | `xi.h` 已稳定、`xi_opt` 框架成熟；只加 pass，不换框架 |
| **换 LLVM backend** | 076 的 runtime hardening 不在 LLVM world；加 LLVM 等于丢弃 076 |
| **Graal 级 partial evaluation** | xray 不是 meta-circular；ROI 低 |
| **TurboFan reducer-based framework** | LLVM NewPM / xray 现有 pass table 功能等价；reducer 是过度工程 |
| **多层 IR（HIR/MIR/LIR）** | 已有 Xi（HIR）+ Xm（LIR），两层够用 |
| **egraph-based GVN**（Cranelift 方向） | 实现复杂度极高；经典 GVN-PRE 足够 |
| **全自动并行化** | xray 语言特性已支持显式 go/channel；隐式并行化 ROI 低 |
| **链接时多模块 PGO** | 先做单模块 PGO；LTO PGO 留给 Xi-4 |
| **IR 层 escape 分析的 precision 升级到 interprocedural** | 当前 intraprocedural + conservative callee escape 够用；跨函数 escape summary 工作量超 ROI |

---

## 能力层级总览

| 层 | 主题 | 时间估算 | 做完后对标 |
|---|---|---|---|
| **Xi-0** | 债务清理（068 遗留 + 现有 pass bug 修复） | 4-6 周 | 内部稳定基线 |
| **Xi-1** | 工业级基础 gap 补齐 | 6-9 月 | **Cranelift 成熟态** |
| **Xi-2** | 循环优化 | 6-9 月 | **LLVM -O2 非 vec** |
| **Xi-3** | Speculative + JIT PGO ⭐ | 9-12 月 | **V8 Maglev** |
| **Xi-4** | 向量化 + LTO + partial eval | 12-18 月 | **LLVM -O3 部分 + HotSpot C2** |

**总时间：33-48 月**（约 3-4 年）。Xi-3 完成后已是开源小语言 JIT 顶尖水平，Xi-4 视项目需求决定是否继续。

### 依赖图

```text
Xi-0 (债务清理)
  ↓ 必须先于所有 Xi-1+
Xi-1 (基础 gap)
  ├─ Xi-1.1 TBAA ───────────┐
  │                          ├─→ Xi-1.9 Alias-LICM → Xi-2.x
  ├─ Xi-1.2 Range ─────────┐ │
  │                         ├─→ Xi-1.3 BCE
  ├─ Xi-1.4 GVN-PRE ───────┴─┴─→ Xi-2.x / Xi-3.x
  ├─ Xi-1.5 Jump threading ─────→ Xi-2.x
  ├─ Xi-1.6 Inline fix + heuristic ┐
  ├─ Xi-1.7 CHA devirt ─────────────┼─→ Xi-3.3 spec inline
  └─ Xi-1.10 Verifier 补全 ─────────┘

Xi-2 (循环 opt)
  ├─ Xi-2.1 IndVar canon ──→ Xi-2.4/2.8/Xi-4.2
  ├─ Xi-2.2 Rotation ──────→ Xi-2.3/2.4
  └─ Xi-2.7 Dom cache ─────→ 所有 Xi-2.x

Xi-3 (speculative)
  ├─ Xi-3.1 IC consume ───→ Xi-3.2/3.3/3.4
  ├─ Xi-3.2 Type narrow ──→ Xi-3.5/3.6
  ├─ Xi-3.3 Spec inline ──→ Xi-3.7
  └─ Xi-3.10 Deopt cost ──→ Xi-3.3/3.5

Xi-4 (向量化 + LTO)
  依赖 Xi-1/2/3 全部完成
```

---

# Xi ↔ Xm 接口契约

Xi 与 076 Xm 在 SSA 层和 machine encoding 层各自独立演化，但它们之间的信号集合需要集中规约。所有跨层依赖只能走下表，禁止在源码里临时打洞。

| 信号 | 生产者 | 消费者 | 载体 | 引入阶段 |
|---|---|---|---|---|
| `mem_group`（TBAA 组） | Xi-1.1 | Xi-1.4 GVN-PRE / Xi-1.9 LICM / 076 lowering（可选） | `XiValue.aux_mem` | Xi-1.1 |
| range `[lo, hi]` | Xi-1.2 | Xi-1.3 BCE / Xi-2.x 循环 opt | `XiValue.aux_range` | Xi-1.2 |
| `XiBlock.frequency` | Xi-3.8 / VM 计数 | 076 block layout | `XiBlock.frequency` | Xi-3.8 |
| `XmIcSnapshot` | VM IC 收集 | Xi-3.1-3.10 / 076 PIC stub | `XmTarget.ic_snapshot` | Xi-3.1 |
| `escape level` | `xi_escape` | `xi_arc` / 076 stackmap | `XiValue.escape` | 已有 |
| `XiGuardCost` | Xi-3.10 | Xi-1.6 inline / Xi-3.3 spec inline | `XiValue.aux_guard_cost` | Xi-3.10 |
| `XiFunc.stage` / `invariant_mask` | 全 pipeline | Xi pass entry / 076 lowering 入口 | `XiFunc.stage` | 已有 |
| `XiHelperId` | helper 调用点 | 076 L2 `xisa/xm/helpers.def` | `XI_CALL_BUILTIN.helper_id` | 076 S1 起 |

**契约约束**：

- 上表是 Xi 与 Xm 之间**唯一允许**的耦合点。任何新的跨层信号必须先入表，再写代码
- 信号缺失（如 IC snapshot 在 AOT 路径不存在）→ 消费 pass 必须能**静态退化**，不允许 `XR_UNREACHABLE` 也不允许 silent 错误
- 076 不允许反向向 Xi 写元数据；信号流向单向：Xi → Xm
- 每个表项必须在文档对应章节有详细 schema（`XiValue.aux_mem` 结构、`XmIcSnapshot` 字段等）

076 应该在 §"Xm 入口期望" 处反向引用本表，二者保持一致；任何一边改 schema 都必须同步更新另一边。

---

# JIT 与 AOT fallback 边界

对齐 076 §"JIT fallback 边界"（Xi → Xm eligibility 可拒绝；Xm → machine 不允许 fallback），Xi 内部边界如下：

| 边界 | 是否允许 fallback | 规则 |
|---|---|---|
| Xi pipeline 预算超时（`budget_ns` 到期） | 允许截断 | 已优化结果保留；剩余 pass 跳过；不丢正确性；不丢已分配资源 |
| Xi pass 内部 assert / verify fail | 不允许 | compiler bug，hard fail；不允许 catch-and-ignore |
| Xi pass OOM | 不允许 | hard fail；不允许"跳过该函数继续编译" |
| Xi 新 op 在某 backend 未支持 | 不允许 | 必须在 076 L1 `ops.def` 声明 `:backend-coverage`；未支持的 op 由 Xi → Xm eligibility 拒绝整体 JIT，而非"silent skip" |
| AOT 路径 Xi pass 出错 | 不允许 | AOT 无解释器可回，必须 hard fail；编译失败由 driver 报告 |
| Xi-3.x speculative pass 在无 IC snapshot 时 | 允许静态退化 | 不算错误；pass 内部 fail-safe，不引入 release-level mode flag |
| Xi-3.x runtime deopt | 允许 | 由 076 deopt table 接管；属运行期机制，不属编译期 fallback |
| Xi → Xm lowering 时 op 缺 `:lowering` | 不允许 | 076 L1 typecheck 直接 build fail |

**禁止**：

- 在 Xi pass 内 `if (verify_fail) { rollback_and_skip; }` —— verify fail 即 compiler bug
- 在 Xi pipeline 加 `--xi-best-effort` 之类总开关 —— 让错误"通过"等于无 fallback 纪律
- 用 `XR_DCHECK` 把 Release 路径的契约违反静默 → 必须 `XR_CHECK` / 显式 if + return error

---

# AOT vs JIT 路径分歧

Xi pipeline 同时服务 JIT（带 IC snapshot / runtime profile）与 AOT（无 runtime 信号）。所有 Xi-3.x 速度优化都依赖 IC snapshot，AOT 路径必须有明确退化策略，**不允许 silent skip**，也**不允许在 AOT 跑 speculative pass 然后期望 deopt 回 VM**（AOT 无解释器可回）。

每个 speculative / PGO 类 pass 的 AOT 行为必须落到下表，新增此类 pass 时强制同步：

| Pass | JIT 行为 | AOT 行为 | 说明 |
|---|---|---|---|
| Xi-3.1 IC snapshot 消费 | 读取 `XmTarget.ic_snapshot` 附加到 IR | skip：AOT 无 snapshot | 后续 Xi-3.x 全部静态退化 |
| Xi-3.2 spec type narrow | 基于 mono IC 插 guard + narrow | skip | AOT 走 Xi-1.7 CHA static devirt 替代 |
| Xi-3.3 spec inline | 基于 IC 选 callee | static heuristic only | 退到 Xi-1.6 cost-based inline |
| Xi-3.4 PIC | runtime stub rewrite | skip | AOT 用静态 vtable dispatch |
| Xi-3.5 guard combine | 合并 Xi-3.2 / 3.3 产生的 guard | skip | 没有 guard 可合并 |
| Xi-3.6 guard motion | hoist / sink guard | skip | 同上 |
| Xi-3.7 partial specialize | 基于 hot const-on-branch | offline profile (可选) | 默认 skip；若有 offline profile artifact 可启用 |
| Xi-3.8 block layout | 基于 VM block freq | offline profile (可选) | 默认走静态 chain formation（无频次） |
| Xi-3.9 OSR tier-up | runtime OSR + profile | N/A | AOT 无 OSR 概念 |
| Xi-3.10 deopt cost model | 消费 guard miss rate | skip | AOT guard 不存在 |

**规则**：

- pass 在 AOT 模式下被 skip 时，必须打印 stats（`XRAY_XI_STATS=1` 可见），不允许"消失"
- AOT 路径调用 `xi_pipeline_run_aot()` 入口，该入口强制 skip 上表标记 skip 的 pass
- 上表覆盖所有 Xi-3.x；新增 Xi-3.x 或 Xi-4 PGO 类 pass 时必须同步本表
- 反向不变量：每个 `requires_inv_mask` 含 `XI_INV_IC_ATTACHED` 的 pass，AOT 入口禁止注册

---

# Stage contract 扩展策略

当前 stage 集合：`RAW → CANONICAL → CLOSED → BACKEND → OWNED`，由 `XiPassDesc.input_stage` / `output_stage` 强制单调递增。

**扩展原则**：

- **优先升级 `invariant_mask` bit，不滥增 stage**
  - 新 SSA 性质（如 `XI_INV_TBAA_ANNOTATED` / `XI_INV_RANGE_ANNOTATED` / `XI_INV_GUARD_COST`）作为 invariant bit 加到 `XiFunc.invariant_mask`
  - 每个 pass 在 `XiPassDesc` 上声明 `requires_inv_mask` / `produces_inv_mask`
  - `xi_pass_order_check()` 启动时校验 produce-require 偏序

- **仅当 IR 形态本质变化才扩展 stage**
  - 例如引入 BACKEND-only op、引入跨函数 summary 元数据
  - 新 stage 必须保持单调递增，且每个 pass 显式声明其 `input_stage` / `output_stage`

- **禁止**新增"可跳"stage（如 `OPTIONAL_TBAA`）—— invariant_mask 已经覆盖

**当前规划的 invariant bits**：

| bit | 引入 | 含义 |
|---|---|---|
| `XI_INV_DOM_VALID` | 已有 | 支配树已建好（cfg 未改时为真） |
| `XI_INV_LOOP_INFO` | 已有 | natural loop 已识别 |
| `XI_INV_TBAA_ANNOTATED` | Xi-1.1 | 每 load/store 带 `mem_group` |
| `XI_INV_RANGE_ANNOTATED` | Xi-1.2 | 每整数 value 带 range |
| `XI_INV_MEM_SSA` | Xi-1.1 | memory phi 已建好 |
| `XI_INV_GUARD_COST_FILLED` | Xi-3.10 | 每 guard 带 cost 元数据 |
| `XI_INV_PROFILE_FREQ` | Xi-3.8 | 每 block 带 frequency |
| `XI_INV_IC_ATTACHED` | Xi-3.1 | call-site 带 IC snapshot |

引入新 invariant bit 时必须：
1. 加 enum + 在 `xi_verify` 加对应检查
2. 在 §反向不变量 表格加 B 类条目（`xi_verify` 检查）
3. 所有"requires 该 bit"的 pass 在 pass table 声明
4. 上线该 bit 的 pass 退出前所有现存 IR 满足该 bit

---

# Xi 新 op 加入流程

加入新的 `XI_*` op（含 Xi-3.x 的 `XI_GUARD_*`、Xi-4.x 的 `XI_VEC_*`、Xi-1.8 的 `XI_TAIL_CALL` 等）是 Xi 与 Xm 之间最关键的契约同步点。流程**强制**如下：

```text
单 PR 内必须完成（按顺序）：

1. src/ir/xi.h           ← 加 XI_* enum 值
2. src/ir/xi_op_name.h   ← 加显示名字符串（与 enum 等长）
3. src/ir/xi_op_table.c  ← 加 arity / type contract / side-effect flag
4. src/ir/xi_verify.c    ← 加 op 形态检查（operand 类型、个数、stage 合法性）
5. xisa/xm/ops.def       ← 若 op 进入 backend，加对应 XmOp + lowering（076 L1）
6. xisa/xm/helpers.def   ← 若 op 借助 helper 实现，加 helper id（076 L2）
7. tests/unit/ir/test_<op_name>.c        ← 正面测试
8. tests/unit/ir/test_<op_name>_negative.c ← 非法操作数测试
9. doc comment           ← 在 xi.h enum 旁写 op 语义、side-effect、stage 限制
```

**任一缺失 → CI fail**：

| 缺失项 | 检查方式 |
|---|---|
| 1-4 缺一 | `scripts/check_xi_invariants.sh --op-name-sync --op-table-sync` |
| 5 缺（backend op） | `scripts/check_xi_invariants.sh --xm-ops-sync` |
| 6 缺（用 helper） | 076 L2 typecheck 直接 build fail |
| 7-8 缺 | `scripts/check_xi_invariants.sh --negative-test` |
| 9 缺 | code review 拦截（无自动） |

**禁止**：

- 多 PR 拆分（先加 enum 后加 lowering）→ 中间状态会让 verifier / typecheck fail
- 跳过 helper registry（直接 raw fn pointer）→ 076 反向不变量禁止
- 用 `XI_OP_UNKNOWN` / `XI_OP_PLACEHOLDER` 占位 → 不允许 silent fallback

---

# Pass 表组织

Xi-0 起 `xi_pass_table[]` 为平铺数组。Xi-1+ 完成后会有 30+ 个 pass，组织策略如下：

**目录布局**：

```
src/ir/
├── xi.h, xi_lower.c, xi_emit.c, xi_verify.c, xi_pipeline.c, xi_analysis.c (核心)
├── xi_tbaa.[ch], xi_range.[ch], xi_memssa.[ch] (分析模块)
└── passes/
    ├── cleanup/      ← constfold / strength_reduce / copy_prop / phi_simplify / dce
    ├── memory/       ← gvn_pre / licm / mem_dedup
    ├── control/      ← jump_threading / block_simplify / ifconv / sccp
    ├── loop/         ← loop_rotate / peel / unroll / split / ivsr / inv_branch
    ├── inline/       ← inline / spec_inline / call_specialize
    ├── speculative/  ← spec_narrow / guard_combine / guard_motion / pic / deopt_cost
    └── vec/          ← slp / loop_vec / reduce_pattern (Xi-4 起)
```

**pass table 组织**：

- 顶层 `xi_pass_table[]` 仍为单一权威数组
- 各组在 `src/ir/passes/<group>/passes.inc` 声明本组 `XiPassDesc[]`
- `xi_pass_table.c` 用 `#include "passes/<group>/passes.inc"` 拼装
- 加新 pass：在对应组的 `passes.inc` 加一行；不允许在顶层 `xi_pass_table.c` 直接添加

**禁止**：

- 用户配置文件控制 pass order —— pass 顺序是代码权威
- 多个 pass table 并存（如 `xi_pass_table_legacy[]`）—— 只有一份真相
- pass 跨组放置（如 `loop_rotate` 放到 `control/`）→ 阻挡查阅与依赖追溯

---

# 源码与生成物目录归属

| 路径 | 进 git | 类型 | 备注 |
|---|---|---|---|
| `src/ir/xi*.[ch]` | ✅ | 核心 IR + pipeline | 真相源 |
| `src/ir/passes/<group>/*.c` | ✅ | Pass 实现 | 按组组织 |
| `bench/xi_opt_bench/*.xr` | ✅ | benchmark 源 | 每子项绑定 |
| `docs/bench/xi-N-baseline.json` | ✅ | 性能基线快照 | 每 Xi-N 完成时打 |
| `tests/unit/ir/test_*.c` | ✅ | 单元测试 | 含 `_negative.c` |
| `tests/fuzz/xi_corpus/` | ✅ | fuzz 种子语料 | 可缩可扩 |
| `scripts/check_xi_invariants.sh` | ✅ | CI 反向不变量 | 反向不变量 §15 落地 |
| `scripts/xi_diff.sh` / `xi_bench.sh` / `xi_coverage.sh` | ✅ | 调试工具 | 详 §工具与基础设施 |
| `build/xi_dashboard/` | ❌ | dashboard HTML | 生成物 |
| `build/fuzz_artifacts/` | ❌ | fuzz crash artifact | 生成物 |
| IC snapshot binary（runtime 产物） | ❌ | runtime 计数 | 不持久化 |
| Profile-guided 数据（运行期收集） | ❌ | runtime 产物 | JIT 启动后生成 |

任何新增产物加入工作流前必须先入本表。

---

# 077-A Foundation Complete

077-A 是本方案的最小可交付闭环。完成后可拿到本方案约 60% 的核心收益，且对 076 的依赖仅限 S0/S1。

| 指标 | 目标 |
|---|---|
| Xi-0 全部子项完成 | inline 多返回值修复 + lowering 硬错误 + verifier 升级 + pipeline 预算接通 |
| `xi_verify` 五类新检查上线 | dominance / arity / type / side-effect / backend contract |
| `xi_tbaa.[ch]` + Memory SSA 骨架落位 | lattice + mem_group 元数据齐全；LICM/GVN 暂不消费 |
| Xi-1.4 GVN-PRE 替换 legacy GVN | 单 PR 完成；不留 legacy 副本 |
| Xi-1.10 verifier 全面升级完成 | 所有 SSA 层 invariant 强制 |
| 反向不变量 15 条全部接 CI | `scripts/check_xi_invariants.sh` 上线 |
| Xi ↔ Xm 接口契约表落位 | mem_group / helper_id 信号已通 |
| 阶段坐标禁入源码 | grep 检查 0 命中 |

**推荐启动顺序**（参考 076 §13）：

```text
[078] Xi-0 整体（4-6 周）
        ↓
[079] xi_tbaa + Memory SSA 骨架（2 周）  ← Xi-1.1 框架
        ↓
[080] Xi-1.4 GVN-PRE 替换（2-3 周）       ← 单 PR 删 legacy
        ↓
[081] 反向不变量 CI 落地（1 周）
        ↓
       077-A done, ship.
```

077-A 完成后，Xi-1.2 / Xi-1.3 / Xi-1.5 / Xi-1.6 / Xi-1.7 / Xi-1.8 / Xi-1.9 可并行推进。

---

# 删除清单

最终必须删除或替换（对齐 076 §9）：

| 类别 | 处理 | 触发阶段 |
|---|---|---|
| `xi_opt_gvn.c`（legacy GVN） | 单 PR 删除，由 GVN-PRE 取代 | Xi-1.4 |
| 旧 inline cost model（仅 value count） | 单 PR 替换 | Xi-1.6 |
| `xi_lower.c` 两处 silent default 分支 | 改 `XR_UNREACHABLE` | Xi-0.2 |
| 现有"verifier 仅结构检查"代码 | 升级为语义检查 | Xi-0.3 |
| 068 E9 弃用的 IC plumbing | 删除 | Xi-0 退出前 |
| `XRAY_XI_TBAA` / `XI_ENABLE_TBAA` / `XI_RANGE_ENABLED` / `XI_GVN_MODE` / `XI_INLINE_MODEL` / `XI_LICM_ALIAS` / `XRAY_XI_SPEC` / `XRAY_XI_SPEC_INLINE` | **绝不引入**，反向不变量 §15 静态阻拦 | 全程 |
| `xi_opt_gvn_legacy.c` 副本 | **绝不引入** | Xi-1.4 |
| 任何"上线 30 天后删除 legacy"段落 | **绝不出现** | 全程 |
| 多 pass table 并存 | **绝不出现** | 全程 |

---

# Xi-0: 债务清理

> 在加新 pass 之前必须先修完 068 遗留问题。这是"现有 pass 是否正确"的问题，所有 Xi-1+ 都建立在这之上。

## Xi-0.1 — 修复 inline pass 多返回值误编译

**风险**：中等 — 对多返回值函数的静默误编译。

**问题**：`src/ir/xi_opt_inline.c` 内联 callee 时仅使用第一个返回值。当 `XI_CALL` 支持多返回值且 `XI_EXTRACT` 提取元组元素时，产生不正确的代码。

**实现**：
1. 内联展开时扫描 `XI_EXTRACT(call_result, index)` 使用点
2. 每个 `XI_EXTRACT` 映射到内联体中 `XI_RETURN_MULTI` 终结器的对应返回值
3. 多个 return block 通过 phi 节点合并
4. 替换所有 `XI_EXTRACT` 使用点后删除原始 `XI_CALL`

**测试**：`let a, b = pair()` + `pair` 可 inline；VM/JIT/AOT 三后端一致。

**DoD**：
- [ ] `tests/unit/ir/test_inline_multi_return.c` 新增，覆盖 2/3/4 返回值
- [ ] `ctest` + 回归测试全绿
- [ ] 所有回归 benchmark 无性能回退

**回滚锚点**：单 commit；回滚即回到 inline 仅单返回值。

---

## Xi-0.2 — Lowering 缺省分支改硬错误

**问题**：`xi_lower.c:2163` 不支持表达式发射 null 占位符；`xi_lower.c:2661` 不支持语句静默跳过。两者都产生不正确运行时行为。

**实现**：
1. 两处 default 分支替换为 `XR_UNREACHABLE("unsupported AST node kind %d", node->type)`
2. Release 构建设置 `ctx->error = true` 并返回 `XI_NOP`
3. 复用 `xi_pipeline` 现有 lowering 错误检查

**DoD**：
- [ ] 两处硬错误就位
- [ ] 现有测试全通过（证明 analyzer 接受的节点都可 lower）
- [ ] 新增 debug 构建 assert 触发测试（故意构造非法 AST）

**回滚**：单 commit，替换宏调用。

---

## Xi-0.3 — Verifier 无条件升级为语义验证器

**现状**：`xi_verify.c` 仅做结构检查（函数 / block / value / phi / CFG / type 非 NULL / op 范围）。

**需添加**：

| 检查 | 描述 |
|---|---|
| Dominance | 每个 use 必须被其 def 支配；phi 参数由前驱块支配 |
| Operand arity | 静态表 `uint8_t expected_narg[XI_OP_COUNT]`，与 `nargs` 校验 |
| Type contract | 比较 → bool；`XI_SELECT` 两臂类型兼容；`XI_BOX`/`XI_UNBOX` rep 合法；`XI_CALL` 多返回值 ↔ `XI_EXTRACT` 范围 |
| Side-effect flag | call / throw / store / alloc / safepoint 必须 `XI_FLAG_SIDE_EFFECT` |
| Backend contract | VM-emittable op 子集 / JIT-lowerable 子集 / AOT-cgen 子集 |

**不设门控标志**，全部无条件运行。暴露的 IR 生成器 bug 就地修复。

**DoD**：
- [ ] 5 类新检查全部实现
- [ ] 每类新增至少 3 个非法 IR 测试
- [ ] 现有测试全通过（修完 IR 生成器暴露的 bug 后）

**回滚**：分 5 个 commit（每类一个），任何一个可独立回滚。

---

## Xi-0.4 — Pipeline 预算与统计对外暴露

**现状**：`xi_opt_run_pipeline_ex` 已支持 `XiPipelineStats` 和 `budget_ns`，但：
- JIT Tier 1 调用 `xm_run_pipeline_ex` 时**未传预算**
- 后台编译同样无预算

**实现**：
1. Tier 1 JIT（同步、主线程）：强制 10ms 预算
2. 后台编译：每函数 50ms 预算
3. 预算超时 → 回退解释器；不"保留部分优化代码"

**DoD**：
- [ ] `src/jit/xm_jit.c` / `src/jit/xjit_compile_queue.c` / `src/jit/xm_pass.c` 全部传预算
- [ ] 大函数压力测试无 JIT hang
- [ ] `XRAY_XI_STATS=1` 能打印每函数 per-pass 耗时

**回滚**：单 commit，预算参数置 0。

---

## Xi-0 退出标准

- [ ] 068 Track A 全部完成（A1-A4）
- [ ] 068 Track C1 完成（verifier 升级）
- [ ] 068 Track E9 完成（死 IC plumbing 清理）
- [ ] 现有 10 个 pass 无已知正确性 bug
- [ ] CI shuffle + check + dump 三工具在全回归上无 crash
- [ ] git tag `xi-0-complete`

---

# Xi-1: 工业级基础 gap 补齐

> 目标：补齐 xray Xi 与 LLVM -O2 核心 pass set 的 5 大 gap。做完后 xray Xi ≈ **Cranelift 成熟态**。

## Xi-1.1 — TBAA + Memory SSA

**动机**：当前 LICM / GVN 不消费 memory dependency 信息；`XI_LOAD_FIELD` 在 call 前后重复 load，无法合并。

**设计**：

### TBAA lattice

```text
top (any memory)
  ├── const     (immutable, e.g. string bytes, class descriptor)
  ├── field     ── class.field
  │    └── field.<field_name>   (per-field disjoint)
  ├── array     ── array[*]
  │    └── array.<element_type>
  ├── xr_owned  (heap objects managed by xr_malloc)
  ├── shared    (module-level shared arrays)
  └── tls       (thread-local, per-isolate)
```

两个 memory op 的 `mem_group` 在 TBAA lattice 上无交集 → 保证不 alias。

### Memory SSA

每个 `XI_LOAD_*` / `XI_STORE_*` / `XI_CALL*` 有一个隐式 memory value：
- Store / Call 定义新 memory version
- Load 消费最近 memory version
- Memory phi 在 CFG 汇合处
- Call 的 memory clobber 由 helper registry（076 L2）的 `effect` 字段决定

### 实现

```c
// src/ir/xi_tbaa.h
typedef enum {
    XI_TBAA_TOP = 0,
    XI_TBAA_CONST,
    XI_TBAA_FIELD_BASE,
    XI_TBAA_ARRAY_BASE,
    XI_TBAA_XR_OWNED,
    XI_TBAA_SHARED,
    XI_TBAA_TLS,
    XI_TBAA_FIELD_SPECIFIC,  /* + field_id in aux */
} XiTbaaGroup;

XR_FUNC bool xi_tbaa_may_alias(const XiValue *a, const XiValue *b);

// src/ir/xi_memssa.h
typedef struct XiMemDef {
    XiValue *def_value;     /* the value that defines this mem version */
    uint32_t version;
    struct XiMemDef *prev;  /* incoming mem version */
} XiMemDef;
```

### 依赖

- Xi-0.3 verifier 已能识别 `XI_FLAG_SIDE_EFFECT`
- 076 L2 helper registry 的 `effect` 字段

### DoD

- [ ] `XiTbaaGroup` lattice + `xi_tbaa_may_alias()` 实现
- [ ] 每 load/store op 在 `xi_lower.c` 填 `mem_group`
- [ ] Memory SSA 构建在 `xi_analysis.c` 中实现
- [ ] 100 个 TBAA 单元测试（disjoint / may-alias / must-alias 三类）
- [ ] LICM / GVN 消费 TBAA 后不改变正确性
- [ ] 基线 benchmark 不回退（TBAA 本身只是元数据，性能提升在 Xi-1.9）

### 回滚锚点

- git tag `xi-1.1-start` / `xi-1.1-done`
- 不引入 `XI_ENABLE_TBAA` 编译 flag、不引入 `XRAY_XI_TBAA` 运行期开关
- 出现严重问题 → 回滚到 `xi-1.1-start` 整体撤回，不留双轨

---

## Xi-1.2 — Range analysis + Value range propagation

**动机**：为 BCE / loop opt / branch elimination 提供数值 range 信息。

**设计**：

每个整数 SSA value 附加 `[lo, hi]` 区间（有符号 i64）：
- Constant：`[c, c]`
- Phi：`join(inputs)` on lattice
- Branch：`if (x < 10) { ... }` 分支内 `x.range = intersect(x.range, [-INF, 9])`
- Loop：归纳变量在 Xi-2.1 完成后可推断

### Lattice 结构

```text
⊤ (unknown)
  ↓
[lo, hi]  (non-empty range)
  ↓
⊥ (bottom / provably unreachable)
```

### API

```c
// src/ir/xi_range.h
typedef struct {
    int64_t lo;
    int64_t hi;
    bool is_top;
    bool is_bot;
} XiRange;

XR_FUNC XiRange xi_range_of(const XiValue *v);
XR_FUNC XiRange xi_range_union(XiRange a, XiRange b);
XR_FUNC XiRange xi_range_intersect(XiRange a, XiRange b);
XR_FUNC bool xi_range_known_positive(XiRange r);
XR_FUNC bool xi_range_known_less_than(XiRange r, int64_t k);
```

### 实现要点

- 与 SCCP 合并：SCCP lattice 升级为 `{const, range, top, bot}`
- Branch narrowing 在 if 两臂的 block 入口处应用
- Phi join 使用区间的 union（不做三值 / 多值精化）
- 处理溢出：signed overflow 退到 top（xray 定义有符号溢出为 UB）

### DoD

- [ ] `xi_range.h/.c` 实现
- [ ] SCCP 升级为 range-aware
- [ ] 200 个 range 单元测试
- [ ] 现有 SCCP 回归测试全通过
- [ ] Branch elimination 新增测试（编译期证明 dead branch）

### 回滚

- git tag `xi-1.2-start` / `xi-1.2-done`
- 不引入 `XI_RANGE_ENABLED` 双轨；出现严重问题整体回滚到 `xi-1.2-start`
- SCCP 上线只保留 range-aware 一份实现，不保留 constant-only 路径

---

## Xi-1.3 — Bounds check elimination (BCE)

**动机**：xray 数组 / 字符串 / map 访问都插 `XI_BOUNDS_CHECK`。90% 可证明为真，消除它们是最大的单项性能提升。

**依赖**：Xi-1.2（需要 range）

**设计**：

### 识别可消除的 BCE

```text
XI_BOUNDS_CHECK(idx, len):
  if range(idx).lo >= 0 && range(idx).hi < range(len).lo → 删除
  if range(idx) dominates a prior check with same (idx, len) → 删除
```

### 循环内的 BCE

对归纳变量 `i in 0..n`：
- 进入循环前插 `check(0 <= 0 && n-1 < len)` （pre-loop check）
- 循环内所有 `check(i, len)` 都可删

这是 LLVM `IndVarSimplify + LoopBoundsCheckElim` 的简化版。本阶段只做**非循环**的 BCE，循环内 BCE 依赖 Xi-2.1 完成后再做。

### DoD

- [ ] `xi_opt_bce.c` 实现
- [ ] 非循环 BCE 覆盖率 > 60%（在 `demos/` 样本上）
- [ ] 加入 pass table，`XI_OPT_FULL` 启用
- [ ] 微基准数组访问加速 2-4×
- [ ] 所有 BCE 消除都有对应的 safety proof（verify pass 验证 range 包含关系）

### 回滚

- git tag `xi-1.3-start` / `xi-1.3-done`
- 调试期通用开关：`XRAY_XI_PASS=bce:enable=0`（属调试机制，不构成 release fallback）

---

## Xi-1.4 — GVN-PRE (Partial Redundancy Elimination)

**动机**：当前 GVN 只消除完全冗余（同一支配路径上的重复计算）。很多冗余是部分冗余（某些路径上计算，某些路径上未计算）。

**设计**：

采用经典 Chow-Dasgupta 算法（也是 LLVM GVN-PRE 的基础）：

1. **Anticipability analysis**：对每 block 计算向前可见的表达式集合
2. **Availability analysis**：对每 block 计算已计算的表达式集合
3. **Partial redundancy detection**：`Ant ∩ ¬Avail` = 需要 insert 的点
4. **Insertion**：在 partial-redundant 位置插入计算，然后消除后继的重复

### 实现要点

- 复用 Xi-1.1 的 TBAA（memory load 的 PRE 需要 alias 信息）
- 复用现有 GVN 的表达式 hash（`src/ir/xi_opt_gvn.c` 中的 `xi_expr_hash` / `xi_expr_equal`）
- Critical edge splitting 先做（insert 时避免代码爆炸）

### DoD

- [ ] `xi_opt_gvn_pre.c` 实现
- [ ] 同一 PR 内完全删除 `xi_opt_gvn.c`（不保留 legacy 副本）
- [ ] 200 个 PRE 单元测试
- [ ] 微基准 hot loop 加速 10-30%（典型 PRE 收益）

### 回滚

- git tag `xi-1.4-start` / `xi-1.4-done`
- 不引入 `XI_GVN_MODE` 双轨；出现严重问题整体回滚到 `xi-1.4-start`
- 不保留 `xi_opt_gvn_legacy.c`：上线即唯一实现

---

## Xi-1.5 — Jump threading + block simplification

**动机**：CFG 中有大量可合并的 block 和可 thread 的跳转。`IfConv` 已部分覆盖，但不处理跨 block 场景。

**设计**：

### Jump threading

```text
  block A: if (x == 0) goto B else goto C
  block B: (x 现在为 0) ...; goto D
  block D: if (x == 0) goto E else goto F
                ^^^^^^^^^^ 可 thread：A → B 路径上 D 的 branch 永远 true
```

### Block merge

- 单前驱 + 单后继 block → 合并到前驱
- 空 block（只有 jmp）→ 删除，重定向前驱

### Critical edge splitting

threading 和 merge 反过来可能需要 split（保证 phi 正确）。

### DoD

- [ ] `xi_opt_jump_threading.c` 实现
- [ ] `xi_opt_block_simplify.c` 实现（block merge + empty block elim）
- [ ] 加入 pass table，`XI_OPT_FULL` 启用
- [ ] 现有 IfConv 可以简化（把跨 block 逻辑交给 threading）

### 回滚

- git tag `xi-1.5-start` / `xi-1.5-done`
- 调试期通用开关：`XRAY_XI_PASS=jump_threading:enable=0` / `block_simplify:enable=0`

---

## Xi-1.6 — Inline heuristic 改进

**动机**：
- 068 A2 的多返回值 bug（已在 Xi-0.1 修复）
- 现在 `xi_opt_inline.c` cost model 太粗（仅 value count）
- 没有消费 call-site 信息（hot/cold / PGO）

**设计**：

### 升级 cost model

```c
typedef struct {
    uint32_t value_count;        // 当前已有
    uint32_t call_count;         // callee 内部 call 数（高 → inline 收益低）
    uint32_t branch_count;       // callee 内部 branch 数
    bool has_loop;               // callee 含循环（可能大量展开）
    bool calls_self;             // 递归（禁止 inline）
    bool has_throw;              // 抛异常（inline 会污染 caller EH）
    uint32_t param_uses[16];     // 每参数被使用次数（参数常量可特化）
} XiInlineCostModel;

typedef struct {
    bool is_hot;                 // PGO: hot call-site（Xi-3.8 提供）
    bool is_cold;                // PGO: cold call-site（避免 inline）
    uint8_t type_profile_bits;   // IC snapshot 信息（Xi-3.1）
    bool all_args_const;         // 全常量实参（高收益特化）
} XiInlineCallSiteInfo;

int xi_inline_benefit(const XiInlineCostModel *cost,
                      const XiInlineCallSiteInfo *site);
```

### Inline 决策

```text
benefit = base_score
  - cost.value_count * 1.0
  + const_args * 10      (常量实参奖励)
  + hot_site ? 50 : 0    (hot call-site 奖励)
  - has_loop ? 20 : 0    (循环 callee 惩罚)
  - cost.branch_count * 2
```

### DoD

- [ ] `XiInlineCostModel` / `XiInlineCallSiteInfo` 实现
- [ ] 现有 cost-based 被新模型替代
- [ ] 微基准整体无回退，典型样本加速 5-15%
- [ ] Callee 小于 4 inline per pass 限制可动态化（根据 caller 总大小）

### 回滚

- git tag `xi-1.6-start` / `xi-1.6-done`
- 不引入 `XI_INLINE_MODEL` 双轨；新 cost model 单 PR 替换旧实现

---

## Xi-1.7 — Class Hierarchy Analysis + 静态 devirtualization

**动机**：面向对象代码中每次方法调用都是 vcall；大多数情况下运行时只有一个子类，可直接改 direct call。

**设计**：

### CHA 构建（analyzer 阶段）

```c
// src/frontend/analyzer/xanalyzer_cha.h
typedef struct XaClassHierarchy {
    XrHashMap /* class_id → XaClassNode */ class_map;
    XrHashMap /* method_sig → XaMethodImpls */ method_map;
} XaClassHierarchy;

typedef struct XaMethodImpls {
    XiFunc **impls;  /* all classes implementing this method */
    uint32_t nimpls;
} XaMethodImpls;
```

### Devirtualization 条件

```text
对 XI_CALL_METHOD(obj, method_sig):
  if obj.static_type == T and T has only one subclass impl:
    替换为 XI_CALL(impl_func, ...)
  if obj.type narrowed by recent guard/check to T:
    同上
  else:
    保留 virtual dispatch
```

### DoD

- [ ] `xanalyzer_cha.c` 实现 CHA 构建
- [ ] `xi_opt_devirt.c` 实现 IR 层 devirtualization
- [ ] LSP / LSP index 复用 CHA 结构（`src/app/lsp/` 已有部分 class index）
- [ ] 面向对象微基准（`demos/03-oop/`）加速 2-5×
- [ ] 新类加入后 CHA 失效策略（全 invalidate + 重分析）

### 回滚

- git tag `xi-1.7-start` / `xi-1.7-done`
- 调试期通用开关：`XRAY_XI_PASS=devirt:enable=0`
- CHA 查询失败时保守保留 vcall（这是 pass 内部 fail-safe 语义，不是 release fallback flag）

---

## Xi-1.8 — Tail call optimization

**动机**：xray 支持协程，尾调用优化在异步 / 回调风格代码中收益显著。当前全走 stack push。

**设计**：

### 识别 tail call

```text
XI_CALL(f, args...)
XI_RETURN(call_result)
  → 可尾调用（self or general）
```

### self-tail-call → 循环

```text
fn f(x) { if (x == 0) return 0; return f(x - 1); }
  → lowering 到 Xi 时：while loop，无 frame push
```

### General tail call

- x64: `jmp target` 代替 `call + ret`
- arm64: `br target`
- riscv64: `jr target`

这涉及 frame cleanup：当前 frame 的 locals 必须在 jmp 前全部释放（arc release / stack dealloc）。

### 依赖

- 076 S5/S6 的 x64/arm64 lowering
- 076 L2 helper registry 对 tail call 的 effect 语义

### DoD

- [ ] `xi_opt_tail_call.c` 实现 self-tail-call → loop
- [ ] `XI_TAIL_CALL` op 加入 IR
- [ ] 076 的 `xisa/x64/isa.def` / `xisa/arm64/isa.def` 支持 `tail_jmp` encoding
- [ ] 尾递归微基准（fib / ackermann）无 stack overflow，常数时间
- [ ] ARC release 在 tail call 前正确执行

### 回滚

- git tag `xi-1.8-start` / `xi-1.8-done`
- 调试期通用开关：`XRAY_XI_PASS=tail_call:enable=0`
- `XI_TAIL_CALL` op 一旦进入 `xi.h` enum，076 lowering 必须支持；不允许 lowering 静默退化为 normal call

---

## Xi-1.9 — Alias-aware LICM

**依赖**：Xi-1.1 (TBAA)

**动机**：当前 LICM 不敢 hoist load 跨 call（因为 call 可能改 memory）。接通 TBAA 后，"可证不 alias" 的 load 可以 hoist。

**设计**：

### 扩展 `xi_opt_licm.c` 的 `licm_is_pure`

```c
static bool licm_can_hoist(const XiValue *v, const XiLoop *loop) {
    if (licm_is_pure(v)) return true;

    // Load：如果其 mem_group 在 loop 内所有 store/call 都 disjoint，可 hoist
    if (v->op == XI_LOAD_FIELD || v->op == XI_INDEX_GET) {
        for (each store/call in loop) {
            if (xi_tbaa_may_alias(v, store)) return false;
        }
        return true;
    }

    return false;
}
```

### Hoist 后的语义

- Hoist 的 load 保留其 `mem_group` 元数据
- 后续 pass（GVN-PRE）可进一步合并

### DoD

- [ ] `xi_opt_licm.c` 扩展
- [ ] 30 个 alias-aware 测试（pure / const / field-disjoint / array-disjoint）
- [ ] 循环内含 load + call 的微基准加速 20-50%
- [ ] 正确性测试：call 改 memory 时 load 必须不 hoist

### 回滚

- git tag `xi-1.9-start` / `xi-1.9-done`
- 不引入 `XI_LICM_ALIAS` 双轨；alias-aware LICM 上线即唯一实现

---

## Xi-1.10 — Verifier 全面升级（068 C1 完成）

> 实际上这是 Xi-0.3 的扩展。此处收尾。

**DoD**：
- [ ] 所有 SSA 层 invariant 强制（dominance / arity / type / side-effect / backend contract）
- [ ] 每 Xi-1.* 新 pass 都带 verifier 支持
- [ ] CI grep 反向不变量检查（见下 §invariants）

---

## Xi-1 退出标准

- [ ] 10 个子项全部完成
- [ ] 整体回归测试全绿
- [ ] `bench/` 微基准整体加速 **2-4×**
- [ ] 数值密集 benchmark（`demos/01-basics/` 数值部分）L1 miss 显著下降
- [ ] IR 大小（dumpinsn 数）在 `XI_OPT_FULL` 下平均减少 30-40%
- [ ] `XRAY_XI_STATS=1` 每 pass 平均耗时 < 5ms（单函数）
- [ ] git tag `xi-1-complete`

---

# Xi-2: 循环优化

> 目标：循环密集 benchmark 达到 LLVM -O2 非向量化水平。做完后 xray Xi ≈ **LLVM -O2 non-vec**。

## Xi-2.1 — Induction variable canonicalization

**依赖**：Xi-1.2（range analysis）

**动机**：所有后续循环 opt 都需要识别归纳变量（IV）。

**设计**：

### IV 识别

```text
block preheader:
  i_init = ...
block header (phi):
  i = phi(i_init from preheader, i_next from body)
block body:
  i_next = i + step   (step 必须是 loop-invariant)
  goto header
```

### 分类

| 类 | 形式 | 例 |
|---|---|---|
| Basic IV | `i = i + c` | `for i in 0..n` |
| Derived IV | `j = c1 * i + c2` | `j = 2 * i + 1` |
| Polynomial IV | `j = i * i` | 多项式归纳（不展开） |

### 规范化

所有 basic IV 规范为 `{start, step, trip_count}` 三元组，存储在 `XiLoop` 结构体上。

### DoD

- [ ] `xi_loop.h` 扩展 IV 信息
- [ ] `xi_loop.c` 实现 IV 识别
- [ ] 覆盖 basic + derived，polynomial 识别但不优化
- [ ] 循环相关测试的 IV 识别率 > 90%

---

## Xi-2.2 — Loop rotation

**动机**：很多循环是 `while (cond) { body }` 形式，rotation 改为 `do { body } while (cond)`，好处：
- body 第一次迭代不需要 back-edge
- LICM 的 preheader 更自然
- 后续 unroll / peel 更规则

**设计**：

```text
before:
  preheader → header (cond check)
  header → body | exit
  body → header

after:
  preheader → cond → body | exit
  body → cond (back-edge)
  cond → body | exit
```

### DoD

- [ ] `xi_opt_loop_rotate.c` 实现
- [ ] 所有 `while`-style 循环 rotate
- [ ] 验证 `do-while` / `for-range` 不重复 rotate（幂等）
- [ ] 循环 CFG 更紧凑（block 数减少 ~10%）

---

## Xi-2.3 — Loop peeling

**动机**：循环第一次或最后一次迭代有特殊模式（边界检查、guard 失败等），peel 出来后主体更干净。

**典型场景**：

```text
for i in 0..n:
    check(i, len)      # 第一次迭代必做；后续 check 在 Xi-1.3 消除
    arr[i] = ...
```

Peel 第一次后：
```text
if n > 0:
  check(0, len); arr[0] = ...
  for i in 1..n:
    arr[i] = ...       # 无 check
```

### DoD

- [ ] `xi_opt_loop_peel.c` 实现
- [ ] Cost model：只在 peel 能触发显著 opt 时执行（eg. 消除 check / guard）
- [ ] 覆盖 first-iteration peel + last-iteration peel
- [ ] 微基准数组遍历加速 15-30%

---

## Xi-2.4 — Loop unrolling (full + partial)

**依赖**：Xi-2.1 (IV)

**动机**：小循环全展能消除 branch overhead；大循环部分展能 amortize。

**设计**：

### Full unroll

trip count 可编译期确定 **且** 小于阈值（默认 16）→ 全展。

```text
for i in 0..4: body(i)
  → body(0); body(1); body(2); body(3)
```

### Partial unroll

trip count 大或未知 → 按 factor（默认 4x/8x）展：

```text
for i in 0..n: body(i)
  → for i in 0..(n/4)*4 step 4:
       body(i); body(i+1); body(i+2); body(i+3)
     for i in (n/4)*4..n: body(i)  # residual
```

### Cost model

- Body 小（< 10 values）→ 积极展
- Body 大（> 50 values）→ 保守
- Body 含 call → 保守
- Body 含 loop-invariant branch → 展后 LICM 能消

### DoD

- [ ] `xi_opt_loop_unroll.c` 实现
- [ ] Full unroll 覆盖所有 trip-count-known 循环
- [ ] Partial unroll factor 可配（`XRAY_XI_UNROLL_FACTOR`）
- [ ] 代码大小膨胀 < 20%（所有 benchmark 平均）
- [ ] 典型循环 benchmark 加速 1.5-3×

---

## Xi-2.5 — Loop split

**动机**：多出口 / 含 break 循环可拆为多个单出口循环，便于后续优化。

**设计**：

```text
for i in 0..n:
    if (cond(i)): break
    body(i)
```

拆为：
```text
find k = min{i | cond(i)} or n
for i in 0..k:
    body(i)
```

这需要能预先计算 `k`（static or via range analysis）。动态情况不 split。

### DoD

- [ ] `xi_opt_loop_split.c` 实现
- [ ] 只处理 break-only（不处理 continue / multiple exits）
- [ ] 覆盖率报告：能拆的循环占比 10-20%

---

## Xi-2.6 — Loop-invariant branch hoisting

**动机**：循环内的循环不变分支 hoist 到 preheader，避免每次迭代判断。

**设计**：

```text
for i in 0..n:
    if (invariant_cond):    # 循环外可求值
        body_a(i)
    else:
        body_b(i)
```

变为：
```text
if (invariant_cond):
    for i in 0..n: body_a(i)
else:
    for i in 0..n: body_b(i)
```

代码膨胀大（循环复制），需严格 cost model：
- 分支两侧 body 小 → 允许
- 两侧都大 → 禁止
- 多层循环不变分支 → 只 hoist 最外层

### DoD

- [ ] `xi_opt_loop_invariant_branch.c` 实现
- [ ] Cost budget：展开后总大小 < 原大小的 1.5 倍
- [ ] 微基准多态 dispatch-in-loop 加速 1.5-2×

---

## Xi-2.7 — Sparse dominator cache

**动机**：每个 pass 都重新计算 dominator。O(pass_count × func_size) 累积起来显著。

**设计**：

### 增量 dominator 更新

- 仅当 CFG 实际改变（`XiPassChange.cfg_changed = true`）才重算
- 小的 CFG 修改（block split / merge）可增量 patch，不全重算
- Pass 在 `XiPassDesc.flags` 声明是否改 CFG

### 实现

```c
// src/ir/xi_analysis.h
typedef struct {
    XiDominatorTree dom;
    uint64_t cfg_version;      // 增量标记
    bool is_valid;
} XiDomCache;

XR_FUNC XiDominatorTree *xi_dom_cache_get(XiFunc *f);  // 自动 invalidate
```

### DoD

- [ ] Dominator cache 实现
- [ ] `xi_opt_run_pipeline_ex` 跨 pass 复用 dom
- [ ] Pipeline 耗时 `XRAY_XI_STATS=1` 减少 20-40%

---

## Xi-2.8 — Induction variable strength reduction

**依赖**：Xi-2.1

**动机**：`a * i + b` 中乘法可改为累加。

**设计**：

```text
for i in 0..n:
    x = arr_ptr + i * 8    # 乘 8
```

变为：
```text
ptr = arr_ptr
for i in 0..n:
    x = ptr
    ptr += 8               # 累加代替乘
```

### 覆盖

- 整数乘 IV（常数步长）
- 数组地址计算
- 多项式 IV（只做一阶）

### DoD

- [ ] `xi_opt_ivsr.c` 实现
- [ ] 覆盖率：循环内乘法 IV 至少 80% 转为累加
- [ ] 微基准数组遍历加速 10-20%

---

## Xi-2 退出标准

- [ ] 8 个子项全部完成
- [ ] 循环密集 benchmark 对标 C `-O2` 在 **5× 以内**
- [ ] `bench/vm_microbench.xr` 整体加速 **2-3×** vs Xi-1 完成时
- [ ] 矩阵乘 / 数值方法典型样本加速显著
- [ ] 代码大小膨胀控制在 20% 以内
- [ ] git tag `xi-2-complete`

---

# Xi-3: Speculative + JIT PGO ⭐

> 目标：把 JIT 推到 **LuaJIT trace + V8 Sparkplug** 的 speculative 水平。做完后 xray Xi + JIT ≈ **V8 Maglev 级**（仍低于 TurboFan / FTL）。
> **技术难度最大**的阶段，建议 Xi-1 / Xi-2 稳定后再动。

## Xi-3.1 — IC snapshot 真正消费

**背景**：068 E9 已指出 `XmTarget.ic_snapshot` 被传递但未充分利用。Xi-3 所有 speculative opt 都依赖这条链路打通。

**设计**：

### IC snapshot 结构

```c
// src/jit/xm_ic_snapshot.h
typedef struct {
    uint32_t pc;               // bytecode pc
    uint8_t ic_kind;           // IC_MONO / IC_POLY / IC_MEGA
    uint32_t n_targets;
    struct {
        uint32_t type_id;       // 目标类型
        uint32_t hit_count;     // 观察到的命中次数
        uint32_t method_or_field_id;
    } targets[4];              // 最多 4-way
} XmIcEntry;

typedef struct {
    XmIcEntry *entries;
    uint32_t n_entries;
    uint64_t total_samples;
} XmIcSnapshot;
```

### 消费链路

```text
VM 执行 → IC 收集 → 稳定后 snapshot
  ↓
JIT 编译入口 → 读 snapshot → 附加到 Xi IR 每 call-site
  ↓
Xi-3.2 / Xi-3.3 / Xi-3.4 消费
```

### DoD

- [ ] `XmIcSnapshot` 完整实现（当前部分骨架扩展）
- [ ] VM 层 IC 收集稳定性（多次运行 snapshot 一致）
- [ ] Xi IR 上每 `XI_CALL_METHOD` / `XI_LOAD_FIELD` 携带 IC 元数据
- [ ] Tier2 编译能读取 snapshot（即使暂时不用，先接通）

### 回滚

- git tag `xi-3.1-start` / `xi-3.1-done`
- IC snapshot 缺失/为空 → Xi-3 各 pass 静态退化（pass 内部语义，不算 release fallback flag）

---

## Xi-3.2 — Speculative type narrowing

**依赖**：Xi-3.1

**动机**：当前 `XI_TAGGED` 值在 JIT 中保持 tagged，带来大量 tag 检查和 box/unbox。Speculative narrowing 基于 IC 把 tagged 缩为具体类型，换来专门路径 + guard/deopt。

**设计**：

### Narrowing 模式

```text
IC observed: LOAD_FIELD pc=42 monomorphic target type=Foo
  ↓
JIT Xi IR:
  guard_type(obj, Foo)     # guard：失败 → deopt
  XI_LOAD_FIELD_DIRECT(obj, Foo.field_offset)
```

### Guard 合并

多个相邻 load 对同一 obj 只需一个 guard。

### DoD

- [ ] `xi_opt_spec_narrow.c` 实现
- [ ] Mono IC 的 type narrowing 覆盖率 > 80%
- [ ] Guard 合并：相邻同 obj narrow 只保留一个 guard
- [ ] 正确性：guard 失败 → deopt 路径正确重建解释器状态
- [ ] Mono 场景下 field load 加速 3-8×

### 回滚

- git tag `xi-3.2-start` / `xi-3.2-done`
- 调试期通用开关：`XRAY_XI_PASS=spec_narrow:enable=0`（不引入 `XRAY_XI_SPEC` 总开关）

---

## Xi-3.3 — Speculative inlining

**依赖**：Xi-3.1 / Xi-3.2 / Xi-1.6（inline heuristic）

**动机**：基于 call-site type profile 内联最热 callee；miss 时 deopt。

**设计**：

### Mono call-site

```text
IC: CALL_METHOD pc=88 mono target=Foo.bar
  ↓
JIT Xi IR:
  guard_type(this, Foo)
  <inline Foo.bar body>
```

### Poly call-site（2-4 way）

```text
IC: CALL_METHOD pc=88 poly [Foo.bar:70%, Bar.bar:25%, Baz.bar:5%]
  ↓
JIT Xi IR:
  switch (this.type):
    case Foo: <inline Foo.bar body>
    case Bar: <inline Bar.bar body>
    default: deopt or vcall
```

### Inline budget

- Mono：积极 inline（小 callee 直接展，大 callee 仍然先做 type narrow + direct call）
- Poly：只 inline top-1 或 top-2；其他走 vcall
- Mega（>4 targets）：不 inline，改 PIC

### DoD

- [ ] `xi_opt_spec_inline.c` 实现
- [ ] Mono case 覆盖率 > 60%
- [ ] Poly 2-way case 覆盖率 > 30%
- [ ] OOP 微基准（`demos/03-oop/`）加速 3-10×
- [ ] Deopt 触发后状态重建正确

### 回滚

- git tag `xi-3.3-start` / `xi-3.3-done`
- 调试期通用开关：`XRAY_XI_PASS=spec_inline:enable=0`
- 关闭 spec inline 时 caller 自动走静态 inline，不需要额外 mode flag

---

## Xi-3.4 — Polymorphic inline cache (runtime)

**依赖**：Xi-3.1

**动机**：Mega 场景（> 4 targets）无法静态 inline，用 PIC 在运行时 dispatch。

**设计**：

### PIC stub（JIT 生成）

```text
Mega call-site：
  entry:
    t = load obj.type
    cmp t, cached_type_1
    je target_1
    cmp t, cached_type_2
    je target_2
    ... (up to 8 cached entries)
    jmp vcall_fallback
```

### PIC 维护

- 每次 miss 触发 stub rewrite（加新 entry）
- 超过上限 → 转 megamorphic（纯 vcall）
- GC 回收 stale type 时清空 PIC

### DoD

- [ ] PIC stub 生成在 076 L3（machine encoding）层
- [ ] PIC 维护逻辑在 `src/jit/xm_pic.c`
- [ ] Mega 场景加速 1.5-3×（即使是 PIC，比纯 vcall 快）
- [ ] PIC 的线程安全（后台 JIT 改写 + 主线程执行）

### 回滚

- git tag `xi-3.4-start` / `xi-3.4-done`
- 调试期通用开关：`XRAY_XI_PASS=pic:enable=0`
- PIC 关闭时 mega call-site 直接走 vcall（pass 内部 fail-safe，非 release flag）

---

## Xi-3.5 — Guard strengthening / weakening

**依赖**：Xi-3.2

**动机**：多个小 guard 合并为一个强 guard，或拆为多个轻量 guard，视场景选择。

**设计**：

### Strengthening

```text
guard_not_null(x)
guard_type(x, Foo)
  → guard_not_null_and_type(x, Foo)  # 一次检查
```

### Weakening（when deopt is expensive）

```text
guard_int_in_range(i, 0, 100)
  → guard_int_nonneg(i) + guard_int_lt(i, 100)
```

在某些情况下拆分可让 LICM 分别 hoist。

### DoD

- [ ] `xi_opt_guard_combine.c` 实现
- [ ] Guard 密度降低 20-40%
- [ ] 正确性：合并 / 拆分不改变 guard 语义

### 回滚

- git tag `xi-3.5-start` / `xi-3.5-done`

---

## Xi-3.6 — Guard sinking + hoisting

**依赖**：Xi-3.2

**动机**：guard 可以移到 cost 更低的位置：
- **Sink 到 cold path**：很少执行的分支不需要 guard，hoist 出去浪费
- **Hoist 到 loop 外**：循环不变量的 guard 移到 preheader，只查一次

**设计**：

### Hoist

Loop 内 `guard_type(this, Foo)` 且 `this` 在循环内不变 → hoist 到 preheader。

### Sink

```text
if (rare_path):
    guard_type(x, Foo)   # 只在 rare path 检
    ...
```

### DoD

- [ ] `xi_opt_guard_motion.c` 实现
- [ ] Guard 在 hot loop 内频次降低 > 50%
- [ ] 正确性：sink 后 fast path 不可漏 guard

### 回滚

- git tag `xi-3.6-start` / `xi-3.6-done`

---

## Xi-3.7 — Partial specialization (constant-on-branch)

**依赖**：Xi-3.3

**动机**：某些参数在 hot path 上总是常量，可生成特化版本。

**设计**：

```text
fn process(x: int, mode: string):
    if (mode == "fast"): fast_path(x)
    else: slow_path(x)

# PGO 发现 mode == "fast" 占 95%
  → 生成特化 process_fast(x)，直接 inline fast_path
  → 非特化路径保留 slow
```

Graal Truffle 做到了极致（partial evaluation）；xray Xi-3 只做"常量参数特化"。

### DoD

- [ ] `xi_opt_spec_const.c` 实现
- [ ] 覆盖 string / enum / bool 常量参数
- [ ] 特化版本数量上限（避免代码爆炸）
- [ ] 典型场景加速 2-5×

### 回滚

- git tag `xi-3.7-start` / `xi-3.7-done`

---

## Xi-3.8 — Profile-guided block layout

**依赖**：Xi-3.1 的 PGO 链路

**动机**：hot path 连续 block 放一起，cold path 放后面；减少 icache miss、提升分支预测。

**设计**：

### Block frequency 收集

- VM 层计数每 bytecode pc 执行次数
- JIT 编译时读取，标注每 Xi block 的 `frequency`
- Block 排布时 hot-first

### Chain formation

类似 Pettis-Hansen：把频次相近且有 edge 的 block 串联。

### DoD

- [ ] Block frequency 收集（VM 侧）
- [ ] `xi_block_layout.c` 实现
- [ ] 076 lowering 遵循 Xi 指定顺序
- [ ] icache miss 在大 benchmark 上下降 10-20%

### 回滚

- git tag `xi-3.8-start` / `xi-3.8-done`

---

## Xi-3.9 — OSR tier-up with feedback

**依赖**：Xi-3.1

**动机**：当前 OSR 只是"从解释器跳到 JIT"，不带 profile。改进后带上运行时 profile 作为 speculative opt 输入。

**设计**：

### 扩展 OSR 入口

```text
OSR entry 不仅传递 slot map，还传：
  - 最近 N 次该函数的 type profile
  - 当前 loop 的 iteration count 估计
  ↓
JIT Xi-3.2 / 3.3 基于这些特化
```

### DoD

- [ ] OSR entry 扩展
- [ ] 076 S7 的 OSR spec 配合更新
- [ ] 长循环 OSR 后加速 2-5× vs 无 profile

### 回滚

- git tag `xi-3.9-start` / `xi-3.9-done`

---

## Xi-3.10 — Deopt cost model

**依赖**：Xi-3.2

**动机**：inline / specialize 决策需要考虑 deopt 概率和成本。

**设计**：

### 每 guard 标注

```c
typedef struct {
    float observed_miss_rate;    // VM 侧观察到的 miss 率
    uint32_t estimated_deopt_cost;  // deopt 重建 + 解释器恢复时间
    uint32_t caller_frequency;   // 该 guard 所在 path 执行频次
} XiGuardCost;
```

### Inline 决策消费

```text
inline benefit = base_benefit
  - (miss_rate × deopt_cost × frequency)
```

### DoD

- [ ] Guard cost 模型实现
- [ ] Inline / specialize 决策消费
- [ ] 统计：deopt 触发频率 < 1%（稳定 pattern 上）

### 回滚

- git tag `xi-3.10-start` / `xi-3.10-done`

---

## Xi-3 退出标准

- [ ] 10 个子项全部完成
- [ ] JIT benchmark 对标 V8 Maglev / JSC DFG
- [ ] Type-specialized path 比 VM 快 **10-50×**（当前 JIT 多在 2-5×）
- [ ] Deopt 频率在长期 stable pattern 上 < 1%
- [ ] OOP / 动态分派密集代码加速 5-15×
- [ ] git tag `xi-3-complete`

---

# Xi-4: 向量化 + LTO + partial eval

> 目标：数据并行与跨模块优化达到 LLVM -O3 / HotSpot C2 水平。可选阶段，视项目需求决定是否做。

## Xi-4.1 — SLP vectorization

**动机**：直接数据并行（straight-line program）的 SIMD 化。

**设计**：

```text
a[0] = b[0] + c[0]
a[1] = b[1] + c[1]
a[2] = b[2] + c[2]
a[3] = b[3] + c[3]
  → vld b[0..3]; vld c[0..3]; vadd; vst a[0..3]
```

### 实现

- 识别可 pack 的 scalar op（相同 op、邻接地址）
- 插入 `XI_VEC_LOAD` / `XI_VEC_OP` / `XI_VEC_STORE` 新 IR ops
- 076 的 `xisa/x64/isa.def` 加 SSE/AVX encoding

### DoD

- [ ] `xi_opt_slp.c` 实现
- [ ] 076 支持 SIMD encoding（新的 S11 阶段）
- [ ] 数值循环加速 2-4×

---

## Xi-4.2 — Simple loop vectorization

**依赖**：Xi-4.1 / Xi-2.4 (unroll)

**动机**：规则循环的自动 vec。

**设计**：

```text
for i in 0..n:
    a[i] = b[i] + c[i]
  → vectorize with VF=4:
     for i in 0..(n/4)*4 step 4:
         va = vld b[i..i+3]
         vc = vld c[i..i+3]
         vst a[i..i+3], va + vc
     for i in (n/4)*4..n: a[i] = b[i] + c[i]  # residual
```

### 限制

- 无 loop-carry dependency
- 无内部 branch（或只有 uniform branch）
- 无 call
- Stride 1 或 编译期已知 stride

### DoD

- [ ] `xi_opt_loop_vec.c` 实现
- [ ] 覆盖 30% 的规则循环
- [ ] 规则数值循环加速 2-4×

---

## Xi-4.3 — Auto-vec cost model

**动机**：并非所有循环 vec 都有收益（内存带宽受限、short-trip 循环等）。

**DoD**：
- [ ] 基于 trip count、body size、memory pattern 的 cost model
- [ ] False-positive vec 率 < 10%

---

## Xi-4.4 — Reduction pattern recognition

**动机**：`sum` / `product` / `min` / `max` 循环可 vec + horizontal reduce。

**DoD**：
- [ ] 识别 4 种经典 reduction 模式
- [ ] Reduction 循环加速 2-4×

---

## Xi-4.5 — Cross-module LTO

**动机**：当前 inline / devirt 只在单 module 内做；跨 module 的热点无法优化。

**设计**：

- AOT 阶段合并所有 module 的 Xi IR
- 跨 module inline / devirt
- 类似 LLVM ThinLTO 的 summary-based 决策

### DoD

- [ ] `src/aot/xi_lto.c` 实现
- [ ] `demos/02-collections/` 跨模块调用加速 10-30%
- [ ] 编译时间控制在合理范围（< 2× 单 module）

---

## Xi-4.6 — Partial evaluation for comptime (071)

**依赖**：071 comptime 任务

**动机**：071 规划的 `@comptime` 特性需要 IR 层 partial evaluation 支持。

**设计**：

- `@comptime fn` 的参数为常量时完全求值
- 求值期间复用 Xi 优化管道（SCCP + constfold + inline + range）
- 求值上限（递归深度 / 时间 / 内存）

### DoD

- [ ] 071 规划的 comptime 基础能力在 Xi 层落地
- [ ] 编译期 factorial / fib 能求值
- [ ] 与 AST macro 的边界清晰

---

## Xi-4.7 — Constant specialization at call boundary

**动机**：Graal 风格的"常量参数 → 函数特化 + 缓存"。

**设计**：

- PGO 发现某 call-site 的某参数 90%+ 是特定常量
- 生成特化版本，call-site 用特化版
- 特化版本数量上限（避免代码爆炸）

### DoD

- [ ] `xi_opt_call_specialize.c` 实现
- [ ] 典型场景加速 2-3×
- [ ] 特化数 < 原函数数的 2 倍

---

## Xi-4.8 — Cross-function escape summary

**动机**：当前 escape 分析是 intraprocedural（callee 被保守视为 HEAP_ESCAPE）。跨函数 summary 可大幅减少不必要的堆分配。

**设计**：

- 每 callee 计算 escape summary：`{参数 → escape level}`
- Caller 在 call-site 根据 summary 传导
- Summary 可序列化为 IR metadata

### DoD

- [ ] `xi_escape.c` 扩展支持 summary
- [ ] Stack alloc 覆盖率提升 30-50%
- [ ] ARC 操作数量下降 20-30%

---

## Xi-4 退出标准

- [ ] 8 个子项全部完成
- [ ] 数值密集 benchmark 达到 C `-O2` 级
- [ ] SIMD-able 循环加速 **2-4×**
- [ ] Cross-module hot path 加速 10-30%
- [ ] comptime 基础能力就绪（与 071 对齐）
- [ ] git tag `xi-4-complete`

---

# 反向不变量

类似 076 的反向不变量，以下模式不得在 Xi IR 层代码 / IR 实例中出现。

- **类型 A — CI grep**：源码级静态检查，统一接入 `scripts/check_xi_invariants.sh`，CI fail-fast
- **类型 B — verify pass**：IR 实例级动态检查，`xi_verify` 跑（每 pass 之间，`XRAY_XI_CHECK=1` 时每 round 跑）
- **类型 C — pass table validator**：启动时 `xi_pass_order_check()` 跑，违反直接 abort

| # | 类型 | 禁止模式 | 检查方式 | 理由 |
|---|---|---|---|---|
| 1 | A | 新 pass 绕过 pass table 直接修改 IR | `rg -n '^(static\s+)?(int\|void\|bool)\s+xi_opt_[a-z_]+\s*\(' src/ir/passes/` 输出每个函数名必须在 `xi_pass_table[]` 中出现 | 所有 pass 必须注册 |
| 2 | A | 新 IR op 未进 `xi_op_name.h` | `scripts/check_xi_invariants.sh --op-name-sync`（diff `xi.h` 的 `XI_*` enum 与 `xi_op_name.h` 数组长度 + 字符串） | 单一真相源 |
| 3 | A | 新 IR op 未进 076 L1 `xisa/xm/ops.def` | `scripts/check_xi_invariants.sh --xm-ops-sync`（每个 backend-emittable `XI_*` 需有对应 `XmOp` lowering） | Xi/Xm 跨层同步 |
| 4 | B | Pass 改 CFG 但未声明 `cfg_changed` | `xi_verify` 在每 pass 后对比 `block_id_hash`，变化但 `change.cfg_changed==false` → fail | Fixed-point 正确性 |
| 5 | A | 使用 `malloc/calloc/realloc/free` | `rg -n '\b(malloc\|calloc\|realloc\|free)\s*\(' src/ir/ stdlib/` 应为 0 | 统一 `xr_malloc/xr_free` |
| 6 | B | SSA value 被多次定义 | `xi_verify` def-count 检查 | SSA 不变量 |
| 7 | C | Stage 倒退（output_stage < input_stage） | `xi_pass_order_check()` 启动校验 | Stage 单调 |
| 8 | B | Pass 宣称 pure 却含 side effect | `xi_verify` 检查 `XI_FLAG_SIDE_EFFECT` 与 op 表 | Escape / ARC 正确性 |
| 9 | B | TBAA 组标注缺失（Xi-1.1+ 之后） | `xi_verify` 检查每 load/store 必带 `mem_group` | alias-aware pass 前提 |
| 10 | A | 新 pass 未加 pipeline 统计 slot | `scripts/check_xi_invariants.sh --stats-sync`（pass table 条目 ↔ `XiPassStats` 槽位一致） | 可观测 |
| 11 | A | 新 pass 无 `XRAY_XI_PASS=name:enable=0` 可关 | `rg -n 'xi_pass_config_register' src/ir/` 覆盖所有 pass name | 调试需要 |
| 12 | A | 新 pass 无对应负面测试 | `scripts/check_xi_invariants.sh --negative-test`（每 `xi_opt_X.c` 必须有 `tests/unit/ir/test_X_negative.c`） | 测试纪律 |
| 13 | A | Pass 函数超过 150 行 | `scripts/check_function_length.sh src/ir/passes/` | C 代码规范 |
| 14 | A | 源码 / commit 出现 `Xi-N.M` / `Phase` 阶段坐标 | `rg -n 'Xi-[0-9]+\.[0-9]+\|^\s*\* Phase\|^\s*// Phase' src/ stdlib/ include/` 应为 0 | 阶段坐标禁入源码 |
| 15 | A | 源码出现 fallback mode flag（如 `XI_GVN_MODE` / `XI_INLINE_MODEL` / `XRAY_XI_SPEC` 等） | `rg -n 'XI_[A-Z]+_MODE\|XI_[A-Z]+_MODEL\|XRAY_XI_SPEC[^=]' src/ir/` 应为 0 | 无 dual-mode 兼容层 |

新增 invariant 必须同步：
1. 加入本表（含类型、grep 命令、理由）
2. 在 `scripts/check_xi_invariants.sh` 加对应 case（A 类）或 `xi_verify` 加 hook（B 类）
3. 引入该 invariant 的 pass 退出前所有现存 IR 必须满足

---

# 度量与退出

## 基线基准

微基准入口：

- `bench/vm_microbench.xr` / `bench/vm_invoke_microbench.xr`（既有）
- `bench/xi_opt_bench/`（新增）— 每子项绑定 1-3 个 `.xr` 文件

命名约定：`bench/xi_opt_bench/<group>_<scenario>.xr`，例如：

- `bench/xi_opt_bench/mem_field_load_in_call.xr`（Xi-1.1 / Xi-1.9）
- `bench/xi_opt_bench/array_sum_bounds.xr`（Xi-1.3 BCE）
- `bench/xi_opt_bench/gvn_pre_partial.xr`（Xi-1.4）
- `bench/xi_opt_bench/oop_dispatch_mono.xr`（Xi-1.7 / Xi-3.3）
- `bench/xi_opt_bench/loop_unroll_inner.xr`（Xi-2.4）
- `bench/xi_opt_bench/spec_narrow_field.xr`（Xi-3.2）
- `bench/xi_opt_bench/slp_vec_add.xr`（Xi-4.1）

每个 Xi-N 完成后冻结基线快照：

```
docs/bench/xi-0-baseline.json
docs/bench/xi-1-baseline.json
docs/bench/xi-2-baseline.json
...
```

格式：每条目记录 `(file, scenario, build_mode, arch, mean_ns, stddev_ns, samples, git_sha)`。

## 关键指标（每条绑定具体 benchmark）

禁止使用「整体加速 N×」这种无法验证的表述。退出标准必须落到具体 `.xr` 文件 + 具体硬件 + 具体 build mode 的耗时区间。

下表是每个 Xi-N 退出时**必须满足**的硬指标（macOS arm64 Release / 同一台机器）：

| Xi-N | benchmark | baseline (xi-(N-1) done) | 退出要求 |
|---|---|---|---|
| Xi-0 | 全部回归 | — | ctest 113/113 不退；`XRAY_XI_CHECK=1` + `XRAY_XI_SHUFFLE=1` 10k 次无 crash |
| Xi-1 | `mem_field_load_in_call.xr` | T0 | ≤ 0.5 × T0 |
| Xi-1 | `array_sum_bounds.xr` | T0 | ≤ 0.4 × T0（BCE 主要收益） |
| Xi-1 | `gvn_pre_partial.xr` | T0 | ≤ 0.7 × T0 |
| Xi-1 | `oop_dispatch_mono.xr` | T0 | ≤ 0.4 × T0（CHA devirt） |
| Xi-2 | `loop_unroll_inner.xr` | T1（xi-1 done） | ≤ 0.5 × T1 |
| Xi-2 | `matrix_mul_naive.xr` | T1 | ≤ 0.6 × T1 |
| Xi-3 | `spec_narrow_field.xr` | T2（xi-2 done） | ≤ 0.2 × T2（type spec 主要收益） |
| Xi-3 | `oop_dispatch_poly.xr` | T2 | ≤ 0.4 × T2 |
| Xi-3 | deopt 频率（长期 stable pattern） | — | < 1% |
| Xi-4 | `slp_vec_add.xr` | T3 | ≤ 0.4 × T3（SLP 收益） |
| Xi-4 | `lto_cross_module.xr` | T3 | ≤ 0.8 × T3（LTO 收益） |

硬件配置变更时（如换机器），整组基线必须重打，并在 `docs/bench/README.md` 留下变更说明。

## 辅助指标

以下是观察指标，不构成硬退出标准：

| 指标 | Xi-0 | Xi-1 | Xi-2 | Xi-3 | Xi-4 |
|---|---|---|---|---|---|
| Pipeline 平均耗时（单函数） | < 10ms | < 20ms | < 30ms | < 40ms | < 50ms |
| IR 大小压缩率（vs canon 后） | 10-20% | 30-40% | 35-45% | 40-50% | 45-55% |
| 代码膨胀（unroll/inline 后 vs 前） | < 10% | < 20% | < 30% | < 40% | < 50% |

## 每子项 DoD 公共条目

每个 Xi-N.M 完成时必须：

- [ ] 单元测试 + 回归测试全绿
- [ ] `XRAY_XI_CHECK=1` 全程无 verify fail
- [ ] `XRAY_XI_SHUFFLE=1` 全程无 crash（10000 次随机种子）
- [ ] `scripts/check_xi_invariants.sh` 全过
- [ ] 绑定 benchmark 达到子项 DoD 中声明的耗时阈值
- [ ] git tag `xi-N.M-done`

---

# 工具与基础设施

## 必备工具

### 1. IR snapshot diff 工具

```bash
scripts/xi_diff.sh <func> <pass>
# 输出 pass 前后 IR 的 diff
```

基于现有 `XRAY_XI_DUMP=func:pass` 扩展。

### 2. Benchmark harness

```bash
scripts/xi_bench.sh <baseline-tag> <current-tag>
# 对比两个 git tag 的 benchmark 结果
```

### 3. Pass 覆盖率工具

```bash
scripts/xi_coverage.sh <test-corpus>
# 统计每 pass 在 corpus 上的触发频次、删除/添加 value 数
```

### 4. Fuzz harness（基于现有 `tests/fuzz/`）

- 生成随机 Xi IR（合法但复杂）
- 跑 pipeline
- 验证：`XRAY_XI_CHECK` 不 fail + 输出等价

### 5. Dashboard（可选）

把 `XRAY_XI_STATS=1` 输出聚合成 HTML dashboard，展示：
- 每 pass 在各 benchmark 上的耗时分布
- 每 pass 的 value 删除/添加数量
- 收敛轮数分布

---

# 与其他 task 的关系

| task | 关系 | 要点 |
|---|---|---|
| **068** (pipeline optimization) | **Xi-0 的债务来源** | 068 A1-A4 / C1 / E9 在 Xi-0 内完成；068 B1-B5 不属 077 范围 |
| **066** (AOT migration) | **Xi-4.5 LTO 的场地** | 066 定义 AOT 管道，Xi-4.5 在其中插入跨模块优化 |
| **071** (comptime) | **Xi-4.6 partial eval 的客户** | 071 定义 `@comptime` 语法，Xi-4.6 提供求值引擎 |
| **072** (JIT method call / tag fix) | **Xi-0.1 的 1120 regex 源头** | 072 修复 JIT method call contract，是 Xi-0.1 inline 修复的前提 |
| **076** (codegen master) | **Xi 的下游** | Xi 输出的 SSA 进入 076 的 L1 lowering；Xi-3.8 block layout 被 076 遵循 |

---

# 常见误区 / 反模式

## 1. 不要为了"看起来工业级"而做 pass

**反例**：实现 polyhedral loop opt 只因 LLVM 有。xray 的主要 workload 不是 HPC，ROI 低。

**正例**：先看 `demos/` + `bench/` 哪些真的慢，优先优化实际 workload。

## 2. 不要在 Xi-3 前追 speculative

**反例**：Xi-1.6 就做 speculative inline。

**正例**：Xi-1.6 是静态 cost model + PGO-ready 接口；speculative 在 Xi-3.3 基于 IC snapshot 做。

## 3. 不要在 Xi-1.1 (TBAA) 前写 alias-aware pass

**反例**：LICM 直接写一个自定义 alias check。

**正例**：TBAA 是共享基础设施，LICM/GVN/GVN-PRE/Loop-LICM 都复用。

## 4. 不要引入"全局 pass order 可配置"

**反例**：让用户通过配置文件自定义 pass 顺序。

**正例**：pass table 是权威，只在 `XiPassOrderConstraint` 加约束；顺序调整走代码 review。

## 5. 不要在 Xi-3 完成前追 Xi-4

**反例**：先做 vectorization 因为"加速效果立竿见影"。

**正例**：speculative opt 的收益通常比 vec 更大（10× vs 2×），且 xray workload 数值密集场景少。

## 6. 不要为 legacy 保留 "fallback 入口"

**反例**：每个新 pass 都保留旧版本作为 fallback；引入 `XI_GVN_MODE=legacy|pre` 等 mode flag。

**正例**：新实现单 PR 替换旧实现，旧文件同 PR 删除，绝不保留 `*_legacy.c` 副本。`xray` 没有外部用户，没有 "灰度上线" 需求。反向不变量 §15 静态阻拦。

## 7. 不要做"假 speculative"

**反例**：加 guard 但 deopt 路径没测过。

**正例**：每个 Xi-3 子项必须有 deopt trigger 测试（故意触发 deopt）。

---

# 风险矩阵

| # | 风险 | 影响 | 可能性 | 缓解 | 回滚锚点 |
|---|---|---|---|---|---|
| R1 | Xi-1.1 TBAA 模型选错 | 高 | 中 | 先做 lattice 设计 + ref（LLVM TBAA）；信号缺失时 must-alias 保守 | `xi-1.1-start` |
| R2 | Xi-1.4 GVN-PRE 代码爆炸 | 中 | 中 | 严格 critical edge split + insertion cost model | `xi-1.4-start` |
| R3 | Xi-2.4 unroll 代码膨胀失控 | 中 | 中 | factor 上限 + 总函数大小硬上限 | `xi-2.4-start` |
| R4 | Xi-3.1 IC 链路不稳定（race） | 高 | 中 | VM IC 内存序严格 acq/rel；snapshot immutable；TSAN 覆盖 | `xi-3.1-start` |
| R5 | Xi-3.3 speculative inline 频繁 deopt | 高 | 中 | Xi-3.10 deopt cost model 必须先就位 | `xi-3.3-start` |
| R6 | Xi-3.4 PIC 线程安全 bug | 高 | 中 | PIC stub rewrite 必须 atomic（单 cacheline）+ JIT debug tool | `xi-3.4-start` |
| R7 | Xi-4.1 SLP 误 pack 非等价 op | 高 | 低 | 严格 op / type / mem_group 匹配 + golden test | `xi-4.1-start` |
| R8 | Xi-4.5 LTO 编译时间爆炸 | 中 | 中 | Summary-based 决策 + 增量 LTO 缓存 | `xi-4.5-start` |
| R9 | 新 op 加入流程被绕过（先 enum 后 lowering） | 高 | 中 | §Xi 新 op 加入流程 + `scripts/check_xi_invariants.sh --op-name-sync --xm-ops-sync` CI 拦 | `xi-N.M-start` |
| R10 | 团队倾向引入 fallback flag 维持新旧并存 | 高 | 高 | 反向不变量 §15 静态阻拦；code review 拒绝 `XI_*_MODE` / `XI_*_MODEL` 等命名 | 全程 |
| R11 | `Xi-N.M` 阶段坐标渗入源码 / commit | 中 | 高 | 反向不变量 §14 grep；`scripts/check_comment_rules.sh` 接入 | 全程 |
| R12 | benchmark 基线被无意 corrupt（机器换 / build mode 换） | 中 | 中 | 整组重打 + 在 `docs/bench/README.md` 记录变更 | 全程 |

每条风险绑定一个回滚锚点 git tag。触发该风险时回滚到对应 tag，root cause 复盘后再前进，不允许跳过。

---

# 完成标准（全 roadmap）

- [ ] Xi-0 全部完成（债务清零）
- [ ] Xi-1 全部完成（工业级基础）
- [ ] Xi-2 全部完成（循环优化）
- [ ] Xi-3 全部完成（speculative + PGO）
- [ ] Xi-4 视需求完成（向量化 + LTO + partial eval）
- [ ] 所有反向不变量 CI 强制
- [ ] 所有度量指标达标
- [ ] 所有 git tag 齐全（`xi-N.M-done` / `xi-N-complete`）
- [ ] `docs/bench/` 有完整基线快照
- [ ] 与 076 的边界清晰（Xi → Xm 入口契约）

---

# 附录 A：077-A 配套首批 PR

给出 077-A Foundation Complete 闭环的可执行 PR 列表，与 076 §13 的推荐启动顺序对齐。说明的是「做什么」，不是「什么时候做」。

## PR-A：Xi-0.1 + Xi-0.2（inline 多返回 + lowering 硬错误）

- 修 `xi_opt_inline` 多返回值（扫描 `XI_EXTRACT`，phi 合并 multi-return）
- `xi_lower` 两处 silent default 改 `XR_UNREACHABLE`
- 加 `tests/unit/ir/test_inline_multi_return.c`
- 不改 pass table 框架
- 估算约 500 行改动

## PR-B：Xi-0.3（verifier 升级）

- `xi_verify` 五类新检查（dominance / arity / type / side-effect / backend contract）
- 修所有现有 IR 生成器在升级后暴露的 bug（root cause 修复，不绕过）
- 加非法 IR 负面测试（每类 ≥ 3）
- 不改 pass table
- 估算约 1500-2500 行（含 bug fix）

## PR-C：Xi-0.4（pipeline 预算接通）

- JIT Tier1 / 后台编译入口传 `budget_ns`
- `XRAY_XI_STATS=1` 打通到 per-pass 耗时
- 估算约 200 行改动

## PR-D：Xi-1.1 框架（TBAA lattice + Memory SSA 骨架）

- `xi_tbaa.[ch]` 实现 lattice
- `xi_memssa.[ch]` 实现 Memory SSA 构建
- 每 load/store 在 lowering 填 `mem_group`
- LICM / GVN 暂不消费（先接入，PR-F 消费）
- 100 个 TBAA 单元测试
- 估算约 2000-3000 行

## PR-E：Xi-1.10（verifier 全面升级 + invariant_mask）

- 加 `XI_INV_TBAA_ANNOTATED` / `XI_INV_MEM_SSA` 等 invariant bit
- `XiPassDesc` 加 `requires_inv_mask` / `produces_inv_mask`
- `xi_pass_order_check()` 校验 produce-require 偏序
- 估算约 600 行改动

## PR-F：Xi-1.4（GVN-PRE 替换 legacy GVN）

- 新实现 `xi_opt_gvn_pre`
- **同 PR 内删除** `xi_opt_gvn.c`
- 不保留 `_legacy.c` 副本
- 消费 PR-D 的 TBAA
- 200 个 PRE 单元测试
- 估算约 2500-3500 行（含删除）

## PR-G：反向不变量 CI 落地

- `scripts/check_xi_invariants.sh` 实现 15 条 invariant
- CI 在 macOS / Linux 接入
- 估算约 400 行（脚本 + 测试夹具）

**到此 077-A 闭环关闭，可打 `xi-077-A-done` tag。**

之后 Xi-1.2 / Xi-1.3 / Xi-1.5 / Xi-1.6 / Xi-1.7 / Xi-1.8 / Xi-1.9 可独立并行推进。

---

# 修订历史

| 日期 | 改动 | 作者 |
|---|---|---|
| 2026-05-12 | 初稿；Xi-0/1/2/3/4 五阶段；Pass table + Stage contract + 10 个 baseline pass + 业界对标 + 公共 DoD | Cascade + xingleixu |
| 2026-05-12 | 与 076 对齐：删除所有 fallback flag / dual-mode 配置；新增「阶段坐标禁入源码」/「启动前提」/「Xi↔Xm 接口契约」/「JIT 与 AOT fallback 边界」/「Stage contract 扩展策略」/「Xi 新 op 加入流程」/「Pass 表组织」/「源码与生成物目录归属」/「077-A Foundation Complete」/「删除清单」十节；反向不变量改为 A/B/C 类 + 可执行 grep；性能指标绑定具体 benchmark；风险矩阵补回滚锚点列；附录 A 改为 077-A 配套首批 PR | Cascade + xingleixu |
