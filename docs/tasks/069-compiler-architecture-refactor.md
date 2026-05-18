# Compiler Architecture Refactor Plan

> 日期：2026-05-05（v3，补充测试真实性与 frontend binding 契约）
> 状态：设计方案
> 范围：frontend、typed AST、Xi IR、VM bytecode emission、JIT lowering、AOT codegen、module/closure metadata、compiler diagnostics、compiler tests
>
> 修订要点：与当前源码（commit 时间点）逐条比对，修正了模块/闭包 metadata 的"已存在但生成方式不对"的现状描述、`xir_*` → `xm_*` 文件路径、intrinsic 已有两套部分注册表的事实、`XmTarget` 已存在但缺函数指针表的事实、以及里程碑 8 的具体死代码靶子（`xm_blueprint`、`XrProto.blueprint`、`xr_compiler_end` 注释残留等）。本版新增测试真实性门槛、frontend symbol/scope binding 契约、AST visitor 覆盖不变量，以及 lowerer 禁止 unresolved variable 静默降级为 `LOADNULL` 的要求。

---

## 0. 目标

Xray 当前已经完成最关键的方向收敛：`xr_compile()` 走统一 Xi IR pipeline，VM/JIT/AOT 都从 Xi 取得语义来源。下一轮重构不应继续堆局部补丁，而应把这条统一管线制度化，形成长期可演进的编译器架构。

本方案目标：

- 让 frontend、IR、backend 的职责边界稳定且可验证。
- 让 Xi 从“单个 typed SSA IR”升级为“有明确状态和不变量的 typed SSA pipeline”。
- 让 `xray test` 与回归脚本先证明测试函数真实执行，避免"零执行但全绿"。
- 让 analyzer 输出的 symbol/scope binding 成为 frontend 到 Xi 的硬契约。
- 让 module、closure、ownership、intrinsic、effect、representation 成为一等元数据或一等 IR 语义，而不是散落在 lowerer、emitter、JIT、AOT 里。
- 让 VM 成为语义参考后端，JIT/AOT 成为从同一 Xi 语义派生出的优化后端。
- 直接采用最佳设计，不保留旧接口、不做兼容层、不维护 fallback 路径。

---

## 1. 设计原则

### 1.1 单一语义来源

所有语言语义只能来自：

```text
Source -> Parser -> Analyzer -> Typed AST Canonicalizer -> Xi pipeline
```

VM、JIT、AOT 不允许各自重新解释：

- module import/export
- closure capture
- shared cell
- enum/class layout
- builtin method semantics
- intrinsic/helper mapping
- coroutine/exception/defer lowering
- ownership/lifetime rules

后端只消费 Xi 的显式结构和 metadata。

### 1.2 明确 IR 状态

Xi 不再只是“一个 IR”。每个函数必须声明自己处于哪个状态；每个 pass 必须声明输入状态、输出状态、依赖分析和失效分析。

推荐状态：

```text
XiRaw
  AST 直接降低后的 typed SSA，允许保留高层语义 op。

XiCanonical
  求值顺序、语法糖、scope、defer、try/finally、for-in、match 已规范化。

XiClosed
  closure environment、upvalue、cell、module live binding、class/enum layout 已显式化。

XiOwned
  move/borrow/share、escape、retain/release、write barrier、safepoint effect 已显式化。

XiRepped
  value representation 已选择，BOX/UNBOX 边界稳定，backend 可以消费。

XiBackend
  只剩后端可直接降低的低层 op、runtime intrinsic、call ABI、safepoint/deopt 信息。
```

### 1.3 后端不补语义

后端可以做：

- instruction selection
- register allocation
- C code emission
- bytecode emission
- runtime helper call emission
- guard/deopt lowering
- safepoint map emission

后端不可以做：

- 推断 import/export 关系
- 重新分析 capture cell
- 重新决定 enum backing type
- 通过函数名字符串猜 helper ABI
- 静默丢弃未知 intrinsic/helper
- 为 VM/JIT/AOT 各自维护一套 builtin 语义

### 1.4 每个根因只处理一次

如果同一类 bug 在 VM、JIT、AOT 同时出现，说明根因属于 IR contract 或 metadata contract，而不是三个后端都需要局部修补。

典型例子：

- `OP_IMPORT + OP_GETPROP` 在 AOT 中反扫或特判，应收敛为 Xi module metadata。
- builtin method 的返回 representation 不一致，应收敛为 intrinsic/effect/rep table。
- helper 未识别被静默丢弃，应收敛为后端 helper registry hard fail。

### 1.5 测试结果必须先证明自己可信

测试框架本身也是编译器基础设施。任何"全绿"结论都必须先满足：

- `@test` 函数真实被 discover 并执行。
- 回归脚本统计的是执行结果，而不是只统计编译退出码。
- 故意失败的 `@test` 必须让命令返回非零。
- 空测试文件必须被明确报告为 0 tests，不能混入 pass 统计。
- 每个回归 bucket 必须记录真实执行数量、失败数量、超时数量。

### 1.6 Frontend binding 是 lowering 的硬输入

Xi lowering 不应重新猜测名字绑定，也不应把 unresolved variable 降级为合法 null。Analyzer 完成后必须满足：

- 每个声明节点拥有稳定唯一的 `symbol_id`。
- 每个普通变量引用拥有非零 `symbol_id`。
- 每个作用域构造拥有可追踪的 scope identity。
- Pass 1 collect 与 Pass 2 infer 进入的 scope tree 一致。
- 每个 AST 子表达式都被 analyzer 访问；需要上下文类型的表达式不得在无上下文路径中过早报错。
- lowerer 遇到未解析变量必须 hard fail，禁止静默生成 `LOADNULL`。

---

## 2. 当前基线

### 2.1 已完成的正确方向

当前主线已经具备成熟编译器的基本骨架：

```text
Source
  -> AST
  -> analyzer
  -> Xi IR typed SSA
  -> Xi verifier
  -> Xi optimization pipeline
  -> backend
       ├─ VM:  xi_emit       -> bytecode (XrProto)
       ├─ JIT: xi_to_xm_lower -> Xm SSA -> machine code
       └─ AOT: xi_cgen        -> C source -> native binary
```

关键进展（已验证）：

- `xr_compile()` 在 `src/frontend/codegen/xcompiler.c` 中只剩 Xi pipeline 一条路径，无 legacy compiler fallback。
- VM import/export 由 `src/ir/xi_emit_cf.c::emit_module_exports()` 在 RETURN 终结块前统一发出 `OP_GETSHARED + OP_EXPORT`。
- AOT 多模块编译在 `src/aot/xaot_driver.c` 已支持依赖拓扑、`xi_module_populate_exports()` 抽出 exports/classes、`xi_cgen_resolve_module_imports()` 在 codegen 前解析跨模块 import。
- JIT 入口 `src/jit/xi_to_xm.c::xi_to_xm_lower()` 直接消费 `proto->xi_func`，不再从 bytecode 反建 SSA；IC feedback 通过 `XmICSnapshot` 显式传入。
- `XiModule / XiModuleExport / XiModuleImport / XiClassData` 已在 `src/ir/xi.h` 存在，可以承载跨后端 metadata。
- `xi_pipeline.c` 已支持 `XI_PIPE_VM` / `XI_PIPE_AOT` 模式开关，`SelectRepresentations + BoxElim` 已作为 pass 接入（AOT 默认开）。
- `XiPipelineStats`、opt level、change tracker、budget 都已落地，`XRAY_XI_STATS=1` 即可输出。
- `xi_verify` 已检查 SSA dominance、phi 谓语对齐、operand arity、type contracts、side-effect flag、CFG succ/pred 对称性。
- AOT 后端有 `XR_INTRIN_*`（30 个，`src/jit/xm_intrinsic.h`）和 `XRT_SYM_*`（70 个，`src/aot/xrt_method_symbols.h`）两套部分注册表，未来 intrinsic 合约表可以收敛它们。

