# Runtime `object/` 层分析（`027`）

> 本轮聚焦 `src/runtime/object/`。
>
> 这一层表面上是“对象实现”，实际已经混合了：
>
> - 容器对象（Array / Map / Set / Json / Iterator）
> - 字符串与全局 intern 服务
> - hidden class / shape 元数据服务
> - 异常传递对象
> - StringBuilder / StrBuf 展示基础设施
>
> 因此本轮重点不是逐个 API 枚举，而是识别：
>
> - 每种对象真正落在哪个堆域
> - backing storage 何时从 GC blob 退化成 malloc
> - write barrier / external memory accounting 是否一致
> - 哪些文件已经不是纯 object 层，而是 runtime 横切服务

## 1. 本轮结论

- **`runtime/object/` 不是统一对象层，而是多种生命周期模型的汇合点。**
  - per-coro GC object
  - system heap shared object
  - global intern pool object
  - isolate-owned metadata object
  - 直接 `xr_malloc` 的“伪 GC 对象”

- **容器对象的 header owner 和 backing storage owner 往往不是同一个。**
  - `XrArray`：header 可在 coro heap，data 初始可能在 GC blob，增长后可能转成 malloc
  - `XrMap`：header 在 coro heap，node[] 可能先是 GC blob，rehash 后常驻 malloc
  - `XrSet`：header 在 coro heap，但 entries[] 当前实现基本直接走 malloc
  - `XrJson`：header 在 coro heap，overflow 永远是 malloc/realloc

- **external memory accounting 明显不一致。**
  - `xarray.c` 在 malloc/realloc backing 上会 `xr_gc_add_external` / `xr_gc_sub_external`
  - `xmap.c` / `xset.c` / `xjson.c` / `xstringbuilder.c` 对外部分配基本没有对应 accounting

- **`xstring` 已经是 runtime 基础服务，不是普通 object。**
  - 短字符串：global pool，shared 标记，不走 per-coro refcount
  - 长字符串：system heap shared + refcount
  - 非 interned string：current coro heap

- **`xshape` 也是元数据服务，不是普通 heap object。**
  - per-isolate registry
  - hidden class transition cache
  - compact layout / GC field info
  - 但当前 ownership/销毁路径不完整

- **`xexception` 的 owner 模型有明显缺口。**
  - 注释写“GC-managed exception”
  - 实际分配走 `XR_ALLOCATE` → `xr_malloc`
  - 没有 visible destroy path 挂到 `g_destroy_funcs[]`

## 2. 模块地图

| 子域 | 关键文件 | 真实职责 |
|---|---|---|
| 数组 | `xarray.*` | typed/untyped 动态数组、slice、join、higher-order helpers |
| 映射 | `xmap.*` | chained scatter hash、Brent 变体、weak map、entries 迭代 |
| 集合 | `xset.*` | open addressing set、weak set、集合运算 |
| Json | `xjson.*` `xshape.*` `xshape_cache.*` | hidden-class JSON、shape transition、overflow field storage |
| 字符串 | `xstring.*` `xutf8.*` | immutable string、global intern、unicode helpers |
| 迭代器 | `xiterator.*` | lazy iteration over Map/Set/Json |
| 展示缓存 | `xstringbuilder.*` `runtime/xstrbuf.*` | mutable builder + malloc buffer |
| 错误/异常 | `xexception.*` | runtime throw/catch payload |
| 叶子对象 | `xrange.*` `xnative_type.*` | small leaf object / native type bridge |

## 3. 各对象的真实 owner

### 3.1 `XrArray`

`xarray.c` 当前是几种 owner 的组合：

- object header：`xr_alloc(coro, sizeof(XrArray), XR_TARRAY)`
- data 初始分配：
  - 若有 `coro_gc` → `xr_coro_alloc_blob(gc, data_bytes)`，`data_on_gc_heap = 1`
  - 否则 fallback `xr_malloc`
- shared array：`xr_array_init_inplace()`，header 由外部 system heap 预分配，data 直接 `xr_malloc`
- slice：header 独立，data 指向 source 偏移，owner 仍是 source

最关键的是 grow/ensure_capacity：

- 如果旧 data 在 GC blob 上，扩容会**强制转成 malloc backing**
- 原因是避免 Immix blob overlap 造成 `memcpy` 未定义行为

因此 `XrArray` 的真实模型不是“data 一直在 GC heap”，而是：

- **初始可在 GC blob**
- **一旦扩容，常转入 external malloc backing**

### 3.2 `XrMap`

