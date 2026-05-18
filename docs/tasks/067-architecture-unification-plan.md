# 067 — Xray 架构统一重构方案

> **状态**：planned
> **目标受众**：编译器、运行时、工具链核心开发者
> **设计原则**：不考虑向后兼容性，直接采用最佳设计，完成一块删除一块旧路径
> **核心目标**：让 XIR Typed SSA 成为唯一官方 IR，消除后端侧语义推断，删除 JIT bytecode→SSA builder

---

## 1. 总目标

Xray 的目标架构：

```text
Source
  ↓
Frontend (parser + analyzer)
  ↓
XIR Typed SSA
  ↓
┌──────────────┬──────────────┬──────────────┐
│ VM Bytecode  │ AOT Native   │ JIT Native   │
└──────────────┴──────────────┴──────────────┘
  ↓
Runtime Core
```

核心结论：

- **唯一官方 IR**：XIR Typed SSA（当前代码中 `Xi` 前缀，JIT builder 删除后重命名为 `Xir`）。
- **VM bytecode**：执行格式，不是语义真相源。
- **AOT C/native**：生产后端，优先保证性能和体积。
- **JIT machine code**：开发期热函数加速后端。
- **Analyzer**：类型、符号、诊断的权威源（现有 `XaAnalyzer` 即是，无需额外抽象层）。
- **Runtime Contract**：显式 value layout + ownership domain + GC stack map。

每个里程碑完成后，直接删除旧路径，不保留兼容层。

---

## 2. 当前基线

已完成的收敛：

- `--native` 是唯一 AOT 入口，legacy C codegen 已删除。
- AOT driver 走 Xi IR pipeline（`xi_pipeline_compile_program`）。
- `xcompiler.c` 中所有编译统一走 Xi IR pipeline 生成 bytecode。
- AOT 不再使用旧 JIT XIR 的 C codegen 路径。

仍需收敛的核心问题（按优先级排序）：

1. **AOT cgen 全局状态**：`g_imports[256]`、`g_shared_funcs[512]`、`g_methods[256]` 都是 static 全局变量，不支持并行编译。
2. **AOT driver 扫描 IR 推断语义**：遍历 IR blocks 找 `XI_SET_SHARED + XI_CLASS_CREATE` 来推断 class metadata。
3. **JIT bytecode→SSA builder**：~400KB 代码（`xir_builder*.c/h`）从 bytecode 逆向重建 SSA，独立推导类型/rep/tag。
4. **JIT builder 死代码**：`aot_mode` 字段永远为 false，~30 个 dead branch。
5. **`Xi` vs `Xir` 命名冲突**：`src/ir/xi_*` 与 `src/jit/xir_*` 共存。

---

## 3. 目录结构与依赖方向

保持现有目录结构，不做大规模移动。`src/ir/` 统一持有整个 Xi pipeline（lowering、optimization、emit、cgen），因为这些模块紧密耦合共享 `XiFunc/XiBlock/XiValue` 数据结构。

```text
src/base/          L0  allocator, arena, vec, hash, platform
src/runtime/       L1-L3  value, type, gc, object, class, symbol, coro
src/frontend/      L4  lexer, parser, analyzer, codegen, format
src/module/        L4  bundle, module graph
src/ir/            L4  xi.h, xi_lower, xi_opt, xi_verify, xi_emit, xi_cgen, xi_pipeline
src/jit/           L5  JIT machine code generation (x64/arm64)
src/aot/           L6  AOT driver + standalone runtime headers (xrt_*.h)
src/api/           L7  embedding/isolate/compiler public API
src/app/           L8  cli, lsp, dap
```

依赖方向（低层不依赖高层）：

```text
L0 base
  → L1 runtime/value, runtime/type
  → L2 runtime/gc, runtime/object
  → L3 runtime/class, runtime/symbol, runtime/coro
  → L4 frontend, module, ir
  → L5 jit
  → L6 aot
  → L7 api
  → L8 app
```

边界铁律：

- `src/ir/` 不 include `src/jit/` 或 `src/aot/`（方向：ir → 被 jit/aot 消费）。
- `src/jit/` 和 `src/aot/` 不反向修改 IR 语义。
- `src/runtime/` 不依赖 frontend、module、ir、jit、aot、app。
- Frontend 不知道 VM/AOT/JIT 的存在。

