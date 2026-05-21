# Xray 统一类系统设计方案（任务 083）

**状态**: 设计完成，已按当前源码校准实施边界
**前置任务**: 081 已完成 user-instantiable Exception with cause
**后续解锁**: Exception subclassing、ADT enum、try!/catch! 糖
**目标读者**: 在新会话中独立实施此重构的工程师（无需依赖前次会话上下文）

---

## 0. 源码校准结论

本方案不是从零建立 class 体系。当前源码已经完成了若干关键前置能力：

- `xr_register_native_type()` 已经通过 `XrClassBuilder` 为 native 类型生成 `XrClass`
- `OP_INVOKE` 已经能通过 `native_type_classes[]` 找到 `XrClass` 并调用 `XMETHOD_PRIMITIVE`
- `XrICMethod` 已经以 `XrClass *` 为 receiver identity
- `OP_INVOKE_BUILTIN` 也已经通过 `XrClass` 查找 primitive 方法，但仍保留独立 `XrICBuiltin`
- class 构造当前不是通用 `OP_NEW`，而是 `OP_INVOKE constructor` 进入 `vm_invoke_class()`

因此，本任务的真实范围不是“建立 unified class”，而是把剩余双轨路径收敛到既有 `XrClass` 基础设施：

- 删除用户可见 native GC tag / native object 双轨
- 删除 Json 专用 shape IC，改为 dynamic-layout class identity
- 删除 property hardcoded dispatch
- 统一 native body 的 init / destroy / traverse / deep_copy / to_shared 生命周期
- 删除 `OP_INVOKE_BUILTIN` / `XrICBuiltin`，让所有普通方法调用归并到 `XrICMethod`
- AOT runtime 最后同步删除扩展 tag，避免 VM/JIT/AOT 同时重构造成故障面过大

---

## 1. 背景与动机

### 1.1 当前双轨架构（事实陈述）

xray 运行时存在两套并行的"类"实现：

**Tier A — Native class**（`@native`）
- 各自独立的 C struct（`XrArray`、`XrMap`、`XrException`、`XrDateTime`、…）
- 各自独立的 GC tag（`XR_TARRAY`、`XR_TMAP`、`XR_TEXCEPTION` 等，30+ 个）
- 各自独立的 GC traverse 函数（`xcoro_gc_traverse.c` 共 17 个）
- 通过 `xr_register_native_type` 注册 `XrClass`，但该 class **没有 instance 字段**——数据在 C struct 里
- 部分类型有专用构造 opcode（`OP_NEWARRAY` / `OP_NEWMAP` / `OP_NEWSET` / `OP_NEWSTRINGBUILDER` / `OP_BYTES_NEW` / `OP_NEWEXCEPTION`）
- 构造器统一用 `XMETHOD_PRIMITIVE`（C 函数指针）
- **不可被用户类继承**

**Tier B — Regular class**（用户 `class`、`struct`）
- 统一 `XrInstance` 表示（GC tag = `XR_TINSTANCE`）
- shape-based 字段布局：`XrClass.fields[]` 描述字段，`XrClass.instance_size` 决定 instance 大小
- 一个通用 GC traverse（`xr_gc_traverse_instance`）
- IC（inline cache）字段访问 + 多态方法分发（`XrICField` mono/poly/mega，`XrICMethod` mono/poly/mega + 16 项哈希）
- 构造器统一用 `XMETHOD_CLOSURE`（字节码闭包）
- 完整继承支持：父类 `instance_size` 之后追加子类字段，layout 完全兼容

### 1.2 双轨制的具体问题

源码盘点显示以下后果：

| # | 问题 | 量化体现 |
|---|------|---------|
| 1 | 用户语言泄漏 `@native` | `stdlib/types/exception.xr` 第一行 `@native class Exception` 暴露给所有读 stdlib 源码的用户 |
| 2 | 重复 GC traverse | `xcoro_gc_traverse.c` 17 个手写函数 + `xgc.c` 中 `g_type_ops[]` 30+ 项分发 |
| 3 | 重复属性访问分发 | `vm_getprop_type_dispatch`（`xvm_cold_object.c`）含 10+ 个 `XR_IS_*` 分支；每新增 native 类型需加一个 |
| 4 | 重复构造路径 | 9 个 `OP_NEW*` 专用 opcode + 字节码下降在 `xi_lower_expr` / `xi_emit_call` / 解释器 / JIT / AOT 各做一遍 |
| 5 | 子类化能力不一致 | `class HttpError extends Exception { super(...) }` 现已失败（`vm_superinvoke` 的 `XMETHOD_PRIMITIVE` 无下降路径）——**这是 Exception subclassing 的直接卡点** |
| 6 | dispatch 双轨 | `OP_GETPROP` 中 `xr_value_is_instance` 走 IC 字段访问，否则走 cold-path 各 native 类的 hardcoded getter dispatch |
| 7 | LSP / 反射 / typeof 双查表 | `xr_value_typeid` 维护 `gctype_to_typeid` 数组（`xvalue.c:133`），把 30+ 种 GC type 映射到 ~10 种 user-visible TypeId |
| 8 | JIT / AOT 边界更复杂 | `xm_jit_runtime.c:550` 有 `DEFENSIVE-TEMP[082]` 注释，heap_type 在 JIT-VM 边界需手动 backfill |
| 9 | 新增 stdlib 类型成本高 | 添加一个新 native 类需改：xgc_header.h（XR_T 枚举）+ 独立 C struct + GC traverse + g_type_ops 表项 + register_native_type + （可选）专用 opcode + xvm_cold_object 分发 + xvalue typeid 表 + xtype_names 表 |

### 1.3 已有可复用基础设施

盘点确认运行时**已具备 unified class 所需的全部底层能力**，不需要从零造轮子：

| 能力 | 现状 |
|------|------|
| 统一对象表示 | `XrInstance` + `XrClass` + flexible-array `XrValue fields[]` |
| 子类布局兼容 | `xclass_builder_finalize.c:322` 字段偏移从父类 `instance_size` 起递增；父类 C 代码读 `inst->fields[parent_offset]` 永远命中正确字段 |
| O(1) 字段查找 | `XrClass.field_symbol_to_index[]` |
| 继承链 instanceof | `XrClass.primary_supers[8]` 直接索引；超过 8 层走 `secondary_supers_hash` 开放寻址 |
| 字段 IC | `XrICField` 支持 mono/poly(4)/mega 三状态，每条 OP_GETPROP/OP_SETPROP 一格 |
| 方法 IC | `XrICMethod` 支持 mono/poly(4)/mega(16 槽 hash)，已有 `XrICState` API 暴露给 JIT |
| 类构造路径 | 当前通过 `OP_INVOKE constructor` → `vm_invoke_class()` 分配 `XrInstance`；`XMETHOD_PRIMITIVE` constructor 已支持，`super()` 仍需补 primitive 路径 |
| 类型反射 | `XrTypeMetadata` + `XrReflectCache` 已挂在 `XrClass` 上，`finalize` 时构建 |
| AOT 类系统 | `src/aot/xrt_class.h` 已有 `xrt_obj_alloc` / `xrt_type_register` / `xrt_instanceof` 三件套；但 `xrt_value.h` 仍有 Array/Map/StringBuilder/Closure 等扩展 tag，AOT 需要在 VM/JIT 稳定后单独收口 |
| Shape ID（将废弃） | XrJson 当前用 `xr_gc_get_shape_id(&gc_header)` 把 shape ID 嵌进 GC header `extra` 字段做 IC；**本方案将其替换为 dynamic-layout class transition**（§3.8），XrShape 整体删除 |

