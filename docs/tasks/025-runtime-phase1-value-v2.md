# Runtime `value/` 层分析 V2（`025`）

> 旧版 `025-runtime-phase1-value.md` 的对账重写。严格对齐当前源码，按"不考虑兼容性"原则给最佳设计。
>
> **范围**：`src/runtime/value/` 全部 36 个文件 + 必要消费者。

## 1. 七条核心结论

- **`runtime/value/` 是 runtime 全栈的"共享契约层"**。VM、JIT、AOT、analyzer、codegen、DAP、profiler、GC 都依赖。这一点 V1 判断准确。

- **`XrValue` 16-byte struct-of-union 是项目里最干净的基础设计**。V1 没充分肯定。布局有 `XR_STATIC_ASSERT`、ARM64 `ldrb` 优化、`descriptor` bulk-load、`heap_type` cache。

- **`XrProto` 才是真正的 god struct**，55 个字段、跨 8 类语义。V1 大致对，V2 字段级清单完整。

- **类型系统是 3 层，不是 V1 说的 4 层**：进程级 frozen singletons / per-pool arena / per-isolate active pool pointer。TLS 只是为了找到 isolate，不是独立层。

- **真正的 file-scope mutable global 只有 2 个**：`xtype.c:g_types_initialized`、`xchunk.c:s_proto_id_counter`。V1 把 `g_type_int` 等 frozen singleton 一起列入风险点不准确。

- **TLS isolate 在 value 层穿透只有 6 个点**，不是 V1 暗示的"广泛"。5 个可一行参数改成显式传 isolate。

- **`xstruct_layout.h:118` 裸 `case 0/1/2/3` 是真问题但带注释**，`XrTypeKind` 枚举值长期固定。P2 不是 P0。

## 2. 子模块地图（修正版）

实际 36 文件，按真实职责分 5 类：

| 类别 | 文件 |
|---|---|
| **value-core** | `xvalue.h/c`、`xvalue_hash.h/c`、`xslot_type.h` |
| **type-system** | `xtype.h/c`、`xtype_pool.h/c`、`xtype_format.c`、`xtype_generic.c`、`xtype_internal.h`、`xtype_names.h/c`、`xtype_feedback.h/c`、`xtype_opt_hint.h/c` |
| **bytecode-contract** | `xchunk.h/c`、`xopcode_def.h`、`xopcode_info.h/c`、`xstruct_layout.h/c` |
| **presentation** | `xvalue_format.h/c`、`xvalue_print.h/c` |
| **builtin-dispatch** | `xmethod_table.h/c`、`xbool/xint/xfloat_methods.h/c` |

## 3. 关键论断核对

### 3.1 `XrValue` 设计基线（V1 漏）

`xvalue.h:76-99`：tag@0 / flags@1 / heap_type@2-3 / ext@4-7 / payload@8-15。
- `descriptor` 是 `[0-7]` uint64 别名，可一次 load 全部元数据。
- `heap_type` cache 让 `XR_IS_ARRAY(v)` 不需要解引用 GC header。
- `STRUCT_REF + ext != 0` 同时编码 array-ref 和 nested-struct-ref。
- 浮点 NaN 在 `xr_value_deep_eq` 里显式判 `isnan`，避开 IEEE 陷阱。

V2 应作为基线保留，零修改建议。

### 3.2 `xr_value_typeid` 双表 lookup（V1 漏）

`xvalue.c:117-162` 用两张静态表（`tag_to_typeid[8]` 和 `gctype_to_typeid[XR_TTASK+1]`），两次表查完成 value→tid 映射。值得作为正面示范。

### 3.3 `DEFINE_VALUE_OPS_*` X-macro（V1 漏）

`xvalue.c:262-285` 集中生成 `xr_value_from/is/to_*` 三族函数，节省 200+ 行重复。`DateTime` 是唯一例外（手写），建议消除不一致。

### 3.4 类型系统真实层次（V1 误读为 4 层）

实际 3 层：

- **Process-level frozen singletons**（`xtype.c:31-46`）：`g_type_int` 等 14 个，`init_singleton` 后 `frozen=true`，跨 isolate 共享是设计意图。
- **Per-pool arena**（`xtype_pool.c`）：`XrTypePool` 自包含，pool 销毁一次性回收。`next_type_id` per-pool。
- **Per-isolate active pool pointer**：`isolate->current_type_pool`。`resolve_isolate(X)` 仅在显式传 NULL 时 fallback 到 TLS 找 isolate，TLS 不是独立层。

### 3.5 file-scope global 精确清单（V1 笼统）

只有两个真正可变：

- `xtype.c:28` — `static _Atomic bool g_types_initialized`：用 atomic CAS 保证一次性 init，跨 isolate 共享。
- `xchunk.c:36` — `static _Atomic uint32_t s_proto_id_counter`：单调递增 proto id。

