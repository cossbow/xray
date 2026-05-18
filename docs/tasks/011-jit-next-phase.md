# JIT 模块下一阶段实施方案

**状态**：基于 `jit_refactoring_plan.md`（已大部分完成）的后续演进
**原则**：不考虑向后兼容，直接采用最佳设计，每阶段独立可合并

## 现状评估

`jit_refactoring_plan.md` 中的基础设施已基本落地：
- ✅ DomTree / LoopTree / DefUse / Alias — 缓存在 `XirFunc` 上
- ✅ SCCP — `xir_pass_sccp.c` 已实现
- ✅ FixedPoint pipeline 驱动器 — `xir_run_fixedpoint` 已实现
- ✅ Coalescing / Peephole — `xir_coalesce.c`, `xir_peephole.c` 已实现
- ✅ BgTask snapshot — `XirBgTask` 在主线程拷贝 feedback/shared_protos
- ✅ Eligibility 独立 — `xir_eligibility.c/h` 已拆出
- ✅ Tiered compile — Tier 1 (O1) → Tier 2 (O2) recompile 路径已存在

**本文档聚焦尚未解决的 6 个维度**，按优先级排列。

---

## 总则

### 开发原则
1. ✅ 直接采用最佳设计，不保留旧接口 / 不做兼容层
2. ✅ 每阶段独立可合并、独立可回滚
3. ✅ 每次代码改动后必须运行测试
4. ❌ 禁止保留旧版代码作"快速回退"，必须直接替换

### 验证基线
```bash
# 快验
cmake --build build -j8
cd build && ctest --output-on-failure
# 完整回归
./scripts/run_regression_tests.sh
# JIT 差分测试
./scripts/jit_diff_test.sh  # 解释器 vs JIT 输出对比
```

### 交付节奏
| 阶段 | 规模 | 风险 | 预估 | 关键产出 |
|------|------|------|------|----------|
| A. 正确性修复 | 3 处核心改动 | 低 | 1 d | 消灭 2 个 P0 bug |
| B. 并发原语 JIT 化 | 5 个 opcode | 中 | 3 d | 并发函数可 JIT |
| C. Tiered Compilation 完善 | 2 处改动 | 中 | 1.5 d | 自适应优化 |
| D. 代码质量提升 | 4 处改动 | 低 | 2 d | 消除硬限制 + peephole |
| E. Code Cache 管理 | 新增 1 文件 | 中 | 1.5 d | 长期运行稳定性 |
| F. x86-64 后端 | 新增 4 文件 | 高 | 7 d | 跨平台 JIT |

**合计**：约 16 个工作日

---

## 阶段 A：正确性修复（P0）

**动机**：`known_failures.txt` 中 2 个 P0 bug + `deopt_reconstruct` 的地址启发式
必须在所有功能扩展之前修复，否则后续改动的测试基线不可信。

### A.1 嵌套 Deopt 级联

**问题**：JIT 函数 A 调用 JIT 函数 B，B deopt 返回 `XIR_DEOPT_MARKER`，
A 将其当作正常返回值处理 → SIGSEGV。影响 `--jit-force` 模式下 ~17 个测试。

**根因**：`xir_jit_call` 正确检测了 `DEOPT_MARKER`，但 JIT→JIT 的 `CALL_KNOWN`
快速路径直接读取 x0 返回值，不检查是否为 marker。

**修复方案**：

在 codegen 中，所有 `CALL_KNOWN` / `CALL_DIRECT` 的 fast path 返回后，
插入 marker 检测：

```c
// xir_codegen_call.c — emit_call_known() 返回后新增：
//   CMP x0, #DEOPT_MARKER_LO     (低 32 位)
//   CCMP x0, #DEOPT_MARKER_HI, ... (高 32 位)
//   B.EQ  → caller's deopt stub
// 或者更简洁：
//   load DEOPT_MARKER into scratch reg (Xtmp)
//   CMP x0, Xtmp
//   B.EQ → cascade_deopt
```

**cascade_deopt stub** 行为：
1. 将被调用函数的 deopt 信息传递给调用方
2. 调用方自身也 deopt 回解释器
3. 解释器恢复后重新解释执行该 CALL 指令

**代价**：每个 CALL_KNOWN 后增加 3 条指令（~12 字节），对热路径影响可忽略。

