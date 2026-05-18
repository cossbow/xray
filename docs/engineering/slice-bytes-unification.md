# Slice / Bytes 统一重构方案

## 问题总结

### 问题 A：Slice 是幽灵类型
- **运行时** `XR_TARRAY_SLICE` 是独立 GC 类型，有独立方法表 (`xslice_builtins.c`, 503 行)
- **分析器**完全不知道 slice 存在——没有 `XR_KIND_SLICE`，slice 表达式推断为源类型 (`Array<T>`)
- **类型注解**无法表达 slice：`fn foo(s: ???)` — 不可能写
- **方法不一致**：分析器认为 slice 有 push/pop（Array 方法），运行时只有 get/set/forEach/map 等

### 问题 B：Bytes 和 Array<uint8> 的身份分裂
- **分析器**：`XR_KIND_BYTES`（独立 kind）
- **运行时**：`XrArray { elem_type=XR_ELEM_U8 }`（就是 Array<uint8>）
- **结果**：每处都要并列处理 `XR_KIND_ARRAY || XR_KIND_BYTES`，遗漏就出 bug
  - 例：`AST_SLICE_EXPR` 只检查 `XR_TYPE_IS_ARRAY(src)`，不检查 `XR_KIND_BYTES`

---

## 设计决策

### 决策 1：消除 `XR_TARRAY_SLICE` — Slice 不是独立类型

**原则**：Slice 是 Array 的一个状态（view），不是独立类型。

**类比**：
- Go: `[]int` 就是 slice，没有 "array" vs "slice" 两种类型
- Rust: `&[T]` 是引用，底层还是 slice
- Xray: `arr[1:3]` 返回的就是 `Array<T>`，只是 capacity=0、source!=NULL

**具体做法**：
- `xr_array_slice()` 分配时用 `XR_TARRAY`（不再用 `XR_TARRAY_SLICE`）
- 判断是否 slice 统一用 `xr_array_is_slice(arr)` = `capacity==0 && source!=NULL`
- 删除 `XR_TARRAY_SLICE` GC 标签
- 删除 `xslice_builtins.c` / `xslice_builtins.h`（独立方法表）
- 删除 `xslice.c` / `xslice.h`（独立 value 转换函数）
- Array 方法中已有 slice guard（push/pop/unshift/shift 检查 `xr_array_is_slice`），保持不变
- 用户写 `fn foo(a: Array<int>)` 可以接受原数组和 slice，类型一致

### 决策 2：消除 `XR_KIND_BYTES` — Bytes 是 Array<uint8> 的别名

**原则**：Bytes 在运行时就是 `XrArray{elem_type=XR_ELEM_U8}`，分析器应反映这个事实。

**具体做法**：
- `Bytes` 在分析器层解析为 `XR_KIND_ARRAY` + `elem.tid = XR_TID_UINT8`
- 删除 `XR_KIND_BYTES` 枚举值
- 删除 `xr_type_new_bytes()` 函数
- `xr_type_to_string` 对 Array<uint8> 显示 `"Bytes"`（sugar，可选）
- Prelude 中 `Bytes` 的 kind 改为 `GENERIC_1`（或引入 `ALIAS` kind 映射到 `Array<uint8>`）
- 所有 `XR_KIND_BYTES` 特判消除——走 Array 统一路径

---

## 影响范围（精确到文件和行）

### A. 消除 XR_TARRAY_SLICE（18 个文件，~41 处引用）

