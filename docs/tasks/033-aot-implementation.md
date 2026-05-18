# AOT 缺陷清单与实施方案（仅基于源码分析）

> 日期：2026-04-27
> 状态：Draft
> 范围：`src/aot/`、`src/app/cli/xcmd_build.c::cmd_build_native`、`src/jit/xir_intrinsic.h`、`src/jit/xir_builder*.c`（`aot_mode` 分支）、`tests/aot/`
> 配套文档：`031-aot-architecture.md`（目标架构）、`032-aot-binary-size.md`（体积策略）

---

## 0. 文档定位

本文是**实施卷**，回答一个问题：**当前代码距离 031 描述的目标架构差什么、按什么顺序补齐**。

不重新论证架构（在 031），不讨论体积切割路径（在 032）。

方法论：仅阅读 `src/aot/` 全部源码、`src/app/cli/xcmd_build.c` 中 AOT 相关函数、`src/jit/xir_intrinsic.h`，未参考任何已有 task / design / memory，结论与方案完全来自代码事实。

设计原则：**不保兼容、直选最佳设计、删半成品**——与 `004-aot-refactor.md` 一致。

---

## 1. 现状盘点

### 1.1 文件清单

| 文件 | 行数 | 角色 |
|---|---:|---|
| `xcgen.c` | 1576 | 模块/函数编译入口、prescan、装配 |
| `xcgen_call.c` | 1620 | call 翻译（含巨大 fn_ptr if-ladder） |
| `xcgen_expr.c` | 1171 | 单条指令翻译 |
| `xcgen_stmt.c` | 243 | block terminator + phi |
| `xcgen_struct.c` | 499 | Json shape → C struct promotion |
| `xcgen_bridge.h` | 60 | JIT helper 前向声明（仅地址比较） |
| `xrt.h` | 41 | umbrella |
| `xrt_value.h` | 129 | tag union、boxing |
| `xrt_arc.h` | 204 | ARC + bump allocator |
| `xrt_arith.h` | 119 | tagged 算术 |
| `xrt_coll.h` | 277 | array/map/strbuf/closure（**裸 malloc**） |
| `xrt_method.h` | 483 | 三个固定 arity 的方法分发器 |
| `xrt_compat.h` | 59 | 源码级 `XrValue` / `XR_TAG_*` 别名 |
| `xrt_module.h` | 93 | 模块导出表（**完全未接入**） |
| `xrt_exception.h` | 84 | setjmp/longjmp |
| `xrt_class.h` | 160 | type table + vtable（**vtable 未使用**） |
| `xrt_runtime.h` | 77 | XrtRuntime（**完全未接入**） |
| `xrt_builtin_table.h` | 166 | X-macro（**未被 codegen 消费**） |

`xcgen_call.c` 1620 行、`emit_call_c` 单函数 ~900 行——**远超 150 行单函数上限**。

### 1.2 当前执行链路

```
xray build --native
  └─ src/app/cli/xcmd_build.c::cmd_build_native (1086 行)
       │  bundle 拓扑 → 逐模块编译 → collect_exports / build_shared_proto_map
       │  / aot_preregister_classes / xcgen_collect_shapes
       │  → aot_build_export_map → 逐 proto xir_build → xcgen_compile_func
       │  → xcgen_emit_source → 写 .c → invoke_cc_standalone
       └─ aot_write_main 直接 emit `int main()` 调用每个 mod_init(NULL)
```

**问题**：核心 600 行编排逻辑写在 `app/cli/xcmd_build.c` 而非 `src/aot/`。

---

## 2. 缺陷清单

每条缺陷给一个稳定 ID（`R1..R18`），后续 commit / 测试 / grep 都以此 ID 引用。

### R1 — `emit_call_c` 默认静默丢弃未识别 helper（严重）

`xcgen_call.c:1515`：

```c
// Default: suppress unknown CALL_C targets (dead code in AOT)
cf->call_args_count = 0;
```

任何 CALL_C 的 `fn_ptr` 不在巨大的 if-ladder 里，整个 call **被默默丢掉**——dst 寄存器读到不确定值。这是 substring/str_repeat/charAt 等多个历史 bug 的根源。

### R2 — 反向扫 bytecode 恢复元数据（严重）

| 推断对象 | 位置 | 扫描窗口 |
|---|---|---|
| 字段类型 | `xcgen_struct.c::infer_field_type` | 64 条指令 |
| 寄存器值类型 | `infer_from_backward_scan` | 64 条 |
| `shared_index → child_proto` | `xcmd_build.c::build_shared_proto_map` | 全函数 |
| 类预注册（含 super） | `aot_preregister_classes` | 全函数 |
| 导出名 → shared_index | `collect_exports` | 全函数 + MOVE 链 16 跳 |
| 构造函数字段数 | `emit_call_known` 扫 OP_TFIELD_SET | 整 ctor |
| 方法 'this' 类型 | `collect_exports` 同段 | 反向 patch |

