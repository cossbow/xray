# Xray 泛型系统重构设计

## 现状总结

Xray 采用**编译期单态化（monomorphization）**策略实现泛型：

```
源码:    class Box<T> { ... } + new Box<int>(42)
          ↓ (xa_mono_pass)
AST:     class Box$i64 { ... } + new Box$i64(42)
          ↓ (xi_lower → xi_emit → VM)
运行时:  Box$i64 = 独立 XrClass，与 Box 无关联
```

### 已发现的缺陷

| # | 问题 | 影响 |
|---|------|------|
| 1 | `is` / `instanceof` 对泛型失效 | `new Box<int>(42) is Box` → false |
| 2 | mangled name 泄露到用户层 | 错误消息/DAP/Reflect 显示 `Box$i64` |
| 3 | reified type args 限制 | 只能存 2 个参数，5-bit tid（0-31） |
| 4 | `XrTypeRef**` / `XrType**` 类型混淆 | mono collector 依赖内存布局巧合 |
| 5 | AST 注入位置依赖 | 单态化类追加末尾导致运行时未定义 |
| 6 | `type_args` 生命周期不清晰 | pass 间信息丢失 |
| 7 | 跨模块泛型未验证 | import 带泛型类的模块可能失败 |
| 8 | 泛型 + 继承组合未验证 | 单态化不保留继承关系 |

---

## 设计目标

1. `new Box<int>(42) is Box` → **true**
2. 所有用户可见层（错误信息、Reflect、DAP）显示原始泛型名
3. 单态化保留泛型起源信息，支持类型检查
4. 消除 `XrTypeRef**` / `XrType**` 类型混淆
5. Reified type args 支持 ≥4 个参数，覆盖所有 type ID
6. 泛型 + 继承正确工作

---

## 核心设计

### 1. XrClass 增加 `generic_origin` 指针

```c
struct XrClass {
    ...
    const char *name;           // "Box$i64" (mangled, internal)
    const char *display_name;   // "Box" (user-visible, stripped)
    struct XrClass *generic_origin;  // → Box (the unspecialized class)
    struct XrClass *super;
    ...
};
```

**效果**：
- `xr_class_instanceof` 检查时，若 `cls->generic_origin == target`，返回 true
- 所有面向用户的 API 用 `display_name`
- `generic_origin` 指向原始泛型类的 XrClass 对象（运行时存在）

### 2. `is` / `instanceof` 语义扩展

```c
static inline bool xr_class_instanceof(const XrClass *cls, const XrClass *target) {
    if (cls == NULL || target == NULL) return false;
    if (cls == target) return true;

    // Monomorphized class matches its generic origin
    if (cls->generic_origin == target) return true;

    // Existing inheritance chain check...
    if (target->depth < 8) {
        return cls->depth >= target->depth &&
               cls->primary_supers[target->depth] == target;
    }
    // secondary supers hash...
}
```

### 3. 统一 `display_name` 抽象

新增内联函数，所有面向用户的输出点统一使用：

```c
static inline const char *xr_class_display_name(const XrClass *cls) {
    if (!cls) return "<null>";
    return cls->display_name ? cls->display_name : cls->name;
}
```

替换位置：
- `xreflect_api.c` — Reflect.typeOf
- `xinstance.c` — debug print / warning messages
- `xdap_debug.c` — DAP debugger variable display
- `xvm_cold_object.c` — runtime error messages
- `xreflect_registry.c` — type registry lookup
- `xreflect_type.c` — Reflect.Type.name

### 4. Reified Type Args：存储在 XrClass 而非 GC Header

#### 4.1 GC Header `extra` 现状分析

```
XrGCHeader (16 bytes):
  [0-7]   gc_next        (8B)  — GC 链表指针
  [8]     type           (1B)  — XrObjType 枚举
  [9]     marked         (1B)  — GC 三色标记 + remembered set (bits 0-4)
  [10-11] extra          (2B)  — 类型相关复用字段
  [12-15] objsize        (4B)  — 分配大小
```

`extra` (16 bits) 按对象类型多态复用：

| 对象类型 | bits 使用 | 用途 |
|----------|-----------|------|
| 所有类型 | bit 0 | shared storage flag |
| XR_TJSON | bits 2-15 | shape_id (14 bits) — JIT IC 依赖 |
| XR_TSTRING | bits 1-6 | interning/pool flags |
| XR_TSTRING | bit 4 | long string flag |
| XR_TINSTANCE | bits 1-12 | reified type args (当前方案) |
| 任何类型 | bit 13 | MMAP flag |

**问题**：Instance 只剩 12 bits 给 type args（2 args × 5-bit tid），无法扩展。
且 JSON 的 shape_id 已占满 bits 2-15，Instance 与 JSON 的编码互斥。

#### 4.2 设计决策：类型参数存储在 XrClass 上（推荐）

单态化后每个泛型特化生成独立的 `XrClass`（如 `Box$i64`），其 type args
在编译期已确定且不变。因此 **type args 是 class 级信息，不是 instance 级信息**。

```c
struct XrClass {
    ...
    uint8_t mono_type_argc;       // 0 if not monomorphized
    XrTypeId *mono_type_args;     // heap array, NULL if argc == 0
    ...
};
```