**验收**：
- `tests/jit/known_failures.txt` 中两个文件移除
- `--jit-force` 模式下 JIT diff CRASH 数归零

### A.2 消除 Deopt Reconstruct 地址启发式

**问题**：`xir_jit_internal.h` 的 `deopt_reconstruct()` 和
`xir_jit_read_multi_ret()` 中存在 `tagged_heuristic` 代码路径：

```c
// 当前危险代码：
if (raw != 0 && (raw & 0x7) == 0 && (uint64_t)raw > 0x10000) {
    v.tag = XR_TAG_PTR;  // 可能误判！
}
```

一个恰好对齐且大于 0x10000 的整数值会被误判为指针 → GC 崩溃。

**修复方案**：

**原则：所有 deopt slot 必须携带精确 `xr_tag`**

1. **Builder 侧**：`xir_builder.c` 在生成 DEOPT_INFO 时，
   确保所有 slot 都设置 `xr_tag`（来源：vreg 的 `vtag` 字段、
   PHI 节点的 merge 类型、或 CALL_C 返回类型）。

2. **Codegen 侧**：deopt stub 在保存寄存器时，
   同时将 `xr_tag` 写入 `deopt_regs[]` 的 tag 通道
   （现有 `XirRtDeoptSlot.xr_tag` 字段已存在，确保全部填充）。

3. **Reconstruct 侧**：删除 `tagged_heuristic` 标签和代码块，
   当 `xr_tag == XR_RTAG_UNKNOWN` 时直接按 machine type 处理
   （I64 → I64, F64 → F64, PTR → NULL check 后 PTR）。

4. **Multi-return 侧**：`xir_jit_read_multi_ret` 中同样删除启发式，
   使用 `ret_tags[i]` 精确重建。

**验收**：
- `grep -n "tagged_heuristic\|0x10000" src/jit/` 无结果
- `--jit-force` 模式下 EXIT_DIFF 数量减少

### A.3 vreg 上限不一致

**问题**：`xir_jit.c:395` 硬拒绝 `nvreg > 512`，但 `xir_pass_limits.h` 定义
`XIR_MAX_FUNC_VREGS = 4096`。GCM/Range Analysis 内部限制也是 512。

**修复方案**：

1. **提升 try_compile 上限**：`xir_jit.c:395` 改为 `XIR_MAX_FUNC_VREGS`（4096）
2. **GCM / Range Analysis 改动态分配**：
   - 删除 `XIR_GCM_MAX_BLOCKS`, `XIR_GCM_MAX_VREGS`, `XIR_RA_MAX_VALUES` 宏
   - 改为 `xr_malloc(func->nblk * sizeof(...))` 动态分配
3. **LSRA**：`xir_regalloc.c` 的 `slot_end[MAX_SPILL_SLOTS]` 改为动态分配
   或与 `xir_current_target->max_spill_slots` 对齐

**验收**：
- `grep -n "512" src/jit/xir_jit.c` 无 vreg 相关硬编码
- 大函数（>512 vreg）的 JIT 编译不再被静默跳过

### 阶段 A 验收清单
- [ ] `known_failures.txt` 清空（或仅剩非 deopt 相关的已知限制）
- [ ] `deopt_reconstruct` 无地址启发式
- [ ] vreg 上限统一为 4096
- [ ] `ctest --output-on-failure` 全过
- [ ] `scripts/run_regression_tests.sh` 全过

---

## 阶段 B：并发原语 JIT 化

**动机**：Xray 以并发为核心卖点（go + channel + select），但当前函数只要
包含任一并发操作码就整体 BAIL_OUT，JIT 对并发代码完全无效。

### B.1 OP_SCOPE_ENTER / OP_SCOPE_EXIT

**当前状态**：`JIT_OP_BAIL_OUT` — 需要 VM scope 跟踪

**分析**：scope enter/exit 本质上是压栈/弹栈操作，在 `coro->scope_stack` 上
记录 structured concurrency 的作用域边界。

**方案**：直接 CALL_C 包装：

```c
// xir_jit_runtime.c
XR_FUNC int64_t xr_jit_scope_enter(XrCoroutine *coro, int64_t scope_kind) {
    // Push scope to coro->scope_stack
    xr_scope_push(coro, (XrScopeKind)scope_kind);
    return 0;
}

XR_FUNC int64_t xr_jit_scope_exit(XrCoroutine *coro, int64_t extra) {
    // Pop scope, cancel children if structured
    xr_scope_pop(coro);
    return 0;
}
```

