# Runtime 横切复盘 V2（`030`）

> 旧版 `030-runtime-cross-cutting-recap.md` 的对账重写。
>
> V1 是高质量的总论。V2 在保留 V1 大部分跨层判断的同时：
>
> 1. 把 024-029 V2 各自发现的"具体证据"汇总到这里
> 2. 把 V1 的"四 owner 现状描述"推到"按最佳设计应当收敛到三 owner"的结论
> 3. 给出可执行的 cross-cutting P0/P1/P2 修复清单
> 4. 给每个核心 runtime type 填好统一 owner matrix

## 1. 最终总论（V2 强化）

V1 主要论断**全部被 024-029 V2 验证**：

- ✅ runtime 不是单一 GC 宇宙，而是四种 owner 并存
- ✅ `XrGCHeader` 不是 owner 真相源
- ✅ 真正的判断标准是"分配入口 / 销毁入口 / traversal 闭合 / 跨 coro 协议"
- ✅ 当前张力来自"统一外观 + 分裂 owner"

V2 推到下一步：

- **当前是四 owner 并存，但这是历史现状不是最佳设计。**
- **按 V2 各章给出的最佳设计建议执行后，应当收敛到三 owner**：per-coro Immix（默认）、sysheap shared/refcount（cross-coro）、malloc graph（控制平面/元数据）。
- **`isolate fixedgc` 应当退役为纯 bootstrap fallback**，不再作为任何"主路径分配域"。

## 2. 四 owner → 三 owner 的收敛路径

### 2.1 当前的四 owner（V1 准确）

| Owner | 入口 | 释放 | 主用途（现状） |
|---|---|---|---|
| **per-coro Immix** | `xr_alloc(coro, ...)` | Immix sweep | 主流 runtime 对象 |
| **isolate fixedgc** | `xr_gc_alloc(&isolate->gc, ...)` | `xr_gc_cleanup` 链表扫 | bootstrap、normal instance、bound method、enum type/value、exception 部分对象 |
| **sysheap shared** | `xr_sysheap_alloc_shared` | `xr_shared_decref` | channel、shared deep-copy 输出 |
| **malloc graph** | `xr_malloc/calloc/realloc`、`xr_sysheap_alloc_class` | 显式 free 或 isolate cleanup | class、symbol、reflect registry、worker / runtime / timer / netpoll、reflection wrapper |

### 2.2 V2 推荐的三 owner 收敛

| Owner | 适用对象 | 收敛动作 |
|---|---|---|
| **per-coro Immix** | array/map/set/json/string/closure/cell/iterator/datetime/bigint/range/**instance**/**exception**/task | normal instance 改走 per-coro（028/029 V2 P0）；exception 改走 per-coro（027 V2 P0）|
| **sysheap shared** | channel、shared deep-copy 输出、global string pool | 不变 |
| **malloc graph (控制平面)** | class metadata、symbol table、reflection registry、reflection wrapper（重新归类）、worker / runtime / timer / netpoll、shape registry | reflection wrapper 用 fixedgc 改为显式接入此 domain，加 destroy hook |

`isolate fixedgc` 在三域模型下退役为：

- bootstrap 阶段未建立 coro_gc 时的临时 fallback
- 不再承担任何 normal-path 分配
- `xr_gc_cleanup` 仅释放 bootstrap 残余

## 3. V2 跨章具体证据汇总

### 3.1 真实内存泄漏（按章节）

| 来源 | 对象 | 证据 |
|---|---|---|
| 027 V2 §3.5 | `XrException` | `XR_ALLOCATE = xr_malloc`，无 `g_destroy_funcs` 入口，无任何 free 路径 |
| 027 V2 §3.6 | `TypeWrapper`/`FieldWrapper`/`MethodWrapper`/`ParameterWrapper` (api.c) | `XR_ALLOCATE`，`XR_TINSTANCE` 假身份，永不释放 |
| 028 V2 §3.3 | `FieldWrapper`/`MethodWrapper` (cache.c) | `XR_ALLOCATE`，`XR_TBLOB` 不在 destroy 表，永不释放 |
| 028 V2 §3.7 | enum side tables (`value_to_index`/`symbol_to_index`/`members`) | `xr_malloc`，`g_destroy_funcs[XR_TENUM_TYPE]` 缺 destructor |
| 027 V2 §3.7 | shape internal state | `xr_gc_destroy_shape` 是 dead code（grep 验证零调用） |