### 2.2 系统性缺口

| 区域 | 当前缺口（具体到代码） | 风险 |
|------|------------------------|------|
| Frontend | `src/frontend/parser/xtype_ref.c`、`xtype_scope.c`、`xparse_type.c` 仍承担符号绑定/类型作用域逻辑；`AstNode` 只有 line/column，没有稳定 `node_id` | parser 与 analyzer 紧耦合，formatter/LSP 难以稳定引用节点 |
| Xi IR | `XiFunc` 无 `stage` 字段；`XiPassDesc` 仅有 `name/fn/min_level/flags`，无 input/output stage | pass 顺序和 backend 输入条件靠约定，`xi_opt_select_rep` 之后没人能拒绝高层 op 回流 |
| Test Harness | 回归测试曾出现 `@test` 未真实执行但脚本报告 pass；缺少 `xray test` 自检和回归执行数量断言 | 绿色结果可能不可信，真实语义 bug 被长期隐藏 |
| Frontend Binding | `symbol_id` 绑定依赖 analyzer visitor 完整遍历；缺少 AST child visit 覆盖测试；lowerer 遇到未解析变量仍可能产生 `LOADNULL` | `symbol_id=0 -> LOADNULL` 会把编译器 bug 伪装成运行期 null |
| Scope | Pass 1 collect 与 Pass 2 infer 的 scope 进入策略分散在多个 visitor；函数体、裸块、控制流 body 容易漂移 | shadowing、closure capture、for/select/catch 绑定变量出现跨阶段不一致 |
| Module | `XiModule` 已存在但 `xi_module_populate_exports()` 仍**事后扫描 IR 模式**（`SET_SHARED + CLOSURE_NEW/CLASS_CREATE`）；VM 路径用 `XiFunc.export_names`，AOT 路径用 `XiModule.exports`，两条数据来源不一致 | metadata 与 IR 双向漂移；新 export 形态需要同时改两处 |
| Closure | `XiCapture` 已存在于 `XiFunc.captures[]`，但没有独立 `XiClosureMeta`，env layout / cell index / mutability 没有显式字段；upvalue 解析与 shared cell 解析在 lowerer 里混合 | mutable capture、嵌套闭包、coroutine 共享在 VM/JIT/AOT 间容易出现微差 |
| Intrinsic | `xm_intrinsic.h::XR_INTRIN_*` 只覆盖 JIT→AOT 桥；`xrt_method_symbols.h::XRT_SYM_*` 只覆盖 method dispatch；VM 对应的 builtin 在 `xanalyzer_builtins.c` 自成一套；三套来源没有合并 | 新增 builtin/helper 要改三处；未识别 helper 在某条路径上仍可能静默丢失 |
| Ownership | `xanalyzer_escape.c` 已做强制 share 检查，但 IR 上没有 effect summary / retain-release op / write barrier op；AOT 仍依赖 runtime 兜底 | AOT ARC、GC barrier、coroutine safety 难以一次到位 |
| Backend | `XmTarget` 只有寄存器/帧布局，没有 `lower_abi / select_instr / emit_func / safepoint_layout` 等函数指针；deopt/safepoint 用的常量散落在 `xm_codegen_*.c` | x64/ARM64/AOT 之间隐式假设增多 |
| Dead code | `src/jit/xm_blueprint.[ch]` 中的 `xr_blueprint_generate()` 没有任何调用方，`xchunk.h:354`、`xm_blueprint.h:57` 注释引用了不存在的 `xr_compiler_end()`；`xm_codegen_stub.c:333` 因为 `proto->blueprint` 永远是 NULL，OSR 直接跳过 | 误导阅读者；OSR 性能特性其实没在跑 |
| Tests | `tests/unit/ir/` 已有 lower/opt/emit/cgen/to_xm 单测，但**没有 stage invariant 测试**，也没有"VM/JIT/AOT 三后端 diff 矩阵"专门桶 | miscompile 发现晚、定位慢 |

---

## 3. 可借鉴的成熟设计

### 3.1 Go：frontend facts、canonicalization、SSA pass 纪律

已定向阅读的源码：

- `src/go/types/api.go`：`Info` 把 `Types / Defs / Uses / Implicits / Selections / Scopes / Instances` 作为 AST 外挂语义事实表。
- `src/go/types/scope.go`、`object.go`：`Scope` 是 parent/children/object map 树；`Object` 是唯一命名实体，记录 parent scope、position、package、name、type。
- `src/cmd/compile/internal/types2/recording.go`：`recordDef / recordUse / recordScope / recordSelection / recordImplicit` 是集中记录入口，避免语义 facts 散落。
- `src/cmd/compile/internal/ssa/compile.go`：SSA pass 列表集中定义；`passOrder` 在 init 时检查顺序约束；`ssa/check/on` 可在每个 pass 后运行 verifier。
- `src/cmd/compile/internal/ssa/check.go`：`checkFunc` 检查 CFG succ/pred 对称、block control 类型、operand arity、aux 类型、phi pred 数量、value 唯一性等。
- `src/cmd/compile/internal/walk/order.go`：在 SSA 前固定求值顺序，引入临时变量，约束 map index / channel receive 等复杂表达式只出现在规范位置。
- `src/cmd/compile/internal/escape/escape.go`、`graph.go`：escape 是独立数据流分析；注释明确要求总是访问完整 AST，防止遗漏子表达式副作用。
- `src/cmd/compile/internal/walk/closure.go`：closure conversion 根据 capture by value/reference 改写参数或 closure object。
- `src/cmd/compile/internal/ir/func.go`、`name.go`：函数元数据集中保存 `ClosureVars / Closures / ClosureParent / Parents / Marks`，name 保存 `Defn / Curfn / Outer / Heapaddr` 等 binding 与 capture 信息。

可直接学习：

- frontend 输出独立 semantic facts table，而不是只把类型和 symbol 零散塞进 AST 字段。
- scope tree 是一等数据结构，scope 与定义它的 AST node 显式关联。
- selector/member access 需要独立 selection fact，记录 kind、receiver type、目标 object、index path、indirect。
- pass 列表、pass 调试开关、pass 顺序约束、pass 后 verifier 应集中管理。
- verifier 不只检查 SSA dominance，还要检查 block/value 结构、operand、aux、phi、control 类型等低级不变量。
- canonicalization 在 lowering 前处理求值顺序、临时变量、复杂表达式落点。
- closure capture 决策应由 escape/capture facts 驱动，后端只消费结果。

落到 Xray：

- 新增 `XaTypedInfo`，作为 analyzer 的权威输出：`types / defs / uses / scopes / selections / implicits / instances`。
- `symbol_id` 对应 Go `Object` 的唯一实体职责；`scope_id` 对应 Go `Scope` 的树结构职责。
- `node_id -> scope_id/type/symbol/selection` 作为 lowerer 唯一输入，lowerer 不重新做名字解析。
- `AST_MEMBER_ACCESS`、`AST_METHOD_CALL`、`AST_INDEX_GET`、`AST_INDEX_SET` 等要记录 selection/index facts，避免 backend 猜字段或方法。
- `XiPassDesc` 除 input/output stage 外，还应有 `required`、debug/dump/stats 开关和 explicit order constraints。
- `XRAY_XI_CHECK=1` 应允许每个 pass 后运行 verifier；debug 模式可随机扰动 block 内 value 顺序，发现 pass 对偶然顺序的依赖。
- Typed AST canonicalizer 要实现类似 Go `order` 的求值顺序规范化：复杂 receiver/index/key/argument 必要时先落临时变量。
- XiClosed/XiOwned 要消费 capture facts，生成 capture kind、cell index、env offset、by-value/by-cell 结果。

### 3.2 Swift：IR stage 和 ownership

可直接学习：

- SIL 有 Raw、Canonical、Lowered 等明确状态。
- ownership/lifetime 是 IR 语义，不是 codegen 补丁。
- mandatory diagnostic passes 与 performance passes 分离。

