# 034 — XIR: Typed SSA IR + Unified Value/GC Contract

> **状态**：active
> **前置依赖**：无（基础设施层，其他重构可并行但都需对齐此文档）
> **目标受众**：编译器/运行时核心开发者
> **设计原则**：不考虑向后兼容性，直接采用最佳设计，零技术债

### 项目优先级

1. **架构稳定、健壮**：单一管线，不维护多套并行路径
2. **AOT 性能最大化**：生产用 AOT native binary，SSA + 优化是必须的
3. **JIT 性能其次**：开发阶段的热函数加速
4. **VM 解释器用于开发体验**：减少 AOT 编译等待，启动性能不是重点

---

## 1. 动机

当前 xray 编译管线缺乏 **single source of truth** 式的语义层：

1. **前端只产出 AST + side table**
   `XaNodeTable` 把 `AstNode*` 映射到 `XrType*`，但没有结构化的带类型 IR。
   后端（codegen、JIT builder、AOT C gen）各自重新解读 bytecode+side table，
   容易出现语义分歧。

2. **三套独立的"类型→表示"映射**
   - `XrType`（22 种 kind）→ `XrRep`（6 值）——前端/codegen
   - `XirTypeKind`（12 种）→ `XR_REP_*`——JIT XIR
   - `XrSlotType`（5 值）——GC stack map
   没有统一的 `XrType → XirType → machine rep` 管道；
   每条路径用各自的 switch 独立推导，已多次引发 bug
   （如 `OP_SUBSTRING` result rep、`MAX/MIN` return type 硬编码）。

3. **VM/AOT 值布局完全不兼容**
   - `XrValue`：16B struct-of-union，`tag@byte[0]`，`payload@byte[8]`
   - `XrtValue`：16B union，`tag@byte[15]`，`payload@byte[0]`
   - `XrJitResult`：`{int64_t payload, uint64_t tag}`——仅用于 JIT↔C 桥接
   - Tag 值也不同（`XR_TAG_I64 = 3` vs `XRT_TAG_I64 = 6`）
   跨后端的语义保证（如 truthiness、boxing、GC scanning）需要人工同步。

4. **GC 扫描信息从类型到 stack map 的传递链条断裂**
   `XrSlotType` 由 `xr_type_to_slot_type()` 在前端推导，
   但 JIT 在 XIR 层又用 `XirType.kind` / `XR_REP_*` 独立推导；
   如果两条路径不一致，GC 会漏扫或误扫堆指针。

### 目标

设计一个 **Typed SSA IR（新 XIR）** + **统一的值表示/GC 契约**，
作为前端产出 → 所有后端消费的唯一权威中间表示。
新 XIR 吸收当前旧 XIR 的 SSA 构造和优化 pass 职责，同时取代当前 AST→bytecode 的直接发射路径，
实现单一管线架构。

---

## 2. 现状清单

### 2.1 前端类型系统

| 组件 | 文件 | 职责 |
|------|------|------|
| `XrTypeKind` | `xtype.h` | 22 种前端类型 kind |
| `XrType` | `xtype.h` | 带泛型参数、nullable、union 的完整类型结构 |
| `XrRep` | `xtype.h` | 6 值机器表示枚举（I64/F64/PTR/TAGGED/VOID/STR） |
| `xr_type_rep()` | `xtype.h` | XrType → XrRep 推导（含 nullable、union 折叠） |
| `xr_type_to_slot_type()` | `xtype.h` | XrType → XrSlotType（GC stack map 用） |
| `xr_type_to_xr_tag()` | `xtype.h` | XrType → XrValueTag（per-PC 类型标注） |

### 2.2 XIR 类型系统

| 组件 | 文件 | 职责 |
|------|------|------|
| `XirTypeKind` | `xir.h` | 12 种 XIR 类型 kind（含 NUMERIC、TAGGED） |
| `XirType` | `xir.h` | `{kind, flags, heap_cid}` 3 字节 |
| `xir_type_to_rep()` | `xir.h` | XirTypeKind → XR_REP_* |
| VTAG 系列 | `xir.h` | VTAG_I64/F64/PTR/BOOL/NUMERIC/NULL/TAGGED |
| `vtag_to_type_kind()` | `xir.h` | VTAG ↔ XirTypeKind 双向映射 |

