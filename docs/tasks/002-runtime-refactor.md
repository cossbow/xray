# `src/runtime/{closure,symbol,value,object}` 重构规划

> **目标**：把 runtime 4 个核心子模块（closure 6 文件、symbol 2 文件、value 14 文件、object 34 文件，共约 **11 600 行**）调整到 "代码简洁 / 性能最优 / 设计一致" 的状态。
>
> **原则**：**不考虑向后兼容性**。Xray 无外部用户，每阶段选最佳设计；旧接口可删、可改名；不做兼容层。
>
> **规模基线**：`xstring.c` 1 452、`xbigint.c` 1 317、`xtype.c` 1 250（最大三）；`xtype.h` 610、`xchunk.h` 671（接近 800 硬线）。
>
> **诊断来源**：近期静态分析 + 根审计；关键判断均带 `@/...:line` 引用。

## 总体原则

1. **每阶段独立可回滚**；阶段结束必过 `/test` + `scripts/run_regression_tests.sh`。
2. **硬性规则对齐**：
   - `.c` ≤ 3000 / `.h` ≤ 800 / 单函数 ≤ 150
   - 统一 `xr_malloc/xr_free`
   - 非 static 必须 `XR_FUNC`/`XRAY_API`
   - **禁止文件作用域可变全局（含 `__thread`）**
   - include 路径以文件所在目录为基准
3. **正确性优先**：P0 bug（UB、heap_type 未初始化、数据竞争）前 3 阶段清零。
4. **性能 KPI**：
   - `bench/array_push_1M`（Array<int> 密集分配）
   - `bench/json_create_1M`（Shape transition）
   - `bench/type_check_dense`（类型推断 50k 节点）
   - `bench/bigint_mul_256`

## 阶段总览

| # | 阶段 | 目标 | 风险 | 工时 | 主要文件 |
|---|------|------|------|------|----------|
| 0 | 杂项清理（include / 死文件 / 错误注释） | 消噪音 | 极低 | 0.5d | 4h、3c |
| 1 | P0 正确性 bug 修复 | UB / 错误语义清零 | 低 | 0.5d | `xrange.h`、`xshape.c`、`xchunk.c` |
| 2 | 死代码大清扫 | 删除未接通的系统 | 低 | 1d | `xnative_type_registry`、`xruntime_type_info`、`XrContext` |
| 3 | 文件作用域可变全局清零 | 合规 + 多 isolate 正确 | 中 | 1.5d | `xsymbol_table`、`xshape`、`xtype` |
| 4 | SymbolId / XrProto 瘦身 | 内存 -30% / proto | 中 | 1d | `xsymbol_table.h`、`xchunk.{h,c}` |
| 5 | `xtype.h` 拆分 + inline 下沉 | 头合规 + 编译范围缩小 | 中 | 1d | `xtype*` |
| 6 | Value 层一致性（deep_eq / hash / 宏统一） | 消除割裂实现 | 低 | 0.5d | `xvalue.c`、`xbound_method.c` |
| 7 | Shape 架构重塑（GC 所有权 + 稀疏映射） | 全局 registry 收口 | 中 | 1d | `xshape*` |
| 8 | 大文件拆分 + 其它性能优化 | 所有 .c ≤ 800 行 + BigInt 64-bit limb | 中 | 1.5d | `xstring*`、`xbigint*` |

**总计**：约 7.5 工作日。

---

## 阶段 0：杂项清理

### 目标
清除零风险噪音；独立可合入。

### 步骤

#### 0.1 统一错误的 include 路径

4 个头文件的相对路径错写成 `"../base/xdefs.h"`（应为 `"../../base/xdefs.h"`）：

- `@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xarray.h:24`
- `@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xarray_class_init.h:18`
- `@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xset_class_init.h:18`
- `@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xmap_instance_methods.h:15`

**操作**：`sed -i '' 's|"../base/xdefs.h"|"../../base/xdefs.h"|g' ...`

#### 0.2 修正 `#endif` 注释

`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xvalue.h:59-61`：`#endif // ========== Value Tag Enum ==========` 注释错粘下节标题。改为 `#endif` 或 `#endif // !XR_64BIT`。

#### 0.3 删除空文件 `xslice_new.c`

`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xslice_new.c` 仅 9 行注释头。`git rm` + 从 CMakeLists.txt 移除。

#### 0.4 清除冗余语句

`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/symbol/xsymbol_table.c:90-91`：`size_t old_size = ...; (void)old_size;` 直接删。

全局扫：`grep -rn '(void)[a-z_]*;$' src/runtime/{closure,symbol,value,object}`。

#### 0.5 删除 `xexception.h` 重复 `XR_IS_EXCEPTION` 定义

`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xvalue.h:172` 已定义。删除 `@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xexception.h:74-77` 的重复块。

#### 0.6 魔数具名

`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xvalue_hash.c:28`：`case XR_TAG_NULL: return 6;` → 定义 `#define XR_HASH_NULL 6u` 并注释说明。

### 验证
- `/build` + `/test`
- `grep -rn '"../base/xdefs.h"' src/runtime/` 应为空

### 风险
极低。每项独立 commit。

---

## 阶段 1：P0 正确性 bug 修复

### 1.1 `xr_value_from_range` 未初始化 heap_type（严重 UB）

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xrange.h:78-83`

```c
static inline XrValue xr_value_from_range(XrRange *r) {
    XrValue v;
    v.tag = XR_TAG_PTR;
    v.ptr = r;
    return v;
}
```

`v.flags / v.heap_type / v.ext` 栈垃圾。`XR_IS_RANGE` 检查 `heap_type == XR_TRANGE`，优化/栈布局变化时**偶发误判**。

**VM 已在用**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm.c:2551,2566`、`@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm_cold_paths.c:1497`。

**修复（一行）**：
```c
static inline XrValue xr_value_from_range(XrRange *r) {
    return XR_FROM_PTR(r);
}
static inline XrRange* xr_value_to_range(XrValue v) {
    return (XrRange*)XR_TO_PTR(v);
}
```

