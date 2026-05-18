# GC 模块重构计划（`src/runtime/gc/`）

**开发原则**：
- ⚡ **不考虑向后兼容**，Xray 无外部用户，直接采用最佳设计
- ✅ 避免临时 workaround 与兼容层，每一步落到"长期最优"
- ✅ 每阶段结束必须 `scripts/run_regression_tests.sh` 全绿才能合并
- ✅ 每阶段 commit 粒度 1~3 次，不留半成品状态
- ✅ 所有新代码带 `XR_DCHECK` 断言 + 同步更新 `docs/rules/gc-memory.md`

---

## 当前基线（2026-04-19）

| 文件 | 行数 | 职责 |
|------|------|------|
| `xcoro_gc.c` | 1934 | Per-coroutine 三色增量 Mark-Sweep + Sticky Immix |
| `ximmix.c` | 582 | 16KB block / 128B line 标记区分配器 |
| `xcoro_gc_traverse.c` | 348 | 按 `XrObjType` 分发遍历 |
| `xsystem_heap.c` | 266 | 共享对象/class arena/coro pool |
| `xgc.c` | 217 | 全局 fixedgc（大多已僵尸） |

**技术债 24 项概要**（详见 2026-04-18 代码审阅）：

- **正确性**：`sweep_phase`/`sweep_block` 字段存在但从未使用（假增量）；`xr_immix_destroy` 漏置 `old_blocks=NULL`；注释/代码阈值不一致（70% vs 400%）
- **性能**：SWEEP 一次性全扫（长暂停）；`prop_count` 步长与 `GCdebt` 脱耦；栈保守全扫；post-alloc 逻辑重复 2 处
- **架构**：全局 `xr_gc_markobj` 系列是僵尸 API；两套遍历分发并存；`xr_coro_gc_grow_stack` 职责越界；魔数 `0x04` 混用
- **内存**：block cache 有 ABA 窗口；gray list 不缩容；大对象无 mmap 分级；worker 间 GC struct 不均衡
- **观测**：只有聚合计时；缺少 INV-5/6/7；无 stress 模式

---

## 阶段总览

| # | 阶段 | 风险 | 工时 | 关键收益 |
|---|------|------|------|----------|
| P1 | 代码卫生 & 僵尸 API 清理 | 极低 | 0.5 d | 删 ~150 行死代码，修正注释与魔数 |
| P2 | 结构解耦与公共辅助抽离 | 低 | 1 d | 消除重复、合并两套分发、`grow_stack` 迁离 |
| P3 | 真正的增量 Sweep 状态机 | 中 | 1.5 d | SWEEP 暂停"全量 → 可控步长" |
| P4 | 基于债务的步长 + 策略调参 | 低 | 1 d | 分配/回收吞吐自动匹配，major GC 阈值合理化 |
| P5 | Thread-local Block Cache | 中 | 1 d | 消除 ABA、减少原子操作 |
| P6 | 大对象 mmap 分级 & Gray list shrink | 低 | 0.5 d | 大对象碎片归零、内存不驻留 |
| P7 | 字节码精准栈扫描（Stackmap） | 高 | 2 d | 栈扫描 O(深度) → O(活跃指针) |
| P8 | 观测性增强 + `XR_GC_STRESS` | 低 | 0.5 d | 调优数据完整、CI 可压测 |
| P9 | 跨 Worker GC Struct 二级池 | 中 | 0.5 d | 消除 worker 间 GC 缓冲不均 |

**总计 8.5 天**。P1→P2→P3→P4 串行骨架；P5/P6/P9 可在 P4 后并行；P7 单独推进（跨模块）；P8 随时可插入。

---

## P1：代码卫生 & 僵尸 API 清理

### 动机

1. **僵尸全局 GC API**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/gc/xgc.c:94-105` 的 `xr_gc_markobj` 只做 white→gray 一次染色，**没有 propagate loop 消费 gray**。对应 `g_traverse_funcs` 从未被调用。
2. **注释漂移**：
   - `xcoro_gc.h:266` "FINALIZE removed" stale
   - `xcoro_gc.c:1452` 注释 "400%"，函数头说 "70%"
   - `xcoro_gc.c:1145` `XGC_PROMOTE_THRESHOLD 40` 实际 ≈ 40.16%
3. **魔数 0x04**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/gc/xsystem_heap.c:248` 用裸数字，应 `STR_FLAG_GLOBAL`
4. **`xr_immix_destroy` 漏 NULL**：`ximmix.c:222-232` free 了 `old_blocks` 但未置 NULL