### 2.3 运行时值表示

| 组件 | 文件 | 布局 |
|------|------|------|
| `XrValue` | `xvalue.h` | 16B, `tag@[0] flags@[1] heap_type@[2-3] ext@[4-7] payload@[8-15]` |
| `XrValueTag` | `xvalue.h` | NULL=0, BOOL=1, I64=3, F64=4, PTR=5, STRUCT_REF=6, NOTFOUND=7 |
| `XrtValue` | `xrt_value.h` | 16B union, `payload@[0-7] tag@[15]` |
| `XRT_TAG_*` | `xrt_value.h` | NULL=0, BOOL=1, I64=6, F64=12, PTR=13, STR=14, ... (19+) |
| `XrJitResult` | `xir_jit_runtime.h` | `{int64_t payload, uint64_t tag}` |

### 2.4 GC 扫描

| 组件 | 文件 | 职责 |
|------|------|------|
| `XrSlotType` | `xslot_type.h` | ANY/I64/F64/PTR/BOOL (stack map tag) |
| `XR_SLOT_IS_GC()` | `xslot_type.h` | `PTR || ANY` → needs scanning |
| `XrGCHeader` | `xgc_header.h` | 16B header: gc_next/type/marked/extra/objsize |
| `XrObjType` | `xgc_header.h` | ~36 种堆对象类型枚举 |

---

## 3. 设计方案

### 3.1 核心决策：Typed SSA IR + 单一管线

新 XIR 采用 **Typed SSA**（带类型的静态单赋值），这是 Go SSA、Swift SIL、Rust MIR
等现代语言的共识设计。

**所有代码都走 SSA，不做双路径。** 这是 Go 编译器的做法：
单一管线最稳定、最易维护、消除了两套 codegen 之间的语义分歧风险。
VM 启动性能不是重点，架构稳定性才是。

| 方案 | 代表 | 类型信息 | 优化友好 | xray 选择 |
|------|------|----------|----------|----------|
| 高级语义 IR（非 SSA） | OCaml Lambda | ✅ 保留 | ❌ 难优化 | ❌ |
| 低级 SSA IR | LLVM IR / Cranelift | ❌ 丢失 | ✅ 易优化 | ❌ |
| **Typed SSA IR** | **Swift SIL / Rust MIR / Go SSA** | **✅ 完整** | **✅ 天然** | **✅** |

### 3.2 命名决策

| 新名称 | 旧名称 | 说明 |
|--------|--------|------|
| **XIR** (Xray IR) | 新建 | xray 的核心 Typed SSA IR，符合 `xir_` 前缀惯例 |
| 无名称 (codegen 内部) | 旧 XIR | JIT machine lowering 的内部实现细节 |
| 删除 | xcodegen | AST→bytecode 直接发射，被单一管线取代 |
| 删除 | XIR builder | bytecode→SSA 反向工程，被新 XIR 取代 |

参考：Go 的 SSA 包直接叫 `ssa`，Swift 叫 `SIL`，Rust 叫 `MIR`。
XIR = Xray IR，简洁且命名空间复用现有前缀。

### 3.3 管线架构

**当前管线（三套并行路径，bug 温床）**：
```
AST + XaNodeTable → xcodegen (AST walk) → bytecode (无类型)     ← 路径 1
                                              ↓ (热函数)
                                          XIR builder (从 bytecode 反向工程) ← 路径 2
                                              ↓
                                          XIR (SSA) → passes → regalloc → native
                                              ↓ (AOT)
                                          xcgen (XIR → C)                    ← 路径 3
```

**目标管线（单一路径，三后端只是优化级别和输出格式不同）**：
```
AST + XaNodeTable ──→ XIR (typed SSA)  ←── ALL code goes through this
                            │
                            ├─→ [prod] full opts → C emission → cc → native binary (AOT)
                            │
                            ├─→ [dev]  some opts → machine lowering → native (JIT)
                            │
                            └─→ [dev]  skip opts → out-of-SSA → bytecode (VM)
```