`XrMap` header 在 coro heap，空表共享静态 `xr_map_dummynode`。

node[] backing 模型：

- 初始带 `XR_MAP_FLAG_NODES_ON_GC`
- `setnodevector()` 在有 `coro_gc` 时可以用 `xr_coro_alloc_blob()`
- fallback 才变成 malloc

但 rehash 逻辑显式做了：

- 先清掉 `XR_MAP_FLAG_NODES_ON_GC`
- 强制新 node[] 走 malloc
- old GC blob 交给 Immix sweep

所以 `Map` 的真实路径是：

- **早期可能是 GC blob-backed**
- **第一次 rehash 后通常永久退化为 malloc-backed**

### 3.3 `XrSet`

`XrSet` 看起来也定义了：

- `XR_SET_FLAG_ENTRIES_ON_GC`

但当前实现里：

- `xr_set_new()` 先把 flag 置上
- `xr_set_resize()` 一开始就把 flag 清掉
- 然后直接 `xr_malloc(alloc_bytes)`

而源码里没有任何 `xr_coro_alloc_blob()` 给 `Set` entries[] 分配的路径。

结论：**`XR_SET_FLAG_ENTRIES_ON_GC` 当前更像历史残留 / 名义支持，实际 entries[] 基本是 malloc-backed。**

### 3.4 `XrJson`

`XrJson` header 是按 shape 的 in-object capacity 一次性在 coro heap 上分配：

- `xr_json_new_with_shape()` / `xr_json_new()` → `xr_coro_gc_newobj()`

但 overflow 部分永远走：

- `xr_realloc(old, sizeof(XrJsonOverflow)+...)`

shared Json 则通过：

- `xr_json_init_inplace()`
- 由外部 system heap 预分配 header

所以 Json 是：

- **header in-object 存储在主对象上**
- **overflow 字段单独 malloc/realloc**
- **shape 不在对象内，而在 isolate registry**

### 3.5 `XrIterator`

`xiterator.c` 比较干净：

- header 在 coro heap
- source 保存 Map / Set / Json 指针
- Json iterator 还额外持有 isolate/context

它本身更像“lazy view object”，owner 明确。

### 3.6 `XrString`

字符串是 object 层里最复杂的一类：

- `xr_string_new()`：non-interned，`string_alloc()` → current coro heap
- `xr_string_intern()`：
  - `length <= 64`：global pool，`xr_malloc`，标记 `GLOBAL + SHARED`
  - `length > 64`：system heap shared，refcount = 1
  - sys_heap 不可用时 fallback 到 current coro heap

也就是说**同一个 `XR_TSTRING` 有至少三种 owner**：

- current coroutine
- system heap shared
- global string pool

### 3.7 `XrShape`

`XrShape` 不是普通 heap object：

- `xr_shape_new()` / `xr_shape_new_compact()` 都走 `xr_calloc`
- shape 注册到 `X->shape_entries[]`
- 生命周期设计上等于 isolate
- `XrJson` 通过 `gc.extra` 里的 `shape_id` 反查 shape

它实际是：

- hidden class descriptor
- compact field layout descriptor
- shape transition cache node
- per-isolate metadata registry entry

### 3.8 `XrException`

`xexception.c` 的分配是：

- `XR_ALLOCATE(XrException)`
- 也就是 `xr_malloc(sizeof(XrException))`

但对象本身又带：

- `XrGCHeader`
- `XR_TEXCEPTION`

而且 GC traverse 也知道怎么扫描它的 children。

这说明它是一个**看起来像 GC 对象，但 owner 却不在当前 GC/system heap 体系内**的对象。

### 3.9 `XrStringBuilder`

`XrStringBuilder` 自身：

- 在 coro heap 上分配

但内部 `XrStrBuf`：

- `xr_strbuf_new()` → `xr_malloc(sizeof(XrStrBuf))`
- `sb->data` 也是 `xr_malloc`

因此这是典型的：

- **GC object header + malloc internal buffer**

## 4. 边界判断

### 4.1 相对纯容器的对象

这几块相对接近“普通对象实现”：

- `xiterator.*`
- `xrange.*`
- `xnative_type.*`

它们状态简单、children 少、和外部控制面交互有限。

### 4.2 已经不是纯 object 的文件

#### `xstring.*`

它已经是：

- object type
- intern pool
- concurrency-safe global service
- shared object refcount participant
- GC special-case participant

GC 里甚至专门对 global pool string 做了：

- `XR_STR_SET_ACCESSED`
- 跳过 shared_refs/refcount 处理

这说明字符串已经是 runtime service。

