# Json 类型统一重构方案

> 设计原则：无外部用户，零兼容负担，每层选择最优设计。
> 前置文档：`docs/tasks/064-json-type-system-refactor.md`（语义层重构）。本文聚焦**运行时表示 + 编译流水线**。

## 现状诊断

### 核心问题：同一语义概念走了三条完全不同的运行时路径

| 来源 | 编译期类型 | 运行时对象 | 字段访问复杂度 |
|------|-----------|-----------|--------------|
| `{x: 1, y: 2}` (Xi IR) | `XR_KIND_JSON` | **XrMap** (OP_NEWMAP) | O(n) hash lookup |
| `Json.parse(str)` | `XR_KIND_JSON` | **XrJson** (Shape) | O(1) Shape IC |
| `type P = {x:int}` | `XR_KIND_OBJECT` | 无独立运行时对象 | 编译期概念 |
| AOT `{x: 1}` | `XR_KIND_JSON` | **xrt_map_t** | O(n) 线性扫描 |

### 具体分裂点

**Xi IR 路径** (`xi_lower_expr.c:1577`):
- `lower_object_literal()` → `XI_ALLOC` + `XI_INDEX_SET`
- `xi_emit_alloc()` → **OP_NEWMAP** (创建 XrMap！)
- `xi_emit_index_set()` → **OP_INDEX_SET** (Map 的通用 set)

**VM 中已有完整 Json 指令集** (但 Xi IR 不使用):
- `OP_NEWJSON` — 用 Shape 创建 XrJson
- `OP_JSON_INIT` / `OP_JSON_INIT_I` / `OP_JSON_INIT_N` — 按 index O(1) 初始化
- `OP_JSON_GET` / `OP_JSON_SET` — 按 index O(1) 读写
- `OP_JSON_GETK` / `OP_JSON_SETK` — 按 Symbol O(1) 读写
- `OP_GETPROP` 对 Json 有 **Shape IC fast path**

**AOT 路径** (`xi_cgen.c:977`):
- `XI_ALLOC` → `xrt_map_new()` (线性数组)
- `XI_LOAD_FIELD` → `xrt_map_get()` (O(n) 扫描)
- `XI_STORE_FIELD` → `xrt_map_set()` (O(n) 扫描)

---

## 目标设计

### 核心思想：一切 `{k:v}` 都是 XrJson + Shape

```
{x: 1, y: 2}     →  XrJson (sealed shape)     O(1) indexed access
Json.parse(str)   →  XrJson (open shape)       O(1) Shape IC access
type P = {x:int}  →  XrJson (sealed shape)     编译器知道全部字段 → O(1)
```

### 类型系统：删除 XR_KIND_OBJECT，用 sealed flag 替代

```c
// xtype.h — 修改后
typedef struct XrObjectType {
    const char **field_names;  // Field names array
    XrType **field_types;      // Field types array (parallel to names)
    bool *field_readonly;      // Per-field readonly flags (from `const` modifier)
    int field_count;           // Number of declared fields
    const char *type_name;     // NULL = anonymous literal
    bool is_sealed;            // NEW: true = 字段集合固定，禁止运行时扩展
} XrObjectType;
```

#### Sealed/Open 判定规则（`...` 修饰符）

`...` 是 sealed/open 的**唯一分界线**，已在 parser 中实现（`xparse_type.c:287`）：

| 源码写法 | `is_sealed` | 说明 |
|----------|-------------|------|
| `type P = { x: int, y: string }` | `true` | 无 `...`，字段集合固定 |
| `type P = { x: int, y: string, ... }` | `false` | 有 `...`，允许运行时添加新字段 |
| `let obj = {x: 1, y: 2}` （无注解） | `true` | 字面量推断为 sealed |
| `let data: Json = {x: 1}` | `false` | 显式 `: Json` 注解 → open |
| `Json.parse(str)` 返回值 | `false` | 动态来源，必须 open |
| `let j: Json = ...` 无字段信息 | `false` | `field_count = 0`，open |