### 实施步骤

1. **删除全局 mark API**
   - 删 `xgc.c:74-105` 的 `reallymarkobject` / `xr_gc_markobj` / `xr_gc_markvalue`
   - 删 `xgc.c:42-45` 的 `g_traverse_funcs[]` 数组
   - 删 `xgc.c:66-68` 的 `get_traverse_func`
   - 删 `xgc.h:34` 的 `#define xr_gc_mark_object` 宏
   - 删 `xgc_internal.h` 中 `g_traverse_funcs` 声明
   - `grep_search xr_gc_markobj` 确认无其他调用者

2. **修正注释**
   - `xcoro_gc.h:266` → `// GC phase: PAUSE/PROPAGATE/ATOMIC/SWEEP`
   - `xcoro_gc.c:1450-1453` 函数头注释 "70%" → "configurable via XGC_MAJOR_TRIGGER_PCT"（代码值 P4 改）
   - `xcoro_gc.c:1145` 宏注释改为 "percentage of live lines to trigger promotion"

3. **统一魔数**
   - 确认 `STR_FLAG_GLOBAL` 在 `src/runtime/object/xstring.h` 导出
   - `xsystem_heap.c:248` 改 `(obj->extra & STR_FLAG_GLOBAL)`
   - 全仓 `grep "extra & 0x"` 修复同类问题

4. **ximmix destroy 补 NULL**
   - `ximmix.c:222` 后加 `heap->old_blocks = NULL;`

### 验证

```bash
cd build && ctest --output-on-failure
scripts/run_regression_tests.sh
grep -rn "xr_gc_markobj\|g_traverse_funcs" src/   # 预期: 0
grep -rn "extra & 0x04" src/                       # 预期: 0
```

**风险**：极低。所有改动零行为变化，经 grep 确认无外部依赖。

---

## P2：结构解耦与公共辅助抽离

### 动机

1. **Post-alloc 逻辑重复**：`xr_coro_gc_newobj`（`xcoro_gc.c:272-314`）与 `xr_jit_alloc_post`（`xcoro_gc.c:1817-1846`）6 步几乎完全相同。JIT 侧已经漂移过一次（后加 `has_finalizers` 标记）。
2. **两套遍历分发**：switch-case（`xcoro_gc_traverse.c:230`）与 `g_traverse_funcs` 表。P1 删表后 P2 统一到 switch。
3. **`xr_coro_gc_grow_stack` 越界**：`xcoro_gc.c:1848-1933` 处理协程栈布局，与 GC 无关。
4. **初始化字段漂移**：`xr_coro_gc_create` vs `xr_coro_gc_reset` 两处字段列表不对齐。

### 目标设计

**新增静态 inline 辅助**（`xcoro_gc.c` 头部 helper 区）：

```c
// 1. Immix post-alloc 6 步共用
static inline void gc_post_immix_alloc(XrCoroGC *gc, XrGCHeader *obj, uint32_t total);

// 2. Runtime state 重置（create/reset 共用；tuning/owner/buffers 不动）
static void gc_init_runtime_state(XrCoroGC *gc);

// 3. Large 对象判断
static inline bool is_large_obj(uint32_t size);
```

**函数迁移**：
- `xr_coro_gc_grow_stack` → `src/coro/xcoro.c`（或新建 `xcoro_stack.c`，看 `xcoro.c` 当前行数）
- 声明从 `xcoro_gc.h` 移到 `coro/xcoroutine.h`

**分发统一**：`xr_gc_traverse_object` switch 补 `XR_TCOROUTINE` / `XR_TCHANNEL` case；删除 `XrGCTraverseFn` typedef 及表驱动路径。

### 实施步骤

1. **抽 `gc_post_immix_alloc`**：替换 `xr_coro_gc_newobj:273-292` 和 `xr_jit_alloc_post:1817-1846`
2. **抽 `gc_init_runtime_state`**：集中 22 个字段的清零，`create`/`reset` 共用
3. **迁移 `grow_stack`**：移到 `src/coro/xcoro_stack.c`（若 `xcoro.c` > 2500 行）
4. **分发统一**：`xcoro_gc_traverse.c` switch 补 case，include 相应头文件

### 验证