**关键洞察**：要做的不是"造新基础设施"，而是把 Tier A 类型**迁移到已有的 Tier B 基础设施上**，同时把 XrShape 合并为 `XrClass` 的 dynamic-layout 子形态。

---

## 2. 目标与非目标

### 2.1 目标

**G1**：所有用户可见 object 类型统一表示为 `XrInstance` + `XrClass`：
`Exception`、`DateTime`、`Logger`、`Channel`、`Iterator`、`Range`、`Regex`、`NetConn`、`NetListener`、`Task`、`EnumValue`、`EnumType`、`BigInt`、`StringBuilder`、`Tuple`、`Bytes`、`Array`、`Map`、`Set`，以及 Json object。

`Json` 类型本身仍表示任意 JSON value；只有 object 形态使用 dynamic-layout instance。

**G2**：消除 `XR_T*` GC type 中除以下少数特殊条目以外的全部条目：
- 保留：`XR_TNULL`、`XR_TBOOL`、`XR_TINT`、`XR_TFLOAT`（tagged value）
- 保留：`XR_TSTRING`（tagged value，已有，不变）
- 保留：`XR_TINSTANCE`（唯一的"普通对象" GC type）
- 保留：`XR_TCLASS`、`XR_TFUNCTION`、`XR_TCFUNCTION`、`XR_TMODULE`、`XR_TBLOB`、`XR_TCELL`、`XR_TBOUND_METHOD`（内部基础设施）
- 删除：`XR_TSHAPE`（XrShape 并入 XrClass，§3.8 详述）
- 保留：`XR_TCOROUTINE`、`XR_TCOROPOOL`（协程内部）
- 删除：`XR_TARRAY`、`XR_TMAP`、`XR_TSET`、`XR_TJSON`、`XR_TEXCEPTION`、`XR_TERROR`、`XR_TDATETIME`、`XR_TREGEX`、`XR_TLOGGER`、`XR_TBIGINT`、`XR_TSTRINGBUILDER`、`XR_TCHANNEL`、`XR_TITERATOR`、`XR_TENUM_TYPE`、`XR_TENUM_VALUE`、`XR_TRANGE`、`XR_TTASK`、`XR_TNETCONN`、`XR_TNETLISTENER`、`XR_TTUPLE`

**G3**：删除以下用户可见构造专用 opcode：`OP_NEWARRAY`、`OP_NEWMAP`、`OP_NEWSET`、`OP_NEWSTRINGBUILDER`、`OP_BYTES_NEW`、`OP_NEWEXCEPTION`。统一用当前源码已有的 `OP_INVOKE constructor`（class + args）。
保留**字面量优化** opcode `OP_ARRAY_LITERAL` / `OP_MAP_LITERAL`（编译期 known-class 优化）。

**G4**：用户语言中**完全消除** `@native` 关键字。stdlib 类型从

```xray
@native class Exception { message: string  stack: string }
```

变为

```xray
class Exception {
    message: string = ""
    stack: string = ""
    cause: Exception? = null
    constructor(message: string = "", cause: Exception? = null) {
        this.message = message
        this.cause = cause
    }
    fn toString() -> string { return "Exception: ${this.message}" }
}
```

`@native` 仅作为编译器内部 attribute，对应"hot 类型专精化路径"（如 Array），**不出现在 `.xr` 源码层**。

**G5**：`class HttpError extends Exception { super(msg) }` 自然 work。Exception subclassing 直接落地。

**G6**：AOT 路径性能不退化（核心目标——xray 真实部署目标是 AOT native）。

**G7**：VM/JIT 路径性能在可接受范围（用户已声明可妥协 5-10%）。

**G8**：消除独立 `XrShape` 类型——shape 信息内嵌进 `XrClass` 的 dynamic-layout 子形态，动态加字段通过 class transition 实现（V8 hidden class style）。`XR_TSHAPE` GC tag、`XrShape` C struct、`xr_gc_get_shape_id`、`xr_shape_*` API 全部删除。Json object 变为 `XR_CLASS_DYNAMIC_LAYOUT` instance；`Json` 类型本身仍是任意 JSON value。

### 2.2 非目标

- Json object 的 class transition（V8 hidden class 等价）**只限于** `XR_CLASS_DYNAMIC_LAYOUT` / `XR_CLASS_DYNAMIC_FIELDS` 标记类；普通用户类 finalize 后 shape 不可变
- **不**改变 tagged value 表示（保留 16 字节 tagged union，不引入 NaN-boxing）
- **不**重写 GC 层（Immix + per-coroutine heap 保持不变）
- **不**改变协程 / 调度 / netpoll 子系统
- **不**改变模块系统、import 解析、bundle 拓扑
- **不**重写 LSP / DAP（仅微调对 Exception 的引用方式）

---

## 3. 目标架构

### 3.1 对象表示总图

```
XrValue (16 字节 tagged union)
├─ XR_TAG_NULL / BOOL / I64 / F64        (immediate values, no heap)
├─ XR_TAG_PTR + heap_type=XR_TSTRING     (string, special — no class)
├─ XR_TAG_PTR + heap_type=XR_TINSTANCE   (everything else — XrInstance + XrClass)
├─ XR_TAG_STRUCT_REF                     (stack-allocated value-type struct, unchanged)
└─ XR_TAG_PTR + heap_type=XR_T<infra>    (XR_TCLASS / XR_TFUNCTION / XR_TCFUNCTION /
                                          XR_TMODULE / XR_TBLOB / XR_TCELL /
                                          XR_TBOUND_METHOD / XR_TCOROUTINE / XR_TCOROPOOL —
                                          内部基础设施，对用户语言不可见)
```

`XR_TINSTANCE` 之下的内存 layout：

```
┌──────────────────────────────────────────────────────────┐
│ XrGCHeader (16 字节)                                      │
│   - gc_next, type=XR_TINSTANCE, marked, extra, objsize    │
├──────────────────────────────────────────────────────────┤
│ XrClass *klass (8 字节)                                   │
├──────────────────────────────────────────────────────────┤
│ XrValue fields[N]                                         │
│   N = klass->field_count（不含 static）                    │
│   字段 0..K-1 = 父类继承字段                                │
│   字段 K..N-1 = 本类自己的字段                              │
├──────────────────────────────────────────────────────────┤
│ Native body (klass->native_body->body_size 字节，可选)     │
│   for Array: { data, length, capacity, elem_type, ... }   │
│   for Map:   { node, sizenode, lastfree, flags, ... }     │
│   for Channel: { buffer, head, tail, condvar, ... }       │
│   for Exception: 无（纯 fields）                           │
└──────────────────────────────────────────────────────────┘
```

**关键**：layout 与今天的 XrInstance **完全一致**——只是把 native 类型也搬过来，并允许在 fields[] 之后追加可选的 native body。

### 3.2 Native body 嵌入策略

少数原 native 类型有**复杂内部状态**（不是 XrValue 字段）。例如 Array 的元素 buffer、Map 的 hash table、Regex 的编译后 NFA。

**解法**：在 `XrClass` 上扩展可选的 "native body descriptor"：

