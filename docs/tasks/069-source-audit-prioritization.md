# 069 — 源码审计与优先级重排

> 审计日期：2026-05-02
> 方法：仅基于当前源码事实，不依赖设计文档或历史记忆
> 版本：CMakeLists.txt `VERSION 0.6.0`

---

## 一、审计方法

逐条核对上一版开发建议中的关键判断，将每条映射到具体源码文件和行为，
确认"成立 / 过时 / 需修正"，然后基于纯源码事实重新排列优先级。

---

## 二、核对结论速查表

| # | 上一版判断 | 结论 | 关键源码证据 |
|---|-----------|------|-------------|
| 1 | 编译器已统一到 Xi IR pipeline | ✅ 成立 | `xr_compile()` 调用 `xi_pipeline_compile_program()`，无 legacy fallback |
| 2 | AOT 还靠旧 bytecode 扫描和 CLI 里的逻辑 | ❌ 过时 | `xaot_build()` 已独立于 CLI，走 Xi IR AOT pipeline |
| 3 | AOT metadata 已完全替代扫描 | ⚠️ 部分成立 | `XiModule` 已存在，但 `xi_module_populate_exports()` 仍 scan IR pattern |
| 4 | AOT cgen 有全局状态和固定上限 | ✅ 成立 | `static XiCgenCtx g_ctx`，`CG_MAX_SHARED=512` 等硬编码上限 |
| 5 | AOT 仍有静默占位输出 | ✅ 成立 | `xi_cgen.c` 中多处 `XR_NULL_VAL /* TODO */` |
| 6 | AOT 测试脚本仍有 retry | ✅ 成立 | `run_aot_tests.sh` 重试 3 次，失败标记为 SKIP |
| 7 | JIT 旧 builder 还在主线 | ❌ 过时 | 当前 JIT 走 `xi_to_xm_lower()`，不再走旧 `xir_builder` |
| 8 | JIT unsupported 会安全 fallback | ⚠️ 部分成立 | unknown op 设 `ctx->error`；但 `XI_DEFER` 等返回 dummy 值不报错 |
| 9 | 回归脚本仍有 `--no-jit` 特例 | ✅ 成立 | 3 个测试强制禁用 JIT |
| 10 | Json/Object 还没引入 `XR_KIND_OBJECT` | ❌ 过时 | `xtype.h` 已有 `XR_KIND_OBJECT` |
| 11 | Json/Object 语义仍需收敛 | ✅ 成立 | analyzer 多处合并 JSON/OBJECT 处理，mono name 统一返回 `"json"` |
| 12 | 网络 IO 都阻塞 worker | ⚠️ 过度简化 | 存在 yieldable path 和 sync-blocking path 两套机制并存 |
| 13 | stdlib 文件 IO 仍是同步阻塞 | ✅ 成立 | `stdlib/io/io.c` 头注释明确标注 synchronous + blocking |
| 14 | CLI 仍是旧路由和 getopt 问题 | ❌ 过时 | 已迁移到 spec-driven CLI |
| 15 | pkg install 还是 placeholder | ❌ 过时 | init/add/install/tree/login/publish 已实现；remove/update 未实现 |
| 16 | LSP/DAP 都只是外围空壳 | ⚠️ 需修正 | LSP 功能较完整；DAP 有核心能力，hot reload 未实现 |
| 17 | stdlib metadata 单一真相源不足 | ✅ 成立 | builtin 信息分散在 analyzer/module/LSP/AOT/stdlib 多处 |

---

## 三、逐条详细分析

### 3.1 编译器管线（成立）

`src/frontend/codegen/xcompiler.c` 中 `xr_compile()` 执行：
1. 类型推断、单态化、逃逸分析
2. `xi_pipeline_compile_program()` — 唯一编译路径

`src/ir/xi_pipeline.h` 支持三种模式：`XI_PIPE_VM` / `XI_PIPE_AOT` / `XI_PIPE_CHECK`。

**结论：Xi IR Typed SSA 是唯一前端到后端的 IR，这是当前最可靠的架构事实。**

### 3.2 AOT driver（上一版过时）

`src/aot/xaot_driver.c` 中 `xaot_build()` 已是独立 AOT 入口：
- bundle 发现模块 → `xi_compile_one()` 使用 `xi_pipeline_aot_config()`
- 构造 `XiModule` → `xi_module_populate_exports()`
- 跨模块导入解析 → C 代码生成

`src/app/cli/xcmd_build.c` 仅负责调用 `xaot_build()` 和后续 CC/link。

**结论：AOT driver 已独立化并走 Xi IR pipeline，不再是"CLI 里一坨逻辑"。**

### 3.3 AOT metadata（部分成立）

`src/ir/xi.h` 已有 `XiModule` / `XiModuleExport` / `XiModuleImport`。

