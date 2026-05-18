# AOT 模块重构实施文档（`src/aot` + `src/app/cli/xcmd_build.c`）

> 日期：2026-04-25
> 状态：Draft
> 范围：`src/aot/`（codegen + runtime headers）、`src/app/cli/xcmd_build.c` 中的 `cmd_build_native()`、相关 XIR builder 中带 `aot_mode` 的分支（`src/jit/xir_builder*.c`）、`tests/aot/`
> 设计依据：`docs/design/aot-design.md`、`docs/design/aot-implementation.md` (v2.0)

---

## 1. 文档定位

本文是对当前 AOT 实现状态的**重构落地方案**，配合 `docs/design/aot-implementation.md` 使用：

- 设计文档说明**最终目标**。
- 本文说明**当前距离目标还差什么、怎么收敛**，并把工作切成可独立合并、可独立回滚的 Phase。

### 1.1 与设计文档的关系

设计文档把 AOT 划分为：

- **Phase A**：多模块编译管线 + ARC runtime
- **Phase B**：Class/OOP + 异常 + 类型特化 + 集合
- **Phase C**：CPS 并发（复用 `src/coro/`）
- **Phase D**：stdlib bridge + 多文件输出 + 增量编译

当前实现状态（基于 2026-04-27 审计，详见 `MEMORY[19b2b7f8]`）：

| Phase | 状态 |
|---|---|
| A | 多模块管线已跑通；allocator 适配、runtime 生命周期、bump allocator 都未真正接线 |
| B | finally / defer / match / class 基础已可用；vtable / instanceof / destructor 只搭了壳；enum 未做 |
| C | **完全未开始**：`src/coro` 没有任何 `XRAY_AOT` 接线，没有 AOT 并发 codegen，没有 `tests/aot/concurrency/` |
| D | **完全未开始**：无 `xrt_stdlib_*.h`，`single_file = true` 硬编码，没有增量编译 |

设计文档描述的是 v2.0 终态，但当前代码仍存在多处**临时桥接 + 半接线结构**。本文不重做设计，而是把这些“看起来差不多但没收口”的部分清理掉。

### 1.2 开发原则

遵循全项目“不保兼容、直选最佳设计”的总纪律，本文额外强调：

- **正确性优先于功能扩张**：在 P0 收敛前，不为新语言特性扩展 AOT。
- **删除半成品优先于补半成品**：未接线的 runtime 抽象层应当先删，而不是先补。
- **显式契约优于隐式约定**：禁止再以“函数指针匹配 / bytecode pattern 反扫”作为生产路径。
- **不把结构性债务伪装成性能优化**：临时实现就标 TODO，不写成“最佳实践”。
- **每个 Phase 独立可合并、独立可回滚**。
- **Phase 内每条改造都必须带验收命令**（参考第 7 节）。

---

## 2. 当前模块地图

### 2.1 当前目录结构与规模

```text
src/aot/
├── xcgen.c              1574 行   主 orchestration、模块输出装配、retype 后处理
├── xcgen.h               294 行   公开 API
├── xcgen_call.c         1680 行   CALL_C / CALL_SELF_DIRECT / CALL_KEEP，emit_call_c 路由
├── xcgen_expr.c         1170 行   表达式 / 索引 / SELECT / ARC 操作 lowering
├── xcgen_stmt.c          242 行   控制流终结符、defer 清理
├── xcgen_struct.c        498 行   shape → struct 提升注册
├── xcgen_struct.h        111 行
├── xcgen_bridge.h         65 行   ❗JIT helper 跨界声明（应消亡）
├── xrt.h                  40 行   runtime 总入口
├── xrt_arc.h             203 行   ARC + bump allocator 适配（默认 off + 未接线）
├── xrt_arith.h           118 行
├── xrt_builtin_table.h   165 行
├── xrt_class.h           159 行   type table + vtable + instanceof（vtable / dtor 接线缺失）
├── xrt_coll.h            276 行   array / map / strbuf / closure（生命周期未纳入 ARC）
├── xrt_compat.h           58 行   源码级类型别名
├── xrt_exception.h        83 行
├── xrt_method.h          678 行   集合 / 字符串 builtin 实现
├── xrt_module.h           92 行   ❗模块描述表（init_fn 全 NULL，整层未真正使用）
└── xrt_value.h           137 行   tagged union + tag 常量
```