Builder 中翻译为 `XIR_CALL_C(xr_jit_scope_enter/exit) + GUARD`。

**改动文件**：
- `xir_opcode_support.h`：改为 `JIT_OP_SUPPORTED`
- `xir_builder_misc.c`：新增 `case OP_SCOPE_ENTER/EXIT` handler
- `xir_jit_runtime.h/c`：新增 2 个 bridge 函数

### B.2 OP_SPAWN_CONT

**分析**：`SPAWN_CONT` 创建子协程并要求 child-first 调度。
JIT 不需要自己实现调度，只需要通过 CALL_C 调用 VM 的 spawn 逻辑。

**方案**：

```c
// xir_jit_runtime.c
XR_FUNC XrJitResult xr_jit_spawn_cont(XrCoroutine *coro, int64_t closure_raw,
                                        int64_t nargs) {
    // Reconstruct closure XrValue, prepare args, spawn child
    // Return child handle or deopt if scheduler intervention needed
    XrValue closure = jit_value_from_tag(closure_raw, XR_TAG_PTR);
    XrCoroutine *child = xr_coro_spawn(coro, closure, (int)nargs);
    if (!child) return (XrJitResult){XIR_DEOPT_MARKER, 0};
    return (XrJitResult){(int64_t)(intptr_t)child, XR_TAG_PTR};
}
```

**关键决策**：spawn 后 child-first 调度不在 JIT 中实现。
JIT 函数 spawn 后立即 deopt 回解释器，让调度器接管。
这是正确的选择 — spawn 是冷路径，不值得在 JIT 中优化。

### B.3 OP_GO / OP_GO_INVOKE

**分析**：`go func(args...)` 启动独立协程。与 SPAWN_CONT 类似，
但不要求 child-first 调度。

**方案**：CALL_C 包装 + 返回后继续 JIT 执行（不需要 deopt）：

```c
XR_FUNC int64_t xr_jit_go(XrCoroutine *coro, int64_t closure_raw, int64_t nargs) {
    // Launch coroutine (fire-and-forget)
    XrValue closure = jit_value_from_tag(closure_raw, XR_TAG_PTR);
    xr_go_launch(coro->isolate, closure, coro->jit_ctx->call_args, (int)nargs);
    return 0;
}
```

`OP_GO_INVOKE` 类似，但 closure 来自 method resolve。

### B.4 Timeout / Select（P2 — Deopt 回退策略）

**方案**：不在 JIT 中实现 select/timeout 的复杂状态机。
改为在 builder 中将函数拆分为两段：

1. select 之前的代码正常 JIT
2. 遇到 `OP_SELECT_START` 时 deopt 回解释器

这需要在 eligibility 中将 select 从 "整函数 BAIL_OUT" 改为
"单指令 deopt"：

```c
// xir_opcode_support.h：新增分类
JIT_OP_DEOPT_FALLBACK  // JIT translates but inserts unconditional deopt
```

Builder 遇到 DEOPT_FALLBACK 指令时发射 `XIR_GUARD_ALWAYS_FAIL`（立即 deopt），
这样函数中 select 之前的热循环仍然可以被 JIT。

### B.5 验收清单
- [ ] `OP_SCOPE_ENTER/EXIT/SPAWN_CONT` 改为 `JIT_OP_SUPPORTED`
- [ ] `OP_GO/GO_INVOKE` 改为 `JIT_OP_SUPPORTED`
- [ ] `OP_SELECT_*` 改为 `JIT_OP_DEOPT_FALLBACK`（不再 BAIL_OUT 整函数）
- [ ] 新增 `tests/jit/076_go_basic.xr`, `077_scope_enter_exit.xr`
- [ ] 含 `go { }` 的函数能编译并通过 JIT diff 测试
- [ ] `ctest` 全过

---

## 阶段 C：Tiered Compilation 完善

**动机**：Tier 1→2 路径已存在但缺乏反馈机制。deopt 后没有
"重新编译更保守版本"的能力，也没有编译时间预算。

### C.1 Deopt 反馈 + 自适应重编译

