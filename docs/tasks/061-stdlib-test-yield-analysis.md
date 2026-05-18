# stdlib/test_yield 分析与优化建议

## 模块职责

`stdlib/test_yield` 是一个用于验证 Yieldable C function 协议和协程调度恢复路径的测试支撑模块。它提供一组人为构造的 yieldable native 函数，用于覆盖：

- C 函数主动 `xr_yield()` 后恢复。
- yieldable 函数携带 C state 跨调度保存和释放。
- 多次 yield 的状态机路径。
- continuation 返回 `XR_CFUNC_ERROR` 的错误路径。
- coroutine cancel 后 continuation 收到 `XR_RESUME_CANCELLED` 的清理路径。
- 并发调用下的全局计数器测试。
- 长任务分片执行和调度公平性测试。

从职责上看，它不是面向用户的标准库能力，而是 runtime/coro/VM 的测试探针。它的合理归属更接近 `tests`、`src/coro` 的测试 fixture，或仅在测试构建中注册的 internal module。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/test_yield/test_yield.c` | 实现测试模块、yieldable 函数和模块导出 |
| `stdlib/test_yield/test_yield.h` | loader 声明 |
| `src/coro/xyieldable.h` | Yieldable C function 协议、resume status、continuation 类型 |
| `src/coro/xyieldable.c` | `xr_yield()`、`xr_yield_for_io()`、continuation frame setup |
| `stdlib/common.h` | `XRS_EXPORT_YIELDABLE` 导出宏 |
| `src/module/xmodule_loaders.h` | `xr_load_module_test_yield` loader 声明 |
| `src/module/xmodule.c` | 将 `test_yield` 注册到 `stdlib_filesystem` |
| `tests/regression/11_coroutine/1128_yield.xr` | 语言级 `yield` 回归测试，不直接覆盖 `test_yield` 模块 |

## 当前导出 API

`xr_load_module_test_yield()` 创建模块名 `test_yield`，导出如下函数：

| API | 类型 | 当前语义 |
|---|---|---|
| `simple()` | yieldable | yield 一次后返回 `42` |
| `add(a, b)` | yieldable | 保存参数到 C state，恢复后返回和 |
| `sync()` | sync | 直接返回 `100`，用于和 yieldable 对照 |
| `multi_yield(steps?)` | yieldable | 最多 100 次分步 yield，返回 `10 + 20 + ...` |
| `chain(n?)` | yieldable | 最多 1000 步，分步累加 `1..n` |
| `error_test(should_error?, error_code?)` | yieldable | yield 后可返回 `XR_CFUNC_ERROR` |
| `cancel_test(resource_id?)` | yieldable | cancel resume 时释放 state 并返回负 resource id |
| `nested()` | yieldable | 多层 yield 后返回 `150` |
| `long_task(n?)` | yieldable | 最多 10000 次分片累加平方和 |
| `counter_inc()` | yieldable | 增加全局 atomic counter，yield 后返回当前值 |
| `counter_get()` | sync | 读取全局 counter |
| `counter_reset()` | sync | 重置全局 counter 并返回旧值 |

这些 API 的返回值是测试约定，不具备通用标准库语义。

## 当前架构优点

- **覆盖 yieldable 协议关键路径**：包含单次 yield、多次 yield、状态保存、错误、取消、并发计数和长任务分片。
- **内存分配基本规范**：C state 使用 `xr_malloc/xr_free`，没有直接 `malloc/free`。
- **取消路径有清理逻辑**：`error_test` 和 `cancel_test` 在 `XR_RESUME_CANCELLED` 下释放 state。
- **并发 counter 使用 atomic**：避免并发测试中出现普通数据竞争。
- **参数上限存在**：`multi_yield`、`chain`、`long_task` 对 steps/n/iterations 有 cap，避免测试调用无限耗时。
- **对 runtime/coro 调试有价值**：它能作为 yieldable C function 协议变更时的烟测模块。

## 主要问题

### 1. 测试模块被当成普通 stdlib 注册

`src/module/xmodule.c` 把 `test_yield` 放入 `stdlib_filesystem`：

```text
io
os
test_yield
```

这意味着用户可以 `import test_yield`，并获得一组测试探针函数。该模块名、API 名和返回值都明显是测试用途，不应出现在生产标准库默认命名空间。

建议：

- 从普通 stdlib 注册表移除。
- 放入仅测试构建启用的 internal module group。
- 或改名为更明确的 internal 名称，并默认不随 release binary 暴露。

### 2. modular build 存在链接风险

CMake 当前逻辑：

- `XR_STDLIB_FULL=ON` 时递归编译所有 `stdlib/*/*.c`，包含 `test_yield`。
- modular build 中 `XR_STDLIB_FILESYSTEM=ON` 只编译 `stdlib/io/*.c` 和 `stdlib/os/*.c`。
- 但 `xmodule_loaders.h` 和 `xmodule.c` 在 `XR_HAS_FILESYSTEM` 下仍声明并注册 `xr_load_module_test_yield`。

因此在 `XR_STDLIB_FULL=OFF` 且 filesystem 启用时，注册表引用 `xr_load_module_test_yield`，但 object 文件未编译进来，可能导致 undefined symbol。

建议：

- 不把 `test_yield` 归入 filesystem group。
- 增加独立构建开关，例如 `XR_STDLIB_TEST_MODULES` 或 `XR_BUILD_TEST_MODULES`。
- CMake source inclusion、loader declaration、module registration 必须由同一个宏控制。

### 3. 存在文件作用域可变全局状态

`test_yield.c` 使用：

```text
static _Atomic int64_t g_counter = 0
```

即使 atomic 避免数据竞争，它仍是进程级共享状态。作为测试 fixture 可以接受，但作为标准库模块会破坏 isolate 隔离和可预测性：

- 不同 isolate 共享 counter。
- 不同测试或用户脚本之间可能互相污染。
- `counter_reset()` 是全局副作用。

建议：

- 如果保留测试模块，counter state 应挂在测试 runtime/isolate/module context 上。
- 如果只用于内部测试，可在文档和构建开关中明确它不属于生产 API。

### 4. include 方向偏高层且暴露 VM 依赖

`test_yield.c` 直接 include：

```text
../../src/vm/xvm.h
../../src/coro/xyieldable.h
../../src/coro/xcoroutine.h
../../src/runtime/xexec_frame.h
```

原因是 `XRS_EXPORT_YIELDABLE` 需要 `xr_vm_yieldable_cfunction_new()`。这说明 yieldable stdlib binding 当前依赖 VM 层构造函数。

风险：

- stdlib native module 与 VM 绑定过深。
- 任何用户可见 stdlib module 若使用 yieldable export，都可能复制这个 include 模式。
- 架构上更理想的是 module/runtime 提供注册抽象，避免 stdlib 直接依赖 VM 头。

建议：

- 短期：将 `test_yield` 标记为内部测试模块，降低架构污染影响。
- 中期：把 yieldable cfunction constructor 下沉或通过 module 层 wrapper 暴露，避免 stdlib include VM。

### 5. loader 定义缺少统一可见性修饰

`xmodule_loaders.h` 中声明为：

```text
XR_FUNC struct XrModule *xr_load_module_test_yield(...)
```

但 `test_yield.h` 和 `test_yield.c` 中使用无修饰符声明/定义：

```text
XrModule *xr_load_module_test_yield(...)
```

这违反“非 static 函数必须带 `XR_FUNC`/`XRAY_API`”的可见性规则，也会让 loader 的导出风格与其他模块注册入口不一致。

建议：

- 若保留该 loader，声明和定义统一使用 `XR_FUNC`。
- 若改为测试 fixture，可移出 stdlib loader public declaration 路径。

### 6. analyzer/LSP 没有声明该模块

`test_yield.c` 没有 `// @module test_yield` 和 `XR_DEFINE_BUILTIN` 声明；analyzer generated metadata 和 LSP 也没有对应符号。

这本身支持“它不应是用户标准库”的判断。但当前 runtime 又公开注册了模块，导致：

- runtime 可 import。
- analyzer/LSP 不知道该模块。
- 用户体验和工具链状态不一致。

建议：

- 如果它是 internal test module，不需要 analyzer/LSP 声明，但必须从普通 stdlib 暴露面移除。
- 如果决定保留为公开模块，就必须补齐类型声明、文档和测试。

### 7. 当前回归测试没有直接覆盖 `test_yield` 模块

搜索 `tests/` 未发现 `import test_yield` 或 `test_yield.*`。现有 `tests/regression/11_coroutine/1128_yield.xr` 覆盖的是语言级 `yield` 语句，不是 native yieldable C function 模块。

这意味着 `test_yield` 的核心价值没有被持续验证：

- `simple/add/multi_yield/chain` 未测。
- `error_test` 的 `XR_CFUNC_ERROR` 路径未测。
- `cancel_test` 的 `XR_RESUME_CANCELLED` 清理路径未测。
- `counter_inc` 的并发 atomic 路径未测。
- `long_task` 的大量 continuation 路径未测。

建议新增专门测试，或如果模块废弃则直接移除。

### 8. `1128_yield.xr.expected` 与脚本输出疑似不一致

`1128_yield.xr` 最后打印：

```text
test5: sums = <a> <b> <c>
```

但 `.expected` 中是：

```text
test5: total = 3
```

这不是 `test_yield` 模块自身的问题，但它位于同一协程/yield 测试区域，说明 yield 相关回归测试可能存在 expected 漂移或测试 runner 未严格比对该文件。

建议：

- 单独检查 regression runner 对该用例的 expected 匹配策略。
- 修正脚本或 expected，避免 yield 测试给出虚假信号。

### 9. 错误路径返回值与异常语义不清晰

`error_test()` 在 continuation 中设置 `*result = xr_int(code)` 后返回 `XR_CFUNC_ERROR`。VM 对 `XR_CFUNC_ERROR` 的处理通常是 runtime error，而不是正常返回该 int。

测试函数的目标可能正是触发错误路径，但 API 名和返回约定容易误导。建议在测试用例中显式断言：

- `XR_CFUNC_ERROR` 是否抛异常。
- `result` 是否会被忽略。
- continuation state 是否释放。

### 10. 取消场景没有真正异步等待

`cancel_test()` 只调用一次 voluntary `xr_yield()`，恢复很快发生。若测试脚本要取消它，可能存在调度时序不稳定：任务可能已经完成，取消路径未必执行。

建议：

- 使用 timeout/I/O wait 或多次 yield 构造更可靠的 pending 状态。
- 或在测试 runner 中显式控制调度点。

## API 归属判断

`test_yield` 不建议作为正式标准库模块保留，原因如下：

- 名称和 API 均是测试语义。
- 有进程级全局状态。
- 没有 analyzer/LSP 声明。
- 没有用户文档价值。
- modular build 注册与编译条件不一致。
- include VM/coro/runtime 内部头，暴露架构测试探针属性。

建议归属：

| 方案 | 建议 |
|---|---|
| 公开 stdlib 模块 | 不建议 |
| full build 默认注册 | 不建议 |
| 测试构建 internal module | 推荐 |
| 移到 `tests/fixtures/native` | 推荐 |
| 作为 yieldable 协议示例 | 可保留但不随 release 暴露 |

## 后续实施建议

1. 增加 `XR_BUILD_TEST_MODULES` 或等价开关，只有测试构建注册 `test_yield`。
2. 从 `stdlib_filesystem` 中移除 `test_yield`，避免把测试模块归入 filesystem 能力组。
3. 让 CMake source inclusion、loader declaration、registration 三者使用同一个宏。
4. 如果保留模块，补齐 `XR_FUNC` loader 定义，并把 counter state 从全局迁移到 isolate/module context。
5. 增加直接 `import test_yield` 的回归测试，覆盖所有导出函数和错误/取消路径。
6. 修正 `1128_yield.xr` 与 `.expected` 的输出漂移。
7. 中期重构 `XRS_EXPORT_YIELDABLE`，让 stdlib 不必直接 include VM 头即可注册 yieldable native function。

## 建议验收清单

- release/full stdlib 不再默认暴露 `test_yield`。
- modular filesystem build 不再引用未编译的 `xr_load_module_test_yield`。
- 测试构建可显式启用 `test_yield` 并通过专门回归测试验证。
- 所有 non-static loader 定义带 `XR_FUNC`，或该 loader 不再处于 stdlib loader public 路径。
- 没有生产路径依赖 `g_counter` 这类进程级测试状态。
- yieldable C function 注册不再要求普通 stdlib 模块 include VM 内部头。

## 结论

`stdlib/test_yield` 的实现对验证 yieldable C function 协议很有价值，但它不是用户标准库。当前最大问题是“测试支撑模块被注册到正式 stdlib 命名空间”，并且 modular build 中编译条件与注册条件不一致。后续应把它迁移为测试专用 internal module，补齐直接回归测试，同时收敛 yieldable export 对 VM 头的依赖。