---

## 4. IR 策略

### 4.1 唯一官方 IR

当前 `src/ir/xi.h` 中的 Xi IR 即是项目唯一 IR（JIT builder 删除后重命名为 XIR）。

XIR 承担：

- **类型权威**：每个 value 持有 `XrType*`。
- **表示权威**：每个 value 持有 `XrRep`。
- **GC 权威**：每个 value 可派生 `XrSlotType`。
- **控制流权威**：block、phi、terminator 显式化。
- **模块权威**：import/export/class metadata 挂在 `XirModule`（待新增）。
- **优化权威**：所有通用优化 pass 在 XIR 上执行。

### 4.2 不是官方 IR 的内容

- **VM bytecode**：执行格式（`xi_emit.c` 产出）。
- **AOT C**：目标输出（`xi_cgen.c` 产出）。
- **JIT machine code**：目标输出（`xir_codegen_arm64.c` / `xir_codegen_x64.c` 产出）。
- **JIT internal SSA**：JIT 内部 lowering form（当前 `XirFunc`，长期目标删除 builder 后直接从 Xi IR 降低）。
- **AST**：frontend representation。

### 4.3 JIT 路径演进

当前路径（要删除）：

```text
bytecode → xir_build_from_proto() → JIT XirFunc → regalloc → machine code
```

目标路径：

```text
XIR metadata snapshot + IC feedback → JIT lowering → regalloc → machine code
```

---

## 5. 里程碑总览（按执行顺序）

```text
M1. 清理死代码：删除 aot_mode + dead branches
M2. AOT metadata 化：XirModule + 去全局状态 + 去 IR 扫描
M3. Pipeline 增强：统一 VM/AOT pass 配置
M4. JIT 从 XIR 接入：删除 bytecode→old XIR builder（最高价值，最高难度）
M5. 命名收敛：Xi → XIR（在 M4 删除 JIT builder 后执行）
M6. 架构强约束：include lint + value layout assert + pass verifier
```

**已删除的里程碑**（判断为过度工程）：
- ~~SemanticModel 形式化~~：现有 `XaAnalyzer` 直接作为语义源已足够，无需额外只读 wrapper。
- ~~工具层服务化~~：现有 CLI/LSP/DAP 分层已合理（`xcli_spec` + `xcli_dispatch` + 各命令文件），不需要 CompilerService/WorkspaceService 抽象。
- ~~Runtime value 全面统一~~：AOT 与 VM 的 value layout 分歧是有意设计（AOT 无 GC header），缩小为 `_Static_assert` 对齐检查。

---

## 6. M1：清理死代码

### 6.1 目标

删除 JIT builder 中所有 AOT 死代码路径。风险极低，净减 ~200 行。

### 6.2 动作

1. 删除 `XirBuilder.aot_mode` 字段。
2. 删除所有 `if (b->aot_mode)` 分支（~30 处，分布在 `xir_builder_call.c`、`xir_builder_misc.c`、`xir_builder_object.c`）。
3. 删除 JIT builder 中仍提到 AOT 的注释。
4. 删除 `xir_build_from_proto_ex` 中 AOT 相关参数（如已无调用者）。

### 6.3 验收

```bash
grep -R "aot_mode" src/jit/  # 应为 0 结果
ctest --output-on-failure
scripts/run_regression_tests.sh
```

---

## 7. M2：AOT metadata 化

### 7.1 目标

消除 AOT 后端的全局状态和 IR 扫描推断。AOT driver 和 C backend 只消费显式 metadata。

当前问题（`xi_cgen.c` 实际代码）：

```c
// 全局状态 — 不支持并行、不支持复用
static CgImportEntry g_imports[CG_MAX_IMPORTS];
static int g_nimports;
static const XiFunc *g_shared_funcs[CG_MAX_SHARED];
static const XiClassData *g_shared_class[CG_MAX_SHARED];
static CgMethodEntry g_methods[CG_MAX_METHODS];
```

```c
// AOT driver 扫描 IR 推断语义（xaot_driver.c:300-315）
// 遍历 IR blocks 找 XI_SET_SHARED + XI_CLASS_CREATE pattern
```

### 7.2 新增 XiModule 结构

