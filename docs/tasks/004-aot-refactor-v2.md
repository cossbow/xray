# AOT 重构方案 v2（彻底重写版）

> 日期：2026-04-26
> 状态：active
> 取代：`004-aot-refactor.md`（v1，保留为对照）
> 配套：`005-aot-implementation.md`（实施细节）、`docs/design/aot-design.md`（终态设计）

---

## 0. 为什么要 v2

v1 的诊断（P0/P1/P2 列表）扎实，但解决方案存在四处系统性偏差：

1. **多次"选项 A vs B"** — 0.2 / 1.4 / 3.4 / 5.1 / 5.2，与 "⚡ 直选最佳设计" 原则冲突。
2. **同一根因被切成多 Phase** — fn_ptr 匹配（0.1 + 2.4）、容器生命周期（0.2 + 3.4）、死字段（0.5 + 5.3），鼓励"假完工感"。
3. **顺序错误** — Phase 5（Phase C/D 决策）应该最先做，因为它影响 Phase 1 的 `XrtRuntime` 字段集；现在被放在最后。
4. **设计层缺口** — 没有 `XrtContext` 字段决策、没有 AOT 与 `src/runtime/gc/` 关系、没有前端配合策略、没有 ABI 决策、ASAN 验收条件实际无效。

v2 的差异（一句话）：**先答 5 个前置决策（Phase −1），再用 4 个 Phase 一次性收敛**，全文无"选项 A/B"，每个根因只在一处处理。

---

## 1. 当前现状（精简版，保留 v1 第 3 节的诊断）

### 1.1 仍存在的红线

| # | 红线 | 证据 |
|---|---|---|
| R1 | `emit_call_c` 默认静默丢未识别 helper | `src/aot/xcgen_call.c:1515-1517` |
| R2 | `xcmd_build.c` 反扫 bytecode 模式恢复元数据 | `build_shared_proto_map` / `collect_exports` / `aot_preregister_classes` |
| R3 | 容器对象事实上"alloc 后永不释放" | `xrt_coll.h` 全部裸 `XRT_MALLOC`，无 `*_free` |
| R4 | standalone 仍走 libc `malloc/free` | CMake 无 `-DXRT_USE_XR_MALLOC` |
| R5 | 生成的 `main()` 给 mod_init 传 `NULL` | `xcmd_build.c::aot_write_main` |
| R6 | `xrt_modules[]` 表生成 `init_fn = NULL`，永远不被调用 | `src/aot/xcgen.c:1546-1571` |
| R7 | `xrt_class.h::xrt_vcall` / `xrt_instanceof` 定义存在但 codegen 零调用 | `src/aot/xrt_class.h` |
| R8 | `xir_intrinsic.h` 全套 ID 定义但 `XIR_CALL_INTRINSIC` opcode 未加入，零 builder 引用 | `src/jit/xir_intrinsic.h` 是孤岛 |
| R9 | `xrt_runtime.h` 已新增但 `aot_write_main` 不调用 init/shutdown | `src/aot/xrt_runtime.h` 是孤岛 |
| R10 | `single_file = true` 硬编码、`emit_debug` 死字段 | `src/aot/xcgen.c:154` |
| R11 | 反扫上限 `synth[64]` / `chain[16]` / `reg_class[256]` 无失败路径 | `xcmd_build.c` |
| R12 | CLI 顶部注释仍写 "AOT + bytecode hybrid, links xray runtime (like Go)" | `xcmd_build.c:13` |
| R13 | bump allocator 块注释 "default: on"，实际默认 0 | `xrt_arc.h:73-77` |

### 1.2 半接线 header（v1 中未明确列出）

R6 / R7 / R8 / R9 共同构成"接口齐全 / 零调用方"的 4 个空骨架。**v2 的 Phase 0 第一动作就是处置它们**：要么真接进去（基于 Phase −1 决策），要么删。

---

## 2. Phase −1：前置决策（必须先答）

下列 5 个决策**必须先于 Phase 0 答完**，因为它们决定 Phase 1 的字段集 / Phase 3 的 dtor 形态 / Phase 5.3 输出形态。