**问题**：函数频繁 deopt（类型不稳定）时，JIT 反复进入同一份
乐观代码 → 反复 deopt → 性能退化。

**方案**：

1. **Deopt 计数器**：`XrProto` 新增 `deopt_count` 字段（`_Atomic uint32_t`）

2. **Deopt 回退策略**：
   ```
   deopt_count < 5   → 正常 deopt 回解释器，继续使用现有 JIT 代码
   deopt_count >= 5  → 标记为 "needs recompile"，下次调用时触发重编译
   deopt_count >= 20 → 禁用 JIT（永久回退到解释器）
   ```

3. **重编译带保守 profile**：recompile 时将 deopt 点的类型信息标记为
   polymorphic（不再做类型特化），生成更通用但不 deopt 的代码。

4. **实现位置**：
   - `xir_jit.c` 的 `xir_jit_call` deopt 路径：`atomic_fetch_add(&proto->deopt_count, 1)`
   - `xir_jit.c` 的 `xir_jit_try_compile`：检查 `deopt_count` 决定是否用保守模式
   - `xir_builder.h`：新增 `conservative` 标志，禁用类型特化 guard

### C.2 编译时间预算

**问题**：`-O2` 管线对极端复杂函数可能编译耗时过长，阻塞 bg worker。

**方案**：

```c
// xir_pass.h
typedef struct XirCompileBudget {
    uint64_t start_ns;         // 编译开始时间戳
    uint64_t deadline_ns;      // start_ns + 50ms (默认)
    bool     timed_out;        // 超时标记
} XirCompileBudget;
```

在 `xir_run_fixedpoint` 的每轮循环开头检查：
```c
if (budget && xr_time_now_ns() > budget->deadline_ns) {
    budget->timed_out = true;
    break;  // 提前终止
}
```

超时时：
- 当前已完成的 pass 结果保留（IR 始终合法）
- 跳过剩余 pass，直接进入 regalloc + codegen
- 产出 Tier 1.5 质量的代码（比 -O0 好，比 -O2 差）

### C.3 JIT 统计导出

**方案**：`--jit-stats` 命令行选项，程序退出时输出：

```
=== JIT Statistics ===
Functions compiled: 47 (Tier1: 35, Tier2: 12)
Total compile time: 234ms (avg 4.9ms/func)
Code cache used: 128KB / 64MB
Deopts: 23 (type mismatch: 15, guard fail: 8)
Top deopt functions:
  process_item  : 8 deopts (type unstable: arg0 int→float)
  render_node   : 5 deopts (shape guard fail)
```

**实现**：
- `XirJitState` 新增统计字段（compile_time_total_ns, deopt_total, etc.）
- `xir_jit_destroy` 时若 `--jit-stats` 则打印摘要

### 阶段 C 验收清单
- [ ] 频繁 deopt 的函数自动降级到保守编译
- [ ] `deopt_count >= 20` 的函数不再尝试 JIT
- [ ] bg worker 编译单函数不超过 50ms（通过 log 验证）
- [ ] `--jit-stats` 输出编译统计
- [ ] `ctest` 全过

---

## 阶段 D：代码质量提升

### D.1 Peephole 规则扩展

**当前状态**：3 条规则，165 行。

**新增规则**（按收益排序）：

| # | 规则 | 预期收益 |
|---|------|----------|
| 4 | **LDP/STP 融合**：连续相邻 LDR/STR 融合为 LDP/STP | spill-heavy 函数 -10% 代码体积 |
| 5 | **冗余 flag 消除**：SUBS + CMP → 删除 CMP | 循环比较 -1 条/iter |
| 6 | **ADD/SUB shifted**：`ADD X,Y; LSL X,X,#N` → `ADD X,Y,LSL #N` | 数组索引 -1 条 |
| 7 | **MOVZ+MOVK 简化**：可用 ORR bitmask 表示时简化 | 常量加载 -1 条 |
| 8 | **B.cond over B**：`B.cond L1; B L2; L1:` → `B.!cond L2` | 分支 -1 条 |

**实现**：扩展 `xir_peephole.c`，保持单遍扫描架构（pattern match + NOP replace）。

### D.2 GCM / Range Analysis 动态分配

**当前**：`XIR_GCM_MAX_BLOCKS = 512`, `XIR_RA_MAX_VALUES = 512`。