在 `src/ir/xi.h` 中新增（当前命名保持 `Xi` 前缀，M5 统一改名）：

```c
typedef struct XiModule {
    const char *path;           /* source path (e.g. "./math_lib") */
    const char *name;           /* C identifier (e.g. "math_lib") */
    XiFunc *init;               /* module init function (top-level) */
    XiFunc **functions;         /* all top-level functions */
    uint16_t nfuncs;
    XiClassData **classes;      /* all class descriptors */
    uint16_t nclasses;
    XiModuleExport *exports;    /* explicit export table */
    uint16_t nexports;
    XiModuleImport *imports;    /* explicit import table */
    uint16_t nimports;
} XiModule;

typedef struct XiModuleExport {
    const char *name;
    uint16_t shared_slot;
    XiFunc *function;           /* NULL if not a function export */
    XiClassData *class_data;    /* NULL if not a class export */
} XiModuleExport;

typedef struct XiModuleImport {
    const char *module_path;
    const char *member_name;
    XiModuleExport *resolved;   /* resolved after module graph linking */
} XiModuleImport;
```

### 7.3 去全局化 C backend

将 `xi_cgen.c` 的全局状态转为显式 context：

```c
typedef struct XiCgenCtx {
    FILE *out;
    const XiModule *module;         /* current module being emitted */
    const XiModule **all_modules;   /* full module graph (for import resolution) */
    uint16_t nmodules;
    /* Method registry (per-compilation, not global) */
    CgMethodEntry methods[CG_MAX_METHODS];
    int nmethod;
    /* Shared slot mapping (per-module) */
    const XiFunc *shared_funcs[CG_MAX_SHARED];
    const XiClassData *shared_class[CG_MAX_SHARED];
    int nshared;
    /* C function name counter */
    int fname_counter;
} XiCgenCtx;
```

所有 `xi_cgen_*` 函数改为接收 `XiCgenCtx *ctx` 参数。删除 `xi_cgen_reset_imports()` 和 `xi_cgen_add_import()`。

### 7.4 去 IR 扫描

AOT driver 中的 `XI_SET_SHARED + XI_CLASS_CREATE` 扫描逻辑移到 lowering 阶段：

1. `xi_lower.c` 在 lowering class 声明时，直接将 `XiClassData` 挂到 `XiModule.classes[]`。
2. `xi_lower.c` 在 lowering `SET_SHARED` 时，直接填充 `XiModule.exports[]`。
3. AOT driver 只做：module graph linking（resolve imports → exports）+ orchestration。

### 7.5 动作清单

1. 新增 `XiModule` / `XiModuleExport` / `XiModuleImport` 到 `xi.h`。
2. `xi_pipeline_compile_program` 返回 `XiModule*` 而非裸 `XiFunc*`。
3. `xi_lower.c` 填充 module exports/classes。
4. 新增 `XiCgenCtx`，重构 `xi_cgen_func` / `xi_cgen_program` / `xi_cgen_module` 签名。
5. `xaot_driver.c` 改为：resolve module graph → pass `XiModule[]` 到 cgen。
6. 删除 `xi_cgen_reset_imports()` / `xi_cgen_add_import()` / `g_imports[]` 全局。
7. 删除 AOT driver 的 IR block 扫描逻辑。

### 7.6 验收

```bash
ctest --output-on-failure
tests/aot/run_aot_tests.sh       # 单模块 + 多模块 + class import
grep -R "g_imports\|g_nimports\|xi_cgen_reset_imports\|xi_cgen_add_import" src/ir/  # 应为 0
```

---

## 8. M3：Pipeline 增强

### 8.1 目标

扩展现有 `XiPipelineConfig` 以区分 VM/AOT 模式，让同一 pipeline 按 mode 选择 pass 序列。

当前 `xi_pipeline.h` 已有基础框架（`run_verify`、`run_optimize`、`run_select_rep`），只需增量扩展。

### 8.2 扩展 pipeline config

```c
typedef enum XiPipelineMode {
    XI_PIPE_VM,      /* lower → verify → light opt → bytecode emit */
    XI_PIPE_AOT,     /* lower → verify → full opt → select_rep → C emit */
    XI_PIPE_CHECK,   /* lower → verify only (for LSP/check command) */
    XI_PIPE_DUMP,    /* lower → verify → dump (for --dump-ir) */
} XiPipelineMode;

/* Extend existing XiPipelineConfig */
typedef struct XiPipelineConfig {
    XiPipelineMode mode;
    bool run_verify;
    bool run_optimize;
    bool run_select_rep;
    bool run_box_elim;
    bool dump_each_pass;
} XiPipelineConfig;
```

