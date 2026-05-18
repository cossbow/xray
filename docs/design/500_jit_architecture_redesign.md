# 500 - Xray JIT 架构重新设计

## 一、问题陈述

Xray 的 JIT 在过去数月中反复出现隐蔽的 tag 传播 bug（Pattern 24/25）、寄存器分配冲突、优化 pass 错误删除副作用指令等问题。这些 bug 的共同特征是：**编译器不报错，运行时大概率不立即崩溃，但值被悄悄误解释**。

根因分析（见 `docs/engineering/bug_patterns.md`）揭示了 4 个架构性问题：

1. **值与类型分离传递**：payload 和 tag 通过独立管道传递，四层间接旁路
2. **两种返回约定共存**：59 个 `int64_t` helper vs 16 个 `XrJitResult` helper
3. **Builder API 语义重叠**：`braun_write_var` vs `builder_set_slot`
4. **手写汇编无声明式抽象**：遗漏任何一行 tag 传播代码都无编译错误

本文档通过分析 V8、LuaJIT、Dart 三个成熟 JIT 实现的源码，提取其核心设计模式，为 xray 设计针对性的架构改进方案。

---

## 二、成熟 JIT 实现源码分析

### 2.1 V8 TurboFan/Turboshaft

**源码位置**：`v8/src/objects/tagged.h`, `v8/src/compiler/`, `v8/src/codegen/`

#### 值表示：Tagged Pointer

```
V8 Tagged<T> — 指针大小的值，LSB 编码类型：
  Smi:        |____int31_value___0|     (tag=0)
  HeapObject: |______address____01|     (tag=01)
  Weak:       |______address____11|     (tag=11)
```

**核心设计**：值自带类型信息。`IsSmi()` 只需检查 `ptr_ & 1 == 0`。不需要旁路 tag 通道。

V8 中**不存在** xray 的 `call_result_tag` 概念——因为返回值本身就是 `Tagged<Object>`，类型信息随值流动。

#### Runtime 调用约定

V8 的 runtime helper（`runtime_entry.h`）统一返回 `Tagged<Object>`。CodeStubAssembler 生成的 stub 也返回 `Tagged<Object>`。codegen 框架通过 `interface-descriptors.h` 声明每个 builtin 的参数/返回类型，**编译时静态检查**。

#### Verifier

V8 有一个 **90K 行的 Verifier**（`v8/src/compiler/verifier.cc`），在每个编译 pass 之后运行，检查：
- 每个 node 的输入类型与声明是否匹配
- 控制流完整性
- 效果链（effect chain）完整性
- 类型推断一致性

**关键启示**：V8 的健壮性来自 **强类型值表示 + 声明式接口 + 全面的验证器**。

---

### 2.2 LuaJIT

**源码位置**：`LuaJIT/src/lj_obj.h`, `LuaJIT/src/lj_ir.h`, `LuaJIT/src/lj_ircall.h`, `LuaJIT/src/lj_snap.c`

#### 值表示：NaN-boxing TValue

```c
// LuaJIT TValue — 64 位，类型编码在 NaN 空间
typedef LJ_ALIGN(8) union TValue {
  uint64_t u64;       // 64 bit pattern overlaps number
  lua_Number n;       // double
  struct {
    int32_t i;        // Integer value
    uint32_t it;      // Internal object tag (overlaps MSW of number)
  };
} TValue;

// GC64 模式：上 13 位必须为 1（NaN），接下来 4 位是 tag，低 47 位是指针
// |1..1|itype|-------GCRef--------|   GC objects
// |1..1|itype|0..0|-----int-------|   integer
//  ------------double--------------   number
```

**核心设计**：tag 嵌入值本身。不可能"忘记设 tag"——因为没有独立的 tag 通道。

#### IRCALLDEF：声明式 C 调用表

```c
// LuaJIT lj_ircall.h — 所有 C 调用在一个表中声明
#define IRCALLDEF(_) \
  _(ANY, lj_str_new,    3, S, STR, CCI_L|CCI_T) \
  _(ANY, lj_tab_new_ah, 3, A, TAB, CCI_L|CCI_T) \
  _(ANY, lj_gc_step_jit,2, S, NIL, CCI_L) \
  ...
//                       ^  ^  ^^^
//                    nargs kind 返回类型！
```

**关键设计**：每个 C helper 的**返回类型在表中静态声明**。汇编后端通过 `CCI_TYPE(ci)` 读取返回类型，自动生成正确的寄存器操作。

**不可能出现 xray 的 Pattern 24 bug**——因为你不能添加一个新 helper 而不声明其返回类型。汇编后端会根据声明的类型自动处理结果。

#### Snapshot 机制

LuaJIT 在每个潜在退出点生成 snapshot，记录：
- 每个 Lua slot 对应哪个 IR ref
- 每个 IR ref 的 IRT 类型（5 bit，嵌入在 IR 指令中）

退出时，`lj_snap_restore()` 根据 IRT 类型重建完整 TValue。**类型信息是 IR 的固有属性**，不是外部旁路。

**关键启示**：LuaJIT 的健壮性来自 **NaN-boxing 值表示 + IRCALLDEF 声明式表 + IR 内嵌类型**。

---

### 2.3 Dart VM

**源码位置**：`dart/runtime/vm/compiler/backend/il.h`, `dart/runtime/vm/compiler/backend/compile_type.h`, `dart/runtime/vm/compiler/backend/il_arm64.cc`

#### 值表示：Tagged Object

Dart 使用与 V8 类似的 tagged pointer：

```
Smi:        LSB=0, 值左移 1 位
HeapObject: LSB=1, 指针 - 1 = 实际地址
```

#### Representation 系统（核心创新）

```c++
// Dart IL: 每个指令声明其输出/输入的 Representation
class Instruction {
  // 该指令产出值的表示形式
  virtual Representation representation() const { return kTagged; }
  // 该指令对第 idx 个输入的要求
  virtual Representation RequiredInputRepresentation(intptr_t idx) const {
    return kTagged;
  }
};
```

当输出的 representation 与下游输入的 `RequiredInputRepresentation` 不匹配时，编译器框架**自动插入 Box/Unbox 指令**。开发者不需要手动传播类型——框架保证一致性。

#### CompileType 格

```c++
// CompileType: 值的编译期类型信息
class CompileType {
  bool is_nullable;    // 可否为 null
  classid_t cid;       // 具体 class ID 或 kDynamicCid
  AbstractType* type;  // 抽象超类型
};
```

CompileType 形成一个格（lattice），有 `None`（底）和 `Dynamic`（顶）。`Union()` 提供 join 操作。类型信息随 IL 指令流动，不需要外部通道。

#### Runtime Entry

所有 runtime 入口在 `runtime_entry_list.h` 中集中声明：

```c++
#define RUNTIME_ENTRY_LIST(V) \
  V(AllocateObject) \
  V(UpdateFieldCid) \
  V(InterruptOrStackOverflow) \
  ...
```