```c
typedef enum XrNativeBodyCopyPolicy {
    XR_NATIVE_BODY_COPY_DEEP,
    XR_NATIVE_BODY_COPY_SHARED,
    XR_NATIVE_BODY_COPY_FORBID,
} XrNativeBodyCopyPolicy;

typedef struct XrNativeBodyDesc {
    uint32_t body_size;              // Body bytes excluding header, class pointer, and fields.
    uint16_t body_align;             // Body start alignment; zero means pointer alignment.
    XrNativeBodyCopyPolicy copy_policy;
    void (*init)(XrInstance *, void *body);
    void (*destroy)(void *body);
    void (*traverse)(XrCoroGC *, void *body);
    bool (*deep_copy)(XrCopyContext *ctx, XrInstance *src, XrInstance *dst);
    bool (*to_shared)(XrayIsolate *X, XrInstance *src, XrInstance *dst);
} XrNativeBodyDesc;

struct XrClass {
    /* ... existing fields ... */
    XrNativeBodyDesc *native_body;   // Non-NULL when this class owns native body storage.
};
```

**访问方式**：
```c
static inline void *xr_instance_native_body(XrInstance *inst) {
    XrClass *klass = inst->klass;
    return (uint8_t *)inst + xr_instance_body_offset(klass);
}
```

GC traverse、destroy、deep_copy、to_shared 不再按用户可见 GC tag 走 `g_type_ops[XR_T*]`，而是从 `XR_TINSTANCE` 统一进入 `klass->native_body` 的回调。**`g_type_ops[]` 退化为只剩基础设施类型条目**。

必须同时修改以下路径，否则 native body 会泄漏或跨协程行为错误：

- `xr_instance_size()`：包含 fields 区和 native body 区，并处理 body alignment
- `xr_instance_new()`：初始化 fields 后调用 `native_body->init`
- `xr_instance_clone()`：复制 fields 后按 copy policy 处理 body
- `xr_gc_traverse_instance()`：扫描 fields 后调用 `native_body->traverse`
- `xr_deep_copy_instance_with_ctx()`：复制 fields 后调用 `native_body->deep_copy`
- `xr_to_shared_instance()`：复制 fields 后调用 `native_body->to_shared`
- sweep / fixedgc cleanup：释放 instance 时调用 `native_body->destroy`

`copy_policy = XR_NATIVE_BODY_COPY_FORBID` 的类型（如 Task、Channel、Iterator 等）跨协程发送或转 shared 时必须报错，不能静默 pass-through。

**继承时的 native body 处理**：
- 子类继承父类的 `native_body`（指针拷贝）
- 子类**不可以** override `native_body`
- 子类可以追加普通 XrValue 字段，但不能追加 native body

这与 V8 ObjectTemplate / SpiderMonkey JSCLASS_RESERVED_SLOTS 的语义完全等价。

### 3.3 字段访问语义统一

| 操作 | 当前 | 新版 |
|------|------|------|
| `e.message`（Exception field） | `vm_getprop_type_dispatch` → `xr_exception_getter_message` | OP_GETPROP fast path: instance + IC field offset → `inst->fields[offset]` |
| `arr.length`（Array field） | `vm_getprop_type_dispatch` → 直接 hardcode | OP_GETPROP fast path: instance + IC → 通过 native getter slot |
| `inst.x`（用户 class 字段） | OP_GETPROP fast path（已 unified） | 不变 |

**关键设计**：Array/Map 等 hot 类型的 `length`、`size`、`capacity` 在 `XrClass.fields[]` 中**显式声明为 native getter**（不是普通 XrValue field）。

当前 `XrICFieldEntry` 只有 `(cls, offset)`，且 `OP_GETPROP` IC 命中后直接读 `inst->fields[field_index]`。因此不能把 `offset` 复用成 getter 索引，否则 IC hit 会把 native getter 当普通 field 读。

需要把 field descriptor 与 IC entry 都扩展为显式访问种类：

```c
typedef enum XrFieldAccessKind {
    XR_FIELD_ACCESS_VALUE,          // inst->fields[index]
    XR_FIELD_ACCESS_NATIVE_GETTER,  // Calls a registered getter function.
    XR_FIELD_ACCESS_DYNAMIC,        // Reads in-object or overflow dynamic field storage.
} XrFieldAccessKind;
```

IC entry 缓存 `(klass, symbol, kind, index)`。访问统一经由 helper：

- 普通 field：直接读写 `inst->fields[index]`
- native getter：调用 getter，setter 默认禁止，除非显式注册 native setter
- dynamic field：`index < in_object_capacity` 读写 in-object，否则读写 overflow

### 3.4 构造器统一

当前源码没有通用 `OP_NEW`。class 构造路径是 `OP_INVOKE constructor` 进入 `vm_invoke_class()`，该路径已经负责分配 `XrInstance` 并调用 constructor。为了减少改动面，本方案不新增 `OP_NEW`，而是扩展现有构造路径：

```
1. `OP_INVOKE` 发现 receiver 是 `XrClass`
2. `vm_invoke_class()` 识别 method 为 `constructor`
3. 分配 XrInstance：
     size = sizeof(XrInstance) + sizeof(XrValue) * cls->field_count
     if (cls->native_body) size += cls->native_body->body_size
4. 写入 GC header (XR_TINSTANCE)、klass = cls
5. 用 cls->field_default_values 初始化 fields[]
6. 若 cls->native_body：调用 native_body->init(inst, body_ptr)
7. 调用 cls 的 constructor（XMETHOD_CLOSURE 或 XMETHOD_PRIMITIVE 都支持）：
   - PRIMITIVE: result = primitive(iso, inst_val, args, nargs)
                如果 PRIMITIVE 返回 XR_NULL，R[A] = inst_val
                否则 R[A] = primitive 返回值（兼容旧 native ctor 自分配的形态）
   - CLOSURE:   设置 this = inst，调用 closure；R[A] = inst_val
```

`new Exception("hi")`、`new Map<int>()`、`new HttpError(404)` 最终都走同一条 `OP_INVOKE constructor` 语义。这彻底替换今天的 `OP_NEWMAP / OP_NEWSET / OP_NEWARRAY / OP_NEWSTRINGBUILDER / OP_NEWEXCEPTION / OP_BYTES_NEW` 的用户可见构造职责。

字面量和热点专精化仍可保留专用 opcode，但它们必须被定义为已知类的快速构造，不再是另一套对象模型。

**子类构造器调用父类**（`super(args)`）：
- `vm_superinvoke` 当前只支持 XMETHOD_CLOSURE
- 新版同时支持 XMETHOD_PRIMITIVE
- PRIMITIVE 父类 ctor 接收的 `this_val` 是已分配好的 subclass instance（layout 兼容父类）；PRIMITIVE 内部直接写 `inst->fields[parent_offset]` 即可

这就是 Exception subclassing 自然 unblock 的机制——不需要任何特殊处理。

### 3.5 GC traverse 统一

```c
void xr_gc_traverse_instance(XrCoroGC *gc, XrGCHeader *obj) {
    XrInstance *inst = (XrInstance *)obj;
    XrClass *klass = inst->klass;

    // Traverse instance fields.
    uint32_t fc = xr_class_instance_field_count(klass);
    for (uint32_t i = 0; i < fc; i++) {
        xr_coro_gc_markvalue(gc, inst->fields[i]);
    }

    // Traverse optional native body.
    if (klass->native_body && klass->native_body->traverse) {
        klass->native_body->traverse(gc, xr_instance_native_body(inst));
    }

    // Mark the class object itself.
    xr_coro_gc_markobject(gc, (XrGCHeader *)klass);
}
```