bytecode 一旦改动，pattern 全失效；失败 = 静默退化。

### R3 — `xrt_modules[]` 表生成时 `init_fn = NULL`（严重）

`xcgen.c:1559` 永远写 `NULL`；`xrt_modules_init / xrt_module_lookup` **从未被调用**。整个 `xrt_module.h` 是死代码。实际跨模块访问走 `xrt_shared[]`。

### R4 — 容器对象事实上"alloc 后永不释放"（严重）

`xrt_coll.h` 的 `xrt_array_t / xrt_map_t / xrt_strbuf_t / xrt_closure_t`：

* 不带 `XrtArcHdr`，直接 `XRT_MALLOC`；
* 没有对应的 `*_free`；
* `xrt_arc_release_val` 只处理 PTR / STR_ARC，**显式跳过** array/map/closure/strbuf。

→ 默认场景下 AOT 程序持续泄漏。

### R5 — bump allocator 默认关，注释说"default: on"

`xrt_arc.h:91`：`int xrt_bump_enabled = 0;` 与第 74 行注释矛盾。即便启用，`xrt_runtime_init`/`xrt_bump_destroy` 也不会运行（见 R6）。

### R6 — 生成的 `main()` 从来不初始化 runtime

`aot_write_main` 只 emit `int main(){ mod_init(NULL); return 0; }`。没有 XrtRuntime 实例，没有 init/shutdown，所有 `xrt_ctx` 永远是 NULL。`xrt_runtime.h` 的 lifecycle 文档与生成代码不符。

### R7 — JIT helper 指针匹配 = AOT/JIT 紧耦合（严重）

`emit_call_c` 是 ~900 行的指针 if-ladder：

```c
if (fn_ptr == (void *)xr_jit_index_get) { ... }
else if (fn_ptr == (void *)xr_jit_throw) { ... }
... 30+ 分支
```

`xcgen_bridge.h` 仅为了取地址做比较而存在。讽刺的是 `src/jit/xir_intrinsic.h` 已经定义了 `XirIntrinsicId` 枚举（GETPROP/INDEX_GET/MAP_SET/THROW/STRBUF_NEW/INVOKE_METHOD…），头注释明确写：

> 3. Update `xcgen_call.c::emit_call_intrinsic` to lower it to C.

但 **`emit_call_intrinsic` 不存在**，`XIR_CALL_INTRINSIC` opcode 也不在 `xir.h` 中。重构计划写了一半就停了。

### R8 — Class deinit 表与 struct deinit 表共享 `XrtArcHdr.type`

`xrt_class.h::xrt_type_register` 让 type_id 从 1 开始递增；`xcgen_struct.c::xcgen_emit_struct_deinits` 又生成 `xrt_deinit_table[nstructs]`，**索引 0..nstructs-1**——和 class type_id 的取值空间**完全重叠**。

```c
// 生成代码
if (type < nstructs && xrt_deinit_table[type]) { 优先 struct }
if (type < xrt_type_count) { class }
```

class type_id == 1 与 struct_idx == 1 冲突时，对该 class 实例 release 调用 struct 的 deinit。当前测试套件未覆盖该混合场景，所以未爆。

### R9 — ARC 生命周期半成品

`xcg_emit_mov` / `xcg_emit_field_store` 在 tagged→tagged 时 retain/release；但函数返回值/参数/本地变量在退出时不 release；闭包/数组/map 没 ARC header；`xrt_str_alloc` rc=1 传给 `xrt_array_push` 后没 retain。**默认 standalone 二进制是双重错误（计数错 + 真释放）的薛定谔态**，靠进程退出兜底。

### R10 — 全局固定大小硬编码

`MAX_STRUCTS=16`、`MAX_STRUCT_FIELDS=32`、`MAX_NONESC_UPVALS=16`、`exc_catch_frame[256]`、`exc_pending_stack[8]`、`exc_has_finally[8]`、`aot_cap=64`、`synth[64]`（**硬截断不报错**）、`chain[16]` MOVE 链、`reg_class[256]`、`XRT_MAX_TYPES=256`、`safe_name[64]`、`hop<8`、64-instruction 扫描窗口、`mp->numparams > 200 → skip`。多数静默截断。

### R11 — 调试 stderr 打印进了 production