每条给出 **RECOMMENDED**（推荐答案 + 理由）+ **影响**（决策对后续 Phase 的影响）。

### D-1：AOT 是否支持并发（goroutine / channel / select / scope）？

- **RECOMMENDED：做**
- **理由**：Xray 核心卖点之一是 native concurrency；AOT 砍掉并发等于砍一半语言能力。设计文档（`aot-design.md`）已选 "M:N thread pool"。
- **影响**：
  - `XrtContext` 必须预留 `coro_root` 字段（Phase 1.1）。
  - XIR 需要 `XIR_SUSPEND` / `XIR_RESUME` opcode 在 AOT 路径生效（独立 PR，不在本 plan 内）。
  - `tests/aot/concurrency/` 必须建（go_basic.xr / channel.xr / select.xr / scope.xr）。
- **如果选不做**：`XrtContext` 简化、模块表彻底删；但与设计文档矛盾，不推荐。

### D-2：是否做 Phase D（stdlib bridge / 多文件输出 / 增量编译）？

- **RECOMMENDED**：
  - **多文件输出：做**（per-module .c，Nim/V 标准做法；便于并行编译 + 调试 + 后续增量基础）
  - **stdlib bridge：缓做**（等 stdlib 自身稳定后再补）
  - **增量编译：不做**（Xray 项目本身规模小，可观测收益低；做了反而引入复杂度）
- **影响**：
  - `single_file = true` 字段直接删（不再硬编码，Phase 5.3 落地）
  - CMake 生成 per-module rules
  - manifest / hash / cache 机制不引入

### D-3：AOT 输出形态？

- **RECOMMENDED**：默认 **per-module `.c` + 自带 `main.c` → `cc` 链接 standalone binary**。可选 `--single-c` 标志合并为单文件（调试 / inline 优化用途）。
- **影响**：
  - `emit_debug` 不再是死字段，默认开启 `#line` 指令
  - `xcgen.c` 的 `comp->single_file` 改为 CLI 标志驱动
  - 无需考虑动态共享库（`.so`/`.dylib`）作为输出形态——那是 stdlib bridge 的事

### D-4：AOT 与 `src/runtime/gc/` 的关系？

- **RECOMMENDED**：**AOT 路径完全脱钩 `src/runtime/gc/`，使用 ARC（默认 calloc）+ bump（opt-in）**。
- **理由**：
  - AOT 场景对象图相对静态，引用关系可在 codegen 阶段插入 retain/release，无需 mark-sweep。
  - GC 的 stop-the-world 与"独立二进制"诉求冲突。
  - VM 路径继续用 `src/runtime/gc/`，AOT 路径专用 ARC，物理上不交叠。
- **影响**：
  - `src/aot/xrt_arc.h` 是 AOT 唯一的内存策略
  - `src/runtime/gc/` 头文件不进 `xcgen_*` 的 include 链
  - 容器对象（Phase 3.1）一律走 `xrt_arc_alloc`（带 `XrtArcHdr`）

### D-5：`XrtContext` 字段集？

基于 D-1（要并发）+ D-4（专用 ARC）推导：

```c
typedef struct XrtRuntime {
    int            initialized;       // 0=pristine, 1=init done
    XrtArenaState  arena;             // bump cursor / blocks; 当 bump_enabled=0 时 unused
    void          *coro_root;         // D-1=做：coroutine scheduler root；D-1=不做：删此字段
    XrtExcFrame   *exc_frame_top;     // 替代 thread-local，多 isolate 可独立
    XrtExitHook   *exit_hooks;        // deferred cleanup chain (atexit-style，但绑定到 rt 生命周期)
} XrtRuntime;
```

**显式不放的字段**（说明理由）：

| 不放的字段 | 理由 |
|---|---|
| `type_table` | codegen 阶段已 resolve；运行时仅需要 `_tid_<ClassName>` 静态变量（每个 class 一份），不需要全局表 |
| `module` 表 | D-2 选了多文件输出 + codegen 已解析所有 import；运行时无需查表 |
| `allocator policy` | 编译期通过 `-DXRT_USE_BUMP` / `-DXRT_USE_XR_MALLOC` 决定，不在运行时切换 |
| `error handler` | 用 `XR_CHECK` / `abort` 即可，不引入回调机制 |