但 `xi_module_populate_exports()` 仍通过扫描 `XI_SET_SHARED(slot, CLOSURE_NEW/CLASS_CREATE)` 
pattern 提取 export 信息；`xi_cgen.c` 中 `cg_prescan_shared()` 也有辅助扫描。

**结论：metadata 层已有，但部分由 IR pattern scanning 生成。优化方向是让 lowering/analyzer 直接产出完整 metadata。**

### 3.4 AOT cgen 全局状态（成立，高风险）

`src/aot/xi_cgen.c`：

```c
#define CG_MAX_SHARED  512
#define CG_MAX_METHODS 256
#define CG_MAX_IMPORTS 256
static XiCgenCtx g_ctx;
```

影响：
- codegen 不可重入
- 并行 AOT 不安全
- 超过固定数量会静默丢信息或退化

### 3.5 AOT 静默占位（成立，高风险）

`src/aot/xi_cgen.c` 中存在危险占位：

| 占位 | 风险等级 |
|------|---------|
| `XR_NULL_VAL /* TODO: indirect call */` | 🔴 高 — 生成可编译但语义错误的 C |
| `XR_NULL_VAL /* closure: unknown */` | 🔴 高 |
| `0 /* unsupported is-check */` | 🔴 高 |
| `XR_NULL_VAL /* TODO: method %d args */` | 🟡 中 |
| `XR_NULL_VAL /* TODO: builtin '%s' */` | 🟡 中 |
| `XR_NULL_VAL /* unresolved import */` | 🔴 高 |
| `XR_NULL_VAL /* TODO: op %d */` | 🔴 高 |

**核心问题：unsupported path 可能生成可编译但语义错误的 C 代码。**

### 3.6 AOT 测试 retry（成立）

`tests/aot/run_aot_tests.sh`：

```bash
# retry up to 3x for intermittent XIR pass crashes
for attempt in 1 2 3; do ...
```

失败后标记为 SKIP 而非 FAIL，掩盖真实问题。

### 3.7 JIT lowering 路径（上一版过时 + 当前有新风险）

当前 JIT 走 `Xi IR → Xm`（`src/jit/xi_to_xm.c`），不再使用旧 builder。

但以下 op 未设置 `ctx->error`，而是返回 dummy 值：

| Op | 行为 | 风险 |
|----|------|------|
| `XI_DEFER` | 返回 `xm_const_i64(0)` | 静默丢弃 defer 语义 |
| `XI_IS` / `XI_AS` | 返回原值或 0 | 类型检查失效 |
| `XI_MULTI_RET` | 只返回第一个值 | 多返回值截断 |
| `XI_IMPORT_REF` | 返回 0 | 模块引用丢失 |
| `XI_COPY` 无参 | 返回 0 | 值丢失 |

### 3.8 回归测试 `--no-jit`（成立）

`scripts/run_regression_tests.sh` 对以下测试强制禁用 JIT：

- `1148_scope_race_stress.xr` — JIT regalloc overlap
- `1205_gc_incremental_pressure.xr` — JIT+GC interaction
- `1207_gc_stress.xr` — JIT+GC interaction

### 3.9 Json/Object 语义（`XR_KIND_OBJECT` 已有，但边界未清）

`src/runtime/value/xtype.h` 已定义：
- `XR_KIND_JSON` — 动态可扩展
- `XR_KIND_OBJECT` — 编译期固定字段，不可扩展

但以下位置仍合并处理：
- `xr_kind_is_object_like()` 把 JSON/OBJECT/INSTANCE/MAP 都归类
- `xanalyzer_mono.c` 对 OBJECT 返回 `"json"` mono name
- `xanalyzer_builtin_interfaces.c` 中 `is_json_type()` 合并 JSON+OBJECT
- `XrObjectType` 同时被 JSON 和 OBJECT 使用

**待定型问题：**
- object literal 默认推断为 `OBJECT` 还是 `JSON`？
- 哪些位置必须严格区分动态 Json 和静态 Object？
- LSP hover/completion 应展示 `object` 还是 `json`？

### 3.10 IO runtime 两套路径并存（需修正）

`src/coro/xnetpoll.c`：
- `xr_netpoll_block_sync()` — `xr_cond_wait()` 阻塞当前线程
  - 注释：used by cfunc coroutines and io.c connect that cannot use yieldable mechanism
- yieldable path — `old > XR_PD_WAIT` 时按 coroutine pointer 唤醒

`src/coro/xsocket.c`：
- `xr_socket_read()` / `xr_socket_write()` 在 EAGAIN 时调用 `xr_netpoll_block_sync()`

`stdlib/io/io.c`：
- 头注释明确：synchronous filesystem wrappers, blocking syscalls

**结论：netpoll 已有 yieldable 机制，但 socket 和文件 IO 仍走 sync-blocking path。**

