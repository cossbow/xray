# Unified Class Dispatch: Builtin Type Method Refactoring

## Problem Statement

当前 VM 有 **3 条方法派发路径**，彼此不一致：

| 路径 | 类型 | 调用约定 | IC 类型 |
|------|------|----------|---------|
| `XrMethodSlot` 静态表 | Int/Float/Bool/BigInt/String/Array/Map/Set/Json/DateTime/Regex | `fn(iso, self, args, argc)` | `XrICBuiltin` |
| `XrClass` 实例方法 | 用户定义类、实例 | `fn(iso, args, nargs)` args[0]=self | `XrICMethod` |
| `native_type_classes` | StringBuilder/ArraySlice | `fn(iso, args, nargs)` args[0]=self | 无 IC |

这导致：
- `xvm_dispatch_invoke.inc.c` 有 **15+ goto labels**，1010 行
- 每种 builtin 类型手动维护 `static const XrMethodSlot[]` 表 → 协议遗漏
- 新增方法需要修改 4+ 个文件（symbol enum、method table、VM dispatch、protocol verify）
- 3 种 IC 类型增加复杂度
- 编译器无法对 builtin 做协议完整性检查

## Target Architecture

```
所有类型 → XrClass → xr_class_lookup_method → vtable/symbol map
```

每个 builtin 类型在 boot 时通过 XrClassBuilder 注册方法。VM 只有一条派发路径。

## 性能分析

| 场景 | 当前 | 统一后 |
|------|------|--------|
| VM cold path (no IC) | `table[type_id][symbol_id].fn` — 2 loads | `class->sym_map[sym] → class->methods[idx]` — 3 loads |
| VM hot path (IC hit) | cache→fn pointer — 1 load | cache→fn pointer — 1 load（**相同**）|
| JIT | Xi IR → XIR_CALL — 不走 method table | 不变 |
| AOT | xi_cgen 直接生成 C 调用 | 不变 |

**结论**: IC 命中（>95% 实际调用）性能完全相同。cold path 多 1 次 load，可忽略。

## Calling Convention Unification

当前两种签名：
```c
// XrMethodSlot (builtin types)
typedef XrValue (*XrMethodFn)(XrayIsolate *iso, XrValue self, XrValue *args, int argc);

// XrCFunctionPtr (class system)
typedef XrValue (*XrCFunctionPtr)(XrayIsolate *iso, XrValue *args, int nargs);
// args[0] = self, nargs includes self
```

**统一为 XrMethodFn**（self 独立参数），因为：
1. 所有 12 个 `*_methods.c` 文件（2150 行）已使用此签名
2. self 语义显式，不依赖 args[0] 隐式约定
3. 与 VM register layout 天然匹配：`fn(iso, R(a+1), &R(a+2), nargs)`

需要更新：
- `XrCFunctionPtr` → 改为 `XrMethodFn` 签名
- XrMethod 中 `as.primitive` 类型改为 `XrMethodFn`
- 少量使用 `XrCFunctionPtr` 的地方需要适配签名
  - StringBuilder (~5 methods)
  - ArraySlice (~3 methods)
  - Map/Set class-init methods
  - xr_bind_instance_method / xr_bind_static_method

## Implementation Plan

### 0. 准备：量化影响面

清点需要修改的文件和函数数量：

```
受影响的 XrCFunctionPtr 回调：
  - stdlib/prelude/prelude.c   (builtin functions — NOT methods, keep as-is)
  - src/runtime/class/*.c      (StringBuilder, ArraySlice class-init — ~8 functions)
  - src/runtime/object/xmap_instance_methods.c  (~5 functions)
  - src/runtime/object/xset_class_init.c        (if present)

需要删除：
  - src/runtime/value/xmethod_table.c           (registry)
  - src/runtime/value/xmethod_table.h           (header)
  - src/runtime/value/x{bool,int,float}_methods.{c,h}  (各自合入 XrClass)
  - src/runtime/object/x{string,array,map,set,json,bigint}_methods.{c,h}  (同上)
  - src/vm/xic_builtin.{c,h}                   (builtin IC)
  - OP_INVOKE_BUILTIN VM handler

需要简化：
  - src/vm/xvm_dispatch_invoke.inc.c            (~1010 → ~300 行)
  - src/ir/xi_emit_call.c                       (合并 XI_CALL_METHOD emit)
```