Parser 当前代码（`xparse_type.c:331-335`）：
```c
// With ... → dynamic Json type (extensible), without → fixed Object type
XrType *result = allow_extension
    ? xr_type_new_json_with_fields(...)   // 当前 → XR_KIND_JSON
    : xr_type_new_object(...)             // 当前 → XR_KIND_OBJECT
```
改为统一调用 `xr_type_new_json_with_fields(..., is_sealed=!allow_extension)`。

#### 字段修饰符（`?` 和 `const`）

两者已在 parser 中实现（`xparse_type.c:303-322`），**不受本次重构影响**：

| 修饰符 | 语法示例 | 语义 | 存储 |
|--------|----------|------|------|
| `?` | `email?: string` | 字段可省略，值可为 null | `field_types[i]` 被包装为 `xr_type_new_optional(T)` → `T?` |
| `const` | `const name: string` | 字段创建后只读 | `field_readonly[i] = true` |
| (无) | `age: int` | required + 可写 | 默认 |

**综合示例：**
```xray
// Sealed：字段固定，email 可选，id 只读
type User = {
    const id: int,
    name: string,
    email?: string
}

// Open：有已知字段，但允许运行时添加新字段
type Config = {
    host: string,
    port: int,
    ...                    // ← 允许 cfg.timeout = 5000
}
```

#### 删除的符号

- `XR_KIND_OBJECT` — 从 `XrTypeKind` 枚举中移除
- `XR_TYPE_IS_OBJECT()` — 删除，调用方改用 `XR_TYPE_IS_JSON(t) && t->object.is_sealed`
- `xr_type_new_object()` / `xr_type_new_object_anonymous()` — 合并入 `xr_type_new_json_with_fields()`

---

### XrJson 动态扩展机制：Shape Transition

XrJson 能动态扩展字段**不是因为它是 Map**，而是靠 **Shape Transition**（类似 V8 Hidden Class）：

```
初始 Shape: {x, y}  →  添加 z  →  新 Shape: {x, y, z}
       ↑                              ↑
   json->shape_id                  json->shape_id (更新)
```

核心代码（`xjson.c:241-266`）：
```c
XrShape *new_shape = xr_shape_transition(X, shape, symbol);
if (new_shape) {
    xr_json_set_shape(json, new_shape);   // 切换到包含新字段的 Shape
    int new_idx = new_shape->field_count - 1;
    if (new_idx < shape->in_object_capacity)
        json->fields[new_idx] = value;    // in-object 区
    else
        /* overflow 区动态扩展 */;
}
```

XrJson 存储分两层：
- **in-object fields**：创建时分配的固定容量数组（`json->fields[]`），O(1) 索引
- **overflow**：超出 in-object 容量后用 `XrJsonOverflow` 动态数组

Sealed Shape 的保护就是在 `xr_shape_transition()` 入口处检查 `is_sealed`，拒绝创建新 Shape。

---

## 五层改动

### Layer 1: Xi IR — 新增 Json 专用操作

```c
// xi.h — 替换 XI_ALLOC，新增以下操作：

XI_JSON_NEW,       // 创建 XrJson: aux = XrObjectType* (字段信息)
                   // args[0] = field_count (const int)
                   // Emit → OP_NEWJSON (Shape 在 emit 阶段预构建)

XI_JSON_INIT_F,    // 按 index 初始化: args[0] = json, args[1] = value
                   // aux_int = field_index
                   // Emit → OP_JSON_INIT

XI_JSON_GET_F,     // 按 index 读: args[0] = json, aux_int = field_index
                   // Emit → OP_JSON_GET

XI_JSON_SET_F,     // 按 index 写: args[0] = json, args[1] = value
                   // aux_int = field_index
                   // Emit → OP_JSON_SET
```

**`XI_ALLOC` 的命运：删除。**
- object literal → `XI_JSON_NEW`
- struct literal → `XI_JSON_NEW` (struct literal 目前也用 XI_ALLOC → OP_NEWMAP，同样需要统一)
- Map literal → `XI_MAP_NEW` (已有，不变)

**lower_object_literal() 改写** (`xi_lower_expr.c`):