Runtime 调用统一通过 `__ CallRuntime(kXxxRuntimeEntry, nargs)` 发起。返回值总是 tagged object，由 `FlowGraphCompiler` 统一处理寄存器保存/恢复和 safepoint 记录。

**关键启示**：Dart 的健壮性来自 **Representation 声明系统 + 自动 Box/Unbox + CompileType 格 + 集中式 Runtime Entry 声明**。

---

### 2.4 Go Runtime

**源码位置**：`go/src/runtime/runtime2.go`, `go/src/runtime/iface.go`

#### 值表示：静态类型 + 16B interface{}

Go 是**静态类型语言**——大部分变量在编译时类型已知，`int64` 就是 8B，`float64` 就是 8B，不需要统一值表示。

只有 `interface{}`/`any` 需要统一表示，它是 **16 bytes**：

```go
// go/src/runtime/runtime2.go
type eface struct {
    _type *_type         // 8B: type descriptor pointer
    data  unsafe.Pointer // 8B: pointer to value (heap-allocated!)
}
```

#### 装箱开销：堆分配

当值类型存入 `interface{}` 时，Go **堆分配**一个副本：

```go
// go/src/runtime/iface.go — convT64
func convT64(val uint64) (x unsafe.Pointer) {
    if val < uint64(len(staticuint64s)) {
        x = unsafe.Pointer(&staticuint64s[val])  // 小值用静态表优化
    } else {
        x = mallocgc(8, uint64Type, false)        // 大值堆分配！
        *(*uint64)(x) = val
    }
    return
}
```

**关键设计**：Go 依赖静态类型系统避免装箱开销。`interface{}` 场景代价高（16B + 堆分配），但占代码比例很小。

**关键启示**：Go 证明了**16B 统一值表示 + 堆分配**在静态类型语言中是可行的，但需要尽量减少动态值的出现。

---

### 2.5 Mono (C# JIT)

**源码位置**：`mono-mini/mini.h`, `mono-mini/method-to-ir.c`, `mono-mini/jit-icalls.h`

#### 值表示：无统一值类型

**C# JIT 内部没有统一的值表示。** 每种类型独立处理：

```c
// mono-mini/mini.h — JIT 内部栈类型
typedef enum {
    STACK_INV,    // invalid
    STACK_I4,     // int32 → GPR
    STACK_I8,     // int64 → GPR
    STACK_PTR,    // native pointer → GPR
    STACK_R8,     // double → FPR
    STACK_MP,     // managed pointer → GPR
    STACK_OBJ,    // object reference → GPR
    STACK_VTYPE,  // value type → stack/memory
    STACK_R4,     // float → FPR
    STACK_MAX
} MonoStackType;
```

编译时根据 .NET 类型映射到栈类型：

```c
// mono-mini/method-to-ir.c
MonoStackType mini_type_to_stack_type(MonoCompile *cfg, MonoType *t) {
    switch (t->type) {
    case MONO_TYPE_I4: case MONO_TYPE_U4: return STACK_I4;
    case MONO_TYPE_I8: case MONO_TYPE_U8: return STACK_I8;
    case MONO_TYPE_R8:                    return STACK_R8;
    case MONO_TYPE_CLASS: case MONO_TYPE_STRING:
    case MONO_TYPE_OBJECT:                return STACK_OBJ;
    case MONO_TYPE_VALUETYPE:             return STACK_VTYPE;
    ...
    }
}
```

#### Helper 函数：精确原始类型签名

C# JIT 的所有 helper 都用**精确的原始类型**，没有统一值类型：

```c
// mono-mini/jit-icalls.h — 注意：全是精确类型，无统一 "Value"
ICALL_EXPORT gint32  mono_idiv  (gint32 a, gint32 b);   // int÷int→int
ICALL_EXPORT double  mono_fdiv  (double a, double b);    // double÷double→double
ICALL_EXPORT gint64  mono_lldiv (gint64 a, gint64 b);   // long÷long→long
ICALL_EXPORT guint64 mono_lshl  (guint64 a, gint32 s);  // long<<int→long
ICALL_EXPORT gint64  mono_fconv_i8 (double v);           // double→long
```

**对比 xray**：xray 的 helper 混合使用 `int64_t` + tag 旁路。

#### 装箱（Boxing）：堆分配对象

```c
// mono-mini/method-to-ir.c — mini_emit_box
MonoInst* mini_emit_box(MonoCompile *cfg, MonoInst *val, MonoClass *klass, ...) {
    alloc = handle_alloc(cfg, klass, TRUE, context_used);  // 堆分配
    // 将值写入对象偏移 MonoObject 头之后
    EMIT_NEW_STORE_MEMBASE_TYPE(cfg, ins, ..., alloc->dreg,
                                MONO_ABI_SIZEOF(MonoObject), val->dreg);
    return alloc;
}
```

C# 只有 `object` 装箱（`int → object`）时才需要统一表示——通过堆分配 + 对象头解决。JIT 积极优化消除装箱（逃逸分析、标量替换、内联）。

**关键启示**：C# 达到高性能的秘诀**不是 NaN-boxing，而是「静态类型 + JIT 利用类型信息 + 精确类型 helper」**。JIT 内部根本没有"统一值类型"的概念。

---

## 三、五个引擎的共同模式（xray 缺失的）

| 维度 | V8 | LuaJIT | Dart | Go | C# (Mono) | xray 现状 |
|------|-----|---------|------|----|-----------|-----------|
| **值表示** | Tagged Ptr 8B | NaN-boxing 8B | Tagged Ptr 8B | 静态类型；`interface{}` 16B | 静态类型；`object` 堆分配 | 16B fat value，JIT 拆分 raw+tag |
| **C 调用声明** | interface-descriptors | IRCALLDEF 宏表 | runtime_entry_list | 静态类型签名 | **精确类型签名** `gint32/double/gint64` | 散落各文件，类型靠约定 |
| **类型传播** | IR node 内嵌 | IR 内嵌 IRT | Representation | 编译时已知 | **MonoStackType 编译时映射** | 四层间接旁路通道 |
| **统一值需求** | 始终（动态语言） | 始终（动态语言） | 始终（半动态） | 仅 interface{} | **仅 object 装箱** | 始终 |
| **int 范围** | Smi (int31/63) | int32 | Smi (int63) | **int64** | **int64** | **int64** |
| **double** | HeapNumber 堆分配 | NaN 内联 | HeapNumber 堆分配 | 内联 8B | 内联 8B | 内联 16B |

**两大阵营**：

1. **动态语言阵营**（V8/LuaJIT/Dart）：所有值需要统一表示 → Tagged Pointer 或 NaN-boxing → int/double 有限制
2. **静态类型阵营**（Go/C#）：大部分值编译时类型已知 → **不需要统一值表示** → int64/double 完整范围 → 仅在少数动态场景（interface{}/object）使用堆分配

**xray 的定位**：typed scripting，目标 C# 级性能，后期主打 AOT → **应走静态类型阵营路线**。

**核心差异**：五个成熟实现都让**类型信息跟随值或在编译时确定**，而 xray 让类型信息通过**独立的共享内存旁路**传递。

---