```bash
cd build && ctest --output-on-failure
cd build && ctest -R jit --output-on-failure   # JIT 专项
wc -l src/runtime/gc/xcoro_gc.c   # 预期减少 ~100 行
```

**风险**：低。纯等价重构。注意 `gc_post_immix_alloc` 必须真 inline（`__attribute__((always_inline))`），否则 JIT 退化。

---

## P3：真正的增量 Sweep 状态机

### 动机

`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/gc/xcoro_gc.c:1516-1537` 的 SWEEP case **一步内** 扫完所有 block + large + reclaim。对 10k block 的堆（~160MB）单次可能 10~50ms 暂停，违背 "incremental" 承诺。现存字段 `sweep_phase`/`sweep_block` 是摆设。

### 目标设计

**SWEEP 子状态枚举**（复用 `gc->sweep_phase`）：

```c
typedef enum {
    XGC_SWEEP_FULL_BLOCKS    = 0,
    XGC_SWEEP_RECYCLE_BLOCKS = 1,
    XGC_SWEEP_CURRENT_BLOCK  = 2,
    XGC_SWEEP_LARGE_OBJECTS  = 3,
    XGC_SWEEP_RECLAIM        = 4,
    XGC_SWEEP_DONE           = 5
} XrSweepPhase;
```

**单步预算**：每次进入 SWEEP 最多处理 `K` 个单位（1 block 或 32 large 对象）。

**关键约束**：SWEEP 期间 mutator 仍分配。新增 block 字段 `uint8_t swept_this_cycle : 1`：
- atomic 开始时清零所有 block 的该位
- `sweep_block` 结束时置位
- 分配路径 `try_current_block_hole` 进入某 recycle block 前，若 `!swept_this_cycle` 先同步 sweep 该 block 再用

**STW 路径保留**：`entergen()` / `fullgc()` 继续一次性全 sweep（已经是 STW 语义），只有 `xr_coro_gc_step` 的 INC 模式 SWEEP 走增量。

### 实施步骤

1. 定义 `XrSweepPhase` 枚举 + `static int sweep_one_unit(XrCoroGC *gc)` + `static int sweep_step_budget(XrCoroGC *gc)`
2. 重写 `XGC_SWEEP` case 为 budget 循环
3. 新增 `gc->sweep_large_cursor`（`XrGCHeader*`），大对象增量扫
4. `XrImmixBlock` 加 `swept_this_cycle` 位；atomic 末尾遍历清零
5. 分配路径加同步 sweep 保护（`try_recycle_blocks` / `try_current_block_hole` 进入块前检查）
6. 抽 `finalize_sweep(gc)`：计时 / `gc_shared_refs_end` / `setpause` / `xr_gc_verify_invariants`

### 验证

```bash
cd build && ctest --output-on-failure

# 压力测试（构造 10k 死对象）
build/xray tests/demos/demo_14_nbody.xr
# 观察 last_gc_time_ns 单步最大应 < 2ms

# Debug 模式
cmake -DCMAKE_BUILD_TYPE=Debug -DXR_GC_DEBUG=1 -B build-debug
cd build-debug && ctest --output-on-failure
```

**风险**：中。最大风险是"未 sweep 的 block 被分配新对象覆盖"→ UAF。
- 对策：`XR_GC_DEBUG` 加 INV-8 "所有被分配的 block 必须是 swept_this_cycle=1"
- 开发期可临时用 abort 断言，稳定后降为 `XR_DCHECK`

---

## P4：基于债务的步长 + 策略调参

### 动机

`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/gc/xcoro_gc.c:1499` 的 `prop_count = 5 + stepmul/20`（默认 15）与 `GCdebt` 完全解耦。mutator 每毫秒分配 1MB 时 mark 永远追不上。
此外 `check_minor_to_major` 阈值 400% 过高，major GC 过度延迟 → 老对象堆积。

### 目标设计

**Mark 步长**（参考 Lua 5.4）：

```c
static int64_t mark_step_size(XrCoroGC *gc) {
    int64_t debt = gc->GCdebt > 0 ? gc->GCdebt : 0;
    int64_t work = debt * gc->gc_stepmul / 100;
    if (work < XGC_MARK_STEP_MIN) work = XGC_MARK_STEP_MIN;
    if (work > XGC_MARK_STEP_MAX) work = XGC_MARK_STEP_MAX;
    return work;
}

case XGC_PROPAGATE: {
    int64_t budget = mark_step_size(gc);
    int64_t scanned = 0;
    while (gc->gray.count > 0 && scanned < budget) {
        scanned += gc->gray.items[gc->gray.count - 1]->objsize;
        propagatemark(gc);
    }
    if (gc->gray.count == 0) gc->gcstate = XGC_ATOMIC;
    break;
}
```