每个泄漏在 long-running isolate 下持续累积。

### 3.2 注释/实现严重不符（按章节）

| 来源 | 文件:行 | promise vs 实际 |
|---|---|---|
| 026 V2 §6.1 | `xgc.h:11-14` | promise "incremental Mark-Sweep with Arena" / 实际 Immix mark-region |
| 026 V2 §6.2 | `xbc_stackmap.h:108` | promise "deduplicate bitmaps" / 实际 `xbc_stackmap.c:103` "no dedup for now" |
| 028 V2 §3.3 | `xreflect_cache.c:91` | promise "GC-managed will be reclaimed by collector" / 实际 `xr_malloc` 永不释放 |
| 028 V2 §3.2 (与 029 V2 §3.5 协同) | VM/JIT 注释 + `xinstance.c:37` | normal instance 注释成"coroutine heap" / 实际 fixedgc |

### 3.3 file-scope mutable global（按章节）

V2 全仓 grep 验证后的清单（仅 4 处）：

| 文件:行 | 标识 | 用途 |
|---|---|---|
| `xtype.c:28` | `g_types_initialized` | 进程级类型 singleton 初始化 guard（一次性） |
| `xchunk.c:36` | `s_proto_id_counter` | 全进程 proto id atomic counter |
| `xcoro_gc.c:50-52` | `g_gc_pool_mu` / `g_gc_pool_head` / `g_gc_pool_count` | 跨 isolate 共享的 L2 GC pool（mutex+链表+计数） |
| `xtype.c:31-46` | `g_type_int` 等 14 个 | frozen-after-init singletons（**只读，不算可变**） |

只有前三组是真正 mutable global。第四组在 `init_singleton` 后 `frozen=true`，跨 isolate 共享是设计意图。

### 3.4 `XrInstance` 三路径漂移（cross-章节 P0）

027 V2 + 028 V2 + 029 V2 三章共同发现：

| 路径 | owner | 入口 |
|---|---|---|
| Normal allocation | isolate fixedgc | `xinstance.c:37` `xr_gc_alloc(&isolate->gc, ...)` |
| Deep-copy to coro | per-coro Immix | `xdeep_copy.c:314` `copy_ctx_alloc → xr_coro_gc_newobj` |
| Deep-copy to shared | sysheap shared | `xdeep_copy.c:586` `xr_sysheap_alloc_shared` |

**同一类型在三种路径下落在三个 owner**。这是 V2 全局最大的漂移点，也是 V2 推荐的"normal instance 走 per-coro"修复（一行代码）的关键证据。

## 4. 统一 owner matrix（V1 §13.1 推荐的样表）

V2 用统一模板填好 24 个核心 runtime type：

### 4.1 per-coro Immix（V2 推荐"扩张到的"）

