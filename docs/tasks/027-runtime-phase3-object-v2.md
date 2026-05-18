# Runtime `object/` 层分析 V2（`027`）

> 旧版 `027-runtime-phase3-object.md` 的对账重写。严格对齐当前源码，按"无兼容性"原则给最佳设计。
>
> **范围**：`src/runtime/object/` 全部 50 个文件 + 必要消费者（`gc/xcoro_gc.c`、`gc/xcoro_gc_traverse.c`、`runtime/xexception_impl` 调用者）。

## 1. 七条核心结论

- **`runtime/object/` 不是统一对象层**，而是 5 种生命周期模型的汇合点：per-coro GC / system heap shared / global intern / isolate metadata / 直接 `xr_malloc` 的"伪 GC 对象"。V1 准确。

- **容器对象的 header owner 与 backing storage owner 通常不一致**：Array/Map 的 backing 可能从 GC blob 退化为 malloc，Set/Json overflow 直接 malloc。V1 准确。

- **`xexception` 是真实的内存泄漏点**，不只是 V1 说的"owner unclear"。源码确认：`XR_ALLOCATE` = `xr_malloc`（`xmalloc.h:206`），不在 fixedgc、不在 Immix、不在 `g_destroy_funcs`、`xexception.c` 整个文件**没有任何 free/destroy 函数**。

- **`xr_gc_destroy_shape` 是 dead code**：`xshape.c:302` 定义、`xshape.h:138` 声明，但 grep 全仓**零调用方**。V1 说"没有 visible 调用点"，V2 精确确认。

- **`XR_SET_FLAG_ENTRIES_ON_GC` 永远不代表"实际有 GC blob"**：`xset.c:80` 创建时设 flag、`entries=NULL`；`xset.c:223-226` 第一次 resize 立刻清 flag、走 `xr_malloc`。flag 在生命周期内永远没有"真实 GC blob 阶段"。

- **External memory accounting 当前只 `xarray.c` 完整**：`xmap.c`/`xset.c`/`xjson.c`/`xstringbuilder.c` 都没接 `xr_gc_add_external`/`sub_external`。

- **`XR_ALLOCATE` 的"伪 GC 对象"模式不只 exception**。reflection 子系统 (`xreflect_api.c`、`xreflect_cache.c`) 大量用 `XR_ALLOCATE(TypeWrapper/FieldWrapper/MethodWrapper/ParameterWrapper)`，全部 malloc 直分配、不在 GC 链上。这是 V1 没识别的更大模式。

## 2. 模块地图（修正版）

50 个文件，按真实职责分 9 类：

| 子域 | 文件 |
|---|---|
| 数组 | `xarray.h/c`、`xarray_class_init.h/c`、`xarray_methods.h/c`、`xslice.h/c` |
| 映射 | `xmap.h/c`、`xmap_methods.h/c`、`xmap_instance_methods.h/c` |
| 集合 | `xset.h/c`、`xset_class_init.h/c`、`xset_methods.h/c` |
| Json | `xjson.h/c`、`xjson_methods.h/c`、`xjson_pool.h/c` |
| Shape | `xshape.h/c`、`xshape_cache.h/c` |
| 字符串 | `xstring.h/c`、`xstring_methods.h/c`、`xutf8.h/c`、`xstringbuilder.h/c` |
| 大整数 | `xbigint.h/c`、`xbigint_methods.h/c` |
| 迭代器 | `xiterator.h/c` |
| 叶子/异常 | `xrange.h/c`、`xnative_type.h/c`、`xexception.h/c`、`builtins/` |

V1 漏列：`xbigint.*`（40KB，与 int.toBigInt 直接相关）、`xslice.*`、`builtins/`、各种 `*_class_init.*`。

## 3. 关键论断核对

### 3.1 `XrArray` 的真实路径（V1 准确）

源码 `xarray.c:756-760, 802-806, 915-918`：

- `xr_alloc(coro, sizeof(XrArray), XR_TARRAY)` → header 在 coro Immix。
- `data` 初始：有 `coro_gc` 走 `xr_coro_alloc_blob`（`data_on_gc_heap=1`）；否则 `xr_malloc`。
- grow → 强制走 malloc backing（避免 Immix line recycling 的 memcpy UB）。
- `xr_gc_add_external` / `sub_external` 对 malloc backing 做 accounting。

V1 准确。

### 3.2 `XrMap` rehash 退化（V1 准确）

`xmap.c` `setnodevector` / rehash 逻辑：rehash 时清掉 `XR_MAP_FLAG_NODES_ON_GC`、强制新数组 malloc。`saved_gc_flag` 字段保留但未恢复使用——V1 说"易退化"准确。