### 1.2 `symbol_to_index` int8_t 溢出

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xshape.h:75`、`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xshape.c:104-111`

`int8_t` 范围 `-128..127`，但 `SHAPE_MAX_TOTAL_FIELDS = 256`，index 128 存成 `0x80` = -128，被误认 "not found"。

**修复**：`int8_t *symbol_to_index` → `int16_t *symbol_to_index`；`memset(..., -1)` 对 int16 仍有效（0xFFFF = -1）。

### 1.3 `xr_array_set_element` 对非数值 XrValue 读垃圾

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xarray.h:213-278`

```c
case XR_ELEM_I8: {
    int64_t v = XR_IS_INT(value) ? XR_TO_INT(value) : (int64_t)XR_TO_FLOAT(value);
```

`value` 为 BOOL/PTR/STRING/NULL 时 `XR_TO_FLOAT` 返回垃圾 double → UB。

**修复**：新增 helper（放 `xvalue.h`）：
```c
static inline int64_t xr_value_to_int64_coerce(XrValue v) {
    if (XR_IS_INT(v))   return XR_TO_INT(v);
    if (XR_IS_FLOAT(v)) return (int64_t)XR_TO_FLOAT(v);
    if (XR_IS_BOOL(v))  return XR_IS_TRUE(v) ? 1 : 0;
    XR_CHECK(false, "type confusion: non-numeric value written to typed array");
    return 0;
}
```

策略选 **panic**（强类型语言风格 fail-fast）；先 `/test` 确认无依赖"静默写 0"的测试。

### 1.4 `xchunk.c` `xr_vm_proto_new` 删除冗余 manual 赋值

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xchunk.c:69-141`

`xr_calloc(1, sizeof(XrProto))` 已 zero-fill，40+ 字段的 `= NULL / 0 / false` 全冗余（且已漏写 `jit_fast_entry`、`stack_map`、`deopt_backoff`）。

**修复**：删所有标量赋值；保留 `DYNARRAY_INIT(...)` 和带状态的 init 调用；加注释：
```c
// NOTE: All scalar fields are zero-initialized by xr_calloc.
// Only containers that require explicit init are called below.
```

### 1.5 `xr_proto_add_symbol` 超限改 panic

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xchunk.c:278-282`

当前 >254 符号打印 warning 后继续写入，但 iABC 的 B/C 是 8 位，后续代码生成会静默错误。

**修复**：`XR_CHECK(local_idx < 255, "proto: too many unique symbols (>254), function too complex")`。

### 验证
- `/build` + `/test` + `/build-asan` + `/test`
- 新增 `tests/unit/test_range_heap_type.c`：`xr_value_from_range` → `XR_IS_RANGE` 在 `-O0` 和 `-O3` 下都返回 true
- 新增 `tests/unit/test_shape_field_index_128.c`：field_count=128 shape 的 index 128 读取正确

### 风险
- 1.3 panic 策略若破坏现有测试，先降级为"静默 0 + warning log"，跟进真实用例决定最终策略。

### 回滚
1.1-1.5 独立 commit，单项可 revert。

---

## 阶段 2：死代码大清扫

### 2.1 删除 `xnative_type_registry.{h,c}`（消除类型名冲突）

**问题**：`XrNativeTypeInfo` 在两个头文件定义**完全不同的结构**：

```@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xnative_type.h:39-45
typedef struct XrNativeTypeInfo {
    const char *name;
    XrObjType gc_type;
    XrNativeMethod *methods; /* ... */
} XrNativeTypeInfo;
```

```@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xnative_type_registry.h:39-49
typedef struct XrNativeTypeInfo {
    uint16_t type_id;
    const char *name;
    size_t basic_size;
    void (*gc_traverse)(struct XrGC*, void*); /* ... */
} XrNativeTypeInfo;
```

当前侥幸不崩：没有 TU 同时 include。未来任何误同时 include → 立刻 `error: redefinition`。

**活跃度**：
- `xnative_type.h` → `stdlib/{regex,datetime,log}` 使用（活跃）
- `xnative_type_registry.h` → **无任何调用 `xr_native_registry_register`**（死代码）

**操作**：
1. `git rm src/runtime/object/xnative_type_registry.{h,c}`
2. 删 `@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/xisolate_internal.h:141` 的 `void *native_type_registry;`
3. 删 `@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/xisolate_api.c:144-155` 的 get/set hook
4. CMakeLists.txt 移除 source
5. `grep -rn 'XrNativeObject\|xr_gc_traverse_native_object\|xr_finalize_native_object'` 清残余

### 2.2 删除 `xruntime_type_info.c`

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xruntime_type_info.c` (137 行)

```@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xruntime_type_info.c:74-81
// TODO: RTI system is defined but not yet integrated into GC/method dispatch.
void runtime_type_info_init(void) {
    // Currently NULL, will be set during integration
}
```

25 个 `rti_*` 单例全 NULL，`runtime_type_info_init` 空实现，从未启用。

**操作**：`git rm src/runtime/value/xruntime_type_info.{c,h}` + CMakeLists.txt 移除。

### 2.3 `XrContext` 彻底迁移到 `XrCell`

**矛盾**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/closure/xcell.h:16-18` 声称 "No new XrContext objects are created"，但 `@/Users/xuxinglei/workspace/xray-lang/xray/src/coro/xdeep_copy.c:247-272` **仍在新建**。

**重构策略**：
1. 梳理 XrContext 现有来源（目前仅 deep_copy）
2. deep_copy 时遍历源 Context 每个 slot，分别克隆为独立 Cell，挂到目标 closure upvalues
3. 删除 `XrContext` 结构 + `XR_TCONTEXT` 枚举 + `deep_copy_context_chain` + GC mark/free 分派
4. 验证：`grep 'XrContext\|XR_TCONTEXT' src/` 为空

**工作清单**：
```bash
grep -rn 'XrContext\|XR_TCONTEXT\|xr_context_' src/ | grep -v '\.md:'
```

### 2.4 `XrProto` 字段审计 + 清理未使用

审计清单（grep < 3 处调用可删）：
- `raw_constants`（与 `constants` 是否重复）
- `bb_leaders` / `loop_headers`（旧 analyzer 产物）
- `return_type_info`（是否被 `XrType *return_type` 完全替代）