### 1. 统一调用约定

**目标**：所有 C 原生方法统一为 `XrMethodFn` 签名。

**改动**：
1. 在 `xclass_system.h` 中将 `XrCFunctionPtr` 改为：
   ```c
   typedef XrValue (*XrCFunctionPtr)(XrayIsolate *iso, XrValue self,
                                      XrValue *args, int argc);
   ```
   注意：此定义现在与 `XrMethodFn` 完全一致。

2. 更新 `XrMethod` 的 `as.primitive` 字段类型（已隐式匹配）。

3. 更新 VM 中所有 `XMETHOD_PRIMITIVE` 调用点，从：
   ```c
   method->as.primitive(isolate, &R(a + 1), nargs + 1);
   ```
   改为：
   ```c
   method->as.primitive(isolate, R(a + 1), &R(a + 2), nargs);
   ```

4. 更新所有使用 `XrCFunctionPtr` 签名的方法体（StringBuilder, ArraySlice, Map/Set class-init methods）适配 `(iso, self, args, argc)` 签名。

**验证**：全量测试通过。

### 2. 在 XrClass 上注册 builtin 方法

**目标**：每个 builtin 类型的 XrClass 携带完整方法。

**改动**：

1. 扩展 `XrClassBuilder` 支持 `XrMethodFn` 注册（如果需要，可能已经通过 `xr_class_builder_add_method` 支持 `XrCFunctionPtr`）。

2. 为每个 builtin 类型创建 `xr_<type>_register_methods(XrayIsolate *iso)` 函数，将现有 `*_methods.c` 中的方法注册到对应 XrClass：
   ```c
   // 示例：xarray_methods.c
   void xr_array_register_methods(XrayIsolate *iso) {
       XrClass *cls = iso->core->arrayClass;
       XrClassBuilder *b = xr_class_builder_reopen(iso, cls);
       xr_class_builder_add_method(b, "push",    m_push,    1, 0);
       xr_class_builder_add_method(b, "pop",     m_pop,     0, 0);
       xr_class_builder_add_method(b, "filter",  m_filter,  1, 0);
       // ...
       xr_class_builder_finalize(b);
   }
   ```

   **关键设计决策**：是否需要 `xr_class_builder_reopen`？
   - 当前 `xr_class_builder_finalize` 是终态操作，不能 re-open。
   - **方案 A**：在 `xr_core_init` 中一次性构建完整 class（方法和 field 一起注册）。
   - **方案 B**：添加 `xr_class_add_primitive_method(cls, symbol, fn)` 直接修改已 finalize 的 class。
   - **推荐方案 A**：在 `xr_core_init` 中直接用 builder 注册方法。避免引入 post-finalize mutation。

3. 修改 `xr_core_init` 中各 builtin class 的创建：

   **当前**（以 Array 为例）：
   ```c
   X->core->arrayClass = xr_array_create_class(X, X->core->objectClass);
   ```
   `xr_array_create_class` 使用 builder 创建 class 但不注册方法。

   **新**：在 `xr_array_create_class` 内部，builder 也注册所有方法。

4. 在 `xisolate_full.c` 初始化时，调用各类型的方法注册。

**验证**：boot 时通过 `xr_class_lookup_method` 能找到所有 builtin 方法。

### 3. VM 统一派发

**目标**：`OP_INVOKE` 用一条路径处理所有类型。

**改动**：