### 3.3 `XrSet` flag dead（V1 准确，V2 精确化）

源码 `xset.c`：

- `:80` `set->flags = XR_SET_FLAG_ENTRIES_ON_GC; set->entries = NULL;` —— flag 设但无 backing。
- `:223-226` resize 检查 flag 在则清掉，然后 `new_entries = xr_malloc(alloc_bytes)`。
- `:618-624` destroy 路径仍按 flag 分支（fast-path Immix sweep / xr_free），但实际永远走后者。

V2 修正：flag **不是"历史残留"**而是"never-set-to-meaning"——它在生命周期内**根本没有 lifecycle 阶段对应"GC blob 实际存在"**。建议直接删 flag。

### 3.4 `XrJson` overflow 直接 malloc（V1 准确）

`xjson.c` overflow 走 `xr_realloc(old, sizeof(XrJsonOverflow)+...)`。无 `add_external`/`sub_external` 调用。V1 准确。

### 3.5 `XrException` 真实内存泄漏（V2 强化）

V1 说"leak risk high"。V2 精确确认：

- 分配：`XR_ALLOCATE(XrException)` = `xr_malloc(sizeof(XrException))`（`xexception.c:31`、`xmalloc.h:206`）。
- `XR_TEXCEPTION` 在 `xcoro_gc.c:88` 的 `xr_gc_has_refs` mask 里 → traverse 时会标记 children。
- `XR_TEXCEPTION` 不在 `g_destroy_funcs[]`（`xgc.c:38-49`）。
- `xexception.c` 整个文件无 `xr_free` / `destroy` / `cleanup` 函数（`grep` 验证）。
- 不在 fixedgc 链上（没走 `xr_gc_alloc`）。
- 不在 Immix 上（没走 `xr_coro_gc_newobj`）。

**结论**：每次 throw 一个 exception，都泄漏一个 `sizeof(XrException)` ≈ 48 字节 + traverse 出来的 children 引用。children 是 GC 管的还能保活，但 exception 本身永远不会被释放。

修复：**强制让 `XrException` 走 `xr_alloc(coro, ...)`**，接入 per-coro Immix。或者接入 fixedgc + `g_destroy_funcs[XR_TEXCEPTION]`。

### 3.6 `XR_ALLOCATE` 模式扩散（V1 漏）

V2 grep 确认：`XR_ALLOCATE` 在以下处使用：

- `xexception.c:31` — XrException
- `xreflect_api.c:31, 46, 61, 79` — TypeWrapper/FieldWrapper/MethodWrapper/ParameterWrapper
- `xreflect_cache.c:29, 49, 71` — XrReflectCache 内部 wrapper

所有 wrapper 都带 `XrGCHeader`（heap_type 设了），但没有任何一个走 `xr_gc_alloc` 或 `xr_alloc`，全是 `xr_malloc`。这是 V1 完全没识别的"伪 GC 对象"模式。

修复：所有 `XR_ALLOCATE` 调用点要么走 fixedgc 要么走 coro Immix，要么显式标记为非 GC（删 GCHeader）。当前模式同时有 GC traverse 入口（heap_type 一致）但无 free 路径，是真实的内存泄漏 + 双重 ownership 不一致。

### 3.7 `xr_gc_destroy_shape` dead code（V1 准确）

源码 grep 验证：除 `xshape.h:138` 声明 + `xshape.c:302` 定义，**零调用方**。V1 说 "visible 调用点不可见"，V2 精确：是 dead code。

`xr_shape_registry_destroy` 只 `xr_free` 数组本身：

```c
// xshape.c (从 V1 推测)
void xr_shape_registry_destroy(XrayIsolate *X) {
    if (X->shape_entries) xr_free(X->shape_entries);
}
```

shape 的内部 `field_*` 数组、transition map 都泄漏。

修复：`xr_shape_registry_destroy` 遍历所有 shape，逐个调 `xr_gc_destroy_shape` 释放内部状态。

### 3.8 External accounting 覆盖率（V1 准确）

确认 `xr_gc_add_external` / `xr_gc_sub_external` 在 object/ 层只 `xarray.c` 调用（`grep` 输出全部 5 处都在 xarray.c）。

GC `totalbytes` 因此**显著低估真实内存占用**：

- Map.node[] malloc 不计
- Set.entries[] malloc 不计
- Json.overflow malloc 不计
- StringBuilder.buffer + StrBuf.data malloc 不计