**Sweep 步长**：同理，`sweep_step_budget` 基于 debt 计算。

**策略常量宏化**（`xcoro_gc.h`）：

```c
#define XGC_PROMOTE_THRESHOLD_PCT    40
#define XGC_MAJOR_TRIGGER_PCT       150   // 从 400% 降到 150%
#define XGC_MARK_STEP_MIN          4096
#define XGC_MARK_STEP_MAX    (256 * 1024)
#define XGC_SWEEP_UNITS_MIN           4
#define XGC_SWEEP_UNITS_MAX         128
#define XGC_PAUSE_MIN                50
#define XGC_PAUSE_MAX               400
```

### 实施步骤

1. 定义宏常量集中到 `xcoro_gc.h`，删除 `xcoro_gc.c` 内 inline 值
2. 替换 PROPAGATE 步长（`:1498-1508`）为 budget-based 循环
3. 替换 SWEEP 步长（P3 的 `sweep_step_budget`）
4. `check_minor_to_major` 用 `XGC_MAJOR_TRIGGER_PCT`（400 → 150）
5. 修复 `alloc_since_gc` 双重重置：集中到 `finalize_sweep` / `finalize_minor_gc`

### 验证

```bash
cd build && ctest --output-on-failure
time build/xray tests/demos/demo_14_nbody.xr
# 预期：GC 占比 -20%~-40%（mark 吞吐跟上分配）
# 堆峰值 RSS -30%（major GC 更积极）
```

**风险**：低。步长有 min/max clamp；阈值调整可能让吞吐 ±5%，但内存峰值显著下降。

---

## P5：Thread-Local Block Cache

### 动机

`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/gc/ximmix.c:70-91` 无 tag 的 lock-free stack 有经典 ABA 窗口。所有 coroutine 竞争同一个 atomic head，扩展性差。

### 目标设计

**两级缓存**：
- **L1（per-worker）**：`XrWorkerProcessor` 加 `block_cache[8]`，无锁
- **L2（全局）**：`pthread_mutex_t` 保护的栈，上限 64
- Worker 退出时 flush L1 → L2

```c
// L1 路径无原子：worker 内单线程
// L2 路径加锁：消除 ABA
static XrImmixBlock* block_get(void) {
    XrWorker *w = xr_current_worker();
    if (w && w->p.block_cache_count > 0)
        return w->p.block_cache[--w->p.block_cache_count];
    pthread_mutex_lock(&g_cache_mu);
    XrImmixBlock *b = g_cache_head;
    if (b) { g_cache_head = b->next; g_cache_count--; }
    pthread_mutex_unlock(&g_cache_mu);
    if (b) return b;
    char *data = alloc_aligned_block();
    return (XrImmixBlock*)data;
}
```

### 实施步骤

1. `src/coro/xworker.h` 扩展 `XrWorkerProcessor`：`block_cache[8]` + `block_cache_count`
2. `xr_worker_init` 清零；`xr_worker_destroy` flush 到 L2
3. 重写 `ximmix.c:44-91` 的 cache 函数为两级
4. 删除 `<stdatomic.h>` 对 block cache 的依赖（原子类型改普通）

### 验证

```bash
cd build && ctest --output-on-failure

# TSan 压测
cmake -DCMAKE_BUILD_TYPE=Debug -DXRAY_SANITIZER=tsan -B build-tsan
# 写多协程并发分配测试
build-tsan/xray tests/gc/mt_stress.xr
# 预期：无 TSan warning
```

**风险**：中。Worker 结构体修改影响面较大。
- `xr_current_worker()` 返回 NULL 时跳过 L1，功能正确
- atexit 清理前 worker 需全部退出（否则 L1 block 泄漏）

---

## P6：大对象 mmap 分级 & Gray list shrink

### 动机

1. **大对象无 mmap 分级**：`xcoro_gc.c:267-271` 对 >4KB 统一 `xr_malloc`，100KB+ 对象频繁 alloc/free 造成 libc 堆碎片化。`xsystem_heap` 已有 64KB mmap 分级可对齐。
2. **Gray list 不缩容**：长寿命 main coro 一次大型 job 后 `gray.capacity` 稳定在高水位。
3. **大对象无字节统计**：调优时看不到大对象占多少。

