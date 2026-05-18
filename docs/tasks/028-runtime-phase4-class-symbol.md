# Runtime `class/ + symbol/` 层分析（`028`）

> 本轮聚焦 `src/runtime/class/` 与 `src/runtime/symbol/`。
>
> 这一层的核心不是“面向对象语法糖”，而是几套 runtime 元数据服务的汇合：
>
> - class / instance layout
> - symbol id 协议
> - builtin method dispatch 表
> - reflection registry / wrapper / cache
> - enum runtime representation
>
> 真正需要回答的是：
>
> - class / instance / enum / reflect wrapper 分别归谁拥有
> - 哪些对象虽然带 `XrGCHeader`，其实不走统一 GC owner 模型
> - symbol table 和 builtin symbol enum 如何成为 VM / object / class 的共同协议
> - 哪些“类系统”代码其实已经是 reflection / API / dispatch 基础设施

## 1. 本轮结论

- **`class/` 不是普通 L3 元数据层，而是“系统堆 + malloc + fixedgc + reflection registry”混合区。**

- **`XrClass` 自身不是 per-coro GC 对象。**
  - `xr_class_builder_finalize()`：优先 `xr_sysheap_alloc_class()`
  - 早期 bootstrap fallback 才 `xr_malloc`
  - `xr_class_free()` 做显式释放

- **`XrInstance` 的 normal path 当前落在 isolate fixedgc，不在 per-coro heap。**
  - `xr_instance_new()` → `xr_gc_alloc(xr_isolate_get_gc(X), size, XR_TINSTANCE)`
  - 与 VM 注释里“normal: allocate on coroutine heap”并不一致

- **`symbol` 已经是整个 runtime 的公共协议层，不只是 class 辅助表。**
  - builtin symbol enum
  - operator overload flag mapping
  - shape/json field lookup
  - builtin method dispatch table indexing
  - VM/AOT builtin invoke

- **reflection wrappers/caches 是第二批“带 GC header 但 owner 不统一”的对象。**
  - `TypeWrapper` / `FieldWrapper` / `MethodWrapper` / `ParameterWrapper`
  - `XrReflectCache`
  - 都主要经由 `XR_ALLOCATE` / `xr_malloc`
  - 但对外又伪装成 `XR_TINSTANCE` / `XR_TBLOB`

- **enum 也是混合生命周期。**
  - enum type/value 头部走 isolate fixedgc
  - enum class 走 class builder → sysheap/malloc
  - members/value map 走 `xr_malloc`

## 2. 模块地图

| 子域 | 关键文件 | 真实职责 |
|---|---|---|
| class 核心 | `xclass.*` `xclass_internal.h` | `XrClass` 结构、lookup、operator flags、free |
| class builder | `xclass_builder*.c` | 构建 immutable class，铺平字段/方法/vtable/itable |
| instance | `xinstance.*` | 实例分配、字段读写、构造器调用 |
| core class system | `xclass_system.*` | builtin/root class bootstrap |
| enum | `xenum.*` | enum type/value 运行时表示 |
| symbol table | `xsymbol_table.*` | builtin + runtime symbol id 体系 |
| reflection registry | `xreflect_registry.*` | `XrClass -> XrTypeMetadata` registry |
| reflection wrapper/cache | `xreflect_api.*` `xreflect_internal.h` `xreflect_cache.*` | zero-copy metadata view、wrapper object、per-class cache |
| builtin dispatch protocol | `runtime/value/xmethod_table.*` | `XrTypeId -> XrMethodSlot[]` 单一真相源 |

## 3. 所有权与堆域

### 3.1 `XrClass`：系统堆 / malloc，不是 per-coro GC

`xr_class_builder_finalize()` 的 class header 分配逻辑非常明确：

- 有 `sys_heap` → `xr_sysheap_alloc_class(...)`
- 否则 → `xr_malloc(sizeof(XrClass))`

后续附属结构几乎全部是 `xr_malloc`：

- `fields`
- `field_default_values`
- `field_symbol_to_index`
- `methods`
- `static_field_values`
- `interfaces`
- `abstract_methods`
- `method_symbol_to_index`
- `vtable`
- `itable`
- `secondary_supers_hash`

`xr_class_free()` 也明确逐项 `xr_free()`，最后 `xr_free(cls)`。