关键变化：
- **单一管线**：所有代码都走 XIR，不再有并行 codegen 路径
- **xcodegen 删除**：当前 AST→bytecode 的直接发射（~4000 行）被取代
- **XIR builder 删除**：当前 bytecode→SSA 的反向工程（~3000 行）被取代
- **优化 pass 只写一次**：常量折叠、DCE、GCM、类型特化全在 XIR 层
- **三后端统一消费**：VM/JIT/AOT 都从同一个 XIR 出发

### 3.4 XIR 设计原则

- **SSA 形式**：每个值只定义一次，phi 节点处理控制流汇合
- **显式类型注解**：每个 XIR value 自带 `XrType*`，不再从 AST 反向推导
- **显式 rep 标注**：每个 XIR value 附带 `XrRep`，GC/JIT/AOT 的权威表示来源
- **显式控制流**：基本块 + 分支/跳转，不保留 AST 的嵌套结构
- **中等操作粒度**：保留高级语义（`XIR_CALL_METHOD`、`XIR_ITER_NEXT`），
  但显式化控制流（`for-in` 拆分为 iterator + branch + phi）

### 3.5 XIR Node 结构（草案）

```c
// SSA value: every definition is unique, carries authoritative type + rep
typedef struct XirValue {
    uint32_t id;         // SSA value ID (unique within function)
    XrType *type;        // authoritative compile-time type
    XrRep rep;           // authoritative machine representation
    uint8_t gc_tag;      // GC slot type (derived from rep, cached)
} XirValue;

// SSA instruction: defines one value, consumes N operands
typedef struct XirInst {
    XirOp op;            // operation kind (mid-level: CALL_METHOD, ITER_NEXT, ...)
    XirValue def;        // result value (SSA definition)
    uint16_t nargs;      // number of operands
    uint32_t *args;      // operand SSA value IDs (uses)
    uint32_t line;       // source location for diagnostics
    uint32_t flags;      // side-effect, may-throw, may-gc, ...
} XirInst;

// Phi node: placed at block entry for control-flow merges
typedef struct XirPhi {
    XirValue def;        // result value
    uint32_t *incoming;  // parallel arrays: [block_id, value_id] pairs
    uint16_t count;
} XirPhi;

// Basic block: linear sequence of instructions, terminated by branch/return
typedef struct XirBlock {
    uint32_t id;
    XirPhi *phis;        // phi nodes at entry
    uint16_t nphi;
    XirInst *insts;      // instructions (last one is terminator)
    uint32_t ninsts;
    uint32_t *succs;     // successor block IDs
    uint16_t nsucc;
} XirBlock;

// Function: entry block is blocks[0], params are block 0 phi-like defs
typedef struct XirFunc {
    const char *name;
    XrType *return_type;
    XirValue *params;    // function parameters as SSA values
    uint16_t nparam;
    XirBlock *blocks;
    uint32_t nblocks;
    uint32_t nvalues;    // total SSA value count (for vreg sizing)
    // upvalue descriptors, captures, source map...
} XirFunc;
```

**每个 XIR value 示例**：
```
v12: Array<int>, rep=PTR, gc=PTR
  ├─ VM codegen:  store to R[3], slot_type=PTR in stack map
  ├─ JIT lowering: vreg v12, needs GC barrier at safepoints
  └─ AOT C gen:   XrArray* _v12 = ...
```

### 3.6 统一的类型→表示管道

**XIR 的 `XrType* + XrRep` 是所有后端的唯一权威。**

```
XrType* ──→ xr_type_rep() ──→ XrRep  ←── stored in XirValue.rep
              │                   │
              │                   ├──→ XR_REP_I64    → raw int64
              │                   ├──→ XR_REP_F64    → raw double
              │                   ├──→ XR_REP_PTR    → GC pointer (barrier)
              │                   ├──→ XR_REP_TAGGED → full XrValue (16B)
              │                   └──→ XR_REP_VOID   → no value
              │
              └──→ xr_type_to_slot_type() ──→ gc_tag in XirValue
```

**删除的独立类型系统**（不再需要）：
- 旧 `XirTypeKind`（12 种）→ 被 `XirValue.type->kind` 取代
- `VTAG_*` 系列 → 被 `XirValue.rep` 取代
- `vtag_to_type_kind()` / `type_kind_to_vtag()` 双向映射 → 删除
- 旧 `xir_type_to_rep()` / `xir_type_from_rep()` → 删除
- 旧 XIR builder 中 ~40 个独立推导 rep 的 switch 语句 → 全部从 XIR 继承

