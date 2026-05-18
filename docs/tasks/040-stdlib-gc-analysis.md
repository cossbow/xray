# stdlib/gc 分析与优化建议

## 模块职责

`stdlib/gc` 暴露运行时 GC 的脚本层控制面和观测面：

- 手动触发 GC：`collect`、`step`
- 启停控制：`disable`、`enable`、`isrunning`
- 内存统计：`count`、`countb`、`objects`、`debt`
- GC 状态：`state`、`info`
- 性能统计：`timems`、`fragmentation`
- 调参：`setpause`、`setstepmul`

这个模块不是普通算法库，而是 runtime 内部状态的用户可见窗口。它必须明确“观测的是当前 coroutine 的 GC，还是 isolate / runtime 全局 GC”。当前代码实际以当前 coroutine GC 为主。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/gc/gc.c` | native binding、runtime GC 状态读取与调参 |
| `stdlib/gc/gc.h` | loader 声明 |
| `src/runtime/gc/xcoro_gc.h` | `XrCoroGC` 结构和 GC API |
| `src/runtime/gc/xcoro_gc.c` | per-coroutine Immix / generational GC 实现 |
| `src/runtime/gc/xalloc_unified.c` | 当前 coroutine GC 获取与分配入口 |
| `tests/regression/10_stdlib/1204_gc.xr` | 基础 GC API 回归 |
| `tests/regression/10_stdlib/1205_gc_incremental_pressure.xr` | GC 压力与对象存活回归 |
| `tests/regression/10_stdlib/1206_gc_enhanced.xr` | 增强 API 和边界回归 |

## 当前行为契约

### 当前 GC 解析

`get_coro_gc()` 的顺序是：

1. `xr_current_coro(isolate)`
2. `xr_isolate_get_main_coro(isolate)`
3. 返回 NULL

因此脚本里 `gc.count()`、`gc.collect()`、`gc.disable()` 等操作的是“当前 coroutine 的 GC”。如果没有当前 coroutine，则 fallback 到 main coroutine。

### GC 控制

- `gc.collect()`：对当前 coroutine 执行 full GC，返回 `gc_count`。
- `gc.step()`：执行一步 GC，返回是否处于 `PAUSE`。
- `gc.disable()`：递增 `gc_disabled`，上限 255。
- `gc.enable()`：递减 `gc_disabled`。
- `gc.isrunning()`：判断 `gc_disabled == 0`。

### GC 统计

- `gc.count()` 返回 KB，类型为 float。
- `gc.countb()` 返回 bytes，类型为 int。
- `gc.objects()` 返回当前 GC 对象计数。
- `gc.info()` 返回一组 map 字段，包含 totalBytes、state、pause、stepMul、block stats 等。

## 依赖与架构边界

### 问题 1：`gc.h` 直接 include isolate internal

`stdlib/gc/gc.h` include `src/runtime/xisolate_internal.h`，但它只需要声明 loader。

影响：

- 标准库 header 暴露 runtime internal 依赖。
- 增加 include 环风险。
- 不利于 loader 声明统一和模块边界收敛。

建议：

- `gc.h` 只 forward declare `XrayIsolate` 和 `XrModule`。
- loader 声明统一到 `xmodule_loaders.h`，模块私有 header 不对外暴露 runtime internal。

### 问题 2：`gc.c` 直接读取 `XrCoroGC` 内部字段

`gc.c` 直接访问：

- `gc->totalbytes`
- `gc->gc_count`
- `gc->gc_disabled`
- `gc->gc_pause`
- `gc->gc_stepmul`
- `gc->immix`
- `gc->GCdebt`
- 多个统计字段

这使 stdlib 与 GC 内部布局强耦合。任何 GC 结构调整都会影响 stdlib。

建议：

- 在 runtime/gc 层提供稳定 introspection API，例如 `xr_coro_gc_get_stats()`、`xr_coro_gc_set_tuning()`。
- stdlib 只负责把 stats 转为脚本值，不直接读结构体字段。
- `XrCoroGC` 的详细结构继续留在 runtime/gc 内部。

### 问题 3：loader 可见性修饰不统一

`gc.c` 中 `xr_load_module_gc()` 没有 `XR_FUNC`，但 `xmodule_loaders.h` 中声明为 `XR_FUNC`。

建议与其他 stdlib loader 一起统一处理。

## 运行时语义问题

### 问题 4：模块注释和实际语义不一致

`gc.h` 写着手动控制函数是 no-op，但 `gc.c` 里 `collect/step/disable/enable/setpause/setstepmul` 都会直接修改当前 coroutine 的 GC。

影响：

- 使用者和维护者无法从 header 判断真实行为。
- 测试 `1206_gc_enhanced.xr` 依赖 disable/enable 真正生效。

建议：

- 明确 `gc` 模块是当前 coroutine GC 控制面。
- 如果未来要保留“只观测不控制”的安全模式，需要另设 feature gate，而不是让注释和实现冲突。

### 问题 5：当前 coroutine 语义不适合全局观测

脚本用户通常会直觉认为 `gc.countb()` 是“程序内存使用”，但当前实现是当前 coroutine 的 per-coro GC。多 coroutine 场景下：

- 主 coroutine 看不到其他 coroutine 的 GC 内存。
- worker 上运行时当前 coroutine 不同，返回值不同。
- `gc.collect()` 不会全局收集所有 coroutine。

建议：

- API 名称或文档明确 `current` 语义。
- 增加单独 API：`gc.current()`、`gc.global()` 或 `gc.all()`。
- runtime 提供 isolate 级聚合 stats，stdlib 再暴露给脚本。

### 问题 6：`gc.disable()` 是计数器，但脚本层没有获取嵌套深度

`gc.disable()` 可重复调用，`gc.enable()` 逐次递减。脚本层只能通过 `isrunning()` 得知是否为 0，无法知道当前 disable depth。

影响：

- 库代码嵌套 disable/enable 时难排查泄漏。
- `xr_coro_reset_for_call()` 会重置 `gc_disabled`，跨调用行为需要文档说明。

建议：

- 增加 `gc.disabledDepth()` 或在 `gc.info()` 中暴露 disable depth。
- 明确 coroutine reset 时 disable 状态会重置。

### 问题 7：`setpause/setstepmul` 上限与 runtime 常量不一致

`stdlib/gc/gc.c` 定义 `GC_PARAM_MAX = 10000`。但 `xcoro_gc.h` 中 runtime 自身有调参常量，例如 `XGC_PAUSE_MIN` / `XGC_PAUSE_MAX`，其中 pause adaptive bound 是 50..400。

影响：

- 脚本可以设置 `pause=10000`，但 runtime adaptive pause 会在某些路径 clamp 到 400。
- `stepmul` 的实际安全范围也没有统一由 runtime 暴露。
- stdlib 的调参规则可能与 GC 实现漂移。

建议：

- 调参范围由 runtime/gc 层定义单一真相源。
- `setpause/setstepmul` 调用 runtime setter，由 setter 返回是否接受。
- 对非法值返回错误或保留旧值，并在 `gc.info()` 暴露最终实际值。

### 问题 8：`gc.step()` 返回值语义在 generational mode 下不直观

`xr_coro_gc_step()` 在 generational mode 下可能执行完整 minor collection，并把 `gcstate` 保持为 `PROPAGATE` 以维持 barrier。`gc.step()` 返回 `gc->gcstate == XGC_PAUSE`，这在 gen mode 下可能长期 false。

建议：

- 重新定义 `gc.step()` 的返回语义，例如“是否做了 GC work”或“是否完成完整 cycle”。
- 对 gen mode 单独处理返回值。
- 增加测试覆盖 gen mode 下 `gc.step()` 返回。

## 安全与鲁棒性

### 问题 9：`gc.info()` 的 map key 每次重复 intern

`MAP_SET` 每次调用 `xrs_string_value_c()`，会反复 intern 字段名。虽然 intern 命中成本不大，但 `gc.info()` 是观测 API，可能被频繁调用。

建议：

- 复用 `stdlib_cache`，为 `gc.info()` key 增加 per-isolate cache。
- 或在 runtime stats API 中返回 C struct，由脚本转换时缓存 key。

### 问题 10：`gc.info()` 的 Map schema 是隐式契约

测试只检查部分字段存在，没有稳定 schema 定义。字段命名已经经历过混用 snake/camel 的历史。

建议：

- 固化 schema：字段名、类型、单位、是否可选。
- 若后续改名，提供一次性迁移或明确不兼容变更。
- 回归测试覆盖完整字段类型。

## 注释与规范风险

本轮只记录，不修改 C 代码。当前存在几处必须后续清理的源码注释风险：

- `gc.c` 注释引用了 `.md` 文档路径。
- `tests/regression/10_stdlib/1206_gc_enhanced.xr` 注释出现临时优化编号式说法。
- `gc.h` 注释说 no-op，与实现冲突。

建议在修改 `gc` 模块前先清理这些注释，避免后续 commit 被规则脚本拦截。

## 测试覆盖

现有覆盖相对较好：

- 基本 API 类型和非负值。
- `collect/step` 不崩溃。
- `disable/enable` 配对。
- `setpause/setstepmul` 返回旧值。
- `info()` 部分 key。
- `fragmentation()` 范围。
- GC 压力下对象存活。

缺口：

1. 多 coroutine 下当前 GC 与其他 coroutine GC 的区分。
2. `gc.collect()` 是否只影响当前 coroutine。
3. `gc.disable()` 嵌套深度与 reset 行为。
4. `setpause/setstepmul` 非法值、边界值和实际 clamp 后结果。
5. `gc.step()` 在 generational mode 下返回值。
6. `gc.info()` 完整 schema 类型校验。
7. `gc.count/countb` 和全局 runtime 内存的关系说明。
8. `gc.state()` 的合法状态列表当前测试包含 `FINALIZE`，但 runtime state name 里没有对应状态，需要确认是测试冗余还是旧状态残留。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | stdlib 直接读取 `XrCoroGC` 内部字段 | GC 结构重构会破坏 stdlib | 增加 runtime stats/tuning API |
| 高 | `gc.h` include isolate internal | 架构边界污染 | forward declare 或统一 loader 声明 |
| 高 | per-coro 语义对用户不明显 | 多 coroutine 下统计/collect 易误解 | API 文档或命名明确 current/global |
| 中 | `setpause` 上限与 runtime adaptive bounds 不一致 | 调参行为漂移 | runtime 提供唯一 setter 和 bounds |
| 中 | `gc.step()` 返回值在 gen mode 下不直观 | 脚本层难判断 cycle 是否完成 | 重定义返回语义并补测试 |
| 中 | disable counter 无可见深度 | 嵌套控制难排查 | 暴露 disabled depth |
| 中 | `gc.info()` schema 隐式 | 调用者依赖不稳定 | 固化字段名、类型、单位 |
| 低 | `gc.info()` 重复 intern key | 高频观测有额外开销 | 缓存 key |
| 低 | 注释与规则冲突 | 后续 lint/commit 风险 | 独立清理注释 |

## 后续实施建议

建议不要直接在 stdlib/gc 里继续加字段，而是先把 runtime 边界收敛：

1. 在 runtime/gc 层定义 `XrGCStats` 和 `xr_coro_gc_get_stats()`。
2. 定义 `xr_coro_gc_set_pause()` / `xr_coro_gc_set_stepmul()`，由 runtime 统一验证范围。
3. stdlib/gc 只做 binding 和 Map 转换。
4. 明确 API 分层：current coroutine stats、isolate aggregate stats、runtime/system stats。
5. 清理 `gc.h` include 和 loader 修饰。
6. 补多 coroutine 与调参边界测试。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