`g_type_ops[]` 缩减到只剩基础设施类型条目，用户类型的 17 个条目全部删除。

### 3.6 类型反射 / typeof / instanceof 统一

| API | 现状 | 新版 |
|-----|------|------|
| `typeof(x)` | `gctype_to_typeid[heap_type]` 表查映射 | `if XR_IS_INSTANCE(x) return x.klass.name;` 否则 tag 表 |
| `x is Foo` / `x as Foo?` | tag check + heap_type 分支 OR `xr_class_instanceof` | 统一走 `xr_class_instanceof(x.klass, foo_class)` |
| `Reflect.typeOf(x)` | 双查表 | 单查表（`klass->reflect_cache` / `klass->type_metadata`） |

### 3.7 throw / catch / Exception 路径

**当前**：`OP_THROW` / `OP_CATCH` 用 `XR_IS_EXCEPTION(v)` 检查（依赖 `heap_type == XR_TEXCEPTION`）。

**新版**：
```c
#define XR_IS_EXCEPTION_OR_SUBCLASS(v) \
    (XR_IS_INSTANCE(v) && xr_class_instanceof( \
        ((XrInstance*)XR_TO_PTR(v))->klass, \
        isolate->core_classes->exceptionClass))
```

`xr_exception_get_message` 等 C API 从读 `XrException->message` 改为读 `inst->fields[EXCEPTION_FIELD_MESSAGE]`（offset 是编译期常量，由 `XrayCoreClasses` 在 prelude 注册时记录）。

`xr_exception_from_value`、`xr_vm_unwind_with_trace`、`record_full_trace` 等都按相同模式改写。

**stack trace 字段** 改为 `Array<string>` 类型的普通 XrValue 字段——读写都是普通 instance 字段访问。

### 3.8 动态字段 / Json：dynamic-layout class 而非完整 nominal class

XrJson 当前用独立 `XrShape` + `xr_gc_get_shape_id()`（存在 GC header `extra` 字段）做 IC。新版删除独立 Shape 系统，但不是把 Json 粗暴变成普通 nominal class，而是引入 **dynamic-layout class**：

- `Json` 在类型系统中仍表示任意 JSON value（null / bool / number / string / array / object）
- `Json object` 的运行时表示统一为 dynamic-layout instance
- dynamic-layout class 只作为 hidden-class identity 和 IC key
- dynamic-layout class 不进入用户可见 class registry，不构建 vtable，不构建普通 reflection cache

**核心设计**：

```c
typedef enum XrClassKind {
    XR_CLASS_NOMINAL,          // Nominal user or builtin class.
    XR_CLASS_DYNAMIC_LAYOUT,   // Json object or object-literal hidden class.
} XrClassKind;

struct XrClass {
    /* ... existing fields ... */
    XrClassKind kind;
    uint32_t flags;
    XrClassTransitionMap *transitions;
    XrClass *transition_parent;
    int transition_symbol;
    uint16_t in_object_capacity;
};
```

**工作机制**（以 `json.x = 1` 为例）：

```
1. inst->klass 当前无字段 x（field_symbol_to_index 查不到）
2. 因为 klass->flags & XR_CLASS_DYNAMIC_FIELDS，允许动态添加
3. 查 klass->transitions[symbol_x]：
   a. 命中：next_klass 已存在，走 transition
   b. 未命中：新建一个 XR_CLASS_DYNAMIC_LAYOUT child class，
              继承当前 klass 全部字段 + symbol_x；
              child->field_count = parent->field_count + 1
              child->transitions = NULL（懒分配）
              child->transition_parent = klass
              klass->transitions[symbol_x] = child
4. inst->klass = next_klass
5. 字段存储使用 "in-object + overflow" 两段式：
   - in-object: 初始分配时预留 N 个 slot（default 8），存在 `inst->fields[0..N-1]`
   - overflow: 超出 in_object_capacity 时，`inst->fields[N-1]` 保留为 overflow pointer
     （指向堆上独立分配的 `XrValue[]`），logical field index ≥ N-1 走 overflow 读写
   - 具体布局：
     ```
     XrInstance (dynamic-layout, capacity=8)
     ┌──────────────────────┐
     │ gc + klass           │
     ├──────────────────────┤
     │ fields[0..6]         │  7 个 in-object field slot
     │ fields[7] = overflow │  → XrValue *overflow_fields (heap array)
     └──────────────────────┘
     ```
   - overflow array 按需 grow（double），GC traverse 必须同时扫描 in-object 和 overflow
   - 普通 nominal class（非 dynamic-layout）不受影响，仍然直接读 `inst->fields[index]`
6. inst->fields[new_idx] = value
```

**关键约束**：`klass->field_count` 是 logical field count，不等于 instance 内联分配的 slot 数。普通 instance 可直接读 `fields[index]`，dynamic-layout instance 必须通过 helper：

```c
XrValue xr_instance_get_dynamic_field(XrayIsolate *X, XrInstance *inst, uint16_t index);
bool xr_instance_set_dynamic_field(XrayIsolate *X, XrInstance *inst, uint16_t index, XrValue value);
```

helper 内部按 `klass->in_object_capacity` 决定读写 `inst->fields[]` 还是 overflow。`xr_gc_traverse_instance()`、deep_copy、to_shared 也必须走同一套 logical field 迭代，不能简单按 `klass->field_count` 扫 flexible array。

**IC 统一**：
- `XrICField` 不再有 `json_shape_id` / `json_field_idx` 特殊字段
- 所有对象（普通 instance 与 dynamic-field instance）都用统一的 `(klass, field_index)` IC 条目
- OP_GETPROP fast path 删除 `xr_value_is_json` 分支，只留 instance 路径
- mono/poly IC 对 Json 同样有效（因为相同结构的 Json 共享同一 dynamic-layout klass——就像 V8 hidden class）

**并发与 shared 语义**：

class transition map 是 isolate 级 mutable state。为避免 shared Json 跨 worker 新增字段时产生 transition race，规则如下：

- 普通 coroutine-owned dynamic object 可以新增字段并触发 transition
- shared dynamic object 只允许更新已有字段
- shared dynamic object 新增字段必须抛错，不能静默创建 transition
- 如果未来需要 shared object 动态扩展，必须给 transition map 加同步或改为 copy-on-write

**性能影响**：
- 取消了 `xr_gc_get_shape_id` 的 GC header extra 字段查找（省一次内存访问）
- IC 代码合并减少 code cache footprint
- transition 创建是 cold path（只在首次赋值时发生）
- 后续字段访问都是 hot IC hit

**删除清单**：
- `src/runtime/object/xshape.{c,h}` —— 整个文件删除（含 `XrShape` struct、`XrTransitionTable`、`XrShapeTransition`、shape registry、compact shape 系统）
- `XR_TSHAPE` GC tag 删除
- `xr_gc_get_shape_id` / `xr_gc_set_shape_id` 删除
- `XrICField.json_shape_id` / `json_field_idx` 字段删除
- `xr_ic_json_lookup` / `xr_ic_json_update` 删除
- `xvm_dispatch_object.inc.c` 中所有 `xr_value_is_json` 分支删除
- Compact shape 系统（`XrCompactType`、`xr_shape_new_compact`、`is_compact`、`field_offsets`、`compact_data_size` 等）一并删除——经源码验证 `xr_shape_new_compact` 当前无调用者，compact shape 从未被实际使用