落到 Xray：

- 给 XiFunc 增加 `stage`。
- verifier 按 stage 检查不同不变量。
- ownership/effect/lifetime 进入 XiOwned。
- diagnostic pipeline 与 optimization pipeline 分离。

### 3.3 Dart：JIT/AOT 共用图和 profile feedback

可直接学习：

- Kernel 是前端统一产物。
- FlowGraph 是 JIT/AOT 共享优化图。
- IC feedback 是 JIT lowering 和 optimization 的正式输入。
- huge method 会降级优化，避免编译时间失控。

落到 Xray：

- JIT/AOT 尽量共享 XiCanonical/XiClosed/XiOwned/XiRepped。
- IC snapshot 成为 JIT lowering 参数，而不是 builder 私有推断。
- Xi pipeline 支持 value/block 数阈值和 budget 降级。

### 3.4 OCaml：closure conversion 和多 IR 分层

可直接学习：

- closure conversion 是独立 pass。
- 高层函数语义和低层 codegen 语义分层清楚。
- module/global constants/layout 在 lowering 阶段形成稳定数据。

落到 Xray：

- 新增 XiClosed 状态。
- closure env layout、module live binding、shared cell、class/enum layout 都在 XiClosed 中显式化。
- 后端不再推断 capture/module 结构。

### 3.5 QBE：小而硬的 backend contract

可直接学习：

- 后端接口极简，但 ABI lowering、isel、liveness、spill、regalloc、emit 顺序硬。
- target 信息集中，不散落在 codegen 分支里。

落到 Xray：

- 统一 target contract。
- 显式记录 call ABI、spill slot、safepoint、deopt、resume frame。
- codegen 之前运行 backend verifier。

### 3.6 QuickJS：模块链接和闭包变量表

可直接学习：

- function def 拥有 scope、var、closure var、constant pool、module context。
- module linking 使用 DFS 状态处理循环依赖。
- compile-only 与 eval-function 分离。

落到 Xray：

- `ModuleRecord`、`ModuleInstance`、`ExportCell`、`ImportCell` 应作为 metadata 模型。
- import/export/live binding 不应由 bytecode 或 AOT 反扫得到。
- VM bytecode emission 只负责发出 metadata 已确定的操作。

---

## 4. 目标架构

```text
Source
  ↓
TokenStream + Trivia
  ↓
Syntax AST / CST
  ↓
Analyzer
  - symbols
  - types
  - diagnostics
  - typed node table
  ↓
Typed AST Canonicalizer
  - evaluation order
  - syntax desugaring
  - normalized declarations
  - structured diagnostics
  ↓
XiRaw
  ↓ verifier(raw)
XiCanonical
  ↓ verifier(canonical)
XiClosed
  - closure conversion
  - module binding
  - class/enum layout metadata
  ↓ verifier(closed)
XiOwned
  - ownership/effect/lifetime/write barrier
  ↓ verifier(owned)
XiRepped
  - representation selection
  - box/unbox boundary
  - intrinsic lowering
  ↓ verifier(repped)
XiBackend
  ↓
Backend
  ├─ VM bytecode emitter
  ├─ JIT lowering -> XIR/Xm -> machine code
  └─ AOT lowering -> C-like IR -> C/native
```

---

## 5. 核心数据契约

### 5.0 Frontend binding contract

Analyzer 输出不只是类型信息，还必须输出可被 lowering 直接消费的 binding/scope 事实。建议新增或补齐：

```text
AstNode
  node_id                  // 稳定节点 ID
  scope_id                 // 节点所属 scope

XaTypedInfo
  node_types[node_id]       // 表达式/声明/类型节点的推断类型
  defs[node_id]             // 声明节点 -> symbol
  uses[node_id]             // 普通标识符引用 -> symbol
  scopes[node_id]           // 定义 scope 的 AST 节点 -> scope
  selections[node_id]       // field/method/index selection fact
  implicits[node_id]        // 隐式声明，如 catch/select/for 生成的绑定
  instances[node_id]        // 泛型实例化 facts

DeclNode / binding pattern
  symbol_id                // 声明产生的唯一 symbol

VariableNode / member-like binding use
  symbol_id                // 解析后的目标 symbol，普通变量引用必须非零

XaScope
  scope_id
  parent_scope_id
  owner_node_id
  kind                     // program/function/block/loop/catch/select/...

XaSelection
  kind                     // field/method/index/static_member/module_export
  receiver_type
  target_symbol_id
  field_index
  is_indirect
  result_type
```

不变量：

- Pass 1 collect 和 Pass 2 infer 必须进入同一棵 scope tree。
- `AST_BLOCK` 作为函数体、控制流 body、裸块 statement 时的 scope 策略必须显式建模，不允许靠调用路径偶然一致。
- 所有 declaration、use、selection、implicit binding 都必须通过集中记录入口写入 `XaTypedInfo`，禁止 visitor 直接散写多处状态。
- `AST_MEMBER_ACCESS`、`AST_METHOD_CALL`、`AST_INDEX_GET`、`AST_INDEX_SET`、module export/import 访问必须有 `XaSelection` 或等价 fact。
- 所有 `AST_*` 节点的子表达式访问必须有覆盖表；新增 AST 字段时必须同步更新覆盖测试。
- 需要 expected type 的表达式（如匿名函数）不能在无上下文预访问路径中提交诊断。
- `xi_lower_expr()` 遇到普通变量 `symbol_id == 0` 或查找失败时必须报 internal/compile error，禁止生成 `LOADNULL`。
- `LOADNULL` 只能来自显式语言语义，例如 null 字面量、optional chain null path、safe cast 失败、解码/查询失败等。

### 5.1 Xi stage contract

建议新增：

```c
typedef enum XiStage {
    XI_STAGE_RAW,
    XI_STAGE_CANONICAL,
    XI_STAGE_CLOSED,
    XI_STAGE_OWNED,
    XI_STAGE_REPPED,
    XI_STAGE_BACKEND,
} XiStage;
```

`XiFunc` 建议增加：

```c
XiStage stage;
uint64_t invariant_mask;
XiModuleMeta *module_meta;
XiClosureMeta *closure_meta;
XiEffectSummary effect;
XiDebugMap *debug_map;
XiDeoptMap *deopt_map;
```

### 5.2 Pass descriptor contract

当前 `XiPassDesc` 已有 `name`、`fn`、`min_level`、`flags`。建议扩展：

```c
typedef struct XiPassDesc {
    const char *name;
    XiPassFn fn;
    XiOptLevel min_level;
    XiStage input_stage;
    XiStage output_stage;
    uint32_t requires;
    uint32_t preserves;
    uint32_t invalidates;
    uint32_t flags;
    bool required;
    uint8_t debug_level;
} XiPassDesc;

typedef struct XiPassOrderConstraint {
    const char *before;
    const char *after;
    const char *reason;
} XiPassOrderConstraint;
```

每个 pass 必须满足：

- 输入 stage 匹配。
- 输出 stage 单调推进或保持不变。
- 修改 CFG 时标记 `cfg_changed`。
- 修改 value/type/effect 时标记对应 change bit。
- debug 构建每个 pass 后运行 verifier。
- pipeline 初始化时检查 `XiPassOrderConstraint`，约束不满足直接失败。
- `required` pass 禁止通过环境变量关闭。
- 每个 pass 支持独立 debug/dump/stats 开关，`all` 开关可批量打开。
- `XRAY_XI_CHECK=1` 下，每个 pass 后运行 verifier；可选随机扰动 block 内 value 顺序，发现 pass 对偶然顺序的依赖。

verifier 必须覆盖：

- CFG succ/pred 对称。
- block kind 与 successor/control arity 匹配。
- branch/select/control value 类型合法。
- operand arity 与 opcode 表一致。
- aux/imm/symbol payload 类型合法。
- phi 参数数量与 predecessor 数量一致。
- value 只出现在一个 block 中，且 `value->block` 回指一致。
- side-effect、memory、safepoint、may-throw/may-suspend 标志与 opcode/effect table 一致。