## 四、Xray JIT 改进方案

### 设计原则

1. **声明优于约定**（Declaration over Convention）：用编译器可检查的声明替代隐式约定
2. **类型随值流动**（Type Follows Value）：消除旁路 tag 通道
3. **单一 API**（Single API）：消除功能重叠的 API
4. **渐进式改进**（Incremental Improvement）：每个改进独立可验证，不需要大爆炸重写

---

### 改进 A（P0）：CALL_C Helper 声明式表

**灵感来源**：LuaJIT `IRCALLDEF`

创建一个集中式宏表，声明每个 JIT C helper 的元信息：

```c
/* xir_helper_table.h — Central declaration of all JIT C helpers */

/*
 * XIR_HELPER_DEF(name, ret_rep, flags)
 *
 * name:     function name suffix (after xr_jit_)
 * ret_rep:  return representation:
 *           XR_REP_RESULT  — returns XrJitResult (x0=payload, x1=tag)
 *           XR_REP_PTR     — always returns pointer (static tag)
 *           XR_REP_I64     — always returns int64 (static tag)
 *           XR_REP_F64     — always returns double (static tag)
 *           XR_REP_BOOL    — always returns bool (static tag)
 *           XR_REP_VOID    — result not used
 * flags:    XIR_HF_SUSPEND — may suspend (CPS)
 *           XIR_HF_DEOPT   — may request deopt
 *           XIR_HF_GC      — may trigger GC
 *           XIR_HF_THROW   — may throw exception
 */
#define XIR_HELPER_DEF(_) \
  /* Property access */ \
  _(getprop,        XR_REP_RESULT,  XIR_HF_GC|XIR_HF_DEOPT) \
  _(setprop,        XR_REP_VOID,    XIR_HF_GC) \
  _(getfield_ic,    XR_REP_RESULT,  0) \
  _(index_get,      XR_REP_RESULT,  XIR_HF_GC|XIR_HF_DEOPT) \
  _(index_set,      XR_REP_VOID,    XIR_HF_GC) \
  /* Builtins & globals */ \
  _(getbuiltin,     XR_REP_RESULT,  0) \
  _(get_shared,     XR_REP_RESULT,  XIR_HF_GC) \
  _(set_shared,     XR_REP_VOID,    XIR_HF_GC) \
  /* Type operations */ \
  _(is_type,        XR_REP_BOOL,    0) \
  _(checktype,      XR_REP_BOOL,    0) \
  _(typeof,         XR_REP_PTR,     0) \
  /* String operations */ \
  _(chr,            XR_REP_PTR,     XIR_HF_GC) \
  _(substring,      XR_REP_PTR,     XIR_HF_GC) \
  _(str_repeat,     XR_REP_PTR,     XIR_HF_GC) \
  _(tostring,       XR_REP_RESULT,  XIR_HF_GC) \
  _(strbuf_new,     XR_REP_PTR,     XIR_HF_GC) \
  _(strbuf_append,  XR_REP_VOID,    XIR_HF_GC) \
  _(strbuf_finish,  XR_REP_RESULT,  XIR_HF_GC) \
  /* Collections */ \
  _(newrange,       XR_REP_PTR,     XIR_HF_GC) \
  _(newset,         XR_REP_PTR,     XIR_HF_GC) \
  _(map_set,        XR_REP_VOID,    XIR_HF_GC) \
  _(map_get,        XR_REP_RESULT,  XIR_HF_GC) \
  _(map_increment,  XR_REP_VOID,    XIR_HF_GC) \
  /* Enum */ \
  _(enum_access,    XR_REP_RESULT,  0) \
  _(enum_name,      XR_REP_PTR,     0) \
  _(enum_convert,   XR_REP_RESULT,  0) \
  /* Object */ \
  _(new_struct,     XR_REP_PTR,     XIR_HF_GC) \
  _(struct_get,     XR_REP_RESULT,  0) \
  _(struct_set,     XR_REP_VOID,    0) \
  _(struct_copy,    XR_REP_PTR,     XIR_HF_GC) \
  _(deep_copy,      XR_REP_PTR,     XIR_HF_GC) \
  _(bytes_new,      XR_REP_PTR,     XIR_HF_GC) \
  _(slice,          XR_REP_PTR,     XIR_HF_GC) \
  /* Closure / upvalue */ \
  _(closure_new,    XR_REP_PTR,     XIR_HF_GC) \
  _(closure_set_upval, XR_REP_VOID, 0) \
  _(upval_get,      XR_REP_RESULT,  0) \
  /* Concurrency */ \
  _(spawn_cont,     XR_REP_RESULT,  XIR_HF_GC|XIR_HF_SUSPEND) \
  _(await_block,    XR_REP_RESULT,  XIR_HF_SUSPEND) \
  _(chan_new,        XR_REP_PTR,     XIR_HF_GC) \
  _(chan_close,      XR_REP_VOID,    0) \
  _(chan_send_block, XR_REP_RESULT,  XIR_HF_SUSPEND) \
  _(chan_recv_block, XR_REP_RESULT,  XIR_HF_SUSPEND) \
  _(chan_try_send,   XR_REP_BOOL,    0) \
  _(chan_try_recv,   XR_REP_RESULT,  0) \
  _(chan_is_closed,  XR_REP_BOOL,    0) \
  /* Invoke / call */ \
  _(invoke_method,  XR_REP_RESULT,  XIR_HF_GC|XIR_HF_DEOPT) \
  _(call_self,      XR_REP_RESULT,  XIR_HF_GC|XIR_HF_DEOPT) \
  /* Array / map runtime */ \
  _(rt_array_new,   XR_REP_PTR,     XIR_HF_GC) \
  _(rt_array_push,  XR_REP_VOID,    XIR_HF_GC) \
  _(rt_array_len,   XR_REP_I64,     0) \
  _(rt_map_new,     XR_REP_PTR,     XIR_HF_GC) \
  /* Typed array */ \
  _(tarray_get,     XR_REP_RESULT,  0) \
  _(tarray_set,     XR_REP_VOID,    0) \
  /* Misc */ \
  _(throw,          XR_REP_VOID,    XIR_HF_THROW) \
  _(dump,           XR_REP_VOID,    0) \
  _(print,          XR_REP_VOID,    0) \
  _(assert,         XR_REP_VOID,    XIR_HF_THROW) \
  _(assert_eq,      XR_REP_VOID,    XIR_HF_THROW) \
  _(assert_ne,      XR_REP_VOID,    XIR_HF_THROW) \
  _(rt_eq,          XR_REP_BOOL,    0) \
  _(eq_value,       XR_REP_BOOL,    0) \
  _(scope_enter,    XR_REP_VOID,    0) \
  _(scope_exit,     XR_REP_VOID,    0) \
  _(typename,       XR_REP_PTR,     0) \
  _(range_unpack,   XR_REP_VOID,    0) \
  /* End */
```

这个表带来的收益：