| Type | Alloc API | Destroy | Traverse | Deep-copy | Shared? | External mem |
|---|---|---|---|---|---|---|
| `XrArray` | `xr_alloc(coro, ..., XR_TARRAY)` | Immix sweep + `g_destroy_funcs[XR_TARRAY]` | yes | per-coro / shared | optional | ✅ accounting |
| `XrMap` | 同上 | 同上 | yes | per-coro / shared | optional | ❌ accounting |
| `XrSet` | 同上 | 同上 | yes | per-coro / shared | optional | ❌ accounting |
| `XrJson` | 同上 | 同上 | yes (含 shape) | per-coro / shared | optional | ❌ accounting (overflow) |
| `XrString` (non-interned) | 同上 | 同上 | yes | passthrough (intern) | n/a | inline |
| `XrClosure` | `xr_alloc` (fallback isolate) | Immix sweep | yes (upvals) | per-coro / shared | optional | inline |
| `XrCell` | 同上 | 同上 | yes (value) | n/a (per-closure) | no | inline |
| `XrIterator` | `xr_alloc(coro, ..., XR_TITERATOR)` | Immix sweep | yes (source) | n/a | no | inline |
| `XrTask` | `xr_alloc(executor, ..., XR_TTASK)` | Immix sweep + `g_destroy_funcs[XR_TTASK]` | yes (children) | n/a (不跨) | no | inline |
| `XrBigInt` | `xr_alloc` | Immix sweep | yes | TBD (V2 §5.6) | TBD | inline |
| `XrRange` | 同上 | 同上 | leaf | TBD | TBD | inline |
| `XrDateTime` | 同上 | 同上 | leaf | per-coro / shared | n/a | inline |
| **`XrInstance`** (V2 修后) | `xr_alloc(coro, ..., XR_TINSTANCE)` | Immix sweep | yes | per-coro / shared | optional | inline |
| **`XrException`** (V2 修后) | `xr_alloc(coro, ..., XR_TEXCEPTION)` | Immix sweep | yes (message/file/stack) | n/a | no | inline |
| `XrStringBuilder` | `xr_alloc` | Immix sweep | leaf | per-coro / shared | n/a | ❌ buffer accounting |

### 4.2 sysheap shared / refcount

| Type | Alloc API | Destroy | Refc | Cross-coro |
|---|---|---|---|---|
| `XrChannel` | `xr_sysheap_alloc_shared(..., XR_TCHANNEL)` | refc=0 → `xr_shared_destroy` | yes | passthrough |
| Shared `XrArray/Map/Set/Json/Instance` | `xr_sysheap_alloc_shared` | 同上 | yes | passthrough |
| Long-string interned | `xr_sysheap_alloc_shared` | 同上 | yes | passthrough |
| Global pool string ≤64 | `xr_malloc` + global pool | global pool sweep | n/a (pool managed) | accessed bit |

### 4.3 malloc graph (控制平面)

| Type | Alloc API | Destroy | GC participation |
|---|---|---|---|
| `XrayIsolate` | `xr_malloc` | `xray_isolate_delete` | header-only |
| `XrClass` | `xr_sysheap_alloc_class` (fallback malloc) | `xr_class_free` 显式 | header-only (不参与 mark) |
| `XrSymbolTable` | `xr_malloc` + hashmap | `xr_symbol_table_destroy` | none |
| `XrTypeRegistry` | `xr_malloc` | `xr_registry_free` | none |
| `XrTypePool` | arena-based | `xr_type_pool_free` | none |
| `XrShape` | `xr_calloc` | **未闭合**（V2 P1） | header-only, dead destructor |
| reflection wrappers (V2 修后) | `xr_gc_alloc(&isolate->gc, ...)` 或 fixedgc 显式 free 列表 | fixedgc cleanup with destructor | header-only |
| `XrRuntime` / `XrWorker[]` / `XrMachine[]` | `xr_calloc` | `xr_runtime_destroy` | none |
| timer wheel / netpoll backend / async pool | `xr_calloc/malloc` | `xr_worker_destroy` | none |
| `XrCoroutine` shell | runtime pool / sysheap arena / malloc | recycle 或 sysheap free | header-only (内部 coro_gc 是真 GC) |
| `XrCoroGC` | per-coroutine | `xr_coro_gc_destroy` 或 reset | self-managed |
| `XrSystemHeap` | isolate (`xr_malloc + sub-arenas`) | `xr_sysheap_destroy` | none |

### 4.4 isolate fixedgc（V2 推荐退役）

| Type | 当前用途 | V2 修复后 |
|---|---|---|
| `XrInstance` (normal) | fixedgc | → per-coro |
| `XrException` | malloc + 假身份 | → per-coro |
| `XrBoundMethod` | fixedgc | 保留 fixedgc（receiver 靠 defensive traversal）or 改 per-coro |
| `XrEnumType/XrEnumValue` | fixedgc | 保留 fixedgc + 加 destructor 释放 side tables |
| `XrCell` (fallback) | fixedgc fallback | bootstrap only |
| `XrClosure` (fallback) | fixedgc fallback | bootstrap only |