### 验证
- `/build` + `/test` + `/build-asan` + `/test`
- 文件数：`ls src/runtime/{object,value,closure} | wc -l` -3~5
- 新增 `tests/unit/test_deep_copy_closure_captures.c`（覆盖单层/嵌套 3 层/协程 go + 闭包读写）

### 风险
- 2.3 deep_copy 改造最大；必须带完整测试。2.1 / 2.2 近零风险。

### 回滚
2.1 / 2.2 / 2.3 / 2.4 各独立 commit。

---

## 阶段 3：文件作用域可变全局清零

### 3.1 `builtin_hash` → 编译期 const 表

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/symbol/xsymbol_table.c:217-242`

违反"禁止文件作用域可变全局"+ 懒初始化 data race → UB。

**最佳设计**：生成脚本 `scripts/gen_builtin_hash.py` 读 `xr_builtin_symbol_names[]` → 计算 FNV-1a → 输出 `xbuiltin_hash.inc`（const table in `.rodata`）。

```c
// xsymbol_table.c
#include "xbuiltin_hash.inc"  // const XrBuiltinHashEntry xr_builtin_hash_table[256]

SymbolId xr_builtin_symbol_from_name(const char *name) {
    uint32_t slot = builtin_hash_fn(name) & (BUILTIN_HASH_SIZE - 1);
    while (xr_builtin_hash_table[slot].name != NULL) {
        if (strcmp(xr_builtin_hash_table[slot].name, name) == 0)
            return xr_builtin_hash_table[slot].id;
        slot = (slot + 1) & (BUILTIN_HASH_SIZE - 1);
    }
    return SYMBOL_INVALID;
}
```

优点：零运行时初始化 + 多线程完全安全 + `.rodata` 只读。新增 builtin symbol 时重跑脚本。

### 3.2 `shape_registry` → `XrayIsolate` 字段

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xshape.c:36-79`（自承"process-global, NOT per-isolate, data race"）

**重构**：
1. `XrayIsolate` 新增：
   ```c
   struct XrShapeRegistry {
       XrShape **entries;
       uint16_t count;
       uint16_t capacity;
       pthread_mutex_t mutex;  // 慢路径，mutex 即可
   };
   ```
2. API 签名统一加 isolate（**不保留兼容层**）：
   ```c
   XR_FUNC XrShape *xr_shape_get_by_id(XrayIsolate *X, uint16_t id);
   XR_FUNC XrShape *xr_shape_new(XrayIsolate *X, uint16_t capacity);  // 已有 X 参数
   ```
3. `xr_json_shape(XrJson*)` 改签名：`xr_json_shape(XrayIsolate *X, XrJson *json)`。不用给 GC header 加 isolate（8B/对象代价过高），**用 `xr_isolate_current()`（TLS）**在 VM 执行上下文内隐式取。
4. shape_id 仍 14 位；**每 isolate 独立命名空间**。

**影响范围评估**：`grep -rn 'xr_shape_get_by_id\|xr_json_shape\|xr_shape_new\b' src/ stdlib/ | wc -l`（预计 30-50 点）。

### 3.3 `g_current_pool` (TLS) → `XrayIsolate::type_pool`

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xtype.c:82`：`static __thread XrTypePool *g_current_pool = NULL;`

`__thread` 仍违反硬线；多 analyzer 嵌套 / 协程迁移时易污染。

**重构**：
1. 删除 TLS；`XrayIsolate.type_pool`
2. 所有 `xr_type_new_*` 加 `XrayIsolate *X` 首参：
   ```c
   XR_FUNC XrType *xr_type_new_int(XrayIsolate *X);
   XR_FUNC XrType *xr_type_new_array(XrayIsolate *X, XrType *element);
   // ~30 个 xr_type_new_* 全部加 X 参数
   ```
3. 只读单例 `g_type_int` / `g_type_float` / `g_type_string` / `g_type_bool` / `g_type_json` 保留（read-only 全局允许）
4. 生命周期：`xr_isolate_create` → pool_create；`xr_isolate_destroy` → pool_destroy

**调用方改造**：编译错误驱动（修签名 → 编译报错列所有调用点）。

### 3.4 通用扫描
```bash
grep -rnE '^static\s+(_Atomic\s+)?[A-Za-z]' src/runtime/{closure,symbol,value,object} \
  | grep -v ' const ' | grep -vE 'static inline|static void|static int|static bool'