#### `xshape.*`

它实际站在：

- object
- symbol
- isolate registry
- compact layout / typed storage
- json access fast path

的交叉点上。

#### `xexception.*`

它不仅是对象，还承担：

- error transport
- throw/catch payload
- stack trace building
- stderr rendering

而且它还暴露 owner 模型不一致问题。

#### `xstringbuilder.*`

它表面是 object，实际上深度依赖：

- `runtime/xstrbuf.*`
- string interning
- formatting pipeline

更像展示基础设施的一部分。

#### `xjson.*`

它本体是 object，但高度依赖：

- `xshape.*`
- symbol table
- hidden class transition
- GC header bit packing

它已经不只是“Map-like data object”。

## 5. GC / barrier / external memory 覆盖

### 5.1 barrier 基本采用 back barrier

当前主要看到：

- `xarray.c`：`XR_GC_BARRIER_BACK_SAFE(xr_current_coro_gc(), arr)`
- `xmap.c`：更新 value 后 back barrier
- `xset.c`：add 后 back barrier
- `xjson.c`：set/transition 后 back barrier

也就是说容器修改主要走：

- **container-level back barrier**
- 而不是对每个 child 做 forward barrier

这和当前 GC 设计是一致的，但也意味着：

- 容器实现必须保证所有 mutation 点都覆盖到 back barrier

### 5.2 external memory accounting 目前只有 Array 相对完整

`xarray.c` 在 malloc-backed data 上：

- grow/ensure_capacity → `xr_gc_add_external`
- destructor → `xr_gc_sub_external`

但 `xmap.c` / `xset.c` / `xjson.c` / `xstringbuilder.c` 当前看不到对应 accounting。

这意味着 GC 对以下对象的外部内存压力感知不足：

- `Map.node[]`
- `Set.entries[]`
- `Json.overflow`
- `StringBuilder.buffer`
- `XrStrBuf.data`

### 5.3 object traverse 已知这些 backing 模型

`xcoro_gc_traverse.c` 当前明确知道：

- Array 的 `source` 和 `data_on_gc_heap`
- Map 的 `XR_MAP_FLAG_NODES_ON_GC`
- Set 的 `XR_SET_FLAG_ENTRIES_ON_GC`
- Json 的 `shape + overflow`
- Iterator 的 source object

这说明对象层的 backing 策略已经直接渗透到 GC 层。

## 6. 已确认高风险点

### 6.1 `xexception` 的 owner 明显不清楚

已确认事实：

- `xr_exception_alloc()` → `XR_ALLOCATE(XrException)` → `xr_malloc`
- `XR_TEXCEPTION` 不在 `g_destroy_funcs[]`
- 当前看不到 dedicated free path

因此至少可以确定：**它没有接入当前主流 owner 模型。**

更保守的说法是：

- ownership unclear
- free path not visible
- leak risk high

### 6.2 `xshape` 的销毁路径也不完整

已确认事实：

- shape 创建走 `xr_calloc`
- `xr_shape_registry_destroy()` 只 free `shape_entries[]` 数组
- `xr_gc_destroy_shape()` 存在，但当前没有 visible 调用点

因此当前 `XrShape` 也存在明显的 isolate-shutdown ownership gap。

### 6.3 `Set` 的 GC flag 与实现已经脱节

`XR_SET_FLAG_ENTRIES_ON_GC` 当前：

- header 里保留
- GC traverse/destroy 仍识别
- 但 entries 分配路径里没有真正的 GC blob 分配

这会误导后续维护者：以为 Set 仍有 GC-blob / malloc 双路径，实际代码不是这样。

### 6.4 `Map` 的 GC-blob 支持是一次性 / 易退化的

`Map` 初始支持 `node[]` 在 GC blob 上，但：

- rehash 强制新数组走 malloc
- `saved_gc_flag` 甚至没有恢复使用

因此“Map 节点在 GC heap”不是稳定属性，只是初始优化路径。

### 6.5 `Json.overflow` 外部内存完全不做 accounting

`overflow_grow()` 直接 `xr_realloc`，但没有：

- `xr_gc_add_external`
- `xr_gc_sub_external`

这会让 Json 外溢字段增长对 GC 压力不可见。

### 6.6 `StringBuilder` / `StrBuf` 也是 GC 盲区

`XrStringBuilder` 是 GC object，但内部：

- `XrStrBuf`
- `XrStrBuf.data`

都直接走 `xr_malloc`，也没有 external accounting。

### 6.7 `shape_id` 直接打包进 `gc.extra`，对象层与 GC header 强耦合