### 3.7 统一值表示：XrtValue → XrValue

**决策**：AOT 放弃 `XrtValue` 独立布局，统一到 `XrValue`。不做兼容层。

理由：xray 无外部用户、代码量小、AOT 复用 runtime+coro 已是既定策略。
保留两套布局只会持续产生同步 bug（已发生多次）。

统一项：

| 项目 | 现状 | 目标 |
|------|------|------|
| 值布局 | XrValue(tag@0) vs XrtValue(tag@15) | **统一为 XrValue** |
| Tag 枚举 | XR_TAG_I64=3 vs XRT_TAG_I64=6 | **统一为 XrValueTag** |
| Boxing | `XR_FROM_INT` vs `xrt_box_int` | **统一宏** |
| Truthiness | `xr_value_truthy()` vs `xrt_truthy()` | **统一函数** |
| GC scanning | `XR_VALUE_NEEDS_GC()` | **AOT 也用同一宏** |
| String | `XrString*` (GC heap) vs `const char*` (static) | 区分 XR_TAG_PTR(XrString) vs static C string |
| ARC | `xrt_arc_retain_val` on XrtValue | 迁移到 XrValue 布局上工作 |

### 3.8 GC 契约统一

**XIR value 的 `rep` + `gc_tag` 是 GC 扫描的唯一权威依据。**

| rep | GC 行为 | 说明 |
|-----|---------|------|
| `XR_REP_PTR` | 必须扫描 | 堆指针，需要 write barrier |
| `XR_REP_TAGGED` | 运行时检查 tag | 完整 16B XrValue，tag==PTR 时扫描 |
| `XR_REP_I64/F64/VOID` | 不扫描 | 原始值，GC 跳过 |

Stack map 生成路径（统一从 XIR 出发）：
- **VM codegen**：emit bytecode 时同步写 `gc_tag` 到 stack map
- **JIT machine lowering**：vreg 的 GC 标记从 XIR value 继承，safepoint 精确
- **AOT C gen**：C 变量类型从 `rep` 派生，retain/release 点也从 `rep` 决定

`XrGCHeader` 16B 布局不变。

---

## 4. 实施路径

### S1：统一 XrRep 为所有后端的权威表示源

**范围**：消除旧 XIR builder 和 AOT C gen 中独立推导 rep 的 switch 语句。

- 新增 `xr_type_to_xir_type(XrType*)` 工具函数（过渡期用）
- JIT builder 使用此函数代替 bytecode pattern scan 推导类型
- AOT `xcgen_struct.h` 的 `xir_type` 字段从 XrType 派生
- 消除所有 `b->aot_mode ? XR_REP_X : XR_REP_Y` 分支

**验证**：ctest + regression + AOT 全量 diff = 0

### S2：XrtValue → XrValue 统一

**范围**：AOT 放弃独立值布局，删除 XrtValue。

- `xrt_value.h` 改为 `#include "xvalue.h"` 的薄封装（过渡），最终删除
- `XRT_TAG_*` 统一为 `XR_TAG_*` 值
- `xrt_box_*` / `xrt_unbox_*` 内联为 `XR_FROM_*` / `XR_TO_*`
- `xrt_truthy()` → `xr_value_truthy()`
- `xrt_arc_retain_val` / `xrt_arc_release_val` 适配 XrValue 布局
- AOT 生成的 C 代码 `#include "xvalue.h"`

**验证**：ctest + regression + AOT diff = 0 + ASAN

### S3：新 XIR Typed SSA 框架

**范围**：在 `src/ir/` 新建 XIR 模块。

- 定义 `XirOp` 操作枚举（中等粒度：保留 CALL_METHOD / ITER_NEXT 等高级语义）
- 实现 `XirFunc` / `XirBlock` / `XirInst` / `XirPhi` / `XirValue` 数据结构
- 实现 `xir_lower(AstNode*, XaAnalyzer*)` → `XirFunc*` 的 lowering pass
  - 变量 → SSA values（含 phi 插入，Braun 算法）
  - 控制流 → 基本块 + 显式分支
  - 类型 → 从 XaNodeTable 直接继承到每个 XirValue