因此结论很清楚：

- **`XrClass` 不是“类对象也在 GC 上”的模型**
- 它是 isolate/system-owned metadata object

`XrClass` 带 `XrGCHeader` 更多是为了：

- 统一类型判断
- 某些 value/class API 的 header protocol

而不是表示它进入了 per-coro tri-color 生命周期。

### 3.2 `XrInstance`：normal path 走 fixedgc

`xinstance.c` 当前实现：

- `xr_instance_new()` → `xr_gc_alloc(xr_isolate_get_gc(X), size, XR_TINSTANCE)`
- `xr_instance_clone()` 同样如此

这说明 normal instance 并不是：

- coro heap object
- system heap shared object

而是：

- **isolate fixedgc object**

但 VM 路径里的注释/分支语义仍在写：

- shared → `xr_sysheap_alloc_shared(..., XR_TINSTANCE)`
- normal → `xr_instance_new(isolate, klass)`，并注释成“allocate on coroutine heap”

所以当前存在一个很重要的现实/叙述偏差：

- **实现：fixedgc**
- **上层心智：per-coro normal object**

这会直接影响后面对 runtime object ownership 的整体判断。

### 3.3 `XrayCoreClasses`：单独的 isolate 容器

`xr_core_init()`：

- `X->core = xr_malloc(sizeof(XrayCoreClasses))`
- 各 builtin class 指针挂到这里

`xr_core_free()`：

- 只 `xr_free(X->core)`
- 不负责释放里面的 class 本体

说明 core container 与 class body 不是同一 owner：

- container：malloc 容器
- classes：sysheap/malloc metadata object

### 3.4 `XrSymbolTable`：isolate-owned malloc object

`xsymbol_table.c` 的生命周期很标准：

- `xr_symbol_table_create()`
  - `xr_malloc(XrSymbolTable)`
  - `xr_hashmap_new()`
  - `xr_malloc(id_to_name[])`
- `xr_symbol_table_destroy()`
  - free runtime-registered names
  - free `id_to_name`
  - free hashmap
  - free table

因此 symbol table 是：

- isolate-owned service object
- 完整手动释放
- 不走 GC/system heap owner 协议

### 3.5 `XrTypeRegistry`：另一份 isolate-owned malloc graph

`xreflect_registry.c`：

- registry 本体 `xr_malloc`
- `types[]` `xr_malloc`
- `type_map` `xr_hashmap_new()`
- metadata 也是 `xr_malloc`

`xr_registry_free()` 会统一释放这些结构。

因此反射 type registry 也是：

- isolate-owned malloc graph
- 不走 GC/system heap

### 3.6 reflection wrapper / cache：最复杂的一批 owner

`xreflect_api.c` / `xreflect_cache.c` 里可以看到两类对象：

#### 包装对象

- `TypeWrapper`
- `FieldWrapper`
- `MethodWrapper`
- `ParameterWrapper`

分配路径：

- `XR_ALLOCATE(...)` → `xr_malloc(sizeof(...))`

但 header/type 设置成：

- `xr_gc_header_init_type(..., XR_TINSTANCE)`
- 或 cache 内部 wrapper 设成 `XR_TBLOB`

#### cache 对象

- `XrReflectCache` 本体：`XR_ALLOCATE(XrReflectCache)`
- `field_wrappers[]` / `method_wrappers[]` 数组：`xr_malloc`
- `xr_reflect_cache_free()` 手动回收 cache 和 wrapper arrays

也就是说这里并不存在单一 owner：

- wrapper value 对外表现得像 runtime object
- 但真实分配是裸 `xr_malloc`
- cache 注释又说 wrapper 会“被 collector 回收”

这是本轮最明显的 ownership tension 之一。

### 3.7 `XrEnumType` / `XrEnumValue`：固定头 + malloc 尾部

`xenum.c`：

- `XrEnumValue` → `xr_gc_alloc(isolate->gc, ..., XR_TENUM_VALUE)`
- `XrEnumType` → `xr_gc_alloc(isolate->gc, ..., XR_TENUM_TYPE)`
- `enum_class` → `xr_class_new()` → sysheap/malloc
- `members[]` → `xr_malloc`
- `value_to_index` / `symbol_to_index` 等辅助表 → `xr_malloc`