**收益**：
- 不受 GC header 位数限制，可存储任意数量的 type args
- 每个 XrTypeId 用完整 8 bits（支持 256 种类型）
- 零运行时 per-instance 开销
- 删除 `OP_INST_TYPE_ARGS` 操作码，简化 IR/VM
- Instance 的 `gc.extra` 完全释放，可用于将来的 shape/IC 优化

**Reflect.typeOf 查询**：直接读 `inst->klass->mono_type_args`。

#### 4.3 OP_INST_TYPE_ARGS 废弃

既然 type args 已在 class 上，不再需要 per-instance 运行时设置。
`new Box<int>(42)` 和 `new Box(42)` 的区别在 mono pass 已决定：
- 前者 → `new Box$i64(42)`（class 上有 `mono_type_args = [XR_TID_INT]`）
- 后者 → `new Box(42)`（class 上 `mono_type_argc = 0`）

#### 4.4 GC Header 扩展备选方案（将来可选）

如果未来需要 per-instance 差异化信息（如同一 class 的不同 instance 携带
不同 type args），可将 `extra` 从 16 bits 扩展到 32 bits：

```c
// objsize 以 8B 为粒度 → uint16_t 可表示 0-512KB
typedef struct XrGCHeader {
    struct XrGCHeader *gc_next;  // 8B
    uint8_t type;                // 1B
    uint8_t marked;              // 1B
    uint32_t extra;              // 4B (原 2B → 4B, 30 bits available)
    uint16_t objsize_units;      // 2B (值 × 8 = 实际字节)
} XrGCHeader;                    // 仍然 16B

#define XR_GC_OBJSIZE(gc) ((uint32_t)(gc)->objsize_units * 8)
```

此方案当前不实施，仅作为架构储备记录。

### 5. 单态化器修正

#### 5.1 注入位置：紧跟原始声明之后（已修复）

#### 5.2 `type_args` 生命周期规则

| AST 节点类型 | mono pass 后 type_args | 原因 |
|---|---|---|
| `NewExprNode` | 清零 | type args 已在 class 的 mono_type_args 中 |
| `CallExprNode` | 清零 | 函数调用无需 reified type info |
| `StructLiteralNode` | 清零 | struct 的 type args 也在 class 上 |

#### 5.3 修复 `XrTypeRef**` / `XrType**` 混淆

mono collector 的 API 应接受 `XrTypeRef**`（语法层类型），内部用 `xr_mono_type_tag_from_tref` 转换：

```c
// Before (wrong):
const char *xa_mono_collector_add(XaMonoCollector *c, const char *generic_name,
                                  XrType **type_args, int type_arg_count);

// After (correct):
const char *xa_mono_collector_add(XaMonoCollector *c, const char *generic_name,
                                  XrTypeRef **type_args, int type_arg_count);
```

### 6. Class Builder 中设置 `generic_origin` 和 `display_name`

单态化后的 class 在 IR lowering 时，需要携带 generic_origin 信息。方案：

1. `ClassDeclNode` 增加 `generic_origin_name` 字段（mono pass 设置）
2. IR lowerer emit `OP_CLASS_CREATE_FROM_DESCRIPTOR` 时，携带 origin class name
3. VM 执行 class creation 时，通过 shared variable lookup 找到 origin class 并设置 `generic_origin`
4. `display_name` 从 `name` 中截取 `$` 前缀

或者更简单：

1. mono pass 在 cloned class AST 上标记 `is_monomorphized = true` + `origin_name = "Box"`
2. class builder finalize 时：
   - `cls->display_name = origin_name`（从 descriptor 传入）
   - `cls->generic_origin` 延迟绑定（runtime lookup by name）

### 7. 运行时 `generic_origin` 绑定时机

问题：mono class `Box$i64` 创建时，原始 `Box` class 可能尚未创建（如果原始泛型类声明不生成 runtime class）。

**关键决策：原始泛型类是否需要在运行时存在？**

- **方案 A**：原始泛型类不存在于运行时（纯编译期概念）
  - `is Box` 改为编译器静态分析，检查 class name prefix
  - 优点：零运行时开销
  - 缺点：`is` 只能用于编译期已知的类型，动态 `is` 检查无法工作

- **方案 B（推荐）**：原始泛型类作为"骨架类"存在于运行时
  - 不可实例化，但可作为 `is` 的检查目标
  - `generic_origin` 指向它
  - 优点：`is` 在动态场景中正确工作
  - 缺点：每个泛型声明多一个 XrClass 对象（开销极小）

选择 **方案 B**：原始泛型类保留为骨架类（无 constructor、无 fields，仅有 name 和 flags）。

---

## 实现步骤

### Step 1: XrClass 结构扩展

文件：`src/runtime/class/xclass.h`

```c
struct XrClass {
    ...
    const char *name;
    const char *display_name;       // NEW: user-visible name (NULL = same as name)
    struct XrClass *generic_origin; // NEW: for mono classes, points to skeleton
    struct XrClass *super;
    ...
    uint8_t mono_type_argc;         // NEW: 0 if not monomorphized
    XrTypeId *mono_type_args;       // NEW: concrete type IDs (heap array)
    ...
};
```

新增标志位：
```c
#define XR_CLASS_GENERIC_SKELETON (1 << 8)  // Uninstantiable generic template
#define XR_CLASS_MONOMORPHIZED   (1 << 9)   // Generated by monomorphization
```