- 实现 `xir_dump(XirFunc*)` 可读文本输出

**验证**：dump 输出与预期语义对照；round-trip 验证（XIR → bytecode → 执行 = 原始执行）

### S4：优化 pass 统一到 XIR 层

**范围**：当前旧 XIR 层的优化 pass 迁移到新 XIR。

- 常量折叠（当前分散在 analyzer + 旧 XIR pass）→ `xir_pass_constfold()`
- 死代码消除 → `xir_pass_dce()`
- GCM（全局代码移动）→ `xir_pass_gcm()`
- 类型特化（当前旧 XIR TFA）→ `xir_pass_specialize()`
- 内联（未来）→ `xir_pass_inline()`

**优势**：所有 pass 可访问完整 `XrType*`，不再需要从 rep/tag 反向猜测类型。

**验证**：ctest + regression + AOT 全量 diff = 0

### S5：后端从新 XIR 消费，删除旧路径

**范围**：VM codegen、JIT、AOT 全部从新 XIR 消费。

- **VM codegen**：XIR → out-of-SSA（phi 消除 + 寄存器分配）→ bytecode + stack map
  - 复用现有 phi_elim + regalloc 算法
- **JIT**：XIR → machine lowering（指令选择 + 寄存器分配）→ native
  - 可借鉴 Go 的做法：在 SSA 内部 lowering（generic ops → arch ops）
- **AOT**：XIR → C code emission
  - SSA 天然适合 C 代码生成（每个 SSA value = 一个 C 局部变量）
  - 类型特化直接读 `XirValue.type`，不再需要 `b->aot_mode` 分支

**删除**：
- `src/frontend/codegen/xcodegen*`（AST→bytecode 直接发射，~4000 行）
- `src/jit/xir_builder*`（bytecode→SSA 反向工程，~3000 行）

**验证**：全量 ctest + regression + AOT diff = 0 + JIT benchmark 无退化 + ASAN

---

## 5. 与现代语言编译器的对比

### 5.1 架构对比

| 语言 | 核心 IR | SSA? | 带类型? | 解释器 | 单一管线? |
|------|---------|------|---------|--------|----------|
| **Go** | Go SSA | ✅ | ✅ | 无 | ✅ 所有代码走 SSA |
| **Swift** | SIL | ✅ | ✅ | 无 | ✅ 所有代码走 SIL |
| **Rust** | MIR | ✅ | ✅ | 无 | ✅ 所有代码走 MIR |
| **Dart VM** | FlowGraph (SSA IL) | ✅ | ✅ | KBC (仅 dynamic modules) | ✅ JIT+AOT 共享同一 IL |
| **V8** | Maglev/Turbofan | ✅ | 部分 | bytecode | ❌ 解释器独立路径 |
| **OCaml** | Typedtree→Lambda→Cmm | Cmm是 | Typedtree是 | bytecode | ❌ bytecode/native 分叉 |
| **xray 当前** | bytecode + 旧XIR | 旧XIR是 | 否 | bytecode | ❌ 3条并行路径 |
| **xray 目标** | **XIR** | **✅** | **✅** | **bytecode** | **✅** |

### 5.2 借鉴优先级

**1. Go SSA（最值得借鉴）**

Go 是最接近 xray 目标架构的参考：
- 单一管线：所有代码都走 SSA，编译速度仍然极快
- SSA 内部 lowering：`OpAdd64 → OpARM64ADD`，**不引入新 IR**
- 有完整类型信息，优化 pass 直接读类型

**2. Swift SIL（ARC 优化参考）**

- SIL 上做 ARC retain/release 优化（消除冗余引用计数）
- 保留高级语义操作（`apply`、`class_method`）
- xray 的 AOT standalone ARC 路径可借鉴同样模式

**3. Rust MIR（语义分析参考）**

- MIR 承载 borrow checking（在 SSA 上做核心语义分析）
- 明确的 drop 点、生命周期标注

关键教训：
- **单一管线是现代共识**：Go/Swift/Rust/Dart 都是所有代码走同一条 IR 管线
- **SSA 内部 lowering**：Go 不引入单独的 machine IR，直接在 SSA 中替换操作
- **优化在 IR 层**：不在后端重复推导类型