1. **编译器强制**：新增 helper 必须在表中声明返回类型
2. **Builder 自动推导 VTAG**：`builder_emit_call_c(helper_id)` 可以从表中查询 `ret_rep`
3. **Codegen 自动处理 tag**：`ret_rep == XR_REP_RESULT` → 从 `call_result_tag` 读取；`ret_rep == XR_REP_PTR` → 静态写 `XR_TAG_PTR`
4. **审计简单**：所有 helper 在一个文件中，一目了然
5. **可生成验证代码**：自动检查 helper 函数签名与表声明一致

**实施方式**：

```c
/* 生成枚举 */
typedef enum {
#define XIR_HELPER_ENUM(name, rep, flags) XIR_HELPER_##name,
  XIR_HELPER_DEF(XIR_HELPER_ENUM)
#undef XIR_HELPER_ENUM
  XIR_HELPER__MAX
} XirHelperId;

/* 生成元信息表 */
typedef struct {
    void       *func;      // function pointer
    uint8_t     ret_rep;   // XR_REP_*
    uint16_t    flags;     // XIR_HF_*
} XirHelperInfo;

extern const XirHelperInfo xir_helper_info[XIR_HELPER__MAX];

/* Builder 使用 */
static inline uint8_t xir_helper_vtag(XirHelperId id) {
    switch (xir_helper_info[id].ret_rep) {
    case XR_REP_RESULT: return VTAG_TAGGED;  // dynamic tag
    case XR_REP_PTR:    return VTAG_PTR;
    case XR_REP_I64:    return VTAG_I64;
    case XR_REP_F64:    return VTAG_F64;
    case XR_REP_BOOL:   return VTAG_BOOL;
    case XR_REP_VOID:   return VTAG_NONE;
    default:            return VTAG_TAGGED;
    }
}

/* Codegen 使用 — 统一的 tag store 逻辑 */
static void emit_call_c_result_tag(CodegenCtx *ctx, XirHelperId id,
                                    int16_t bc_slot) {
    uint8_t rep = xir_helper_info[id].ret_rep;
    if (bc_slot < 0 || bc_slot >= 256) return;
    int32_t tag_off = (int32_t)XIR_JIT_SLOT_RUNTIME_TAGS_OFFSET + bc_slot;
    
    switch (rep) {
    case XR_REP_RESULT:
        // Dynamic: read from call_result_tag (set by XrJitResult.tag via x1)
        a64_buf_emit(&ctx->buf, a64_ldrb(SCRATCH_REG2, JIT_CTX_REG,
                     XIR_JIT_CALL_RESULT_TAG_OFFSET));
        a64_buf_emit(&ctx->buf, a64_strb(SCRATCH_REG2, JIT_CTX_REG, tag_off));
        break;
    case XR_REP_PTR:
        a64_buf_emit(&ctx->buf, a64_movz(SCRATCH_REG2, XR_TAG_PTR, 0));
        a64_buf_emit(&ctx->buf, a64_strb(SCRATCH_REG2, JIT_CTX_REG, tag_off));
        break;
    case XR_REP_I64:
        a64_buf_emit(&ctx->buf, a64_movz(SCRATCH_REG2, XR_TAG_I64, 0));
        a64_buf_emit(&ctx->buf, a64_strb(SCRATCH_REG2, JIT_CTX_REG, tag_off));
        break;
    case XR_REP_BOOL:
        a64_buf_emit(&ctx->buf, a64_movz(SCRATCH_REG2, XR_TAG_BOOL, 0));
        a64_buf_emit(&ctx->buf, a64_strb(SCRATCH_REG2, JIT_CTX_REG, tag_off));
        break;
    case XR_REP_VOID:
        // No tag store needed
        break;
    }
}
```

---

### 改进 B（P0）：统一所有 Helper 返回 XrJitResult

**灵感来源**：V8 所有 runtime 返回 `Tagged<Object>`，Dart 所有 runtime 返回 tagged value

将 59 个 `int64_t` helper 全部改为返回 `XrJitResult`，**彻底消除两种返回约定**。

```c
/* 便利宏 — xir_jit_runtime.h */
#define XR_JIT_NULL()   ((XrJitResult){ 0, XR_TAG_NULL })
#define XR_JIT_INT(v)   ((XrJitResult){ (int64_t)(v), XR_TAG_I64 })
#define XR_JIT_FLOAT(v) ((XrJitResult){ .payload = 0, .tag = XR_TAG_F64 })  
#define XR_JIT_BOOL(v)  ((XrJitResult){ (int64_t)(v), XR_TAG_BOOL })
#define XR_JIT_PTR(p)   ((XrJitResult){ (int64_t)(uintptr_t)(p), XR_TAG_PTR })
#define XR_JIT_VAL(v)   ((XrJitResult){ (v).i, (uint64_t)(v).tag })
#define XR_JIT_OK()     XR_JIT_NULL()  // for void-like helpers
```

改造示例：

```c
// Before: 可能忘记设 call_result_tag
int64_t xr_jit_getbuiltin(XrCoroutine *coro, int64_t extra_arg) {
    int idx = (int)(extra_arg & 0xFFFF);
    XrValue val = isolate->builtins[idx];
    coro->jit_ctx->call_result_tag = val.tag;  // 容易忘！
    return val.i;
}

// After: tag 作为返回值的一部分，不可能忘
XrJitResult xr_jit_getbuiltin(XrCoroutine *coro, int64_t extra_arg) {
    int idx = (int)(extra_arg & 0xFFFF);
    XrValue val = isolate->builtins[idx];
    return XR_JIT_VAL(val);  // tag 自动随值返回
}
```

`call_c_stub` 已经正确处理 `XrJitResult`（x0=payload, x1=tag），无需修改。统一后，x1 总是有效 tag。

**之前手动设 `call_result_tag` 的代码可以全部删除**。

---

### 改进 C（P0）：封装 braun_write_var

**灵感来源**：Dart 的 `InsertBefore/After` 统一 API，LuaJIT 的 `emitir()` 统一宏

将 `braun_write_var` 设为 **static 内部函数**，不再对外暴露。对外只提供：

```c
/* 写入当前块的 slot（现有 API，保持不变） */
void builder_set_slot(XirBuilder *b, int slot, XirRef ref);

/* 新增：写入指定块的 slot（用于 CPS suspend 等跨块场景） */
void builder_set_slot_in_block(XirBuilder *b, uint32_t blk_id,
                                int slot, XirRef ref);
```

`builder_set_slot_in_block` 内部调用 `braun_write_var` 并同时设置 `bc_slot`：

```c
void builder_set_slot_in_block(XirBuilder *b, uint32_t blk_id,
                                int slot, XirRef ref) {
    braun_write_var(b, blk_id, slot, ref);
    // Ensure bc_slot is set for runtime_tags propagation
    if (xir_ref_is_vreg(ref)) {
        uint32_t vi = XIR_REF_INDEX(ref);
        if (vi < b->func->nvreg) {
            XirVReg *vreg = &b->func->vregs[vi];
            if (vreg->bc_slot < 0)
                vreg->bc_slot = (int16_t)slot;
            // Also update slot_map for OSR
            if (slot < 256)
                b->func->slot_map[slot] = ref;
        }
    }
}
```