---

## 4. 详细的迁移目录

### 4.1 类型分级与处理优先级

| 类型 | 当前 GC tag | native body? | 顺序 | 备注 |
|------|------------|-------------|------|------|
| **Exception** | XR_TEXCEPTION | 无（纯 fields） | 先迁 | 解锁 Exception subclassing |
| **DateTime** | XR_TDATETIME | 小 body（time_t、timezone） | 先迁 | 简单，先迁 |
| **Range** | XR_TRANGE | 无 | 先迁 | 三字段 (start/end/step) |
| **Logger** | XR_TLOGGER | 小 body（fd、buffer） | 先迁 | 类似 DateTime |
| **Iterator** | XR_TITERATOR | 中 body（state、current、source） | 中迁 | 多种子类型 |
| **EnumValue** | XR_TENUM_VALUE | 无（variant index + payload） | 中迁 | ADT enum 依赖 |
| **EnumType** | XR_TENUM_TYPE | 中 body（variants） | 中迁 | 同上 |
| **Regex** | XR_TREGEX | 大 body（compiled NFA） | 中迁 | destroy 关键 |
| **NetConn** | XR_TNETCONN | 中 body（fd、tls ctx） | 中迁 | destroy 关闭 fd |
| **NetListener** | XR_TNETLISTENER | 中 body | 中迁 | 同上 |
| **Task** | XR_TTASK | 大 body（coro handle） | 中迁 | 与协程交互 |
| **Channel** | XR_TCHANNEL | 大 body（buffer + condvar） | 后迁 | 多核并发 |
| **BigInt** | XR_TBIGINT | 中 body（limb 数组） | 后迁 | 较冷 |
| **StringBuilder** | XR_TSTRINGBUILDER | 中 body（动态 buffer） | 后迁 | 半 hot |
| **Tuple** | XR_TTUPLE | 无（仅 fields） | 后迁 | 已经接近 instance |
| **Bytes / Blob** | XR_TBLOB | 大 body（u8 array） | 后迁 | 与 Array<u8> 重合 |
| **Json object** | XR_TJSON | 无 native body（dynamic-layout fields + overflow） | 后迁 | dynamic-layout class 已在基础设施块落地，此处迁移 GC tag 与 VM fast path |
| **Array** | XR_TARRAY | 大 body（typed data buffer） | 热路径 | 性能最敏感 |
| **Map** | XR_TMAP | 大 body（hash table） | 热路径 | 同上 |
| **Set** | XR_TSET | 大 body（hash set） | 热路径 | 同上 |
| **保留**: Coroutine、CoroPool、Closure、CFunction、Module、Class、Cell、BoundMethod、Blob、StringRaw | （内部）| —— | 不迁移 | 不对用户暴露 |

### 4.2 用户语言可见的 builtin 类型清单（迁移完成后）

prelude 提供的用户可见类型列表（无 `@native`）：
- Object（所有类的基类）
- Exception, Error
- Array<T>, Map<K,V>, Set<T>, Tuple
- StringBuilder, Bytes
- Json（类型系统中的任意 JSON value；object 形态为 dynamic-layout instance）
- Range, Iterator<T>
- DateTime, Regex, BigInt
- Logger
- NetConn, NetListener
- Task, Channel<T>
- Result<T,E>, Option<T>（ADT enum 完成后）

这些类型统一暴露为 class 或 class-like runtime type；是否可继承由类型自身的 `final` / sealed 策略决定。`Json` 是任意 JSON value 类型，不作为普通 nominal base class 对用户开放继承；Json object 仅在运行时使用 dynamic-layout class。

---

## 5. 隐藏险阶（必读）

迁移过程中已识别的潜在陷阱，按风险等级排序：

### R1 [HIGH] — 跨协程 deep_copy / to_shared 协议

`xgc.c:53` 的 `g_type_ops[]` 不只是 traverse，还有 `deep_copy_*` 和 `to_shared_*`。这些用于跨协程消息传递（`Channel.send`）和共享存储。

**对策**：
- `XrNativeBodyDesc` 必须提供 `deep_copy(src_inst, dst_inst)` 与 `to_shared` 回调
- 普通用户类原本不需要 deep copy native body（无 body）
- 迁移类型必须实现 deep_copy；Channel / Iterator / Task 等"不可跨协程"类型保持 deep_copy = NULL（错误时报告"not transferable"）

### R2 [HIGH] — JIT 边界类型断言

`xm_jit_runtime.c:550` 已有 `DEFENSIVE-TEMP[082]` 的 heap_type 重建逻辑。

**对策**：JIT lowering 阶段读 IC 状态，monomorphic 时把 klass 指针 inline 进 deopt guard，generic 时 deopt 到 VM。这与今天的 `XM_GUARD_KLASS` 机制完全一致——只是把对 native heap_type 的检查也走这条路径。

### R3 [HIGH] — AOT 路径的扩展 tag

`src/aot/xrt_value.h:73` 定义了 AOT-only 扩展 tag：`XR_TAG_ARRAY=15`、`XR_TAG_MAP=16`、`XR_TAG_STRBUF=17`、`XR_TAG_CLOSURE=18`。

**对策**：AOT runtime 引入 `XrtArcHdr.type` 字段（已有，`xrt_class.h:131`），其值就是 `xrt_type_table[]` 的 type_id。所有对象访问统一走 `XRT_ARC_HDR(ptr)->type`。AOT 扩展 tag 删除。

这一步不能和 VM/JIT 表示迁移同时展开。`xrt_class.h` 的 `xrt_obj_alloc` / `xrt_instanceof` 已经基于 type_id，但 `xrt_value.h` 仍有 Array / Map / StringBuilder / Closure / String 等 AOT 扩展 tag，集合方法和字符串路径仍大量依赖 tag-specialized runtime。AOT 必须在 VM/JIT 表示稳定后单独收口，并用 VM-AOT diff 测试验证行为一致。

### R4 [HIGH] — XrShape 合并进 XrClass（已纳入本方案）

**决议**：删除独立 `XrShape`，但不把 Json 变成普通 nominal class。采用 §3.8 的 `XR_CLASS_DYNAMIC_LAYOUT`：dynamic-layout class 只作为 hidden-class identity 和 IC key，不进入用户可见 class registry，不构建普通 vtable / reflection cache。

**关键设计要点**：
- `XrClass.field_symbol_to_index` 已经是 shape 的核心数据，直接复用
- `XrClass.kind = XR_CLASS_DYNAMIC_LAYOUT` 标记 hidden-layout class
- `XrClass.transitions`（哈希表 symbol → child class）存 transition 表
- IC `XrICField` 不再区分 json/instance 路径，统一 `(klass, kind, index)`
- `xvm_dispatch_object.inc.c::OP_GETPROP` 中 Json 与 instance fast path 合并，删除 `xr_value_is_json` 分支
- `klass->field_count` 表示 logical field count；dynamic-layout instance 通过 in-object capacity + overflow helper 访问字段

**对实施的影响**：
- 基础设施阶段新增 dynamic-layout class transition
- Json object 从 hot collection 迁移中提前出来（shape 合并后它只是 dynamic-layout instance）
- IC 代码简化（XrICField 删除 json_shape_id 等字段）

