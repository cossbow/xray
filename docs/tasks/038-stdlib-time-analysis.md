# stdlib/time 分析与优化建议

## 模块职责

`stdlib/time` 提供脚本层可用的时间 API：

- 当前 Unix 时间戳
- 进程 CPU 时间
- 单调时钟
- 高精度单调时钟
- 协程友好的 sleep

该模块应该只负责时间读取和等待控制，不应该承担调度器内部细节、平台 clock 实现细节或测试辅助职责。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/time/time.c` | native binding 实现与 loader |
| `stdlib/time/time.h` | loader 声明 |
| `src/os/os_time.h` | 跨平台 monotonic/realtime/sleep 抽象 |
| `src/os/unix/time_unix.c` | POSIX 实现 |
| `src/os/win/time_win.c` | Windows 实现 |

现有导出：

| API | 当前语义 |
|---|---|
| `time.now()` | Unix epoch 毫秒 |
| `time.clock()` | 进程 CPU 毫秒，优先 `CLOCK_PROCESS_CPUTIME_ID` |
| `time.monotonic()` | 单调时钟毫秒 |
| `time.nanos()` | 单调时钟纳秒 |
| `time.micros()` | 单调时钟微秒 |
| `time.sleep(ms)` | yieldable sleep，毫秒单位 |

## 当前行为契约

### wall clock 与 monotonic clock

`time.now()` 使用 `xr_time_realtime_ns()`，适合人类可见时间戳，但可能受系统时间调整影响。

`time.monotonic()` / `time.nanos()` / `time.micros()` 使用 `xr_time_monotonic_ns()`，适合 elapsed time 和 timeout。

这个分工是正确的，但脚本层文档和测试注释没有明确强调“now 不应用于耗时计算”。

### sleep

`time.sleep()` 当前实现为 yieldable C function，调用 `xr_yield_for_timeout()`，不会直接阻塞 worker。这是正确方向。

实现接受 `int` 或 `float`，但类型声明是 `(ms: int): void`。float 会被截断为整数毫秒，负数和 0 直接返回 null。

## 依赖与架构边界

### 问题 1：`time.c` 依赖不必要的 VM internal 头

`time.c` include 了 `src/vm/xvm_internal.h`，注释写的是需要 `XrCoroState` 和 `XrCoroutine`，但当前实现没有直接使用这些类型。

影响：

- 标准库 time 模块不必要地依赖 VM internal。
- 后续拆分 VM、AOT runtime 或标准库动态构建时，这类 include 会放大耦合。

建议：

- 删除不必要的 `xvm_internal.h` include。
- `time.sleep()` 只保留 yieldable C function 所需的最小头。
- 若 `XrCFuncResult` 来自 runtime 执行帧，应通过稳定的 binding 头暴露，而不是依赖 VM internal。

### 问题 2：loader 声明和定义缺少统一可见性修饰

`time.h` 中声明：

```c
struct XrModule *xr_load_module_time(XrayIsolate *isolate);
```

`time.c` 中定义也没有 `XR_FUNC`。但 `src/module/xmodule_loaders.h` 中同一个 loader 使用了 `XR_FUNC` 声明。

影响：

- 同一函数在不同头里声明风格不一致。
- 违反项目对非 static C 函数可见性修饰的统一要求。
- 后续跨平台符号导出和静态检查容易误判。

建议：

- 所有 stdlib loader 的 public declaration 和 definition 统一加 `XR_FUNC`。
- 只保留一个权威 loader 声明来源，避免 `time.h` 与 `xmodule_loaders.h` 重复。

### 问题 3：`time.h` 依赖过重且不够自包含

`time.h` 只需要声明 `XrayIsolate` 和 `XrModule`，却 include 了 runtime value 头。

建议：

- 改为 include 基础可见性头并 forward declare `XrayIsolate` / `XrModule`。
- 或让 `xmodule_loaders.h` 成为 loader 声明唯一入口，模块私有 header 不暴露 loader。

## 内存与生命周期

`time` 模块本身没有动态分配，也没有跨调用缓存。生命周期风险较低。

需要注意的是 `time.sleep()` 的 continuation 当前不持有上下文，比较简单；如果后续增加 cancel token、deadline handle 或 timer object，需要明确所有权与取消路径。

## 并发、阻塞与协程语义

### 已做对的点

`time.sleep()` 已经从阻塞 sleep 变为 `xr_yield_for_timeout()`，不会占用 worker 等待。

### 问题 4：超大 sleep 值没有上限

`time.sleep()` 接受 int/float 毫秒值，未设置最大值。极端大值可能在 timer wheel、deadline 计算或平台转换中引出溢出风险。

建议：

- 定义脚本层最大 sleep 毫秒值。
- 超出上限返回错误或 clamp，二者必须明确。
- 增加边界测试：0、负数、1、很大值、float。

### 问题 5：`float` 参数被静默截断

实现接受 float 并强转为 `int64_t`，但类型声明只说 int。

建议二选一：

- 如果 API 只支持 int：拒绝 float，并让 analyzer / runtime 行为一致。
- 如果 API 支持 float：类型声明改为 `int | float`，并定义四舍五入、向下取整或至少等待语义。

## 安全与鲁棒性

### 问题 6：平台 clock 错误处理不完整

`src/os/unix/time_unix.c` 的 `clock_gettime()` 返回值未检查。正常系统上失败概率很低，但跨平台抽象层应明确失败策略。

建议：

- OS 层选择 fail-fast、fallback 或返回 0 的统一策略。
- 对 monotonic/realtime clock 失败添加断言或错误处理。

### 问题 7：`time.clock()` 直接使用平台 API，没有进入 OS 抽象层

`time.clock()` 在 `time.c` 里直接使用 `clock_gettime(CLOCK_PROCESS_CPUTIME_ID)`，Windows fallback 使用 `clock()`。这让平台逻辑散落到 stdlib 模块中。

建议：

- 在 `src/os/os_time.h` 增加 `xr_time_process_cpu_ns()`。
- `stdlib/time` 只做单位转换和 binding。

## API 与文档一致性

### 问题 8：模块 header 仍说“所有时间单位都是 milliseconds”

`time.h` 写着所有时间单位为 milliseconds，但模块已经导出：

- `time.nanos()`
- `time.micros()`

回归测试 `1010_time.xr` 也写着所有时间单位统一为毫秒。

建议：

- 文档改为按函数明确单位。
- API 命名保留 `nanos/micros`，但在测试和语言文档中说明它们是 monotonic clock。

### 问题 9：`sleep` 返回值声明为 void，实现返回 null

脚本层把 void/null 如何映射需要统一。当前 builtin signature 是 `void`，实现写入 `xr_null()`。

建议：

- 明确 native void binding 的实现约定：返回 `xr_null()` 是否就是 void。
- 在 binding helper 或测试中统一验证。

## 测试覆盖

现有回归：

- `tests/regression/10_stdlib/1010_time.xr`
- `tests/regression/10_stdlib/1011_time_edge.xr`

覆盖点：

- `now()` 为正且是毫秒级时间戳
- `clock()` 非负 / 单调不倒退
- `monotonic()` 不倒退
- `sleep(0)` 和 `sleep(10)` 基本行为

缺口：

1. `nanos()` / `micros()` 无覆盖。
2. `sleep` 的负数、float、超大值无覆盖。
3. `now()` 与 `monotonic()` 的语义差异无文档化测试。
4. `clock()` 没有覆盖 CPU time 与 wall time 的区别。
5. 没有 C 层 unit test 验证 OS time abstraction。
6. 没有类型检查测试验证 `time.sleep` 的 signature 与实现一致。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | loader 声明/定义缺少统一 `XR_FUNC` | 违反可见性规则，符号导出不一致 | loader 声明来源统一，定义加修饰 |
| 高 | `time.c` include VM internal 头 | stdlib 与 VM internal 耦合 | 移除不必要 include，暴露稳定 binding 头 |
| 中 | `sleep` 类型声明与实现不一致 | analyzer、文档、运行时行为可能漂移 | 决定 int-only 或 int/float |
| 中 | `time.h` 单位说明过时 | 用户和测试误解 API | 按函数声明单位 |
| 中 | 超大 sleep 无上限 | timer/deadline 潜在溢出 | 定义最大值和错误策略 |
| 中 | `time.clock()` 平台逻辑在 stdlib 内 | OS 抽象不完整 | 增加 process CPU time OS API |
| 低 | `nanos/micros` 缺少回归测试 | 高精度 API 可退化无感知 | 增加语义测试 |
| 低 | `time.c` 有未使用 include 和编号式注释 | 维护噪音 | 清理 include 与注释 |

## 后续实施建议

建议把 `time` 作为第一个小型修复库，因为它体量小、风险低、能验证标准库重构流程：

1. 整理 include，移除 VM internal 依赖。
2. 统一 loader 可见性修饰。
3. 明确 `sleep` 参数类型和边界策略。
4. 更新模块文档和回归测试，补 `nanos/micros`。
5. 如有必要，把 process CPU time 下沉到 OS 抽象层。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