---

## 3. Phase 0：去半成品 + 显式契约（一次性收敛）

> **目标**：在不引入新功能的前提下，**所有半接线、隐式契约、stale 文案一次清掉**。Phase 0 完成后，仓库内不存在"接口齐全 / 零调用方"的 header。

### 3.1 引入 `XIR_CALL_INTRINSIC` 取代全部 fn_ptr 匹配（合并 v1 的 0.1 + 2.4）

**为什么合并**：v1 把它切成两步——0.1 改"未识别 abort"，2.4 才真正引入 INTRINSIC——这意味着 0.1 完成后到 2.4 之前，fn_ptr 匹配仍是生产路径。v2 一步到位。

**做法**：

1. `src/jit/xir_ops.h` 加入 `XIR_CALL_INTRINSIC` opcode（`xir_intrinsic.h` 的 ID 终于真实使用）
2. `src/jit/xir_builder*.c` 所有当前发出 `XIR_CALL_C(fn=xr_jit_*, ...)` 的站点：
   - JIT 模式：保留 `XIR_CALL_C`（运行期 fn_ptr 仍是必要的）
   - AOT 模式（`b->aot_mode == true`）：改发 `XIR_CALL_INTRINSIC(id=XR_INTRIN_*, encoded)`
3. `src/aot/xcgen_call.c` 新增 `emit_call_intrinsic()`，按 ID 路由（switch on `XirIntrinsicId`），**不再依赖 fn_ptr 比较**
4. 删除 `src/aot/xcgen_bridge.h`（v1 的 2.4 验收条件之一）
5. `emit_call_c` 默认 fallback：`abort()` + 输出 fn_ptr 名（用 `dladdr`）—— 任何遗留的 `XIR_CALL_C` 进 AOT 都立刻报错而非静默
6. `xcgen_call.c` / `xcgen.c` 不再 include 任何 `xr_jit_*` 头

**验收**：

```bash
# 必须为空
grep -rn "fn_ptr ==" src/aot/
grep -rn "xcgen_bridge" src/aot/
grep -rn "xr_jit_" src/aot/    # 除 emit_call_c 错误打印外，不应有任何引用
```

### 3.2 删 4 个空骨架 header

**v1 的 1.4 选 A/B；v2 一律选 A：删**。

| 文件/段落 | 处置 |
|---|---|
| `src/aot/xrt_module.h` | **整体删除**。codegen 已 resolve 所有 import（D-2），运行时不需要 module 表 |
| `src/aot/xrt_runtime.h::XrtRuntime` | **保留但扩字段**（按 D-5 决策）；同时 `aot_write_main` 真接（见 Phase 1） |
| `src/aot/xrt_class.h::xrt_vcall` / `xrt_instanceof` | **保留，但 Phase 3 必须真接**；Phase 0 完成时给这两个函数加 `__attribute__((unused))` 警告抑制即可，不允许在 Phase 0 后状态保留 |
| `src/jit/xir_intrinsic.h` | **保留**，3.1 真接进去 |

**Phase 0 验收**：仓库不存在"声明在但 codegen 零调用"的 header（`grep` `_tid_` / `xrt_vcall` / `xrt_instanceof` / `XIR_CALL_INTRINSIC` 都至少有 1 个生成代码引用，或者那个文件被删）。

### 3.3 死字段直接删（合并 v1 的 0.5 + 5.3）

| 字段 | 处置 |
|---|---|
| `XcgenCompilation::single_file` | **直接删**。D-2 决定多文件输出，未来 `--single-c` 是 CLI 标志而非 struct 字段 |
| `XcgenCompilation::emit_debug` / `XcgenModule::emit_debug` | **直接删**。D-3 决定 `#line` 默认开（在 codegen 内常量化），未来如需关闭再走 CLI |
| `xcgen.c::aot_write_main` 中的 `mod_init(NULL)` | Phase 1 改 `mod_init(&rt)` |