### 5.3 Module metadata contract

> 现状：`src/ir/xi.h` 已有 `XiModule / XiModuleExport / XiModuleImport / XiClassData`。本节描述把它从"可选附件"升级为"一等契约"需要补齐的字段与不变量。

目标模型（在现有 struct 上扩展）：

```text
XiModule (现存)
  module_name, module_path, init                     // 已有
  exports[],  imports[], classes[]                   // 已有
+ dependency_edges[]                                 // 新增：跨模块依赖图
+ scc_id, link_status                                // 新增：SCC 编号与 link 状态
+ generated_at_lowering : bool                       // 新增：标记是否在 lowering 时产出（而非事后扫描）

XiModuleExport (现存)
  name, shared_slot, function, class_data            // 已有
+ value_type        : XrType*                        // 新增：导出值类型（VM/JIT/AOT 共用）
+ is_live_binding   : bool                           // 新增：mutable export 语义
+ cell_index        : int32_t                        // 新增：跨模块 cell 表索引

XiModuleImport (现存)
  module_path, import_name, ...                      // 已有
+ binding_kind      : { value, function, class, namespace }
+ source_export     : XiModuleExport*                // 新增：linker 解析后回填
+ cell_index        : int32_t
```

实施约束：

- exports/imports/classes 必须在 `xi_pass_close()` 阶段产出，**禁止再依赖 `xi_module_populate_exports()` 做事后 IR 扫描**。该函数最终降级为 verifier 的"再扫一遍并断言一致"工具。
- VM emit (`xi_emit_cf.c::emit_module_exports`) 改为读取 `XiModule.exports`，不再读 `XiFunc.export_names`；后者作为 lowering 内部缓冲在 close pass 后清零。
- JIT lowering (`xi_to_xm.c`) 通过 `XiModule.exports[i].cell_index` 解析 GET_SHARED/SET_SHARED 的目标。
- AOT codegen (`xi_cgen.c::cg_lookup_method`、`xi_cgen_resolve_module_imports`) 改为只读 metadata，不再做 IR 模式匹配。

### 5.4 Closure metadata contract

> 现状：`src/ir/xi.h` 已有 `XiCapture` 数组直接挂在 `XiFunc.captures[XI_MAX_CAPTURES]`，且 `XiFunc.children[]` 维护父子函数关系。但没有独立的 metadata 结构、没有 env layout、没有 mutability/cell-index 字段，也没有"capture kind"枚举。

目标：把 capture 数据从 `XiFunc` 散字段升级为独立的 `XiClosureMeta`：

```text
XiClosureMeta
  function_id              // 对应的 XiFunc 引用
  parent_func              // 词法外层函数（已有：XiFunc.parent，需补背向指针）
  child_funcs[]            // 已有：XiFunc.children
  env_layout               // 新增：cell 在闭包对象上的偏移布局
  captures[]               // 重构自 XiFunc.captures，结构升级如下

XiCaptureEntry            // 替代当前 XiCapture
  source_symbol            // 来源 var symbol_id（已有：XiCapture.source_index）
  capture_kind             // 新增：见下
  value_type               // 已有：但当前写在 XiFunc，需迁入
  cell_index               // 新增：与 XiModule.exports.cell_index 同空间
  env_offset               // 新增：在 closure object 内的偏移
  is_mutable               // 新增：决定是否需要 cell 间接
  is_shared                // 新增：是否为协程共享 cell
  is_reassigned            // 新增：是否在 capture 后重新赋值
  is_address_taken         // 新增：是否取地址或逃逸到引用上下文
  capture_size             // 新增：用于 by-value/by-cell 决策
```

capture kind 取值：

- `CAPTURE_BY_VALUE` —— 按值拷贝（immutable 标量/字符串/不可变结构）
- `CAPTURE_BY_IMM_REF` —— 按只读引用
- `CAPTURE_BY_MUT_CELL` —— 通过 mutable cell 间接
- `CAPTURE_MODULE_LIVE` —— 直接挂到 `XiModule.exports[i].cell_index`
- `CAPTURE_CORO_SHARED` —— 协程间共享 cell（受 escape analyzer 管辖）

约束：

- VM/JIT/AOT 全部从 `XiClosureMeta` 读取 layout；当前 lowerer 中 upvalue 解析 (`xi_lower_resolve_upvalue`) 与 shared cell 解析 (`xi_lower_find_shared`) 的混合逻辑迁移到 `xi_pass_close()`。
- AOT 不再通过"`SET_SHARED + CLOSURE_NEW`"模式回推 closure 关系。
- mutable capture 在 IR 上对应显式 `XI_CELL_LOAD / XI_CELL_STORE`，而不是 `XI_LOAD_UPVAL` 双重含义。
- capture 决策由 escape/capture facts 决定：未重新赋值、未取地址、体积足够小的 immutable 值可 `CAPTURE_BY_VALUE`；可变或跨协程共享值必须走 cell。
- direct closure call 可以在 canonicalizer 中改写为普通函数调用并显式传入 capture args，避免不必要的 closure object。
- closure env object 的字段顺序、offset、cell index 必须在 XiClosed 固定，后端不得重新排列。

### 5.5 Intrinsic/effect contract

> 现状：当前已有两套**部分**注册表：
> - `src/jit/xm_intrinsic.h::XR_INTRIN_*`：30 个 ID，覆盖 JIT→AOT 桥接（property/array/map/string/method dispatch/throw/typeof…）。
> - `src/aot/xrt_method_symbols.h::XRT_SYM_*`：70 个 method symbol ID，与 `xsymbol_table.h::SYMBOL_*` 通过 `_Static_assert` 对齐。
> - VM 路径的 builtin 描述散落在 `src/frontend/analyzer/xanalyzer_builtins.c` / `xanalyzer_builtins_generated.h`。
>
> 三者尚未合并，新增 builtin 通常需要改三处。

目标：建立单一 intrinsic table，让上述三套都从同一数据源导出：

```text
XiIntrinsicDesc
  id                    // 与 XR_INTRIN_* 同空间，向后兼容现有使用
  name                  // 与 XRT_SYM_* 关联（method 路径）
  argc, arg_types, arg_reps
  return_type, return_rep
  effects               // reads/writes/may_alloc/may_throw/may_suspend/safepoint
  vm_lowering           // 函数指针：发出对应 bytecode 序列
  jit_lowering          // 函数指针：emit XM_CALL_INTRINSIC 或内联
  aot_lowering          // 函数指针：发出 C 调用或内联展开
```

实施规则：

- builtin method（length/get/push/…）、stdlib bridge（json/regex/…）、runtime helper（xr_jit_*、xrt_*）都必须注册。
- 三套现有 enum 通过代码生成器从同一数据源导出，编译期 `_Static_assert` 守住一致性。
- 后端遇到未知 intrinsic 必须 hard fail（VM bytecode emit、JIT lowering、AOT cgen 三处统一）。
- 多态返回值（如 `Array.pop()` 可能返回 null）必须在 desc 中显式建模 return rep set，不允许在某个后端硬编码 `XR_REP_TAGGED`。

### 5.6 Backend target contract

> 现状：`src/jit/xm_target.h::XmTarget` 已经定义了寄存器清单（`gpr_alloc/fpr_alloc/ngpr_caller_save/...`）和帧布局（`frame_base/spill_base/max_spill_slots`），并提供 `xm_target_arm64`、`xm_target_x64` 两个实例。**缺的是**与 codegen pipeline 对接的函数指针，目前 ABI lowering、isel、emit 仍写死在 `xm_codegen_arm64.c / xm_codegen_x64.c` 各自分支里。

目标：在现有 `XmTarget` 结构上叠加 `XmTargetOps` 函数指针表：