### Step 2: `xr_class_display_name` + `xr_class_instanceof` 扩展

文件：`src/runtime/class/xclass.h`

### Step 3: ClassDeclNode 扩展

文件：`src/frontend/parser/xast_nodes_decl.h`

```c
typedef struct ClassDeclNode {
    ...
    bool is_monomorphized;          // NEW: set by mono pass
    const char *generic_origin_name; // NEW: "Box" for Box$i64
    XrTypeRef **mono_type_args;     // NEW: concrete type refs
    int mono_type_arg_count;        // NEW
} ClassDeclNode;
```

### Step 4: Mono pass 标记 cloned class

文件：`src/frontend/analyzer/xanalyzer_mono.c`

- `inject_mono_decls` 中，cloned class 设置 `is_monomorphized = true`
- 保留 `generic_origin_name`
- 保留原始 `type_args` 的解析后类型（`mono_type_args`）

### Step 5: IR lowerer emit class creation with origin info

文件：`src/ir/xi_lower_stmt.c`

- 原始泛型类（`type_param_count > 0`）emit 骨架类（标记 `XR_CLASS_GENERIC_SKELETON`）
- 单态化类（`is_monomorphized = true`）emit 时携带 origin name + type args

### Step 6: VM class creation 绑定 `generic_origin`

文件：`src/vm/xvm_dispatch_object.inc.c` 或 `src/vm/xvm_cold_object.c`

- 新增 opcode 或扩展现有 descriptor：设置 `generic_origin`
- 通过 shared variable name lookup 找到 skeleton class

### Step 7: Mono collector 接口修正

文件：`src/frontend/analyzer/xanalyzer_mono.c`, `xanalyzer_mono.h`

- `xa_mono_collector_add` 改为接受 `XrTypeRef**`
- 内部 `compute_rep_signature` 改用 `XrTypeRef.kind` 而非 `XrType.kind`
- 消除所有 `-Wincompatible-pointer-types` 警告

### Step 8: Reflect.typeOf 优化

文件：`src/runtime/class/xreflect_api.c`

- 优先从 `inst->klass->mono_type_args` 读取（始终正确）
- GC header 快速路径作为可选优化保留
- `display_name` 替代 `$` 截断逻辑

### Step 9: 全局替换 `klass->name` → `xr_class_display_name(klass)`

涉及文件：
- `xinstance.c` — warning/debug print
- `xdap_debug.c` — DAP variable display
- `xvm_cold_object.c` — error messages
- `xreflect_type.c` — Reflect.Type.name
- `xreflect_registry.c` — type registry display（lookup 仍用 internal name）

### Step 10: 删除 OP_INST_TYPE_ARGS 及 GC header type args 编码

type args 完全存储在 XrClass 上后，以下代码应删除：

- `xi_lower_expr.c` — `XI_INST_TYPE_ARGS` emit 逻辑 + `tref_to_tid`
- `xi_emit.c` — `XI_INST_TYPE_ARGS` → `OP_INST_TYPE_ARGS` 编码
- `xvm_dispatch.inc.c` — `vmcase(OP_INST_TYPE_ARGS)` handler
- `xgc_header.h` — `XR_INST_TYPE_ARGC` / `XR_INST_TYPE_ARG0` / `XR_INST_SET_TYPE_ARGS` 宏
- `xdeep_copy.c` — propagate type args 的 gc.extra 拷贝逻辑
- `xvm_dispatch_convert.inc.c` — instance copy 的 gc.extra 传播
- `xreflect_api.c` — 从 gc.extra 读取 type args 的代码（改读 class）

Instance 的 `gc.extra` bits 1-12 恢复为 spare，供将来 inline cache / shape 使用。

### Step 11: 修复 rep-sharing 对 class/struct 泛型的误合并

文件：`src/frontend/analyzer/xanalyzer_mono.c`, `xanalyzer_mono.h`

- `XaMonoInstance` 增加 `bool is_class_generic` 字段
- `xa_mono_collector_add` 增加 `bool is_class_generic` 参数
- 当 `is_class_generic=true` 时，去重使用 `strcmp(mangled_name)` 而非 `rep_signature`
- 当 `is_class_generic=false` 时（函数泛型），保留现有 rep-sharing 行为
- `collect_instantiation_sites` 中根据 registry 的 `is_class` 标记传递参数
- 更新 `xa_mono_collector_lookup` 同理

### Step 12: AOT class runtime helpers

文件：`src/aot/xrt.h`, `src/aot/xrt_class.c`（新建）, `src/aot/xi_cgen.c`

- 定义 `XrtClassDesc` 结构体（static class descriptor for AOT）
- 实现 `xrt_class_create(XrtClassDesc*)` — 从 descriptor 创建 XrClass
- 实现 `xrt_is_class(XrValue, XrClass*)` — class instanceof 检查
- 实现 `xrt_is_class_or_generic(XrValue, XrClass*)` — 含 generic_origin 检查
- `xi_cgen.c` 的 `XI_CLASS_CREATE` 生成 static descriptor + `module_init()` 调用
- `xi_cgen.c` 的 `XI_IS` 扩展 class 类型检查路径

### Step 13: 测试覆盖补全

- T9: struct 泛型 `is` + `Reflect.typeOf`
- T10: rep-sharing 引用类型不误合并验证
- AOT 回归测试覆盖 class `is` + generic `is`

---

