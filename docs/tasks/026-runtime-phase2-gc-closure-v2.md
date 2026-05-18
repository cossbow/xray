# Runtime `gc/ + closure/` 分析 V2（`026`）

> 旧版 `026-runtime-phase2-gc-closure.md` 的对账重写。严格对齐当前源码，按"不考虑兼容性"原则给最佳设计。
>
> **范围**：`src/runtime/gc/` 全部 17 文件、`src/runtime/closure/` 全部 6 文件，以及消费者：`src/vm/xvm.c`、`src/vm/xvm_dispatch_closure.inc.c`、`src/frontend/codegen/xcompiler.c`、`src/frontend/codegen/xemit.c`。

## 1. 七条核心结论

- **三种内存域并存**：`XrGC`（isolate fixedgc）/ `XrCoroGC`（per-coroutine Immix）/ `XrSystemHeap`（pool + arena + shared refcount）。V1 判断准确。

- **closure 模型 = flat upvals snapshot + 可选 `XrCell` 间接层**。`OP_CLOSURE` 只复制不解析，没有 Lua-style open upvalue 链。V1 判断准确。

- **`XrBoundMethod` 永远走 isolate fixedgc**，不像 `XrClosure`/`XrCell` 优先走 per-coro。V1 判断准确。`xr_coro_gc_markobject()` 的 defensive traversal 是支撑这套混合模型的关键安全阀。

- **`xbc_stackmap` 是"safepoint live-prefix map"，不是 precise stack map**。V1 判断准确，V2 补上一个 V1 漏报：`xbc_stackmap.h:108` 注释还在 promise "deduplicate bitmaps"，但 `.c:103` 实现明确"no dedup for now"——头/实现注释漂移。

- **`UpvalInfo.slot_type` 不是 closure GC 输入**，是 JIT/AOT/backend metadata。V1 准确。

- **`g_gc_pool_*` 是真实的 process-wide mutable global**（`xcoro_gc.c:50-52`）：互斥锁 + 链表 + 计数。结构体池跨 isolate 共享。

- **`xgc.h` 头注释已经严重过时**（`xgc.h:11-12` 说 "incremental Mark-Sweep with Arena"），实际是 Immix mark-region。V2 P1 修。

## 2. 模块地图（修正版）

实际 17 个 gc 文件 + 6 个 closure 文件：

| 子域 | 文件 |
|---|---|
| **fixed/global GC** | `xgc.h/c`、`xgc_header.h`、`xgc_internal.h` |
| **per-coro GC 引擎** | `xcoro_gc.h/c`、`ximmix.h/c`、`xstackmap.h` |
| **per-type traverse** | `xcoro_gc_traverse.h/c` |
| **system heap** | `xsystem_heap.h/c` |
| **统一分配桥** | `xalloc_unified.h/c` |
| **bytecode root map** | `xbc_stackmap.h/c` |
| **closure 层对象** | `xclosure.h/c`、`xcell.h/c`、`xbound_method.h/c` |

V1 把 `xstackmap.h`（JIT stackmap forward decl）漏列。

## 3. 关键论断核对

### 3.1 三堆域的精确边界

源码确认（`xgc.c:38-49`、`xcoro_gc.h`、`xsystem_heap.h`）：

- **`XrGC`**：仅维护 `fixedgc` 链表 + `g_destroy_funcs[XGC_MAX_TYPES]`（`.rodata` 编译期常量表）。`xr_gc_alloc` 直接 `xr_malloc` 加链。
- **`XrCoroGC`**：Immix mark-region，含 `gray`/`grayagain`/`weak`/`large_objects`/`shared_refs` + write barrier。
- **`XrSystemHeap`**：coroutine 结构体 pool + class/module arena + shared refcount。**对象不参与 per-coro tri-color**。

`xr_alloc` 的真实路径（`xgc.c:149-171`）：

```
xr_alloc(coro, size, type)
  → xr_coro_ensure_gc(coro)           // lazy create coro_gc
  → xr_coro_gc_newobj(gc, type, size) // 主路径
  → fallback: xr_gc_alloc(&isolate->gc, ...)  // 当 ensure_gc 失败
```