**当前没有 `.c > 3000` 越线**，但 `xcgen_call.c (1680)` / `xcgen.c (1574)` 已是“第二梯队大文件”，继续叠功能就会越线。`xrt_method.h (678)` 也接近 `.h ≤ 800` 的硬线。

### 2.2 期望的内部依赖方向

```text
xrt_value (L0)
  → xrt_compat / xrt_arith
    → xrt_arc (allocator + 对象头)
      → xrt_coll  (array/map/closure/strbuf)
      → xrt_class (type table + vtable + instanceof)
      → xrt_exception
        → xrt_method
        → xrt_module
          → xrt.h（runtime 入口）

src/aot/xcgen* (host-side codegen)
  → src/jit/xir.h（lowering 输入）
  → src/aot/xrt_*（生成代码所需的运行时声明）
```

补充约束：

- `xcgen_*` **不应**继续依赖 `xcgen_bridge.h` / 任何 `xr_jit_*` helper 指针匹配。
- AOT runtime header **不应**包含 `<malloc.h>` 直调；统一通过 `XRT_MALLOC` 适配层。
- `cmd_build_native()` **不应**反扫 bytecode 模式恢复元数据；元数据应来自前端/IR。

---

## 3. 已确认问题总览

下列每条都有源码证据，已收录于 `MEMORY[19b2b7f8]`，不再展开重复证明，只汇总到 P0/P1/P2。

### 3.1 P0：架构边界 / 正确性硬伤

1. **JIT helper 指针匹配作为生产路径**
   - `xcgen_call.c` 通过 `fn_ptr == (void *)xr_jit_xxx` 路由 CALL_C，未识别时**默认静默丢弃**。
   - 这已经导致过 `OP_SUBSTRING` / `xr_jit_str_repeat` / `xr_jit_chr` 静默消失（见 `MEMORY[edaf880a]`）。
   - 这是**隐式契约**，不能继续作为最终设计。

2. **`xcmd_build.c` 反扫 bytecode 模式恢复元数据**
   - `build_shared_proto_map()`：识别 `CLOSURE+SETSHARED` / `CLASS_FROM_DESC+[MOVE]+SETSHARED` 模式。
   - `collect_exports()`：反向追 `OP_MOVE` 链 + `GETSHARED` / `SETSHARED` 找 shared_idx。
   - `aot_preregister_classes()`：反扫 `OP_CLASS_CREATE_FROM_DESCRIPTOR` 配合 `OP_MOVE` 跟踪 register→class，并 patch `proto->param_types`。
   - 任意字节码格式或优化器变化都可能让 AOT 默默走错路径。

3. **内存模型分裂**
   - `xrt_arc_release_val()` 只处理 `XRT_TAG_PTR` / `XRT_TAG_STR_ARC`。
   - `xrt_coll.h` 中 array / map / closure / strbuf 直接走 `XRT_MALLOC/REALLOC`，**不进入 ARC**。
   - 同时 `xrt_type_register()` 在 codegen 中又传 `destructor = NULL`。
   - 当前对象生命周期事实上是“裸 malloc + 不释放”，文档承诺的 ARC 没真正落地。

4. **半接线的 runtime 抽象层**
   - `xrt_module.h` / 生成的 `xrt_modules[]`：`init_fn` 全是 `NULL`，`xrt_modules_init()` / `xrt_module_lookup()` 没有真实使用点。
   - `xrt_class.h` 的 `xrt_vcall()` / `xrt_instanceof()`：当前 codegen 没有发出对它们的调用。
   - `XrtContext` 全程传 `NULL`。
   - 这些“接口在但没接线”比直接缺失更危险——它会持续制造“看起来已完成”的错觉。

### 3.2 P1：runtime 完整性 / 可控性