## 测试矩阵

### 必须通过的测试

```xray
// T1: instanceof with generic
let b = new Box<int>(42)
assert(b is Box)             // generic_origin check
assert(b is Box<int>)        // exact match (if supported)

// T2: Reflect.typeOf
assert_eq(Reflect.typeOf(b), "Box<int>")

// T3: 泛型 + 继承
class TypedBox<T> extends Box<T> {
    constructor(v: T) { super(v) }
}
let tb = new TypedBox<int>(1)
assert(tb is Box)            // inheritance + generic_origin
assert(tb is TypedBox)

// T4: 嵌套泛型
let nested = new Box<Array<int>>(new Array<int>())
assert_eq(Reflect.typeOf(nested), "Box<Array>")  // tid 限制

// T5: 错误消息不含 $
// (verify via stderr capture that error messages use display_name)

// T6: 跨模块泛型
// module_a.xr: export class Stack<T> { ... }
// module_b.xr: import "./module_a" as a
//              let s = new a.Stack<int>()
//              assert(s is a.Stack)

// T7: 多个类型参数
class Triple<A, B, C> { ... }
let t = new Triple<int, string, bool>(1, "x", true)
assert_eq(Reflect.typeOf(t), "Triple<int, string, bool>")

// T8: 无类型参数的泛型实例
let plain = new Box(42)
assert(plain is Box)
assert_eq(Reflect.typeOf(plain), "Box")

// T9: Struct 泛型
struct Pair<A, B> {
    first: A
    second: B
}
let p = Pair<int, string>{first: 1, second: "x"}
assert(p is Pair)
assert_eq(Reflect.typeOf(p), "Pair<int, string>")

// T10: Rep-sharing 正确性（引用类型不误合并）
class Wrapper<T> {
    value: T
    constructor(v: T) { this.value = v }
}
let ws = new Wrapper<string>("hello")
let wa = new Wrapper<Array<int>>(new Array<int>())
assert_eq(Reflect.typeOf(ws), "Wrapper<string>")
assert_eq(Reflect.typeOf(wa), "Wrapper<Array>")
// 两者必须是不同的 XrClass（不能被 rep-sharing 合并）
```

---

## VM / JIT / AOT 三后端统一设计

### 架构全景：单态化在分叉点之前完成

```
Source → Parser → AST → Analyzer
                           ↓
                     xa_mono_pass            ← 泛型消除在此完成（共享）
                           ↓
                    Monomorphized AST
                           ↓
                       xi_lower              ← 共享 IR 生成
                           ↓
                      Xi IR (typed SSA)
                           ↓
              ┌────────────┼────────────┐
              │            │            │
         xi_emit      xi_to_xm      xi_cgen
              │            │            │
          Bytecode      XmFunc       C source
              │          (JIT SSA)       │
           VM解释        ARM64/x64    system CC
              │         native code      │
          (Tier 0)      (Tier 1/2)   native binary
```

**关键保证**：`xa_mono_pass` 在管线分叉之前执行。三个后端看到的是**完全相同的
单态化后 Xi IR**。所有泛型相关的 AST 变换（clone → substitute → inject → rewrite）
只发生一次，不存在后端差异。

### 各后端的泛型处理路径

#### VM（xi_emit → Bytecode → 解释执行）

| Xi IR 操作 | 生成的字节码 | 运行时行为 |
|------------|-------------|-----------|
| `XI_CLASS_CREATE` (skeleton) | `OP_CLASS_CREATE` + `XR_CLASS_GENERIC_SKELETON` flag | 创建骨架 XrClass，不可实例化 |
| `XI_CLASS_CREATE` (mono) | `OP_CLASS_CREATE` + descriptor 带 origin_name | 创建 mono XrClass，绑定 `generic_origin` |
| `XI_IS` | `OP_IS` | 调用 `xr_class_instanceof`（含 generic_origin 检查） |
| `XI_INST_TYPE_ARGS`（将删除） | `OP_INST_TYPE_ARGS`（将删除） | 设置 gc.extra（将删除） |

VM 路径**零额外开销**：`xr_class_instanceof` 增加一行 `generic_origin` 指针比较，
位于现有 `cls == target` 之后。骨架类创建只在初始化时执行一次。

#### JIT（xi_to_xm → XmFunc → 机器码）

当前状态（`xi_to_xm.c:937-939`）：

```c
case XI_IS:
case XI_AS:
    return lower_call(ctx, blk, v);  // deopt 到 VM runtime
```

JIT 对 `XI_IS`、`XI_CLASS_CREATE` 均 **fallback 到 runtime call**。
这意味着：
- `is` 检查通过 `xr_class_instanceof` — 与 VM 走同一函数
- class creation 通过 `xr_jit_runtime_call` — 与 VM 走同一 class builder
- **语义自动一致**，无需 JIT 单独实现泛型逻辑

**JIT 优化机会（AOT 优先后的可选项）**：

```
Tier 1 (当前)：XI_IS → lower_call → runtime bridge → xr_class_instanceof
                开销：call + branch，但 is 检查本身不在热循环中

Tier 2 (将来)：XI_IS → inline guard:
    if (obj->klass == cached_class) return true;    // monomorphic IC
    if (obj->klass->generic_origin == target) return true;
    fallback to xr_class_instanceof
```