```
剩余匹配逐项评估：只读 → 加 `const`；可变 → 挂 isolate。

### 验证
- `/build` + `/test` + `/build-asan` + `/test`
- `grep -n 'static.*builtin_hash' src/runtime/symbol/*.c` 空
- `grep -n 'static XrShape \*\*' src/runtime/object/*.c` 空
- `grep -rn '__thread.*XrTypePool\|g_current_pool' src/` 空
- 新增 `tests/unit/test_multi_isolate_shape.c` + `test_multi_isolate_type_pool.c`（双线程独立 isolate 各自 parse + analyze）

### 风险
**本阶段影响面最大**。对策：
- 一次 commit 改一个字段（3.1 / 3.2 / 3.3 分别一 commit）
- 编译错误驱动 + `/test` 全量通过后才 push

### 回滚
3.1 / 3.2 / 3.3 各独立 commit；3.2 / 3.3 是 breaking change，合入前充分 review。

---

## 阶段 4：SymbolId / XrProto 瘦身

### 4.1 `SymbolId: int32_t → uint16_t`

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/symbol/xsymbol_table.h:32`

**收益**：
- `XrProto.symbols[]` 内存减半（每 proto 约 -1 KB）
- `XrShape.field_symbols[]` 减半
- cache line 友好性提升

**重构**：
1. `typedef uint16_t SymbolId; #define SYMBOL_INVALID ((SymbolId)UINT16_MAX)`
2. printf `%d` → `%u`；显式 `(int32_t)` 转换清查
3. 序列化 chunk (`xchunk_serialize.c`) 若固定 int32 — **保持磁盘格式 32-bit**，运行时降 16-bit，load 时做 uint16 overflow check
4. 编译期断言：`_Static_assert(sizeof(SymbolId) == 2, "...");`

**扫描**：`grep -rn 'SymbolId\b\|int32_t.*[Ss]ymbol' src/ stdlib/`

### 4.2 `XrProto` JIT 状态外置为 `XrProtoJitState *jit`

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xchunk.h:496-629`

**现状**：~18 个 JIT 字段混在 XrProto，冷函数白占 ~240 字节。

**重构**：

1. 新建 `src/runtime/value/xproto_jit.h`：
   ```c
   typedef struct XrProtoJitState {
       void *jit_entry;
       void *jit_fast_entry;
       void *jit_resume_entry;
       _Atomic(void *) jit_entry_pending;
       void *osr_entries;
       uint32_t nosr;
       void *deopt_table;
       uint32_t ndeopt;
       void *stack_map;
       XrBlueprint *blueprint;
       XrTypeFeedback *type_feedback;
       _Atomic uint32_t call_count;
       _Atomic uint32_t exec_count;
       uint8_t deopt_count;
       uint8_t jit_opt_level;
       uint32_t deopt_backoff;
       uint32_t deopt_reset_at;
       bool osr_pending;
   } XrProtoJitState;

   XR_FUNC XrProtoJitState *xr_proto_jit_state_ensure(XrProto *proto);
   XR_FUNC void xr_proto_jit_state_destroy(XrProtoJitState *jit);
   ```

2. `XrProto` 只留 `_Atomic(XrProtoJitState *) jit;`

3. **lazy 分配 + CAS 防并发**：
   ```c
   XrProtoJitState *xr_proto_jit_state_ensure(XrProto *proto) {
       XrProtoJitState *jit = atomic_load_explicit(&proto->jit, memory_order_acquire);
       if (jit) return jit;
       XrProtoJitState *new_jit = xr_calloc(1, sizeof(XrProtoJitState));
       XrProtoJitState *expected = NULL;
       if (atomic_compare_exchange_strong_explicit(
               &proto->jit, &expected, new_jit,
               memory_order_acq_rel, memory_order_acquire)) {
           return new_jit;
       }
       xr_free(new_jit);
       return expected;
   }
   ```

4. 访问点：`proto->jit_entry` → inline helper `xr_proto_get_jit_entry(proto)`（NULL 安全）

5. `xr_vm_proto_free` 调 `xr_proto_jit_state_destroy(proto->jit)`

### 4.3 `XrProto` 字段访问集中为 inline getter

对所有被 JIT 字段外置的访问点，用 `static inline` helper：
```c
static inline void *xr_proto_jit_entry(XrProto *p) {
    XrProtoJitState *jit = atomic_load_explicit(&p->jit, memory_order_acquire);
    return jit ? jit->jit_entry : NULL;
}
static inline uint32_t xr_proto_call_count(XrProto *p) {
    XrProtoJitState *jit = atomic_load_explicit(&p->jit, memory_order_acquire);
    return jit ? atomic_load_explicit(&jit->call_count, memory_order_relaxed) : 0;
}
```

### 验证
- `/build` + `/test`
- 内存对比：`/usr/bin/time -l ./build/bin/xray bench_json_create.xr`（期望 maximum RSS -几 MB）
- 编译时间：`time cmake --build build -- -j8` 与拆分前对比
- 新增 `tests/unit/test_proto_jit_lazy.c`：冷 proto `proto->jit == NULL`；热 proto 调用后 non-NULL

### 风险
- 4.1 如果序列化格式写 int32，必须保持向后兼容的磁盘格式（内存窄/磁盘宽）
- 4.2 atomic 乱序：`_Atomic` + `acquire/release` 必须到位

### 回滚
4.1 / 4.2 独立 commit；4.2 可再拆 "引入空 XrProtoJitState → 逐字段迁移 → 删原字段" 3 子 commit。

---

## 阶段 5：`xtype.h` 拆分 + inline 下沉

### 5.1 现状

`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xtype.h` 610 行，接近 800 硬线；60+ 个 `XR_FUNC`；大量 inline 污染全项目 include 链。

### 5.2 目标布局

```
src/runtime/value/
├── xtype.h            ~300 行：XrType 结构 + 核心 new/copy/destroy API
├── xtype_check.h      ~150 行：xr_type_equals / assignable / kind_is_* inline
├── xtype_rep.h        ~120 行：rep / slot_type / xr_tag / gc_tag 映射 (inline)
├── xtype_format.h     （已存在）
├── xtype_feedback.h   （已存在，重命名成员见 5.4）
├── xtype.c            ~400 行：全局 init / nullable / generic helpers
├── xtype_ctor.c       新 ~400 行：所有 xr_type_new_*
├── xtype_union.c      新 ~250 行：union 规范化与合并
├── xtype_check.c      新 ~300 行：equals / assignable / subtype 运算
└── xtype_copy.c       新 ~100 行：xr_type_copy
```

### 5.3 拆分步骤

1. **inline 识别**：body ≤ 10 行 + 纯计算 → 保头文件；否则下沉 .c 改 `XR_FUNC`
2. **`xtype_check.h`**：`xr_kind_is_primitive / container / numeric`、`xr_type_is_any / never`、`XR_TYPE_IS_*` 宏
3. **`xtype_rep.h`**：`xr_type_rep / xr_type_to_slot_type / xr_type_to_xr_tag / xr_type_element_gc_tag / xr_is_json_coercion`
4. **`xtype.h`** 瘦身：只留 XrType 结构 + XrTypeKind 枚举 + `xr_type_new_*` 声明 + `xr_type_copy/destroy/equals/assignable` 声明
5. **`xtype.c` 拆分**（按职能）：
   - `xtype_ctor.c` ← xtype.c:104-410
   - `xtype_union.c` ← xtype.c:423-680
   - `xtype_copy.c` ← xtype.c:692-792
   - `xtype_check.c` ← xtype.c:833-~1100
6. **include 传播修正**：按需 include 子头，不需要 new_* 的位置改 include `xtype_check.h` / `xtype_rep.h`

### 5.4 `XirTypeFeedback` → `XrTypeFeedback`

`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xtype_feedback.h:61` `Xir` 是 VM IR 历史前缀，文件已声明"runtime shared"。

```bash
sed -i '' 's/XirTypeFeedback/XrTypeFeedback/g' $(grep -rl 'XirTypeFeedback' src/ stdlib/)
sed -i '' 's/xir_type_feedback/xr_type_feedback/g' $(grep -rl 'xir_type_feedback' src/ stdlib/)
```

### 5.5 `xchunk.h` 同步瘦身

阶段 4.2 已拆 `XrProtoJitState`；本阶段进一步：
- VM 执行结构移到 `xexec_frame.h` / `xexec_state.h`
- `xchunk.h` 只留 `XrProto` + `xr_vm_proto_new/free` 声明，≤ 400 行

### 验证
- `/build` + `/test`
- `wc -l src/runtime/value/*.h | awk '$1 > 800'` 空
- `wc -l src/runtime/value/xtype.h` ≤ 350
- 触发 `xtype_check.h` 改动的重编范围对比（应只重编 analyzer 少量 TU）

### 风险
- 循环依赖：先画 DAG（`xforward_decl.h → xtype.h → xtype_check.h/xtype_rep.h → xtype_format.h → xtype_feedback.h`）
- inline → `XR_FUNC` 后 hot path 可能退化几 ns；`-O2 + LTO` 通常解决，退化 > 5% 的点保留 inline

### 回滚
5.1-5.5 各独立 commit；5.4 机械替换可单独回滚。

---

## 阶段 6：Value 层一致性

### 6.1 `xr_value_deep_eq` 覆盖所有值语义容器

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xvalue.c:88-111`

**现状**：只对 string/json/array/map 深比较，BigInt/Set/DateTime/Range 走指针比较 → **`BigInt(1) == BigInt(1)` 返回 false**（严重语义 bug）。

**重构**：

```c
bool xr_value_deep_eq(XrValue a, XrValue b) {
    if (a.tag != b.tag) return false;
    if (a.tag != XR_TAG_PTR) { /* ... 现有标量处理 ... */ }
    if (a.i == b.i) return true;
    uint8_t ta = XR_HEAP_TYPE(a);
    if (ta != XR_HEAP_TYPE(b)) return false;
    switch (ta) {
    case XR_TSTRING:   return xr_string_equal(XR_TO_STRING(a), XR_TO_STRING(b));
    case XR_TJSON:     return xr_json_equals_deep(XR_TO_JSON(a), XR_TO_JSON(b));
    case XR_TARRAY:    return xr_array_equals_deep(XR_TO_ARRAY(a), XR_TO_ARRAY(b));
    case XR_TMAP:      return xr_map_equals_deep(XR_TO_MAP(a), XR_TO_MAP(b));
    case XR_TSET:      return xr_set_equals_deep(XR_TO_SET(a), XR_TO_SET(b));       // 新增
    case XR_TBIGINT:   return xr_bigint_cmp(XR_TO_BIGINT(a), XR_TO_BIGINT(b)) == 0; // 新增
    case XR_TDATETIME: return xr_datetime_equals(XR_TO_DATETIME(a), XR_TO_DATETIME(b)); // 新增
    case XR_TRANGE: {
        XrRange *ra = xr_value_to_range(a), *rb = xr_value_to_range(b);
        return ra->start == rb->start && ra->end == rb->end && ra->step == rb->step;
    }
    /* 按引用语义的保持指针比较（已返回 false） */
    case XR_TCLOSURE:
    case XR_TBOUND_METHOD:
    case XR_TCHANNEL:
    case XR_TTASK:
    case XR_TCOROPOOL:
    case XR_TEXCEPTION:
    case XR_TREGEX:
    default: return false;
    }
}
```

**新函数**：
- `xr_set_equals_deep`：遍历 a 每元素 `xr_set_has(b)`，再比 size
- `xr_datetime_equals`：比 `time_t` / `tv_usec`

### 6.2 `tag_to_typeid` 占位 sentinel 化

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xvalue.c:117-125`