### 5.3 Dart VM 架构深度参考

Dart 是唯一同时支持 **VM(interpreter) / JIT / AOT** 三端的工业级语言，
且三端共享同一个 SSA IL（FlowGraph），与 xray 的 XIR 设计目标最接近。

源码位于 `runtime/vm/compiler/`（工作区 `参考代码/dart`）。

#### 5.3.1 管线总览

```
Dart Source → [pkg/front_end] → Kernel Binary (序列化 AST)
                                     │
          ┌──────────────────────────┘
          │
          ▼
  kernel_to_il.cc / kernel_binary_flowgraph.cc
          │
          ▼
     FlowGraph (Typed SSA IL)    ← 单一权威 IR
          │
          ├── Unoptimized JIT:  直接 → regalloc → native code（无优化）
          ├── Optimized JIT:    30+ pass → regalloc → native code
          └── AOT (precompiler): AOT 特化 pass → native code → ELF snapshot
```

**关键事实**：Dart 即使 unoptimized 模式也编译到原生机器码（不解释字节码）。
KBC 字节码解释器仅用于 `DART_DYNAMIC_MODULES` 场景（2024 年新增）。

#### 5.3.2 JIT/AOT 共享同一 Pass 管线

Dart 用枚举 `PipelineMode { kJIT, kAOT }` 区分，
但 **JIT 和 AOT 走同一个 `RunPipeline()` 函数**，
区别仅在于 AOT 多跑几个 `INVOKE_PASS_AOT(...)` 步骤：

```
// compiler_pass.cc:286 — 共享管线（简化）
RunPipeline(mode):
  ComputeSSA
  INVOKE_PASS_AOT(ApplyClassIds)      // AOT-only
  INVOKE_PASS_AOT(TypePropagation)    // AOT-only
  ApplyICData
  TypePropagation
  Inlining
  TypePropagation                     // 多次 Type Propagation 是有意的
  ConstantPropagation
  SelectRepresentations               // ← 核心：决定 box/unbox
  CSE
  LICM
  DSE
  RangeAnalysis
  DCE
  SelectRepresentations_Final         // ← 第二次，确保所有 rep 匹配
  EliminateWriteBarriers
  AllocateRegisters                   // linear scan
  GenerateCode
```

**xray 借鉴**：XIR 也应采用 `enum XirPipelineMode { XIR_MODE_VM, XIR_MODE_JIT, XIR_MODE_AOT }`，
共享 pass 管线，仅在特定 pass 前检查模式。

#### 5.3.3 可直接借鉴的设计：SelectRepresentations

Dart 最值得 xray 借鉴的单一设计。

`SelectRepresentations` pass 在 SSA 图上为每个 value 选择最优的物理表示
（tagged object / unboxed int64 / unboxed double / unboxed SIMD），
然后**自动插入 Box/Unbox 转换指令**：

```cpp
// flow_graph.cc:2509
void FlowGraph::SelectRepresentations() {
  // 1. 为每个 phi 决定是否 unbox
  PhiUnboxingHeuristic phi_unboxing_heuristic(this);
  for (phi : all_phis) {
    phi_unboxing_heuristic.Process(phi);
  }
  // 2. 为每个 definition 插入 Box/Unbox 转换
  for (def : all_definitions) {
    InsertConversionsFor(def);  // 如果 use 需要 tagged 但 def 是 unboxed → 插入 BoxInstr
  }
}
```

**对 xray 的意义**：当前 xray 的 `BOX_I64`/`UNBOX_F64` 分散在 20+ 个 codegen 文件中，
是逐表达式的局部决策。XIR 应实现一个等价的 `xir_pass_select_rep()` pass：
- 每个 `XirValue` 附带 `rep` 字段
- pass 遍历所有 def-use 对，在 rep 不匹配处插入 `XIR_BOX` / `XIR_UNBOX`
- 局部变量可以全函数保持 `XR_REP_I64`，只在边界（函数调用、闭包捕获、多态点）box

这解决了 035 文档中识别的「local 永远 tagged」瓶颈。

#### 5.3.4 可直接借鉴的设计：CompileType lattice

Dart 的 `CompileType`（`compile_type.h`）是一个三维格（lattice）：