```text
XmTargetOps                          // 新增：与 XmTarget 关联或合并
  lower_abi          (XmFunc*)        // 把抽象 call/ret 降到目标 ABI
  select_instr       (XmFunc*)        // 指令选择
  emit_func          (CodegenCtx*)    // 函数主体发射
  emit_final         (CodegenCtx*)    // 文字池/常量池/relocs
  safepoint_layout                    // safepoint frame slot 布局
  deopt_layout                        // deopt snapshot 布局
```

`XmTarget` 中已有的字段（`arg_regs/ret_regs/caller_save/callee_save/scratch_regs/stack_align/max_spill_slots`）保留，由 ops 函数读取，不再在 codegen 里散布常量。

所有 target 必须通过：

- frame layout verifier
- call ABI verifier
- safepoint verifier
- deopt map verifier
- suspend/resume frame verifier（受 `XR_INTRIN_*` 中 `may_suspend` 标志驱动）

---

## 6. 里程碑 0：测试真实性与 frontend binding 地基

### 6.1 目标

先让测试结果可信，并把 analyzer 到 lowerer 的 symbol/scope 输入变成可验证契约。该里程碑不追求重构所有 frontend 代码，但必须先建立能拦住假通过、漏访问、错 scope、未解析变量静默变 null 的质量门。

### 6.2 改动范围

- `src/app/cli/xcmd_test.c`
- `src/test/` 或当前 test runner 所在模块
- `scripts/run_regression_tests.sh`
- `src/frontend/analyzer/`
- `src/frontend/parser/xast_nodes*.h`
- `src/ir/xi_lower*.c`
- `tests/unit/frontend/`
- `tests/unit/cli/` 或 `tests/unit/test_runner/`
- `tests/regression/`

### 6.3 实施内容

1. 增加 `xray test` self-test：
   - 单文件 1 个 `@test` 必须报告 1 executed。
   - 多文件多个 `@test` 必须报告准确执行数。
   - 故意失败的 `@test` 必须返回非零。
   - 空测试文件必须报告 0 executed，不能计入 pass。
   - timeout 测试必须返回 timeout 状态。
2. 回归脚本必须校验执行数量：
   - 每个 regression 文件必须至少执行 1 个测试，除非显式标记为 compile-only。
   - 输出 summary 必须包含 files、tests executed、passed、failed、timeout。
   - `0 tests executed` 对普通 regression 文件是失败。
3. 建立 AST visitor 覆盖表：
   - 每个 AST 节点列出所有 child expression / child stmt / binding pattern 字段。
   - analyzer pass 2 必须访问所有需要 binding/type 的子节点。
   - visitor 只允许通过集中记录入口写入 `XaTypedInfo`。
   - 新增 AST 字段未登记时测试失败。
4. 建立 symbol binding invariant：
   - 声明节点、参数、解构 pattern、for/select/catch 绑定变量必须有 `symbol_id`。
   - 普通变量引用在 analyzer 后必须有非零 `symbol_id`。
   - 特殊变量（synthetic/import/builtin）必须显式标记来源，不能与未解析变量混用 `0`。
5. 建立 scope tree invariant：
   - Pass 1 collect 与 Pass 2 infer 对相同 AST 节点使用相同 scope。
   - 裸块、函数体、控制流 body、catch/select body 的 scope 策略必须有测试覆盖。
6. 建立 selection invariant：
   - member/method/index/module export 访问必须记录 receiver type、目标 symbol、field index、result type。
   - sealed JSON、class field、method call、module namespace 不再由 lowerer/backend 重新猜测。
7. 修改 lowerer：
   - 普通变量未解析时 hard fail。
   - `LOADNULL` fallback 只允许显式 null 语义路径。

### 6.4 验收标准

- `xray test` self-test 覆盖 pass、fail、timeout、0 tests、multi-file。
- 回归脚本能拒绝普通文件的 `0 tests executed`。
- AST visitor 覆盖测试能捕获遗漏 child expression 的 case。
- `XaTypedInfo` 至少覆盖 defs、uses、scopes、selections 四类 facts。
- shadowing、nested closure、destructure、for-in、ternary、template string、call arg、channel buffer size 都有 targeted regression。
- `ctest` 和至少一个 smoke regression bucket 都通过。

### 6.5 风险

| 风险 | 控制方式 |
|------|----------|
| 短期暴露大量真实失败 | 先建立 smoke bucket 与分类基线，不用假通过掩盖 |
| `symbol_id == 0` 当前被 synthetic 变量复用 | 增加 explicit binding kind，逐步替换裸 0 语义 |
| scope 修复影响 legacy-like 测试路径 | 以当前 Xi pipeline 为唯一权威路径，同时用 scope invariant 定位不一致 |

---

## 7. 里程碑 1：Xi stage 与 verifier 地基

### 7.1 目标

把 Xi 从“无状态 IR”升级为“状态明确、可验证、pass 可声明合约”的 IR pipeline。

### 7.2 改动范围

- `src/ir/xi.h`
- `src/ir/xi_verify.c/h`
- `src/ir/xi_pass.h`
- `src/ir/xi_opt.c`
- `src/ir/xi_pipeline.c/h`
- `src/ir/xi_dump.c`
- `tests/unit/ir/`（已有 `test_xi.c / test_xi_pipeline.c / test_xi_opt.c` 等，需新增 stage 专项）

### 7.3 实施内容

1. 新增 `XiStage`。
2. `XiFunc` 增加 `stage` 和 `invariant_mask`。
3. `xi_verify()` 拆成 stage-specific verifier：
   - `verify_raw`
   - `verify_canonical`
   - `verify_closed`
   - `verify_owned`
   - `verify_repped`
   - `verify_backend`
4. 扩展 `XiPassDesc`，声明输入/输出 stage。
5. 新增 `XiPassOrderConstraint` 表，pipeline 初始化时检查 pass 顺序合法性。
6. 增加环境变量：
   - `XRAY_XI_CHECK=1`：每个 pass 后 verify。
   - `XRAY_XI_DUMP=func:pass`：指定函数和 pass dump。
   - `XRAY_XI_STATS=1`：已有 stats 继续保留。
   - `XRAY_XI_PASS=pass:flag=value`：控制单个 pass 的 debug/dump/stats/enable。
7. `xi_func_dump()` 打印 stage、invariant mask、当前 pass 名称。
8. 扩展 `xi_verify()` 的低层结构检查：
   - CFG succ/pred 对称。
   - block kind/control/successor arity。
   - operand arity 和 aux payload。
   - phi 参数数量与 predecessor 数量。
   - value 唯一性和 block 回指。
   - effect flag 与 opcode table 一致。
9. `XRAY_XI_CHECK=1` 下提供可选 value order randomization，用于发现 pass 对 block 内 value 顺序的隐式依赖。

### 7.4 验收标准

- 所有现有 Xi lowering 后至少处于 `XI_STAGE_RAW`。
- 现有 VM pipeline 可以从 `XiRaw` 推进到 VM 可 emit 状态。
- 现有 AOT/JIT pipeline 可以拒绝错误 stage，而不是继续运行。
- 每个 pass 都声明 stage contract。
- pass order constraint 启动时通过。
- debug 构建每个 pass 后 verifier 通过。
- required pass 不能被关闭；unknown pass flag 必须报错。

### 7.5 风险

| 风险 | 控制方式 |
|------|----------|
| 初期 stage 划分过细导致大量改动 | 先实现 enum 和 verifier 框架，部分 stage 可以暂时等价通过 |
| pass contract 填错 | 启动时检查 pass order，测试覆盖所有 mode |
| dump 输出变化影响测试 | golden dump 测试统一更新 |

---

## 8. 里程碑 2：Typed AST canonicalizer

### 8.1 目标

在里程碑 0 的 binding/scope 契约基础上，把求值顺序、语法糖、部分结构化语义从 Xi lowering 前移，降低 lowerer 复杂度。

### 8.2 改动范围

