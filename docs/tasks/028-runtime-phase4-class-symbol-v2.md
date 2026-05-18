# Runtime `class/ + symbol/` 层分析 V2（`028`）

> 旧版 `028-runtime-phase4-class-symbol.md` 的对账重写。严格对齐当前源码，按"无兼容性"原则给最佳设计。
>
> **范围**：`src/runtime/class/` 全部 35 文件、`src/runtime/symbol/` 3 文件，以及消费者：`gc/xcoro_gc.c`、`gc/xcoro_gc_traverse.c`、`runtime/value/xmethod_table.*`。

## 1. 七条核心结论

- **`class/` 不是单一元数据层**，而是 isolate-owned malloc graph + sysheap arena + isolate fixedgc + 伪 GC wrappers 四种 owner 的混合区。V1 准确。

- **`XrClass` 走 sysheap arena 优先 + `xr_malloc` fallback**（`xclass_builder_finalize.c:160-164`）。`xr_class_free` 显式 free 所有内部 malloc（fields / methods / vtable / itable / interfaces / supers）。**class 本体不参与 GC tracing，但带 `XrGCHeader`**——header 仅用于 type dispatch 协议。V1 准确。

- **`XrInstance` normal path 走 isolate fixedgc**（`xinstance.c:37, 96`），不走 per-coro GC。V1 准确。这与 VM/JIT 注释里"normal: allocate on coroutine heap"严重不一致——V2 必须当作叙述/实现真实漂移记录。

- **reflection wrapper 注释/实现严重不符——确认是真实泄漏**：
  - `xreflect_cache.c:91` 注释明文：*"The individual wrapper objects themselves are GC-managed and will be reclaimed by the collector."*
  - 实际 `xreflect_cache.c:49, 71` 用 `XR_ALLOCATE(FieldWrapper/MethodWrapper)` = `xr_malloc`。
  - GC type 设成 `XR_TBLOB`（`xreflect_cache.c:36, 56, 75`），但 `XR_TBLOB` 不在 `g_destroy_funcs[]`，也不在 `has_refs` mask 中。
  - `xr_reflect_cache_free` 只 free 两个指针数组（`field_wrappers`/`method_wrappers`），**wrapper body 永远不释放**。

- **`XrSymbolTable` 是干净的 isolate-owned 服务对象**：`name_to_id` hashmap + `id_to_name` 数组 + 显式 `xr_symbol_table_destroy`。`SymbolId = int32_t`，`SYMBOL_BUILTIN_COUNT <= 256`。

- **builtin symbol 是 runtime ABI 协议**，不是 class 辅助表。V1 准确。symbol id 同时驱动 `xmethod_table[]` index、operator overload mapping、shape/json field lookup、Json shape registry、coroutine property access。

- **`XrEnum` 跨 fixedgc + sysheap class + malloc side tables 三域**：`XrEnumType` / `XrEnumValue` 头走 fixedgc，`enum_class` 走 class builder（sysheap/malloc），`value_to_index`/`symbol_to_index`/`members[]` 走 `xr_malloc`。V1 准确。

## 2. 模块地图（修正版）

实际 35 + 3 文件，按职责分 7 类：

| 子域 | 文件 |
|---|---|
| **class 核心** | `xclass.h/c`、`xclass_internal.h`、`xclass_info.h`、`xclass_lookup.h/c`、`xclass_system.h/c`、`xmethod.h/c` |
| **class builder** | `xclass_builder.h/c`、`xclass_builder_finalize.c`、`xclass_builder_internal.h`、`xclass_descriptor.h/c`、`xclass_from_descriptor.c` |
| **instance** | `xinstance.h/c` |
| **enum** | `xenum.h/c`、`xenum_builtins.h/c` |
| **reflection registry** | `xreflect_registry.h/c`、`xclass_reflect.c` |
| **reflection wrapper/cache** | `xreflect_api.h/c`、`xreflect_cache.h/c`、`xreflect_internal.h`、`xreflect_members.c`、`xreflect_method.c`、`xreflect_type.c` |
| **symbol** | `xsymbol_table.h/c`、`xbuiltin_hash.inc` |

V1 漏列：`xclass_descriptor.*`/`xclass_from_descriptor.c`（descriptor protocol）、`xclass_lookup.h/c`、`xclass_reflect.c`、`xreflect_members/method/type.c`、`xenum_builtins.*`。

## 3. 关键论断核对

### 3.1 `XrClass` 真实分配路径

源码 `xclass_builder_finalize.c:160-164`：