5. **runtime 生命周期未真正建立**
   - `xrt_arc_init()` / `xrt_bump_destroy()` 没有任何调用点。
   - 生成的 `main()` 直接调 `mod_init(NULL)`，没有 allocator init / type table init / module init 序列。

6. **bump allocator 注释与代码不一致**
   - `xrt_arc.h` 的 `xrt_bump_enabled = 0`，但注释写 “default: on”。
   - 设计意图与实际行为矛盾，需要二选一。

7. **`XRT_USE_XR_MALLOC` 未在 CMake 启用**
   - `xrt_arc.h` 提供了 `XRT_MALLOC` 适配层；
   - 但 standalone 构建路径未定义 `XRT_USE_XR_MALLOC`，实际仍走 libc `malloc/free`。
   - 与项目“禁止直接 malloc/free”的红线表述不一致，应在生成代码侧明确策略。

8. **`--native` 被 `XRAY_HAS_JIT` 强行 gate**
   - `xcmd_build.c` 在不开 JIT 时直接报 `--native requires JIT support`。
   - AOT lowering 仍在借 JIT builder 活着；这不是终态。

9. **stale 注释与 CLI help 漂移**
   - `xcmd_build.c` 顶部仍写 native 是 “AOT + bytecode hybrid, links xray runtime”，实际已是 standalone C。
   - `xray build --native` help 文案（见 `MEMORY[9368bfa8]`）也对不上当前实现。

### 3.3 P2：缺失功能 / 文档一致性

10. **Phase C 完全未实施**
    - `src/coro/` 没有任何 `XRAY_AOT` / `cps_step` / `cps_frame` / `xr_coro_create_cps` 接线。
    - 没有 `XIR_SUSPEND` / await / channel / select 的 AOT codegen。
    - `tests/aot/` 没有 `concurrency/` 目录，文档承诺的 `go_basic.xr` / `channel.xr` / `select.xr` / `scope.xr` 不存在。

11. **Phase D 完全未实施**
    - 没有 `src/aot/xrt_stdlib_*.h`。
    - `xcgen.c` 硬编码 `comp->single_file = true`，`emit_debug` / `single_file` 字段事实上是死字段。
    - 没有 manifest / hash / cache 的增量编译机制。
    - 没有 ARC move / last-use 优化机制。

12. **enum 在 AOT 路径完全缺失**
    - AOT codegen / runtime / 测试都没有 enum 支持。
    - `tests/aot/` 下没有任何 enum 测试用例。

13. **集合方法实现路径偏离设计**
    - `map` / `filter` 在 AOT 下被 lower 成显式循环（`tests/aot/basic/collection_filter.xr` 验证），**不是**走 `xrt_method.h`。
    - 这点性能上更优，但与设计文档不符；应在文档侧承认这是更优实现，而不是在代码侧补一份多余的 method-table 调用。

14. **AOT 路径中分布的固定上限**
    - `xcmd_build.c`：`aot_cap = 64`、`synth[64]`、`chain[16]`、`reg_class[256]`。
    - `xrt_class.h`：`XRT_MAX_TYPES = 256`。
    - `xcgen.h`：异常 frame pending depth `8`。
    - 这些数字都没有显式校验失败的报错路径。

15. **AOT vtable / instanceof / dtor 集成测试缺失**
    - 即便 vtable runtime 后续接线，目前也没有针对“多态 vtable 派发”“instanceof 父链”“destructor 调用”的 AOT 端到端测试。

---

## 4. Phase 划分

每个 Phase 内部可以再切多个 PR；每个 Phase 自身满足“可独立合并、独立回滚”。

### Phase 0：先把已知不正确性收敛

> **目标**：去掉静默错误路径和事实性矛盾，**不引入新功能**。

- **0.1 让未识别的 `XIR_CALL_C` 直接编译失败**
  - `xcgen_call.c::emit_call_c` 默认 fallback 路径：当 `fn_ptr` 不在已识别集合内时，输出**显式 `#error` 或 abort 编译**，并在 stderr 报告函数名 / fn_ptr。
  - 配合现有 `xrt_invoke_method_sentinel` / `xr_jit_*` 已识别集合，建一份白名单常量数组。
  - 临时仍允许靠指针匹配，但**任何未匹配项都会失败而非静默放过**。