### 8.3 后端模式 pass 序列

```text
VM:   lower → verify → light opt (copy_prop + dce) → bytecode emit
AOT:  lower → verify → full opt → select_rep → box_elim → C emit
CHECK: lower → verify (no emit)
```

### 8.4 pipeline 返回 XiModule

```c
typedef struct XiPipelineResult {
    XiPipelineStatus status;
    const char *error_msg;
    XiModule *module;    /* populated with exports/imports/classes */
    XrProto *proto;      /* VM bytecode (NULL in AOT/CHECK mode) */
} XiPipelineResult;
```

### 8.5 验收

- VM 和 AOT 使用同一 pipeline 入口，只切换 mode。
- debug build 每个 pass 后自动运行 verifier。
- 全量测试通过。

---

## 9. M4：JIT 从 XIR 接入

这是**最高价值**的里程碑（删除 ~400KB builder 代码），也是**最高难度**的。

### 9.1 目标

删除 `bytecode → xir_build_from_proto() → JIT XirFunc → machine code` 路径，让 JIT 直接消费 Xi IR 派生的信息。

### 9.2 核心难点

当前 JIT builder 除了将 bytecode 转为 SSA 外，还负责：

1. **Deopt 信息生成**：每个 potential deopt point 记录 bytecode PC + 所有 live value 的 slot→vreg 映射，用于 JIT→VM 退出时重建帧。
2. **IC 反馈集成**：从 proto 的 `type_feedback` snapshot 读取 inline cache 数据，驱动投机类型特化。
3. **Resume entry 生成**：coroutine suspend→resume 从特定 bytecode PC 恢复执行。
4. **OSR entry**：loop 中从 VM 跳入 JIT 的 on-stack-replacement。

这些能力在新路径中必须保留。

### 9.3 新架构：XiProtoMeta

编译时在 Xi pipeline 中保存 compact metadata（附着在 `XrProto` 上）：

```c
typedef struct XiProtoMeta {
    /* Type information (from analyzer) */
    XrType **param_types;       /* parameter types */
    uint8_t nparam;
    XrType *return_type;

    /* SSA value → bytecode slot mapping (for deopt frame reconstruction) */
    XiSlotMap *slot_map;
    uint16_t nslots;

    /* Shared/upvalue metadata */
    uint16_t *shared_slots;     /* which shared slots this function accesses */
    uint8_t nshared;

    /* Module context (for cross-module calls) */
    const XiModule *module;

    /* Source location table (for debug info) */
    uint32_t source_start;
    uint32_t source_end;
} XiProtoMeta;
```

### 9.4 渐进迁移策略

#### 9.4.1 让 builder 从 XiProtoMeta 读取类型信息

先不删 builder，切断其类型推导：
- 参数类型、局部变量类型从 `XiProtoMeta` 读取（不再自己从 bytecode 猜）。
- IC feedback 仍由 runtime 提供，但只做优化 hint，不决定正确性。

#### 9.4.2 新增 Xi → JIT lowering path

逐步将 Xi IR lowering 为 JIT 内部 `XirFunc`：

```text
XiFunc (from XiProtoMeta or re-lower)
  → xi_to_jit_lower()
  → JIT XirFunc (same register allocator + codegen)
  → machine code
```

关键：复用现有的 regalloc (`xir_regalloc.c`) 和 codegen (`xir_codegen_arm64.c` / `xir_codegen_x64.c`)。只替换 SSA construction 阶段。

覆盖顺序（按热路径频率）：
1. arithmetic / comparison / branch / phi
2. function call / return
3. property access / method call
4. closure / upvalue
5. array / map operations
6. exception handling
7. coroutine suspend/resume

#### 9.4.3 双路径验证

同一热函数同时比较 old builder 和 new Xi lowering 的输出：

```text
old: bytecode → xir_build_from_proto_jit() → XirFunc → machine code → output
new: XiProtoMeta → xi_to_jit_lower() → XirFunc → machine code → output
```