### 目标设计

**三级分配**（`xr_coro_gc_newobj`）：
- `total ≤ 4KB` → Immix
- `4KB < total < 256KB` → `xr_malloc` + `large_objects`
- `total ≥ 256KB` → `mmap` + `large_objects`（flag 标 `XR_GC_FLAG_MMAP`）

**`XR_GC_FLAG_MMAP` 位置迁移**：从 `marked` 字段（与 tri-color 位冲突风险）搬到 `extra` 字段空闲位。

**Gray list shrink**：`finalize_sweep` 末尾调用：
```c
static void maybe_shrink_graylist(XrGCGrayList *list) {
    if (list->capacity > 64 && list->capacity > list->peak * 4) {
        int newcap = list->peak * 2;
        if (newcap < 64) newcap = 64;
        XrGCHeader **p = xr_realloc(list->items, newcap * sizeof(XrGCHeader*));
        if (p) { list->items = p; list->capacity = newcap; }
    }
    list->peak = 0;
}
```

### 实施步骤

1. `xcoro_gc.h` 加 `#define XR_MMAP_THRESHOLD (256 * 1024)`
2. 审查 `xgc_header.h` 位布局，把 `XR_GC_FLAG_MMAP` 从 `marked` 搬到 `extra`（新 bit 位）
3. `xr_coro_gc_newobj` 大对象分支分两档：malloc / mmap
4. `gc_free_large_objects` / `sweeplargeobjects` 释放时判断 flag 选 `xr_free` 或 `munmap`
5. `XrCoroGC` 加 `int64_t large_bytes`，alloc/free 同步更新
6. `XrGCGrayList` 加 `int peak`；`xr_gclist_push` inline 中更新
7. `finalize_sweep` 末尾调 `maybe_shrink_graylist(&gc->gray)` / `&gc->grayagain`
8. `xr_coro_gc_print_stats` 输出 `large_bytes`

### 验证

```bash
cd build && ctest --output-on-failure

# 大对象循环压测
cat > /tmp/big.xr <<'EOF'
for (var i = 0; i < 100; i++) {
    var big = make_array(1000000)   // ~8MB
    big = null
}
EOF
build/xray /tmp/big.xr
# RSS 应稳定，不随循环累积
```

**风险**：低。mmap 路径参考 `xsystem_heap.c` 已有实现。注意 flag 位迁移需同步修改 `xsystem_heap.c:134` 的写入。

---

## P7：字节码精准栈扫描（Stackmap）

### 动机

`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/gc/xcoro_gc.c:590-595` 保守全扫所有槽，O(栈深度 × 帧数)。JIT 侧已有 stackmap（`:673-816`）做精准扫描，interpreter 应对齐。

### 目标设计

**编译期生成** `XrBytecodeStackMap`：

```c
typedef struct XrBytecodeStackMap {
    uint32_t pc_count;
    uint32_t *pc_list;            // sorted, safepoint PCs
    XrBytecodeStackMapEntry *entries;
} XrBytecodeStackMap;

typedef struct XrBytecodeStackMapEntry {
    uint32_t bitmap_offset;       // 指向共享位图池
    uint16_t live_slots_count;
    uint16_t num_words;
} XrBytecodeStackMapEntry;
```

**Safepoint 位置**：MVP 只在"明确的分配指令"后记录：
- `OP_NEWARRAY` / `OP_NEWMAP` / `OP_NEWJSON` / `OP_CONCAT`
- `OP_FORIN_INIT` / 其他显式 alloc
- CALL / INVOKE **暂不记录**（回退保守扫）

**运行时查询**：
```c
// mark_coro_roots 字节码帧分支
uint32_t pc = frame->savedpc - frame->closure->proto->code;
XrBytecodeStackMapEntry *e = stackmap_lookup(proto->stackmap, pc);
if (e) {
    // 精准扫
    for_each_bit(bitmap, slot) {
        xr_coro_gc_markvalue(gc, stack[base + slot]);
    }
} else {
    // 回退保守扫（CALL 等未记录的 PC）
    for (i = base; i < frame_end; i++)
        xr_coro_gc_markvalue(gc, stack[i]);
}
```

### 实施步骤