| 文件 | 改动 |
|------|------|
| `runtime/gc/xgc_header.h` | 删除 `XR_TARRAY_SLICE` 枚举值 |
| `runtime/gc/xgc.c` | 删除 `[XR_TARRAY_SLICE]` GC 表项 |
| `runtime/gc/xcoro_gc_traverse.c` | 删除 `TARRAY_SLICE` 断言分支 |
| `runtime/object/xarray.c:979` | `xr_alloc(..., XR_TARRAY_SLICE)` → `xr_alloc(..., XR_TARRAY)` |
| `runtime/object/xarray.c` (4处) | 删除 `TARRAY_SLICE` 断言分支，简化 |
| `runtime/object/xslice.c` | **整文件删除** |
| `runtime/object/xslice.h` | **整文件删除** |
| `runtime/object/builtins/xslice_builtins.c` | **整文件删除** (503 行) |
| `runtime/object/builtins/xslice_builtins.h` | **整文件删除** |
| `runtime/class/xclass_system.c` | 删除 `xr_array_slice_register_native_type` 调用 |
| `runtime/class/xclass_system.h` | 删除 `arraySliceClass` 字段 |
| `runtime/class/xreflect_api.c` (2处) | `TARRAY_SLICE` → 统一走 `TARRAY` |
| `runtime/value/xvalue.h` | 删除 `XR_IS_ARRAY_OR_SLICE` 宏 → `XR_IS_ARRAY` 天然匹配 |
| `runtime/value/xvalue.c` | 删除 `[XR_TARRAY_SLICE]` 映射 |
| `runtime/value/xvalue_print.c` (2处) | 删除 `TARRAY_SLICE` case |
| `runtime/value/xvalue_format.c` | 删除 `TARRAY_SLICE` case |
| `vm/xvm_dispatch_collection.inc.c` | `XR_IS_ARRAY_OR_SLICE` → `XR_IS_ARRAY`（已由上一步自然解决）|
| `vm/xvm_cold_object.c` | 删除 xslice.h include |
| `vm/xvm.c`, `xvm_cold_call.c`, `xvm_cold_chan.c`, `xvm_cold_coro.c` | 删除 xslice.h include |
| `coro/xdeep_copy.c` | 删除 `TARRAY_SLICE` case（走 TARRAY） |
| `jit/xm_jit_runtime.c` | 删除 `TARRAY_SLICE` 分支 |
| `app/dap/xdap_variables.c` (3处) | 删除 `TARRAY_SLICE` 特判 |
| `app/dap/xdap_debug.c` | 删除 `TARRAY_SLICE` 引用 |

### B. 消除 XR_KIND_BYTES（8 个文件，~19 处引用）

| 文件 | 改动 |
|------|------|
| `runtime/value/xtype.h` | 删除 `XR_KIND_BYTES`；从 `xr_kind_is_container` 等 inline 函数中删除 |
| `runtime/value/xtype.c` | 删除 `xr_type_new_bytes()`；删除 `Bytes ↔ Array` 兼容判断 |
| `runtime/value/xtype_format.c` | `XR_KIND_BYTES → "Array<uint8>"` 判断删除，用 elem_tid 判断 |
| `runtime/value/xtype_generic.c` (2处) | 删除 `XR_KIND_BYTES` 接口判断 |
| `frontend/analyzer/xanalyzer_builtins.c` (2处) | Bytes → Array<uint8> |
| `frontend/analyzer/xanalyzer_builtin_interfaces.c` | 删除 `is_bytes_type()` |
| `frontend/analyzer/xanalyzer_visitor_expr.c` | `new Bytes()` → `XR_KIND_ARRAY + elem U8` |
| `frontend/analyzer/xanalyzer_visitor.c` | `AST_SLICE_EXPR` 自动覆盖（Bytes 变成 Array） |
| `frontend/analyzer/xtype_ref_resolve.c` | `XR_PRELUDE_KIND_BYTES` → 走 Array<uint8> 路径 |
| `runtime/class/xreflect_method.c` | 删除 `XR_KIND_BYTES` case |
| `stdlib/prelude/prelude.h` | 删除 `XR_PRELUDE_KIND_BYTES` |
| `stdlib/prelude/prelude_types.def` | `Bytes` 改为别名机制 |

---

## 实施步骤（原子提交，每步可测试）

### Step 1：消除 XR_TARRAY_SLICE

1. `xr_array_slice()` 改用 `XR_TARRAY` 分配
2. 删除 `XR_IS_ARRAY_OR_SLICE` 宏，所有调用点改为 `XR_IS_ARRAY`
3. 删除 `xslice.c/h`, `xslice_builtins.c/h` 四个文件
4. 删除 `xclass_system` 中 `arraySliceClass` 注册
5. GC 表、deep_copy、reflect、DAP、JIT runtime 中删除 `TARRAY_SLICE` 分支
6. 最后从 `xgc_header.h` 删除 `XR_TARRAY_SLICE` 枚举值
7. CMakeLists.txt 中移除删掉的 .c 文件
8. 构建 + 全量测试

**预期效果**：~600 行净删除，所有 slice 走 Array 统一路径

### Step 2：消除 XR_KIND_BYTES

1. `Bytes` 在分析器中生成 `Array<uint8>` 类型（`XR_KIND_ARRAY` + `elem_tid=XR_TID_UINT8`）
2. 删除 `xr_type_new_bytes()`
3. 删除所有 `XR_KIND_BYTES` 分支
4. Prelude 注册改为别名机制
5. `xr_type_to_string` 可选 sugar：`Array<uint8>` 显示为 `Bytes`
6. 删除 `XR_KIND_BYTES` 枚举值
7. 构建 + 全量测试

**预期效果**：~30 行净删除，消除所有 Array/Bytes 并列判断

---

## 验证计划