`xjson.h` 通过：

- `xr_gc_get_shape_id()`
- `xr_gc_set_shape_id()`

把 shape metadata 编码进 GC header extra bits。

这虽然高效，但带来：

- object/json 对 GC header bit layout 的直接耦合
- shape 扩展空间受 bit budget 限制

### 6.8 object 层里仍有静态全局可变状态

至少可见：

- `xmap.c` 的 `xr_map_dummynode`

它是共享静态节点，风险比可变表小，但 object 层整体仍没有完全回到“owner 明确、状态局部”的方向。

## 7. 正向发现

### 7.1 `Array` 的 typed storage 设计是清楚的

`XrArray` 已经把：

- `elem_type`
- `elem_tid`
- `has_gc_ptrs`
- `data_on_gc_heap`

拆清楚了。

这让 GC、AOT/JIT、容器语义都能共享同一份 object layout 信息。

### 7.2 `Json + Shape` 的 hidden class 方案比较成型

- in-object fields
- overflow spill
- immutable shape transition
- symbol→index table
- compact layout 支持

这已经是完整的对象布局服务，不是临时优化。

### 7.3 字符串的 global pool 与 GC 之间已有明确协作

- global strings 打 `SHARED`
- coroutine GC 碰到 global pooled string 不做 refcount tracking
- 改为设置 `ACCESSED`
- `xr_global_pool_sweep()` 做池内清扫

这套协作虽然复杂，但方向是自洽的。

### 7.4 Iterator 仍然保持了 lazy、轻量、owner 明确

在 object 层里，`xiterator.*` 是少数心智模型清楚、没有堆域混乱的实现。

## 8. 改进建议

### 8.1 先把 object owner taxonomy 显式化

建议至少在接口层区分清楚：

- coro-owned object
- shared/system-owned object
- global pooled object
- isolate metadata object
- malloc-backed helper object

否则后续继续叠加新对象类型时，很容易再复制当前 `string/shape/exception` 的混乱模式。

### 8.2 统一 external memory accounting

至少应该覆盖：

- `Map.node[]`
- `Set.entries[]`
- `Json.overflow`
- `StringBuilder.buffer`
- `XrStrBuf.data`

否则 GC 看到的 `totalbytes` 只代表 header + GC blob，而不是对象真实占用。

### 8.3 清理死标记 / 半退役路径

优先考虑：

- `XR_SET_FLAG_ENTRIES_ON_GC`
- `Map` rehash 后的 GC flag 语义
- 各对象 header 注释里对 backing storage 的过时描述

### 8.4 把 `xshape` 从“像 GC object”改成更明确的 metadata owner

当前最需要的是：

- 明确销毁路径
- 明确 registry owns shape
- 不要再给读者造成“shape 也是普通 GC object”的错觉

### 8.5 修正 `xexception` 的分配与释放契约

这块优先级很高。至少要二选一：

- 真正接入某个 owner 体系（coro/fixedgc/system)
- 或明确提供 manual destroy/arena owner

现在是中间态。

### 8.6 把 `string` 单独看作 runtime 服务

后续如果继续分析边界，`xstring.*` 最好不要再和普通容器对象放在同一心智层级。

它更接近：

- string runtime
- intern service
- shared object service
- unicode helper

## 9. 给下一轮 `class/ + symbol/` 的输入

下一轮建议重点沿这些线继续：

- `XrShape` 与 class field layout / instance field access 的边界
- symbol table 如何驱动 shape transition、Json field access、builtin method lookup
- reflection wrapper / metadata object 是否也存在 `XR_ALLOCATE` 风格 owner 缺口
- compact shape / compact field type 与 class/value-layout 的真实关系
- `xvalue_format` / `xvalue_print` 如何继续穿过 object → class/symbol 边界

---

## 10. 本轮状态

- **已完成**
  - Array/Map/Set/Json/String/Shape/Exception/StringBuilder 的 owner 与 backing 模型梳理
  - object 层 barrier / external memory accounting 覆盖面判断
  - string / shape / exception / stringbuilder 的跨层职责识别
  - `Set` flag 失活、`Map` rehash 退化、`Json` overflow 无 accounting 三类热点识别

- **未完成**
  - 还没系统梳理 `object/*_methods.c` 对 VM call/runtime invoke 的依赖面
  - 还没验证 reflection wrapper / xshape_cache 的 owner 问题
  - 还没回看 shared object deep-copy 与 object mutator 的覆盖率

- **下一步**
  - 开始 `028-runtime-phase4-class-symbol.md`
