# Xray 统一类系统设计方案（任务 083）

**状态**: 设计完成，待实施
**前置任务**: 081 Phase 1a 已完成（user-instantiable Exception with cause）
**后续解锁**: 081 Phase 1b（subclass Exception）、Phase 2（ADT enum）、Phase 3（try!/catch! 糖）
**目标读者**: 在新会话中独立实施此重构的工程师（无需依赖前次会话上下文）

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
| 5 | 子类化能力不一致 | `class HttpError extends Exception { super(...) }` 现已失败（`vm_superinvoke` 的 `XMETHOD_PRIMITIVE` 无下降路径）——**这是 081 Phase 1b 的直接卡点** |
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
| 类构造路径 | `OP_NEW` + `XMETHOD_FLAG_CONSTRUCTOR`；XMETHOD_PRIMITIVE 已支持作为静态方法 |
| 类型反射 | `XrTypeMetadata` + `XrReflectCache` 已挂在 `XrClass` 上，`finalize` 时构建 |
| AOT 类系统 | `src/aot/xrt_class.h` 已有 `xrt_obj_alloc` / `xrt_type_register` / `xrt_instanceof` 三件套，**与 Tier B 对齐** |
| Shape ID（将废弃） | XrJson 当前用 `xr_gc_get_shape_id(&gc_header)` 把 shape ID 嵌进 GC header `extra` 字段做 IC；**本方案将其替换为 class transition**（§3.8），XrShape 整体删除 |

**关键洞察**：要做的不是"造新基础设施"，而是把 Tier A 类型**迁移到已有的 Tier B 基础设施上**，同时把 XrShape 也合并进 XrClass。

---

## 2. 目标与非目标

### 2.1 目标

**G1**：所有用户可见类型统一表示为 `XrInstance` + `XrClass`：
`Exception`、`DateTime`、`Logger`、`Channel`、`Iterator`、`Range`、`Regex`、`NetConn`、`NetListener`、`Task`、`EnumValue`、`EnumType`、`BigInt`、`StringBuilder`、`Tuple`、`Bytes`、`Array`、`Map`、`Set`、`Json`。

**G2**：消除 `XR_T*` GC type 中除以下少数特殊条目以外的全部条目：
- 保留：`XR_TNULL`、`XR_TBOOL`、`XR_TINT`、`XR_TFLOAT`（tagged value）
- 保留：`XR_TSTRING`（tagged value，已有，不变）
- 保留：`XR_TINSTANCE`（唯一的"普通对象" GC type）
- 保留：`XR_TCLASS`、`XR_TFUNCTION`、`XR_TCFUNCTION`、`XR_TMODULE`、`XR_TBLOB`、`XR_TCELL`、`XR_TBOUND_METHOD`（内部基础设施）
- 删除：`XR_TSHAPE`（XrShape 并入 XrClass，§3.8 详述）
- 保留：`XR_TCOROUTINE`、`XR_TCOROPOOL`（协程内部）
- 删除：`XR_TARRAY`、`XR_TMAP`、`XR_TSET`、`XR_TJSON`、`XR_TEXCEPTION`、`XR_TERROR`、`XR_TDATETIME`、`XR_TREGEX`、`XR_TLOGGER`、`XR_TBIGINT`、`XR_TSTRINGBUILDER`、`XR_TCHANNEL`、`XR_TITERATOR`、`XR_TENUM_TYPE`、`XR_TENUM_VALUE`、`XR_TRANGE`、`XR_TTASK`、`XR_TNETCONN`、`XR_TNETLISTENER`、`XR_TTUPLE`

**G3**：删除以下专用 opcode：`OP_NEWARRAY`、`OP_NEWMAP`、`OP_NEWSET`、`OP_NEWSTRINGBUILDER`、`OP_BYTES_NEW`、`OP_NEWEXCEPTION`。统一用 `OP_NEW`（class + args）。
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

**G5**：`class HttpError extends Exception { super(msg) }` 自然 work。081 Phase 1b 直接落地。

**G6**：AOT 路径性能不退化（核心目标——xray 真实部署目标是 AOT native）。

**G7**：VM/JIT 路径性能在可接受范围（用户已声明可妥协 5-10%）。