- **0.2 修正 `xrt_arc.h` 与 `xrt_coll.h` 的 release 不对称**
  - 选项 A（推荐）：在 `xrt_arc_release_val()` 中处理 `XRT_TAG_ARRAY / XRT_TAG_MAP / XRT_TAG_CLOSURE / XRT_TAG_STRBUF`，调用对应的 `xrt_array_free` / `xrt_map_free` / `xrt_closure_free` / `xrt_strbuf_free`。
  - 同步在 `xrt_coll.h` 增加 `*_free` 函数（即便 P1 才接 ARC，本 Phase 先把对称性补齐，避免长期裸漏）。
  - 选项 B：明确写一行注释“当前不释放容器对象”，并在 `xrt_arc_release_val` 处用 `XR_DCHECK` 显式断言不会传容器对象进来。
  - **Phase 0 内禁止**两种都不做。

- **0.3 修正 `xrt_arc.h` 注释与默认值**
  - 把“default: on”要么改注释为“default: off, opt-in via xrt_arc_init()”，要么默认开启。
  - 与 P1 的 runtime init 衔接。

- **0.4 修正 `xcmd_build.c` 顶部注释 / `xray build --native` help 文案**
  - 文档实事求是写：standalone C 输出 + 链接独立可执行 + 不依赖 `libxray_core`。
  - 移除“AOT + bytecode hybrid”等已过时描述。

- **0.5 把 stale 字段标死**
  - `xcgen.h` / `xcgen.c` 中 `single_file` / `emit_debug`：要么 P1 实施，要么先注释为 “reserved, currently no-op”。

**验收**：现有 24/24 AOT 测试 + 70/70 ctest 全绿，且 `xcgen_call.c` 在被人为去掉某个 helper 注册时**编译报错**，而不是默默出错。

---

### Phase 1：让 AOT runtime 真正“启动 / 关闭”

> **目标**：建立显式的 runtime 生命周期 + allocator 策略，把 `XrtContext` 从 `NULL` 占位变成真正的运行根。

- **1.1 设计 `XrtRuntime` / `XrtContext` 实体**
  - 新增 `src/aot/xrt_runtime.h` + `xrt_runtime.c`（按需）。
  - 字段最小集：
    - allocator 策略（malloc / bump / xr_malloc）
    - type_table 引用
    - module 表引用
    - 异常 frame stack 顶（替换当前线程局部变量）
    - 退出钩子
  - 提供 `xrt_runtime_init() / xrt_runtime_shutdown()`。

- **1.2 让生成的 `main()` 真正初始化 runtime**
  - `aot_write_main()`（在 `xcmd_build.c` 内）改为：
    1. `XrtRuntime rt; xrt_runtime_init(&rt, ...)`
    2. 按拓扑序 `mod_init(&rt)`
    3. 调用 entry
    4. `xrt_runtime_shutdown(&rt)`
  - 所有生成函数签名的第一参数 `xrt_ctx` 真正绑定到 `&rt`，不再传 `NULL`。

- **1.3 落地 allocator 策略**
  - 在 standalone 模式启用 `XRT_USE_XR_MALLOC` 或者明确改成 `xrt_alloc()` 抽象。
  - bump allocator 与 ARC 二选一作为默认；与 0.3 的注释保持一致。

- **1.4 删除或贯通 `xrt_module.h`**
  - 选项 A（推荐）：彻底**删除** `xrt_modules[]` / `xrt_modules_init()` / `xrt_module_lookup()`，因为当前所有 import 已经在 codegen 阶段 resolve 成 direct C call / `xrt_shared[]`，无需运行时模块表。
  - 选项 B：补完 init_fn / lookup 实路径。
  - 不允许保留“接口存在但永远空”的状态。