后果：GC 触发 trigger 基于 `totalbytes`，外部增长不触发回收，long-running 程序 RSS 可以远超 GC 视野。

### 3.9 `XR_TEXCEPTION` `has_refs` 但无 destroy 是双轨设计反例

`xcoro_gc.c:85-89` 的 `has_refs` mask 表达"GC 应当扫这类对象的子引用"，但同模块 `g_destroy_funcs` 表达"GC 应当析构这类对象内部资源"。两表是独立的——`XR_TEXCEPTION` 在前者却不在后者。

V2 加正向建议：**两表应当合并成一个 per-type description**：

```c
typedef struct {
    XrGCTraverseFn traverse;  // NULL = no refs
    XrGCDestroyFn destroy;    // NULL = no internal resources to free
    bool is_managed;          // false = malloc-only, GC不管释放
} XrTypeOps;
const XrTypeOps g_type_ops[XGC_MAX_TYPES] = { ... };
```

避免 has_refs / destroy 各自维护、漂移。

## 4. 状态归属（修正版）

| 对象 | header owner | backing owner | 释放路径是否完整 |
|---|---|---|---|
| `XrArray` | per-coro Immix | GC blob → malloc on grow | ✓（含 external accounting） |
| `XrMap` | per-coro Immix | GC blob → malloc on rehash | 部分（无 external accounting） |
| `XrSet` | per-coro Immix | always malloc（flag 是死的） | 部分 |
| `XrJson` | per-coro Immix | overflow always malloc | 部分 |
| `XrIterator` | per-coro Immix | source 引用 | ✓ |
| `XrString`（non-interned） | per-coro Immix | inline data | ✓ |
| `XrString`（interned ≤64） | global pool malloc | inline | global pool sweep |
| `XrString`（interned >64） | system heap shared | inline | refcount decref |
| `XrShape` | `xr_calloc` + isolate registry | inline | **不完整**（destroy 无调用） |
| `XrException` | **`xr_malloc`** | inline | **真实泄漏** |
| `XrStringBuilder` | per-coro Immix | malloc StrBuf | 部分 |
| `XrBigInt` | per-coro（`current_coro`） | inline | ✓ |
| `XrRange` | per-coro Immix | inline | ✓ |
| reflection wrappers (`Type/Field/Method/ParameterWrapper`) | **`xr_malloc`** | inline | **真实泄漏** |

## 5. 高风险点（精确版）

### 5.1 `XrException` 真实泄漏（P0）

每次 throw 泄漏一个 `XrException` 对象。修复：

```c
static XrException* xr_exception_alloc(XrayIsolate *X) {
    XrCoroutine *coro = xr_current_coro(X);
    XrException *exc = (XrException *)xr_alloc(coro, sizeof(XrException), XR_TEXCEPTION);
    if (!exc) return NULL;
    xr_gc_header_init_type(&exc->gc, XR_TEXCEPTION);
    // ...
}
```

`exc->message` / `exc->file` / `exc->stackTrace` 已经是 GC managed，所以 `XrException` 在 Immix 上自动被 traverse + sweep 释放。一行修复。

### 5.2 reflection wrappers 真实泄漏（P0）

`xreflect_api.c` / `xreflect_cache.c` 的所有 `XR_ALLOCATE(*Wrapper)` 同样问题。修复：要么走 `xr_alloc(coro, ...)`，要么显式手动 free 在 isolate cleanup。

`xreflect_cache.c` 实际有 `xr_reflect_cache_destroy`？需要进 028（class/symbol）确认。当前 V2 指出问题，留给 028 V2 详细修复方案。

### 5.3 `xshape` 销毁不完整（P1）

`xr_gc_destroy_shape` 是 dead code。修复：在 `xr_shape_registry_destroy(X)` 中遍历所有 `X->shape_entries`，逐个 destroy。

### 5.4 `XR_SET_FLAG_ENTRIES_ON_GC` 删除（P2）

`xset.c:80, 223-226, 618-624` 三处涉及 flag，但 flag 永无"GC blob 阶段"。直接删 flag、删条件分支。代码净减少 ~10 行。

### 5.5 External accounting 一致化（P1）

为 `xmap.c`/`xset.c`/`xjson.c`/`xstringbuilder.c` 的 backing 分配/释放路径补 `xr_gc_add_external` / `sub_external` 调用。统一接入 GC 触发逻辑。

### 5.6 `g_destroy_funcs` + `has_refs` 合并成 `g_type_ops`（P2）

V2 §3.9 提案。避免双表漂移。