`g_type_int` 等 14 个 `static XrType` 不属于"可变全局"——`init_singleton` 后只读。V2 修正：从风险点剔除。

### 3.6 `proto_id` 风险评估温和化（V1 夸大）

V1 说"IC table 被迫向很大稀疏索引增长"。源码（`xexec_frame.h:182`、`xchunk.c:36`）：IC tables 是 `XrVMContext` 上 lazy grow 的指针数组。
- 新 isolate 不会一次性分配大数组。
- 长寿进程中 `proto_id` 单调累加，IC tables 持续 grow，但永远稀疏（每槽 8 字节指针 + 子表 lazy alloc）。
- 真实开销可控，是 P2 不是 P0。

### 3.7 `XrProto` 字段全清单（V1 不全）

源码 `xchunk.h:222-366` 实际 55 字段，分 12 组：

| 组 | 字段数 | 代表 |
|---|---|---|
| 编译期 bytecode | 11 | code/constants/protos/upvalues/lineinfo/locvars/source_file/name/return_type/maxstacksize/numparams |
| 符号与 shared | 4 | symbols/symbol_count/shared_offset/num_globals |
| struct ABI | 3 | num_spill_slots/struct_area_size/struct_layouts |
| 测试属性 | 3 | test_attr/test_timeout/is_coro_safe |
| raw constants | 3 | raw_constants/count/capacity |
| 类型管线（compile-inferred） | 5 | param_types/inst_types/return_type_info |
| JIT compile metadata | 6 | bb_leaders/inline_hint/is_recursive/tfield_float_bitmap/loop_headers |
| JIT runtime state | 11 | jit_entry/type_feedback/call_count/exec_count/deopt_count（Atomic）/jit_opt_level/deopt_backoff |
| OSR / Deopt | 4 | osr_entries/nosr/deopt_table/ndeopt |
| GC stack maps | 2 | stack_map（JIT）/bc_stackmap（解释器） |
| Blueprint | 1 | compiler-generated JIT meta |
| Identity | 2 | proto_id/enclosing |

### 3.8 `xr_value_to_string` 已显式 isolate（V1 描述过时）

`xvalue_format.c:357` 当前实现：`XR_DCHECK(isolate != NULL)`。**已没有 TLS fallback**。

唯一仍走 `xray_isolate_current()` 的是 `xvalue_print.c:46, 314`，那是 VM 没起来时早期 print 的合理 fallback。

### 3.9 `xr_type_to_string` 返回值生命周期（V1 准确）

`xtype_format.c:27`：简单类型返回 `TYPE_NAME_*` 静态常量；复杂类型 `xr_pool_strdup(pool, buf)`，pool 由 `xr_type_get_current_pool()` 取（不传 isolate）。

调用契约 = "pool 存在且未 reset/destroy"。隐藏在签名外。V2 建议改为显式 `(XrTypePool *pool, XrType *type)`。

### 3.10 `xstruct_layout.h:118` 裸数字

带注释 `// XR_KIND_BOOL` 等，但应该直接 `case XR_KIND_BOOL:`。零代价修复。

### 3.11 `xr_valuearray_add` deep_eq 依赖（V1 准确）

`xchunk.c:57-73` 用 `xr_value_deep_eq` 做 dedup → JSON 走 TLS 取 shape。编译期常量 99% 是 primitive，shallow + 类型 ID + memcmp(string) 就够。V2 建议简化。

### 3.12 `xint_methods.h` 反向 include `xcoroutine.h`（V1 准确）

为了 `toBigInt()` 内联实现需要 `xr_current_coro()`。修复：`toBigInt` 移到 `.c`，`.h` 只留 extern declaration。

## 4. 状态归属（修正版）

| 模块 | 真实所有者 | 是否依赖 isolate |
|---|---|---|
| `g_type_*` singletons | 进程（const-after-init） | 否 |
| `XrTypePool` | analyzer / isolate | 否（自包含） |
| `current_type_pool` 指针 | isolate | 是 |
| `XrProto` | compiler / module | 否 |
| `proto_id` counter | 进程（atomic） | 否 |
| IC tables | `XrVMContext`（per-coro） | 间接 |
| `tag_to_typeid`/`gctype_to_typeid` | 进程（const） | 否 |
| `xr_builtin_method_tables` | 进程（const） | 否 |
| `XrStructLayout` | `XrClass` | 否 |
| `type_feedback`/`bc_stackmap`/`stack_map`/`Blueprint`/`osr/deopt` | `XrProto` | 否 |

## 5. 高风险点（精确版）

### 5.1 file-scope mutable global（2 个）

- `g_types_initialized`：删除。改为幂等纯写入。
- `s_proto_id_counter`：移到 `XrayIsolate.next_proto_id`，per-isolate 单调。