---

### 改进 D（P1）：扩展 JIT Verifier

**灵感来源**：V8 Verifier（90K 行），Dart FlowGraph verifier

在现有 V1（regalloc overlap）和 V2（side-effect preservation）基础上添加：

#### V3: bc_slot 完整性验证器

```c
/* Run after builder, before codegen */
static void verify_bc_slot_completeness(XirFunc *func) {
    for (uint32_t b = 0; b < func->nblock; b++) {
        XirBlock *blk = &func->blocks[b];
        for (uint32_t i = 0; i < blk->nins; i++) {
            XirIns *ins = &blk->ins[i];
            if (ins->op != XIR_CALL_C) continue;
            if (!xir_ref_is_vreg(ins->dst)) continue;
            uint32_t vi = XIR_REF_INDEX(ins->dst);
            if (vi >= func->nvreg) continue;
            XirVReg *vreg = &func->vregs[vi];
            // If vreg is TAGGED, it MUST have bc_slot for runtime_tags
            if (vreg->tag == VTAG_TAGGED || vreg->tag == VTAG_UNKNOWN) {
                XR_DCHECK(vreg->bc_slot >= 0,
                    "V3: CALL_C result vreg v%u is TAGGED but bc_slot=%d",
                    vi, vreg->bc_slot);
            }
        }
    }
}
```

#### V4: Helper 表一致性验证器

```c
/* Run at JIT init, verify helper function signatures match table */
static void verify_helper_table(void) {
    // For each helper in XIR_HELPER_DEF:
    //   - Verify function pointer is non-NULL
    //   - Verify ret_rep is valid enum value
    //   - If ret_rep != XR_REP_VOID, verify function returns XrJitResult
    for (int i = 0; i < XIR_HELPER__MAX; i++) {
        XR_DCHECK(xir_helper_info[i].func != NULL,
            "V4: helper %d has NULL function pointer", i);
        XR_DCHECK(xir_helper_info[i].ret_rep <= XR_REP_VOID,
            "V4: helper %d has invalid ret_rep %d",
            i, xir_helper_info[i].ret_rep);
    }
}
```

#### V5: CALL_C 参数 tag 完整性验证器

```c
/* Verify that all CALL_C arg vregs with UNKNOWN type have valid bc_slot */
static void verify_call_arg_tags(XirFunc *func) {
    for (uint32_t b = 0; b < func->nblock; b++) {
        XirBlock *blk = &func->blocks[b];
        for (uint32_t i = 0; i < blk->nins; i++) {
            XirIns *ins = &blk->ins[i];
            if (ins->op != XIR_CALL_C && ins->op != XIR_CALL_C_LEAF) continue;
            if (!xir_ref_is_vreg(ins->dst)) continue;
            uint32_t vi = XIR_REF_INDEX(ins->dst);
            if (vi >= func->nvreg) continue;
            XirVReg *vreg = &func->vregs[vi];
            // Check each arg in pool
            for (uint16_t a = 0; a < vreg->call_nargs; a++) {
                XirRef arg = func->call_arg_pool[vreg->call_arg_start + a];
                if (!xir_ref_is_vreg(arg)) continue;
                uint32_t ai = XIR_REF_INDEX(arg);
                if (ai >= func->nvreg) continue;
                XirType ct = func->vregs[ai].ctype;
                if (ct.kind == XIR_TYPE_UNKNOWN) {
                    int16_t bs = func->vregs[ai].bc_slot;
                    XR_DCHECK(bs >= 0 && bs < 256,
                        "V5: CALL_C arg v%u has UNKNOWN type but "
                        "bc_slot=%d (needed for runtime tag lookup)",
                        ai, bs);
                }
            }
        }
    }
}
```

---

### 改进 E（P2）：Codegen 模板化

**灵感来源**：Dart 的 `EmitNativeCode` 虚方法模式，LuaJIT 的 DynASM

将重复的 codegen 模式提取为函数：

```c
/* Unified tag propagation after CALL_C */
static void emit_call_c_tag_writeback(CodegenCtx *ctx, XirIns *ins) {
    if (!xir_ref_is_vreg(ins->dst)) return;
    uint32_t vi = XIR_REF_INDEX(ins->dst);
    if (vi >= ctx->func->nvreg) return;
    int16_t bc_slot = ctx->func->vregs[vi].bc_slot;
    if (bc_slot < 0 || bc_slot >= 256) return;
    
    // Use helper table to determine tag source
    XirHelperId hid = ins->helper_id;  // new field on XIR_CALL_C
    emit_call_c_result_tag(ctx, hid, bc_slot);
}

/* Unified safepoint + spill writeback before CALL_C */
static void emit_call_c_prologue(CodegenCtx *ctx) {
    emit_ptr_spill_writeback(ctx);
    uint32_t smap_id = record_safepoint(ctx);
    a64_buf_emit(&ctx->buf, a64_movz(SCRATCH_REG2, (uint16_t)smap_id, 0));
    a64_buf_emit(&ctx->buf, a64_str_w(SCRATCH_REG2, A64_FP, FRAME_SMAP_ID_OFFSET));
    a64_buf_emit(&ctx->buf, a64_str_w(SCRATCH_REG2, JIT_CTX_REG,
                 XIR_JIT_ACTIVE_SMAP_ID_OFFSET));
}
```

---

### ~~改进 F（已废弃）~~

> 原方案 F 建议"在 A-E 之后考虑 NaN-boxing"。在"不考虑向后兼容"原则下，
> 这个优先级评估是错误的。见下文"第五章：激进方案"。

---

## 五、激进方案：根因消除（不考虑向后兼容）

> **设计原则**：Xray 没有外部用户，代码量小，可以大胆重写。
> 每个阶段都选择最佳设计。不保留旧接口，不做兼容层。

上面的改进 A-E 是**保守的补丁方案**——在有缺陷的架构上加防护。
在"不考虑向后兼容"原则下，应该**从根因消除问题**。

### 5.1 根因重新分析

所有 tag 传播 bug 的根因不是"忘记设 tag"，而是：

**tag 和 value 通过独立通道传递这个架构本身就是 bug 温床。**

当前 xray JIT 的值流动：

```
                    ┌─────────────┐
                    │  XrValue    │  VM 层：16B 统一体
                    │  tag + val  │  tag 和 value 不可分离
                    └──────┬──────┘
                           │ JIT 入口（OSR）
                    ┌──────┴──────┐
                    │   拆 分     │  ← 问题起源点
                    ├─────────────┤
        ┌───────────┤ value (GPR) │  寄存器：只存 payload
        │           ├─────────────┤
        │      ┌────┤ tag (memory)│  内存旁路：slot_runtime_tags[bc_slot]
        │      │    └─────────────┘
        │      │         ↓
        │      │    需要 bc_slot 索引 ← Pattern 25 bug 源
        │      │    需要 call_result_tag 传递 ← Pattern 24 bug 源
        │      │    需要 call_arg_tags 传递
        │      │    需要 vtag_to_value_tag 转换
        │      │         ↓
        │      │    四层间接旁路 → 任何一层遗漏 = 隐蔽 bug
        └──────┴─────────┘
```