1. 新建 `src/runtime/gc/xbc_stackmap.{h,c}`：结构定义 + encode/decode + 二分 lookup
2. `src/frontend/codegen/xcompiler.c` 在生成 alloc 指令处调 `codegen_record_safepoint(cc, live_slots)`，`live_slots` 来自 analyzer liveness
3. `XrProto` 加 `XrBytecodeStackMap *stackmap` 字段
4. `mark_coro_roots` 的字节码帧分支改用 stackmap（有则精准，无则保守）
5. AOT / 模块缓存若涉及 `XrProto` 持久化，stackmap 同步 encode/decode（破坏向后兼容，符合项目原则）
6. 新测试：`tests/gc/test_stackmap_precise.xr`

### 验证

```bash
cd build && ctest --output-on-failure
time build/xray tests/demos/demo_14_nbody.xr
# 预期 mark 时间 -40%
```

**风险**：高。跨 frontend/codegen + runtime/gc + vm 三模块。
- 最大风险：漏标活跃槽 → UAF
- 对策：
  - MVP 只处理 4~6 种明确 alloc 指令
  - 其余 PC 继续保守扫
  - `XR_GC_DEBUG` 加 INV-9："precise ⊆ conservative"
- **可推迟**：若 P1~P6 已充分缓解压力，P7 可下一 sprint

---

## P8：观测性增强 + `XR_GC_STRESS`

### 动机

统计只有聚合计时；无 stress 开关，CI 难暴露 write barrier 漏洞。

### 目标设计

**新增统计字段**：
```c
// XrCoroGC 扩展
uint64_t mark_time_ns, sweep_time_ns, finalize_time_ns;
uint32_t objects_marked, objects_swept, objects_finalized;
uint32_t objects_promoted_young_to_old;
uint32_t rem_set_size, rem_set_hits;       // remembered set 命中率
```

**`XR_GC_STRESS` 编译开关**：
```c
// xgc_header.h
#ifndef XR_GC_STRESS
#define XR_GC_STRESS 0
#endif

// xr_coro_gc_newobj 末尾
#if XR_GC_STRESS
    if (gc->gc_disabled == 0 && !gc->in_gc)
        xr_coro_gc_step(gc);  // 每次分配都 step
#endif
```

**补充 Invariants**：
- **INV-5**：黑对象无白子节点（`xr_gc_verify_invariants` 已预留注释）
- **INV-6**：`totalbytes == Σ block.alloc_bytes + large_bytes`
- **INV-7**：`object_count == Σ block.alloc_count + count(large)`

**CLI**：`xray --gc-stats` 输出所有统计字段。

### 实施步骤

1. 扩展 `XrCoroGC` 字段；在 mark / sweep / finalize 各阶段起止打时间戳
2. 新增 `xr_gc_verify_invariants` INV-5/6/7
3. `CMakeLists.txt` 加 `XRAY_GC_STRESS` 选项（`add_compile_definitions(XR_GC_STRESS=1)`）
4. `xray --gc-stats` CLI 参数
5. CI 增一个 `build-stress` 矩阵，跑 smoke 测试

### 验证

```bash
cmake -DXRAY_GC_STRESS=ON -B build-stress
cd build-stress && ctest --output-on-failure   # stress 模式全绿说明无 barrier 漏洞
```

**风险**：低。纯观测性/开关，不改 hot path 行为（stress 只在非默认编译开启）。

---

## P9：跨 Worker GC Struct 二级池

### 动机

`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/gc/xcoro_gc.c:78-86` worker 本地 `gc_free_list` 上限 256，**无跨 worker 迁移**。高频 spawn/退出时 worker A 持 256 个闲置、worker B 不停 `xr_malloc` → 不均衡。

### 目标设计

**两级池**：
- **L1 per-worker**（现有）：上限 32（从 256 降，避免 worker 囤积）
- **L2 全局**：`pthread_mutex_t` 保护的栈，上限 256

Worker 本地满时溢出到 L2；本地空时从 L2 取。

### 实施步骤

1. `xcoro_gc.c` 新增 `static pthread_mutex_t g_gc_pool_mu` + `static XrCoroGC *g_gc_pool_head` + count
2. `xr_coro_gc_create` 本地空时先取 L2 再分配
3. `xr_coro_gc_destroy` 本地满时推 L2；L2 满时真释放
4. worker 退出时 flush L1 → L2
5. `xcoro.c` 中的 coro 重置路径（`:826-830`）复用 reset 路径，无需改动