**预期收益**：
- IC fast path 代码减半（去掉 Json 分支）
- Json object 字段访问与普通 instance 共用 class guard / field index IC
- AOT 路径上“Json klass == XrClass” 让 known-class 优化对 Json 也生效

### R5 [MID] — DAP 异常断点协议

`src/app/dap/xdap_debug.c` 在 OP_THROW 的 debug hook 里检查异常。

**对策**：在 `XrayCoreClasses` 暴露 `exception_message_offset` 常量；DAP 读 `inst->fields[exception_message_offset]`。在 prelude 注册 Exception 时记录此 offset。

### R6 [MID] — 反射 / typeof（XrType / XrTypeId）

`xr_value_typeid` 用 `gctype_to_typeid[XR_T*]` 表把 GC type 映射到 user-visible TypeId。删除 GC type 后必须从 class 取。

**对策**：
- `XrTypeId` 保留为前端 / IR 类型推断用的"形状 ID"，与 GC type 解耦
- runtime `typeof(x)`：if `XR_IS_INSTANCE(x)` return `x.klass.display_name`，否则按 tag 查表
- 编译期已知 class 的代码路径直接用 class name

### R7 [LOW] — Tuple 表示

迁移到 instance 后：`gc_header + klass + fields[length]`，**字段就是 tuple 元素**。`tuple.0` / `tuple.1` 在编译期下降到 `OP_GETPROP` 通过特殊符号（`SYMBOL_TUPLE_0`、`SYMBOL_TUPLE_1`...）。已有 IC 路径直接命中。

### R8 [LOW] — 字面量优化

`[1, 2, 3]` / `{a: 1, b: 2}` 当前编译为 `OP_NEWARRAY` + `OP_ARRAY_PUSH * 3`。

**对策**：保留这些**优化 opcode**（重命名为 `OP_ARRAY_LITERAL` / `OP_MAP_LITERAL`），它们只是 OP_NEW + 一组初始化的 sugar。是编译期 known-class 专精化的典型例子。

### R9 [LOW] — 静态字段与 class 元数据

`XrClass.static_field_values` 已经存在；迁移时把 `xr_*_register_native_type` 路径输出的 class 也写入 static fields（如果有）。

### R10 [LOW] — FFI

xray 没有传统 FFI（不能从 .xr 直接调任意 C 函数），native 实现都通过 register_native_type 暴露。所以 FFI 不是这次的风险点。

### R11 [LOW] — weakref / 弱引用

`Map.weak` 和 `Set.weak` 标记位仍在，但与 unified class 无关——它们是 native body 内部状态。迁移后 weakref 行为完全不变。

### R12 [MID] — `is`/`as` 类型检查的 fast path

编译期已知类型时，`x is Array<int>` 目前直接生成 `XR_IS_ARRAY(x)`（单指令）。

**对策**：
- 编译期已知类型时，IR lowering 把 `is Array` 下降到一条 `OP_INSTANCEOF_KLASS`
- VM dispatch 中这条 opcode = 一次 ptr load + 一次 cmp（与 `primary_supers` O(1) 检查）
- 与今天单条 `XR_IS_ARRAY` 的代价相差只有 **一次额外内存 load**（取 klass）

### R13 [LOW] — coroutine 安全性

Channel 是协程间通信的核心，迁移到 unified 后 deep_copy / to_shared 必须正确实现。这是 R1 的子集。

---

## 6. 实施计划（分阶段）

整个重构按以下顺序落地，**每个可回滚边界都必须独立验证 ctest + regression**。

### 实施块 0 — 基础设施 + dynamic-layout class + XrShape 合并

**目的**：在动任何 native 类型之前，把 unified 路径的全部骨架搭好（含 dynamic fields）。

**改动 A——native body 基础设施**：
- `src/runtime/class/xclass.h`：扩展 `XrClass` 加 `XrNativeBodyDesc *native_body`
- `src/runtime/class/xinstance.c`：`xr_instance_size` 考虑 fields、body alignment、native body
- `src/runtime/class/xinstance.c`：
  - `xr_instance_new` 分配时考虑 `klass->native_body->body_size`
  - 新增 `xr_instance_native_body(inst)` 内联函数
- `src/runtime/class/xclass_builder.h`：新增 `xr_class_builder_set_native_body(builder, desc)`
- `src/runtime/gc/xcoro_gc_traverse.c`：扩展 `xr_gc_traverse_instance` 处理 native body
- `src/coro/xdeep_copy.c`：扩展 instance deep_copy / to_shared 处理 native body copy policy

**改动 B——dynamic-layout class transition + XrShape 删除**：
- `src/runtime/class/xclass.h`：新增 `XrClassKind`、`XR_CLASS_DYNAMIC_FIELDS` flag、`XrClassTransitionMap *transitions`、`transition_parent`、`transition_symbol`、`in_object_capacity`
- 新建 `src/runtime/class/xclass_transition.{c,h}`：实现 `xr_class_transition_for_field(klass, symbol) -> XrClass*`
- `src/runtime/class/xinstance.c`：新增 dynamic field helper，用于 transition + in-object/overflow 读写
- 删除 `src/runtime/object/xshape.{c,h}` 整个文件
- 删除 `XR_TSHAPE` GC tag
- 删除 `xr_gc_get_shape_id` / `xr_gc_set_shape_id`
- `src/vm/xic_field.h`：删除 `json_shape_id` / `json_field_idx` 字段；删除 `xr_ic_json_lookup` / `xr_ic_json_update`；新增 field access kind
- `src/vm/xvm_dispatch_object.inc.c`：删除所有 `xr_value_is_json` 分支，把 Json 走统一的 instance fast path

**新前端能力**：

- **`vm_superinvoke` 加 XMETHOD_PRIMITIVE 路径**：
  当前 `xvm_cold_call.c:659` 硬检查 `method->type != XMETHOD_CLOSURE` 直接报错。
  新版在 CLOSURE 分支之前加 PRIMITIVE 分支：

  ```c
  if (method->type == XMETHOD_PRIMITIVE && method->as.primitive != NULL) {
      // this_val is the already-allocated subclass instance.
      // Parent PRIMITIVE ctor writes inst->fields[parent_offset..] directly.
      // Layout compatibility is guaranteed by xclass_builder_finalize:
      // subclass fields start after parent's instance_size.
      XrValue result = method->as.primitive(isolate, this_val, &base[arg_base], nargs);
      // PRIMITIVE ctor returns XR_NULL when it has written fields in-place
      // (matching vm_invoke_class convention); otherwise returns a replacement value.
      if (!XR_IS_NULL(result)) {
          // Should not happen for in-place PRIMITIVE ctors, but handle gracefully.
          base[a] = result;
      }
      return VM_COLD_BREAK;
  }
  ```

  关键约定：PRIMITIVE 父类 ctor 接收的 `this_val` **始终是子类 instance**（layout 兼容父类——
  子类字段在父类字段之后追加）。PRIMITIVE ctor 内部通过 `inst->fields[parent_field_offset]` 写入，
  不需要知道子类字段的存在。

- 在 `XrayCoreClasses` 中预留 `exception_class`、`exception_message_offset`、`exception_stack_offset`、`exception_cause_offset`、`exception_code_offset`、`exception_data_offset` 常量

**验收**：所有现有测试通过；新增测试验证：(a) 带 native_body 的 class 能正常分配 / traverse / destroy；(b) dynamic class + transition 能正确加字段、IC 命中。