`xcgen.c:923`：`fprintf(stderr, "DEBUG_HEAD vreg %u rep=%u\n", i, vt);` —— 每次编译每 vreg 一行。

### R12 — 死字段 / 死代码

| 死代码 | 位置 |
|---|---|
| `cf->shadow_stack_count` | 只读不写；触发的 `xrt_shadow_sp` 全局未声明 |
| `comp->emit_debug` | 只赋值，不读 |
| `comp->single_file` | 硬编码 true |
| `XcgenExport->c_var` | 注释说"populated"，无人写 |
| `xcg_lookup_proto_cf()` | 注释说"避免 dangling"，实际就 dangling |
| `xrt_module.h` 全部 | 见 R3 |
| `xrt_runtime.h` 全部 | 见 R6 |
| `xrt_class.h::xrt_vcall / xrt_instanceof` | codegen 零调用 |
| `xrt_builtin_table.h::xrt_builtin_find` | codegen 不查 |

### R13 — 同语义辅助重复实现

* `xcgen_expr.c::xcg_emit_ref_as_tagged` 与 `xcgen_call.c::emit_ref_as_tagged` 完全相同。
* `xrt_method_N` runtime fallback 与 `xcgen_call.c` 的 inline 特化 codegen——同方法两份实现。

### R14 — 符号 ID 三重声源

`xrt_method.h::XRT_SYM_*`、`xrt_builtin_table.h` X-macro、`xrt_symbol_check.c` _Static_assert、`runtime/symbol/SYMBOL_*`。新增方法要改 4 处。

### R15 — `xrt_arc_deinit` 是 generated 与 runtime 间 API 黑洞

`xrt_arc.h:159` forward decl，由 `xcgen_emit_struct_deinits()` 生成定义。runtime header 语义飘在 codegen 上，runtime 无法独立单测。

### R16 — STORE/LOAD_FIELD 偏移 hack

`xcgen_expr.c:493/580`：`int64_t adj_offset = offset - 24;` —— 魔术数 24 = JIT 侧 `XIR_JSON_FIELDS_OFFSET`，跨层泄露。

### R17 — `xrt_value.h` 与 `xrt_arc.h` 循环

`xrt_value.h` 末尾 forward decl `xrt_str_alloc/concat`，由 `xrt_arc.h` 实现。L0 反向依赖 L1。

### R18 — class 继承全靠 bytecode 后扫

`aot_preregister_classes` 维护 `XrClass *reg_class[256]` 跟踪 OP_MOVE / OP_CLASS_CREATE 推断 super_override，依赖具体 register allocation。

---

## 3. 重构总目标（落 031 架构）

1. **AOT codegen 不依赖任何 JIT 符号地址**——通过 `XirIntrinsicId` 元数据（031 §D6）。
2. **runtime（xrt_*）能独立编译/单测**——不再有 codegen 生成核心定义（031 §3）。
3. **生命周期单一**：默认 per-coroutine Immix GC；bump-only / malloc 是 opt-in（031 §D3 + 032 §3.1）。
4. **AOT 有自己的 driver**：把 `xcmd_build.c` 中所有反扫 bytecode 元数据恢复逻辑挪进 `src/aot/driver/`（031 §D7）。前端 compile 时直接产出元数据（031 §D8）。
5. **没有静默退化路径**：未识别 intrinsic / 未声明类型 / 容器没有 dtor —— 全部硬错误。
6. **没有固定上限的硬编码**：所有 `MAX_*` 都做动态扩张或表驱动。
7. **每文件 ≤ 800/3000 行；每函数 ≤ 150 行**——按红线拆分。

---

## 4. 分模块重构细则（与缺陷一一对应）

### 4.1 Intrinsic 派发：终结指针匹配（解决 R7、R1）

1. 新增 XIR opcode `XIR_CALL_INTRINSIC`（`src/jit/xir.h`）：
   * `args[0]`：const i64 = `XirIntrinsicId`
   * `args[1..]`/`call_arg_pool`：实参（同 CALL_C）
2. `xir_builder*` 在 `b->aot_mode` 下所有 helper site emit `XIR_CALL_INTRINSIC`，不再 bake fn_ptr。JIT mode 保持 CALL_C。
3. 新建 `src/aot/codegen/xcgen_intrinsic.c`：