`XR_TAG_PTR / STRUCT_REF / NOTFOUND` 都占位 `XR_TID_NULL`，与真正 NULL 混淆。

**修复**：
```c
#define XR_TID_NEEDS_HEAP_TYPE  ((XrTypeId)0xFE)
#define XR_TID_NOTFOUND         ((XrTypeId)0xFF)

static const XrTypeId tag_to_typeid[8] = {
    [XR_TAG_NULL]       = XR_TID_NULL,
    [XR_TAG_TRUE]       = XR_TID_BOOL,
    [XR_TAG_FALSE]      = XR_TID_BOOL,
    [XR_TAG_I64]        = XR_TID_INT,
    [XR_TAG_F64]        = XR_TID_FLOAT,
    [XR_TAG_PTR]        = XR_TID_NEEDS_HEAP_TYPE,
    [XR_TAG_STRUCT_REF] = XR_TID_NEEDS_HEAP_TYPE,
    [XR_TAG_NOTFOUND]   = XR_TID_NOTFOUND,
};

XrTypeId xr_value_type_id(XrValue v) {
    XrTypeId tid = tag_to_typeid[v.tag];
    if (tid == XR_TID_NEEDS_HEAP_TYPE)
        return gctype_to_typeid[XR_HEAP_TYPE(v)];
    return tid;
}
```

### 6.3 `bound_method` 统一宏 + 支持 coro 堆

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/closure/xbound_method.c`

**重构**：
1. 签名改 `xr_bound_method_new(struct XrCoroutine *coro, XrValue receiver, XrValue method)`；优先 `xr_coro_gc_newobj`，coro NULL 才 fallback `isolate->gc`
2. 删 `xbound_method.c` 手写的 `xr_value_from/is/to_bound_method`，改在 `xvalue.c` 末尾：
   ```c
   DEFINE_VALUE_OPS_WITH_TYPE(bound_method, XR_TBOUND_METHOD, XrBoundMethod)
   ```
3. 所有调用点迁移

### 6.4 `xr_value_to_json` 补类型检查

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xjson.h:128-130`