### 5.2 `XrProto` god struct（P0）

55 字段直接暴露在头文件。建议拆：

```c
typedef struct XrProto {
    XrProtoCore     core;       // bytecode + constants + symbols
    XrProtoTypes    types;      // param/inst types
    XrProtoStruct   structs;    // struct_layouts
    XrProtoTest     test;       // test_*
    XrProtoCallinfo callinfo;   // entry_type, numparams, ...
    XrProtoMeta    *meta;       // optional: blueprint, bc_stackmap
    XrProtoJitState *jit_state; // optional: jit_entry, feedback, osr
    XrProto        *enclosing;
    uint32_t        proto_id;
} XrProto;
```

`meta`/`jit_state` 可选指针：bytecode-only bundle 可省 ~30% 内存。

### 5.3 TLS isolate 穿透 6 个点

精确清单（`grep xray_isolate_current src/runtime/value/`）：

- `xvalue.c:325` — `xr_json_equals_deep` 取 shape
- `xvalue_print.c:46` — `xr_value_fprint` 早期 fallback（合理）
- `xvalue_print.c:189` — `dump_value_internal` json shape
- `xvalue_print.c:314` — `dump_value_internal` 早期 fallback（合理）
- `xtype.c:86, 92, 99, 714` — `set/get_current_pool`、`resolve_isolate`、`xr_type_copy`

修复：`xr_json_equals_deep(X, a, b)` 显式 → constant pool dedup 不再依赖 TLS；`xtype.c` 删 `resolve_isolate`，强制 X 非空。

### 5.4 其他

- `xstruct_layout.h:118` 裸数字 → 用 `case XR_KIND_*:`
- `xr_valuearray_add` deep_eq → shallow + type id + memcmp
- `xr_type_to_string` 显式 pool 参数

## 6. 正向资产

V1 已识别、V2 强化记录的：

- `XrValue` 16-byte struct-of-union — 基线设计
- `xr_value_typeid` 双表 lookup — value/type 桥的范例
- `DEFINE_VALUE_OPS_*` X-macro — 减重复
- `xmethod_table` 单一真相源
- `xopcode_def.h` X-macro — opcode/info/jumptab 三处自动同步
- `xtype_pool` arena — 与 isolate 解耦
- `xexec_state.h` 注释级文档化范例（024 V2 已肯定）

## 7. 最佳设计建议

按"无兼容性"原则，10 条直接可执行：

1. 拆 `XrProto`（§5.2）
2. 删 `xtype.c:g_types_initialized`，幂等 init
3. `proto_id` 移到 isolate
4. value 层 strict isolate 化（删 `resolve_isolate`、所有 fallback）
5. `xr_type_to_string(XrTypePool *pool, XrType *type)` 显式签名
6. `xr_valuearray_add` 改 shallow eq
7. `xstruct_layout.h:118` 用枚举常量
8. builtin method `.h` 不反向 include 高层（`toBigInt` 移 `.c`）
9. 物理重组目录：value-core / types / bytecode / format / dispatch
10. `XrCFunction` 类型族搬出 `xexec_frame.h` 到 `runtime/cfunction.h`（与 024 V2 §6.x 协同）

## 8. 给下一轮 `gc/closure/` 的输入

- `XrSlotType` 枚举到 `XrGCHeader` traversal 的接线（`xstruct_layout.h:55` 提到 `XR_NATIVE_STRING` 是 GC-traced）
- `XrProto.bc_stackmap` 是 per-PC live mask，与 GC 安全点关系
- `xtype_feedback` 是否在 GC 周期被 traverse（应该不需要，纯统计）
- IC tables ctx-local 释放 vs proto 永久存在的语义边界
- `gctype_to_typeid` 数组长度是 `XR_TTASK + 1`，扩展类型超出范围怎么处理

## 9. 与 V1 的主要差异

| 点 | V1 | V2 |
|---|---|---|
| `XrValue` 设计 | 简单提 | 详述布局，列为基线 |
| 类型系统层次 | 4 层 | 3 层 |
| file-scope mutable global | "两类" | 精确 2 个（其他是 frozen singleton） |
| `proto_id` 风险 | 夸大 | 温和（P2） |
| `XrProto` 字段表 | 大类 | 55 字段 12 组全列 |
| `xr_value_to_string` | 依赖 fallback | 实际已显式 |
| `xr_value_typeid`/X-macro/双表 | 漏 | 列为正面示范 |
| 改进建议 | 6 条方向 | 10 条可执行 |

## 10. 本轮状态

- **已完成**：value/ 36 文件全核对、V1 论断逐项判断、最佳设计 10 条
- **未完成**：value 层 API consumer 量化（留给后续 audit）
- **下一步**：`026-runtime-phase2-gc-closure-v2.md`