```c
typedef void (*XcgenIntrinLower)(XcgenBuf *, XirFunc *, XirIns *, XcgenFunc *);

static const XcgenIntrinLower xcgen_intrin_table[XR_INTRIN_COUNT] = {
    [XR_INTRIN_GETPROP]    = lower_getprop,
    [XR_INTRIN_INDEX_GET]  = lower_index_get,
    [XR_INTRIN_THROW]      = lower_throw,
    /* ... */
};

void xcg_emit_intrinsic(XcgenBuf *b, XirFunc *f, XirIns *ins, XcgenFunc *cf) {
    int64_t id = xcg_resolve_const_i64(...);
    XR_CHECK(id > 0 && id < XR_INTRIN_COUNT, "unknown intrinsic %lld", id);
    XR_CHECK(xcgen_intrin_table[id], "intrinsic %s missing AOT lowering",
             xir_intrinsic_name(id));
    xcgen_intrin_table[id](b, f, ins, cf);
}
```

4. **删 `xcgen_bridge.h`**、删 `emit_call_c` 中所有 `xr_jit_*` / `xrt_*_sentinel` 比较。
5. **静默 default 路径删除**：未识别 intrinsic 直接 `XR_CHECK` 失败。

### 4.2 元数据前移（解决 R2、R18）

把 `cmd_build_native` 中所有 pattern matching 移到**前端编译期**，proto 直接带元数据：

| 字段 | 数据 | 替代 |
|---|---|---|
| `XrProto::aot_meta::ctor_field_count` | 类构造函数字段数 | 替代 `emit_call_known` 扫 OP_TFIELD_SET |
| `XrProto::aot_meta::shared_protos[]` | shared_index → child proto | 替代 `build_shared_proto_map` |
| `XrProto::aot_meta::exports[]` | name + shared_index + is_const | 替代 `collect_exports` |
| `XrClassDescriptor::aot_meta::super_class` | 父类 XrClass* | 替代 `aot_preregister_classes` |
| `XrShape::field_xir_types[]`（新增） | 字段语义类型 | 替代 64-window 扫 |

新建 `src/aot/driver/xaot_metadata.{c,h}`：把元数据整理成 codegen 友好结构。CLI 只调用 `xaot_compile_bundle(bundle, output, options)`。`cmd_build_native` 缩减至 ≤ 50 行。

### 4.3 内存模型：默认 GC，bump 可选（解决 R4、R5、R8、R9、R15）

**决策**（与 031 §D3 / 032 §3.1 对齐）：

* **默认 `XR_FEAT_GC_FULL`**：复用 `runtime/gc/xcoro_gc`。
* **`XR_FEAT_GC_BUMP`**：opt-in，不允许搭配 `XR_FEAT_CORO`。
* **`XR_FEAT_GC_NONE`**：实验级，仅栈分配（受限子集）。

实施：

1. **新 `xrt_alloc.h`**：

```c
#if defined(XR_FEAT_GC_FULL)
   #define xrt_alloc(rt, sz, type) xcoro_gc_alloc(...)
#elif defined(XR_FEAT_GC_BUMP)
   #define xrt_alloc(rt, sz, type) xrt_bump_alloc(&(rt)->bump, sz)
#elif defined(XR_FEAT_GC_NONE)
   #error "GC_NONE not yet implemented"
#else
   #error "Pick one XR_FEAT_GC_*"
#endif
```

2. **删除 ARC**：`xrt_arc.h` 的 retain/release/deinit 全删；GC 模式下不需要；bump 模式下进程退出兜底；malloc 模式（如果将来实现）由生成代码显式 free。
3. **删 R8 双 deinit table**：class 的析构由 GC 在 sweep 时调；struct 不再独立维护表。
4. **容器统一**：array/map/strbuf/closure 不再带 ARC header，全走 `xrt_alloc`，纳入 GC 扫描根。
5. **`xrt_arc_deinit` 死亡**：runtime header 不再有"被 codegen 填的占位"。
6. **MOV/STORE_FIELD 移除 retain/release codegen**——纯赋值。
7. **`xrt_bump_enabled` 标志删**——编译期决定。
8. `xcg_emit_terminator` 中 `xrt_shadow_sp` 死代码全删。

### 4.4 真正接入 `XrtRuntime` lifecycle（解决 R6）

`xrt_runtime.h` 改造成 031 §D5 的样子：

```c
typedef struct XrtRuntime {
    XrIsolate     *isolate;
    XrCoroSched   *sched;
    XrtTypeTable   types;
    XrtExcFrame   *exc_top;
    XrValue       *shared;
    int            shared_count;
} XrtRuntime;

void xrt_runtime_init(XrtRuntime *rt, int argc, char **argv);
int  xrt_runtime_run (XrtRuntime *rt, XrtCoroEntryFn entry);
void xrt_runtime_shutdown(XrtRuntime *rt);
```