### 3.4 CLI 文案 + bump 注释（合并 v1 的 0.3 + 0.4）

- `xcmd_build.c` 顶部注释改写为：standalone C 输出，不依赖 `libxray_core`
- `xray build --native` help 文案同步
- `xrt_arc.h` 块注释 "default: on" 改为 "default: off (calloc); enable via -DXRT_USE_BUMP"
- 与 D-4 决策一致

### 3.5 错误处理统一（v1 没提，v2 补）

当前 `src/aot/` 内五种错误处理形态共存：`XR_CHECK_BOUNDS` / `abort()` / `fprintf(stderr,..) abort()` / 静默丢弃 / 静默 truncate。

**v2 立全局策略**：

- 编译期错误（codegen 阶段）：`fprintf(stderr, ...) + abort`（一行宏 `XCG_FATAL(fmt, ...)`）
- 运行时错误（生成代码内）：`xrt_panic(msg)`（abort + 输出位置信息）
- 不允许任何"silent drop / silent truncate"

**Phase 0 验收**：
```bash
grep -rn "// suppress\|// silent" src/aot/   # 必须为空
```

---

## 4. Phase 1：runtime lifecycle 真接线

> **目标**：让 D-5 决定的 `XrtRuntime` 字段真正流入生成代码，`aot_write_main` 完整 init/shutdown。

### 4.1 `XrtRuntime` 扩字段

按 D-5 推导结果实施。`XrtArenaState` 把当前 file-local 的 `xrt_bump_cursor` / `xrt_bump_end` / `xrt_bump_blocks` 三个全局**移入 struct**——多 isolate 不再共享 bump 状态。

### 4.2 `aot_write_main` 真接

```c
// 生成代码模板
int main(int argc, char **argv) {
    XrtRuntime rt = {0};
    xrt_runtime_init(&rt);
    xr_module_a__module_init(&rt);
    xr_module_b__module_init(&rt);
    xr_main__module_init(&rt);   // 入口
    int rc = xrt_runtime_shutdown(&rt);   // 返回 0 / 退出码
    return rc;
}
```

每个生成函数签名第一参数 `XrtContext xrt_ctx`（即 `XrtRuntime *`）真正绑定 `&rt`，全仓**不再传 `NULL`**。

### 4.3 allocator 策略一次定（合并 v1 的 1.3）

CMake 决策表：

| 编译开关 | XRT_MALLOC 路由 | 用途 |
|---|---|---|
| 无 | `malloc/free` | 默认 standalone（用户独立编译生成代码） |
| `-DXRT_USE_XR_MALLOC` | `xr_malloc/xr_free` | 在 xray 主项目内编译 AOT 测试时启用，遵守"禁止直接 malloc/free"红线 |
| `-DXRT_USE_BUMP=1` | bump（仍走 `XRT_MALLOC` 申请大块） | 性能场景 opt-in，单进程 / 短命任务 |

主仓 CMake 同时启用 `XRT_USE_XR_MALLOC`，确保 AOT 测试在主项目内编译时不违反红线。

### 4.4 单元测试

`tests/unit/aot/test_runtime_lifecycle.c`（新增，v1 的 cross-phase C-3 落地点之一）：

- init → shutdown → 不泄漏
- init → 模拟 mod_init 错误 → shutdown 仍正确清理
- 多次 init 幂等

### 4.5 Phase 1 验收

- 全部 AOT 测试运行结束前调用 `xrt_runtime_shutdown`
- ASAN 在 Phase 3 前**预期会报漏**（容器对象未释放），这是正常的；Phase 3 完成后必须无漏报
- `tests/unit/aot/test_runtime_lifecycle.c` 通过

---

## 5. Phase 2：去 bytecode 反扫（**双边 PR**）

> **目标**：所有 AOT 元数据来源切换为前端显式字段；删除 cli 端反扫逻辑。

### 5.1 v1 的盲点：这是双边 PR