```c
// BEFORE (当前代码):
XiValue *obj_val = xi_value_new(l->func, l->cur_block, XI_ALLOC, result_type, 1);
obj_val->args[0] = cap;
for (int i = 0; i < n; i++) {
    // XI_INDEX_SET with string key → OP_INDEX_SET → Map.set
    XiValue *key = xi_const_str(..., obj->keys[i]->as.literal.raw_value.string_val, ...);
    XiValue *set = xi_value_new(..., XI_INDEX_SET, ..., 3);
    set->args = {obj_val, key, val_vals[i]};
}

// AFTER (新设计):
XiValue *obj_val = xi_value_new(l->func, l->cur_block, XI_JSON_NEW, result_type, 1);
obj_val->args[0] = xi_const_int(l->func, l->cur_block, count, l->type_int);
obj_val->aux = (void *)result_type;  // 携带 XrObjectType 字段信息
for (int i = 0; i < n; i++) {
    // XI_JSON_INIT_F with direct index → OP_JSON_INIT
    XiValue *init = xi_value_new(..., XI_JSON_INIT_F, ..., 2);
    init->args[0] = obj_val;
    init->args[1] = val_vals[i];
    init->aux_int = i;  // 字段索引
    init->flags |= XI_FLAG_SIDE_EFFECT;
}
```

### Layer 2: Emit — Shape 预构建 + OP_NEWJSON

**xi_emit_json_new()** (`xi_emit_object.c`, 替换 `xi_emit_alloc()`):

```c
XR_FUNC void xi_emit_json_new(EmitCtx *ctx, XiValue *v, uint8_t dst) {
    XrType *type = (XrType *)v->aux;
    int field_count = 0;
    XrShape *shape = NULL;

    if (type && (type->kind == XR_KIND_JSON) && type->object.field_count > 0) {
        field_count = type->object.field_count;
        // 1. Intern symbols for each field name
        SymbolId *syms = /* alloca or temp array */;
        for (int i = 0; i < field_count; i++) {
            syms[i] = xr_symbol_intern(ctx->isolate->symbol_table,
                                       type->object.field_names[i]);
        }
        // 2. Build Shape at emit time (cached per field-name-set)
        shape = xr_shape_new(ctx->isolate, syms, field_count);
        if (type->object.is_sealed) {
            shape->is_sealed = true;
        }
    } else {
        // Dynamic: empty shape
        shape = xr_shape_empty(ctx->isolate);
    }

    // 3. Store Shape pointer as integer constant in constant pool
    int k_idx = add_constant(ctx, xr_int((int64_t)(intptr_t)shape));
    emit_inst(ctx, CREATE_ABC(OP_NEWJSON, dst, (uint8_t)k_idx, 0));
}
```

**xi_emit_json_init_f()** (`xi_emit_object.c`):

```c
XR_FUNC void xi_emit_json_init_f(EmitCtx *ctx, XiValue *v, uint8_t dst) {
    (void)dst;
    uint8_t json_reg = reg_of(ctx, v->args[0]);
    uint8_t val_reg  = reg_of(ctx, v->args[1]);
    int field_idx    = (int)v->aux_int;
    emit_inst(ctx, CREATE_ABC(OP_JSON_INIT, json_reg, (uint8_t)field_idx, val_reg));
}
```

### Layer 3: Analyzer — Sealed 字段解析 + 索引内联

**字段访问优化** (`xanalyzer_visitor_expr.c`):

当 receiver 类型是 `XR_KIND_JSON` 且 `field_count > 0` 时：

```c
// obj.fieldName 访问
if (recv_type->kind == XR_KIND_JSON && recv_type->object.field_count > 0) {
    int field_idx = -1;
    for (int i = 0; i < recv_type->object.field_count; i++) {
        if (strcmp(recv_type->object.field_names[i], field_name) == 0) {
            field_idx = i;
            break;
        }
    }
    if (field_idx >= 0) {
        // 标注 field index 供 lower 使用
        node->links.json_field_index = field_idx;
        node->links.result_type = recv_type->object.field_types[field_idx];
    } else if (recv_type->object.is_sealed) {
        // sealed Json 访问不存在的字段 → 编译期报错
        xa_error(ctx, "field '%s' not found in sealed type '%s'",
                 field_name, recv_type->object.type_name);
    }
    // else: open Json with declared fields, field not found → fallback to dynamic
}
```