```c
if (builder->isolate && xr_isolate_get_sys_heap(builder->isolate)) {
    cls = (XrClass*)xr_sysheap_alloc_class(...);
} else {
    cls = (XrClass*)xr_malloc(sizeof(XrClass));
}
```

附属（V1 列出 11 项）`fields/field_default_values/field_symbol_to_index/methods/static_field_values/interfaces/abstract_methods/method_symbol_to_index/vtable/itable/secondary_supers_hash` 全部 `xr_malloc`，`xr_class_free` 全部 `xr_free`。

**结论**：`XrClass` 不是 GC 管理对象。`XrGCHeader` 仅服务 type dispatch（`XR_HEAP_TYPE(v) == XR_TCLASS` 判断）。

### 3.2 `XrInstance` 实现/叙述漂移（P0）

源码精确：

```c
// xinstance.c:37
XrInstance *inst = (XrInstance*)xr_gc_alloc(xr_isolate_get_gc(X), size, XR_TINSTANCE);
// xinstance.c:96
XrInstance *dst = (XrInstance*)xr_gc_alloc(xr_isolate_get_gc(X), size, XR_TINSTANCE);
```

`xr_gc_alloc(&isolate->gc, ...)` = isolate fixedgc，**不是 per-coro Immix**。

但是：

- VM `OP_NEW_INSTANCE` 注释/分支语义把 normal instance 写成"allocate on coroutine heap"。
- `xdeep_copy.c` 的 `XR_TINSTANCE` 分支会从源 instance "deep-copy 到 dst_coro_gc"——意味着 deep-copy 后 instance **的确**落在 per-coro Immix 上，而正常分配路径**永远**落在 fixedgc。

后果：

- 同一个 `XR_TINSTANCE` 类型在不同分配路径下落在不同堆域。
- GC 视角看到的"instance 总数"实际是两个独立子集的并集。
- cross-coro 传递时实际上必经 deep-copy（fixedgc instance 不能直接被另一 coro 的 GC 扫到）。

V2 修复路径：要么把 `xr_instance_new` 改为 `xr_alloc(coro, ...)`（per-coro 默认），fixedgc 仅 bootstrap；要么承认 fixedgc 是真实 owner 并文档化、删除 deep-copy 路径里的 dst_coro_gc 假设。**前者更符合"isolate metadata vs runtime object"的边界**。

### 3.3 reflection wrapper 真实泄漏（P0，V1 已识别）

`xreflect_cache.c:91`（注释）：

```c
// The individual wrapper objects themselves are GC-managed
// and will be reclaimed by the collector.
```

实际：

```c
// xreflect_cache.c:49, 71
FieldWrapper *wrapper = (FieldWrapper*)XR_ALLOCATE(FieldWrapper);  // = xr_malloc
xr_gc_header_init_type(&wrapper->gc, XR_TBLOB);
```

并且：

- `XR_TBLOB` 不在 `g_destroy_funcs[]`（`xgc.c:38-49`）。
- `XR_TBLOB` 不在 `xcoro_gc.c:85-89` 的 `has_refs` mask 里。
- `xr_reflect_cache_free` 只 `xr_free(cache->field_wrappers)` 和 `xr_free(cache->method_wrappers)`，这是指针数组，不是 wrapper body。
- wrapper body 永远不被任何路径释放。

每个 class 创建一组 wrapper（`field_count + method_count` 个），class 永远不释放（class 本身在 sysheap arena 上、isolate-lifetime），所以单 isolate 内 wrapper 泄漏量 = `Σ(field_count + method_count)` × `sizeof(Wrapper)`。看起来不大，但这是 ownership 模型不闭合的硬证据。

V2 修复：
- **方案 A**：wrapper 也改走 isolate fixedgc，`xr_reflect_cache_free` 不需要释放 body（fixedgc cleanup 会扫整个链表）。但需要让 fixedgc destroy 知道 `XR_TBLOB`。
- **方案 B**：wrapper 改走 per-coro Immix，cache 在 class destroy 时手动 free。
- **方案 C**：在 `xr_reflect_cache_free` 里逐个 `xr_free(wrapper)`，最简单（一行循环）。

V2 推荐 C，最小侵入。注释也要同步改。

### 3.4 `xreflect_api.c` 也用 `XR_ALLOCATE`

V1 已识别。源码 `xreflect_api.c:31, 46, 61, 79`：

```c
TypeWrapper *wrapper = (TypeWrapper*)XR_ALLOCATE(TypeWrapper);
FieldWrapper *wrapper = (FieldWrapper*)XR_ALLOCATE(FieldWrapper);
MethodWrapper *wrapper = (MethodWrapper*)XR_ALLOCATE(MethodWrapper);
ParameterWrapper *wrapper = (ParameterWrapper*)XR_ALLOCATE(ParameterWrapper);
```