```
CompileType = (can_be_null: bool, cid: intptr_t, type: AbstractType*)

          ⊤ = nullable Dynamic (cid=kDynamicCid)
         / \
        /   \
    non-null  specific cid (e.g. _Smi, _Double)
        \   /
         \ /
          ⊥ = None (dead code)
```

提供明确的 `Union()`（join）操作用于 phi 汇合，`IsSubtype()` 用于 type guard 消除。

**对 xray 的意义**：xray 的 `XrType` 已有 22 种 kind + nullable + union 信息，
但缺少形式化的 lattice 操作（join/meet）。XIR 的 TypePropagation pass 需要：
- `xr_type_join(XrType*, XrType*)` — phi 汇合点的类型合并
- `xr_type_meet(XrType*, XrType*)` — type guard 后的类型收窄
- `xr_type_is_subtype(XrType*, XrType*)` — 判断是否可消除 type check

#### 5.3.5 可直接借鉴的设计：CompilerPass 框架

Dart 的 pass 框架极其干净（`compiler_pass.h/cc`，~570 行）：

```cpp
// 定义一个 pass：仅需宏 + lambda
COMPILER_PASS(ConstantPropagation, {
  ConstantPropagator::Optimize(flow_graph);
});

// 每个 pass 可独立禁用、打印前后 IR、计时
CompilerPass::Get(kConstantPropagation)->Run(state);
```

特性：
- **可禁用**：`--compiler-passes=-ConstantPropagation` 直接跳过
- **可观测**：`--compiler-passes=*` 打印每个 pass 前后的 IR
- **可重复**：`COMPILER_PASS_REPEAT` 自动 canonicalize → 重跑
- **DEBUG 模式自动校验**：每个 pass 后跑 `FlowGraphChecker`

**对 xray 的意义**：XIR 应实现类似框架：
```c
// 目标 API
typedef bool (*XirPassFn)(XirFunc *func, XirPassState *state);
void xir_register_pass(const char *name, XirPassFn fn);
void xir_run_pipeline(XirFunc *func, XirPipelineMode mode);
// 调试：XIR_DUMP_PASSES=constfold,dce 打印指定 pass 前后的 XIR
```

#### 5.3.6 Dart vs xray 的关键差异

| 维度 | Dart | xray | 影响 |
|------|------|------|------|
| **IL 输出** | 全部→原生机器码 | bytecode + native + C | xray 需要额外的 XIR→bytecode lowering |
| **Regalloc 目标** | 物理寄存器（~30 GPR） | bytecode register（255 个） | xray 的 regalloc **更简单**（几乎无 spill） |
| **IL 规模** | ~200 种指令，~21K 行 | ~60-80 种，~2K 行 | xray 目标是 bytecode 指令集 + C codegen，不需要多 ISA |
| **Linear scan** | ~4,400 行 | greedy coloring ~300 行足矣 | 255 个虚拟 reg 远超需求 |
| **Deoptimization** | 完整 deopt 框架（~48K 行） | 无需（bytecode fallback） | xray JIT 可简单回退到 bytecode，无需 deopt metadata |
| **类型反馈** | ICData / call site feedback | inst_types side table | XIR 自带类型，JIT 可额外利用 runtime profiling |

**核心结论**：Dart 的 FlowGraph 是 ~60K 行的工业级 IL，
xray 的 XIR 只需其 1/10 规模即可覆盖全部需求，
但 SelectRepresentations + CompileType lattice + Pass 框架三个设计可直接复用模式。

---
## 6. 风险与约束

| 风险 | 影响 | 缓解 |
|------|------|------|
| XIR 设计过大，前端改动量爆炸 | 编译管线停摆 | S1/S2 先行不依赖新 XIR；S3 增量实施 |
| XrtValue → XrValue 破坏 AOT ARC | 内存管理失效 | S2 同步验证 ARC 与 XrValue 布局 |
| xcodegen + XIR builder 删除后管线中断 | 编译不可用 | S5 逐函数迁移，保留旧路径 fallback 直到全量切换 |
| XIR 增加编译内存开销 | 大 bundle 可能明显 | arena 分配 + 按函数 lower + 编译后释放 |
| out-of-SSA 生成 bytecode 的质量 | VM 解释性能退化 | 复用旧 XIR 已有的 phi_elim + regalloc，算法成熟；VM 性能非重点 |