v1 把 Phase 2 写成 cli 单边任务，但**反扫逻辑的替代方案在前端**。v2 显式声明：**Phase 2 = 前端补字段 + cli 切读字段，单 PR 同时完成**。不允许"前端字段加了 cli 没切"或反之的中间态。

### 5.2 前端补字段（`src/frontend/`）

`XrProto` 新增三组字段：

```c
struct XrProto {
    ... 原有字段 ...

    // Shared variable → defining proto map
    // Filled at OP_CLOSURE / OP_SETSHARED / OP_CLASS_FROM_DESC lowering time
    XrSharedProtoEntry *shared_protos;   // [shared_index] = { proto, is_ctor }
    int                 shared_protos_count;

    // Module-level exports
    // Filled at OP_EXPORT lowering time
    XrExportEntry      *exports;          // [i] = { name, shared_index, is_const }
    int                 exports_count;

    // Class descriptors (for AOT type table codegen)
    // Filled at OP_CLASS_CREATE_FROM_DESCRIPTOR lowering time
    XrAotClassEntry    *classes;          // [i] = { class_name, parent_name, ctor_proto, instance_field_count }
    int                 classes_count;
};
```

**关键约束**：这三组字段在 `bytecode` lowering **当时**填，不是事后扫；任何 codegen 优化对 bytecode 的改写都自动反映在字段里。

### 5.3 cli 切读字段

删除：

- `xcmd_build.c::build_shared_proto_map()` 全函数
- `xcmd_build.c::collect_exports()` 中所有反扫 OP_MOVE/SETSHARED/GETSHARED 的逻辑（保留 synthetic shared 分配的部分，但改读 `proto->exports` + 必要的 frontend 配合）
- `xcmd_build.c::aot_preregister_classes()` 中 `reg_class[256]` + bytecode 扫描，改为遍历 `proto->classes`

### 5.4 移除固定上限（v1 的 2.5）

`reg_class[256]` 是 5.3 完成后的副产物——不再反扫就不需要这个数组。

`synth[64]` / `chain[16]` 同理。

`aot_cap = 64` 改为 `xr_array` 动态 grow（`src/base/xarray.h`）。

`XRT_MAX_TYPES = 256`：与 D-4 决策（一律 ARC）配合，改为运行期动态 grow（每模块一份 `_tid_*` 静态变量，不再全局表上限）。

### 5.5 Phase 2 验收

```bash
# 必须为空
grep -n "OP_MOVE\|OP_SETSHARED\|OP_GETSHARED\|OP_CLASS_CREATE_FROM_DESCRIPTOR" src/app/cli/xcmd_build.c

# 必须存在
grep -n "proto->shared_protos\|proto->exports\|proto->classes" src/app/cli/xcmd_build.c
```

---

## 6. Phase 3：内存模型 + class runtime 一线收口

> **目标**：容器对象 / class 对象的 lifecycle 在**单一模型**下统一处理。一次性接 vtable / instanceof / dtor / 容器 ARC，不留 v1 的"两套并行路径"风险。

### 6.1 容器对象一律带 `XrtArcHdr`（v1 的 0.2 + 3.4 合并）

v1 在 0.2 列了选项 A/B，在 3.4 又列了选项 A/B。v2 直接定：**容器一律带 ArcHdr**。

```c
// xrt_coll.h
typedef struct {
    int64_t  len;
    int64_t  cap;
    XrtValue *data;
} xrt_array_t;

// 分配路径变更
static inline XrtValue xrt_array_new(int64_t cap) {
    if (cap < 4) cap = 4;
    xrt_array_t *a = (xrt_array_t *)xrt_arc_alloc(sizeof(xrt_array_t));
    a->len = 0;
    a->cap = cap;
    a->data = (XrtValue *)XRT_CALLOC((size_t)cap, sizeof(XrtValue));
    XR_ARC_HDR(a)->type = XRT_TYPE_ARRAY;
    XR_ARC_HDR(a)->flags |= XRT_ARC_HAS_DEINIT;
    return xrt_mkptr(a, XRT_TAG_ARRAY);
}
```