**G8**：消除独立 `XrShape` 类型——shape 信息内嵌进 `XrClass`，动态加字段通过 class transition 实现（V8 hidden class style）。`XR_TSHAPE` GC tag、`XrShape` C struct、`xr_gc_get_shape_id`、`xr_shape_*` API 全部删除。Json 变为「flags 含 `XR_CLASS_DYNAMIC_FIELDS` 的普通 class」。

### 2.2 非目标

- Json 的 class transition（V8 hidden class 等价）**只限于** `XR_CLASS_DYNAMIC_FIELDS` 标记类；普通用户类 finalize 后 shape 不可变
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
typedef struct XrNativeBodyDesc {
    uint32_t body_size;              // body 字节数（不含 GC header / klass / fields）
    void (*init)(XrInstance *, void *body);          // 初始化
    void (*destroy)(void *body);                     // 析构
    void (*traverse)(XrCoroGC *, void *body);        // GC 遍历
    void (*deep_copy)(XrInstance *src, XrInstance *dst);
    void (*to_shared)(XrInstance *src, XrInstance *dst);
} XrNativeBodyDesc;

struct XrClass {
    /* ... 现有字段 ... */
    XrNativeBodyDesc *native_body;   // 非 NULL 时此类有 body
};
```

**访问方式**：
```c
static inline void *xr_instance_native_body(XrInstance *inst) {
    XrClass *klass = inst->klass;
    return (uint8_t *)inst + sizeof(XrInstance) + klass->instance_size;
}
```

GC traverse、destroy、deep_copy、to_shared 不再走 `g_type_ops[XR_T*]`，而是走 `klass->native_body` 的回调。**`g_type_ops[]` 退化为只剩基础设施类型条目**。

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

**关键设计**：Array/Map 等 hot 类型的 `length`、`size`、`capacity` 在 `XrClass.fields[]` 中**显式声明为 native getter**（不是普通 XrValue field）。用 `XrFieldDescriptor.flags` 的新位 `XR_FIELD_NATIVE_GETTER` 区分；offset 字段当 IC 命中时用作"getter 函数索引"。

### 3.4 构造器统一

`OP_NEW R[A] = new R[B](C args at R[A+1..A+C])`：

```
1. 取 cls = R[B]（必须是 XrClass*）
2. 分配 XrInstance：
     size = sizeof(XrInstance) + sizeof(XrValue) * cls->field_count
     if (cls->native_body) size += cls->native_body->body_size
3. 写入 GC header (XR_TINSTANCE)、klass = cls
4. 用 cls->field_default_values 初始化 fields[]
5. 若 cls->native_body：调用 native_body->init(inst, body_ptr)
6. 调用 cls 的 constructor（XMETHOD_CLOSURE 或 XMETHOD_PRIMITIVE 都支持）：
   - PRIMITIVE: result = primitive(iso, inst_val, args, nargs)
                如果 PRIMITIVE 返回 XR_NULL，R[A] = inst_val
                否则 R[A] = primitive 返回值（兼容旧 native ctor 自分配的形态）
   - CLOSURE:   设置 this = inst，调用 closure；R[A] = inst_val