* 全局可变状态全部进 `XrtRuntime`——满足红线"禁止文件作用域可变全局"。
* 生成 main() 由 `xaot_emit_main.c` 输出：

```c
int main(int argc, char **argv) {
    XrtRuntime rt;
    xrt_runtime_init(&rt, argc, argv);
    xr_mod_a__init(&rt);
    xr_main__init(&rt);
    int rc = xrt_runtime_run(&rt, &co_user_main);
    xrt_runtime_shutdown(&rt);
    return rc;
}
```

* 每个生成函数签名 `xr_foo(XrtRuntime *rt, ...)`；`XrtContext` 别名保留。
* `xrt_modules[] / xrt_module_lookup / xrt_module.h` **全删**。

### 4.5 Class / Struct deinit 统一（解决 R8）

4.3 选 GC 后，dtor 派发由 GC sweep 阶段调用 `XrClass::finalizer`；struct 没有析构需求（POD）。R8 双表问题自然消失。仅保留 type_id 用于反射 / instanceof / 调试 dump。

### 4.6 `xcgen_call.c` 拆分（解决文件膨胀）

* `xcgen_call.c`（≤ 400 行）：CALL_KNOWN / CALL_SELF_DIRECT / CALL_DIRECT 三个真正的"用户代码间调用"。
* `xcgen_intrinsic.c`：4.1 的 intrinsic table + lowering（覆盖所有 helper）。
* `xcgen_class.c`：constructor 调用、虚调用、instanceof。
* `xcgen_method.c`：builtin 方法 inline 特化。

每文件聚焦一种关注点，函数 ≤ 100 行。

### 4.7 方法元数据：单声源 + 表驱动（解决 R13、R14）

`xrt_builtin_table.h` X-macro 变成**唯一权威**：

```c
#define XRT_BUILTIN_METHODS(X) \
    X(LENGTH, "length", 0, RECV(STR|ARR|MAP), RET_I64, IMPL(rt_length, codegen_length))
```

由 X-macro 同时生成：

* `XRT_SYM_*` 常量；
* `xrt_method_dispatch[]` runtime fallback 表；
* `xcgen_method_emit_table[]` codegen 特化表；
* `_Static_assert` 与 `SYMBOL_*` 对齐。

新增方法 = 改 1 个 X-macro 行 + 1 个 runtime 函数 + 1 个 codegen lowering。

### 4.8 配置/限制全部动态化（解决 R10）

| 当前 | 替换 |
|---|---|
| `XCGEN_MAX_STRUCTS=16` / `MAX_STRUCT_FIELDS=32` | dynarray |
| `XRT_MAX_TYPES=256` | dynarray，`XrtRuntime->types` |
| `cf->exc_catch_frame[256]` / `pending_stack[8]` / `has_finally[8]` | dynarray |
| `aot_cap=64` / `synth[64]` / `chain[16]` / `reg_class[256]` | 4.2 后删除 |
| 64-instruction 扫描窗口 | 4.2 后删除 |
| `mp->numparams > 200` | 删除 sanity 上限 |
| `safe_name[64]` / `name_buf[64]` | 动态 strdup |

超限改成 `XR_CHECK` hard fail 而非静默截断。

### 4.9 死代码与 debug 残留清理（解决 R11、R12）

* 立删：`fprintf(stderr, "DEBUG_HEAD ...")`。
* 删字段：`shadow_stack_count`、`emit_debug`、`single_file`、`XcgenExport::c_var`。
* 删函数：`xcg_lookup_proto_cf`（让调用者直接用 func_idx）。
* 删整文件：`xrt_module.h`、`xrt_compat.h`、`xcgen_bridge.h`。

### 4.10 LOAD/STORE_FIELD 偏移泄漏（解决 R16）

4.2 给 shape 加 `aot_meta::field_byte_offsets[]`，codegen 直接读，不再 `offset - 24` 减魔术数。JIT 改 layout 不影响 AOT。

### 4.11 头文件循环（解决 R17）

* `xrt_value.h`：删 `xrt_str_alloc/concat` 的 forward decl。
* 新建 `xrt_string.h`（依赖 `xrt_value.h` + `xrt_alloc.h`）。
* `xrt_arith.h::xrt_add` 改调 `xrt_string.h::xrt_string_concat_value`。
* DAG 严格单向：见 031 §3。

---

## 5. 实施分阶段（每阶段独立可发布）

### S1 · 现状清理（1 PR）