`xrt_array_t::data`、`xrt_strbuf_t::buf`、`xrt_map_entry_t[]` 等**子缓冲**仍走 `XRT_REALLOC`（不需 ArcHdr，因为它们的生命周期与父对象绑定）；deinit 函数（`xrt_arc_deinit`）针对 `XRT_TYPE_ARRAY` 等做内部 free。

`xrt_arc_release_val` 不再需要 case-by-case 处理 array/map/strbuf/closure：所有这些都走统一 ArcHdr 路径。

### 6.2 vtable 真接线（v1 的 3.1）

`aot_preregister_classes`（Phase 2 改后）从 frontend `proto->classes` 收集每个 class 的 instance method slot 顺序（vtable 顺序），传给 `xrt_type_register`：

```c
// 生成代码
{
    static XrtMethodFn _vt_Animal[] = { (XrtMethodFn)Animal__sound, (XrtMethodFn)Animal__name };
    if (!_tid_Animal) {
        _tid_Animal = xrt_type_register("Animal", 0, _vt_Animal, 2, xrt_default_dtor, sizeof(Animal_data));
    }
}
```

多态调用点（codegen 检测到 `is_polymorphic = true`）：

```c
// 替代原来的 direct call
((RetType (*)(XrtContext, XrValue, ArgTypes...))xrt_vcall(obj.ptr, /*slot*/0))(ctx, obj, ...);
```

### 6.3 instanceof 真接线（v1 的 3.2）

`OP_TYPEOF` / `is X` / `match X is Y` codegen 路径输出 `xrt_instanceof(obj.ptr, _tid_X)`。

### 6.4 destructor 真接线（v1 的 3.3）

`xrt_type_register` 默认 dtor = `xrt_default_dtor`（释放容器子缓冲 + 递归 release 字段）；用户 `~ClassName` 编译为重写 dtor。

### 6.5 测试（合并 v1 的多处验收要求）

- `tests/aot/basic/class_vtable_polymorphic.xr`
- `tests/aot/basic/class_instanceof.xr`
- `tests/aot/basic/dtor_count.xr`（用全局 sentinel 计数验证 dtor 触发次数）
- `tests/aot/basic/container_lifecycle.xr`（创建大量 array/map/closure，确认全部释放）

### 6.6 Phase 3 验收（**这是真验收，不是空验收**）

```bash
# ASAN/LSAN 在所有 AOT 测试无内存泄漏（Phase 3 关键门槛）
/build-asan
bash tests/aot/run_aot_tests.sh
# 必须没有 LeakSanitizer 报告
```

如果验收时仍有泄漏，说明 6.1 容器路径或 6.4 dtor 路径有遗漏，**不允许"标 reserved 后续修"**。

---

## 7. Phase 4：enum + dead code 收口

### 7.1 enum AOT（v1 的 4.1）

**RECOMMENDED**：编译为 `int64_t` + 静态名字表（最小表达力，覆盖 95% 用例）。

```c
// 生成代码
typedef int64_t Color;
#define Color_Red    INT64_C(0)
#define Color_Green  INT64_C(1)
#define Color_Blue   INT64_C(2)
static const char *const Color_names[] = { "Red", "Green", "Blue" };
```

`tests/aot/basic/enum_basic.xr`：包含 enum 声明、模式匹配、转字符串、跨函数传递。

### 7.2 集合方法路径（v1 的 4.2）

设计文档侧承认"map / filter 等高阶方法 lower 成显式循环是最佳实现"。

**xrt_method.h 拆分**（v1 的 cross-phase C-2 落地）：

| 新文件 | 内容 | 行数上限 |
|---|---|---|
| `xrt_method_str.h` | 字符串方法（contains / startswith / split / replace 等） | ≤ 250 |
| `xrt_method_arr.h` | 数组方法（length / push / pop / sort / join 等） | ≤ 250 |
| `xrt_method_map.h` | Map 方法（get / set / has / keys / values 等） | ≤ 200 |

合并 include 入口 `xrt_method.h` 仍保留为聚合，仅 `#include` 三个子 header。