### 验证

```bash
cd build && ctest --output-on-failure
# spawn 密集测试
build/xray tests/coroutine_safety/*.xr
```

**风险**：中。路径与 P5 类似，需确保 mutex 正确、atexit 清理前 worker 全退出。

---

## 跨阶段检查清单

每合并一个阶段 PR 前必做：

```bash
# 1. 快速回归
cd build && ctest --output-on-failure

# 2. 完整回归
scripts/run_regression_tests.sh

# 3. Debug + invariants
cmake -DCMAKE_BUILD_TYPE=Debug -DXR_GC_DEBUG=1 -B build-debug
cd build-debug && ctest --output-on-failure

# 4. Sanitizer（阶段涉及内存改动时必做）
/build-asan  # 对应 workflow
/build       # tsan：对应 workflow

# 5. 性能回归（P3/P4/P7 必做）
time build/xray tests/demos/demo_14_nbody.xr

# 6. Memory benchmark（P3/P4/P6 必做）
/usr/bin/time -l build/xray tests/demos/demo_14_nbody.xr   # macOS: peak RSS
```

**每阶段交付物**：
- 代码 PR + 通过 CI
- `docs/rules/gc-memory.md` 相应条目更新
- 若涉及新字段/常量：`xcoro_gc.h` 注释中写明单位和用途
- 若涉及接口变更：全仓 grep 确认调用方已同步

---

## 风险回滚策略

每阶段单独成 PR（不串接），出问题可独立 revert：
- P1/P2/P8：纯重构/清理，revert 零副作用
- P3：若 INV-8 断言频繁触发，revert 该 PR，SWEEP 回到原全量路径
- P4：若性能/内存回退，单独 revert，策略常量可精调
- P5/P9：若 TSan 报 race，revert 该 PR，回到 atomic 版本
- P6：若 flag 位冲突导致 GC 异常，revert + 重审位布局
- P7：跨模块高风险，分 3 个子 PR（数据结构 / codegen 记录 / runtime 查询），任一出问题只 revert 该子 PR

---

## 预期总收益

| 指标 | 当前 | 目标（全部完成后） |
|------|------|--------------------|
| 单次 SWEEP 最大暂停 | 10~50ms（10k blocks） | < 2ms |
| Mark 吞吐 | 15 对象/step 固定 | 与分配速率匹配 |
| 栈扫描开销（interpreter） | O(stack_depth × frames) | O(活跃指针数) |
| 大对象 RSS 驻留 | 不归还物理页 | ≥256KB 归还 |
| Worker 间 GC 缓冲均衡 | 不均衡 | 两级池自动均衡 |
| Gray list 高水位驻留 | 永不缩容 | 每 cycle 自动 shrink |
| 代码行数 | ~3500 | ~3200（删僵尸 + 抽辅助） |
| 可观测统计维度 | 2 | 10+ |
| Block cache 正确性 | ABA 窗口 | 无并发漏洞 |

---

## 附录：代码位置速查

| 主题 | 文件:行号 |
|------|-----------|
| `xr_coro_gc_newobj` | `xcoro_gc.c:258-317` |
| `xr_jit_alloc_post`（需合并） | `xcoro_gc.c:1817-1846` |
| SWEEP case（需改增量） | `xcoro_gc.c:1516-1537` |
| PROPAGATE 步长 | `xcoro_gc.c:1498-1508` |
| `check_minor_to_major` | `xcoro_gc.c:1450-1454` |
| 栈扫描（需改精准） | `xcoro_gc.c:590-595` |
| Block cache（ABA） | `ximmix.c:44-91` |
| Large 对象分配 | `xcoro_gc.c:267-271` |
| `xr_immix_destroy`（漏 NULL） | `ximmix.c:213-233` |
| `xr_coro_gc_grow_stack`（迁离） | `xcoro_gc.c:1848-1933` |
| 全局 mark（僵尸） | `xgc.c:74-105` |
| 魔数 `0x04` | `xsystem_heap.c:248` |
| `sweep_phase` 死字段 | `xcoro_gc.h:278-280`、`xcoro_gc.c:241,1093` |
| `XrCoroGC` 结构体 | `xcoro_gc.h:255-323` |
| `XrImmixBlock` 结构体 | `ximmix.h:77-101` |
| `XrGCHeader`（16B） | `xgc_header.h:106-114` |
