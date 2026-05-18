# JIT 模块重构计划（`src/jit`）

**目标**：在"不考虑向后兼容"的前提下，把当前偏 Tier1 自举质量的 JIT 升级为一个
结构清晰、算法现代化、大函数仍能优化、并发编译可扩展的工业级实现。

## 总则

### 开发原则（由用户强制约束）
1. ✅ 直接采用最佳设计，不保留旧接口 / 不做兼容层 / 不做 deprecated 路径。
2. ✅ 每阶段独立可合并、独立可回滚；阶段间是严格前置关系。
3. ✅ 每次代码改动后必须运行测试（快验 `cd build && ctest --output-on-failure`，
   阶段切换前跑完整回归 `scripts/run_regression_tests.sh`）。
4. ❌ **禁止**在同一次改动里掺入无关修复；每个阶段只专注自己的目标。
5. ❌ **禁止**保留旧版算法代码作"快速回退"，必须直接替换。

### 交付节奏
| 阶段 | 规模 | 风险 | 预估 | 关键产出 |
|------|------|------|------|----------|
| 1. Must-Fix（正确性 + 规范） | 8 文件 | 低 | 0.5 d | 消除内存 / 越界 / 静默失败 |
| 2. 基础设施（DomTree/LoopTree/DefUse/Limits） | 4 新头 | 中 | 1.5 d | 所有后续 pass 的统一 API |
| 3. 核心算法（SCCP / Type worklist / GVN / EA / LICM / Alias） | 6 pass | 高 | 3 d | 优化质量大跃升 |
| 4. Pipeline（不动点 + ChangeTracker） | 1 文件 | 中 | 0.5 d | 收敛 / 可诊断 |
| 5. Regalloc + Codegen（hole-aware / remat / peephole） | 3 文件 | 中 | 1.5 d | 寄存器利用率提升 |
| 6. 编译驱动（多线程 bg / TFA invalidation / shared_protos 快照） | 3 文件 | 中 | 1 d | 多核扩展 |
| 7. 代码拆分（符合 3000/800 行规范） | 12 新文件 | 低 | 1 d | 可维护性 |

**合计**：约 9 个工作日（不含 review + 回归）。

### 验证基线
每阶段完成前：
```bash
# 快验
cmake --build build -j8
cd build && ctest --output-on-failure
# 完整回归
./scripts/run_regression_tests.sh
# JIT 专项
./scripts/jit_stress_test.sh   # 若存在，否则跑 tests/jit/
```

每阶段完成后：
- `scripts/check_architecture.sh` errors 只能减少、不能增加；
- `wc -l src/jit/*.{c,h}` 所有文件 ≤ 3000 / 800 行；
- `grep -r "TODO\|FIXME\|XXX\|HACK" src/jit/` 数量只减不增；
- JIT 性能基准（`tests/demos/demo_14_nbody.xr` 等）相对 baseline 至少持平。

---

## 阶段 1：Must-Fix（正确性与规范）

**动机**：把当前所有违反编码规范的点和静默错误在阶段 2 前清掉，避免后续在错误
的地基上继续盖楼。

### 1.1 `xr_realloc` 统一用 `XR_REALLOC_OR_ABORT`

| 位置 | 现状 | 修复 |
|------|------|------|
| `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_regalloc.c:1037` | `*wl = xr_realloc(*wl, …)` 无 NULL 检查 | 改为 `XR_REALLOC_OR_ABORT(*wl, …, "regalloc wl_insert")` |
| `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir.c:161` | `new_arr = xr_realloc(func->blocks, …); if (!new_arr) return NULL;` | 改为 `XR_REALLOC_OR_ABORT(func->blocks, …, "xir_func add_block")` |
| `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir.c:269` | 同上（vregs 扩容） | 同上 |
| `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir.c:387` | 同上（call_arg_pool） | 同上 |
| `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir.c:459` / `:496` | 同上（consts） | 同上 |
| `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_tfa.c:78` / `:88` / `:98` / `:703` | 同上（summaries/calls/worklist/stack） | 同上 |

**原则**：`xr_malloc/xr_calloc` 失败必须立即 abort；保留 `if (!new_arr) return` 的分支
会制造"悄悄跳过扩容 → 后续越界写"的隐蔽 bug。

**验收**：
```bash
grep -rn "= xr_realloc(" src/jit/  # 应仅剩 XR_REALLOC_OR_ABORT 宏展开
```

### 1.2 `xir_dominates` 移除 256 步硬上限

`@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir.c:785-793`：

**改法**：
```c
bool xir_dominates(uint32_t *idom, uint32_t a, uint32_t b) {
    while (b != a) {
        if (b == 0) return (a == 0);
        uint32_t next = idom[b];
        if (next == UINT32_MAX || next == b) return false;
        b = next;
    }
    return true;
}
```
用 `idom[b] == b` 判环（仅 entry 自循环），不再用计数器。