- `src/frontend/parser/`
- `src/frontend/analyzer/`
- `src/frontend/codegen/` 中与 Xi lowering 入口相关部分
- 新增 `src/frontend/canonical/` 或放入 analyzer 后置 pass
- `tests/compile_errors/`
- `tests/frontend/`

### 8.3 实施内容

1. AST node 增加稳定 `node_id`。
2. analyzer 输出 typed node table，并包含 `node_id -> type/scope/symbol` 的 binding facts。
3. 新增 canonicalizer：
   - 固定函数调用参数求值顺序。
   - 对复杂 receiver、index、map key、call argument、channel send/recv value 引入临时变量。
   - 规定 index/map/channel receive 等带副作用或 runtime ABI 约束的表达式只出现在规范位置。
   - 展开复合赋值。
   - 展开 short-circuit 逻辑为显式控制流形态。
   - 规范化 `for-in`、`match`、`defer`、`try/finally`。
   - 规范化 top-level declaration 顺序。
   - 规范化 direct closure call：可直接调用的匿名函数在 canonical form 中显式传入 capture args。
4. parser 保持语法职责，避免符号/类型绑定逻辑继续扩散。
5. 统一 diagnostics 数据结构：
   - code
   - severity
   - primary range
   - secondary ranges
   - notes
   - fixits
   - related symbol

### 8.4 验收标准

- lowerer 不再处理基础语法糖展开。
- lowerer 不再负责求值顺序临时变量插入。
- parser 不依赖 analyzer。
- formatter/LSP 可以通过 `node_id` 稳定定位节点。
- compile error golden 保持稳定或更精确。
- 复合赋值、index getter/setter、method receiver、call args、channel send/recv 的副作用顺序有 targeted regression。

---

## 9. 里程碑 3：Module 与 closure metadata 一等化

### 9.1 目标

把 module import/export、live binding、closure capture、shared cell、child function 关系从后端路径和事后扫描中抽离出来，集中到 XiClosed 阶段的 metadata。`XiModule / XiCapture` 已存在但**生成方式不对**（事后扫 IR），本里程碑解决"在 lowering/close pass 时正确产出"的问题。

### 9.2 改动范围

- `src/ir/xi.h`（扩展 `XiModule`、新增 `XiClosureMeta`）
- `src/ir/xi_lower.c / xi_lower_*.c`
- `src/ir/xi_verify.c`
- `src/ir/xi_emit.c / xi_emit_cf.c`
- `src/jit/xi_to_xm.c`
- `src/aot/xi_cgen.c / xaot_driver.c`
- `src/ir/xi.c`（重写 `xi_module_populate_exports` 为 verifier）
- `src/module/`
- `src/vm/`
- `tests/regression/modules/`、`tests/aot/modules/`、`tests/coroutine_safety/`

### 9.3 实施内容

1. 在 `src/ir/xi.h` 扩展 `XiModule / XiModuleExport / XiModuleImport` 字段（见 §5.3）。
2. 新增 `XiClosureMeta`，把 `XiFunc.captures[]` 重构进去（见 §5.4）。
3. 新增 `xi_pass_close()`：
   - 解析 closure env layout 与 cell offset。
   - 消费 reassigned/address-taken/escape facts，决定 capture kind。
   - 把 mutable capture 改写为显式 `XI_CELL_LOAD / XI_CELL_STORE`。
   - 解析 module live binding cell index，回填到 `XiModuleExport`。
   - 生成 import/export cell index（与 closure cell 同空间）。
   - 生成 child function ownership 关系（`XiClosureMeta.parent_func`）。
   - 对 canonicalizer 已标记的 direct closure call，生成显式 capture args 调用形态。
4. `xi_module_populate_exports()` 降级为 verifier：仅断言 metadata 与 IR 一致，不再产出。
5. VM bytecode emitter 改为读取 metadata：
   - `emit_module_exports()` 改读 `XiModule.exports[]`，而不是 `XiFunc.export_names`。
   - 不在 emit 过程中重新决定 export 名字或 cell index。
6. JIT lowering (`xi_to_xm.c`) 改为读取 metadata：
   - GET_SHARED/SET_SHARED 通过 `XiModuleExport.cell_index` 解析。
   - 闭包 capture 通过 `XiClosureMeta.captures[i].env_offset` 解析。
   - 当前 `XI_IMPORT_REF -> xm_const_i64(0)` 占位降为正式 cell 引用。
7. AOT codegen (`xi_cgen.c`) 改为读取 metadata：
   - `cg_lookup_method` / `cg_lookup_class_ctor` 改为查 `XiModule.classes[]` / `exports[]`。
   - `xi_cgen_resolve_module_imports` 改为只填 `XiModuleImport.source_export` 指针，不再做名字匹配。
8. module linker 支持 SCC：
   - declaration instantiation
   - binding resolution
   - evaluation order（写入 `XiModule.scc_id`、`link_status`）

### 9.4 验收标准

- VM/JIT/AOT 对 module import/export 的结果一致（新增 diff bucket 验证）。
- 循环 import 的错误或行为由 module linker 统一决定。
- `xi_emit_cf.c::emit_module_exports` 只读 `XiModule.exports`，不再读 `XiFunc.export_names`。
- AOT `cg_lookup_method` 不再做名字+前缀的字符串拼接，只走 metadata 指针。
- closure mutable capture 在 VM/JIT/AOT diff 中一致。
- direct closure call 不分配 closure object，且与普通 closure 调用结果一致。
- `xi_module_populate_exports` 在 close pass 之后调用为 no-op（assert 已就绪）。

---

## 10. 里程碑 4：Intrinsic、builtin、runtime helper 合约表

### 10.1 目标

消除 VM/JIT/AOT 对 builtin/helper 的重复硬编码和静默失败路径。

### 10.2 改动范围

- `src/ir/`（新增 `xi_intrinsic.def` 或等价单一数据源）
- `src/jit/xm_intrinsic.h/c`（`XR_INTRIN_*` 改为生成）
- `src/aot/xrt_method_symbols.h`（`XRT_SYM_*` 改为生成）
- `src/runtime/symbol/xsymbol_table.h`（`SYMBOL_*` 改为生成）
- `src/frontend/analyzer/xanalyzer_builtins.c / xanalyzer_builtins_generated.h`
- `src/vm/`、`src/aot/xi_cgen.c`、`src/ir/xi_emit.c`
- `stdlib/`
- `tests/regression/builtins/`、`tests/aot/`、新增 `tests/unit/ir/test_xi_intrinsic.c`

### 10.3 实施内容

1. 新增 `xi_intrinsic.def`（X-Macro）作为唯一数据源，覆盖：
   - 当前 `XR_INTRIN_*`（30 项 JIT/AOT 桥）
   - 当前 `XRT_SYM_*`（70 项 method symbol）
   - 当前 analyzer builtins（length/push/...）
2. 从单一数据源生成：
   - `XR_INTRIN_*` enum（保留 ID 数值，不破坏现有 wire format）
   - `XRT_SYM_*` 与 `SYMBOL_*` 同步表（保留现有 `_Static_assert` 守住一致性）
   - name table、arity table、arg/return type+rep table、effect flags
   - VM/JIT/AOT 三套 lowering handler 表
3. 所有 builtin method 统一注册（`xanalyzer_builtins_generated.h` 改为从 def 生成）。
4. 所有 runtime helper（`xr_jit_*` / `xrt_*_sentinel`）统一注册。
5. 三处统一 hard fail：
   - VM `xi_emit_call_builtin` 遇到未注册 ID → emit error。
   - JIT `xi_to_xm.c` 处理 builtin call 时未知 ID → 拒绝 lower。
   - AOT `xi_cgen.c` 遇到未知 ID → 编译期报错。
6. polymorphic builtin（如 `Array.pop()` 可能返回 null）在 desc 中显式建模 return rep set。

### 10.4 验收标准