* 删 R11 stderr DEBUG。
* 删 R12 死字段/死代码。
* `cf->shadow_stack_count` 死分支彻底消失。
* 加 `-ffunction-sections / -fdata-sections / -dead_strip` 链接 flag（032 §7 联动）。
* 验证：`ctest` + `tests/aot/run_aot_tests.sh` 全绿。

### S2 · Intrinsic 化（核心，1-2 PR）

* 加 `XIR_CALL_INTRINSIC` opcode + builder 改写（`aot_mode` 全走 intrinsic）。
* 新建 `xcgen_intrinsic.c` 表驱动。
* 删 `xcgen_bridge.h` 与 `emit_call_c` 中所有 helper 比较。
* 默认分支 hard fail。
* 加每条 intrinsic lowering 的单测。

### S3 · 元数据前移（1 PR）

* `XrProto / XrShape / XrClassDescriptor` 加 `aot_meta` 字段（前端 codegen 时填）。
* `cmd_build_native` 反扫函数迁入 `src/aot/driver/xaot_metadata.c`，改为读 `aot_meta`。
* 顺手做 032 §4.1 的 `AotFeatureSet` 推断器（结构骨架）。
* 删 magic 上限 `chain[16] / reg_class[256] / synth[64]` / 64-window。

### S4 · 内存模型决策（1 PR）

* 引入 `XR_FEAT_GC_FULL/BUMP/NONE` 三档。
* 删 `xrt_arc.h` retain/release/deinit；ARC 头仅记 type_id（4.5）。
* 删 `xcgen_emit_struct_deinits / xrt_deinit_table / xrt_arc_deinit` 三联体。
* 删 `xcg_emit_mov / xcg_emit_field_store` 中 retain/release。
* CMake 主仓默认 `XR_FEAT_GC_FULL`；CLI `--gc=full|bump` 选项。

### S5 · runtime lifecycle 真接入（1 PR）

* file-scope global 全部进 `XrtRuntime`。
* `aot_write_main` 输出 `xrt_runtime_init/run/shutdown`。
* 生成函数签名 `XrtRuntime *rt` 替换 `XrtContext`。
* 删 `xrt_module.h`、`xrt_compat.h`。

### S6 · 文件拆分 + driver 抽出（1-2 PR）

* `xcgen_call.c` 拆为 call/class/method/intrinsic 四份。
* `xcgen.c` prescan 拆为 `xcgen_prescan.c`。
* `cmd_build_native` 缩为 ≤ 50 行；AOT 编排进 `src/aot/driver/`。
* `tests/unit/aot/` 建立。
* 落 `--features` CLI 选项 + size report（032 §4.3）。

### S7 · 方法元数据合一（1 PR）

* `XRT_BUILTIN_METHODS` X-macro 同时生成 runtime fallback、codegen 特化、_Static_assert。
* 删 `xrt_method.h` 散户 `if/else` 大块。

### S8 · 头文件 DAG 化（1 PR）

* 拆 `xrt_string.h`，断 R17 循环。
* `xrt_compat.h` 内容并入 `xrt_value.h`。
* `xrt_arith.h` 中字符串相关移除。

### S9 · feature 切割落地 + lint + 文档同步（1 PR）

* 把 `runtime/` 与 `coro/` 切成 OBJECT lib（032 §3.3）。
* 落 `XR_FEAT_CORO/CHANNEL/TIMER/...` 编译期 gate。
* 替换桩 `xrt_*_stubs.c` 给关掉的特性。
* `scripts/check_comment_rules.sh` / `check_architecture.sh` 跑过。
* `clang-format` 重过。
* 同步 `004-aot-refactor.md` / `005-aot-implementation.md`，删除过期文字。

---

## 6. 取舍决策表

| 取舍 | 选择 | 理由 |
|---|---|---|
| ARC vs Bump vs GC | **per-coro Immix（默认）+ Bump opt-in** | 与 VM 同款，CPS 协程下 bump 不可行；ARC 跨容器同步复杂度过高 |
| Intrinsic 集中 vs 分散 enum 文件 | **集中** | 一表一处，零飘移 |
| `xrt_module.h` 改造 vs 删除 | **删除** | `xrt_shared[]` 已是事实路径 |
| `xrt_runtime.h` 完善 vs 删除 | **完善（变成真正运行时根）** | 全局状态须有归宿 |
| Class type_id 与 Struct deinit 表合并 vs 分离 | **GC 化后两表都消失** | 杜绝 R8 |
| 容器加 ARC 头 vs 进 GC | **进 GC** | 不做半 ARC |
| 三套符号 ID 声源 vs X-macro 单源 | **X-macro 单源** | 加方法只改一处 |
| `c-only` 单文件 vs 每模块独立 .c | **每模块独立 + 入口 .c** | 增量编译 + 链接自然 |
| 反扫 bytecode vs 前端元数据 | **前端元数据** | 对语言变更无感 |
| 是否保留 `xcgen_bridge.h` | **删除** | intrinsic 化后零必要 |
| runtime 头 forward decl 由 codegen 填的函数 | **删除该模式** | 头不应被 codegen 寄生 |
| `xrt_compat.h` 独立头 | **删除并入 `xrt_value.h`** | 源码级别名不需要独立文件 |
| AOT 是否嵌 VM 解释器 | **不嵌**，复用 runtime + coro | 见 031 §D1 |
| 协程实现 | **CPS 翻译 + 复用 `src/coro` 调度器** | 见 031 §D2 |