```

`new Exception("hi")`、`new Map<int>()`、`new HttpError(404)` 全走同一条 OP_NEW。这彻底替换今天的 `OP_NEWMAP / OP_NEWSET / OP_NEWARRAY / OP_NEWSTRINGBUILDER / OP_NEWEXCEPTION / OP_BYTES_NEW`。

**子类构造器调用父类**（`super(args)`）：
- `vm_superinvoke` 当前只支持 XMETHOD_CLOSURE
- 新版同时支持 XMETHOD_PRIMITIVE
- PRIMITIVE 父类 ctor 接收的 `this_val` 是已分配好的 subclass instance（layout 兼容父类）；PRIMITIVE 内部直接写 `inst->fields[parent_offset]` 即可

**这就是 081 Phase 1b 自然 unblock 的机制**——不需要任何特殊处理。

### 3.5 GC traverse 统一

```c
void xr_gc_traverse_instance(XrCoroGC *gc, XrGCHeader *obj) {
    XrInstance *inst = (XrInstance *)obj;
    XrClass *klass = inst->klass;

    // 1. 遍历 fields[]
    uint32_t fc = xr_class_instance_field_count(klass);
    for (uint32_t i = 0; i < fc; i++) {
        xr_coro_gc_markvalue(gc, inst->fields[i]);
    }

    // 2. 遍历 native body（如有）
    if (klass->native_body && klass->native_body->traverse) {
        klass->native_body->traverse(gc, xr_instance_native_body(inst));
    }

    // 3. 标记 klass 自身
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

### 3.8 动态字段 / Json：class transition 而非独立 Shape

XrJson 当前用独立 `XrShape` + `xr_gc_get_shape_id()`（存在 GC header `extra` 字段）做 IC。新版直接把这个能力融进 `XrClass`：

**核心设计**：

```c
struct XrClass {
    /* ... 现有字段 ... */
    uint32_t flags;                    // 新增 XR_CLASS_DYNAMIC_FIELDS bit
    XrClassTransitionMap *transitions; // symbol -> child XrClass*无序 map
    XrClass *transition_parent;        // transition 从哪个 parent 派生
    int transition_symbol;             // 派生时新增的 symbol
};
```

**工作机制**（以 `json.x = 1` 为例）：

```
1. inst->klass 当前无字段 x（field_symbol_to_index 查不到）
2. 因为 klass->flags & XR_CLASS_DYNAMIC_FIELDS，允许动态添加
3. 查 klass->transitions[symbol_x]：
   a. 命中：next_klass 已存在，走 transition
   b. 未命中：新建一个 child XrClass，继承当前 klass 全部字段 + symbol_x；
              child->field_count = parent->field_count + 1
              child->transitions = NULL（懒分配）
              child->transition_parent = klass
              klass->transitions[symbol_x] = child
4. inst->klass = next_klass
5. inst->fields 可能需 realloc（如果空间不足）——
   实际上用 "in-object + overflow" 两段式：
   - in-object: 初始分配时预留 N 个 slot（default 8）
   - overflow: 超出时 fields 拓展到堆上的 XrValue* overflow_fields
6. inst->fields[new_idx] = value
```

**IC 统一**：
- `XrICField` 不再有 `json_shape_id` / `json_field_idx` 特殊字段
- 所有对象（普通 instance 与 dynamic-field instance）都用统一的 `(klass, field_index)` IC 条目
- OP_GETPROP fast path 删除 `xr_value_is_json` 分支，只留 instance 路径
- mono/poly IC 对 Json 同样有效（因为相同结构的 Json 共享同一 klass——就像 V8 hidden class）

**性能影响**：
- 取消了 `xr_gc_get_shape_id` 的 GC header extra 字段查找（省一次内存访问）
- IC 代码合并减少 code cache footprint
- transition 创建是 cold path（只在首次赋值时发生）
- 后续字段访问都是 hot IC hit

**删除清单**：
- `src/runtime/object/xshape.{c,h}` —— 整个文件删除
- `XR_TSHAPE` GC tag 删除
- `xr_gc_get_shape_id` / `xr_gc_set_shape_id` 删除
- `XrICField.json_shape_id` / `json_field_idx` 字段删除
- `xr_ic_json_lookup` / `xr_ic_json_update` 删除
- `xvm_dispatch_object.inc.c` 中所有 `xr_value_is_json` 分支删除

---

## 4. 详细的迁移目录

### 4.1 类型分级与处理优先级

| 类型 | 当前 GC tag | native body? | 优先级 | 备注 |
|------|------------|-------------|--------|------|
| **Exception** | XR_TEXCEPTION | 无（纯 fields） | P0 | 解锁 081 Phase 1b |
| **DateTime** | XR_TDATETIME | 小 body（time_t、timezone） | P0 | 简单，先迁 |
| **Range** | XR_TRANGE | 无 | P0 | 三字段 (start/end/step) |
| **Logger** | XR_TLOGGER | 小 body（fd、buffer） | P0 | 类似 DateTime |
| **Iterator** | XR_TITERATOR | 中 body（state、current、source） | P1 | 多种子类型 |
| **EnumValue** | XR_TENUM_VALUE | 无（variant index + payload） | P1 | 081 Phase 2 ADT 依赖 |
| **EnumType** | XR_TENUM_TYPE | 中 body（variants） | P1 | 同上 |
| **Regex** | XR_TREGEX | 大 body（compiled NFA） | P1 | destroy 关键 |
| **NetConn** | XR_TNETCONN | 中 body（fd、tls ctx） | P1 | destroy 关闭 fd |
| **NetListener** | XR_TNETLISTENER | 中 body | P1 | 同上 |
| **Task** | XR_TTASK | 大 body（coro handle） | P1 | 与协程交互 |
| **Channel** | XR_TCHANNEL | 大 body（buffer + condvar） | P2 | 多核并发 |
| **BigInt** | XR_TBIGINT | 中 body（limb 数组） | P2 | 较冷 |
| **StringBuilder** | XR_TSTRINGBUILDER | 中 body（动态 buffer） | P2 | 半 hot |
| **Tuple** | XR_TTUPLE | 无（仅 fields） | P2 | 已经接近 instance |
| **Bytes / Blob** | XR_TBLOB | 大 body（u8 array） | P2 | 与 Array<u8> 重合 |
| **Json** | XR_TJSON | 无 native body（纯 dynamic fields） | P2 | class transition 已在 Phase 0 落地，此处只是迁移 GC tag |
| **Array** | XR_TARRAY | 大 body（typed data buffer） | P3 | 性能最敏感 |
| **Map** | XR_TMAP | 大 body（hash table） | P3 | 同上 |
| **Set** | XR_TSET | 大 body（hash set） | P3 | 同上 |
| **保留**: Coroutine、CoroPool、Closure、CFunction、Module、Class、Cell、BoundMethod、Blob、StringRaw | （内部）| —— | 不迁移 | 不对用户暴露 |

### 4.2 用户语言可见的 builtin 类型清单（迁移完成后）

prelude 提供的 class 列表（全部为普通 `class`，无 `@native`）：
- Object（所有类的基类）
- Exception, Error
- Array<T>, Map<K,V>, Set<T>, Tuple
- StringBuilder, Bytes
- Json
- Range, Iterator<T>
- DateTime, Regex, BigInt
- Logger
- NetConn, NetListener
- Task, Channel<T>
- Result<T,E>, Option<T>（Phase 2 ADT 完成后）

每个都是统一表示，可以被用户继承。

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

这一步**实际上已经接近完成**：`xrt_class.h` 的 `xrt_obj_alloc` / `xrt_instanceof` 已经基于 type_id。剩余工作是把 hot 路径的扩展 tag 检查改为 type_id 检查。

### R4 [HIGH] — XrShape 合并进 XrClass（已纳入本方案）

**决议**：直接采用方案 B（一步到位，不留过渡层）。设计详见 §3.8。

**关键设计要点**：
- `XrClass.field_symbol_to_index` 已经是 shape 的核心数据，直接复用
- `XrClass.flags |= XR_CLASS_DYNAMIC_FIELDS` 标记可加字段类
- `XrClass.transitions`（哈希表 symbol → child class）存 transition 表
- IC `XrICField` 不再区分 json/instance 路径，统一 `(klass, offset)`
- `xvm_dispatch_object.inc.c::OP_GETPROP` 中 Json 与 instance fast path 合并，删除 `xr_value_is_json` 分支

**对实施的影响**：
- Phase 0 新增 class transition 基础设施（~300 LOC）
- Json 从 P3 降级到 P2（shape 合并后它只是一个“dynamic-fields class”）
- IC 代码简化（XrICField 删除 json_shape_id 等字段）

**预期收益**：
- IC fast path 代码减半（去掉 Json 分支）
- 用户语言中 Json 可被继承（`class Config extends Json`）
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

整个重构按以下顺序落地，**每阶段独立 commit、独立 ctest + regression 验收**。

### Phase 0 — 基础设施 + class transition + XrShape 合并（2 天）

**目的**：在动任何 native 类型之前，把 unified 路径的全部骨架搭好（含 dynamic fields）。

**改动 A——native body 基础设施**：
- `src/runtime/class/xclass.h`：扩展 `XrClass` 加 `XrNativeBodyDesc *native_body`
- `src/runtime/class/xclass.c`：`xr_instance_size` 考虑 native body
- `src/runtime/class/xinstance.c`：
  - `xr_instance_new` 分配时考虑 `klass->native_body->body_size`
  - 新增 `xr_instance_native_body(inst)` 内联函数
- `src/runtime/class/xclass_builder.h`：新增 `xr_class_builder_set_native_body(builder, desc)`
- `src/runtime/gc/xcoro_gc_traverse.c`：扩展 `xr_gc_traverse_instance` 处理 native body

**改动 B——class transition + XrShape 删除**：
- `src/runtime/class/xclass.h`：新增 `XR_CLASS_DYNAMIC_FIELDS` flag、`XrClassTransitionMap *transitions`、`transition_parent`、`transition_symbol`
- 新建 `src/runtime/class/xclass_transition.{c,h}`：实现 `xr_class_transition_for_field(klass, symbol) -> XrClass*`（O(1) 哈希查找，缺失时分配新子 class）
- `src/runtime/class/xinstance.c`：新增 `xr_instance_add_field(inst, symbol, value)` 用于 dynamic field 添加（transition + field write）
- 删除 `src/runtime/object/xshape.{c,h}` 整个文件
- 删除 `XR_TSHAPE` GC tag
- 删除 `xr_gc_get_shape_id` / `xr_gc_set_shape_id`
- `src/vm/xic_field.h`：删除 `json_shape_id` / `json_field_idx` 字段；删除 `xr_ic_json_lookup` / `xr_ic_json_update`
- `src/vm/xvm_dispatch_object.inc.c`：删除所有 `xr_value_is_json` 分支，把 Json 走统一的 instance fast path

**新 opcode**：扩展现有 `OP_NEW` 支持 native body init + XMETHOD_PRIMITIVE ctor。

**新前端能力**：
- `vm_superinvoke` 加 XMETHOD_PRIMITIVE 下降路径
- 在 `XrayCoreClasses` 中预留 `exception_class`、`exception_message_offset`、`exception_stack_offset`、`exception_cause_offset` 常量

**验收**：所有现有测试通过；新增测试验证：(a) 带 native_body 的 class 能正常分配 / traverse / destroy；(b) dynamic class + transition 能正确加字段、IC 命中。

### Phase 1 — Exception 迁移（解锁 081 Phase 1b，1 天）

**目的**：把 Exception 从 Tier A 降级到 Tier B（普通 class，无 native body），验证整个 unified 路径。

**改动**：
- 删除 `XrException` C struct
- 重写 `stdlib/types/exception.xr` 为普通 class（无 `@native`）
- `xr_exception_get_message` 等 C API 改为读 `inst->fields[EXCEPTION_FIELD_MESSAGE]`
- `XR_IS_EXCEPTION(v)` 重定义为 `XR_IS_EXCEPTION_OR_SUBCLASS(v)`
- `xr_exception_alloc`、`xr_exception_new`、`xr_exception_from_value` 改为分配 XrInstance + 设字段
- `xr_exception_register_native_type` 删除（exception.xr 自带 ctor）
- `OP_NEWEXCEPTION` 删除（用 OP_NEW + Exception class）
- `XR_TEXCEPTION` GC tag 直接删除（不留别名）
- `xcoro_gc_traverse.c::xr_gc_traverse_exception` 删除
- `xgc.c::g_type_ops[XR_TEXCEPTION]` 删除
- DAP / LSP 中所有 `XR_TEXCEPTION` 引用改为 instanceof Exception
- `XR_IS_EXCEPTION` 宏直接重定义为 instanceof 检查

**关键测试**：
- `class HttpError extends Exception { code: int; constructor(code, msg) { super(msg); this.code = code } }` 必须 work
- 现有 081 Phase 1a regression 全部通过
- exception stack trace 完整保留

**验收**：081 Phase 1b 收尾，commit 标记为"081 Phase 1b: Exception subclassing via unified class"。

### Phase 2 — Cold native 类型批量迁移（2-3 天）

**目的**：批量迁移性能不敏感的 cold native 类型。

**目标类型**：DateTime、Range、Logger、Iterator、EnumValue、EnumType、Regex、NetConn、NetListener、Task、BigInt、Channel、StringBuilder、Tuple、Bytes、**Json**。

Json 在 Phase 0 class transition 落地后只需迁移 GC tag（已不需 native body，纯 dynamic fields）。

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

**验收**：每迁移 2-3 个类型 commit 一次；regression 必须保持 100%；性能对比基线（无明显退化）。

### Phase 3 — Hot collection 迁移（最难，1 周）

**目的**：迁移 Array、Map、Set 这三个性能极敏感的类型（Json 已在 Phase 2 完成）。

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

### Phase 4 — 清理与文档（1-2 天）

**目的**：最终清理，确保代码干净。

**改动**：
- 确认所有 `XR_T<NAME>` 用户类型 GC tag 已在 Phase 1-3 直接删除（无别名残留）
- 删除所有 `xr_*_register_native_type` 函数（统一为 `xr_*_register_class`）
- 删除 `gctype_to_typeid` 表
- 简化 `xr_value_typeid`
- 简化 `xvm_cold_object.c::vm_getprop_type_dispatch`（只剩基础设施类型）
- 简化 `xgc.c::g_type_ops`（只剩基础设施类型）
- 确认 `src/runtime/object/xshape.{c,h}` 已在 Phase 0 删除
- 更新 `docs/rules/architecture.md` 反映新架构
- 更新 `docs/rules/c-coding-standards.md` 关于 class registration 的章节
- 更新 `docs/rules/gc-memory.md` 关于 GC traverse 的章节
- 删除 `@native` 关键字的前端解析（直接删除，不留 deprecated）

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
   - 081 Phase 1b 解锁：`class HttpError extends Exception` 自然 work

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

## 8. 实施顺序与时间盒

| Phase | 时间 | 输出 | 解锁 |
|-------|------|------|------|
| Phase 0 | 2 天 | 基础设施 + class transition + XrShape 合并 | —— |
| Phase 1 | 1 天 | Exception 迁移 | **081 Phase 1b** |
| Phase 2 | 2-3 天 | 16 个 cold native 类型（含 Json） | 081 Phase 2（部分 enum 依赖） |
| Phase 3 | 1 周 | Array / Map / Set | —— |
| Phase 4 | 1-2 天 | 清理 + 文档 | —— |
| **总计** | **~2 周** | | |

可以与 081 后续阶段并行：Phase 0 + 1 完成后立即推进 081 Phase 1b/Phase 2，不必等到 Phase 3 完成。

---

## 9. 开放问题（实施前需确认）

实施者在开工前需要与用户确认以下决策：

**已决议**（按用户原则：无向后兼容、不保留旧接口、一步到位）：

1. ~~**Q1**：`XR_TEXCEPTION` 等过渡别名保留多久？~~ **决议：不保留过渡别名，每个 Phase 直接删除对应 GC tag。**
2. ~~**Q2**：Array/Map/Set/Json 是否作为 compiler-internal special class 保留？~~ **决议：全部迁移到统一 class 模型；字面量优化保留为专用 opcode（OP_ARRAY_LITERAL / OP_MAP_LITERAL）。**
3. ~~**Q3**：`@native` 关键字是否在前端语法中保留？~~ **决议：完全删除，不留 deprecated。**
4. ~~**Q4**：AOT 扩展 tag（`XR_TAG_ARRAY` 等）是否保留？~~ **决议：Phase 3 与 hot collection 迁移同步删除。**
5. ~~**Q5**：XrShape 与 XrClass 合并是否拆分为 084？~~ **决议：直接纳入本任务 Phase 0，一步到位，不拆分。**
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
  3. `src/runtime/class/xclass_transition.{c,h}`（**Phase 0 新建**） — class transition 哈希表与 child class 派生
  4. `src/runtime/object/xnative_type.{c,h}` — 现有 register_native_type（最终改名为 register_class）
  5. `src/runtime/object/xexception.{c,h}` — 081 Phase 1a 留下的 Exception 实现（迁移起点）
  6. `src/runtime/object/xshape.{c,h}`（**Phase 0 删除**） — 当前 Json shape 实现，需要全部迁移到 XrClass
  7. `src/runtime/object/xjson.{c,h}` — 当前动态字段对象，迁移到 dynamic-fields class
  8. `src/vm/xvm_dispatch_object.inc.c` — OP_GETPROP / OP_SETPROP fast path（删除 Json 分支）
  9. `src/vm/xvm_dispatch_invoke.inc.c` — OP_INVOKE 与 IC 集成
  10. `src/vm/xic_field.{c,h}` + `src/vm/xic_method.{c,h}` — IC 实现（删除 json_shape_id 字段）
  11. `src/runtime/gc/xcoro_gc_traverse.c` + `src/runtime/gc/xgc.c` — GC 集成
  12. `src/aot/xrt_class.{h}` + `src/aot/xrt_value.h` — AOT runtime（部分已 unified）

- **每阶段验收脚本**：
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

- **commit 规范**：`083 Phase N: <短描述>`，commit message 仅描述事实和原因，不引用文档路径，不写"Phase A.1"等阶段说辞（参考 `.windsurf/rules/main.md` 注释与 commit 铁律）。

- **若发现新险阶**：立即追加到本文档第 5 节 R1-R13 列表（或新建 R14+），不可绕过。