执行 V2 P0/P1 后，fixedgc 仅承担 enum + bound method + bootstrap，规模显著降低。

## 5. cross-cutting 修复清单（按优先级）

### 5.1 P0（必须修，否则真实泄漏 / 真实数据漂移）

| ID | 来源 | 动作 |
|---|---|---|
| P0-01 | 027 V2 §5.1 | `xr_exception_alloc` 改走 `xr_alloc(xr_current_coro(X), ...)` |
| P0-02 | 027 V2 §5.2 | `xr_reflect_cache_free` 加 wrapper body 释放循环 |
| P0-03 | 028 V2 §5.2 | `xreflect_api.c` 4 类 wrapper 改走 fixedgc 或显式 free 列表 |
| P0-04 | 028 V2 §5.4 | `xreflect_cache.c:91` 注释删除或修复实现（同步） |
| P0-05 | 028 V2 §5.1 | `xr_instance_new` 改走 `xr_alloc(coro, ..., XR_TINSTANCE)` |
| P0-06 | 029 V2 §5.4 / `coro_audit_plan.md` CORO-01 | `xr_runtime_wake_channel*` cross-worker 改走 inbox |
| P0-07 | 026 V2 §6.4 | `g_gc_pool_*` 迁移到 `XrSystemHeap` per-isolate |

### 5.2 P1（重要修复）

| ID | 来源 | 动作 |
|---|---|---|
| P1-01 | 026 V2 §6.1 | `xgc.h:11-14` 头注释重写（删除 Mark-Sweep 表述） |
| P1-02 | 026 V2 §6.2 | `xbc_stackmap` dedup 实现或文档对齐 |
| P1-03 | 026 V2 §6.6 | `bound_method_stub` 改为显式 throw |
| P1-04 | 027 V2 §5.3 | `xr_shape_registry_destroy` 调 `xr_gc_destroy_shape` |
| P1-05 | 027 V2 §5.5 | external accounting 一致化（map/set/json/stringbuilder） |
| P1-06 | 028 V2 §5.3 | `g_destroy_funcs[XR_TENUM_TYPE]` 加 destructor |
| P1-07 | 029 V2 §3.6 | task tree + scope child list 合并 |
| P1-08 | 029 V2 §3.7 | 拆 `xr_coro_wake_waiter` |
| P1-09 | 025 V2 §5.2 | 拆 `XrProto` god struct |
| P1-10 | 026 V2 §6.7 | `mark_struct_string_fields` 加 fail-fast assertion |

### 5.3 P2（清理 / 一致性）

| ID | 来源 | 动作 |
|---|---|---|
| P2-01 | 024 V2 §6.x | TLS isolate 在 value 层 6 个穿透点改显式参数 |
| P2-02 | 025 V2 §5.4 | `xstruct_layout.h:118` 用枚举常量 |
| P2-03 | 025 V2 §5.5 | `xr_valuearray_add` 改 shallow eq |
| P2-04 | 026 V2 §6.8 | GC traverse 函数表化 |
| P2-05 | 026 V2 §6.9 | 删除 `root_callbacks`（YAGNI） |
| P2-06 | 027 V2 §5.4 | 删除 `XR_SET_FLAG_ENTRIES_ON_GC` |
| P2-07 | 027 V2 §5.6 | 合并 `g_destroy_funcs` + `has_refs` → `g_type_ops` |
| P2-08 | 028 V2 §5.5 | `XR_TBLOB` 决策（进 GC 系统或删除用法） |
| P2-09 | 024 V2 §6.10 | debug_hooks accessor 强约束 |
| P2-10 | 029 V2 §5.6 | deep-copy 明确所有 GC 类型的跨 coro 行为 |

## 6. 关键 V1→V2 收敛点