---

## 7. 验证策略

### 7.1 每阶段必过

```bash
cd build && ctest --output-on-failure
tests/aot/run_aot_tests.sh
scripts/run_regression_tests.sh   # VM/AOT diff 必须为 0
```

### 7.2 新增 AOT 单元测试（`tests/unit/aot/`）

| 测试 | 目标缺陷 |
|---|---|
| `test_intrinsic_dispatch.c` | R7 / R1 |
| `test_runtime_lifecycle.c` | R5 / R6 |
| `test_alloc_policy.c` | R4 / R5（GC / bump 两档分别构建） |
| `test_metadata_export.c` | R2 |
| `test_class_dtor_unified.c` | R8 |
| `test_struct_field_offset.c` | R16 |
| `test_no_silent_truncation.c` | R10 |
| `test_feature_gate.c` | 032 §3 — 关掉的 feature 在 user code 用到时 hard fail |
| `test_size_report.c` | 032 §4.3 — auto 推断与显式 `--features` 一致 |

### 7.3 ASAN gating

`-fsanitize=address` 跑 AOT 测试集：

* GC 模式：验证 sweep 后无悬挂指针；
* bump 模式：验证编译期阻止 `XR_FEAT_CORO`；
* feature 关闭模式：验证替换桩调用即终止（有诊断信息）。

### 7.4 反向回归（防 R1/R7 复辟）

CI 加断言：

```bash
! grep -q "Default: suppress unknown" src/aot/codegen/*.c
! grep -q "xr_jit_" src/aot/codegen/*.c
! grep -q "xcgen_bridge.h" src/aot/codegen/*.c
! grep -q "fprintf(stderr, \"DEBUG_" src/aot/codegen/*.c
```

### 7.5 体积回归（联动 032 §8）

CI 加 size 门禁：

```bash
xray build --native examples/hello.xr --features=auto
size build/aot/hello | awk '/__TEXT/{ if ($1 > 600*1024) exit 1 }'
```

每条预设档位（`auto / min / full`）的体积上限作为门禁。

---

## 8. 重构后的不变量

合入后必须永远满足（与 031 §6 联合执行）：

1. AOT codegen 不引用任何 `xr_jit_*` 符号——所有跨层走 `XirIntrinsicId`。
2. runtime 头文件不被 codegen 填充——`xrt_*.h` 自包含、可独立单测。
3. 没有 file-scope 可变全局——所有可变状态在 `XrtRuntime` 内。
4. 没有静默截断或静默丢弃——所有边界 `XR_CHECK` 断言。
5. 所有限制都是动态扩张——除了语言层真正的常量（如 tag bit 位宽）。
6. 生成的 main() 总是成对调用 `xrt_runtime_init/shutdown`。
7. 新增 builtin 方法 = 改 1 X-macro 行 + 1 runtime 函数 + 1 codegen lowering。
8. 新增 intrinsic = 改 1 enum + 1 lowering。
9. 每 .c ≤ 3000 行，目标 < 800；每函数 ≤ 150 行，目标 < 80。
10. `xrt_module.h`、`xrt_compat.h`、`xcgen_bridge.h` 永远不复活。
11. AOT 不依赖 `src/vm/`：grep 无 `#include "xvm`。
12. `--gc=bump|none` 与 `--feat=coro` 同时启用 → 编译期 hard fail（032 §6）。

---

## 9. 风险与缓解

| 风险 | 缓解 |
|---|---|
| `XIR_CALL_INTRINSIC` 改动牵连 JIT | aot_mode 与 jit_mode 分流；先在 aot_mode 全量切换 |
| 元数据前移破坏前端 incremental | `aot_meta` 字段 lazily 填充，仅 build --native 路径计算 |
| GC 切换让原生程序集成更难 | 提供 bump opt-in，文档明确 |
| `aot_preregister_classes` 删除丢 CHA 信息 | S3 阶段保证 super_override 从 ClassDescriptor 直接拿 |
| 大量重命名引入合并冲突 | S1..S9 串行 PR，每个 ≤ 800 行 diff |
| feature 切割爆发组合数 | 自动推断（032 §4.1）覆盖 95% 用例，手动 `+x/-x` 处理剩余 |