所以 enum 不是单体对象，而是：

- fixedgc header object
- 指向 class metadata
- 挂着多个 malloc side table

## 4. symbol 体系的真实职责

### 4.1 builtin symbol enum 是跨模块协议，不只是 symbol table 细节

`xsymbol_table.h` 的 builtin enum 现在已经驱动：

- 容器/String builtin 方法名
- operator overload symbol
- coroutine/task/channel property symbol
- logger / bigint / regex / datetime 等 runtime 名称

这意味着 symbol id 不是可替换实现细节，而是：

- **runtime ABI-ish 协议**

### 4.2 symbol table 不复用 string pool，而是自持名字副本

`xr_symbol_register_in_table()`：

- 对 runtime symbol name 做 `xr_malloc + memcpy`
- 存进 `id_to_name[]`
- 再注册进 hashmap

builtin names 直接引用 string literal，不复制。

所以 symbol table 与 string intern pool 是平行体系，不共享所有权：

- string pool：`XrString` object
- symbol table：`const char *` owned copy

### 4.3 `xr_symbol_intern()` 本质是“symbol table stable name resolver”

它不是返回 `XrString*`，而是：

- 在 symbol table 注册 name
- 返回 `id_to_name` 中稳定的 `const char*`

这让 class/enum/metadata 可以直接保存：

- `const char *name`

而不是保存 `XrString*`。

## 5. builtin dispatch 与 class/symbol 的交点

### 5.1 `xmethod_table.*` 已经是 value/object/class 共用协议

`xmethod_table.h` 明确了：

- 每个 builtin type 拥有自己的 `static const XrMethodSlot[]`
- 全局 `xr_builtin_method_tables[XR_TID_COUNT]`
- VM `OP_INVOKE_BUILTIN` 与 AOT 都从这里 resolve

这使 symbol id 成为：

- table index
- method dispatch key
- backend specialization key

### 5.2 迁移并不完整，仍有 legacy/native_type_classes 双轨

`xmethod_table.c` 里当前只填了部分类型：

- bool/int/float/bigint/set/map/json/array/string/datetime/regex

而注释也明确：

- Iterator / StringBuilder 等仍走旧的 `native_type_classes` / legacy dispatch

因此 class system 目前同时承载：

- builtin method table dispatch
- native_type_classes-based class dispatch

## 6. 正向发现

### 6.1 class builder 已经形成单一收敛点

`xr_class_new()` 不再自己手搓 class，而是统一走：

- `xr_class_builder_new()`
- `xr_class_builder_finalize()`

这让：

- builtin class
- enum class
- user-defined class

至少在构建路径上走向一致。

### 6.2 eager reflection 已经接入 finalize

`finalize_eager_reflection()` 会做两件事：

- `xr_reflect_cache_create()`
- `xr_registry_register_class()`

这比旧的 lazy scattered registration 清晰很多。

### 6.3 symbol enum + builtin name table 的顺序约束是明确的

`xsymbol_table.c` 用：

- `xr_builtin_symbol_names[]`
- `SYMBOL_BUILTIN_COUNT`
- `XR_CHECK(BUILTIN_NAME_COUNT == SYMBOL_BUILTIN_COUNT - 1)`

把 enum 顺序和 runtime 注册顺序绑死，这对 VM/AOT 共用 dispatch 很重要。

### 6.4 `XrClass` 的 lookup 结构已经明显偏静态编译期风格

例如：

- `field_symbol_to_index`
- `method_symbol_to_index`
- `primary_supers`
- `secondary_supers_hash`
- `vtable`
- `itable`

这些都说明 class 层已经在往“只读 metadata + O(1) lookup”方向稳定。

## 7. 已确认高风险点

### 7.1 `XrInstance` 所有权与上层语义不一致

当前最关键的问题之一：

- `xr_instance_new()` 实现落 fixedgc
- VM/JIT/注释仍把 normal instance 当作 coroutine-heap object 来谈

这会影响：

- GC owner 判断
- cross-coro deep copy 语义
- object/class 层边界认知

### 7.2 reflection wrappers 是“伪 instance”

`xreflect_api.c` 里：