- `ctest --output-on-failure`（全量 ctest）
- `scripts/run_regression_tests.sh`（全量回归）
- 重点验证文件：
  - `tests/regression/14_typed_array/1401_bytes_slice.xr` — Bytes slice 索引
  - 所有 `14_typed_array/` 下的测试
  - `tests/regression/06_builtin_types/` 下 array/slice 相关测试

## 代码净减估算

| 项目 | 删除 | 新增 | 净 |
|------|------|------|-----|
| xslice.c/h | -107 | 0 | -107 |
| xslice_builtins.c/h | -545 | 0 | -545 |
| 其他文件 TARRAY_SLICE 分支 | -80 | +5 | -75 |
| XR_KIND_BYTES 相关 | -40 | +10 | -30 |
| **合计** | **~-770** | **~+15** | **~-755** |

---

# 延伸：Xray 全局设计审计

受 Slice/Bytes 分析启发，用同一个审查模式（**运行时 vs 分析器身份是否一致？是否有并列判断遗漏风险？**）扫描整个类型系统。以下是代码已验证的发现：

---

## D-01：XrObjType 枚举有 3 个僵尸槽位

```
XR_TCLASS_BUILDER_UNUSED  // "Reserved slot, keep enum values stable"
XR_TRESERVED              // 只有 xvalue.c 映射到 TID_NULL
XR_TCONTEXT               // "DEPRECATED: kept for enum stability"
```

**问题**：注释说"保持枚举值稳定"——但 Xray 没有外部用户、没有序列化格式依赖枚举数值。保留它们只增加 `gctype_to_typeid[]` 表大小和认知负担。

**建议**：直接删除这 3 个条目，重新编号。所有依赖数值的地方（`xvalue.c` 的 `gctype_to_typeid[]`、`xgc.c` 的 GC dispatch 表）都是编译期静态数组，枚举名变了就自动对齐。

**影响**：~3 个文件，零逻辑改动。

---

## D-02：Error vs Exception 双重 GC 类型

```
XR_TERROR      — 7 个文件引用
XR_TEXCEPTION  — 11 个文件引用
```

`xvalue.c` 中两者都映射到 `XR_TID_EXCEPTION`。`xexception.c` 中 `XR_TERROR` 的唯一用途是在 `xr_exception_from_value` 中检测并转换为 Exception：

```c
if (XR_GC_GET_TYPE(gc) == XR_TERROR) {
    return xr_exception_from_error(X, (XrError *) gc);
}
```

**问题**：用户看到的都是 `Exception`。`Error` 是内部轻量错误结构，但暴露为独立 GC 类型增加了每处 switch/dispatch 的分支数。

**建议**：评估是否可以将 `XR_TERROR` 合并为 `XR_TEXCEPTION` 的一种变体（类似 WeakMap 用 flag 区分），或者至少在 GC 表和 typeof 映射中统一，只在内部结构转换时区分。

**复杂度**：中等（需要审计 Error 的分配和使用路径）。

---

## D-03：XR_TID_BYTES / XR_TID_TYPED_ARRAY / XR_TID_UPVALUE / XR_TID_ERROR 是幽灵 TID

```c
// Internal/GC types (not user-visible, used by RuntimeTypeInfo)
XR_TID_BYTES,        // 33
XR_TID_TYPED_ARRAY,  // 34
XR_TID_UPVALUE,      // 35
XR_TID_ERROR,        // 36
```

- `XR_TID_BYTES`：仅 `xi_lower_expr.c` 的 instanceof 表引用。消除 `XR_KIND_BYTES` 后应同步删除
- `XR_TID_TYPED_ARRAY`：仅 `xi_lower_expr.c` 引用，语义与 `XR_TID_ARRAY` 重复
- `XR_TID_UPVALUE`：零使用（grep 只在定义处出现）
- `XR_TID_ERROR`：同上

**建议**：Step 2 消除 `XR_KIND_BYTES` 时一并清理这 4 个 TID。Upvalue 和 Error 都不是用户可见类型，不需要 TID。

---

## D-04：AST_SLICE_EXPR 类型推断漏洞（Bytes 已修但 Map 未来可能踩）

```c
// xanalyzer_visitor.c:1057
if (src && XR_TYPE_IS_ARRAY(src)) {
    result = src;
} else if (src && XR_TYPE_IS_STRING(src)) {
    result = xr_type_new_string(NULL);
} else {
    result = xr_type_new_unknown(NULL);  // <- 所有非 Array/String -> unknown
}
```

当前问题：`XR_KIND_BYTES` 的 slice 返回 `unknown`（修完 Bytes->Array 后自动解决）。

**隐含风险**：未来如果 Map/Json 支持 slice 语义，这里会再次需要扩展。建议加注释标记此处为"扩展点"，但不过度工程。

---

## D-05：XR_KIND_FIXED_ARRAY — 编译期类型无运行时对应物