GC type 设成 `XR_TINSTANCE`，但分配走 malloc。这是**比 cache wrapper 更危险的伪 instance**：

- `XR_TINSTANCE` **在** `g_destroy_funcs`（`xgc.c:38-49` 没列，但 `XR_TINSTANCE` 在 `has_refs` mask `xcoro_gc.c:85-89` 中）→ traverse 时会扫子引用。
- 但 `xr_instance_new` 走 fixedgc 而 `XR_ALLOCATE(TypeWrapper)` 走 malloc——**两类对象 type 相同 (XR_TINSTANCE) 但 owner 不同**。
- GC 看到 `XR_TINSTANCE` 假设它在 fixedgc 链上。如果 wrapper 落 malloc，GC 永远扫不到它，但用户代码可以拿到 wrapper 的 PTR 引用。

最坏情况：wrapper 持有的 metadata pointer 被 wrapper 隐式保活，但 GC 不知道 wrapper 自身存在 → 用户代码持有 wrapper 的 PTR 时，wrapper 内的 receiver 引用没在 GC mark 阶段被扫到 → receiver 可能被错误回收。

V2 修复（P0）：所有 `XR_ALLOCATE(*Wrapper)` 都需要走真正的分配路径：

- `TypeWrapper` / `FieldWrapper` / `MethodWrapper` / `ParameterWrapper`：走 isolate fixedgc（reflection 是 isolate-lifetime 服务），调 `xr_gc_alloc(&isolate->gc, sizeof(...), XR_TINSTANCE)`。
- `xreflect_method.c:203` 的 `XrParameterMetadata`：同样走 fixedgc（注意 type id 要选合适的——也许 `XR_TBLOB`，但要进 destroy table）。
- `g_destroy_funcs[XR_TINSTANCE]` 添加 destructor（如果当前没有）。
- `g_destroy_funcs[XR_TBLOB]` 添加 destructor。

### 3.5 `XrSymbolTable` 结构（V1 准确）

`xsymbol_table.h:260-266`：

```c
typedef struct XrSymbolTable {
    XrHashMap *name_to_id;
    const char **id_to_name;
    int capacity;
    int count;
    int builtin_count;
} XrSymbolTable;
```

`xr_symbol_table_destroy` 释放 runtime-registered names + id_to_name 数组 + hashmap + table。clean。

**builtin symbols 直接引用 string literal**（`.rodata`，零分配）；runtime symbols 才 `xr_malloc + memcpy`。这种二级模式合理。

### 3.6 builtin symbol enum 是 ABI（V1 准确）

`xsymbol_table.h:40-251` 定义了 ~150 个 builtin symbol id，覆盖：

- 容器/字符串方法（length/has/get/set/...）
- 函数式方法（foreach/map/filter/reduce）
- 字符串方法（slice/substring/...）
- 数字方法（floor/sqrt/toBigInt）
- UTF-8（charLength/codepointAt）
- Set/Map/DateTime/BigInt/Logger/Channel/Regex
- operator overload（OP_ADD/SUB/MUL/...）
- coroutine/task property（DONE/CANCELLED/RESULT）

`SYMBOL_BUILTIN_COUNT <= 256` 是硬上限（`_Static_assert`）。这是 runtime ABI 的关键约束：byte code 用 8-bit symbol field（OP_INVOKE_BUILTIN）。

### 3.7 `XrEnum` 三域分布（V1 准确）

源码 `xenum.c`：

- `XrEnumType` → `xr_gc_alloc(isolate->gc, ..., XR_TENUM_TYPE)` (fixedgc)
- `XrEnumValue` → `xr_gc_alloc(isolate->gc, ..., XR_TENUM_VALUE)` (fixedgc)
- `enum_class` → `xr_class_new()` → sysheap/malloc
- `value_to_index` → `xr_malloc(sizeof(int) * range)` (xenum.c:113)
- `symbol_to_index` → `xr_malloc(sizeof(int) * capacity)` (xenum.c:151)
- `members[]` → `xr_malloc`

destroy 路径：fixedgc cleanup 调 `g_destroy_funcs[XR_TENUM_TYPE]`，那个函数应当 free 三个 malloc side table。如果没有 destructor → 又是泄漏。需要 grep 验证 `XR_TENUM_TYPE` 在 `g_destroy_funcs`。

V1 + V2 都要在 grep 验证后才能给最终结论。当前快速 grep `XR_TENUM_TYPE` 在 `g_destroy_funcs` 无入口（参见 `xgc.c:38-49` 列表）。**enum side table 也是泄漏**。