**同时删除**：`GVN_MAX_BLOCKS = 256` 这个熔断（由 idom 正确性保证）。

### 1.3 Spill slots 常量一致性

当前三处常量不一致：
- LSRA `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_regalloc.c:95` `MAX_SPILL_SLOTS = 256`（`slot_end[]` 数组大小）
- Codegen `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_codegen_internal.h:44` `MAX_SPILL_SLOTS = 64`（**事实上不被 frame_size 计算使用**，只在 SUSPEND 路径做 16 的二次截断）
- Target `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_target_arm64.c:56` `max_spill_slots = 64`（同上，未被检查）

**改法**：
1. 把 "spill 上限" 作为**target 唯一真相源**：`xir_target_arm64.c` 里把 `max_spill_slots`
   调整到 **256**（对齐 LSRA 能力）或干脆删除这个字段（frame 动态增长）。
2. 删除 `xir_codegen_internal.h` 的 `MAX_SPILL_SLOTS`（无实际作用）。
3. LSRA 在 `next_spill` 超过 `xir_current_target->max_spill_slots` 时显式 `ctx->had_error = true`
   并返回 false，不要再写越界 `slot_end[]`。
4. SUSPEND 路径的 `if (ns > 16) ns = 16`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_codegen_mem.c:1148`）：
   - 要么把 `XrSuspendState.spill[]` 扩到 target 上限；
   - 要么在 LSRA 发现会 suspend 的函数 spill > 16 时主动 refuse JIT（一致的 eligibility）。

### 1.4 Opcode 白名单归一到 `xir_opcode_support.h`

**现状**：两处白名单必须手工同步：
- `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_jit.c:282-394` 的大 switch；
- `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_opcode_support.h` 的 `jit_op_support[]` 表；
- `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_builder.c` 的 `translate_instruction` 分派。

**改法**：
1. 让 `jit_op_support[]` 成为唯一真相源；使用 X-macro 展开填充。
2. `is_jit_eligible` 只保留 `xir_opcode_jit_support(op) == JIT_OP_SUPPORTED` 判定。
3. Builder 的 fallback 改为：若 `support != JIT_OP_SUPPORTED` 则置 `ops_skipped++`，
   由调用方通过 eligibility 先行拒绝（不靠 builder 事后 bail out）。

**验收**：`xir_jit.c` 里关于 opcode 的代码少 100+ 行；新增 opcode 时只改一处。

### 1.5 `is_jit_eligible` 拆分到独立文件

`xir_jit.c` 当前 3820 行，其中 200+ 行是 `is_jit_eligible` 的巨型 switch。
拆到 `xir_eligibility.c`，顺便把"复杂度 / 参数 / 返回类型 / deopt backoff"都归到一起。

### 1.6 后台编译的共享字段快照

`@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xjit_compile_queue.c:73-74`：

**问题**：bg 线程 `proto->type_feedback = NULL`；主线程同时可能调用 `xfb_record_arg`。

**改法**：
1. 新增 `XirBgTask` 结构，在主线程 `xjit_queue_push` 时**拷贝快照**：
   ```c
   typedef struct XirBgTask {
       XrProto *proto;
       bool is_recompile;
       XirTypeFeedback feedback_snapshot;   // 拷贝，不是指针
       int nshared;
       XrProto *shared_protos[32];          // 拷贝，最多 32 个
       struct XrShape *shape_hint;
   } XirBgTask;
   ```
2. bg 线程只读 task 里的 snapshot，**完全不改 `proto->*` 字段**。
3. 删除 `proto->type_feedback = NULL; ... proto->type_feedback = saved_fb;` 的临时换指针。
4. 顺带让 bg 线程也能享受 `shared_protos` → `CALL_KNOWN` 优化
   （当前因竞争风险被禁用，导致 Tier1 bg 版本永远少优化一层）。

### 1.7 LICM preheader 识别独立化

现状依赖"preheader_bi = header_bi - 1"的块排序巧合（`@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_pass.c:1249`）。

**改法**（阶段 1 里先做最小修复，阶段 3 再做大重构）：
```c
// Preheader = 唯一的、循环外的、单出口指向 header 的块
static XirBlock *find_preheader(XirFunc *func, uint32_t hdr, uint32_t lat) {
    XirBlock *header = func->blocks[hdr];
    XirBlock *pre = NULL;
    for (uint32_t p = 0; p < header->npred; p++) {
        XirBlock *cand = header->preds[p];
        uint32_t ci;
        for (ci = 0; ci < func->nblk; ci++) if (func->blocks[ci] == cand) break;
        if (ci >= hdr && ci <= lat) continue;       // 循环内（latch）
        if (pre) return NULL;                        // 多个 → 不是单 preheader
        pre = cand;
    }
    return pre;
}
```
若找不到单 preheader，`xir_pass_licm` 当次循环跳过（下一阶段再做 preheader 插入）。

### 阶段 1 验收清单
- [ ] `grep -rn "xr_realloc(" src/jit/` 仅剩宏内部
- [ ] `xir_jit.c` 行数 < 3500
- [ ] 新增 `xir_eligibility.c/h`
- [ ] bg queue 压测（16 个 worker × 10000 次 yield + await）无 race 告警
- [ ] `ctest --output-on-failure` 全过
- [ ] `scripts/run_regression_tests.sh` 全过

---

## 阶段 2：基础设施

**动机**：阶段 3-5 的所有新 pass / 新算法都需要共用 "dominator / loop / defuse /
alias / change tracker" 五件套。先把这些持久化到 `XirFunc` 里，后续所有 pass 都改成
**读取 + 失效**模式，拒绝每个 pass 内部自己重算。

### 2.1 `XirDomTree`：常数时间支配查询

新增 `src/jit/xir_domtree.h/c`：

```c
typedef struct XirDomTree {
    uint32_t *idom;        // [nblk]
    uint32_t *dfs_in;      // [nblk] DFS-in 序号
    uint32_t *dfs_out;     // [nblk] DFS-out 序号
    uint32_t *children;    // 扁平存储：所有 dom-children 索引
    uint32_t *child_start; // [nblk+1] children 的起止
    uint32_t  nblk;
} XirDomTree;