### 实施块 1 — Exception 迁移（解锁 Exception subclassing）

**目的**：把 Exception 从 Tier A 降级到 Tier B（普通 class，无 native body），验证整个 unified 路径。

**XrException 字段迁移映射表**：

当前 `XrException` C struct 有 8 个字段，迁移到 `XrInstance` 后的处理如下：

| # | 原 C 字段 | 类型 | 迁移策略 | 新 field index 常量 |
|---|----------|------|---------|-------------------|
| 0 | `message` | `XrString*` | → XrValue field（`string`） | `EXCEPTION_FIELD_MESSAGE` |
| 1 | `stackTrace` | `XrArray*` | → XrValue field（`Array<string>`） | `EXCEPTION_FIELD_STACK` |
| 2 | `cause` | `XrException*` | → XrValue field（`Exception?`） | `EXCEPTION_FIELD_CAUSE` |
| 3 | `code` | `XrErrorCode` | → XrValue field（`int`）。C API `xr_exception_get_code` 改为读 `inst->fields[EXCEPTION_FIELD_CODE]`。用户侧不暴露（internal），但 VM / DAP 需要读 | `EXCEPTION_FIELD_CODE` |
| 4 | `file` | `XrString*` | 合并进 `stackTrace`。当前 `file`/`line`/`column` 仅在 `xr_exception_add_frame` 首次调用时有值，与 stackTrace 重复。迁移后删除独立字段，stack trace 的首帧已包含此信息 | 删除 |
| 5 | `line` | `int` | 同上，合并进 `stackTrace` | 删除 |
| 6 | `column` | `int` | 同上，合并进 `stackTrace` | 删除 |
| 7 | `userData` | `XrValue` | → XrValue field（`Json?`）。当前仅被 `xr_exception_from_value` 在 wrap non-exception throw 时写入 | `EXCEPTION_FIELD_DATA` |

迁移后的 Exception class 声明（stdlib/types/exception.xr）：

```xray
class Exception {
    message: string = ""
    stack: Array<string> = []
    cause: Exception? = null
    code: int = 0
    data: Json? = null
    constructor(message: string = "", cause: Exception? = null) {
        this.message = message
        this.cause = cause
    }
    fn toString() -> string { return "Exception: ${this.message}" }
}
```

field index 常量在 `XrayCoreClasses` 中注册（prelude init 时记录），C 代码通过常量访问：

```c
#define EXCEPTION_FIELD_MESSAGE 0
#define EXCEPTION_FIELD_STACK   1
#define EXCEPTION_FIELD_CAUSE   2
#define EXCEPTION_FIELD_CODE    3
#define EXCEPTION_FIELD_DATA    4
```

**改动**：
- 删除 `XrException` C struct
- 重写 `stdlib/types/exception.xr` 为普通 class（无 `@native`），按上述字段表
- `xr_exception_get_message` / `xr_exception_get_code` 等 C API 改为读 `inst->fields[EXCEPTION_FIELD_*]`
- `xr_exception_user_construct` 重写：不再分配 `XrException`，改为写 `inst->fields[]`（或改为 closure constructor 由 exception.xr 自带）
- `XR_IS_EXCEPTION(v)` 重定义为 `XR_IS_EXCEPTION_OR_SUBCLASS(v)`
- `xr_exception_alloc`、`xr_exception_new`、`xr_exception_from_value` 改为分配 XrInstance + 设字段
- `xr_exception_register_native_type` 删除（exception.xr 自带 ctor）
- `OP_NEWEXCEPTION` 删除（用 `OP_INVOKE constructor` + Exception class）
- `XR_TEXCEPTION` GC tag 直接删除（不留别名）
- `xcoro_gc_traverse.c::xr_gc_traverse_exception` 删除
- `xgc.c::g_type_ops[XR_TEXCEPTION]` 删除
- DAP / LSP 中所有 `XR_TEXCEPTION` 引用改为 instanceof Exception
- `XR_IS_EXCEPTION` 宏直接重定义为 instanceof 检查

**关键测试**：
- `class HttpError extends Exception { code: int; constructor(code, msg) { super(msg); this.code = code } }` 必须 work
- 现有 Exception regression 全部通过
- exception stack trace 完整保留

**验收**：Exception subclassing 测试通过，异常 stack trace 与 cause 语义保持一致。

### 实施块 2 — Cold native 类型批量迁移

**目的**：批量迁移性能不敏感的 cold native 类型。

**目标类型**：DateTime、Range、Logger、Iterator、EnumValue、EnumType、Regex、NetConn、NetListener、Task、BigInt、Channel、StringBuilder、Tuple、Bytes、**Json object**。

Json object 在 dynamic-layout class 落地后只需迁移 GC tag 与 VM fast path（已不需 native body，字段存储为 dynamic-layout fields + overflow）。

**对每个类型**：
1. 写出 `stdlib/types/<name>.xr` 普通 class
2. 把 C struct 字段拆成"普通 fields"和"native body"
3. 实现 `XrNativeBodyDesc`：init / destroy / traverse / deep_copy / to_shared
4. 调整 register 函数：用 `xr_class_builder_set_native_body`
5. 直接删除 `XR_T<NAME>` GC tag（不留别名）
6. 删除 `xr_gc_traverse_<name>`
7. 删除 `g_type_ops[XR_T<NAME>]`
8. 删除 `XR_IS_<NAME>` 宏（或重定义为 instanceof）
9. 调整 `vm_getprop_type_dispatch` 删除该类型分支（fields/methods 走 IC 路径）

**验收**：每迁移 2-3 个类型验证一次；regression 必须保持 100%；性能对比基线（无明显退化）。

### 实施块 3 — Hot collection 迁移

**目的**：迁移 Array、Map、Set 这三个性能极敏感的类型（Json object 已在 cold native 类型迁移中完成）。

**关键挑战**：`arr[i]`、`m["k"]` 等热路径不能退化。

**对策**：
- 每个类型的"hot 字段"（length、size、data）通过 `XrFieldDescriptor.flags |= XR_FIELD_NATIVE_GETTER` 暴露为 IC-friendly 路径
- 对 Array/Map/Set 数据访问保留专用 opcode（`OP_ARRAY_GET` / `OP_ARRAY_SET` / `OP_MAP_GET` / `OP_MAP_SET`）——它们语义保持，只是参数从 `XrArray*` 变为 `XrInstance*` + 已知 class
- 字面量构造保留 `OP_ARRAY_LITERAL` / `OP_MAP_LITERAL` 优化
- IC 路径监控 mono / poly / mega 比例，验证没有退化到 mega

**验收**：
- 全量 regression 100%
- benchmark suite 性能不退化超过 10%
- AOT 输出代码大小 / 性能与现状对比

### 实施块 4 — 清理、AOT 收口与文档

**目的**：最终清理，确保代码干净。

**改动**：
- 确认所有 `XR_T<NAME>` 用户类型 GC tag 已在对应实施块直接删除（无别名残留）
- 删除所有 `xr_*_register_native_type` 函数（统一为 `xr_*_register_class`）
- 删除 `gctype_to_typeid` 表
- 简化 `xr_value_typeid`
- 简化 `xvm_cold_object.c::vm_getprop_type_dispatch`（只剩基础设施类型）
- 简化 `xgc.c::g_type_ops`（只剩基础设施类型）
- 确认 `src/runtime/object/xshape.{c,h}` 已在基础设施块删除
- 更新 `docs/rules/architecture.md` 反映新架构
- 更新 `docs/rules/c-coding-standards.md` 关于 class registration 的章节
- 更新 `docs/rules/gc-memory.md` 关于 GC traverse 的章节
- 删除 `@native` 关键字的前端解析（直接删除，不留 deprecated）
- AOT runtime 删除 `XR_TAG_ARRAY` / `XR_TAG_MAP` / `XR_TAG_STRBUF` 等扩展 tag，改用 type table 和 ARC header type
- 增加 VM-AOT diff 覆盖集合、字符串、Json object、Exception 子类