### 3.11 pkg 命令（大部分已实现）

`src/app/cli/xcmd_pkg.c` 实现状态：

| 子命令 | 状态 |
|--------|------|
| `pkg init` | ✅ 完成 |
| `pkg add` | ✅ 完成（registry 查询 + 安装 + lockfile） |
| `pkg install` | ✅ 完成（含 native package CMake build） |
| `pkg tree` | ✅ 完成 |
| `pkg login` | ✅ 完成 |
| `pkg publish` | ✅ 完成（tarball + checksum + upload） |
| `pkg remove` | ❌ 返回 `XR_CLI_EXIT_UNAVAILABLE` |
| `pkg update` | ❌ 返回 `XR_CLI_EXIT_UNAVAILABLE` |

### 3.12 LSP / DAP（不是空壳）

**LSP** (`src/app/lsp/`)：
- analysis, completion, code action, call hierarchy, formatting, semantic tokens
- rename, workspace indexing, imports, inlay hints, async queue
- formatting 已正确 include `frontend/format/xfmt.h`

**DAP** (`src/app/dap/`)：
- breakpoints, controller, eval, inspect, protocol, transport, variables
- hot reload 返回 `false`（明确标注 Not Implemented）

### 3.13 stdlib/builtin metadata 分散（成立）

builtin 信息分布在至少 8 个位置：
- `src/frontend/analyzer/xanalyzer_builtins.c`
- `src/frontend/analyzer/xanalyzer_builtins_generated.h`
- `src/module/xbuiltin_method_defs.h`
- `src/module/xbuiltin_decl.h`
- `src/app/lsp/xlsp_stdlib.c`
- `src/app/lsp/xlsp_builtins.c`
- `stdlib/*/*.c`
- `src/aot/xaot_driver.c` 中 `stdlib_flag_for_import()` 手动表

### 3.14 README 版本不一致

- `CMakeLists.txt`：`VERSION 0.6.0`
- `README.md`：`We're at v0.5.x`

---

## 四、应删除或降级的上一版建议

| 建议 | 处置 | 原因 |
|------|------|------|
| 新增 `XR_KIND_OBJECT` | 删除 | 已在 `xtype.h` 实现 |
| CLI old arg routing 是主要风险 | 删除 | 已迁移到 spec-driven CLI |
| formatter 仍跨 app 依赖 CLI | 删除 | LSP 已 include `frontend/format/xfmt.h` |
| AOT 没有 metadata | 降级 | `XiModule` 已有，待收敛 |
| LSP/DAP 都是外围空壳 | 降级 | LSP 功能较完整，DAP 有核心能力 |
| pkg 全是 placeholder | 降级 | 6/8 子命令已实现 |

---

## 五、更新后的优先级路线图

### P0 — AOT cgen 正确性收敛

**目标：AOT 不再静默生成语义错误的 C 代码。**

| 任务 | 涉及文件 | 说明 |
|------|---------|------|
| 让 `xi_cgen` 返回错误状态 | `xi_cgen.c/h` | unsupported op 直接失败，不再只写 `FILE *` |
| 清理危险 `XR_NULL_VAL` 占位 | `xi_cgen.c` | indirect call / builtin / import / type check / default op |
| 对 marker 类占位做白名单 | `xi_cgen.c` | 非白名单占位全部禁止 |
| 移除 AOT 测试 retry | `run_aot_tests.sh` | 如果 crash 则当测试失败；如已修复则清理脚本 |

### P1 — JIT lowering 安全边界

**目标：不支持的 op 让函数 ineligible，不产生 silent semantic degradation。**

| 任务 | 涉及文件 | 说明 |
|------|---------|------|
| `XI_DEFER` 不支持则 ineligible | `xi_to_xm.c` | 当前返回 dummy 0 |
| `XI_IMPORT_REF` 不支持则 ineligible | `xi_to_xm.c` | 当前返回 0 |
| `XI_MULTI_RET` 禁止只返回第一个值 | `xi_to_xm.c` | 应 fallback 到 VM |
| `XI_IS` / `XI_AS` 确认语义等价 | `xi_to_xm.c` | 否则 fallback |
| 3 个 `--no-jit` 特例变为明确 JIT bug backlog | `run_regression_tests.sh` | 追踪修复而非永久禁用 |

### P2 — AOT cgen 架构债务

**目标：消除全局状态和固定容量限制。**

| 任务 | 涉及文件 | 说明 |
|------|---------|------|
| `g_ctx` 改为显式 `XiCgenCtx *` 参数 | `xi_cgen.c/h` | 为重入/并行做准备 |
| `CG_MAX_*` 改为动态扩容 | `xi_cgen.c` | 不再静默丢 import/method/class |
| AOT metadata 前移 | `xi_lower.c`, `xi.c` | 减少 pattern scanning |