**const 字段写保护** (`xanalyzer_visitor_expr.c` / assignment check):

```c
// obj.fieldName = value 赋值
if (field_idx >= 0 && recv_type->object.field_readonly
    && recv_type->object.field_readonly[field_idx]) {
    xa_error(ctx, "cannot assign to readonly field '%s'", field_name);
}
```

**Xi Lower 利用 field index** (`xi_lower_expr.c`):

```c
// lower_member_access() — 改写
if (node->links.json_field_index >= 0 && recv_type->kind == XR_KIND_JSON) {
    // 编译器已知 field index → XI_JSON_GET_F (直接索引)
    XiValue *get = xi_value_new(l->func, l->cur_block,
                                XI_JSON_GET_F, result_type, 1);
    get->args[0] = obj;
    get->aux_int = node->links.json_field_index;
    return get;
}
// fallback: XI_LOAD_FIELD (走 GETPROP → Shape IC)
```

### Layer 4: Shape — is_sealed 运行时保护

**xshape.h:**
```c
struct XrShape {
    ...
    bool is_sealed;           // NEW: reject field additions via transition
};
```

**xshape.c — xr_shape_transition():**
```c
XrShape *xr_shape_transition(XrayIsolate *X, XrShape *from, SymbolId sym) {
    // Sealed shape rejects extension
    if (from->is_sealed) {
        return NULL;  // caller should raise runtime error
    }
    // ... existing transition logic ...
}
```

**xjson.c — xr_json_set() 增加 sealed 检查:**
```c
void xr_json_set(XrayIsolate *X, XrJson *json, SymbolId sym, XrValue val) {
    XrShape *shape = xr_json_shape(json);
    int idx = xr_shape_find(shape, sym);
    if (idx >= 0) {
        xr_json_set_field(json, (uint16_t)idx, val);  // existing field — always OK
        return;
    }
    // New field: check sealed
    if (shape->is_sealed) {
        xr_runtime_error(X, "cannot add field to sealed Json object");
        return;
    }
    // ... existing transition + set logic ...
}
```

### Layer 5: AOT — xrt_json_t 替代 xrt_map_t

**新增 `xrt_json_t`** (`xrt_coll.h`):

```c
// Fixed-field Json: fields stored as XrValue array, O(1) indexed access
typedef struct {
    int64_t field_count;
    XrValue fields[];  // flexible array member
} xrt_json_t;

static inline XrValue xrt_json_new(int n) {
    xrt_json_t *j = (xrt_json_t *)XRT_MALLOC(sizeof(xrt_json_t) + n * sizeof(XrValue));
    if (!j) { fprintf(stderr, "xrt_json_new: OOM\n"); abort(); }
    j->field_count = n;
    for (int i = 0; i < n; i++)
        j->fields[i] = (XrValue){.i = 0, .tag = XR_TAG_NULL};
    return xr_mkptr(j, XR_TAG_JSON);
}

static inline XrValue xrt_json_get(XrValue jv, int idx) {
    return ((xrt_json_t *)jv.ptr)->fields[idx];
}

static inline void xrt_json_set(XrValue jv, int idx, XrValue val) {
    ((xrt_json_t *)jv.ptr)->fields[idx] = val;
}
```

**xi_cgen.c 改写:**

```c
// XI_JSON_NEW → xrt_json_new(field_count)
case XI_JSON_NEW: {
    XrType *type = (XrType *)v->aux;
    int fc = type ? type->object.field_count : 0;
    fprintf(out, "xrt_json_new(%d)", fc);
    break;
}

// XI_JSON_INIT_F → xrt_json_set(obj, index, val)
case XI_JSON_INIT_F: {
    fprintf(out, "xrt_json_set(");
    emit_vref(out, v->args[0]);
    fprintf(out, ", %d, ", (int)v->aux_int);
    emit_vref(out, v->args[1]);
    fprintf(out, ")");
    break;
}

// XI_JSON_GET_F → xrt_json_get(obj, index)
case XI_JSON_GET_F: {
    fprintf(out, "xrt_json_get(");
    emit_vref(out, v->args[0]);
    fprintf(out, ", %d)", (int)v->aux_int);
    break;
}

// XI_JSON_SET_F → xrt_json_set(obj, index, val)
case XI_JSON_SET_F: {
    fprintf(out, "xrt_json_set(");
    emit_vref(out, v->args[0]);
    fprintf(out, ", %d, ", (int)v->aux_int);
    emit_vref(out, v->args[1]);
    fprintf(out, ")");
    break;
}
```