---

## 7. 非目标

- **不设计新的 GC 算法**（GC 策略不变，只统一扫描信息来源）
- **不修改 XrGCHeader 布局**（16B header 保持稳定）
- **不引入增量编译**（XIR 是全量 lower，按函数粒度）
- **不改变 xray 语言语义**（XIR 是编译器内部表示）
- **不改变 VM 解释器**（VM 仍执行 bytecode，XIR → bytecode 的 lowering 对 VM 透明）

---

## 8. 已决策项

| 问题 | 决策 | 理由 |
|------|------|------|
| XIR 是否 SSA？ | **是，full SSA + phi** | 现代共识；旧 XIR 已做 SSA，上移统一 |
| 单一管线 vs 双路径？ | **单一管线** | 架构稳定优先；VM 性能非重点；消除语义分歧 |
| 操作粒度？ | **中等**：保留 CALL_METHOD 等高级语义，显式化控制流 | 方便 AOT 类型特化，同时对优化 pass 友好 |
| AOT 值布局？ | **统一到 XrValue** | 无历史包袱，AOT 复用 runtime 是既定策略 |
| 旧 XIR 是否保留？ | **退化为 machine lowering 内部细节** | 不再是独立 IR 层 |
| xcodegen 是否保留？ | **删除** | 单一管线不需要 AST→bytecode 直接发射 |
| AOT ARC vs tracing GC？ | **统一到 per-coro tracing GC**（standalone 模式保留 ARC 选项） | AOT 复用 coro 已是既定策略 |

## 9. 剩余开放问题

1. **XIR lowering 的 SSA 构造算法？**
   选项：Cytron 算法（经典）、Braun 算法（on-the-fly，适合单 pass lowering）。
   **倾向 Braun**：实现简单，直接在 AST→XIR lowering 过程中构造 SSA。

2. **XIR 是否需要 CFG dominance tree？**
   GCM 和部分优化需要 dominator 信息。
   **需要**：作为 XIR 的标准元数据，按需计算（lazy）。

3. **JIT 热函数的 XIR 生命周期？**
   选项 A：XIR 编译后释放，JIT 从 bytecode 重建（保守）。
   选项 B：XIR 保留到函数变冷（激进，省重建开销）。
   选项 C：XIR 编译后释放，但生成丰富的 per-opcode 元数据供 JIT 使用（折中）。
   **倾向 C**：内存友好，JIT 无需完整 XIR 也能获得权威类型信息。

---

## 附录 A：术语表

| 术语 | 含义 |
|------|------|
| XIR | Xray IR — 带类型的静态单赋值中间表示（新，核心 IR） |
| 旧 XIR | JIT machine lowering 内部表示（退化为 codegen 内部细节） |
| XrRep | Machine Representation — 编译时确定的机器类型（I64/F64/PTR/TAGGED/VOID） |
| XrSlotType | GC Stack Map Tag — 描述栈槽内容的 GC 可见标签 |
| XrType | 前端完整类型（含泛型、union、nullable） |
| XrValue | 运行时 16B 带标签值（VM + AOT 统一） |
| XrJitResult | JIT↔C 桥接返回值（payload + tag，JIT 内部协议） |
| out-of-SSA | SSA → 寄存器分配的变换（phi 消除 + copy insertion） |
| xcodegen | 当前 AST→bytecode 直接发射（将被删除） |

## 附录 B：代码净减预估

| 删除项 | 估计行数 | 说明 |
|--------|----------|------|
| xcodegen (`xcodegen*.c`) | ~4000 | AST→bytecode 直接发射，被单一管线取代 |
| XIR builder (`xir_builder*.c`) | ~3000 | bytecode→SSA 反向工程，被新 XIR 取代 |
| XirTypeKind / VTAG 映射 | ~200 | 被 XIR `XrType*` 取代 |
| `xrt_value.h` | ~200 | 统一到 `xvalue.h` |
| AOT `b->aot_mode` 分支 | ~100+ | XIR 统一后端，无需区分模式 |
| **总计** | **~7500** | 净减少（XIR 框架本身预估 ~2000 行） |