- 新增 builtin 只需改一个 `.def` 文件。
- 未注册 helper 无法通过编译（三处 hard fail）。
- VM/JIT/AOT builtin diff 覆盖 string、array、map、math、json、enum、regex。
- `_Static_assert` 守住 `XR_INTRIN_* / XRT_SYM_* / SYMBOL_*` 与 def 文件的一致性。
- 不再出现"某个 backend 消失一次 CALL_C / CALL_INTRINSIC"的情况。

---

## 11. 里程碑 5：Ownership、effect、lifetime 模型

### 11.1 目标

让 move/share/borrow、escape、retain/release、write barrier、safepoint 成为 IR 不变量，为安全并发和 AOT runtime 稳定性打地基。

### 11.2 改动范围

- `src/frontend/analyzer/`
- `src/ir/`
- `src/runtime/gc/`
- `src/aot/`
- `src/coro/`
- `tests/coroutine_safety/`
- `tests/aot/`

### 11.3 实施内容

1. Analyzer 输出 ownership summary。
2. Xi 增加 effect summary：
   - reads memory
   - writes memory
   - allocates
   - may throw
   - may suspend
   - may call user code
   - requires safepoint
3. 新增 XiOwned pass：
   - escape analysis
   - move check
   - capture isolation check
   - retain/release insertion 或标记
   - write barrier insertion 或标记
4. VM 路径可以保守解释 ownership op。
5. AOT 路径根据 ownership op 生成 ARC/runtime calls。
6. JIT 路径根据 effect/safepoint 生成 deopt/safepoint maps。

### 11.4 验收标准

- coroutine safety tests 由 analyzer + XiOwned 双层保护。
- AOT 容器/string/class 生命周期不再依赖“永不释放”假设。
- may-suspend 函数的 frame layout 由 verifier 检查。
- write barrier 不由后端自行猜测。

---

## 12. 里程碑 6：Representation 与 XiRepped 收敛

### 12.1 目标

让 int/float/bool/string/object/tagged 的 representation selection 成为稳定 stage，后端只消费显式 rep。`xi_opt_select_rep` + `xi_opt_box_elim` 已作为 opt-in pass 存在（`xi_pipeline.c::run_select_rep`，AOT 默认开），本里程碑把它们升级为 stage transition + verifier，并扩到 JIT。

### 12.2 改动范围

- `src/ir/xi_rep.h / xi.h`（`XiValue.rep` 字段如已有则补完，未有则新增）
- `src/ir/xi_opt.c`（`xi_opt_select_rep` / `xi_opt_box_elim` 升级为 stage 推进）
- `src/ir/xi_verify.c`（新增 `verify_repped`）
- `src/ir/xi_pipeline.c`（JIT 路径补上 select_rep）
- `src/jit/xi_to_xm.c`（消费显式 rep，去掉 type-kind 猜测）
- `src/aot/xi_cgen.c`
- `tests/unit/ir/test_xi_compare.c`、`tests/aot/`

### 12.3 实施内容

1. `xi_opt_select_rep` 从"可选 pass"升级为 stage transition：进入时必须 `XI_STAGE_OWNED`，退出时 `XI_STAGE_REPPED`。
2. 每个 XiValue 在 `XiRepped` 必须有稳定 rep（`XR_REP_I64 / F64 / BOOL / TAGGED / PTR_*`）。
3. BOX/UNBOX 边界由 `verify_repped` 检查（每条跨 rep 边都有显式 BOX/UNBOX）。
4. 多态 builtin 返回 rep 通过里程碑 4 的 intrinsic table 决定，不再依赖 `xi_to_xm.c` / `xi_cgen.c` 内的 hard-code。
5. 后端不再自行用 type kind 猜 rep；现有 `xi_to_xm.c` 中"按 `type->kind` 推 rep"的分支移除，改为读 `value->rep`。
6. `XiRepped` 后禁止高层 semantic op（`XI_PRINT / XI_GET_BUILTIN / XI_CALL_METHOD` 等）出现，必须在 close/own/rep 阶段降到 intrinsic call。

### 12.4 验收标准

- JIT 路径默认开 `select_rep`（与 AOT 对齐），可通过环境变量降级用于对比测试。
- VM/JIT/AOT 对 numeric/string/enum/builtin 结果一致。
- AOT 不再需要针对某些 helper 手工修正返回 rep。
- `xi_verify(repped)` 可以拒绝缺 rep、错 rep、漏 box/unbox 的 IR。
- `xi_opt_box_elim` 在 stage transition 后仍作为 peephole 运行，但不再回到 `XiOwned`。

---

## 13. 里程碑 7：Backend contract 与 codegen verifier

### 13.1 目标

统一 JIT target、AOT C emission、VM bytecode emission 的输入 contract，消除隐式 ABI 假设。

### 13.2 改动范围

- `src/jit/xm_target.h/c`（在现有 `XmTarget` 上叠加 `XmTargetOps`）
- `src/jit/xm_codegen_arm64.c / xm_codegen_x64.c / xm_codegen_stub.c`
- `src/jit/xm_regalloc.c`
- `src/jit/xm_intrinsic.h/c`（与里程碑 4 协同）
- `src/aot/`
- `src/vm/`
- `src/ir/xi_emit.c`
- `tests/jit/`、`tests/aot/`

### 13.3 实施内容

1. 在 `XmTarget` 上叠加 `XmTargetOps`（见 §5.6），把 codegen 中的 ABI/isel/emit 分支收回函数指针。
2. codegen 前运行 verifier：
   - call ABI
   - argument placement
   - return placement
   - spill slot bounds（与 `XmTarget.max_spill_slots` 比对）
   - scratch register usage
   - suspend/resume frame（受里程碑 4 中 `may_suspend` effect 标志驱动）
   - safepoint map
   - deopt snapshot
3. JIT IC feedback 作为 lowering 输入（当前 `XmICSnapshot` 已有 `ic_fields/ic_methods`，需扩展）：
   - field IC
   - method IC
   - class id
   - guard shape
   - deopt reason
4. huge method 策略：
   - value count 超阈值降级 opt level
   - block count 超阈值降级 opt level
   - budget 超时停止 pipeline（接 `XiPipelineConfig.budget_ns`）
5. AOT C-like IR 可选引入：
   - explicit locals
   - explicit calls
   - explicit control flow
   - explicit ownership ops

### 13.4 验收标准

- x64/ARM64 target contract 字段完整，`XmTargetOps` 全部 target 实现。
- spill/suspend/deopt 不再使用分散常量。
- JIT 编译失败必须有明确 reason，不静默 fallback。
- AOT helper emission 未识别时 hard fail。
- `XmICSnapshot` 通过 `xi_to_xm_lower` 一路传到 codegen，途中没有重新发现 IC。

---

## 14. 里程碑 8：死路径删除与目录收敛

### 14.1 目标

删除所有不再属于单一 Xi pipeline 的旧接口、fallback、半接线文件和重复实现。

### 14.2 清理原则

- 不保留 legacy builder。
- 不保留 bytecode 反建 SSA 路径。
- 不保留"未来可能会接"的 header。
- 不保留无调用方 API。
- 不保留双数据源表。
- 不保留静默 fallback。

### 14.3 已识别的具体靶子

| 靶子 | 现状 | 处理 |
|------|------|------|
| `src/jit/xm_blueprint.[ch]` | `xr_blueprint_generate()` 全工程零调用方；header 注释引用不存在的 `xr_compiler_end()`；`xm_codegen_stub.c:333` 因 `proto->blueprint==NULL` 永远跳过 OSR | 删除整个 blueprint 模块；OSR 入口直接读 `XmFunc` 上的 live 信息 |
| `XrProto.blueprint` 字段 | 仅在 `xchunk.c::165-170` 释放路径出现 | 删字段，更新 `xchunk.h` 注释 |
| `xchunk.h::299` 注释 | 引用 `xr_compiler_end()`（不存在） | 改为引用 `xi_emit` 的实际生成点 |
| `xi_module_populate_exports` | 里程碑 3 完成后从"产出"降级为"verifier" | 保留为 assert 工具或合并入 `xi_verify` |
| `XI_IMPORT_REF -> xm_const_i64(0)` 占位 | `xi_to_xm.c::828-830` 占位返回常数 0 | 里程碑 3 完成后改为正式 cell 引用，无需此分支 |
| `XiFunc.export_names` | 里程碑 3 完成后变为 lowering 内部缓冲 | 在 close pass 之后清零并断言不再被读 |
| `cg_lookup_method` 的字符串前缀拼接 | `xi_cgen.c::1023-1037` 通过 class_name + 模块前缀字符串组合查方法 | 里程碑 3 完成后由 metadata 指针替代 |