**改法**：
```c
// Before:
uint32_t early[XIR_GCM_MAX_BLOCKS];
// After:
uint32_t *early = xr_malloc(func->nblk * sizeof(uint32_t));
// ... use ...
xr_free(early);
```

同时删除 `xir_pass_limits.h` 中对应的 512 限制宏，
保留 `XIR_MAX_FUNC_VREGS = 4096` 作为函数级安全上限。

### D.3 异常处理优化

**当前**：try/catch 在 JIT 中编译，但 throw 永远 deopt。
catch 块虽然生成了代码但从未在 JIT 中真正执行。

**改法（最小方案）**：

不实现完整异常表，而是优化 "热路径 try + 冷路径 catch" 模式：

1. **try 块内代码**：正常 JIT，不产生额外开销
2. **throw**：仍然 CALL_C deopt（throw 是冷路径，开销可接受）
3. **catch 块后的代码**：deopt 恢复后由解释器执行 catch 并继续

**关键优化**：确保 try 块内部不因为 "可能 throw" 而影响优化。
当前 LICM 不提升 try 块内的 LOAD — 如果 LOAD 不可能抛异常
（纯内存读），应该允许提升。

添加到 `xir_alias.h`：
```c
// Returns true if instruction cannot throw (pure memory access)
static inline bool xir_ins_nothrow(XirIns *ins) {
    switch (ins->op) {
    case XIR_LOAD_FIELD: case XIR_CONST_I64: case XIR_CONST_F64:
    case XIR_ADD_I64: case XIR_SUB_I64: case XIR_MUL_I64:
    // ... all pure arithmetic and loads ...
        return true;
    default:
        return false;
    }
}
```

LICM 中：`if (xir_ins_nothrow(ins) || !blk->exception_handler) → 可提升`。

### D.4 Profile-Guided Block Layout

**当前**：`xir_pass_reorder_blocks` 用贪心算法排列基本块。

**改法**：利用 `proto->type_feedback` 中的分支 profile 信息：

1. 如果有 feedback，按 taken/not-taken 频率构造 trace
2. 将 hot path 块连续排列（减少 taken branch）
3. 将 cold path（deopt stub、异常处理）移到函数末尾

**实现**：在 `xir_pass_cfg.c` 的 `xir_pass_reorder_blocks` 中，
若有 feedback 则使用 trace-based layout，否则退回当前贪心。

### 阶段 D 验收清单
- [ ] peephole 规则从 3 条扩展到 8 条
- [ ] `XIR_GCM_MAX_BLOCKS`, `XIR_RA_MAX_VALUES` 宏被删除
- [ ] try 块内纯算术/LOAD 可被 LICM 提升
- [ ] nbody benchmark 代码体积相对阶段 C **减少 ≥ 5%**

---

## 阶段 E：Code Cache 管理

**动机**：当前 `XirCodeAlloc` 是纯 bump 分配器，无回收机制。
长期运行的服务器程序（HTTP server）会持续 mmap 直到内存耗尽。

### E.1 Code Cache Budget

**方案**：

```c
// xir_code_alloc.h 新增
#define XIR_CODE_CACHE_DEFAULT_MB  64
#define XIR_CODE_CACHE_MAX_MB      256

typedef struct XirCodeAlloc {
    XirCodePage *pages;
    XirCodePage *current;
    size_t total_allocated;
    size_t total_used;
    size_t budget;              // NEW: max bytes allowed
    uint32_t n_pages;           // NEW: page count for LRU
} XirCodeAlloc;
```

当 `total_allocated > budget` 时：
- **不分配新页**，触发 code eviction
- 根据 proto 的最近使用时间（`exec_count` 或 `last_exec_time`）
  选择最冷的编译结果淘汰
- 淘汰 = `proto->jit_entry = NULL` + munmap 对应页面

### E.2 旧代码安全释放

**问题**：Tier 1 → Tier 2 重编译后，旧的 Tier 1 代码仍然可能被
正在执行的协程引用（OSR 路径、resume entry）。

**方案**：Epoch-based reclaim

```c
typedef struct XirCodeGarbage {
    void *code;
    size_t size;
    uint64_t retire_epoch;      // 退役时的 epoch
    struct XirCodeGarbage *next;
} XirCodeGarbage;
```