Tier 2 优化只有在 `is` 检查成为 profiling 热点时才值得做。
当前 JIT 的 runtime call 方案已足够——`is` 检查极少出现在内循环。

#### AOT（xi_cgen → C → 编译器优化）

当前 `xi_cgen.c` 对 `XI_IS` 的处理（行 786-833）是**纯 tag 比较**：

```c
case XI_IS:
    // int: v.tag == XR_TAG_I64
    // float: v.tag == XR_TAG_F64
    // string: XR_IS_STR(v)
    // class: tag == class_tag (当前缺失 generic_origin 检查)
```

**AOT 重构后的 `XI_IS` 处理**：

```c
case XI_IS:
    if (target is primitive type) {
        // 编译期常量折叠：tag == TAG_xxx → 单指令比较
        emit: (v.tag == XR_TAG_I64)
    }
    else if (target is class) {
        if (target has XR_CLASS_GENERIC_SKELETON flag) {
            // 泛型骨架类：需检查 generic_origin
            emit: xrt_is_class_or_generic(v, &target_class_ref)
        } else {
            // 普通类：直接 class pointer 比较
            emit: xrt_is_class(v, &target_class_ref)
        }
    }
```

AOT 的优势：
- **基本类型 `is`**：编译为单指令 `tag == CONST`，C 编译器可进一步优化
- **class `is`**：编译为 `xrt_is_class()` 调用，可被 LTO inline
- **generic `is`**：编译为 `xrt_is_class_or_generic()`，逻辑与 VM 的
  `xr_class_instanceof` 一致但可 inline
- `XrClass` 在 AOT 中是**静态分配的全局变量**，`generic_origin` 指针在
  初始化阶段（`module_init()`）设置，运行时无 lookup 开销

### 共享运行时层：三后端的统一保障

```
三后端 → 统一 runtime 层（不分 VM/JIT/AOT）
  ├─ XrClass              结构一致，同一头文件定义
  ├─ xr_class_instanceof  同一函数实现
  ├─ xr_class_display_name 同一 inline 函数
  ├─ mono_type_args        同一内存布局
  └─ generic_origin        同一指针语义
```

这是 xray 泛型设计一致性的**根本保证**：所有运行时类型信息（`XrClass`、
`generic_origin`、`mono_type_args`、`display_name`）在 C 结构体层面定义一次，
三个后端共享同一组头文件和实现。

### AOT 优先原则的具体体现

| 设计决策 | AOT 考量 | VM/JIT 影响 |
|---------|---------|------------|
| type args 存 XrClass 而非 GC header | AOT 中 XrClass 是 static 全局变量，直接编码；GC header 是运行时分配，AOT 无法在编译期填充 | VM/JIT 也受益：删除 OP_INST_TYPE_ARGS，简化 IR |
| generic_origin 用指针而非名字 | AOT 中用 `&Box_class`（全局变量地址），零运行时开销 | VM 用 shared variable lookup（一次），JIT 同 VM |
| display_name 编译期截取 | AOT 中 display_name 作为 string literal 编入 .rodata | VM/JIT 同样受益 |
| 骨架类静态分配 | AOT 的骨架类 = `static XrClass Box_class = { .flags = SKELETON, ... }` | VM 运行时创建，开销可忽略 |
| `is` 检查内联 | AOT：基本类型 `is` 编译为 `tag == CONST`（一条 cmp 指令）；class `is` 编译为 inline 函数调用，LTO 可进一步优化 | VM：一次 `OP_IS` dispatch；JIT：runtime call（可选 IC 优化） |

### 性能排序（设计目标）

```
AOT > JIT Tier 2 > JIT Tier 1 > VM

AOT:       is 检查 → 1 cmp (primitive) / inline call (class)
JIT Tier2: is 检查 → monomorphic IC guard → inline compare
JIT Tier1: is 检查 → runtime call (xr_class_instanceof)
VM:        is 检查 → opcode dispatch → xr_class_instanceof

class creation:
AOT:       static initializer in module_init() → 零运行时 alloc
JIT/VM:    runtime XrClassBuilder → heap alloc (仅首次执行)
```

### 不需要后端特殊处理的操作

以下泛型相关操作在**所有后端中行为完全相同**，无需任何特殊化：

- **`new Box<int>(42)`** → mono pass 已变为 `new Box$i64(42)`，所有后端看到的是普通的 `XI_CALL`（构造函数调用），无泛型痕迹
- **`Reflect.typeOf(b)`** → 读 `inst->klass->mono_type_args` + `inst->klass->display_name`，纯 runtime C 函数，三后端共享
- **方法调用** → 单态化后 `Box$i64.get()` 就是普通方法调用，dispatch 路径与非泛型类完全一致
- **字段访问** → 单态化后字段布局在编译期确定，三后端使用相同的 field descriptor

---

## 文档审计：发现的设计缺口

### ⚠️ 关键缺陷：Rep-sharing 破坏 reified type args

**现象**：`compute_rep_signature`（`xanalyzer_mono.c:884`）使用 `xr_type_to_slot_type`
计算去重签名。而 `xr_type_to_slot_type`（`xtype.h:346-356`）将**所有引用类型**
（string、array、map、instance、channel、class 等）统一映射为 `XR_SLOT_PTR = 3`。