V2 补充：fallback 不只是"bootstrap"，**任何时刻 `xr_coro_ensure_gc` 失败都会落到 isolate fixedgc**。这是真实的隐式堆域选择，应当在 API 文档明确。

### 3.2 closure 三件套的所有者

| 类型 | 大小 | 所有者 | 路径 |
|---|---|---|---|
| `XrClosure` | `sizeof + nuv*16` | per-coro（fallback isolate） | `xclosure.c:32-36` |
| `XrCell` | 32B | per-coro（fallback isolate） | `xcell.c:26-30` |
| `XrBoundMethod` | `sizeof(XrBoundMethod)` | **永远 isolate fixedgc** | `xbound_method.c:24` |

V1 判断 100% 准确。源码引用：

```c
// xbound_method.c:24
XrBoundMethod *bm = (XrBoundMethod *)xr_gc_alloc(&isolate->gc, ...);
```

无 coro 路径，无 fallback。

### 3.3 `xbc_stackmap` 是 live-prefix map

源码 `xcompiler.c:563-594` 实际逻辑：

```c
int freereg = xreg_get_freereg(compiler->regalloc);
// ...
for (uint16_t w = 0; w < num_words; w++) {
    bitmap[w] = (nbits >= 64) ? ~(uint64_t)0 : ((uint64_t)1 << nbits) - 1;
}
```

bitmap 把 `[0, freereg)` 全置 1。这是 V1 准确的论断。

V2 补充 V1 漏的 documentation drift：

- `xbc_stackmap.h:108` API 文档：`// Finalize: sort entries, deduplicate bitmaps, produce compact XrBcStackMap.`
- `xbc_stackmap.c:103` 实现注释：`// Phase 1: compute bitmap pool size (simple: no dedup for now)`

头文件 promise 了 dedup，实际没做。要么实现 dedup（同一 freereg 反复出现，dedup 收益高），要么删头注释。

### 3.4 `xgc.h` 头注释严重过时

源码 `xgc.h:10-18`：

```
 * KEY CONCEPT:
 *   - Per-Coroutine GC: each coroutine has isolated heap (XrCoroGC)
 *   - incremental Mark-Sweep with Arena allocation
 *   - Bulk deallocation when coroutine ends (Arena bulk free)
 *   - System heap for global objects (Class, Module) - not GC'd
 */
```

实际：

- 不是 Mark-Sweep，是 **Immix mark-region**（`ximmix.h/c`）。
- 不是 Arena bulk free，是 Immix block/line recycling。
- "global objects ... not GC'd" 是 V1 没强调的关键事实——`Class`/`Module` 走 system heap arena，**完全不参与 GC tracing**。

### 3.5 `xcoro_gc.c` 的 process-wide mutable state

精确源码 `xcoro_gc.c:50-52`：

```c
static pthread_mutex_t g_gc_pool_mu = PTHREAD_MUTEX_INITIALIZER;
static XrCoroGC *g_gc_pool_head = NULL;
static int g_gc_pool_count = 0;
```

L2 pool 上限 `XR_GC_POOL_L2_MAX = 256`。意图是结构体池跨 isolate / 跨 worker 共享，提升 short-lived coroutine 复用率。

代价：

- 跨 isolate 共享 mutable state，违反"isolate 隔离"长期方向。
- mutex 是真锁（不是 atomic），高并发 coroutine 创建/销毁会有竞争。

### 3.6 `g_destroy_funcs` 是 `.rodata` 常量（V1 漏的正面发现）

`xgc.c:38-49`：

```c
const XrGCDestroyFn g_destroy_funcs[XGC_MAX_TYPES] = {
    [XR_TARRAY]         = xr_gc_destroy_array,
    ...
};
```

这是 V1 该列为正面示范的设计：编译期常量、零运行时初始化、跨 isolate 共享天然安全（只读）。扩展类型走 `isolate->ext_destroy_funcs[type]`（`xgc.c:78`）。

### 3.7 `XrBoundMethod` 当前还含 method handler stub

`xbound_method.c:57-61`：

```c
static XrValue bound_method_stub(XrayIsolate *isolate, XrValue receiver,
                                 XrValue *args, int argc) {
    (void)isolate; (void)receiver; (void)args; (void)argc;
    return xr_null();
}
```