**三个成熟 JIT 都不做这种拆分**——它们的值自带类型：

| 引擎 | 值大小 | 类型编码方式 | JIT 内部 value/tag 拆分？ |
|------|--------|-------------|------------------------|
| V8 | 8B | LSB tag | **否**——Tagged Pointer 自带类型 |
| LuaJIT | 8B | NaN 高位 | **否**——NaN-boxing 自带类型 |
| Dart | 8B | LSB tag | **否**——Tagged Pointer 自带类型 |
| **xray** | **16B** | **独立 tag 字段** | **是**——JIT 拆成 GPR + memory side-channel |

### 5.2 方案 G（备选）：NaN-boxing 全局值表示

**彻底消除 tag 旁路通道——从"无法忘记"升级为"概念不存在"。**

#### 编码方案

```
64-bit NaN-boxing（类似 LuaJIT/JSC，适配 xray 类型系统）

IEEE 754 double 的 quiet NaN: 0x7FF8_0000_0000_0000 起
利用 NaN 空间编码非 double 值：

  ┌── 高 16 位 ──┐┌────────── 低 48 位 ──────────┐
  │   tag prefix  ││         payload              │
  └───────────────┘└──────────────────────────────┘

  Double:     原样存储（非 signaling NaN 的任意 double）
  Int48:      0xFFFC_XXXX_XXXX_XXXX  (48 位有符号整数，±140 万亿)
  Pointer:    0x0000_XXXX_XXXX_XXXX  (低 48 位，ARM64 用户空间地址)
  Null:       0xFFF8_0000_0000_0000  (quiet NaN + 0)
  True:       0xFFF9_0000_0000_0001
  False:      0xFFF9_0000_0000_0000
  NotFound:   0xFFFA_0000_0000_0000  (sentinel)
  StructRef:  0xFFFB_XXXX_XXXX_XXXX  (栈分配 struct 指针)
```

#### 类型检测（全部是位操作，零分支）

```c
typedef uint64_t XrNaN;

// Pointer: top 16 bits are 0x0000 (ARM64 user space)
static inline bool xr_is_ptr(XrNaN v)    { return (v >> 48) == 0; }
// Double: NOT a quiet NaN tag pattern (not 0xFFF8-0xFFFF)
static inline bool xr_is_double(XrNaN v) { return (v & 0xFFF8000000000000ULL) != 0xFFF8000000000000ULL && !xr_is_ptr(v); }
// Int48: tag prefix 0xFFFC
static inline bool xr_is_int(XrNaN v)    { return (v >> 48) == 0xFFFC; }
// Null: exact value
static inline bool xr_is_null(XrNaN v)   { return v == 0xFFF8000000000000ULL; }
// Bool: tag prefix 0xFFF9
static inline bool xr_is_bool(XrNaN v)   { return (v >> 48) == 0xFFF9; }

// Unbox
static inline int64_t xr_to_int(XrNaN v) {
    // Sign-extend 48-bit to 64-bit
    return ((int64_t)(v << 16)) >> 16;
}
static inline double xr_to_double(XrNaN v) {
    union { uint64_t u; double d; } u = { .u = v };
    return u.d;
}
static inline void* xr_to_ptr(XrNaN v)   { return (void*)(uintptr_t)v; }
static inline bool xr_to_bool(XrNaN v)   { return (v & 1); }

// Box
static inline XrNaN xr_box_int(int64_t i) {
    return 0xFFFC000000000000ULL | ((uint64_t)i & 0x0000FFFFFFFFFFFFULL);
}
static inline XrNaN xr_box_double(double d) {
    union { double d; uint64_t u; } u = { .d = d };
    return u.u;
}
static inline XrNaN xr_box_ptr(void *p) {
    return (uint64_t)(uintptr_t)p;  // top bits naturally 0
}
#define XR_NULL  0xFFF8000000000000ULL
#define XR_TRUE  0xFFF9000000000001ULL
#define XR_FALSE 0xFFF9000000000000ULL
```

#### 整数范围：int48 够用吗？

| 范围 | 值 | 场景覆盖 |
|------|-----|---------|
| int48 | ±140,737,488,355,327 (±140 万亿) | 覆盖 99.99% 日常编程场景 |
| int53 (JS safe int) | ±9,007,199,254,740,991 | JavaScript 安全整数范围 |
| int64 (当前 xray) | ±9.2 × 10^18 | 仅密码学/hash 场景需要 |

**xray 已有 BigInt 支持**（`src/runtime/object/xbigint.h`），
超出 int48 的场景用 heap-allocated BigInt 处理（自动提升，对用户透明）。

#### NaN-boxing 带来的架构革命

**彻底删除的概念和代码**（不是修改，是删除）：

| 删除项 | 当前代码量 | 原因 |
|--------|-----------|------|
| `slot_runtime_tags[256]` | ~139 处引用（15个文件） | **不存在** — 值自带类型 |
| `call_result_tag` | ~30 处引用 | **不存在** — 返回值自带类型 |
| `call_arg_tags[]` | ~20 处引用 | **不存在** — 参数自带类型 |
| `bc_slot` tag 传播用途 | ~50 处引用 | **不存在** — 无 tag 索引需求 |
| `vtag_to_value_tag()` | 转换函数 | **不存在** — 无两套 tag 系统 |
| `XR_RTAG_NUMERIC/UNKNOWN` | 信号常量 | **不存在** — 值自描述 |
| `deopt_reconstruct()` | 重建函数 | **大幅简化** — 值已完整 |
| Pattern 24/25 bug 整类 | 历史 bug | **概念不可能出现** |

**简化的 CALL_C 调用约定**：

```c
// Before (16B split): helper 返回 int64_t，tag 通过旁路
int64_t xr_jit_getprop(XrCoroutine*, int64_t, int64_t);
// codegen 还需要：
//   1. 从 call_result_tag 读 tag
//   2. 写入 slot_runtime_tags[bc_slot]
//   3. bc_slot 必须正确设置
//   4. helper 内部必须设 coro->jit_ctx->call_result_tag

// After (NaN-boxing): helper 返回 XrNaN，值自带类型
XrNaN xr_jit_getprop(XrCoroutine*, XrNaN obj, XrNaN key);
// codegen：直接 mov dst_reg, x0。结束。
```

**简化的栈帧和数组**：

```
Before: XrValue slot = 16 bytes  →  256 slots = 4KB 栈帧
After:  XrNaN slot   = 8 bytes   →  256 slots = 2KB 栈帧

Before: XrArray 元素 = 16 bytes/elem → 1000 个 int = 16KB
After:  XrArray 元素 = 8 bytes/elem  → 1000 个 int = 8KB
                                         → L1 cache 命中率翻倍
```

#### 代价与应对