Output 必须完全一致。

#### 9.4.4 删除 old builder

覆盖率达到 100% 后删除：

```text
src/jit/xir_builder.c          (~3000 lines)
src/jit/xir_builder.h
src/jit/xir_builder_call.c     (~1500 lines)
src/jit/xir_builder_misc.c     (~2000 lines)
src/jit/xir_builder_object.c   (~1000 lines)
src/jit/xir_builder_internal.h
```

### 9.5 Deopt/Resume 策略

Deopt 信息在新路径中的生成方式：

```text
Xi IR value → 对应 bytecode slot (通过 XiProtoMeta.slot_map)
JIT compile 时生成 deopt descriptor:
  { machine_pc, [(vreg → bytecode_slot, rep)] }
Deopt 触发时:
  从 machine state 重建 VM CallFrame
```

Resume entry 策略：
- 协程 suspend point 在 Xi IR 中有显式 `XI_YIELD` / `XI_AWAIT` 标记。
- 编译时为每个 suspend point 生成 resume entry address。
- Resume 时跳入对应 machine code 位置。

### 9.6 验收

```bash
# JIT E2E 测试
ctest -R jit --output-on-failure
# VM/JIT output diff
scripts/run_regression_tests.sh --jit
# builder 删除后
grep -R "xir_build_from_proto" src/jit/  # 应为 0
grep -R "XirBuilder" src/jit/            # 应为 0
```

---

## 10. M5：命名收敛（Xi → XIR）

### 10.1 前提

M4 完成后 JIT builder 已删除，`src/jit/xir_*` 不再存在命名冲突。此时可以安全地将 `Xi` 前缀统一为 `Xir`。

### 10.2 重命名映射

| 当前 | 目标 | 说明 |
|---|---|---|
| `XiFunc` | `XirFunc` | 函数级 SSA |
| `XiValue` | `XirValue` | SSA value/instruction |
| `XiBlock` | `XirBlock` | CFG basic block |
| `XiOp` | `XirOp` | IR operation enum |
| `XI_ADD` | `XIR_ADD` | op 命名 |
| `XiModule` | `XirModule` | module metadata |
| `xi_pipeline_*` | `xir_pipeline_*` | pipeline API |

文件重命名：

```text
src/ir/xi.h         → src/ir/xir.h
src/ir/xi.c         → src/ir/xir.c
src/ir/xi_lower.c   → src/ir/xir_lower.c
src/ir/xi_opt.c     → src/ir/xir_opt.c
src/ir/xi_verify.c  → src/ir/xir_verify.c
src/ir/xi_emit.c    → src/ir/xir_emit.c
src/ir/xi_cgen.c    → src/ir/xir_cgen.c
src/ir/xi_cgen.h    → src/ir/xir_cgen.h
src/ir/xi_dump.c    → src/ir/xir_dump.c
src/ir/xi_pipeline.h → src/ir/xir_pipeline.h
src/ir/xi_pipeline.c → src/ir/xir_pipeline.c
src/ir/xi_rep.h     → src/ir/xir_rep.h
```

### 10.3 执行原则

- 纯机械 rename，**不混入逻辑改动**。
- 单次 git commit，方便 blame 跳过。
- 加入 `.git-blame-ignore-revs`。

### 10.4 验收

```bash
grep -rn "XiFunc\|XiValue\|XiBlock\|XiOp\b" src/ir/ src/aot/ src/frontend/  # 应为 0
ctest --output-on-failure
tests/aot/run_aot_tests.sh
```

---

## 11. M6：架构强约束

### 11.1 Include 层级检查

增强 `scripts/check_architecture.sh`：

```text
runtime 不能 include frontend/module/ir/jit/aot/app
frontend 不能 include jit/aot/app
ir 不能 include jit/aot/app
jit/aot 可以 include ir，但不能反向修改 IR 语义
```

### 11.2 Dead path 检查

CI 禁止以下 pattern 回归：

```bash
grep -R "aot_mode" src/jit/                    # 必须为 0
grep -R "xir_build_from_proto" src/jit/        # 必须为 0（M4 完成后）
grep -R "xi_cgen_reset_imports\|xi_cgen_add_import" src/  # 必须为 0（M2 完成后）
```