```
XR_KIND_FIXED_ARRAY  // Fixed-length array: [N]T (compile-time length, runtime Array<T>)
```

分析器有 `XR_KIND_FIXED_ARRAY`，运行时是 `XrArray`。`xtype.c` 有 3 处 `XR_KIND_FIXED_ARRAY <-> XR_KIND_ARRAY` 兼容判断。

**现状**：这个设计是正确的——`[N]T` 是编译期长度约束，运行时退化为 `Array<T>` 是合理的。与 Bytes 的区别在于：Fixed_Array 真的需要携带额外信息（length），不能简单别名化。

**建议**：保留现状。但确保每次新增 Array 相关路径时，也检查 FIXED_ARRAY 兼容判断。

---

## D-06：OP_INVOKE_BUILTIN 独立于 OP_INVOKE 的双轨 dispatch

VM 有两套 method invoke 路径：
- `OP_INVOKE`：通用路径（IC -> XrClass lookup -> 慢路径）—— 688 行
- `OP_INVOKE_BUILTIN`：硬编码按 GC 类型 dispatch 到 C 方法表

这在 unified-class-dispatch 计划中已有详细分析。与 Slice 问题的关联是：
每新增一个 GC 类型（如 `XR_TARRAY_SLICE`），`OP_INVOKE_BUILTIN` 都需要新增分支，否则该类型的方法调用全部 fallthrough 到错误路径。

**这就是 `xslice_builtins.c` 存在的原因**——它本质上是双轨 dispatch 的后果。

**建议**：unified-class-dispatch 计划实施后，`OP_INVOKE_BUILTIN` 消失，这类问题自然根治。Slice/Bytes 清理是该计划的良好前置步骤。

---

## D-07：native_type_defs 方法定义与运行时方法表的真相源分裂

分析器的方法签名来自 `xnative_type_defs.inc`（嵌入的 `.xr` 伪源码）。运行时的方法来自 `*_builtins.c` 或 `*_methods.c` 的 C 函数注册。

**风险**：两边不同步 -> 分析器认为方法存在但运行时没有（或反过来）。

例：Array 的 `splice/concat/flat/copyWithin` 在 `xnative_type_defs.inc` 中声明了，但如果运行时还没实现，用户代码编译通过却运行时崩溃。

**建议**：长期应有自动化验证（CI 脚本）比对 native_type_defs 声明与 `*_builtins.c` 注册的方法名集合，报告不匹配。短期在 native_type_defs.inc 顶部加注释说明真相源关系。

---

## D-08：`xr_kind_is_container()` 等分类函数需要随 KIND 增删同步更新

```c
static inline bool xr_kind_is_container(XrTypeKind k) {
    return k == XR_KIND_ARRAY || k == XR_KIND_MAP || k == XR_KIND_SET || k == XR_KIND_BYTES;
}
static inline bool xr_kind_is_builtin_iterable(XrTypeKind k) {
    return k == XR_KIND_ARRAY || k == XR_KIND_MAP || k == XR_KIND_SET || k == XR_KIND_STRING ||
           k == XR_KIND_BYTES;
}
```

每个这样的函数都是一个潜在的"忘了加新类型"bug 点。

**建议**：消除 `XR_KIND_BYTES` 后减少了一个需要并列的项。长期可考虑 bit-flag 方案（`kind_flags[XR_KIND_COUNT]`，编译时初始化），但当前规模下 inline 函数足够，只需在 `XR_KIND_COUNT` 附近加 `_Static_assert` 防止悄悄新增 kind 而忘了更新分类函数。

---

## 优先级排序

| 编号 | 改动 | 收益 | 成本 | 优先级 |
|------|------|------|------|--------|
| **Slice/Bytes Step 1** | 消除 XR_TARRAY_SLICE | 删 ~600 行，根治 slice 类型不可注解 | 中 | **P0** |
| **Slice/Bytes Step 2** | 消除 XR_KIND_BYTES | 删 ~30 行，消除所有并列判断 | 低 | **P0** |
| **D-01** | 删除 3 个僵尸 GC 枚举 | 减噪 | 极低 | P1 |
| **D-03** | 清理 4 个幽灵 TID | 减噪 | 极低 | P1（随 Step 2 一起） |
| **D-06** | 统一 method dispatch | 删 ~700 行 | 高 | P2（独立计划已有） |
| **D-07** | native_type_defs CI 校验 | 防回归 | 中 | P2 |
| **D-02** | Error/Exception 合并 | 减少分支 | 中 | P3 |
| **D-08** | kind 分类 static_assert | 防遗漏 | 极低 | P3 |
| **D-04/D-05** | 无需改动 | — | — | — |