`xr_array_get_handler`、`xr_set_get_handler`、`xr_string_get_handler` 在遇到 closure-taking method（foreach/map/filter/reduce/find）时返回 stub。这是 V1 没提的真实 TODO：bound method 还没完整覆盖 closure-taking 方法，用 silent null fallback 兜着。

P1 修：要么实现 bytecode-interpreting adapter，要么显式 throw "bound method on closure-taking builtin not yet supported"，避免 silent null。

### 3.8 `XrCell` 是 `_Static_assert` 守卫的固定 32B

`xcell.h:41`：

```c
_Static_assert(sizeof(XrCell) == 32, "XrCell must be 32 bytes");
```

V1 没强调，但这是 closure 子系统设计可靠性的标志：layout 由编译器静态保证，GC traverse 可以按固定 offset 取 `value` 字段（`xcoro_gc_traverse.c` 直接按 layout 偏移）。

## 4. GC root scan 的真实根集合（V1 准确，补全）

`mark_coro_roots()` 实际扫描：

- VM stack（逐帧，遵循 `bc_stackmap` 否则 conservative）
- VM frames 自身的 `closure`
- struct_area（`mark_struct_string_fields` 仅 `XR_NATIVE_STRING`）
- entry closure
- 主协程的 `vm.shared` 数组
- `coro->task`、`coro->result`/`error`/`pending_closure_result`
- blocked send 的 `send_value`
- C frame 的 `cfunc_result`
- JIT 路径：`jit_ctx->call_closure`、`XrStackMapTable`、innermost frame regs/spills、FP chain JIT frames、`jit_frame_stack`
- external root callbacks

V2 补：V1 漏的两点：

- `xstruct_layout.h` 已经定义 `XR_NATIVE_STRUCT` 和 `XR_NATIVE_ARRAY`，但 `mark_struct_string_fields` 只处理 `XR_NATIVE_STRING`。一旦 native struct 字段类型扩展，root scan 必须同步扩张，**当前没有 assertion 防护**。
- `root_callbacks` 机制源码层有，但 grep 不到 in-tree 注册者（V1 已点出，V2 确认）。

## 5. 状态归属（修正版）

| 模块 | 真实所有者 | 关键边界 |
|---|---|---|
| `XrGC` (fixedgc + destroy table) | isolate | `g_destroy_funcs` 是 .rodata；`ext_destroy_funcs` 在 isolate |
| `XrCoroGC` Immix | per-coroutine | gray/weak/large_objects/shared_refs/owner |
| `XrSystemHeap` | isolate | coroutine struct pool + class/module arena + shared malloc |
| L2 GC pool | **进程**（`g_gc_pool_*`） | 跨 isolate / 跨 worker 共享 |
| `XrClosure` | per-coro（fallback isolate） | flat upvals[]，`OP_CLOSURE` 复制 |
| `XrCell` | per-coro（fallback isolate） | 固定 32B，单 mutable 槽 |
| `XrBoundMethod` | **isolate fixedgc**（无 fallback） | receiver 可能 per-coro，靠 defensive traversal 保活 |
| `bc_stackmap` | per-`XrProto` | live-prefix bitmap，未 dedup |
| `stack_map` (JIT) | per-`XrProto` | XirStackMapTable，opaque |
| root_callbacks | per-coro `XrCoroGC` | 当前无 in-tree 消费者 |

## 6. 高风险点（精确版）

### 6.1 `xgc.h` 头注释过时（P1）

`xgc.h:11-14` 说 "incremental Mark-Sweep with Arena"。实际是 Immix。直接修复：

```c
/*
 * KEY CONCEPT:
 *   - XrGC (this file): isolate-level fixedgc + destroy func table.
 *     Allocates fixed objects (main coroutine, bound methods) and is
 *     used as fallback when XrCoroGC is unavailable.
 *   - XrCoroGC (xcoro_gc.h): per-coroutine Immix mark-region heap;
 *     primary runtime allocator for collected objects.
 *   - XrSystemHeap (xsystem_heap.h): coroutine struct pool, class /
 *     module arenas, shared object refcount. Not tracing-managed.
 *
 * ALLOCATION PATH:
 *   xr_alloc(coro, size, type)
 *     -> xr_coro_ensure_gc(coro) -> xr_coro_gc_newobj
 *     -> fallback: xr_gc_alloc(&isolate->gc, ...)
 */
```