// a dominates b  ⇔  dfs_in[a] ≤ dfs_in[b] < dfs_out[a]   —— O(1)
static inline bool xir_dom_covers(const XirDomTree *dt, uint32_t a, uint32_t b) {
    return dt->dfs_in[a] <= dt->dfs_in[b] && dt->dfs_in[b] < dt->dfs_out[a];
}

XR_FUNC XirDomTree *xir_func_get_domtree(XirFunc *func);
XR_FUNC void xir_func_invalidate_domtree(XirFunc *func);
```

**实现**：Semi-NCA（比 Cooper 慢但代码简短；在 100-1000 块规模可忽略差别）。

**替换**：`xir_dominates`、`xir_func_get_idom` 的所有调用点全部改走 `xir_dom_covers`。

**破坏**：删除 `xir.c:706-793` 的旧 idom 代码、删除 `XirBlock::idom` 字段。

### 2.2 `XirLoopTree`：嵌套循环树

新增 `src/jit/xir_looptree.h/c`：

```c
typedef struct XirLoop {
    struct XirLoop *parent;   // 外层循环
    struct XirLoop *child;    // 第一个内层循环
    struct XirLoop *sibling;
    XirBlock *header;
    XirBlock *preheader;      // 已规范化：必定存在，且单出口指向 header
    XirBlock *latch;          // 多 latch 时取"第一个"；用 body[] 访问全部
    XirBlock **body;          // 所有属于此循环的块
    uint32_t  nbody;
    uint32_t  depth;
    uint32_t  id;
} XirLoop;

typedef struct XirLoopInfo {
    XirLoop  *root_list;      // 顶层循环链表
    XirLoop **block_to_loop;  // [nblk]，每个块所属的最内层循环
    XirLoop **all_loops;      // 扁平数组
    uint32_t  nloop;
} XirLoopInfo;

XR_FUNC XirLoopInfo *xir_func_get_loops(XirFunc *func);
XR_FUNC XirBlock    *xir_ensure_preheader(XirFunc *func, XirLoop *loop);
```

**关键不变量**：`xir_func_get_loops` 保证每个循环都有**独占的 preheader**
（若原本没有就自动插入一个空块）。LICM/GCM/range 据此简化。

**破坏**：
- 删除 `XirBlock::loop_depth` 字段（改为查 loopinfo）；
- 删除 `xir_pass_licm` 里的 `LicmLoop loops[LICM_MAX_LOOPS]` 固定数组；
- `LICM_MAX_LOOPS / LICM_MAX_STORE_OBJS / LICM_MAX_ITERATIONS` 全部删除。

### 2.3 `XirDefUse`：持久化 + 增量维护

现状：每个 pass 自建 `XirDefUse du; xir_defuse_build(&du, func); ... xir_defuse_free(&du);`。

**改法**：`XirFunc::defuse` 字段，`xir_func_get_defuse` 懒构建 + 返回缓存：
- Pass 修改 IR 时通过 `xir_defuse_update_def(du, old_vreg, new_vreg)` 增量更新；
- `XIR_RUN_PASS` 宏把 `xir_rebuild_vreg_defs` 改为 `xir_defuse_rebuild_if_dirty`，
  由 pass 显式声明"我改了 CFG / vreg definitions"来标记 dirty。

**破坏**：删掉 `xir_rebuild_vreg_defs` 无条件在 pass 后全扫的行为。

### 2.4 `XirAliasInfo`：ALLOC provenance

新增 `src/jit/xir_alias.h/c`：

```c
typedef enum {
    XIR_ALIAS_UNKNOWN,
    XIR_ALIAS_FRESH_ALLOC,    // 来自 XIR_ALLOC，且未泄漏
    XIR_ALIAS_PARAM,          // 来自参数
    XIR_ALIAS_GLOBAL,         // 来自 GETSHARED / GETBUILTIN
} XirAliasSource;