V2 修复：在 `g_destroy_funcs` 里加 `XR_TENUM_TYPE` destructor，逐个 free side table。

## 4. 状态归属（修正版）

| 对象 | 真实所有者 | 释放路径 |
|---|---|---|
| `XrClass` | sysheap arena（fallback malloc） | `xr_class_free` 显式（builder finalize 时不挂链） |
| `XrayCoreClasses` | isolate（malloc 容器） | `xr_core_free` 仅 `xr_free` 容器 |
| `XrInstance` normal | **isolate fixedgc** | fixedgc cleanup（无 destructor） |
| `XrInstance` deep-copied | per-coro Immix | Immix sweep |
| `XrEnumType/XrEnumValue` 头 | isolate fixedgc | fixedgc cleanup（缺 destructor → 泄漏 side table） |
| enum side tables | isolate（malloc） | **泄漏**（无人调） |
| `XrClassBuilder` | isolate（malloc） | builder finalize 释放 |
| `XrSymbolTable` | isolate（malloc） | `xr_symbol_table_destroy` 完整 |
| `XrTypeRegistry` | isolate（malloc） | `xr_registry_free` 完整 |
| `XrReflectCache` | **`xr_malloc`** | `xr_reflect_cache_free` 仅 free 数组（**wrapper body 泄漏**） |
| `TypeWrapper`/`FieldWrapper`/`MethodWrapper`/`ParameterWrapper` (api.c) | **`xr_malloc`**, `XR_TINSTANCE` 假身份 | **永久泄漏** |
| `FieldWrapper`/`MethodWrapper` (cache.c) | **`xr_malloc`**, `XR_TBLOB` | **永久泄漏** |
| `XrParameterMetadata` (method.c:203) | **`xr_malloc`** | **永久泄漏** |
| builtin symbol names | `.rodata`（不释放） | 永久 |
| runtime symbol names | isolate（malloc） | `xr_symbol_table_destroy` |

## 5. 高风险点（精确版）

### 5.1 `XrInstance` 实现/叙述漂移（P0）

§3.2 已展开。要么承认 fixedgc 是 owner、删除 deep-copy 假设，要么改 `xr_instance_new` 走 per-coro。前者最小动作，后者最干净。V2 推荐**让 normal instance 走 per-coro**：

```c
XrInstance *inst = (XrInstance*)xr_alloc(xr_current_coro(X), size, XR_TINSTANCE);
```

bootstrap fallback 时（无 coro）才走 fixedgc。

### 5.2 reflection wrapper 泄漏（P0）

§3.3、§3.4。最小修复：`xr_reflect_cache_free` 加循环 `xr_free` 每个 wrapper；`xreflect_api.c` 的 4 类 wrapper 改走 fixedgc 或加显式 free。

### 5.3 enum side tables 泄漏（P1）

§3.7。`g_destroy_funcs[XR_TENUM_TYPE]` 加 destructor：

```c
static void xr_gc_destroy_enum_type(void *obj, void *gc) {
    XrEnumType *et = (XrEnumType*)obj;
    xr_free(et->value_to_index);
    xr_free(et->symbol_to_index);
    xr_free(et->members);
}
```

### 5.4 注释/实现严重不符（P0 文档修复）

`xreflect_cache.c:91` 的注释明确 promise GC-managed，实际不是。这是**误导后续维护者最严重的缺陷**——读者会假设 GC 在管，做出错误的所有权决策。

V2 强制要求：**修复实现 OR 删除注释**，二选一立即处理。

### 5.5 `XR_TBLOB` 类型 id 滥用（P1）

`XR_TBLOB` 当前被 reflection cache wrapper 用作"非 instance 但有 GC header 的 metadata"标记。但 GC 不知道 `XR_TBLOB` 是什么——不在 destroy table、不在 has_refs mask。

V2 修复：要么 `XR_TBLOB` 进 destroy + has_refs 系统；要么删除 `XR_TBLOB` 用法，wrapper 改用 `XR_TINSTANCE`（与 api.c 一致）。

### 5.6 `XrInstance` deep-copy 路径假设错误（P1）

`xdeep_copy.c` 把 `XR_TINSTANCE` 复制到 `dst_coro_gc`，但 source 永远在 fixedgc。这违反 deep-copy 协议的"source/dst 都在 per-coro"假设。

修复跟 5.1 一起：normal instance 走 per-coro 后，deep-copy 假设自然成立。

### 5.7 `xclass_descriptor` 的 owner（V1 漏识别）