1. 实现 `xr_value_get_class_fast(XrayIsolate *iso, XrValue v)` — 从值到 XrClass 的 O(1) 映射：
   ```c
   static inline XrClass *xr_value_get_class_fast(XrayIsolate *iso, XrValue v) {
       if (XR_IS_INT(v))    return iso->core->intClass;
       if (XR_IS_FLOAT(v))  return iso->core->floatClass;
       if (XR_IS_BOOL(v))   return iso->core->boolClass;
       if (XR_IS_NULL(v))   return iso->core->nullClass;
       if (!XR_IS_PTR(v))   return NULL;

       XrGCHeader *gc = (XrGCHeader *)XR_TO_PTR(v);
       switch (XR_GC_GET_TYPE(gc)) {
           case XR_TSTRING:   return iso->core->stringClass;
           case XR_TARRAY:    return iso->core->arrayClass;
           case XR_TMAP:      return iso->core->mapClass;
           case XR_TSET:      return iso->core->setClass;
           case XR_TINSTANCE:  return xr_value_to_instance(v)->klass;
           case XR_TCLASS:    return xr_value_to_class_obj(v);
           // ... other types ...
           default:
               return iso->native_type_classes[XR_GC_GET_TYPE(gc)];
       }
   }
   ```

2. **重写 `OP_INVOKE` handler**（~150 行替代 ~770 行）：
   ```c
   vmcase(OP_INVOKE) {
       int a = GETARG_A(i);
       int method_symbol = PROTO_SYMBOL(cl->proto, GETARG_B(i));
       int nargs = GETARG_C(i);
       XrValue receiver = R(a + 1);
       savepc();

       // Channel hot path (blocking IO — must be inlined)
       if (xr_value_is_channel(receiver)) { ... keep inlined ... }

       // Unified class-based dispatch
       XrClass *klass = xr_value_get_class_fast(isolate, receiver);
       if (!klass) { VM_RUNTIME_ERROR(...); }

       // IC lookup
       XrICMethodTable *ic = xr_vm_ctx_ensure_ic_methods(vm_ctx, cl->proto);
       size_t cache_index = pc - PROTO_CODE_BASE(cl->proto) - 1;
       XrICMethod *cache = xr_ic_method_table_get(ic, cache_index);
       XrMethod *method = cache
           ? xr_ic_method_lookup(cache, klass, method_symbol)
           : xr_class_lookup_method(klass, method_symbol);

       if (!method || method->type == XMETHOD_NONE) {
           VM_RUNTIME_ERROR(XR_ERR_TYPE_NO_METHOD, ...);
       }

       if (method->type == XMETHOD_PRIMITIVE) {
           R(a) = method->as.primitive(isolate, receiver, &R(a+2), nargs);
           VM_BUILTIN_INVOKE_CHECK_EXC();
           vmbreak;
       }

       if (method->type == XMETHOD_CLOSURE) {
           // Push new call frame (existing logic)
           ...
       }
   }
   ```

3. **Channel/Task/Coro 特殊处理保持内联**：
   这些类型有复杂的阻塞语义（send/recv 可能 block coro），不能走通用 class dispatch。
   保留在 OP_INVOKE 开头作为 fast-path 特判。

4. **删除 `OP_INVOKE_BUILTIN`**：
   编译器不再 emit 此 opcode。所有方法调用统一为 `OP_INVOKE`。

**验证**：全量测试 + 回归测试通过。

### 4. 编译器清理

**改动**：
1. `xi_emit_call.c` 中 `xi_emit_call_method` — 始终 emit `OP_INVOKE`（已经是这样）。
2. 删除 `xi_emit_call_builtin` 中 `OP_INVOKE_BUILTIN` 的 emit 路径。
3. Xi IR 中如果有 `XI_CALL_BUILTIN` 与 `XI_CALL_METHOD` 的区分，统一为 `XI_CALL_METHOD`。
4. 更新 `xi_lower_expr.c` / `xi_lower_stmt.c` 中的 lowering 逻辑。

### 5. IC 统一

**改动**：
1. 删除 `XrICBuiltin` / `XrICBuiltinTable`（src/vm/xic_builtin.c/h）。
2. 所有方法调用共用 `XrICMethod` / `XrICMethodTable`。
3. `xr_vm_ctx_ensure_ic_builtin` → 删除。
4. `xr_vm_ic.c` 中合并相关代码。

### 6. 清理遗留代码

**删除文件**：
- `src/runtime/value/xmethod_table.c` — 静态注册表
- `src/runtime/value/xmethod_table.h` — 头文件
- `src/vm/xic_builtin.c` / `src/vm/xic_builtin.h` — builtin IC