| 代价 | 应对方案 |
|------|---------|
| int 从 64 位缩到 48 位 | BigInt 自动提升（xray 已有 BigInt） |
| descriptor 中的 flags/heap_type 丢失 | flags 移到 GC 对象头；heap_type 已在对象头中 |
| 需要改 XrValue/VM/GC/stdlib | xray 代码量小，可控。BigInt 无改动 |
| Double NaN payload 被占用 | 用 canonical NaN (0x7FF8000000000000) 替代所有 NaN 输出 |

#### 工作量评估

| 模块 | 预计工作量 | 说明 |
|------|-----------|------|
| `xvalue.h` + 值操作函数 | 2 天 | 用 `XrNaN` 替代 `XrValue` |
| VM 解释器 (`xvm.c`) | 3 天 | 指令的值操作全部改为 NaN-boxing |
| GC (`xgc*.c`, `xcoro_gc.c`) | 2 天 | 扫描指针：检测 `xr_is_ptr(v)` |
| JIT builder (`xir_builder*.c`) | 2 天 | 删除所有 tag 旁路代码 |
| JIT codegen (`xir_codegen*.c`) | 2 天 | 简化：删除 runtime_tags 写入 |
| JIT helpers (`xir_jit.c`) | 1 天 | 改签名为 `XrNaN` 返回 |
| stdlib (`stdlib/`) | 2 天 | XrValue → XrNaN |
| 回归测试修复 | 2 天 | 调整测试期望值 |
| **合计** | **~16 天（3-4 周）** | |

---

### 5.3 方案 H（替代）：Tag-as-VReg — 保持 16B，消除旁路

如果 int64 是**不可妥协**的语言特性（不接受 int48 + BigInt），
可以在保持 16B XrValue 的前提下，把 tag 提升为 IR 一等公民。

#### 核心思想

```
当前：value 在寄存器，tag 在 slot_runtime_tags[] 内存旁路
改为：value 在寄存器，tag 也在寄存器（独立 vreg）

                    ┌─────────────┐
                    │  XrValue    │
                    │  tag + val  │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │   拆 分     │
                    ├─────────────┤
         ┌──────── │ v_payload   │ → GPR x_n（payload vreg）
         │         ├─────────────┤
         │    ┌─── │ v_tag       │ → GPR x_m（tag vreg）← 新：也是寄存器！
         │    │    └─────────────┘
         │    │
         │    │    Tag 由寄存器分配器跟踪，不走内存旁路
         │    │    CALL_C 后：mov v_tag, x1（tag 进入 vreg）
         │    │    需要 tag 的操作：直接引用 v_tag vreg
         │    │    
         │    │    ✓ 无 slot_runtime_tags
         │    │    ✓ 无 bc_slot 索引
         │    │    ✓ 无 call_result_tag
         └────┴────→ 寄存器分配器统一管理 payload + tag
```

#### 实现要点

```c
// CALL_C 在 builder 中产生 TWO vregs:
XirRef v_payload = builder_new_vreg(b, VTAG_TAGGED);
XirRef v_tag     = builder_new_vreg(b, VTAG_I64);  // tag is a small int

// CALL_C 指令携带 tag dst:
ins->dst       = v_payload;  // x0 → payload vreg
ins->tag_dst   = v_tag;      // x1 → tag vreg (new field)

// 需要 tag 的操作直接引用 v_tag:
// e.g., INDEX_SET 需要知道 value 的 tag
ins_index_set->args[0] = v_payload;
ins_index_set->args[1] = v_tag;  // tag 作为显式参数

// Deopt reconstruct:
deopt_slot = { .value = v_payload, .tag = v_tag };
// 不再需要 bc_slot 查 slot_runtime_tags
```

#### Tag-as-VReg 的收益与代价

**收益**：
- 删除 `slot_runtime_tags[]`、`call_result_tag`、`call_arg_tags[]`
- Tag 由寄存器分配器管理——不可能"忘记传播"
- 保持 int64 完整范围
- 改动范围比 NaN-boxing 小（不改 VM/GC/stdlib）

**代价**：
- 寄存器压力增加（每个 TAGGED 值占 2 个 GPR）
- ARM64 有 ~15 个可用 GPR，TAGGED 值多时会频繁 spill
- XirIns 结构需要新增 `tag_dst` 字段
- 寄存器分配器需要理解 "paired" vreg 概念
- 栈帧和数组仍然 16B/元素（无内存优化）

---

### 5.4 方案 I（★推荐★）：C#/Dart 式 — 保持 16B + JIT 全面 unbox + 消除 tag 旁路

> **核心洞察**：C# 达到高性能的秘诀不是 NaN-boxing，而是「静态类型 + JIT 利用类型信息」。
> xray 是 typed scripting，应走 C#/Dart 路线，而非 LuaJIT/JSC 路线。

#### 核心思想

```
方案 I = 保持 16B VM 表示 + 改进 A（helper 声明表）+ 方案 H（Tag-as-VReg）+ 强化类型推断

1. VM 层：保持 16B XrValue（安全、int64 完整范围、简单）
2. JIT 层已知类型：直接用原始寄存器（已有机制，继续强化）
   - VTAG_I64 → GPR（int64 完整范围）
   - VTAG_F64 → FPR（double 完整范围）
   - VTAG_PTR → GPR
   - VTAG_BOOL → GPR
3. JIT 层未知类型（VTAG_TAGGED）：Tag-as-VReg 消除旁路
   - tag 提升为独立 vreg，寄存器分配器统一管理
   - 删除 slot_runtime_tags / call_result_tag / call_arg_tags
4. Helper 函数：改进 A（声明式表）+ 精确类型签名（学 Mono）
5. 长期方向：强化类型推断 → 减少 VTAG_TAGGED → 趋近 C# 模式
6. AOT：完全不需要统一值表示（类型已知）→ 天然受益
```

#### 与 C# (Mono) 的对应关系

| C# (Mono JIT) | xray (方案 I) | 状态 |
|----------------|---------------|------|
| `STACK_I4`/`STACK_I8` → GPR | `VTAG_I64` → GPR | ✅ 已有 |
| `STACK_R8` → FPR | `VTAG_F64` → FPR | ✅ 已有 |
| `STACK_OBJ` → GPR (pointer) | `VTAG_PTR` → GPR | ✅ 已有 |
| `STACK_VTYPE` → stack/memory | `VTAG_TAGGED` → Tag-as-VReg | ← **方案 H 改进** |
| `mono_idiv(gint32,gint32)` | 精确类型 helper | ← **改进 A + 签名精确化** |
| `mini_emit_box`/`mini_handle_unbox` | XIR_BOX / XIR_UNBOX | ✅ 已有 |
| MonoStackType 编译时映射 | XrVRegTag 编译时推断 | ✅ 已有（TFA 强化中） |

#### 为什么不选 NaN-boxing？

NaN-boxing 是**动态语言的优化方向**（LuaJIT, JSC, SpiderMonkey）。xray 目标 C# 级性能：