任何**已确认 dead** 的入口（map/filter/reduce 等已被 inline）直接删除，不留"可能将来用得上"的存量。

### 7.3 Phase 4 验收

- `tests/aot/basic/enum_basic.xr` 通过，VM-AOT diff 为零
- `xrt_method_*.h` 各文件均 ≤ 上限
- `xrt_method.h` 主聚合文件 ≤ 50 行

---

## 8. 跨 Phase 工作（与具体 Phase 解耦，可独立 PR）

| ID | 工作 | 落地 Phase 关联 |
|---|---|---|
| C-1 | `xcgen_call.c` 拆分为 `xcgen_call.c`（路由 + CALL_KNOWN/SELF）+ `xcgen_call_intrinsic.c`（intrinsic 实现） | Phase 0 完成后立刻做，避免单文件继续膨胀 |
| C-2 | `xrt_method.h` 拆分（见 7.2） | Phase 4 内做 |
| C-3 | `tests/unit/aot/` 子目录新增：`test_runtime_lifecycle.c` / `test_intrinsic_dispatch.c` / `test_vtable.c` / `test_arc.c` | Phase 1 / 3 / 4 各贡献一份 |
| C-4 | `bug_patterns.md` 新增两条 anti-pattern | Phase 0 / 2 落地后立即记录 |
| C-5 | 错误处理统一宏 `XCG_FATAL` / `xrt_panic` | Phase 0.5 |

---

## 9. 不做项（明确避免 scope creep）

继承 v1 第 6 节，无变化：

- JIT 后端 parity（见 `006-jit-stabilization.md` / `009-jit-x64-parity.md`）
- 协程调度器本身（见 `017-coro-audit.md`）
- DAP / LSP / CLI / pkg（见对应 plan）
- 前端 lexer / parser / analyzer 重构（见 `012-frontend-refactor.md`）
- 语言层新特性

**新增不做项**：
- 增量编译（D-2 决策不做）
- stdlib bridge（D-2 决策缓做）
- AOT `.so` / `.dylib` 输出（D-3 决策不做）

---

## 10. 验收命令模板

每个 Phase 合并前必须跑：

```bash
# 完整 ctest（含 AOT）
( cd build && ctest --output-on-failure )

# 完整回归
scripts/run_regression_tests.sh

# AOT 专项
bash tests/aot/run_aot_tests.sh
```

**Phase 3 起额外**（容器 ARC + dtor 接线后必须）：

```bash
/build-asan
bash tests/aot/run_aot_tests.sh
# 期望：0 leaks reported by LeakSanitizer
```

**Phase 2.5（移除上限）起额外**：

```bash
# stress 用例：>64 函数 / >256 类的模块
bash tests/aot/stress/run_stress.sh
```

---

## 11. 文档同步责任

每个 Phase 合并 PR 时必须同步更新：

| 文档 | 同步内容 |
|---|---|
| `docs/tasks/005-aot-implementation.md` | "已完成" 表 + "关键差距" 表 |
| `docs/engineering/audit_baseline.md` | AOT 行项 |
| `docs/engineering/architecture_decisions.md` | D-1 ~ D-5 决策记录为 ADR-012 ~ ADR-016 |
| `docs/design/aot-design.md` | 若 Phase −1 决策修正了某条设计选型，同步 |

**v1 的死引用修正**：v1 反复引用 `docs/design/aot-implementation.md`（已 archive）。v2 的同步责任目标**仅指向 `005-aot-implementation.md`** 和 `aot-design.md`，不再提 archive 文件。

---

## 12. 与 v1 的对应关系（迁移指引）