typedef struct {
    XirAliasSource source;
    XirRef         origin;    // 指向 ALLOC / PARAM 定义
} XirAliasInfo;

XR_FUNC const XirAliasInfo *xir_func_get_alias(XirFunc *func, XirRef vreg);
```

LICM / escape / store_to_load / DSE 用它做精确的 "可能别名" 判断，消除
"任一写对任一 LOAD 都杀"的粗糙近似。

### 2.5 `xir_pass_limits.h`：集中配置

现状的固定限制散落在 8 处（见阶段 0 的分析章节）。新文件：

```c
// src/jit/xir_pass_limits.h

// 基础：所有数组基于 func->nvreg / func->nblk / func->ntotal_ins 动态分配。
// 仅保留少量"压根不该超"的安全上限，并在超限时走 eligibility 拒绝、不再静默跳过。

#define XIR_MAX_FUNC_VREGS      4096    // JIT 拒绝的硬上限（原 512）
#define XIR_MAX_FUNC_BLOCKS     4096    // 原 GVN 256 的扩大
#define XIR_MAX_FUNC_TOTAL_INS  65536

// LICM / escape / inline / ifconv 的"经验阈值"
#define XIR_LICM_MAX_ITER        8
#define XIR_INLINE_MAX_COUNT     16
#define XIR_INLINE_MAX_SIZE      500
#define XIR_IFCONV_MAX_INS       6      // 原 2，放宽
#define XIR_IFCONV_MAX_PHIS      4      // 原 2，放宽
```

**破坏**：所有 pass 文件顶端的 `#define XXX_MAX_YYY` 全部搬到这里；
剩下的存储（哈希表、追踪数组）全部改成 `xr_malloc` 动态分配。

### 2.6 `XirPassChangeTracker`：驱动不动点

新增到 `xir_pass.h`：

```c
typedef struct XirPassChange {
    bool cfg_changed;       // 块/边/终结器变化 → idom/loops 失效
    bool vreg_defs_changed; // 新增/删除 vreg 定义 → defuse 失效
    bool ins_changed;       // 仅指令 op/args 变化（更细粒度）
    uint32_t n_ins_removed;
    uint32_t n_ins_added;
    uint32_t n_vregs_dead;
} XirPassChange;

// 新签名：每个 pass 返回 XirPassChange
typedef XirPassChange (*XirPassFn)(XirFunc *func);
```

**破坏**：把全部现有 `void xir_pass_xxx(XirFunc*)` 改成 `XirPassChange xir_pass_xxx(XirFunc*)`。

### 阶段 2 验收清单
- [ ] 新增 `xir_domtree.{h,c}`, `xir_looptree.{h,c}`, `xir_alias.{h,c}`, `xir_pass_limits.h`
- [ ] `XirBlock::idom`, `XirBlock::loop_depth` 字段被删除
- [ ] `xir_pass_licm.c` 中不再出现 `LICM_MAX_LOOPS`
- [ ] 所有 pass 函数签名返回 `XirPassChange`
- [ ] 编译时 `wc -l src/jit/xir.c` 减少（dom 算法迁出）
- [ ] `ctest` 全过（本阶段 IR 和算法行为不应变化，仅 API 重构）

---

## 阶段 3：核心算法升级

**动机**：阶段 2 已准备好 DomTree / LoopInfo / DefUse / Alias，现在可以把几个
核心 pass 从"保守、小固定表、局部"升级到"SSA-worklist 驱动、支持大函数"。

### 3.1 SCCP（取代 `const_prop` + `branch_simp` + `remove_unreachable`）

**目标**：Sparse Conditional Constant Propagation —— 同时折叠常量和探测不可达块。

**新文件**：`src/jit/xir_pass_sccp.c`

**算法**（Wegman-Zadeck）：
```
Lattice:  TOP > I64(k) | F64(k) | BOOL(k) | PTR(null) > BOT
Worklist: (ssa_edges, cfg_edges)
Init:     entry 可达；所有 vreg = TOP
Iterate:
  取一个 cfg_edge → 标记 target 块 reachable，把块内所有指令入 ssa queue
  取一个 ssa_edge → 按 meet 更新 vreg 值；若降格则把所有 uses 再入 ssa queue
Done when worklists empty
Rewrite:  常量 vreg → CONST_*；BR(const cond) → JMP；不可达块 → 删除
```