### 14.4 验收标准

- `src/ir/` 只保留 IR core、lowering、verification、analysis、optimization、bytecode emission、pipeline。
- `src/jit/` 只保留 Xi-to-Xm lowering、Xm IR、passes、regalloc、codegen、runtime、queue（无 `xm_blueprint*`）。
- `src/aot/` 拥有所有 AOT backend 和 AOT runtime 头。
- `scripts/check_architecture.sh` 通过。
- 无反向 include。
- `grep -rn "xr_blueprint\|xr_compiler_end" src/` 应只命中 `docs/archive/`，不应命中活代码或活注释。

---

## 15. 测试与验收矩阵

### 15.1 每次代码改动后

```bash
cmake --build build -j8
cd build && ctest --output-on-failure
```

如果改动涉及 test runner、analyzer、lowerer、emitter、VM/JIT/AOT 任一路径，还必须至少运行一个 targeted regression bucket，并确认输出中 tests executed 非 0。

### 15.2 每个里程碑完成后

```bash
scripts/check_architecture.sh
scripts/check_comment_rules.sh
scripts/run_regression_tests.sh
```

同时运行：

- test harness self-tests
- smoke regression bucket
- Xi verifier tests
- Xi compare tests
- VM/JIT diff tests
- VM/AOT diff tests
- module tests
- coroutine safety tests
- compile error golden tests
- AOT module tests
- JIT stress tests

### 15.3 新增测试类别

| 测试 | 目的 |
|------|------|
| `tests/unit/test_runner/` | 验证 `@test` discover、执行数量、失败退出码、timeout、0 tests 行为 |
| `tests/regression/smoke/` | 每次核心改动后快速确认真实语言行为，不依赖全量回归 |
| `tests/unit/frontend/test_ast_visit_coverage.c` | 每个 AST 节点的 child expression / binding pattern 都被 analyzer 覆盖 |
| `tests/unit/frontend/test_symbol_binding_invariants.c` | 声明和普通变量引用的 `symbol_id` 不变量 |
| `tests/unit/frontend/test_scope_tree_invariants.c` | Pass 1 / Pass 2 scope tree 一致性 |
| `tests/unit/frontend/test_selection_facts.c` | member/method/index/module 访问都生成 selection facts |
| `tests/regression/eval_order/` | receiver、index、map key、call args、channel send/recv 的求值顺序 |
| `tests/unit/ir/test_unresolved_variable.c` | lowerer 遇到未解析变量 hard fail，不生成 `LOADNULL` |
| `tests/unit/ir/stage/` | 每个 XiStage verifier 正确拒绝非法 IR |
| `tests/unit/ir/pass_order/` | pass 顺序约束测试 |
| `tests/unit/ir/pass_flags/` | required pass 不能关闭，unknown pass flag 报错，dump/stats/debug 开关可控 |
| `tests/unit/ir/value_order_randomization/` | verifier 模式下扰动 block 内 value 顺序，pass 结果仍稳定 |
| `tests/regression/modules/cycle/` | module SCC/link/eval 顺序 |
| `tests/regression/closure/` | mutable capture、nested closure、module live binding |
| `tests/regression/closure/direct_call/` | direct closure call 显式 capture args，不分配 closure object |
| `tests/unit/ir/test_xi_intrinsic.c` | intrinsic table 完整性和 unknown helper hard fail |
| `tests/regression/ownership/` | move/share/escape/lifetime（与现有 `coroutine_safety/` 合并） |
| `tests/jit/backend_contract/` | ABI、spill、deopt、safepoint verifier |
| `tests/unit/ir/test_xi_compare.c` 扩桶 | VM/JIT/AOT 三后端语义等价矩阵（已有该测试，扩展 diff bucket） |

### 15.4 质量门槛

- `.c` 文件不超过 3000 行。
- `.h` 文件不超过 800 行。
- 新增非 static 函数必须带 `XR_FUNC` 或 `XRAY_API`。
- 禁止新增可变文件作用域全局变量。
- 禁止直接 `malloc/free/calloc/realloc`。
- 禁止跨层 include。
- 普通 regression 文件执行 0 个 `@test` 必须失败。
- 普通变量 unresolved 必须 hard fail，禁止生成 `LOADNULL`。
- 所有 unknown backend lowering 都必须 hard fail。

---

## 16. 推荐执行顺序

推荐顺序如下：

```text
0. test harness truth + frontend binding invariant
1. Xi stage + verifier foundation
2. pass contract + per-pass verifier + dump/stats/check env
3. typed AST canonicalizer
4. module/closure metadata + XiClosed
5. intrinsic/effect table
6. ownership/effect/lifetime + XiOwned
7. representation stage + XiRepped
8. backend contract verifier
9. dead path deletion and directory cleanup
10. full regression and benchmark stabilization
```

原因：

- 测试真实性和 binding invariant 是判断后续改动是否正确的前提。
- stage/verifier 是所有后续改动的地基。
- canonicalizer 必须建立在 analyzer binding/scope 已可信的基础上，才能降低 lowerer 复杂度。
- module/closure 必须早于 ownership，因为 capture/cell/live binding 会影响 escape/lifetime。
- intrinsic/effect 必须早于 representation，因为返回 rep 和 side effect 会影响 select_rep。
- backend contract 应在 IR 语义稳定后收敛，否则会反复返工。

---

## 17. 不做事项

本方案明确不做：

- 不维护 legacy compiler fallback。
- 不维护 bytecode -> SSA 反建 JIT 路径。
- 不为旧 AOT 反扫 proto 逻辑保留兼容。
- 不允许 VM/JIT/AOT 各自定义 builtin 语义。
- 不把 module linking 留给后端。
- 不把 ownership/lifetime 留给 AOT runtime 猜测。
- 不引入 LLVM 级别的大型 target framework。
- 不做增量编译缓存，直到 module/export summary 稳定后再评估。

---

## 18. 最终状态

完成后，Xray 编译器应达到如下状态：

```text
Frontend:
  parser 只负责语法；analyzer 输出 XaTypedInfo；canonicalizer 只负责规范化。
  defs/uses/scopes/selections/implicits/instances 是 frontend 到 Xi 的正式输入。

Xi:
  每个函数有明确 stage；每个 stage 有 verifier；每个 pass 有 contract。
  pass order、required pass、debug/dump/stats/check 由统一 pipeline 管理。

Module/Closure:
  import/export/live binding/capture/cell/layout 都是一等 metadata。
  capture kind、env offset、cell index、direct closure call 形态在 XiClosed 固定。

Intrinsic/Effect:
  builtin/helper/runtime call 都由单一数据源定义，所有后端共享。

Ownership/Rep:
  lifetime、write barrier、safepoint、BOX/UNBOX、value rep 都是显式 IR 事实。

Backend:
  VM 是语义参考；JIT/AOT 是优化后端；三者不重复解释语言语义。

Testing:
  test harness self-test、frontend facts invariant、IR invariant、pass order、多后端 diff、backend contract 共同守住正确性。
```

这套架构能直接支撑后续：

- 更强的 JIT speculative optimization
- 更稳的 AOT standalone runtime
- module/package 编译
- LSP incremental semantic index
- coroutine safety 深化
- ownership/move model 深化
- 更可靠的跨后端性能优化