| 对比维度 | NaN-boxing (G) | C# 式 (I) | 说明 |
|---------|----------------|-----------|------|
| **int 范围** | int48 (±140万亿) | **int64** (完整) | C#/Go/Java 都是 int64 |
| **double** | 内联（好） | 内联（好） | 两者一样 |
| **阵营归属** | 动态语言（LuaJIT/JSC） | **静态类型（C#/Dart/Go）** | xray 目标是 C# |
| **AOT 价值** | 无（AOT 类型已知） | **天然契合** | xray 后期主打 AOT |
| **改动范围** | 全栈（VM/GC/stdlib） | **仅 JIT** | 风险更低 |
| **tag 旁路消除** | 是（概念不存在） | **是（Tag-as-VReg）** | 两者都消除 |
| **内存** | 8B（减半） | 16B（不变） | 但 JIT unbox 后无差异 |

**关键论证**：在 JIT/AOT 编译后的代码中，值几乎全部 unboxed（GPR/FPR），16B vs 8B 的差异仅影响 VM 解释器和未 JIT 编译的冷代码。**热路径上两者性能一致**。

#### VTAG_TAGGED 出现频率分析

xray 是 typed scripting，大部分变量编译时类型已知。VTAG_TAGGED 仅出现在：

| 场景 | 频率 | 说明 |
|------|------|------|
| `any` 类型参数 | 低 | xray 鼓励精确类型 |
| `int \| float` 异构 union | 低 | 特定场景 |
| 动态属性 getprop/setprop | 中 | 逐步被 IC 和字段偏移优化取代 |
| 多态方法调用返回值 | 中 | TFA 可推断收窄 |
| 无类型注解变量 | 低 | 类型推断兜底 |

**结论**：VTAG_TAGGED 在 xray 中占比很小（预估 <15% 的 vreg），Tag-as-VReg 的寄存器压力增加可控。随着类型推断增强，这个比例会持续降低。

---

### 5.5 四种路线对比

| 维度 | 保守 A-E | NaN-boxing (G) | Tag-as-VReg (H) | ★ C# 式 (I) |
|------|---------|----------------|-----------------|-------------|
| **根因解决** | 否（加防护） | 是（概念消除） | 是（消除旁路） | **是（消除旁路）** |
| **Pattern 24/25** | 大幅降低 | 不可能出现 | 不可能出现 | **不可能出现** |
| **int 范围** | int64 | int48 + BigInt | int64 | **int64** |
| **double** | 内联 16B | 内联 8B | 内联 16B | **内联 16B** |
| **内存优化** | 无 | 减半 | 无 | 无（JIT 后无差异） |
| **改动范围** | 仅 JIT | 全栈 | 仅 JIT | **仅 JIT** |
| **阵营** | — | 动态语言 | — | **静态类型（C#/Dart）** |
| **AOT 契合度** | 低 | 低 | 中 | **高** |
| **C# 性能目标** | 不相关 | 偏离方向 | 部分契合 | **完全契合** |
| **工作量** | ~4 周 | ~3-4 周 | ~2-3 周 | **~3 周** |
| **复杂度风险** | 低 | 中（全栈） | 中（regalloc） | **中（regalloc）** |

### 5.6 推荐路线

```
★ 推荐：方案 I (C#/Dart 式)

理由：
1. 与 xray 定位一致 — typed scripting，目标 C# 性能，主打 AOT
2. 与 C#/Dart/Go 的方法论统一 — 静态类型 + JIT 利用类型信息
3. 保持 int64 完整范围 — 无 NaN-boxing 的 int48 限制
4. 保持 double 内联 — 无 V8/Dart 的 HeapNumber 堆分配开销
5. 消除 tag 旁路（Tag-as-VReg）— Pattern 24/25 不可能出现
6. 改动范围仅 JIT — 不改 VM/GC/stdlib，风险最低
7. AOT 天然受益 — 类型完全已知，无统一值表示开销
8. 渐进演进 — 类型推断增强 → TAGGED 越来越少 → 趋近 C# 性能

组合内容：
  - 改进 A（P0）：Helper 声明式表（LuaJIT IRCALLDEF 灵感）
  - 方案 H（P0）：Tag-as-VReg（消除 tag 旁路）
  - 改进 C（P0）：封装 braun_write_var
  - 改进 D（P1）：扩展 Verifier
  - 改进 E（P2）：Codegen 模板化
  - 长期：强化 TFA 类型推断，减少 VTAG_TAGGED
```

---

## 六、方案 I 实施路线图

```
Phase 1（Week 1）— 基础设施
├── I1: 实现改进 A — XIR_HELPER_DEF 声明式表
├── I2: 统一 helper 签名（学 Mono：精确类型优先，TAGGED 用 XrJitResult）
├── I3: 实现改进 C — 封装 braun_write_var
└── 基础回归测试

Phase 2（Week 2）— Tag-as-VReg 核心改造
├── I4: XirIns 新增 tag_dst 字段
├── I5: Builder: CALL_C 产生 payload + tag 两个 vreg
├── I6: Regalloc: 支持 paired vreg（tag vreg 优先分配 caller-saved）
├── I7: Codegen: CALL_C 后 mov tag_vreg, x1
├── I8: Deopt: XirDeoptSlot 使用 tag vreg 而非 bc_slot 查表
├── I9: 删除 slot_runtime_tags / call_result_tag / call_arg_tags
└── JIT 回归测试

Phase 3（Week 3）— 验证器 + Codegen 模板化 + 清理
├── I10: 实现改进 D — V3/V4/V5 验证器
├── I11: 实现改进 E — Codegen 模板化
├── I12: 审计所有 VTAG_TAGGED 路径，确认 tag vreg 正确传播
├── I13: 全面回归测试 + ASan + JIT force 测试
└── 更新文档

长期（持续）— 类型推断强化
├── 强化 TFA：减少 VTAG_TAGGED/UNKNOWN 的产生
├── IC (Inline Cache) 优化：getprop/setprop 走精确类型路径
├── 方法返回类型推断：减少多态调用的 TAGGED 结果
└── 目标：VTAG_TAGGED 占比 < 5%
```

---

## 七、预期收益总结

| 维度 | Before (16B + tag 旁路) | After (方案 I: C# 式) |
|------|------------------------|----------------------|
| **值大小** | 16 bytes | 16 bytes（VM 不变） |
| **int 范围** | int64 | **int64**（完整保持） |
| **double** | 内联 | **内联**（完整保持） |
| **类型信息传播** | 独立内存旁路（4层间接） | Tag-as-VReg（寄存器流动） |
| **CALL_C 返回** | x0+x1 + 4层间接写入 | x0=payload, x1=tag_vreg（直接） |
| **tag 传播 bug** | 反复出现 | **不可能出现** |
| **slot_runtime_tags** | 139 处引用 | **删除** |
| **call_result_tag** | 30 处引用 | **删除** |
| **call_arg_tags** | 20 处引用 | **删除** |
| **Helper 声明** | 散落各文件 | **集中式声明表** |
| **改动范围** | — | 仅 JIT（VM/GC/stdlib 不改） |
| **AOT 兼容性** | — | **完全兼容**（AOT 类型已知，无统一值表示开销） |
| **C# 性能路线** | 偏离 | **契合** |