- `TypeWrapper` / `FieldWrapper` / `MethodWrapper` / `ParameterWrapper`
- 都用 `XR_ALLOCATE`
- 却把 header type 设成 `XR_TINSTANCE`

但它们并没有：

- `XrInstance` 的真实布局
- 统一 GC owner
- 对应 class-instance field contract

这表示：**reflection wrapper 在 value 层伪装成 instance，但分配与布局并不遵循 instance owner 模型。**

### 7.3 `XrReflectCache` 注释与实现也存在张力

代码注释说 wrapper “will be reclaimed by the collector”，但当前实现里：

- wrapper 分配走 `XR_ALLOCATE`
- cache 自己只 free wrapper arrays，不 free wrapper body

如果 wrapper body 没有挂到统一 GC owner，这里的所有权就不闭合。

### 7.4 `XrClass` 带 `XrGCHeader`，但本质不是 GC object

和前一轮的 `XrShape` 类似，`XrClass` 也容易误导阅读者：

- 有 GC header
- 可以出现在 value/type 判断里
- 但分配/释放完全不走 per-coro GC

### 7.5 symbol table 与 string pool 双份名字所有权

当前：

- string runtime 自己维护 `XrString`
- symbol table 自己复制 `const char *`

这会让名字系统出现：

- 一份 object-level string
- 一份 symbol-level stable C string

虽可工作，但 owner graph 更复杂。

### 7.6 enum 同时跨 fixedgc / class sysheap / malloc 三域

enum 的 header、class、side table 分落三处：

- `XrEnumType/XrEnumValue` fixedgc
- `enum_class` sysheap/malloc
- `members/value_to_index` malloc

这是当前 class 层里 ownership 最混合的类型之一。

## 8. 改进建议

### 8.1 先统一“class metadata object”和“runtime heap object”的词汇

至少应该明确区分：

- class metadata (`XrClass`, registry, symbol table)
- fixedgc runtime object (`XrInstance`, enum type/value)
- reflection wrapper pseudo-object

否则后续每次看到 `XrGCHeader` 都容易误判 owner。

### 8.2 重新审视 `XrInstance` 的 normal allocation 策略

当前要么：

- 承认并文档化：instance 就是 fixedgc object
- 要么改回真正的 coroutine-owned object

现在实现和叙述不一致，是最危险的状态。

### 8.3 为 reflection wrappers 建立单一 owner 模型

目前至少需要在三种方案里选一个：

- 真正接入 fixedgc/system heap/专用 arena
- 明确 cache owner 并集中 free wrapper body
- 或改成非 pseudo-instance 的纯 metadata handle

### 8.4 收敛 class/reflect 的 pseudo-GC 类型使用

当前 `XR_TINSTANCE` / `XR_TBLOB` 被多种非标准布局对象借用：

- reflect wrapper
- reflect cache

这对 GC/traverse/value-format/type checks 都是潜在歧义源。

### 8.5 评估 symbol table 与 string pool 的去重机会

不一定要立即合并，但至少要明确：

- 为什么 symbol name 需要独立副本
- 何处必须是 `const char *`
- 何处可以安全地指向 pooled string storage

## 9. 给下一轮 `src/coro/` 的输入

下一轮建议重点从这些接口切入：

- class/object/shared/fixedgc 在 coroutine 深拷贝中的行为差异
- task/channel/instance/shared let 与 symbol/method lookup 的交点
- `xr_current_coro()` / `xr_current_worker()` 如何影响 class/object/runtime API 的分配语义
- VM/JIT 对 `new Class()` / constructor invoke 的真实 owner 假设

---

## 10. 本轮状态

- **已完成**
  - class / instance / symbol / reflect / enum 的 owner 与堆域梳理
  - builtin symbol enum 与 method table 的协议定位
  - reflection wrapper/cache 的 pseudo-GC owner 缺口识别
  - `XrInstance` normal path 落 fixedgc 的事实确认

- **未完成**
  - 还没系统梳理 `xmethod_call` / legacy native_type_classes 的剩余覆盖范围
  - 还没回看 `xproperty_descriptor` / `xconstructor` 等更细粒度元数据类型
  - 还没把 class/symbol 与 `src/coro/` 的 deep-copy / shared 规则对上

- **下一步**
  - 开始 `029-runtime-phase5-coro.md`