- **1.5 `--native` 与 JIT 解耦的可行性评估**
  - `cmd_build_native()` 仍然依赖 `xir_build_from_proto_aot_ex()`（来自 `src/jit/xir_builder.c`），这是 lowering 必需的，不算违规。
  - 但 `XRAY_HAS_JIT` 这一编译开关同时控制了“是否 ship JIT”和“是否 ship AOT lowering”。
  - **本 Phase 不强制拆分**，但要求：
    - 在 `docs/engineering/architecture_decisions.md` 里 **新增一条 ADR**：明确 “AOT lowering 当前依赖 XIR builder，作为编译期 host 工具被 JIT 模块导出，最终态考虑独立 lowering subsystem 名字”。
    - 不再把 “AOT 不依赖 JIT” 写进设计文档。

**验收**：
- 所有 AOT 测试可执行文件运行结束前调用 `xrt_runtime_shutdown()`，并通过 ASAN/LSAN 不漏报关键容器对象（Phase 0.2 已经把容器纳入 release 路径）。
- `tests/unit/aot/` 新增至少 1 个 runtime lifecycle 单元测试（init→shutdown→不泄漏）。

---

### Phase 2：替换 AOT 元数据来源（去 bytecode 反扫）

> **目标**：把 `cmd_build_native()` 里所有“反扫 bytecode 模式”的位置，全部改为读取**前端 / IR 一侧的显式元数据**。

- **2.1 export / shared 元数据显式化**
  - 在 `XrProto` / bundle 中显式记录：
    - 每个 export 的 `(name, shared_index, is_const, defining_proto_or_synthetic)`
  - 编译期 `OP_EXPORT` lowering 时直接写入这张表。
  - 删除 `collect_exports()` 中的“反扫 OP_MOVE 链 + GETSHARED/SETSHARED 配对”逻辑。

- **2.2 shared → child proto 映射显式化**
  - 在前端构建 closure / class 时，直接记录 `shared_index → child_proto*` + `is_ctor`。
  - 删除 `build_shared_proto_map()` 的两种字节码 pattern 识别。

- **2.3 class 元数据显式化**
  - `aot_preregister_classes()` 改为：
    - 直接读 `proto->classes[]`（或 frontend 输出的 class descriptor 表）。
    - 不再扫 `OP_CLASS_CREATE_FROM_DESCRIPTOR` + `OP_MOVE`，不再追 register→class。
  - 同步把 `class_infos` 注入 `XcgenCompilation` 的 API 标注为“official metadata”，不再带 `register-tracking` 的 hack 注释。

- **2.4 builtin / intrinsic 显式化**
  - 引入 `enum XrtIntrinsic { XRT_INT_SUBSTRING, XRT_INT_STR_REPEAT, XRT_INT_CHR, ... }`。
  - XIR builder 把 `xr_jit_substring` 等替换为 `XIR_CALL_INTRINSIC(XRT_INT_SUBSTRING, ...)`。
  - `xcgen_call.c` 路由 `XIR_CALL_INTRINSIC`，不再 match raw fn_ptr。
  - 等价地把 `xcgen_bridge.h` 标记 deprecated → 在 Phase 2 完成时删掉。

- **2.5 移除固定上限**
  - `xcmd_build.c` 中的 `aot_cap = 64` / `synth[64]` / `chain[16]` 改成动态 grow。
  - 越界路径加 `XR_CHECK_BOUNDS` / 报错而非静默截断。

**验收**：
- `xcmd_build.c` 中**不再含有**反扫 `OP_MOVE` / `OP_CLOSURE` / `OP_SETSHARED` / `OP_CLASS_CREATE_FROM_DESCRIPTOR` 的代码段（grep 验收）。
- `xcgen_bridge.h` 文件被删除。
- `xcgen_call.c` 的 `emit_call_c` fallback 不再依赖 helper 指针，而是 dispatch on `XirIntrinsicId`。
- 所有 AOT + JIT 测试 + ctest 全绿。

---

### Phase 3：补完 class runtime + 内存模型

> **目标**：让 vtable / instanceof / destructor / 容器对象生命周期成为**真实路径**，而不是仅有壳。