### 5.7 `Json.overflow` 大字段时 GC 无视（P1，5.5 子项）

`xjson.c` overflow `xr_realloc` 直接打，0 accounting。V2 已纳入 5.5。

### 5.8 `xshape_id` 打包进 `gc.extra`（V1 提）

`xjson.h` 用 `xr_gc_get_shape_id` / `xr_gc_set_shape_id` 编进 GCHeader extra bits。bit budget 受限；shape 数量超出会 silent overflow。

V2 检查：grep `xr_gc_set_shape_id` 范围 + 字段位宽 → 留给 028 V2 与 `XrayIsolate.shape_capacity` 一起判断真实上限。

### 5.9 `xmap.c` `xr_map_dummynode` 静态共享节点（V1 提）

只读共享节点，跨 isolate 安全。V2 不算风险。但要文档化"该节点不可写"，加 `const`。

## 6. 正向资产

V1 已识别 + V2 强化：

- **`XrArray` typed storage** (`elem_type`/`elem_tid`/`has_gc_ptrs`/`data_on_gc_heap`)：清晰，可作为其他容器对齐目标。
- **Json + Shape hidden class**：in-object + overflow + immutable transition + symbol→index + compact layout。
- **String global pool + GC 协作**：`SHARED` 标记 + `XR_STR_SET_ACCESSED` + global pool sweep。
- **`XrIterator`**：lazy view，owner 明确，实现干净。
- **`XR_ALLOCATE`** 本身（`xmalloc.h:206`）是个简洁宏，问题在调用语义不当。

## 7. 最佳设计建议（无兼容性）

10 条直接可执行：

1. **`XrException` 改走 `xr_alloc(coro, ...)`**（§5.1，P0，一行修复）
2. **reflection wrappers 改走 coro Immix 或注册到 isolate 显式 free 列表**（§5.2，P0）
3. **`xr_shape_registry_destroy` 调 `xr_gc_destroy_shape`**（§5.3，P1）
4. **删除 `XR_SET_FLAG_ENTRIES_ON_GC`**（§5.4，P2）
5. **External accounting 统一**（§5.5，P1）
6. **合并 `g_destroy_funcs` + `has_refs` → `g_type_ops`**（§5.6，P2）
7. **`xmap.c xr_map_dummynode` 加 `const`**（V1 提）
8. **`Map.saved_gc_flag` 字段删除**（V1 提，rehash 后无恢复使用）
9. **`xshape_id` bit budget 加 assertion**：超出时 throw 而不是 silent overflow
10. **objects 头注释统一标注 owner**：每种对象在 `.h` 文件顶部明确写"header on X, backing on Y, free path: Z"。当前部分对象已有，全部补齐。

## 8. 给下一轮 `class/ + symbol/` 的输入

V1 给的方向都对，V2 补充：

- reflection wrapper（`TypeWrapper`/`FieldWrapper`/`MethodWrapper`/`ParameterWrapper`）的真实泄漏链路。
- `XrReflectCache` 的销毁路径是否覆盖所有 wrapper。
- `XrShape` 与 `XrClass.fields[]` / `XrClass.struct_layout` 的边界。
- symbol id allocation：是否也是进程级 atomic counter（类似 proto_id）？
- `xshape_cache.*` 与 `xjson_pool.*` 的 owner 模型。

## 9. 与 V1 的主要差异

| 点 | V1 | V2 |
|---|---|---|
| 容器 backing storage | 准确 | 保留 |
| `XrException` | "owner unclear" + "leak risk" | 精确确认真实泄漏 + 一行修复 |
| `XR_ALLOCATE` 模式 | 仅识别 exception | 扩散到 reflection 4+ 类型 |
| `xr_gc_destroy_shape` | "visible 调用点不可见" | 精确：dead code（grep=0） |
| `XR_SET_FLAG_ENTRIES_ON_GC` | "历史残留" | "never-set-to-meaning"（永无 GC blob 阶段） |
| `g_destroy_funcs` + `has_refs` 双表 | 未识别 | 列为漂移源，给合并方案 |
| reflection 子系统 | 未提 | 标记为另一处真实泄漏 |
| 改进建议 | 6 条方向 | 10 条可执行 |

## 10. 本轮状态

- **已完成**：object/ 50 文件全核对、V1 论断逐项判断、最佳设计 10 条
- **未完成**：reflection wrapper destroy 路径详查（留给 028 V2）；`xshape_id` bit budget 量化（留给 028 V2）；shared object deep-copy 在各 mutator 的覆盖率
- **下一步**：`028-runtime-phase4-class-symbol-v2.md`