```
Box<string>       → mangled "Box$str"     → rep_sig = 3 (PTR)
Box<MyClass>      → mangled "Box$MyClass" → rep_sig = 3 (PTR)
Box<Array<int>>   → mangled "Box$arr"     → rep_sig = 3 (PTR)
```

`xa_mono_collector_add` 按 `(generic_name, rep_signature, type_arg_count)` 去重。
当 `Box<string>` 先注册后，`Box<MyClass>` 被视为重复，返回 `"Box$str"` 的 mangled name。

**后果**：
- `new Box<MyClass>(obj)` 被改写为 `new Box$str(obj)` — **类错误**
- `Reflect.typeOf` 返回 `"Box<string>"` 而非 `"Box<MyClass>"` — **类型信息丢失**
- `mono_type_args` 只反映第一个注册的类型 — **所有后续类型都错**

注意：基本类型不受此影响。int (`XR_SLOT_I64=1`)、float (`XR_SLOT_F64=2`)、
bool (`XR_SLOT_BOOL=4`) 各有独立的 slot type，不会被误合并。

**根因**：`xr_mono_type_tag` 为每种类型生成了不同的 tag（"str" vs "MyClass"），
但 `compute_rep_signature` 把它们都压缩为同一个 PTR 值，绕过了语义区分。

**修复方案**：

```
方案 A（推荐）：class/struct 泛型按 mangled name 去重，禁用 rep-sharing
  - xa_mono_collector_add 增加 is_class_generic 参数
  - 如果 is_class_generic=true，改用 mangled_name 字符串匹配做去重
  - 函数泛型保留 rep-sharing（函数无 XrClass identity）
  优点：语义正确，mono_type_args 精确
  代价：引用类型泛型会生成更多类（Box<string>、Box<MyClass> 各一份）

方案 B：完全禁用 rep-sharing
  - compute_rep_signature 直接返回 mangled name 的 hash
  - 所有泛型（函数 + class）都按精确类型去重
  优点：简单统一
  代价：函数泛型的代码膨胀增加（但 xray 函数泛型使用较少）

方案 C：class 泛型跳过 rep-sharing，函数泛型保留
  - 在 collect_instantiation_sites 阶段区分 class vs function
  - 给 XaMonoInstance 加 bool is_class_generic 标记
  - add/lookup 时根据标记选择去重策略
```

**选择方案 A**。实施时修改 `xa_mono_collector_add`：当 `is_class_generic=true`
时，用 `strcmp(mangled_name)` 去重代替 `rep_signature` 比较。

### 缺口 2：Struct 泛型的 `is` / `generic_origin` 覆盖

`struct_decl` 复用 `ClassDeclNode` 布局（`xast_nodes.h:99`），已有 `type_params`。
但设计文档的步骤全部聚焦 class，未提及 struct：

- struct 泛型是否需要 `generic_origin`？ → **是**，`Pair<int, string> is Pair` 需要
- struct 泛型是否需要 `display_name`？ → **是**，`Reflect.typeOf` 需要
- struct 泛型是否需要 `mono_type_args`？ → **是**，同理

struct 和 class 在运行时都对应 `XrClass` 对象（struct 通过 `XR_CLASS_STRUCT` flag
区分）。因此 **struct 泛型的设计完全等同于 class 泛型**，步骤 1-10 自动覆盖，
无需单独设计。但实现步骤和测试矩阵需要**显式包含 struct 用例**。

### 缺口 3：`Reflect.typeOf` 输出格式规范

当前实现（`xreflect_api.c:429-449`）通过 `$` 截断 + GC header type args 拼装：

```c
const char *dollar = strchr(cls_name, '$');
int name_len = dollar ? (int)(dollar - cls_name) : (int)strlen(cls_name);
int argc = XR_INST_TYPE_ARGC(&inst->gc);
// → "Box<int>" or "Box<int, string>"
```

新设计改为读 `inst->klass->mono_type_args`，需要明确格式：

| 场景 | 输出 | 说明 |
|------|------|------|
| `Box<int>` | `"Box<int>"` | 单参数 |
| `Map<string, int>` | `"Map<string, int>"` | 多参数，逗号+空格分隔 |
| `Box<Array<int>>` | `"Box<Array>"` | 嵌套泛型仅显示外层名（XrTypeId 无法表达嵌套参数） |
| `Triple<int, string, bool>` | `"Triple<int, string, bool>"` | 3+ 参数 |
| `Box` (非泛型实例) | `"Box"` | `mono_type_argc == 0` |

**type arg 名称来源**：`mono_type_args[i]` 是 `XrTypeId`，通过 `xr_typeid_name()`
转为用户可读名。对于用户定义类（`XR_TID_INSTANCE`），需要额外存储 class name。

**决策**：保持当前 `xr_typeid_name()` 方案。对于嵌套泛型和用户类，
`XrTypeId` 精度不足是已知限制（见设计目标 #5 的说明）。
将来如需支持完整泛型签名，可扩展 `mono_type_args` 为 `XrType**` 而非 `XrTypeId*`。

### 缺口 4：`mono_type_args` 内存所有权

`XrClass.mono_type_args` 是 `XrTypeId*`（heap array）。生命周期规则：

| 执行模式 | 分配时机 | 所有者 | 释放时机 |
|---------|---------|--------|---------|
| VM/JIT | class builder finalize | XrClass（GC 管理） | class 被 GC 回收时 |
| AOT | `module_init()` 静态初始化 | static storage | 进程退出 |