- **3.1 vtable 真正接线**
  - `xrt_type_register(...)` 调用点（`xcgen_call.c` 中）传入：
    - `vtable[]`：从前端 class descriptor 收集 method slot
    - `vtable_size`
    - `destructor`：默认 `xrt_default_object_dtor`，或 user-provided
  - 在 codegen 里：当方法是 polymorphic 时使用 `xrt_vcall(obj, slot)`，已 devirtualize 的仍走直接 C 调用。
  - 新增 `tests/aot/basic/class_vtable_polymorphic.xr` 端到端测试。

- **3.2 `instanceof` 真正接线**
  - 在 codegen 里把 `OP_TYPEOF` / `is X` 等映射到 `xrt_instanceof`。
  - 新增 `tests/aot/basic/class_instanceof.xr`。

- **3.3 destructor 真正接线**
  - ARC release 到 0 时调用 `xrt_type_table[type].destructor`。
  - 新增析构函数被调用次数的端到端测试（VS 输出 / sentinel 计数）。

- **3.4 容器对象的真实生命周期**
  - 与 Phase 0.2 衔接：
    - `xrt_array_free` / `xrt_map_free` / `xrt_strbuf_free` / `xrt_closure_free` 实现
    - 容器分配时进入 ARC（即对象头 + retain/release 一致）
  - 决策：`xrt_array_t` / `xrt_map_t` / `xrt_strbuf_t` / `xrt_closure_t` 是否带 `XrtArcHdr`：
    - 若带：与 class 对象统一；codegen 与 release 都简化
    - 若不带：每种类型显式 free 路径，且必须在 release_val 显式分发
  - 决定后**不留两套并行路径**。

**验收**：
- `tests/aot/basic/class_vtable_polymorphic.xr` / `class_instanceof.xr` / 析构计数测试全部通过。
- ASAN/LSAN 在所有 AOT 测试上无内存泄漏。
- `xrt_class.h` 注释与实际行为一致：所有声明的对外接口都至少在 codegen 中有调用点。

---

### Phase 4：Phase B 的最后两个空白点

> **目标**：把 enum 与“集合方法实现路径”这两个偏离设计的点收口。

- **4.1 enum AOT 支持**
  - 决策一：把 `enum X { ... }` 在 AOT 下编译为 typed `int64_t` + 名字表（最小表达力）。
  - 决策二：编译为带 tag 的 `XrtValue`，附额外 metadata。
  - 推荐：先做决策一覆盖 90% 用例。
  - 新增 `tests/aot/basic/enum_basic.xr`。

- **4.2 集合方法路径决议**
  - 当前 map / filter 已经被 lower 成显式循环（更优）。
  - 在设计文档侧把这点**正式承认**为最佳方案，并在 `aot_refactor_plan.md` / `aot-implementation.md` 之间保持一致。
  - 移除 / 收缩 `xrt_method.h` 中已不会被调用的 helper（确认 dead）。

**验收**：
- enum 在 AOT 下与 VM 运行结果一致（VM-AOT diff 为零）。
- `xrt_method.h` 文件规模下降；不再保留任何已确认 dead 的入口。

---

### Phase 5：Phase C / D 的“砍 vs 做”决策点

> **目标**：必须在文档与代码之间二选一，不能继续保持 “代码空 + 文档当作完成” 的错觉。

- **5.1 对 Phase C（CPS 并发）正式做决策**
  - 选项 A（推荐先做）：在 `aot-implementation.md` 中标记 “Phase C 当前未开始”，并把对应章节状态从 “planned” 改为 “not started”。
  - 选项 B：按设计真做 CPS lowering + `src/coro/` 的 `XRAY_AOT` 接线。
  - 不允许介于两者之间。

- **5.2 对 Phase D（stdlib bridge / 多文件输出 / 增量编译）正式做决策**
  - 同 5.1。