1. 每次 GC safepoint 时递增全局 `jit_epoch`
2. 重编译时，旧代码不立即释放，而是放入 garbage list 并记录当前 epoch
3. 当所有活跃协程的 `saved_epoch >= retire_epoch` 时，安全 munmap

### E.3 W^X 验证

**当前状态**：`xir_code_alloc.h` 注释提到 macOS 用 `MAP_JIT +
pthread_jit_write_protect_np`，Linux 用 `mmap + mprotect`。

**验证**：
- 确认 `xir_code_alloc.c` 在 Apple Silicon 上使用了 `MAP_JIT`
- 确认 codegen 在写入代码后调用了 `xir_code_make_executable`
- 确认 peephole（修改已发射代码）前后正确调用了 write protect toggle

### 阶段 E 验收清单
- [ ] Code cache 默认 64MB budget
- [ ] 超 budget 时触发 LRU eviction（通过 log 验证）
- [ ] 重编译后旧代码通过 epoch-based 安全释放
- [ ] 长期运行测试（10min stress）内存稳定在 budget 以内

---

## 阶段 F：x86-64 后端

**动机**：当前只有 ARM64 后端。Intel/AMD 覆盖 80% 服务器和开发机。

### F.1 架构

```
src/jit/
├── xir_target_arm64.c      (现有)
├── xir_arm64.h/c           (现有)
├── xir_codegen.c           (现有，ARM64 specific)
├── xir_codegen_call.c      (现有)
├── xir_codegen_mem.c       (现有)
├── xir_target_x64.c        (新增) ← target 描述
├── xir_x64.h/c             (新增) ← 指令编码
├── xir_codegen_x64.c       (新增) ← 代码生成主文件
└── xir_codegen_x64_call.c  (新增) ← 调用约定
```

**关键设计决策**：

1. **不抽象 codegen**：ARM64 和 x86-64 的差异太大（3-operand vs 2-operand、
   flags 语义、calling convention），强行抽象会导致两边都写不好。
   直接为 x86-64 写独立的 codegen，共享 XIR pass pipeline。

2. **共享部分**：XIR IR、所有 optimization passes、regalloc、deopt 表结构
   这些全部共享。只有"XIR → machine code"这最后一步是平台相关的。

3. **Calling convention**：x86-64 System V ABI（rdi, rsi, rdx, rcx, r8, r9）
   vs ARM64（x0-x7）。JIT bridge 函数签名不变，codegen 负责映射。

### F.2 xir_target_x64.c — 目标描述

```c
const XirTarget xir_target_x64 = {
    .name = "x86-64",
    // GP registers: rax, rcx, rdx, rbx, rsi, rdi, r8-r15 (14 allocatable)
    // Exclude: rsp, rbp (frame), r15 (coro ptr)
    .gp_regs = { RAX, RCX, RDX, RBX, RSI, RDI, R8, R9, R10, R11, R12, R13, R14 },
    .ngp_allocatable = 13,
    .ngp_caller_saved = 9,   // rax,rcx,rdx,rsi,rdi,r8,r9,r10,r11
    // FP registers: xmm0-xmm15 (all allocatable, xmm0-xmm7 caller-saved)
    .fp_regs = { XMM0, XMM1, ..., XMM15 },
    .nfp_allocatable = 16,
    .nfp_caller_saved = 8,
    // Special registers
    .coro_reg = R15,          // coroutine pointer (callee-saved)
    .scratch_reg = R11,       // scratch (caller-saved)
    .frame_base_offset = -8,  // rbp-relative
    .max_spill_slots = 256,
};
```

### F.3 xir_x64.h — 指令编码

最小指令集（Tier 0）：

| 类别 | 指令 |
|------|------|
| 算术 | ADD, SUB, IMUL, IDIV, NEG, AND, OR, XOR, NOT, SHL, SHR, SAR |
| 比较 | CMP, TEST |
| 跳转 | JMP, Jcc, CALL, RET |
| 移动 | MOV reg/imm/mem, LEA, MOVSX, MOVZX, CMOV |
| 浮点 | ADDSD, SUBSD, MULSD, DIVSD, CVTSI2SD, CVTTSD2SI, MOVSD, UCOMISD |
| 栈 | PUSH, POP |

**编码复杂度**：x86-64 变长编码比 ARM64 固定 4B 编码复杂得多。
使用 byte buffer + REX prefix helper：

