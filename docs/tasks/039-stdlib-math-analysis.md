# stdlib/math 分析与优化建议

## 模块职责

`stdlib/math` 提供脚本层数学函数和常量，包括：

- 基础数值函数：`abs/floor/ceil/round/trunc`
- 幂与根：`sqrt/pow/cbrt/hypot`
- 三角与双曲函数
- 对数与指数函数
- 比较与约束：`min/max/clamp`
- 随机数：`random/randomInt`
- 数值判断：`sign/isNaN/isFinite`
- 常量：`PI/E/TAU/SQRT2/.../INF/NAN`

模块定位应该是薄封装 C 数学函数，同时把 Xray 的 int/float 类型语义定义清楚。当前实现已有较多边界修复，但类型声明、错误语义和整数范围处理仍需收敛。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/math/math.c` | native binding、常量导出、随机数实现 |
| `stdlib/math/math.h` | loader 声明 |
| `src/os/os_random.h` | CSPRNG 字节源 |
| `tests/regression/10_stdlib/1020_math.xr` | 基础 math 回归 |
| `tests/regression/10_stdlib/1021_math_advanced.xr` | 高级函数与 NaN/Inf |
| `tests/regression/10_stdlib/1022_math_type_preserve.xr` | 类型保留与边界 |
| `tests/regression/10_stdlib/1023_math_new_functions.xr` | 新函数和常量 |

## 当前行为契约

### 数字参数转换

`get_number()` 接受 int/float：

- int 转 double
- float 保持 double
- 非数字返回 NaN

这比“非数字当 0”更安全，但缺少统一的错误策略。缺参时很多函数仍返回 0 或 null，和非数字返回 NaN 的语义不一致。

### 类型保留

部分函数会尽量保留 int：

- `abs(int)` 返回 int，`INT64_MIN` 返回 float
- `floor/ceil/round/trunc(int)` 返回 int
- `floor/ceil/round/trunc(float)` 若结果可放入 int64 返回 int，否则返回 float
- `min/max/clamp` 在全 int 参数时返回 int，否则返回 float

这是实用设计，但当前 builtin signature 大多仍写成 float 入参 / float 或 int 返回，无法准确表达“preserve int where possible”。

### 随机数

`random()` 使用 `xr_random_bytes()` 生成 64-bit 随机数，取高 53 bit 转为 `[0, 1)` double。这个设计是正确的。

`randomInt(min, max)` 使用 rejection sampling 规避 modulo bias，方向正确，但整数范围计算存在严重边界问题。

## 依赖与架构边界

### 问题 1：`math.h` 依赖过重

`math.h` 只需要声明 loader，却 include：

- `src/module/xmodule.h`
- `src/vm/xvm.h`

这让一个简单 loader 头依赖 VM 层。

建议：

- loader 头只 include `xdefs.h` 并 forward declare `XrayIsolate` / `XrModule`。
- 或删除模块私有 loader 声明，统一由 `xmodule_loaders.h` 提供。

### 问题 2：loader 声明和定义缺少统一可见性修饰

`math.h` 和 `math.c` 中的 `xr_load_module_math()` 没有 `XR_FUNC`，但 `xmodule_loaders.h` 使用 `XR_FUNC` 声明。

建议：

- stdlib loader 全部统一声明和定义。
- 用静态检查覆盖所有 `xr_load_module_*`。

### 问题 3：builtin type declaration 与实际返回类型不匹配

示例：

- `math.abs` 声明 `(x: float): float`，但 int 输入返回 int。
- `math.min/max` 声明 `(...args: float): float`，但全 int 返回 int。
- `math.floor/ceil/round/trunc` 声明返回 int，但超出 int64 范围时返回 float。
- `math.clamp` 声明 float，但全 int 返回 int。

建议：

- 引入 union/generic descriptor 表达“int 输入保留 int”。
- 在类型系统尚未支持前，文档至少应明确实际返回类型。
- AOT / XIR 类型推导应与 VM runtime 行为保持一致。

## 内存与生命周期

`math` 模块没有长期堆对象或缓存。随机数调用 `xr_random_bytes()`，失败策略由 OS random 层 fail-fast 处理。

主要生命周期风险不在内存，而在数值溢出和类型不一致。

## 并发、阻塞与协程语义

`math` 函数都是 CPU 本地计算，不应 yield，也不应标记 SLOW。

`random()` / `randomInt()` 调用系统 CSPRNG。当前 `xr_random_bytes()` 是同步函数，但一般不会长期阻塞。若未来在某些平台可能阻塞，需要由 OS random 层统一定义策略，而不是在 math 模块单独处理。

## 数值语义与边界问题

### 问题 4：`randomInt()` 的全 int64 范围计算存在未定义行为

当前代码意图支持 `[INT64_MIN, INT64_MAX]`：

```c
uint64_t range = (uint64_t) (max_val - min_val) + 1;
```

但 `max_val - min_val` 先以 signed int64 计算。如果 `min_val = INT64_MIN` 且 `max_val = INT64_MAX`，减法在 cast 之前已经溢出，属于 C 未定义行为。

后续 `min_val + (int64_t) r` 在全范围情况下也可能触发 signed overflow。

影响：

- 极端输入下结果不可预测。
- 优化编译器可能基于未定义行为做错误优化。
- 随机边界用例缺少测试，当前回归无法发现。

建议：

- 使用纯 unsigned arithmetic 表达区间宽度。
- 对 full int64 range 单独构造结果，避免 signed overflow。
- 增加边界测试：`MIN_INT/MAX_INT`、单点区间、反向区间、大跨度区间。

### 问题 5：缺参和非数字参数错误语义不统一

当前行为：

- 非数字参数通过 `get_number()` 变为 NaN。
- 缺参时很多函数返回 `0`、`0.0` 或 `null`。

示例：

- `sqrt()` 返回 `0.0`
- `min()` 返回 `null`
- `abs()` 返回 `0`
- `clamp()` 返回 `null`

建议：

- 标准库统一决定“参数错误”是返回 null、NaN、抛错还是诊断。
- math 模块至少内部一致：缺参应与非数字参数策略一致，或明确区分 arity error 与 type error。

### 问题 6：NaN 传播语义没有统一定义

`min/max` 当前比较逻辑会忽略非首位 NaN：

- 如果第一个参数是 NaN，结果通常是 NaN。
- 如果后续参数是 NaN，比较为 false，结果可能保持前一个数值。

这与许多语言的 `Math.min/Math.max` “任一 NaN 则 NaN”不一致。`clamp` 也没有明确 NaN 规则。

建议：

- 明确 Xray math 的 NaN policy。
- 若采用 IEEE 风格传播，任何参数为 NaN 时返回 NaN。
- 为 `min/max/clamp/sign/isNaN/isFinite` 增加 NaN/Inf 测试。

### 问题 7：`isNaN()` 与 `get_number()` 语义不一致

`get_number(non-number)` 返回 NaN，但 `math.isNaN(non-number)` 只对 float 检查，非 float 直接 false。

这可能导致：

- `math.sqrt("x")` 结果是 NaN。
- `math.isNaN("x")` 结果是 false。

建议：

- 若 math 是 strict numeric API：非数字应是 type error，不应走 NaN。
- 若 math 采用 coercion API：`isNaN` 应使用同一 coercion 规则。
- 当前混合策略应统一。

### 问题 8：`clamp(min > max)` 未定义

当前实现没有处理 `lo > hi`。用户传入反向边界时行为依赖比较顺序。

建议：

- 明确是否自动交换 `min/max`，还是返回 null/NaN。
- 与 `randomInt()` 当前会自动交换 min/max 的行为保持一致，或刻意区分并写入文档。

## API 与 builtin 方法重复

`math.min/max` 与 number builtin method 中的 `max/min` 语义可能重叠。此前 AOT 对 `max/min` 多态返回类型已有历史问题，因此 math 模块需要成为语义对齐点之一。

建议：

- 统一 `math.min/max`、number method `max/min`、AOT bridge 的返回类型规则。
- 把“全 int 返回 int，否则 float/NaN”写成单一 helper，避免 VM/AOT/stdlib 分叉。

## 测试覆盖

现有回归覆盖较多，尤其是类型保留和新增函数。缺口集中在极端边界：

1. `randomInt(math.MIN_INT, math.MAX_INT)`。
2. `randomInt` 反向区间和单点区间。
3. `abs(math.MIN_INT)` 返回 float 的语义。
4. `floor/ceil/round/trunc` 对超出 int64 的 float 返回 float。
5. `min/max/clamp` 含 NaN、Inf、混合 int/float。
6. `isNaN` / `isFinite` 对非数字输入的语义。
7. 缺参行为是否稳定。
8. builtin type declaration 与 analyzer 类型检查是否一致。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 严重 | `randomInt()` signed range 计算可能溢出 | 极端范围未定义行为 | 改为 unsigned range 计算并补边界测试 |
| 高 | builtin signature 与实际返回类型不一致 | analyzer/AOT/文档漂移 | 引入 union/generic descriptor 或更新声明策略 |
| 高 | loader 可见性修饰不统一 | 违反 C 可见性规范 | 所有 `xr_load_module_*` 统一 `XR_FUNC` |
| 中 | `math.h` include VM 头 | 不必要架构耦合 | 改为 forward declare 或统一 loader 声明入口 |
| 中 | 缺参/非数字错误语义不统一 | 脚本层难以可靠处理错误 | 明确 arity/type error policy |
| 中 | NaN 传播语义不明确 | `min/max/clamp` 边界不稳定 | 定义 NaN policy 并测试 |
| 中 | `isNaN` 与 `get_number` 不一致 | 用户语义困惑 | strict 或 coercion 二选一 |
| 低 | `clamp(min > max)` 未定义 | 行为不可预期 | 明确交换或报错 |

## 后续实施建议

建议 `math` 的实施顺序如下：

1. 先修 `randomInt()` 的 signed overflow，并补极端边界测试。
2. 统一 loader 修饰和 `math.h` include 边界。
3. 明确并文档化参数错误、NaN、Inf、clamp 反向边界语义。
4. 梳理 builtin signature，至少让文档、类型声明、实际返回保持一致。
5. 把 `min/max` 多态语义与 number builtin method、AOT bridge 对齐。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