---

## 全部受影响文件清单

### 删除 XR_KIND_OBJECT (约 20 处)

| 文件 | 改动 |
|------|------|
| `src/runtime/value/xtype.h` | 删除枚举值、合并 case、删除 `XR_TYPE_IS_OBJECT` 宏、`XrObjectType` 增加 `is_sealed` |
| `src/runtime/value/xtype.c` | 删 `xr_type_new_object`/`xr_type_new_object_anonymous`，给 `xr_type_new_json_with_fields` 增加 `is_sealed` 参数 |
| `src/runtime/value/xtype_format.c` | `XR_KIND_OBJECT` case 合并入 `XR_KIND_JSON` |
| `src/runtime/class/xreflect_method.c` | 合并 case |
| `src/frontend/analyzer/xanalyzer_mono.c` | 合并 case |
| `src/frontend/analyzer/xanalyzer_builtin_interfaces.c` | `is_json_type()` 简化 |
| `src/frontend/parser/xparse_type.c` | `allow_extension` → 统一调用 `xr_type_new_json_with_fields(..., is_sealed=!allow_extension)` |
| `src/ir/xi_dump.c` | 合并 case |
| `src/ir/xi_emit_arith.c` | 合并 case |

### Xi IR 新增 Json ops

| 文件 | 改动 |
|------|------|
| `src/ir/xi.h` | 新增 `XI_JSON_NEW` / `XI_JSON_INIT_F` / `XI_JSON_GET_F` / `XI_JSON_SET_F`，删除 `XI_ALLOC` |
| `src/ir/xi_lower_expr.c` | 重写 `lower_object_literal()` 和 `lower_struct_literal()` |
| `src/ir/xi_emit_object.c` | 删 `xi_emit_alloc`，新增 `xi_emit_json_new` / `xi_emit_json_init_f` / `xi_emit_json_get_f` / `xi_emit_json_set_f` |
| `src/ir/xi_emit.c` | 更新 dispatch table |
| `src/ir/xi_emit_internal.h` | 更新函数声明 |

### Analyzer 字段索引 + const 检查

| 文件 | 改动 |
|------|------|
| `src/frontend/analyzer/xanalyzer_visitor_expr.c` | member access 增加 json field index 解析；const 字段赋值检查 |
| `src/frontend/analyzer/xanalyzer_symbol.h` | `XaSymbolLinks` 增加 `json_field_index` |

### Shape sealed flag

| 文件 | 改动 |
|------|------|
| `src/runtime/object/xshape.h` | `XrShape` 增加 `is_sealed` |
| `src/runtime/object/xshape.c` | `xr_shape_transition()` 检查 sealed |
| `src/runtime/object/xjson.c` | `xr_json_set()` 检查 sealed |

### AOT

| 文件 | 改动 |
|------|------|
| `src/aot/xrt_coll.h` | 新增 `xrt_json_t` / `xrt_json_new` / `xrt_json_get` / `xrt_json_set` |
| `src/aot/xi_cgen.c` | 新增 `XI_JSON_NEW` / `XI_JSON_INIT_F` / `XI_JSON_GET_F` / `XI_JSON_SET_F` 的 C codegen，删除 `XI_ALLOC` 处理 |

### JIT (可延后)

| 文件 | 改动 |
|------|------|
| `src/jit/xi_to_xm.c` | 新增 `XI_JSON_*` → Xm 的 lowering（已有 OP_NEWJSON 支持） |
| `src/jit/xm_tfa.c` | 已经处理 `OP_NEWJSON`，无需修改 |

---

## 实施顺序