```c
// xir_x64.h
typedef struct X64Emit {
    uint8_t *buf;
    size_t pos;
    size_t cap;
} X64Emit;

static inline void x64_rex(X64Emit *e, bool w, bool r, bool x, bool b) {
    uint8_t rex = 0x40;
    if (w) rex |= 0x08;  // 64-bit operand
    if (r) rex |= 0x04;  // ModRM.reg extension
    if (x) rex |= 0x02;  // SIB.index extension
    if (b) rex |= 0x01;  // ModRM.rm / SIB.base extension
    e->buf[e->pos++] = rex;
}

// Example: ADD r64, r64
static inline void x64_add_rr(X64Emit *e, X64Reg dst, X64Reg src) {
    x64_rex(e, true, src > 7, false, dst > 7);
    e->buf[e->pos++] = 0x01;  // ADD r/m64, r64
    x64_modrm(e, 0b11, src & 7, dst & 7);
}
```

### F.4 分阶段实施

| 子阶段 | 内容 | 可运行的测试 |
|--------|------|-------------|
| F.4.1 | target 描述 + 空 codegen 框架（编译通过但生成失败） | 编译测试 |
| F.4.2 | 整数算术 + 控制流 | 001-010 |
| F.4.3 | 函数调用 + 参数传递 | 012-013 |
| F.4.4 | 浮点运算 | 002, 006, 011 |
| F.4.5 | 对象/数组/字符串（CALL_C bridge） | 015-023 |
| F.4.6 | Deopt + OSR | 033-036 |
| F.4.7 | 完整功能（剩余测试） | 全部 |

**构建集成**：
```cmake
# CMakeLists.txt
if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64")
    target_sources(xray PRIVATE
        src/jit/xir_target_x64.c
        src/jit/xir_x64.c
        src/jit/xir_codegen_x64.c
        src/jit/xir_codegen_x64_call.c
    )
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|arm64")
    target_sources(xray PRIVATE
        src/jit/xir_target_arm64.c
        src/jit/xir_arm64.c
        src/jit/xir_codegen.c
        src/jit/xir_codegen_call.c
        src/jit/xir_codegen_mem.c
    )
endif()
```

### 阶段 F 验收清单
- [ ] `xir_target_x64` 在 x86-64 机器上被正确选择
- [ ] `tests/jit/001_arithmetic_int.xr` 在 x86-64 上通过
- [ ] F.4.2 完成后 10 个基础测试通过
- [ ] F.4.7 完成后全部 JIT 测试通过
- [ ] ARM64 机器上的编译和测试不受影响

---

## 风险矩阵

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| A.1 级联 deopt 的 codegen 改动影响正常 CALL 性能 | 低 | 中 | 只在 CALL_KNOWN 后加检测，不影响 CALL_C |
| B.x 并发 bridge 函数的线程安全 | 中 | 高 | 所有 bridge 函数必须通过 TSan 验证 |
| C.1 保守重编译产出的代码质量过低 | 中 | 中 | 保守模式只禁用类型特化，不禁用 LICM/GVN 等结构优化 |
| D.1 peephole 规则的正确性 | 中 | 中 | 每条规则配一个专门的 JIT 单测 |
| E.2 epoch-based reclaim 的 epoch 溢出 | 低 | 低 | 64-bit epoch，不会溢出 |
| F.x x86-64 指令编码 bug | 高 | 高 | 子阶段递进，每步只加少量指令并跑测试 |

## 中止 / 回退策略

- 每阶段做 git 标签 `jit-next-phaseX`
- 阶段 N 失败 → `git reset --hard jit-next-phase(N-1)` 并在新分支重做
- **阶段 F 可独立取消**：不影响 A-E 的成果（ARM64 仍然完整可用）

## 明确排除

- ❌ **Trace-based JIT**：method-based + OSR 已足够，trace 的复杂度不值得
- ❌ **Sea of Nodes**：IR 已经足够好，切换的侵入性过高
- ❌ **LLVM/Cranelift 后端**：违背"轻量独立"定位
- ❌ **32-bit 支持**：只支持 64-bit（ARM64 / x86-64）
- ❌ **JIT→JIT 直接调用（无 bridge）**：复杂度高，收益小（CALL_KNOWN 已足够快）