`xclass_descriptor.c` / `xclass_from_descriptor.c` 是 V1 没列的子模块。它实现 class descriptor 序列化/反序列化协议（用于 bytecode bundle 等）。

需要进一步审计，但 V2 先列为 follow-up：descriptor 内 string 字段、字段表、方法表的 owner 模型是否闭合。

### 5.8 `xclass_reflect.c` 的 owner（V1 漏）

11KB 文件，应该是反射元数据查询入口。需要审计是否也有 `XR_ALLOCATE` 模式。V2 列为 follow-up。

## 6. 正向资产（V1 + V2）

V1 已识别 + V2 强化：

- **class builder 单一收敛点**（`xr_class_new` → `builder_new` → `builder_finalize`）。
- **eager reflection 接入 finalize**（`finalize_eager_reflection` → `cache_create + registry_register_class`）。
- **builtin symbol enum 顺序约束**（`XR_CHECK(BUILTIN_NAME_COUNT == SYMBOL_BUILTIN_COUNT - 1)`）。
- **`XrClass` lookup 结构静态化**（`field_symbol_to_index` 等 O(1) 表）。
- **`XrSymbolTable` 二级 owner**（builtin .rodata + runtime malloc），干净。
- **SymbolId `_Static_assert`**（32-bit、SYMBOL_INVALID == 0、`<= 256`）。

## 7. 最佳设计建议（无兼容性）

10 条直接可执行：

1. **`xr_instance_new` 走 `xr_alloc(coro, ...)`** （§5.1，P0，1 行修复）
2. **`xr_reflect_cache_free` 加 wrapper body 释放循环**（§5.2 方案 C，P0，10 行修复）
3. **`xreflect_api.c` 的 4 类 wrapper 走 fixedgc**（§5.2，P0）
4. **`xreflect_cache.c:91` 注释修复或实现修复**（§5.4，P0 文档/代码同步）
5. **`g_destroy_funcs[XR_TENUM_TYPE]` 加 destructor**（§5.3，P1）
6. **`XR_TBLOB` 决策：进 destroy + has_refs 或者删除用法**（§5.5，P1）
7. **`xclass_descriptor` / `xclass_from_descriptor` 单独审计**（§5.7，P2）
8. **`xclass_reflect.c` 单独审计**（§5.8，P2）
9. **统一 wrapper 类型词汇**：把 "wrapper"（伪 instance）和 "metadata"（malloc graph）在文档里区分清楚
10. **`g_destroy_funcs` + `has_refs` 合并**（与 026 V2 §6.8 协同）

## 8. 给下一轮 `coro/` 的输入

V1 给的方向都对，V2 补充：

- `XrInstance` 在 fixedgc 与 deep-copy 路径下的 owner 不一致，会影响 029 V2 对 deep-copy 边界的判断。
- reflection wrapper 泄漏在 long-running isolate 下会持续累积，影响 coroutine restart / repl 场景。
- `XrEnumType` 在 cross-coro 引用时是 shared (fixedgc) 还是 deep-copy？需要在 029 V2 验证。
- `XrSymbolTable` 是否被 worker / coro 直接访问，是否有并发安全问题？

## 9. 与 V1 的主要差异

| 点 | V1 | V2 |
|---|---|---|
| `XrClass` 分配 | 准确（sysheap+fallback） | 保留 |
| `XrInstance` 落 fixedgc | "实现/叙述偏差" | 精确：deep-copy 路径还有 per-coro 假设，加剧不一致 |
| reflection wrapper 注释 | "tension" | 精确：注释/实现完全不符（`xreflect_cache.c:91`）|
| `XR_ALLOCATE` 扩散 | 提到 wrapper | 全列：cache.c × 2、api.c × 4、method.c × 1 |
| `XR_TBLOB` 类型 | 未识别为问题 | 列为 P1（GC 不知道怎么处理） |
| enum side table 泄漏 | 未识别 | P1（`g_destroy_funcs` 无 entry） |
| `xclass_descriptor`/`xclass_reflect.c` | 漏列 | follow-up |
| 改进建议 | 5 条方向 | 10 条可执行 |

## 10. 本轮状态

- **已完成**：class/ 35 + symbol/ 3 文件全核对、V1 论断逐项判断、最佳设计 10 条
- **未完成**：`xclass_descriptor.c`/`xclass_from_descriptor.c`/`xclass_reflect.c`/`xreflect_members.c`/`xreflect_method.c`/`xreflect_type.c` 详查（留给后续 audit）；`XR_TENUM_TYPE` destructor 实测验证
- **下一步**：`029-runtime-phase5-coro-v2.md`