**验收**：
- 全量 ctest + regression + compile-error 测试通过
- 代码量统计：预期减少 2000-3000 行 C 代码
- 文档更新完成

---

## 7. 接受标准（最终验收）

迁移全部完成后，必须满足：

1. **功能性**
   - ctest 100% 通过
   - regression 100% 通过
   - compile-error tests 100% 通过
   - 所有原 native 类型的用户可见行为不变（`new Array(...)`、`m.set(k, v)`、`new Exception("msg")` 等）
   - `class Foo extends <NativeType>` 对所有原 native 类型 work（除 final class 如 String）
   - Exception subclassing 解锁：`class HttpError extends Exception` 自然 work

2. **架构性**
   - `XrObjType` 枚举条目 ≤ 12 项（XR_TSHAPE 也已删除）
   - `xcoro_gc_traverse.c` 中的 traverse 函数 ≤ 5 个（基础设施 + xr_gc_traverse_instance）
   - `g_type_ops[]` 条目 ≤ 12 项
   - `vm_getprop_type_dispatch` 不再含用户类型分支
   - `OP_NEW*` 专用 opcode 全部删除（保留 OP_ARRAY_LITERAL / OP_MAP_LITERAL 字面量优化）
   - `.xr` 源码中无 `@native` 标记
   - 用户语言 spec 中无 "native class" / "regular class" 概念区分
   - `src/runtime/object/xshape.{c,h}` 已删除；无任何代码引用 `XrShape` / `xr_gc_get_shape_id`
   - `XrICField` 中无 `json_shape_id` / `json_field_idx` 字段
   - OP_GETPROP / OP_SETPROP fast path 中无 `xr_value_is_json` 分支

3. **性能**
   - VM 解释器：micro-benchmark 退化 ≤ 10%
   - JIT：hot-path benchmark 退化 ≤ 5%
   - AOT：原生代码 benchmark 退化 ≤ 3%

4. **文档**
   - `docs/design/unified_class.md`（本文档）保持最新
   - `docs/rules/architecture.md` 反映新架构
   - `docs/rules/gc-memory.md` 更新 traverse 章节
   - LSP / 反射 API 文档同步

---

## 8. 实施顺序

| 实施块 | 输出 | 解锁 |
|--------|------|------|
| 实施块 0 | 基础设施 + dynamic-layout class + XrShape 合并 | —— |
| 实施块 1 | Exception 迁移 | Exception subclassing |
| 实施块 2 | cold native 类型（含 Json object） | enum / Result / Option 的 runtime 统一 |
| 实施块 3 | Array / Map / Set | collection hot path 统一 |
| 实施块 4 | 清理 + AOT 收口 + 文档 | 发布前架构闭环 |

Exception 迁移完成后即可推进 Exception subclassing；不必等待集合 hot path 与 AOT 收口。

---

## 9. 开放问题（实施前需确认）

实施者在开工前需要与用户确认以下决策：

**已决议**（按用户原则：无向后兼容、不保留旧接口、一步到位）：

1. ~~**Q1**：`XR_TEXCEPTION` 等过渡别名保留多久？~~ **决议：不保留过渡别名，每个实施块直接删除对应 GC tag。**
2. ~~**Q2**：Array/Map/Set/Json 是否作为 compiler-internal special class 保留？~~ **决议：Array/Map/Set 迁移到统一 class 模型；Json object 迁移到 dynamic-layout instance；字面量优化保留为专用 opcode（OP_ARRAY_LITERAL / OP_MAP_LITERAL）。**
3. ~~**Q3**：`@native` 关键字是否在前端语法中保留？~~ **决议：完全删除，不留 deprecated。**
4. ~~**Q4**：AOT 扩展 tag（`XR_TAG_ARRAY` 等）是否保留？~~ **决议：最终删除；但在 VM/JIT 表示稳定后单独收口，不能和 hot collection 迁移混在同一个故障面里。**
5. ~~**Q5**：XrShape 与 XrClass 合并是否拆分为 084？~~ **决议：纳入本任务的基础设施块，采用 dynamic-layout class，不把 Json 变成普通 nominal class。**
6. ~~**Q6**：Tuple 是否走字面量优化？~~ **决议：保留 OP_TUPLE_LITERAL（性能敏感的字面量构造路径）。**

**剩余真正需要确认的开放问题**（如有）：
- 暂无。设计已闭环，可直接开工。

---

## 10. 实施提示（给新会话工程师）

- **必读 user_rules**（`.windsurf/rules/`）：
  - C 代码注释用英文，对话用中文
  - 每次代码改动必须运行测试验证
  - Bug 立即处理，不允许 workaround
  - 命令行避免 PTY 卡死：超过 1 行 / 200 字符走文件参数
- **必读现有架构文档**：`docs/rules/architecture.md`、`docs/rules/gc-memory.md`、`docs/rules/c-coding-standards.md`
- **关键参考代码**（按重要性）：
  1. `src/runtime/class/xinstance.{c,h}` — instance 分配 / 字段 / 方法
  2. `src/runtime/class/xclass.{c,h}` + `xclass_builder*.{c,h}` — class 构建
  3. `src/runtime/class/xclass_transition.{c,h}`（基础设施块新建） — class transition 哈希表与 child class 派生
  4. `src/runtime/object/xnative_type.{c,h}` — 现有 register_native_type（最终改名为 register_class）
  5. `src/runtime/object/xexception.{c,h}` — user-instantiable Exception 工作留下的实现（迁移起点）
  6. `src/runtime/object/xshape.{c,h}`（基础设施块删除） — 当前 Json shape 实现，需要迁移到 dynamic-layout class
  7. `src/runtime/object/xjson.{c,h}` — 当前动态字段对象，迁移到 dynamic-layout instance
  8. `src/vm/xvm_dispatch_object.inc.c` — OP_GETPROP / OP_SETPROP fast path（删除 Json 分支）
  9. `src/vm/xvm_dispatch_invoke.inc.c` — OP_INVOKE 与 IC 集成
  10. `src/vm/xic_field.{c,h}` + `src/vm/xic_method.{c,h}` — IC 实现（删除 json_shape_id 字段）
  11. `src/runtime/gc/xcoro_gc_traverse.c` + `src/runtime/gc/xgc.c` — GC 集成
  12. `src/aot/xrt_class.{h}` + `src/aot/xrt_value.h` — AOT runtime（部分已 unified）

- **每个实施块验收脚本**：
  ```bash
  # 增量构建
  cmake --build build -j 8

  # 单元测试
  (cd build && ctest --output-on-failure)

  # 回归
  XRAY_SKIP_BUILD=1 scripts/run_regression_tests.sh

  # 编译错误测试
  bash scripts/run_compile_error_tests.sh
  ```

- **若发现新险阶**：立即追加到本文档第 5 节 R1-R13 列表（或新建 R14+），不可绕过。