**破坏**：
- 删除 `xir_pass_const_prop` / `xir_pass_branch_simp` / `xir_pass_remove_unreachable`
  三个 pass（SCCP 一次完成）。
- 删除 `@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_pass.c:181-760` 大段常量传播代码。

**pipeline 中的位置**：Phase 0 `select_rep` 后，`type_prop` 前。

### 3.2 TypePropagation 改 worklist

**现状**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_pass_advanced.c:2028` 的 `xir_pass_type_prop` 是 O(pass rounds × all ins)。

**改法**：
```c
XirPassChange xir_pass_type_prop(XirFunc *func) {
    XirDefUse *du = xir_func_get_defuse(func);
    Worklist wl = wl_init_all_ins(func);
    while (!wl_empty(&wl)) {
        XirIns *ins = wl_pop(&wl);
        if (type_prop_update(ins, func)) {
            // 只把 uses 入队
            for (uint32_t u : du_uses_of(du, ins->dst)) wl_push(&wl, u);
        }
    }
}
```

Pipeline 里跑**一次**即可，不再跑 5 轮。

**破坏**：pipeline Phase 0/1/2/3/4/5 里的多次 `xir_pass_type_prop` 合并成一次。

### 3.3 GVN 重写：支持大函数 + PRE + LOAD 值编号

**目标**：
1. 移除 `GVN_MAX_BLOCKS = 256` 熔断。
2. 支持 LOAD_FIELD 的值编号（依赖 `XirAliasInfo`）：若两次 `LOAD_FIELD(obj, k)`
   之间无别名写，编号相同。
3. 可选做 Partial Redundancy Elimination（PRE）—— 如果时间预算不够，留给后续。

**实现骨架**：
```c
XirPassChange xir_pass_gvn(XirFunc *func) {
    XirDomTree *dt = xir_func_get_domtree(func);
    DynTable table = dyntable_init(max(128, total_ins * 2));
    // DFS 遍历 dom-tree（不是块数组顺序）
    gvn_visit_dom(dt, 0 /* entry */, &table);
    dyntable_free(&table);
}
```

`dyntable` 使用 `xr_malloc` 动态扩容，取代阶段前的 `GVN_MIN_TABLE=128 / MAX=2048` 上限。

**破坏**：删除 `GVN_MAX_BLOCKS`；改 `xir_pass.c` 里现有实现。

### 3.4 Escape Analysis 跨块化

**现状**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_pass_advanced.c:746` 只做 block-local。

**改法**：
1. 构建 function-wide 的 `escape_set`：
   - 所有 phi args、RET、跨块 use、CALL_C arg、call_arg_pool、store-field-as-value
     都使其 operand 的 alloc origin 加入 `escape_set`。
2. `XIR_ALLOC` 不在 `escape_set` 中时做 scalar replacement，支持跨块：
   - 为每个 field 创建 SSA phi（在 join 点）而非单块 vreg；
   - 用 Mem2Reg 的标准算法把 STORE_FIELD/LOAD_FIELD 改写成 phi + MOV。

**扩展**：**Partial Escape Analysis**（可选）—— 如果对象仅在某个分支逃逸，
在另一分支做 scalar replace；此路径与 `alloc_sink` 配合。

**破坏**：
- 删除 `EA_MAX_ALLOC = 16`；
- 删除"2d pass: 扫所有其他块看是否用到" 的 `O(nalloc × nblk²)` 片段。

### 3.5 LICM 基于 LoopTree 重写

**改法**：
```c
XirPassChange xir_pass_licm(XirFunc *func) {
    XirLoopInfo *li = xir_func_get_loops(func);  // 已有 preheader
    XirAliasInfo *ai = xir_func_get_alias(func);
    // 从最内层循环向外层处理（保证内层 hoist 的结果被外层看见）
    for (XirLoop *loop : iter_loops_innermost_first(li)) {
        licm_process_loop(func, loop, ai);
    }
}
```

`licm_process_loop` 用 `ai` 判定 LOAD 是否真的被循环内 STORE 写过（按 origin），
而不是"obj vreg 相同就 kill"。

**破坏**：
- 删除 `LICM_MAX_LOOPS / LICM_MAX_STORE_OBJS / LICM_MAX_ITERATIONS`；
- 删除 `preheader_bi = header_bi - 1` 的假设（改用 loop->preheader）；
- 删除 `licm_build_def_block` 函数（`XirDefUse` 提供 O(1) def 查询）。

### 3.6 Store-to-Load / DSE 用 Alias 图 + 动态哈希

**改法**：
1. S2L 表改为 `xr_malloc` 动态结构，按 block 的实际 store/load 数分配；
2. 利用 `XirAliasInfo.origin`：两个来自不同 `XIR_ALLOC` 的 obj 互不别名，
   不再 "different obj → kill all"。