**VM/JIT 实现**：class builder 中用 `xr_malloc(argc * sizeof(XrTypeId))`
分配，赋给 `cls->mono_type_args`。class 析构函数需要 `xr_free(cls->mono_type_args)`。

**AOT 实现**：`xi_cgen` 生成 `static const XrTypeId Box_i64_type_args[] = { XR_TID_INT };`，
class 初始化时赋指针。无需动态释放。

### 缺口 5：AOT class 创建的当前现实

文档中 VM/JIT/AOT 统一设计一节提到"AOT 的骨架类 = `static XrClass`"，
这是**目标状态，非当前实现**。

当前 AOT 对 `XI_CLASS_CREATE` 的处理（`xi_cgen.c:1264`）：

```c
case XI_CLASS_CREATE:
    fprintf(out, "XR_NULL_VAL /* class descriptor */");
    break;
```

即 AOT 当前**跳过 class 创建**，只处理 constructor call 路径。这意味着：
- AOT 中 class 不作为 first-class 对象存在
- `is` 检查对 class 类型不可用
- `Reflect.typeOf` 对 class instance 不可用

**修复方案**（实施步骤中需要新增）：

1. `xi_cgen` 为每个 class 生成一个 `static XrtClassDesc` 结构体
2. `module_init()` 中调用 `xrt_class_create(&desc)` 初始化
3. mono class 的 desc 包含 `generic_origin` 指向骨架 class 的 desc
4. `xrt_is_class(val, class_ptr)` 替代 tag 比较
5. 新增 `xrt.h` 中的 class runtime helper：
   - `xrt_class_create(XrtClassDesc*)` — 从 descriptor 创建 XrClass
   - `xrt_is_class(XrValue, XrClass*)` — class instanceof 检查
   - `xrt_is_class_or_generic(XrValue, XrClass*)` — 含 generic_origin 检查

---

## 风险与边界

1. **代码膨胀**：每种类型组合生成一个 class，但 XrClass 对象很小（~300B）
2. **循环泛型**：`class Node<T> { next: Node<T>? }` 不会无限单态化（Node<int> 只生成一份）
3. **骨架类的 GC**：骨架类不可实例化，但需要被 GC root（shared variable 持有）
4. **AOT 兼容**：mono_type_args 存储在 class 上，AOT 可直接编码为 static initializer
5. **AOT `is` class 检查**：当前 `xi_cgen.c` 的 `XI_IS` 仅做 tag 比较，
   需要扩展支持 class instanceof + generic_origin（通过 `xrt_is_class_or_generic` 调用）
6. **JIT deopt 正确性**：JIT 对 `XI_IS` fallback 到 runtime call，
   deopt snapshot 必须正确恢复 VM 状态（当前已满足，无需额外工作）

---

## 不做的事情（明确排除）

- ❌ 泛型约束（`where T: Comparable`）— 未来独立 feature
- ❌ 协变/逆变（`out T` / `in T`）— 未来独立 feature
- ❌ type erasure 方案 — 单态化是 xray 的设计选择
- ❌ 运行时泛型实例化 — 所有泛型在编译期决定

---

## 参考语言源码分析

### Rust — `compiler/rustc_monomorphize/src/collector.rs`

**核心架构**：图驱动的 MonoItem 收集器

```
Phase 1: 从 HIR 发现 roots（公共非泛型函数）
Phase 2: 从 roots 出发遍历 MIR，发现 uses（含泛型实例化）
结果:    MonoItem 有向图 → 每个节点生成一个 LLVM artifact
```

**xray 可借鉴的关键设计**：

1. **`generic_owner` 概念**：每个 `FuncInstance` 存储 `generic_owner: DefId`，
   即原始泛型声明的 identity。这不是字符串名，而是编译器内部 ID。
   → xray 应用**指针引用**（`XrClass *generic_origin`）而非名字查找

2. **无限特化循环检测**：Rust 有 `RecursionLimit` + 深度计数器，
   防止 `Box<Box<Box<...>>>` 导致编译器 hang。
   → xray 应添加 `XR_MONO_MAX_DEPTH`（建议 128）

3. **跨 crate 单态化**：当泛型定义在 crate A、使用在 crate B 时，
   B 负责生成 MonoItem（如果 MIR 可用）。
   → xray 跨模块泛型需要确保 mono pass 能看到被 import 的泛型声明

4. **Lazy vs Eager 策略**：Lazy 只实例化被使用的；Eager 实例化所有可能的。
   → xray 当前 = Lazy（仅收集有 call site 的），正确

### Swift — `include/swift/ABI/GenericContext.h` + `lib/IRGen/ClassMetadataVisitor.h`

**核心架构**：运行时 Type Metadata + 可选特化

```
GenericContextDescriptorHeader:
  NumParams          — 泛型参数数量
  NumRequirements    — 约束数量
  NumKeyArguments    — "key" 参数数量（参与类型 identity）

ClassMetadata 布局:
  [标准类头]
  [父类 vtable entries]
  [**generic type argument pointers**]  ← 关键：类型参数直接嵌入 metadata
  [field offsets]
  [vtable entries]
```

**xray 可借鉴的关键设计**：

1. **Type args 存在 class metadata 中**：Swift 的 `addGenericFields()` 把
   每个泛型参数的 metadata 指针直接放在 class metadata 的 trailing fields。
   → 验证了 xray 的 `mono_type_args` 存在 XrClass 上的设计正确性