### P3 — Json/Object 语义边界收敛

**目标：完成 Json 与 Object 的设计边界定型。**

| 任务 | 涉及文件 | 说明 |
|------|---------|------|
| 明确 assignability/coercion 规则 | `xtype.h`, analyzer | Json↔Object 何时允许隐式转换 |
| mono name 区分 | `xanalyzer_mono.c` | Object 不应返回 `"json"` |
| builtin interface 区分 | `xanalyzer_builtin_interfaces.c` | `is_json_type()` 拆分 |
| object literal 默认类型定型 | analyzer | 推断为 OBJECT 还是 JSON |
| LSP 展示类型一致 | `xlsp_completion.c` 等 | hover 应反映真实 kind |
| 编写 compile error 测试 | `tests/compile_errors/` | 覆盖 Json/Object 边界 |

### P4 — IO runtime 收敛

**目标：消除 worker 线程上的阻塞 IO，统一 IO 等待模型。**

| 任务 | 涉及文件 | 说明 |
|------|---------|------|
| 文件 IO 走 async pool | `stdlib/io/io.c` | 当前直接 `fopen/fread/fwrite` |
| socket read/write 走 yieldable wait | `xsocket.c` | 当前调用 `xr_netpoll_block_sync()` |
| cfunc/native blocking IO 进 syscall handoff | `xnetpoll.c`, `xworker.c` | 明确哪些 IO yield、哪些阻塞 |

### P5 — stdlib/builtin metadata 单一真相源

**目标：builtin/member 信息从单一 descriptor 生成或校验。**

| 任务 | 涉及文件 | 说明 |
|------|---------|------|
| 建立 canonical builtin descriptor | 新文件 | 统一 analyzer/LSP/AOT/runtime |
| `stdlib_flag_for_import()` 一致性测试 | `xaot_driver.c` | 与实际 stdlib registry 对齐 |
| 消除 LSP builtins 与 analyzer builtins 重复 | `xlsp_builtins.c`, `xanalyzer_builtins.c` | 共享数据源 |

### P6 — 生态补齐

| 任务 | 涉及文件 | 说明 |
|------|---------|------|
| `pkg remove` 实现 | `xcmd_pkg.c` | 当前返回 UNAVAILABLE |
| `pkg update` 实现 | `xcmd_pkg.c` | 当前返回 UNAVAILABLE |
| DAP hot reload | `xdap_variables.c` | 当前返回 false |
| README 版本修正 | `README.md` | `v0.5.x` → `v0.6.0` |
| README 状态表修正 | `README.md` | 与当前源码/测试脚本一致 |

---

## 六、关键风险矩阵

| 风险 | 影响 | 当前缓解 | 建议 |
|------|------|---------|------|
| AOT 生成语义错误的 C | 用户运行 native binary 得到错误结果 | 无（静默） | P0：cgen failure API |
| AOT crash 被 retry 掩盖 | AOT 稳定性假象 | 3 次 retry | P0：移除 retry |
| JIT silent semantic degradation | hot path 执行结果错误 | 部分 op 有 `ctx->error` | P1：全覆盖 ineligible |
| JIT+GC interaction bug | 压力测试不稳定 | `--no-jit` 绕过 | P1：追踪修复 |
| AOT `g_ctx` 全局状态 | 无法并行/重入 | 单线程使用 | P2：context 化 |
| AOT 固定容量上限 | 大项目静默丢信息 | 大多数项目未触及 | P2：动态扩容 |
| Json/Object 语义混合 | 类型检查不精确 | 运行时 OP_CHECKTYPE | P3：定型规则 |
| 文件 IO 阻塞 worker | 协程调度被卡住 | 单协程场景无感知 | P4：async pool |

---

## 七、测试现状快照

| 测试类别 | 规模 | 位置 |
|---------|------|------|
| 单元测试 | ~70+ test executables | `tests/unit/` |
| 回归测试 | ~330 .xr 文件 | `tests/regression/` |
| AOT 测试 | ~28 文件（含多模块） | `tests/aot/` |
| JIT 测试 | ~87 文件 | `tests/jit/` |
| 编译错误测试 | ~29 文件 | `tests/compile_errors/` |
| 协程安全测试 | ~12 文件 | `tests/coroutine_safety/` |
| 网络测试 | ~22 文件 | `tests/network/` |
| Work-stealing 测试 | ~9 文件 | `tests/work_stealing/` |
| Fuzz 测试 | lexer + parser | `tests/fuzz/` |

---

## 八、不做的事（显式排除）

- 不在此文档中规划新语言特性
- 不规划新后端（ARM32 / RISC-V64 / LoongArch）
- 不规划 1.0 发布时间线
- 不做代码实现，仅做分析和优先级规划