3. DSE 同理：依 origin 精确判定，允许跨多个独立对象做 DSE。

**破坏**：删除 `S2L_TABLE_SIZE = 32`, `DSE_MAX_TRACKED = 32`。

### 3.7 If-Conversion 放宽

新阈值（来自 `xir_pass_limits.h`）：
- `XIR_IFCONV_MAX_INS = 6`
- `XIR_IFCONV_MAX_PHIS = 4`

再增加 **nested diamond 支持**：允许 then / else 分别再嵌套一个短 diamond。
（如果风险过高可留到阶段 3 的后半。）

### 阶段 3 验收清单
- [ ] 新增 `xir_pass_sccp.c`，老 const_prop/branch_simp/remove_unreachable 被删除
- [ ] `type_prop` 在 pipeline 中只出现一次
- [ ] `GVN_MAX_BLOCKS`、`LICM_MAX_*`、`S2L_TABLE_SIZE`、`DSE_MAX_TRACKED`、`EA_MAX_*` 全删
- [ ] `ctest`、`scripts/run_regression_tests.sh` 全过
- [ ] `tests/demos/demo_14_nbody.xr` 性能相对阶段 2 baseline **≥ +5%**
- [ ] 新增 1-2 个针对 scalar-replacement 跨块能力的 JIT 单测

---

## 阶段 4：Pipeline 重构

**动机**：阶段 3 把各个 pass 的算法做对，但 pipeline 仍是"线性、固定次数"。
阶段 4 让它**按需驱动、自动收敛**。

### 4.1 引入 FixedPoint 驱动器

新 API：
```c
typedef struct XirPassDesc {
    const char    *name;
    XirPassFn      fn;
    uint32_t       flags;     // XIR_PASS_PURE / XIR_PASS_MODIFIES_CFG / ...
} XirPassDesc;

XR_FUNC void xir_run_fixedpoint(XirFunc *func,
                                 const XirPassDesc *passes, uint32_t npass,
                                 uint32_t max_rounds);
```

`xir_run_fixedpoint` 每轮跑 passes[]，只要任一 pass 返回 `ins_changed | cfg_changed`
就继续；达到 `max_rounds` 或无 change 则停。

### 4.2 新 pipeline（层次化）

```c
void xir_run_pipeline_ex(XirFunc *func, XirOptLevel opt, XrProto *proto) {
    // --- Canonicalize: SCCP + TypeProp + SelectRep + Canon + DCE ---
    static const XirPassDesc canon[] = {
        {"select_rep",    xir_pass_select_rep,    0},
        {"sccp",          xir_pass_sccp,          XIR_PASS_MODIFIES_CFG},
        {"type_prop",     xir_pass_type_prop,     0},
        {"specialize",    xir_pass_specialize,    0},
        {"canonicalize",  xir_pass_canonicalize,  0},
        {"dce",           xir_pass_dce,           0},
    };
    xir_run_fixedpoint(func, canon, ARRAY_SIZE(canon), 5);

    if (opt < XIR_OPT_BASIC) goto finish;

    // --- Basic: CSE + copy_prop + store-to-load + escape(local) ---
    static const XirPassDesc basic[] = {
        {"cse",           xir_pass_cse,           0},
        {"copy_prop",     xir_pass_copy_prop,     0},
        {"store_to_load", xir_pass_store_to_load, 0},
        {"phi_simp",      xir_pass_phi_simp,      XIR_PASS_MODIFIES_CFG},
        {"dce",           xir_pass_dce,           0},
    };
    xir_run_fixedpoint(func, basic, ARRAY_SIZE(basic), 4);

    if (opt < XIR_OPT_FULL) goto finish;

    // --- Full: inline + loop opts + range analysis ---
    if (proto) xir_pass_auto_inline(func, proto);
    xir_run_fixedpoint(func, canon, ARRAY_SIZE(canon), 2);

    static const XirPassDesc loop[] = {
        {"licm",          xir_pass_licm,          0},
        {"elim_guards",   xir_pass_elim_guards,   0},
        {"gvn",           xir_pass_gvn,           0},
        {"gcm",           xir_pass_gcm,           XIR_PASS_MODIFIES_CFG},
        {"store_to_load", xir_pass_store_to_load, 0},
        {"dse",           xir_pass_dse,           0},
        {"ifconvert",     xir_pass_ifconvert,     XIR_PASS_MODIFIES_CFG},
    };
    xir_run_fixedpoint(func, loop, ARRAY_SIZE(loop), 3);

    // --- Cleanup ---
    xir_pass_alloc_sink(func);
    xir_pass_escape_analysis(func);
    xir_pass_insert_redefines(func);
    xir_pass_range_analysis(func);
    xir_run_fixedpoint(func, canon, ARRAY_SIZE(canon), 2);
    xir_pass_split_critical_edges(func);

finish:
    xir_pass_reorder_blocks(func);
    xir_insert_write_barriers(func);
    xir_pass_elim_write_barriers(func);
}
```