**保留但简化文件**：
- `src/runtime/value/x{bool,int,float}_methods.c` → 方法体保留（注册到 XrClass），header 中的 extern table 声明删除
- `src/runtime/object/x{string,array,map,set,json,bigint}_methods.c` → 同上
- `stdlib/{datetime,regex}/*_methods.c` → 同上

**修改文件**：
- `src/api/xisolate_full.c` — 删除 `xr_method_table_verify_protocols()` 调用
- `src/module/xbuiltin_method_defs.h` — 更新同步规则注释

## 实施顺序与风险

| 步骤 | 改动范围 | 风险 | 可回滚 |
|------|---------|------|--------|
| 1. 统一调用约定 | ~15 个方法体签名 | 低 | ✅ |
| 2. 注册方法到 XrClass | ~12 个注册函数 | 中（boot 顺序依赖） | ✅ |
| 3. VM 统一派发 | 1 个核心文件 | 高（影响所有方法调用） | ✅ 可保留旧 code 并用 flag 切换 |
| 4. 编译器清理 | ~3 个文件 | 低 | ✅ |
| 5. IC 统一 | ~4 个文件 | 中 | ✅ |
| 6. 清理遗留 | 删除文件 | 低 | ❌ 单向 |

**建议**：每一步完成后做全量测试。步骤 3 是最关键的一步，建议先保留旧代码路径作为 fallback，用 `#if XR_UNIFIED_DISPATCH` 控制，验证通过后再删除。

## 代码量预估

| 类别 | 变化 |
|------|------|
| 删除 | ~1200 行（method_table + IC_builtin + dispatch spaghetti）|
| 新增 | ~300 行（注册函数 + 统一 dispatch）|
| 修改 | ~200 行（签名适配）|
| **净变化** | **-700 行** |

## 长期收益

1. **协议完整性**：编译器可对 builtin class 做接口检查（未来 Xray 自定义 builtin 时）
2. **一条派发路径**：VM dispatch ~300 行（当前 ~1800 行）
3. **一种 IC**：删除 XrICBuiltin（与 XrICMethod 重复）
4. **方法注册无遗漏**：通过 XrClassBuilder 注册 → finalize 时自动建 symbol map 和 vtable
5. **可扩展**：未来 Xray 语言自定义 builtin 时，方法体可以直接用 Xray 写

---

# Phase 2: Zero-Overhead Xray-Written Builtins

## 动机

步骤 1-6（统一 dispatch）消除了架构复杂度，但方法体仍然是手写 C。
手写 C 的 bug 来源：

| Bug 类型 | 示例 |
|----------|------|
| 协议遗漏 | Regex 缺 toString（已修复） |
| GC root 遗漏 | filter() 循环中 callback 触发 GC，result 被回收 |
| 调用约定错误 | XrCFunctionPtr vs XrMethodFn 签名不一致 |
| 边界条件 | argc 检查遗漏、null receiver 未处理 |
| 类型混淆 | XR_TO_ARRAY 用在非 Array 值上 |

根治办法：**用 Xray 写方法体**。编译器的类型系统、GC root 追踪、null safety
在编译期消灭这些 bug。

## 零开销原理

Xray 已有 AOT 管线：

```
Xi IR (typed SSA) → xi_cgen → C source → system CC → .o
```

如果 builtin 方法用 Xray 写，**构建时** AOT 编译为 C，链入 libxray.a。
生成的 C 与手写等价 —— 零运行时开销。

```
构建时（一次性）：
  stdlib/core/array.xr  ──┐
  stdlib/core/map.xr    ──┤→ xray AOT → build/gen/stdlib_*.c → cc → libxray.a
  stdlib/core/set.xr    ──┤
  stdlib/core/string.xr ──┘

运行时：
  libxray.a 中方法体 = 等价手写 C 的机器码
```

## 对比：手写 C vs Xray 源码

### Array.filter — 手写 C（当前）