- **5.3 多文件输出收口**
  - `comp->single_file = true` 硬编码：
    - 若 Phase D 不做：删除 `single_file` / `emit_debug` 字段。
    - 若 Phase D 要做：补 per-module .c 输出 + Makefile/CMake snippet 生成。
  - 不允许继续以 “后续优化” 为名留死字段。

**验收**：
- `docs/design/aot-implementation.md` 与代码事实一致。
- `xcgen.h` 中无 dead field。

---

## 5. 跨 Phase 的工作清单（并发可做）

下列项与具体 Phase 不强相关，可独立 PR：

- **C-1**：把 `xcgen_call.c (1680)` 拆成至少 2 个文件（路由 / intrinsics 实现）。
- **C-2**：把 `xrt_method.h (678)` 按字符串 / 数组 / Map 拆成 3 个 header（`xrt_method_str.h` / `xrt_method_arr.h` / `xrt_method_map.h`）。
- **C-3**：补 `tests/unit/aot/`：runtime lifecycle、intrinsic dispatch、class vtable、ARC release 单元测试，作为非 e2e 的快速回归底盘。
- **C-4**：在 `MEMORY[edaf880a]` 类的修复都纳入 `bug_patterns.md`：
  - “AOT 静默丢弃 helper”→ 列为一条 anti-pattern。
  - “AOT 反扫 bytecode” → 列为一条 anti-pattern。

---

## 6. 不做项（明确避免 scope creep）

下列内容**本计划不处理**，避免与既有重构计划冲突：

- JIT 后端 parity / x64 完整化：见 `jit_x64_parity_plan.md` / `jit_stabilization_plan.md`。
- 协程调度器本身的稳定性：见 `coro_audit_plan.md` / `coro_refactor_plan.md`。
- DAP / LSP / CLI / pkg 相关重构：见对应 plan。
- 前端 lexer / parser / analyzer / codegen 重构：见 `frontend_refactor_plan.md`。
- 语言层面的新特性。

本计划仅负责把 **AOT 已有承诺**收敛到真实的、可维护的、与文档一致的状态。

---

## 7. 验收命令模板

每个 Phase 合并前必须至少跑：

```bash
# 完整 ctest（含 AOT 用例）
( cd build && ctest --output-on-failure )

# 完整回归
scripts/run_regression_tests.sh

# AOT 专项
bash tests/aot/run_aot_tests.sh
```

Phase 涉及生命周期 / 容器释放时，额外：

```bash
# ASAN 构建并跑全部 AOT 测试
/build-asan
bash tests/aot/run_aot_tests.sh
```

Phase 涉及 element 数量限制移除时，额外：

```bash
# 跑大模块用例（>64 函数 / >256 类等）专项 stress 测试
# 由 Phase 2.5 引入 tests/aot/stress/*.xr
```

---

## 8. 风险与回滚

- **Phase 0 → Phase 2**：变更主要在 `xcgen_call.c` / `xcmd_build.c` / `xrt_arc.h` / `xrt_module.h`，影响面集中，回滚以单 PR 粒度。
- **Phase 3**：vtable / 容器 ARC 是真正动结构的部分，**必须在 ASAN 下跑过整套 AOT 测试**才算合并完成。
- **Phase 5**：决策点；推荐先合“先标记 not started”，避免长期文档/代码漂移。

---

## 9. 与设计文档的同步责任

每个 Phase 合并 PR 时同步更新：

- `docs/design/aot-implementation.md`：状态字段、未完成列表。
- `docs/engineering/audit_baseline.md`：刷新 AOT 行项。
- 关键决策（如内存模型、模块表存废、intrinsic 表）：写入 `docs/engineering/architecture_decisions.md`。

不允许出现 “代码已经改了，但 `aot-implementation.md` 还停留在旧状态” 的情况。

---

## 10. 优先级一句话总结

> **先把 AOT 的“看起来完成”的假完工感清掉，再考虑 Phase C / D。**
> P0 是删半成品 + 去隐式契约；P1 是真正建立 runtime 生命周期；P2 是把 vtable / 容器 / 内存模型补成一条线；P3 是 enum 收尾；P4 是对 Phase C / D 做诚实决策。