2. **Type Descriptor 分离**：Swift 有两层：
   - `TypeDescriptor`（编译期常量，描述泛型签名）
   - `Metadata`（运行时实例化，包含具体 type args）
   → xray 的 `ClassDeclNode.is_monomorphized` + `XrClass.mono_type_args` 对应此双层

3. **特化深度/宽度限制**：`TypeDepthThreshold = 50`, `TypeWidthThreshold = 2000`
   → xray 应添加限制，防止病态泛型导致编译爆炸

4. **`is` 检查语义**：Swift 的 `DynamicCast.cpp` 在做类型检查时，
   会比较 metadata pointer（包含具体 type args）。
   `Box<Int>` 和 `Box<String>` 是**不同的 metadata**，因此不能互相 `as`。
   但 `Box<Int>` **is** `Box`（擦除参数后的基类检查）是通过继承链实现的。
   → 验证了 xray 的 `generic_origin` 设计：`Box$i64 is Box` 通过 `generic_origin == Box`

### Zig — `src/InternPool.zig`

**核心架构**：InternPool 去重 + comptime 即泛型

```zig
pub const FuncInstance = struct {
    analysis: FuncAnalysis,
    owner_nav: Nav.Index,
    ty: Index,
    branch_quota: u32,
    generic_owner: Index,    // ← 指向原始泛型声明
    // trailing: comptime_args: []Index  ← 具体类型参数
};
```

**xray 可借鉴的关键设计**：

1. **无 name mangling**：Zig 不用 `Box$i64` 这样的 mangled name。
   每个实例由 `(generic_owner, comptime_args)` 元组唯一标识，
   通过 InternPool hash map 去重。名字只是 debug 用途。
   → xray 的 mangled name 应降级为**内部 debug 标识**，不进入任何用户可见路径

2. **`generic_owner` 是直接引用**：不是字符串，不是索引，而是指向
   原始 `func_decl` 的 `Index`。
   → 强化 xray 设计：`XrClass.generic_origin` 是 `XrClass*` 指针，零查找开销

3. **类型是 first-class value**：Zig 中 `type` 本身是 comptime 值，
   泛型参数和普通 comptime 参数无区别。
   → xray 可以考虑将来统一 comptime 和泛型（但当前先完成单态化修正）

4. **去重在 InternPool 层面**：相同 `(generic_owner, comptime_args)` 只创建一份。
   → xray 的 `xa_mono_collector` 已做签名去重，设计正确

---

## 三语言对比 & xray 设计验证

| 设计点 | Rust | Swift | Zig | xray（采纳） |
|--------|------|-------|-----|-------------|
| 泛型策略 | 编译期单态化 | 运行时 metadata + 可选特化 | comptime 全特化 | 编译期单态化 |
| origin 引用方式 | `DefId`（编译器内部 ID） | TypeDescriptor 指针 | `Index`（InternPool ref） | **`XrClass*` 指针** |
| type args 存储 | 无需（完全单态化后类型擦除） | class metadata trailing fields | InternPool trailing `comptime_args` | **`XrClass.mono_type_args`** |
| `is` 检查 | trait object vtable（不同概念） | metadata pointer 比较 + 继承链 | 编译期完成 | **`generic_origin` 指针匹配** |
| name mangling | 复杂 symbol mangling | 有 mangling 但 runtime demangle | 无 mangling | **仅内部使用，display_name 去除** |
| 递归/深度限制 | `RecursionLimit`（默认 128） | `TypeDepthThreshold = 50` | `branch_quota`（默认 1000） | **新增 `XR_MONO_MAX_DEPTH = 64`** |
| 去重 | 全局 MonoItem set | metadata cache | InternPool hash | **`xa_mono_collector` 签名去重** |
| 跨模块 | 跨 crate 实例化（MIR 传递） | 跨 module metadata 实例化 | 跨 package comptime eval | **待实现** |

---

## 从三语言中采纳的新设计点

### A. 递归深度限制（from Rust + Swift）

```c
#define XR_MONO_MAX_DEPTH 64

// In xa_mono_pass: track instantiation depth
if (depth > XR_MONO_MAX_DEPTH) {
    xr_diag_error(loc, "generic instantiation depth exceeds limit (%d)", XR_MONO_MAX_DEPTH);
    return;
}
```

### B. `generic_origin` 必须是指针而非名字查找（from Zig）

Zig 用 InternPool Index 直接引用 owner。xray 用 `XrClass*` 指针。
**禁止通过 shared variable name 做运行时 lookup**——这是 O(n) 且脆弱。

绑定时机：mono class 的 class builder finalize 在骨架类**之后**执行，
此时骨架类已存在于 shared variable slot 中，可直接取指针。

### C. Mangled name 降级为 debug-only（from Zig）

Zig 完全不 mangle。xray 保留 mangled name 作为内部唯一标识（用于符号查找、
链接），但**任何面向用户的路径**必须使用 `display_name`。

### D. 特化去重验证（from all three）

三种语言都做去重。xray 的 `xa_mono_collector` 已通过签名 hash 去重，
但需要验证：
- 相同 `(generic_name, type_args)` 不会生成两个 AST clone
- 跨文件 import 同一泛型类的不同模块不会重复单态化