### 6.2 `xbc_stackmap` 文档/实现漂移（P1）

头注释 promise dedup，实现明说 no dedup。两条路：

- **路径 A**：实现 dedup。bitmap pool 复用 + entries 共享 offset。同一 freereg 模板反复出现，dedup 收益是 O(safepoints) → O(unique-freereg)。
- **路径 B**：删头注释 dedup 那一句，明确"current implementation is non-deduped".

V2 推荐 A：dedup 实现简单（hash bitmap → offset），且 long-running 函数中 safepoint 数量大。

### 6.3 `xbc_stackmap` 的 "precise" 命名过强（P2）

V1 已点出。命名/文档建议改：

- 类型名 `XrBcStackMap` → 保留（无歧义）。
- API 注释从 "precise GC" → "live-prefix safepoint map（保守扫描的下一步精化）"。

### 6.4 `g_gc_pool_*` process-wide global（P0）

源码 `xcoro_gc.c:50-52`。最佳设计：

- L2 pool 移到 `XrSystemHeap`，per-isolate 持有。
- coroutine 创建/销毁 99% 在同 isolate 内，pool 局部化收益不变。
- 跨 isolate 共享需求几乎不存在（不同 isolate 各有 type pool/global object，coroutine 实际不可跨）。

直接迁移代价：把 `gc_pool_l2_pop`/`push` 改为 `(XrSystemHeap *sh)` 参数，调用点已经能拿到 isolate。

### 6.5 `XrBoundMethod` 永远 isolate fixedgc（P2）

V1 已点出。两条路：

- **保持现状**：明确文档化"bound method 是 isolate-lifetime"，依赖 defensive traversal。
- **统一到 per-coro**：`xr_bound_method_new` 接受 coro，走 `xr_coro_gc_newobj`。代价：bound method 跨 coro 传递时需要深拷贝或 shared 处理。

V2 推荐保持现状但显式文档（bound method 通常很少，isolate fixedgc 开销可接受）。

### 6.6 `bound_method_stub` silent null（P1）

`xbound_method.c:57-61`。closure-taking builtin（foreach/map/filter）在被绑定为 method value 时静默返回 null。

修：要么实现，要么 throw `"BoundMethodNotSupported: builtin '<name>' takes a closure and cannot be bound as a value (yet)"`。silent null 是最差选项。

### 6.7 `mark_struct_string_fields` 只处理 STRING（P2）

`xstruct_layout.h` 定义了 `XR_NATIVE_STRUCT`/`XR_NATIVE_ARRAY`，但 GC 只扫 `XR_NATIVE_STRING`。当前 `xr_type_kind_to_native` 只生成 bool/int/float/string，所以工作。

防御：在 `mark_struct_string_fields` 加 `XR_DCHECK(field.native_type != XR_NATIVE_STRUCT && field.native_type != XR_NATIVE_ARRAY, "GC scan: nested native struct field not yet supported")`。这样未来扩展时会立即 fail-fast。

### 6.8 `xcoro_gc_traverse.c` 跨层依赖（V1 已提）

`xcoro_gc_traverse.c` 直接知道 Array/Map/Set/Json/Closure/Cell/BoundMethod/Instance/Module/Task 的内部布局。

最佳设计：把 traverse 函数表化，每个对象 owner 模块自己提供 traverse 函数（与 `g_destroy_funcs` 同样模式）：

```c
typedef void (*XrGCTraverseFn)(XrCoroGC *gc, XrGCHeader *obj);
const XrGCTraverseFn g_traverse_funcs[XGC_MAX_TYPES] = {
    [XR_TARRAY]   = xr_gc_traverse_array,    // 在 xarray.c
    [XR_TMAP]     = xr_gc_traverse_map,      // 在 xmap.c
    [XR_TCLOSURE] = xr_gc_traverse_closure,  // 在 xclosure.c
    ...
};
```

`gc/` 目录只保留 dispatch loop。每个对象的 traverse 实现回到 owner 模块。