### Round 1: Shape sealed + 类型系统统一
1. `XrShape` 增加 `is_sealed` 字段
2. `XrObjectType` 增加 `is_sealed` 字段
3. 删除 `XR_KIND_OBJECT`，全部合并入 `XR_KIND_JSON`
4. 更新所有 ~20 处 `XR_KIND_OBJECT` 引用
5. `xparse_type.c` 的 `allow_extension` → 统一调用 `xr_type_new_json_with_fields(..., is_sealed)`
6. ✅ 测试：ctest 全量通过（行为不变，仅类型系统清理）

### Round 2: Xi IR → OP_NEWJSON + OP_JSON_INIT
1. `xi.h` 新增 `XI_JSON_NEW` / `XI_JSON_INIT_F`，删除 `XI_ALLOC`
2. `xi_lower_expr.c` 重写 `lower_object_literal()` / `lower_struct_literal()`
3. `xi_emit_object.c` 新增 emit 函数，构建 Shape 存入常量池
4. `xi_emit.c` 更新 dispatch table
5. ✅ 测试：ctest + regression diff tests（VM 行为应等价）

### Round 3: 编译期字段索引 + const 保护
1. `xanalyzer_symbol.h` 增加 `json_field_index`
2. `xanalyzer_visitor_expr.c` 解析 sealed/open Json 已知字段 → field index
3. `xanalyzer_visitor_expr.c` const 字段赋值 → 编译期报错
4. `xi_lower_expr.c` 利用 index → `XI_JSON_GET_F` / `XI_JSON_SET_F`
5. Emit → `OP_JSON_GET` / `OP_JSON_SET`
6. ✅ 测试：ctest + regression + sealed 字段访问报错 + const 赋值报错

### Round 4: AOT
1. `xrt_coll.h` 新增 `xrt_json_t`
2. `xi_cgen.c` 增加 `XI_JSON_NEW` / `XI_JSON_INIT_F` / `XI_JSON_GET_F` / `XI_JSON_SET_F` codegen
3. ✅ 测试：AOT test suite + VM-AOT diff = zero

### Round 5: Sealed 运行时保护
1. `xr_shape_transition()` 拒绝 sealed 扩展
2. `xr_json_set()` sealed 新字段 → runtime error
3. ✅ 测试：sealed violation 应报错的测试用例

---

## 性能预期

| 操作 | 当前 | 改后 | 提升 |
|------|------|------|------|
| `{x:1,y:2}` 创建 | OP_NEWMAP + 2×OP_INDEX_SET | OP_NEWJSON + 2×OP_JSON_INIT | **~3x** (无 hash、无 key 装箱) |
| `obj.x` 读取 (VM) | OP_INDEX_GET→Map O(n) | OP_JSON_GET→fields[i] O(1) | **~5-10x** |
| `obj.x` 读取 (IC) | 无 (Map 无 IC) | Shape IC O(1) | **∞ → O(1)** |
| `obj.x` 读取 (AOT) | xrt_map_get O(n) | xrt_json_get O(1) | **~5-10x** |
| sealed 类型安全 | 无检查 | 编译期报错 + 运行时保护 | 正确性提升 |
| const 字段保护 | 无检查 | 编译期报错 | 正确性提升 |

---

## 与 064-json-type-system-refactor 的关系

| 维度 | 064 文档 | 本文档 |
|------|---------|--------|
| 焦点 | 语义层：删除 JsonValue、Json 语义扩展、json stdlib 合并、Json.decode | 实现层：运行时统一 XrJson+Shape、编译流水线、性能优化 |
| XR_KIND_OBJECT | 新增（从 XR_KIND_JSON 拆分） | 删除（合并回 XR_KIND_JSON + is_sealed） |
| 互斥？ | 否 | 否 |

**决策：** 两个文档描述了同一问题的不同视角。064 的 S2（新增 XR_KIND_OBJECT）和本文档（删除 XR_KIND_OBJECT）存在方向矛盾。
最终方案以本文档为准：**不新增 XR_KIND_OBJECT，而是用 `is_sealed` flag 区分**。理由：
- 运行时统一为 XrJson + Shape，无需按 kind 分发
- `...` 修饰符的 sealed/open 语义天然映射到 `is_sealed` flag
- 减少 switch-case 分支，降低维护成本