| 点 | V1 立场 | V2 立场 |
|---|---|---|
| 四 owner | 现状描述 | 推到三 owner 收敛建议 |
| `XrGCHeader` 角色 | "不是真相源" | 保留 + 加伪 GC 对象具体清单 |
| pseudo-GC 对象 | "稳定风险源" | 列出 5 类具体对象 + 修复路径 |
| external accounting | "局部有效" | P1 一致化建议 |
| 注释漂移 | "提到风险" | 列 4 条具体 promise vs 实际不符 |
| instance 漂移 | "实现/叙述偏差" | 三路径漂移精确 + 一行修复 |
| improvement 建议 | 4 条方向 | 30 条 P0/P1/P2 可执行清单 |

## 7. V2 级"如果只允许 5 处修改" 优先级

按收益/代价比，V2 推荐前 5 个动作：

1. **`xr_instance_new` 改走 per-coro Immix**（一行）—— 同时消除 027/028/029 的三路径漂移。
2. **`xr_exception_alloc` 改走 per-coro Immix**（一行）—— 同时消除真实 throw-leak。
3. **`xr_reflect_cache_free` 加 wrapper 释放循环**（10 行）—— 修复永久泄漏。
4. **`g_gc_pool_*` 迁到 `XrSystemHeap`**（~50 行）—— 消除最后一处真正的 process-wide mutable global。
5. **`xreflect_cache.c:91` 注释 + 实现同步**（任选其一）—— 解除文档/实现严重不符的最大单点。

这 5 处修复后，runtime 的 owner 模型在 95% 路径下统一到"per-coro / shared / malloc graph 元数据"三域，符合 V2 §2.2 推荐的最佳设计。

## 8. 给后续 audit 的输入

V1 没专门写。V2 列出尚未完成的子领域 audit：

- `xclass_descriptor.c` / `xclass_from_descriptor.c` 的 owner（028 V2 §5.7）
- `xclass_reflect.c` 的 owner（028 V2 §5.8）
- `xreflect_members/method/type.c` 的详细 owner
- netpoll 各 backend 的 lifecycle（029 V2 §10）
- timer wheel destroy 路径（与 CORO-07 协同）
- socket / async / balance / yieldable 子系统
- `XrTask.on_completion` listener 同步契约（029 V2 §5.5）
- write barrier 在各 mutator 的覆盖率（026/027 V2 §5.5 协同）

## 9. 与 V1 的总差异

| 维度 | V1 | V2 |
|---|---|---|
| 论断风格 | 现状描述 + 风险列表 | 现状 + 收敛建议 + 修复清单 |
| 跨章证据汇总 | 没有 | 5 表（泄漏/注释漂移/global state/instance 漂移/owner matrix） |
| 改进路径 | 抽象方向（matrix / pseudo-GC 单列 / 消差异） | 30 条具体 ID + 优先级 + 来源 |
| owner 数量收敛建议 | 维持 4 类 | 收敛到 3 类，fixedgc 退役 |
| 优先级 | 没有 | P0 / P1 / P2 三档 |
| 5 处修改优先级 | 没有 | 显式列出 |

## 10. 一句话总结

V1 已经准确定性 runtime 是"多 owner 协议共存"。V2 的贡献是把"为什么共存"和"应该收敛到什么"用 30 条可执行修复落到代码层。

> **Xray runtime 当前是四 owner 并存的真实历史现状，最佳设计是收敛到 per-coro Immix（默认） + sysheap shared（cross-coro） + malloc graph（控制平面）三 owner，让 isolate fixedgc 退役为纯 bootstrap fallback。**

## 11. 本轮状态

- **已完成**：6 章 V2 全部完成（024-029 + 030）；跨章证据汇总；30 条 P0/P1/P2 修复清单；统一 owner matrix 实例化
- **未完成**：尚未实施任何修复（按用户要求"不动源码"）；尚未做 V1 vs V2 全面对比 / 取舍报告（todo list 中的 `compare`）
- **下一步**：等用户确认是否进入"取长补短"阶段；或开始按 P0 优先级实施修复