### 6.9 `root_callbacks` 无 in-tree 消费者（P3）

V1 已点出。机制完整但没用过。两条路：

- **删除**：API + 调用 + 字段全清。等真有需求再加。
- **保留 + 在文档明确**："reserved for future extension"。

V2 推荐删除，YAGNI。

## 7. 正向资产（V1 已识别，V2 强化）

- **三堆域分离**清晰且文档化。
- **`xalloc_unified` 依赖减压**：把 worker/coroutine 重头文件局限在 `.c`。
- **`xr_coro_gc_reset()` + coroutine struct pool** 配合得当。
- **`xr_coro_gc_markobject()` defensive traversal** 是混合所有权模型的关键安全阀。
- **`g_destroy_funcs` 编译期常量表**（V1 漏的正面发现）。
- **`XrCell` `_Static_assert` 32B 守卫**（V1 漏）。
- **flat upvals + cell indirection 设计**优于 Lua-style open upvalue 链：无 close 操作、无链表维护、零分配 const capture。

## 8. 最佳设计建议（无兼容性）

按"无兼容性 + 按现有架构最佳化"的 10 条：

1. **`xgc.h` 头注释重写**（§6.1）
2. **`xbc_stackmap` dedup 实现 + 文档对齐**（§6.2）
3. **`g_gc_pool_*` 迁移到 `XrSystemHeap`**（§6.4，P0）
4. **`bound_method_stub` 改为显式 throw**（§6.6）
5. **`mark_struct_string_fields` 加 fail-fast assertion**（§6.7）
6. **GC traverse 函数表化**（§6.8），把 per-type 实现回到 owner 模块
7. **删除 `root_callbacks`** 直到有真实消费者（§6.9）
8. **`UpvalInfo` 字段语义注释**：`source`=VM 输入、`storage_mode`=compile/serialize、`slot_type`=backend、`type_info`=optimization
9. **`XrBoundMethod` 文档明确**："isolate-lifetime, receiver kept alive via defensive traversal"
10. **`xstackmap.h` 与 `xbc_stackmap.h` 命名澄清**：JIT 用 `XrStackMap`，bytecode 用 `XrBcStackMap`，避免读者混淆

## 9. 给下一轮 `object/` 的输入

V1 给的方向都对，V2 补充：

- Array/Map/Set/Json 各自的 `data_on_gc_heap` / `nodes_on_gc` / `entries_on_gc` 标志位的真实语义和 traverse 接线。
- shared object 的写屏障（`xr_coro_gc_shared_ref_add`）覆盖率：哪些 mutator 已经接、哪些没接。
- `xmethod_table` extern declaration 在 object methods header 里的反向 include 链（与 025 V2 §3.12 协同）。
- Json shape registry（`isolate->shape_*`）与 GC traverse 的边界。
- `xclass.c` 的 class 创建是否走 system heap arena vs coro_gc。

## 10. 与 V1 的主要差异

| 点 | V1 | V2 |
|---|---|---|
| 三堆域 | 准确 | 保留 |
| closure 三件套 owner | 准确 | 保留 + 加 `XrCell` 32B static_assert |
| `xbc_stackmap` dedup 漂移 | 漏 | 头/实现注释不一致，列为 P1 |
| `g_destroy_funcs` 编译期常量 | 漏 | 列为正面示范 |
| `bound_method_stub` silent null | 漏 | 列为 P1（应 throw） |
| `mark_struct_string_fields` 未来漂移 | 提到 | 加 fail-fast assertion 建议 |
| root_callbacks 处理 | 描述能力但无消费者 | YAGNI 删除 |
| GC traverse 跨层 | 提到 | 给函数表化具体方案 |
| `g_gc_pool_*` | 风险 | P0 + 给迁移路径 |
| 改进建议 | 6 条方向 | 10 条可执行 |

## 11. 本轮状态

- **已完成**：gc/ 17 文件 + closure/ 6 文件全核对、V1 论断逐项判断、最佳设计 10 条
- **未完成**：`ximmix.c` 内部 block/line 策略对对象布局的影响（留给后续）；write barrier 在各 mutator 中的覆盖率（留给 027）
- **下一步**：`027-runtime-phase3-object-v2.md`