### 4.3 Pass 统计框架

在 `xir_run_fixedpoint` 中记录每个 pass 的：耗时、执行轮数、改变的指令数 / vreg 数。
`jit->verbose = true` 时打印：

```
[JIT-pipe] func=solar Tier=2
  sccp          : 2 rounds, 14ms, removed  43 ins, killed  12 blocks
  type_prop     : 2 rounds,  3ms, refined  67 vregs
  cse           : 1 rounds,  2ms, removed  11 ins
  licm          : 1 rounds,  5ms, hoisted   8 ins
  gvn           : 2 rounds,  9ms, removed  24 ins
  ...
```

### 阶段 4 验收清单
- [ ] `xir_run_pipeline` 不再出现裸 `XIR_RUN_PASS` 链式调用
- [ ] `--jit-verbose` 打印的是结构化 pass 统计
- [ ] pipeline 代码从 ~80 行减到 ~40 行
- [ ] baseline 用例的编译时间 ≤ 阶段 3（fixedpoint 不应大幅变慢）

---

## 阶段 5：Regalloc + Codegen 升级

### 5.1 Hole-aware `first_isect`

**目标**：让 LSRA 在 "range A 的 hole 期间使用同寄存器" 成为合法重用。

**改法**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_regalloc.c:318-330` 精确按 `LsInterval *` 链表
求交集；利用现有 Active/Inactive 状态机，当 A 处于 Inactive 时其 hole 期视为空闲。

**验证**：在 `tests/jit/regalloc/` 增加一个"宽 phi live range + hole 中另一个 vreg 使用"
的回归测试，断言两者分到同一寄存器。

### 5.2 Rematerialization 扩展

**现状**：`is_remat` 只识别 `XIR_CONST_*`。

**改法**：
- `XIR_LOAD_CORO_BYTE const_offset` → 冷重算；
- `XIR_MOV param_vreg`（来自参数）→ 无需 spill；
- `XIR_I2F CONST_I64`、`XIR_F2I CONST_F64`。

**破坏**：spill 代码路径同步改（`@/Users/xuxinglei/workspace/xray-lang/xray/src/jit/xir_codegen.c:124-133`）。

### 5.3 Peephole pass

新 `src/jit/xir_peephole.c`，在 codegen 最后一步扫描 `ctx->buf.code[]`：
- 连续 `STR x, [SP, #N]; LDR x, [SP, #N]` → 删后者；
- `MOV xN, xN` → 删除；
- `CMP xN, #0; B.EQ/B.NE` → `CBZ/CBNZ`；
- 立即数合并（两条 `MOV+MOVK` 合为 `MOV imm16:imm16`）。

### 5.4 Profile-guided block layout

**改法**：`xir_pass_reorder_blocks` 若有 `proto->type_feedback` 或 `proto->call_counts`，
按 edge frequency 做 trace 构造；否则退回当前贪心。

### 阶段 5 验收清单
- [ ] 去掉"range 占满整个 [start,end)"的注释
- [ ] LSRA 单测：phi hole 重用通过
- [ ] peephole 在 demo_14_nbody 上节省 ≥ 3% 代码大小
- [ ] regalloc 相关的 spill 数（verbose 日志）相对阶段 4 减少

---

## 阶段 6：编译驱动与并发

### 6.1 多线程后台编译

**改法**：`XirCompileQueue` 改为 MPMC 队列，启动 `N = min(4, nCPU-1)` 个 bg 线程。
每线程独立 `bg_code_alloc`；`install_bg_result` 已是 CAS，本身线程安全。

**接口变化**：`XJIT_QUEUE_CAPACITY` 增大到 256（原 64）；加 `n_workers` 字段。

### 6.2 TFA 模块失效

**改法**：
- `jit->tfa_ran` 改为 `per-module` 标记，放到 `XrModule::tfa_analyzed`；
- 模块加载（`xr_module_load`）后清标记，下次对该模块里的 proto 编译前重跑 TFA；
- 多模块的 TFA 结果合并到 `jit->tfa`。

### 6.3 `shared_protos` 快照（已在阶段 1.6 解决，此处确认完整覆盖）

阶段 1.6 让 bg 能用 `shared_protos`，阶段 6 检查：
- OSR 路径下的 bg 编译；
- recompile（Tier1 → Tier2）路径下的 bg 编译；
都应共享同一 `XirBgTask` 构造函数。

### 阶段 6 验收清单
- [ ] `jit->bg_queue->n_workers > 1`
- [ ] 多核机器上 JIT warmup 时间相对阶段 5 **至少减少 40%**
- [ ] 模块动态加载后 TFA 重新跑（通过日志验证）
- [ ] stress 测试 10 分钟无 race 告警（`--thread-sanitizer` 模式）

---

## 阶段 7：代码拆分（规范合规）

**目标**：所有 `.c` ≤ 3000 行，所有 `.h` ≤ 800 行，单函数 ≤ 150 行。

### 7.1 `xir_jit.c` 拆分

| 新文件 | 内容 | 大致行数 |
|--------|------|----------|
| `xir_jit.c` | `init / destroy / try_compile` 主流程 | ~800 |
| `xir_jit_call.c` | `xir_jit_call / xir_jit_resume / xir_jit_osr_*` | ~1000 |
| `xir_jit_deopt.c` | `deopt_recover / deopt_reconstruct / OSR trigger` | ~700 |
| `xir_eligibility.c` | `is_jit_eligible` + opcode 白名单（实际已在阶段 1.4 走表） | ~200 |
| `xir_jit_runtime.c`（已有） | `xr_jit_*` C helper | 现状保留 |
| `xir_jit_dominant_shape.c` | `find_dominant_shape` 等辅助 | ~200 |

### 7.2 `xir_pass_advanced.c` 拆分

| 新文件 | 内容 |
|--------|------|
| `xir_pass_inline.c` | `xir_inline_function / xir_pass_auto_inline` |
| `xir_pass_escape.c` | `xir_pass_escape_analysis / xir_pass_alloc_sink` |
| `xir_pass_type.c` | `xir_pass_type_prop / xir_pass_specialize / xir_pass_insert_redefines` |
| `xir_pass_range.c` | `xir_pass_range_analysis` |
| `xir_pass_gcm.c` | `xir_pass_gcm / xir_pass_elim_guards / xir_pass_propjnz` |

### 7.3 `xir.h` 拆分

| 新文件 | 内容 |
|--------|------|
| `xir.h` | 核心类型 + XirRef + 核心 struct（≤ 500 行） |
| `xir_api.h` | 便捷 inline helper、opcode 判定、访问器 |
| `xir_ops.h` | `XirOpcode` 枚举 + 操作数描述（当前混在 xir.h） |

### 阶段 7 验收清单
- [ ] `wc -l src/jit/*.c src/jit/*.h` 最大 ≤ 3000 / 800
- [ ] `scripts/check_architecture.sh` 无新告警
- [ ] 所有文件头注释遵循 `docs/rules/c-coding-standards.md` 模板
- [ ] `grep -c "static.*(" src/jit/*.c` 无超 150 行的函数（配合 `awk` 检查）

---

## 风险矩阵

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 阶段 3 改动过大，JIT 产生错误代码导致 deopt 风暴 | 中 | 高 | 每个 pass 替换前后都跑全量 JIT 回归；保留 `--jit-verify-after-each-pass` 调试开关 |
| 阶段 5 hole-aware 引入活跃度分析 bug | 中 | 中 | 扩展 `@/Users/xuxinglei/workspace/xray-lang/xray/docs/engineering/jit_verifier_framework.md` 框架，regalloc 后做 use-before-def 验证 |
| 阶段 6 多线程 bg 编译的隐式共享 | 高 | 高 | 必须跑 `--thread-sanitizer` 模式；阶段 6 前所有 `proto->*` 的 bg 写入都要过 code review |
| 阶段 7 拆分导致 include 循环 | 低 | 低 | 拆分前先做依赖图（`scripts/check_architecture.sh`） |
| SCCP 实现错误导致类型错误未被发现 | 低 | 高 | 保留 `XIR_RUN_PASS_STRICT_SE` 校验；SCCP 完成后立即跑 `xir_verify_types` |

## 中止 / 回退策略

- 每阶段做一次 git 标签 `jit-refactor-phaseN`。
- 阶段 N 发现不可修复缺陷 → `git reset --hard jit-refactor-phase(N-1)`，在新分支重做该阶段。
- **禁止**在下一阶段里"顺便"修上一阶段的缺陷（会把测试失败原因搞混）。

## 参考实现

- Dart Dart VM（分层 pass pipeline + FlowGraphChecker）
- V8 TurboFan（Sea of Nodes GVN + SCCP）
- LLVM（DomTree / LoopInfo / DependenceAnalysis 接口设计）
- LuaJIT（linear scan + trace）

## 不做的事（明确排除）

- ❌ 切换到 Sea of Nodes：侵入性过高、收益不足以抵消改造成本。
- ❌ 改成 graph-coloring 寄存器分配：linear scan 配合 hole-aware 已足够；graph coloring 会大幅增加编译时间。
- ❌ 引入 MLIR / LLVM bitcode：偏离 "轻量、独立、面向 xray 语义" 的项目定位。
- ❌ Trace-based JIT：当前 method-based + OSR 已能覆盖 99% 热点；trace 带来的复杂度不划算。