---

## 10. 不在本次范围

* JIT 后端本身（codegen / 寄存器分配）
* 协程内部（worker / sched / channel）实现细节
* Stdlib 内部实现优化
* 增量编译
* 调试信息 / source map

这些项一旦本次重构落地，会因干净的 intrinsic + runtime 边界 + feature gate **显著更易**实施。

---

## 附录 A · `XirIntrinsicId` 对应关系（来自 `src/jit/xir_intrinsic.h`）

| Intrinsic | 当前 fn_ptr 形态（待消除） |
|---|---|
| GETPROP | `xr_jit_getprop` |
| INDEX_GET / SET | `xr_jit_index_get` / `xr_jit_index_set` |
| TARRAY_GET / SET | `xr_jit_tarray_get` / `xr_jit_tarray_set` |
| MAP_GET / SET / INCREMENT | `xr_jit_map_*` |
| STRBUF_NEW / APPEND / FINISH | `xrt_strbuf_*_sentinel` |
| SUBSTRING / STR_REPEAT / CHR | `xr_jit_substring` / `str_repeat` / `chr` |
| INVOKE_METHOD | `xrt_invoke_method_sentinel` |
| GET_SHARED / SET_SHARED | `xr_jit_get_shared` / `set_shared` |
| PRINT | `xr_jit_print` |
| RT_ADD / SUB / MUL / DIV / MOD | `xr_jit_rt_*` |
| THROW | `xr_jit_throw` |
| JSON_NEW_SHAPE | `xr_json_new_with_shape` |

枚举已存在；所有 builder 与 lowering 都"差最后一公里"。

## 附录 B · 当前固定上限一览（4.8 全部动态化）

| 常量 | 位置 | 替换 |
|---|---|---|
| `XCGEN_MAX_STRUCT_FIELDS=32` | xcgen_struct.h | dynarray |
| `XCGEN_MAX_STRUCTS=16` | xcgen_struct.h | dynarray |
| `XCGEN_MAX_NONESC_UPVALS=16` | xcgen_call.c | dynarray |
| `XRT_MAX_TYPES=256` | xrt_class.h | `XrtRuntime->types` |
| `cf->exc_catch_frame[256]` | xcgen.h | dynarray，按 block 数 |
| `cf->exc_pending_stack[8]` | xcgen.h | dynarray |
| `cf->exc_has_finally[8]` | xcgen.h | dynarray |
| `chain[16]` / `reg_class[256]` / `synth[64]` | xcmd_build.c | 删除（4.2 后不需要） |
| 64-instruction 扫描窗口 | xcgen_struct.c | 删除（4.2 后） |
| `mp->numparams > 200` | xcmd_build.c | 删除 sanity |
| `hop < 8` MOV 链跳数 | xcgen_call.c | 用 SSA def 直链替代 |

---

## 11. 总结

`src/aot` 当前的核心症结**不是 AOT 算法本身有问题**，而是：

1. **边界没立住**——CLI/codegen/runtime/JIT 互相穿透，`xcmd_build.c` 在做编译器工作，`xcgen_bridge.h` 在引用 JIT 符号，`xrt_arc.h` forward decl 等 codegen 填，全都是边界沦陷。
2. **决策没落地**——bump vs malloc、ARC vs 不要 ARC、`xrt_module` vs `xrt_shared`、`xrt_runtime` lifecycle 接 vs 不接，每一对都同时保留两边的代码，结果是双方都不工作。
3. **静默退化**——R1（默认丢 helper）、R10（硬截断）、R8（错位 dispatch）三个静默错误模式潜伏多处，没爆是因测试覆盖窄。

按本文 S1..S9 重构后：

* 文件 ≤ 800 行、函数 ≤ 100 行；
* 每条规则只有一处实现（intrinsic、方法、内存策略）；
* 边界单向（DAG），删 4 个文件（`xrt_module.h`、`xrt_compat.h`、`xcgen_bridge.h`、`xrt_runtime.h` 中 stale 段）；
* AOT 真正自洽——给一个 standalone 可执行 + 一个 init/shutdown 完整的 runtime；
* 加新方法 / 加新 intrinsic 是单点改动，对语言演进零摩擦；
* 体积按 032 卷的 feature gate 自动推断 + 可手动调整。