```c
static XrValue m_filter(XrayIsolate *iso, XrValue self, XrValue *args, int argc) {
    XrArray *arr = XR_TO_ARRAY(self);
    if (argc < 1) return xr_null();
    XrValue predicate = args[0];
    XrArray *result = xr_array_new(iso);       // ← GC root?
    for (int i = 0; i < arr->length; i++) {
        XrValue item = arr->data[i];
        XrValue ok = xr_call(iso, predicate, &item, 1);  // ← 可触发 GC
        if (xr_truthy(ok))
            xr_array_push(result, item);       // ← result 还活着吗?
    }
    return xr_array_value(result);
}
```

### Array.filter — Xray 源码

```xray
impl<T> Array<T> : Iterable<T> {
    fn filter(predicate: (T) -> bool): Array<T> {
        let result = Array<T>()
        for item in this {
            if predicate(item) {
                result.push(item)
            }
        }
        return result
    }
}
```

### AOT 编译器生成的 C（自动，不需维护）

```c
static XrValue array_filter(XrayIsolate *iso, XrValue self, XrValue *args, int argc) {
    XrArray *arr = XR_TO_ARRAY(self);
    XrValue predicate = args[0];
    XrArray *result = xrt_array_new();
    // 编译器自动插入 GC root registration
    xrt_gc_push_root((XrValue *)&result);
    int len = arr->length;
    for (int i = 0; i < len; i++) {
        XrValue item = arr->data[i];
        XrValue ok = xrt_call(iso, predicate, &item, 1);
        if (xrt_truthy(ok))
            xrt_array_push(result, item);
    }
    xrt_gc_pop_root();
    return xr_array_value(result);
}
```

**关键差异**：编译器自动插入了 `gc_push_root / gc_pop_root`。
手写 C 中这一步靠程序员记忆 —— 遗漏即 crash。

## Intrinsic 函数体系

方法体用 Xray 写，但底层内存操作（数组扩容、哈希桶访问）不能用 Xray
表达 —— 需要 **intrinsic 函数**：编译器识别并直接内联为 C 操作。

### 声明方式

```xray
// stdlib/core/_internal/array_intrinsic.xr
// 此模块仅 stdlib/core/ 内部可见，用户代码不可 import

extern fn __array_raw_get<T>(arr: Array<T>, index: int): T
extern fn __array_raw_set<T>(arr: Array<T>, index: int, value: T)
extern fn __array_get_len<T>(arr: Array<T>): int
extern fn __array_set_len<T>(arr: Array<T>, n: int)
extern fn __array_ensure_cap<T>(arr: Array<T>, n: int)
extern fn __array_new<T>(): Array<T>
```

### 编译器处理

编译器看到 `__array_raw_get(arr, i)` 时，**不生成函数调用**，
直接在 xi_cgen 中内联为：

```c
arr->data[i]          // __array_raw_get
arr->data[i] = value  // __array_raw_set
arr->length           // __array_get_len
arr->length = n       // __array_set_len
```

### 完整 intrinsic 清单（~20 个）

```
Array kernel (5):
  __array_new, __array_raw_get, __array_raw_set,
  __array_get_len, __array_set_len, __array_ensure_cap

Map kernel (5):
  __map_new, __map_raw_get, __map_raw_set,
  __map_raw_delete, __map_get_count

Set kernel (4):
  __set_new, __set_raw_add, __set_raw_has, __set_raw_delete

String kernel (3):
  __string_raw_byte_at, __string_get_len, __string_slice_raw

GC / allocation (3):
  __gc_alloc, __gc_write_barrier, __gc_push_root, __gc_pop_root
```

高层方法（push/pop/filter/map/reduce/join/...）全部用 **普通 Xray 代码**
组合这些 intrinsic 实现。

## 不需要注解

| 问题 | 答案 | 原因 |
|------|------|------|
| 方法体需要 `@builtin` 注解吗？ | **不需要** | 普通 Xray 代码，无特殊标记 |
| Intrinsic 需要注解吗？ | **不需要** | 声明为 `extern fn`，编译器通过函数名识别 |
| `impl` 块需要 `@native` 注解吗？ | **不需要** | 编译器已知 Array/Map/Set 是 builtin 类型（`prelude_types.def`） |
| 可见性控制需要注解吗？ | **可选** | 用模块路径约定即可：`_internal/` 前缀 = 不可 import |

**理由**：