| v1 子项 | v2 对应 | 备注 |
|---|---|---|
| 0.1 未识别 CALL_C abort | 3.1（合并 2.4） | 一步到位引入 INTRINSIC，不再两步走 |
| 0.2 release / coll free 对称 | 6.1 | 容器一律 ArcHdr，无 A/B 选项 |
| 0.3 bump 注释 | 3.4 | 与 D-4 决策对齐 |
| 0.4 CLI 文案 | 3.4 | |
| 0.5 stale 字段标 reserved | 3.3（合并 5.3） | 直接删，不留 reserved |
| 1.1 XrtRuntime 字段集 | D-5 + 4.1 | 字段决策前置 |
| 1.2 main 真接 | 4.2 | |
| 1.3 allocator 策略 | 4.3 | |
| 1.4 xrt_module 删/贯通 | 3.2 | 一律删 |
| 1.5 JIT 解耦 ADR | ✅ ADR-011 已记录，v2 无需重做 | |
| 2.1 export 显式化 | 5.2 + 5.3 | |
| 2.2 shared map 显式化 | 5.2 + 5.3 | |
| 2.3 class 显式化 | 5.2 + 5.3 + 6.2 | 与 vtable 收集合流 |
| 2.4 CALL_INTRINSIC | 3.1 | 提前到 Phase 0 |
| 2.5 移除固定上限 | 5.4 | 反扫删后自然消失 |
| 3.1 vtable | 6.2 | |
| 3.2 instanceof | 6.3 | |
| 3.3 dtor | 6.4 | |
| 3.4 容器 ArcHdr | 6.1 | 与 0.2 合并 |
| 4.1 enum | 7.1 | |
| 4.2 集合方法 | 7.2 | |
| 5.1 Phase C 决策 | D-1 | 决策前置 |
| 5.2 Phase D 决策 | D-2 | 决策前置 |
| 5.3 多文件输出 | 3.3 + D-2/D-3 | 死字段直删 |
| C-1 拆 xcgen_call | 8 C-1 | |
| C-2 拆 xrt_method | 7.2 | |
| C-3 tests/unit/aot | 8 C-3 | |
| C-4 bug_patterns | 8 C-4 | |

---

## 13. 风险 / 回滚

| Phase | 主要风险 | 回滚粒度 |
|---|---|---|
| Phase 0 | 引入 INTRINSIC 可能漏处理某些 helper | 单 PR；可单独回滚某个 helper 的 INTRINSIC 改造，回到 fn_ptr 路径 |
| Phase 1 | runtime lifecycle 接错可能 crash 启动 | 单 PR；保留 NULL 路径作为 fallback 调试，CI 通过后才删 |
| Phase 2 | 双边 PR（前端 + cli），影响面较大 | **必须单 PR 合并**；不允许"前端先合 cli 后合"中间态 |
| Phase 3 | 容器 ArcHdr 改动 codegen 大面积；ASAN 验收硬门槛 | 单 PR；如果 ASAN 验收不过，整体 revert，不允许"标 reserved 后续修" |
| Phase 4 | enum / xrt_method 拆分；影响面小 | 单 PR |

---

## 14. 优先级一句话

> **Phase −1（决策） → Phase 0（去半成品 + INTRINSIC + 死字段一次清） → Phase 1（runtime 真接） → Phase 2（双边消反扫） → Phase 3（容器 ArcHdr + vtable 一线收） → Phase 4（enum + dead code）**

每一步只做一件事，每一步都不留妥协。

---

## 附录 A：v1 v2 关键差异速查

| 维度 | v1 | v2 |
|---|---|---|
| 选项 A/B 数 | 5 处 | 0 处 |
| 同根因切多 Phase | 3 组 | 0 组 |
| Phase −1（前置决策）| 无 | 5 项决策 + RECOMMENDED |
| ASAN 验收有效性 | 无效（容器永不 free 也通过） | Phase 3 起真实有效 |
| 双边 PR 显式声明 | 无（Phase 2 实际跨前端 + cli 但未声明）| Phase 2 显式声明 |
| 错误处理统一策略 | 无 | 3.5 |
| `XrtContext` 字段决策 | 模糊（只列"最小集"）| D-5 字段决策表 + 排除清单 |
| 与 `src/runtime/gc/` 关系 | 未提 | D-4 明确脱钩 |
| AOT 输出形态 | 默认 single_file 但未理性决策 | D-3 多文件 + 可选 single |
| 文档同步目标 | 引用 archive 文件 | 修正为 005 + design 文件 |