### 11.3 Value layout drift 检查

保留并增强 `xrt_value_check.c` 中的 `_Static_assert`：

- `sizeof(XrValue)` == 16（VM 和 AOT 一致）。
- Tags 0-7 数值在 VM 和 AOT 中完全相同。
- Payload offset 一致。
- Boxing/unboxing 语义一致（`XR_INT`/`XR_FLOAT`/`XR_BOOL` 的 bit pattern）。

AOT 独有的 tags 8+（`XR_TAG_STR=14`、`XR_TAG_ARRAY=15` 等）是合法的设计分歧：AOT standalone 无 GC header，需要在 tag 中编码对象类型。这不需要"统一"，但需要 assert 基础 schema 不 drift。

### 11.4 Pass verifier

debug build 每个 pass 后可选运行：

```text
xi_verify       — CFG well-formedness
xi_verify_type  — every value has valid XrType*
xi_verify_rep   — XrRep consistent with type
```

### 11.5 验收

- `scripts/check_architecture.sh` 在 CI 中运行，违反则 fail。
- value layout `_Static_assert` 在 AOT 编译时自动检查。

---

## 12. 删除清单

全部里程碑完成后，以下内容应已被删除：

| 删除项 | 在哪个 M 删除 |
|---|---|
| `XirBuilder.aot_mode` + ~30 dead branches | M1 |
| `xi_cgen_reset_imports()` / `xi_cgen_add_import()` / `g_imports[]` | M2 |
| AOT driver 的 IR block 扫描推断逻辑 | M2 |
| `xir_builder*.c` / `xir_builder*.h`（~400KB） | M4 |
| `xir_build_from_proto*` 函数族 | M4 |
| old `XirTypeKind` / `VTAG` / JIT 独立类型推导 | M4 |
| `Xi` 前缀（全部改为 `Xir`） | M5 |

**不删除**（有意保留）：
- `src/aot/xrt_value.h`：AOT standalone value layout（tags 8+ 是合法设计）。
- `src/jit/xir_codegen_*.c`：machine code generation（只删 SSA builder，不删 codegen）。
- `src/jit/xir_regalloc.c`：register allocator（复用）。

---

## 13. 验收矩阵

每个里程碑至少运行：

```bash
ctest --output-on-failure
scripts/run_regression_tests.sh
tests/aot/run_aot_tests.sh
scripts/check_architecture.sh
```

涉及 JIT 的改动（M1, M4）还需要：

```bash
ctest -R jit --output-on-failure
# ARM64 + x64 JIT E2E
# VM/JIT output diff
```

涉及 AOT 的改动（M2, M3）还需要：

```bash
tests/aot/run_aot_tests.sh     # 单模块 + 多模块 + class import
# VM/AOT output diff 必须为 0
```

ASAN 验证（M2, M4）：

```bash
cmake -DCMAKE_BUILD_TYPE=Debug -DENABLE_ASAN=ON ..
ctest --output-on-failure
```

---

## 14. 风险与缓解

| 风险 | 影响 | 缓解 |
|---|---|---|
| M2 AOT metadata 上移破坏多模块 | import/class/export 输出不一致 | 保留 VM/AOT diff 测试，先加 `XiModule` 再删扫描逻辑 |
| M4 JIT 切到 XIR 成本大 | JIT 短期不可用 | 双路径对比 → 100% 覆盖后才删除旧路径 |
| M4 deopt/resume 语义复杂 | JIT 正确性回归 | 先让 builder 从 metadata 读类型（渐进切断） |
| M5 改名范围过大 | 容易混入逻辑改动 | 纯机械 rename 单独 commit，加入 blame-ignore |

---

## 15. 最终完成标准

所有里程碑完成后：

- Xi IR（重命名后 XIR）是唯一官方 IR。
- VM bytecode、AOT C、JIT machine code 都是 XIR 后端输出。
- Backend 不自行推断语言语义。
- JIT 不从 bytecode 反推 SSA。
- AOT driver 不扫描 IR 猜 import/export/class metadata。
- AOT cgen 无全局状态。
- 架构 lint 能阻止旧路径回归。
- Value layout 基础 schema 有 `_Static_assert` 保证不 drift。

一句话：

> XIR 是所有语义的唯一载体；所有后端只负责 lowering 与 emission。