1. Intrinsic 的 `extern fn` 声明已经是 Xray 现有语法，不需要新语法
2. 编译器通过 `__` 前缀 + 已知函数名表（编译器内部硬编码 ~20 个名字）
   识别 intrinsic，不需要运行时反射或注解
3. `impl Array<T>` 中编译器已经知道 `Array` 的 GC type 是 `XR_TARRAY`
   （来自 `prelude_types.def`），可以直接生成正确的 cast 和内存访问
4. 可见性控制用 **模块路径约定**（`stdlib/core/_internal/` = 不导出给用户）
   比注解更简单，且与 Xray 的模块系统一致

如果未来需要更精细的控制（例如 unsafe 标记），可以加一个 `@internal`
访问修饰符，但这属于常规访问控制，不是 builtin 专用注解。

## 需要的基础设施

| 组件 | 现状 | 需要做的 |
|------|------|----------|
| AOT 编译器（xi_cgen） | ✅ 已存在 | 增加 intrinsic lowering（识别 `__xxx` 并内联） |
| 构建时 AOT | ❌ | CMake target: `.xr → .c → .o`，编入 libxray.a |
| Intrinsic 声明 | ❌ | ~20 个 `extern fn` 定义 |
| 接口/协议编译期检查 | ⚠️ 部分 | analyzer 对 `impl T : Interface` 检查方法完整性 |
| stdlib .xr 源文件 | ❌ | 从现有 C 方法体逐个翻译 |

## 迁移路线

**每次只迁移一个类型**，新旧并存，验证通过再继续。

| 顺序 | 类型 | C 行数 | 方法数 | 难度 | 理由 |
|------|------|--------|--------|------|------|
| 1 | Bool | 26 | 1 | 极低 | 最小试验田，仅 toString |
| 2 | Set | 190 | ~10 | 低 | 方法简单，无复杂迭代 |
| 3 | Map | 172 | ~8 | 低 | 结构类似 Set |
| 4 | Int/Float | 176 | ~12 | 中 | 涉及数值转换 intrinsic |
| 5 | Array | 358 | ~20 | 中 | 方法多，涉及 sort callback |
| 6 | Json | 174 | ~10 | 中 | 嵌套结构 |
| 7 | BigInt | 77 | ~4 | 低 | 方法少 |
| 8 | String | 482 | ~30 | 高 | 最复杂，UTF-8 处理 |

## 编译期 Bug 消除矩阵

| Bug 类型 | 手写 C | Xray 源码 | 消除机制 |
|----------|--------|-----------|----------|
| 协议遗漏 | 运行时 panic | **编译期错误** | `impl T : Iterable` 缺 iterator → 编译失败 |
| GC root 遗漏 | 隐蔽 use-after-free | **自动** | 编译器跟踪跨 call-point 存活的 GC 对象，自动 root |
| Null 解引用 | 靠 DCHECK | **类型系统** | `T?` 强制 null check，非 nullable 编译器保证非 null |
| 越界访问 | 靠手动检查 | **debug build 自动** | `__array_raw_get` 在 debug 模式自动加 bounds check |
| 调用约定不一致 | 3 种签名 | **统一** | 编译器生成正确的 calling convention |
| argc 检查遗漏 | 运行时异常 | **类型系统** | 函数签名固定参数个数，编译器在调用点检查 |

## 完整路线图

```
Phase 1: 统一 dispatch（步骤 1-6）
  └── 所有类型走 XrClass，消除 3 条派发路径
  └── 预期：-700 行代码

Phase 2: Xray 写 builtin（本节）
  ├── 2a: Intrinsic 基础设施（~20 个 extern fn + xi_cgen 内联）
  ├── 2b: 构建系统（CMake .xr → .c → .o pipeline）
  ├── 2c: 逐个类型迁移（Bool → Set → Map → ... → String）
  └── 预期：C 方法体 2150 行 → ~200 行 intrinsic 声明 + ~800 行 Xray 源码
            C 内核（内存布局 + GC）保持不变

验收标准：
  - 全量测试通过（ctest + regression）
  - 生成的 C 与手写等价（可 diff 验证）
  - 无新的运行时 annotation / 反射开销
```