```c
static inline XrJson* xr_value_to_json(XrValue v) {
    return xr_value_is_json(v) ? (XrJson*)XR_TO_PTR(v) : NULL;
}
// 若有无检查需求，另提供 xr_value_to_json_unchecked
```

### 6.5 `xhashmap` key 所有权核查

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/symbol/xsymbol_table.c:181-194`

读 xhashmap 实现确认 set 是拷贝 key 还是存指针。
- 若拷贝：destroy 时必须一次只释放（当前 `id_to_name[i]` 和 hashmap 内部各一份 → **内存泄漏**）
- 若存指针：只能由 id_to_name 释放，**正确**

根据核查结果修正 destroy 路径或改用 `xr_strdup` 避免歧义。

### 验证
- `/build` + `/test`
- 新增 `tests/unit/test_value_deep_eq_all_types.c`：BigInt / Set / DateTime / Range 跨对象相等应返回 true
- `tests/unit/test_bound_method_coro_gc.c`：协程内创建 bound_method，协程 GC 后可回收

### 风险
- 6.3 bound_method 签名变更影响所有 `xr_bound_method_new(` 调用点，数量较少（grep 确认 < 10 处）

### 回滚
6.1-6.5 各独立 commit。

---

## 阶段 7：Shape 架构重塑

**前置**：阶段 3.2 完成（`shape_registry` 已迁 isolate）。本阶段收口 Shape 的 GC 所有权、稀疏映射、transition 查找性能。

### 7.1 Shape GC 所有权统一

**现状**：`xshape_new` 用 `xr_malloc` 分配（**不在 GC 堆**），但结构首字段是 `XrGCHeader gc`，且 `xr_gc_traverse_shape` 把 `shape->parent` / `shape->transitions[i].target` 当作 GC 对象 mark（`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xshape.c:310-327`）。

**问题**：
- Shape 的 `gc` 字段 calloc 留 `type=0 (XR_TNULL)`，从未 init
- Shape 生命周期绑定 isolate（不走 GC 收集），但又被 GC mark

**决策（最佳设计）**：

**方案 A（推荐）**：Shape 永远是 isolate 级元对象，**完全脱离 GC**：
1. 删除 `XrGCHeader gc` 字段（不再首字段）
2. 删除 `xr_gc_traverse_shape`
3. GC 对 XR_TJSON 等对象 mark 时，通过 json->shape_id 从 isolate shape_registry 查 Shape 结构，但**不 mark Shape 本身**（因为它不在 GC 堆）
4. Shape 生命周期：在 `xr_isolate_destroy` 中由 registry 统一释放

**方案 B（保留现状最小改动）**：Shape 真正进 GC 堆：
- `xr_shape_new` 改用 `xr_gc_newobj(isolate->gc, XR_TSHAPE, size)`
- 正确 init GC header
- GC 回收 Shape 时同时清 registry 槽

**推荐方案 A**：Shape 是类型元数据（类似 class），概念上不是"值"，不该在 GC 堆；且 registry 已是 isolate 字段，天然与 isolate 共生共死。

### 7.2 稀疏 `symbol_to_index` 降级为线性扫描

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xshape.c:102-110`

当 `max_symbol - min_symbol` 大（稀疏），bitmap 表内存浪费严重；field_count 小（≤8）时线性扫描反而快。

**重构**：
```c
if (max_sym - min_sym > 64 && field_count <= 8) {
    shape->symbol_to_index = NULL;  // 不分配，走线性扫描
    shape->use_linear_scan = true;
} else {
    // 原 bitmap 分配路径
}

// xr_shape_field_index
int xr_shape_field_index(XrShape *shape, SymbolId sym) {
    if (shape->use_linear_scan) {
        for (int i = 0; i < shape->field_count; i++)
            if (shape->field_symbols[i] == sym) return i;
        return -1;
    }
    // 原 bitmap lookup
}
```

### 7.3 Transition table 改小哈希

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xshape.c:201-210`

当前 O(N) 线性扫描 transitions；N ≥ 8 时改小哈希（4-bit 分桶，16 bucket）。

**重构**：
```c
typedef struct XrShapeTransitionTable {
    uint16_t count;
    uint16_t capacity;
    struct XrShapeTransition *entries;
    // 当 count >= 8 时启用：
    uint8_t *hash_buckets;  // 16 buckets, 每个存 entries 索引的首节点
    uint8_t *hash_next;     // 冲突链
} XrShapeTransitionTable;

XrShape *xr_shape_transition_find(XrShape *shape, SymbolId sym) {
    XrShapeTransitionTable *t = &shape->transitions;
    if (!t->hash_buckets) {
        // 线性扫（count < 8）
        for (uint16_t i = 0; i < t->count; i++)
            if (t->entries[i].symbol == sym) return t->entries[i].target;
    } else {
        uint8_t slot = sym & 0xF;
        uint8_t idx = t->hash_buckets[slot];
        while (idx != 0xFF) {
            if (t->entries[idx].symbol == sym) return t->entries[idx].target;
            idx = t->hash_next[idx];
        }
    }
    return NULL;
}
```

### 7.4 Shape 序列化（可选）

Shape registry 迁 isolate 后，若需 chunk 跨进程携带（AOT 编译产物），需要：
- Shape → 字节流（含 field_symbols + parent_id）
- Load 时在目标 isolate 重建

**不在本次重构范围**，记录为 TODO。

### 验证
- `/build` + `/test`
- 新增 `bench/shape_sparse_symbols.xr`（大 symbol ID + 少 field）对比 7.2 前后内存
- `tests/unit/test_shape_transition_large.c`（单 shape 100 个 transitions，O(1) 查找正确）

### 风险
- 7.1 方案 A 破坏 GC mark 假设（Shape 从不被 mark）；确认所有 GC 遍历代码都不再引用 Shape 的 XrGCHeader

### 回滚
7.1 / 7.2 / 7.3 各独立 commit。

---

## 阶段 8：大文件拆分 + 其它性能优化

### 8.1 `xstring.c` 按职能拆分

**现状**：1452 行，接近模块瓶颈。

**目标拆分**：
```
src/runtime/object/
├── xstring.h            （已存在）
├── xstring_core.c       ~500 行：创建、intern、比较、hash
├── xstring_methods.c    ~500 行：slice / split / replace / translate / find
├── xstring_char.c       ~300 行：UTF-8 处理、charcodeAt、char class
└── xstring_class_init.c （已存在，保持）
```

### 8.2 `xbigint.c` 拆分

**现状**：1317 行。

**目标**：
```
├── xbigint_core.c       ~400 行：结构、new、copy、cmp、utility
├── xbigint_arith.c      ~500 行：add / sub / mul / div / mod / pow
├── xbigint_bitops.c     ~200 行：and / or / xor / shl / shr / bit_length
└── xbigint_format.c     ~250 行：to_string / from_string / to_double
```

### 8.3 `xtype.c` 拆分

阶段 5.3 已完成（`xtype_ctor.c / xtype_union.c / xtype_copy.c / xtype_check.c`），本阶段仅收尾验证行数。

### 8.4 BigInt limb 从 32-bit 升级到 64-bit（可选性能优化）

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xbigint.h:44-52`

现代 64-bit 平台 `uint64_t limb + __uint128_t` 乘积可减半 limb 数，mul 性能提升 30-50%。

**重构**：
1. `limbs[]` 类型从 `uint32_t` → `uint64_t`
2. 所有 arith 算法改 64×64→128 乘积（`__uint128_t` 中间结果）
3. `from_string` / `to_string` 基数仍用 10 的 19 次方（64-bit 能容纳）
4. 序列化格式变更 → 按 "不兼容" 原则直接改（无外部用户）

**前置条件**：`__uint128_t` 在 GCC / Clang 上稳定，MSVC 需替代方案（分 hi/lo）。可先 `#if defined(__GNUC__) || defined(__clang__)` 分支。

### 8.5 `xjson_pool` profile + 决定去留

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xjson_pool.c`

**步骤**：
1. 加 `XR_DEBUG_JSON_POOL` 宏（仅 debug 构建启用）打印 pool stats
2. 跑 `bench/json_create_1M.xr`，看 hits / misses / overflows
3. 决策：
   - 命中率 > 60% → 保留
   - 命中率 < 30% → 删除整个 `xjson_pool.{c,h}`，依赖 Immix 短命对象路径（代码 -5.7 KB）

### 8.6 Array iterator 支持（可选扩展）

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/object/xiterator.h`

`XrIteratorType` 当前只列 MAP/SET/JSON。若要让 Array 支持 `Iterable<T>` 协议（通过 for-in 而非索引）：
1. 新增 `XR_ITERATOR_ARRAY`
2. `XrIterator` 扩展 array-specific 字段（data ptr + length snapshot）
3. `xr_array_iter_next` 实现

**不急迫**：现有 `for i in 0..arr.length` 路径已覆盖；扩展仅为泛型 Iterable 协议。

### 8.7 XrCell 小对象池（性能优化）

**位置**：`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/closure/xcell.c`

Cell 固定 32 字节，在 capture-heavy closure 里频繁分配。可做：
- Size-class freelist（per-coroutine）
- 或直接走 Immix 的 line-alloc fast path（若 GC 已支持）

**先 profile**（`bench/closure_capture_heavy.xr`）看 Cell 分配占比，再决定是否优化。

### 验证
- 所有 `.c` ≤ 800 行：`wc -l src/runtime/object/*.c src/runtime/value/*.c | awk '$1 > 800'` 应 ≤ 1 条（仅 xstring/xbigint/xtype 拆完）
- `bench/bigint_mul_256` 对比 8.4 前后（期望 +30%）
- `bench/json_create_1M` 对比 8.5 决策前后

### 风险
- 8.4 BigInt 算法改写风险高；必须带算法级单元测试（特别边界：0、负数、limb 溢出、Karatsuba 阈值）
- 8.1 / 8.2 拆分 header include 传播需谨慎

### 回滚
每个子项独立 commit。8.4 最大改动可进一步拆为 "引入 uint64_t 类型别名 → 迁 add/sub → 迁 mul → 迁 div → 迁 bitops" 5 子 commit。

---

## 阶段尾：一致性收口

每阶段结束固定动作：
1. `/test`（快速）+ `scripts/run_regression_tests.sh`（完整）
2. `/build-asan` + `/test`
3. 检查硬线指标：
   ```bash
   wc -l src/runtime/{closure,symbol,value,object}/*.c | awk '$1 > 3000'  # 空
   wc -l src/runtime/{closure,symbol,value,object}/*.h | awk '$1 > 800'   # 空
   ```
4. 函数行数检查：脚本列出 ≥150 行函数应为空
5. commit 规范：每阶段 1-N commit，message 英文（见 `main.md`）
6. 更新本文档：勾选复选框、记录 KPI

## KPI 基线记录表（待填）

| 基准测试 | 阶段 0 前 | 阶段 3 后 | 阶段 5 后 | 阶段 8 后 |
|----------|-----------|-----------|-----------|-----------|
| `bench/array_push_1M` ms | _ | _ | _ | _ |
| `bench/json_create_1M` ms | _ | _ | _ | _ |
| `bench/type_check_dense` ms | _ | _ | _ | _ |
| `bench/bigint_mul_256` μs | _ | _ | _ | _ |
| 最大 `.c` 行数 | 1452 | _ | _ | _ |
| 最大 `.h` 行数 | 671 | _ | _ | _ |
| XrProto sizeof | _ | _ | _ | _ |

## 执行进度追踪（复选框）

### 阶段 0：杂项清理
- [ ] 统一 include 路径（4 处）
- [ ] 修正 `xvalue.h:61` `#endif` 注释
- [ ] 删除 `xslice_new.c`
- [ ] 清除冗余 `(void) xxx;` 语句
- [ ] 删除 `xexception.h` 重复 `XR_IS_EXCEPTION` 定义
- [ ] `XR_HASH_NULL` 魔数具名

### 阶段 1：P0 bug 修复
- [ ] `xr_value_from_range` 改 `XR_FROM_PTR`
- [ ] `symbol_to_index` int8 → int16
- [ ] `xr_array_set_element` 非数值防御
- [ ] `xr_vm_proto_new` 删冗余 manual 赋值
- [ ] `xr_proto_add_symbol` 超限 panic

### 阶段 2：死代码清扫
- [ ] 删除 `xnative_type_registry.{h,c}` + isolate hook
- [ ] 删除 `xruntime_type_info.{c,h}`
- [ ] `XrContext` 迁移到 Cell + 删除
- [ ] `XrProto` 字段审计 + 清理未使用

### 阶段 3：全局变量清零
- [ ] `builtin_hash` → 编译期 const 表（gen_builtin_hash.py）
- [x] `shape_registry` → `XrayIsolate::shape_registry`
- [x] `g_current_pool` → `XrayIsolate::type_pool`（`g_fallback_pool` 已删除）
- [x] `xr_type_new_*` 全部加 `XrayIsolate *X` 首参，561+ 调用点更新
- [x] `xa_analyzer_new(XrayIsolate *X)` 显式接收 isolate
- [x] `s_json_value_type` → `X->json_value_type`（per-isolate 缓存）
- [x] `g_types_initialized` → `_Atomic bool` + CAS（线程安全）
- [x] `test_analyzer` 创建真实 `XrayIsolate`
- [ ] 通用扫描 static 可变全局清零

### 阶段 4：SymbolId / XrProto 瘦身
- [ ] `SymbolId` int32 → uint16
- [ ] 引入 `XrProtoJitState` 并拆 18 个 JIT 字段
- [ ] JIT 字段访问改 inline helper

### 阶段 5：`xtype.h` 拆分
- [ ] 拆 `xtype_check.h`
- [ ] 拆 `xtype_rep.h`
- [ ] 拆 `xtype_ctor.c` / `xtype_union.c` / `xtype_copy.c` / `xtype_check.c`
- [ ] `XirTypeFeedback` → `XrTypeFeedback`
- [ ] `xchunk.h` 瘦身 ≤ 400 行

### 阶段 6：Value 层一致性
- [ ] `xr_value_deep_eq` 覆盖 Set / BigInt / DateTime / Range
- [ ] `tag_to_typeid` 占位 sentinel 化
- [ ] `bound_method` 统一宏 + 支持 coro 堆
- [ ] `xr_value_to_json` 补类型检查
- [ ] `xhashmap` key 所有权核查 + 修正

### 阶段 7：Shape 架构重塑
- [ ] Shape 脱离 GC（方案 A）
- [ ] 稀疏 `symbol_to_index` 降级线性扫描
- [ ] Transition table N ≥ 8 改小哈希

### 阶段 8：大文件拆分 + 其它
- [ ] `xstring.c` → core / methods / char 拆分
- [ ] `xbigint.c` → core / arith / bitops / format 拆分
- [ ] BigInt limb 32 → 64 bit 升级
- [ ] `xjson_pool` profile + 决定去留
- [ ] XrCell 小对象池（按 profile 决策）

## 附录 A：每阶段独立 PR 模板

```
refactor(runtime): [Phase N] <one-line summary>

Phase N of docs/tasks/002-runtime-refactor.md.

Changes:
- ...
- ...

Metrics:
- <benchmark>: before X -> after Y (+Z%)
- wc -l: max <LINE>
- XrProto sizeof: before X -> after Y

Tests:
- ctest: all pass
- ASan: no findings
- Regression: scripts/run_regression_tests.sh PASS
```

## 附录 B：风险缓解策略

1. **每阶段一个 feature branch**：`refactor/runtime-phase-N`，merge 前 rebase on main
2. **多 isolate 测试优先**：阶段 3 的改动必须配对应的 multi_isolate 单元测试
3. **ASan 必跑**：阶段 1 / 3 / 6 强制（heap_type UB、atomic、deep_eq）
4. **TSan 补充**（如环境可用）：阶段 3 合入前跑 TSan 全量测试
5. **Rollback plan**：每阶段独立 revertable；阶段 3、5 最关键，合入后观察 3 天无异常再进入下一阶段

## 附录 C：发现问题的完整清单（便于 review 对照）

按优先级分类，每项对应到上面阶段的章节号。

### P0（阻塞 / bug）
- `xr_value_from_range` 未初始化 heap_type → **1.1**
- `XrNativeTypeInfo` 双重定义 → **2.1**
- 全局 `shape_registry` 数据竞争 → **3.2**
- 全局 `builtin_hash` 懒初始化 data race → **3.1**

### P1（架构债）
- `g_current_pool` TLS 全局 → **3.3**
- `xtype.h` 过大（610 行） → **5.2**
- `XrProto` 臃肿（JIT 字段混合） → **4.2**
- `symbol_to_index` int8 溢出 → **1.2**
- XrShape GC 交互不一致 → **7.1**
- `XrContext` 注释与现实冲突 → **2.3**

### P2（清理 / 性能）
- `SymbolId` 4 字节过大 → **4.1**
- `xvalue.c deep_eq` 不全 → **6.1**
- `xchunk.c` 冗余 manual 赋值 → **1.4**
- `xexception.h` 重复 `XR_IS_EXCEPTION` → **0.5**
- include 路径错误（4 处） → **0.1**
- `xr_value_to_json` 缺类型检查 → **6.4**
- `xr_array_set_element` 非数值 UB → **1.3**
- `bound_method` 风格割裂 → **6.3**
- `xruntime_type_info.c` 死代码 → **2.2**
- Shape transition 线性扫描 → **7.3**
- `xjson_pool` 使用存疑 → **8.5**
- 大文件拆分（`xstring.c` / `xbigint.c` / `xtype.c`） → **8.1-8.3**

### P3（风格 / 命名）
- `XirTypeFeedback` → `XrTypeFeedback` → **5.4**
- `xvalue.h:61` `#endif` 错注释 → **0.2**
- `xvalue_hash.c` null hash 魔数 6 → **0.6**
- `xslice_new.c` 空文件 → **0.3**
- `(void)old_size;` 等冗余语句 → **0.4**
- BigInt 32-bit limb 升级 → **8.4**
