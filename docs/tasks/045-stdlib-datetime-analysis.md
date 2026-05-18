# stdlib/datetime 分析与优化建议

## 模块职责

`stdlib/datetime` 提供 DateTime native type 和日期时间操作：

- 当前时间：local / UTC
- 从组件或 timestamp 创建 DateTime
- ISO-like 字符串解析
- 格式化输出
- 年/月/日/时/分/秒/毫秒等组件访问
- 日期加减与差值
- UTC/local 转换
- 闰年、月天数等工具方法

该模块应明确区分两层语义：

- 时间点：Unix timestamp + milliseconds
- 展示视图：UTC 或 local timezone 下的年月日时分秒

当前实现以 `timestamp` 为绝对时间点，以 `is_utc` 决定格式化和组件访问时使用 `gmtime` 还是 `localtime`。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/datetime/datetime.h` | `XrDateTime` 结构、C API、loader 声明 |
| `stdlib/datetime/datetime.c` | 创建、解析、格式化、组件、比较、算术、loader |
| `stdlib/datetime/datetime_methods.c` | native method table 的另一套方法 dispatch |
| `src/os/os_time.h` | 当前实时时钟抽象 |
| `tests/unit/stdlib/test_datetime.c` | C-level 组件、格式、比较、offset 单测 |
| `tests/regression/10_stdlib/1421_datetime_format.xr` | 脚本层格式化回归 |
| `tests/regression/10_stdlib/1422_datetime_calc.xr` | 脚本层加减、diff、timezone、比较、月加法回归 |
| `src/frontend/analyzer/xanalyzer_builtins_generated.h` | DateTime builtin 生成签名 |
| `src/app/lsp/xlsp_stdlib.c` | LSP datetime 手写提示 |

## 当前 API

### 模块函数

| API | 当前语义 |
|---|---|
| `datetime.now()` | 当前 local DateTime |
| `datetime.utc()` | 当前 UTC DateTime |
| `datetime.create(...)` | 组件创建 local DateTime |
| `datetime.createUTC(...)` | 组件创建 UTC DateTime |
| `datetime.fromTimestamp(ts)` | 从 Unix 秒创建 UTC DateTime |
| `datetime.fromTimestampMs(ts)` | 从 Unix 毫秒创建 UTC DateTime |
| `datetime.parse(s, format?)` | 解析字符串，失败返回 null |
| `datetime.offset()` | 当前 local UTC offset，单位分钟 |

### DateTime 方法

| API | 当前语义 |
|---|---|
| `format(pattern?)` | 按占位符格式化 |
| `toISOString()` | ISO-like 输出，UTC 用 `Z`，local 用 offset |
| `year/month/day/hour/minute/second/millisecond` | 组件 getter |
| `weekday/yearday/timestamp` | 周几、年内日、Unix 秒 |
| `add(amount, unit)` | 加日期或时间单位 |
| `diff(other, unit?)` | 两个 DateTime 时间点差值 |
| `toUTC/toLocal` | 切换展示视图 |
| `isBefore/isAfter/equals` | 时间点比较 |
| `isLeapYear/daysInMonth` | 日历工具 |

## 依赖与架构边界

### 问题 1：`datetime.c` 依赖 isolate internal GC

`datetime_alloc()` 使用：

```c
xr_gc_alloc(&isolate->gc, sizeof(XrDateTime), XR_TDATETIME)
```

因此 `datetime.c` include `xisolate_internal.h` 并直接访问 isolate GC。

影响：

- stdlib native type 与 isolate 内部结构耦合。
- 如果 runtime 迁移到 per-coroutine allocation 或统一 object factory，datetime 需要同步修改。

建议：

- 提供 `xr_datetime_new()` 或 runtime object allocation API。
- native type 的分配路径不要直接访问 isolate 内部 GC 字段。

### 问题 2：`datetime.h` 暴露 GC header 和 runtime value 细节

`XrDateTime` 结构体在 header 中公开，并包含 `XrGCHeader`、`XrValue` 相关宏。

影响：

- C-level 使用者依赖对象布局。
- 单测甚至复制了 `XrDateTime` layout，说明 header 自包含和测试边界不理想。

建议：

- 若 DateTime 只作为 runtime heap object，应考虑隐藏结构体布局。
- C-level 只暴露 accessor 和 constructor。
- 测试直接 include 正式 header，而不是复制结构。

### 问题 3：`datetime.c` 与 `datetime_methods.c` 存在两套 method binding

`datetime.c` 中有 `datetime_methods[]` / `datetime_getters[]`，`datetime_methods.c` 又定义 `xr_datetime_method_table`。

风险：

- 方法名、arity、flag、实现可能漂移。
- AOT/JIT/builtin method 调度如果使用不同表，行为会不一致。

建议：

- 确认当前 runtime 到底使用哪一套 method table。
- 将 DateTime 方法绑定单一化。
- 如果两套分别服务 native type 和 symbol fast path，需要明确生成源或同步机制。

## 时间与时区语义

### 问题 4：local offset 使用当前 offset，不是目标时间点 offset

`xr_datetime_local_offset()` 基于 `time(NULL)` 计算当前本地 offset。

`datetime.create()`、`now()`、`toLocal()` 都用当前 offset 填入 `dt->tz_offset`。

影响：

- 创建历史或未来时间时，`toISOString()` 显示的 offset 可能不是该时间点的 DST/时区 offset。
- `datetime.create(1970, ...)` 在 DST 地区可能显示错误 offset。

建议：

- offset 应根据目标 timestamp 计算。
- 增加 `xr_datetime_offset_at(timestamp)`。
- `tz_offset` 可以作为缓存字段，但必须与 timestamp 一致。

### 问题 5：`toUTC()` / `toLocal()` 只切换视图，不改变 timestamp

这符合“同一时间点不同视图”的设计。但 `tz_offset` 当前计算不随 timestamp 精确，导致 local ISO 字符串可能有偏差。

建议：

- 文档明确转换不改变时间点。
- 修复 offset-at-time 后补 DST 测试。

### 问题 6：解析带 offset 的字符串丢失原始 offset

`xr_datetime_parse()` 解析 `+HH:MM/-HH:MM` 后会转换到 UTC timestamp，并设置：

- `tz_offset = 0`
- `is_utc = 1`

因此原始输入 offset 不再可见。

建议：

- 如果 DateTime 只是时间点，当前可以接受。
- 如果需要保留 zoned datetime，应新增字段或新类型。
- 文档明确 parse offset 后归一到 UTC。

## 日期有效性与算术

### 问题 7：`create()` 没有严格校验日期组件

`xr_datetime_create()` 直接填 `struct tm` 后调用 `mktime/timegm`。C 库会自动归一化非法日期：

- 2024-02-31 可能变成 2024-03-02。
- 13 月可能进位到下一年。
- 24 点可能进位到下一天。

影响：

- 脚本层 `datetime.create()` 永远返回 DateTime，错误输入不易发现。
- 类型系统无法提前捕获日历无效值。

建议：

- 对 year/month/day/hour/minute/second 做严格范围校验。
- day 根据 year/month 校验。
- 非法参数返回 null 或抛出标准错误。
- 如果保留 normalize 行为，应命名为 `normalizeCreate` 或文档说明。

### 问题 8：`parse()` 也缺少严格 roundtrip 校验

`parse()` 最终也依赖 `mktime/timegm`，没有确认归一化后的组件与输入一致。

建议：

- parse 后用 `gmtime/localtime` 回读组件，确认 year/month/day/hour/min/sec 一致。
- 非法日期返回 null。

### 问题 9：`fromTimestampMs()` 对负毫秒处理错误

当前逻辑：

```c
dt->timestamp = timestamp_ms / 1000;
dt->milliseconds = timestamp_ms % 1000;
```

C 中负数取模会产生负余数。因此 `fromTimestampMs(-1)` 会得到：

- timestamp = 0
- milliseconds = -1

这违反 `milliseconds` 的 0..999 不变量。

建议：

- 采用 floor division 语义。
- 保证 milliseconds 始终 0..999。
- 增加负 timestamp 毫秒测试。

### 问题 10：add/diff 可能发生 int64 溢出

`xr_datetime_add()` 中有：

- `dt->timestamp * 1000 + dt->milliseconds + amount`
- `amount * 60/3600/86400/604800`
- `dt->timestamp + seconds`

`xr_datetime_diff()` 中有：

- `(dt1->timestamp - dt2->timestamp) * 1000`

都缺少 overflow guard。

建议：

- 增加 checked arithmetic helper。
- 溢出返回 null 或 clamp，并统一错误语义。

### 问题 11：unknown unit 直接写 stderr 并默认 seconds

`xr_datetime_add()` 对未知 unit：

```c
fprintf(stderr, "datetime.add(): unknown unit '%s'\n", unit);
seconds = amount;
```

影响：

- 标准库函数直接写 stderr，不符合统一错误模型。
- 用户 typo 会得到错误结果而非失败。

建议：

- 未知 unit 返回 null 或抛标准错误。
- `diff()` 对未知 unit 也不应默认 seconds。

## 格式化与解析

### 问题 12：format pattern 能力有限但未明确

当前只支持：

- `YYYY`
- `MM`
- `DD`
- `HH`
- `mm`
- `ss`
- `SSS`

不支持 timezone、weekday、month name、escaping literal。短期可以接受，但应文档化。

### 问题 13：`xr_datetime_format()` C API 仍保留固定 buffer 截断语义

内部已用 `XrCtxBuf` 构造完整输出，但 C API 最后只复制能放入 caller buffer 的前缀并返回截断长度。

建议：

- C API 返回所需长度，或提供 heap-alloc 格式化 API。
- buffer 不足不应被误认为成功输出完整结果。

### 问题 14：parse 的 format 参数不是格式字符串

`parse(s, format?)` 目前只识别：

- null / ISO8601 / iso
- date
- time
- 其他 fallback 到固定 datetime 格式

它不解析 `YYYY-MM-DD` 这类 format pattern。

建议：

- 改名为 `kind` / `mode`，或真正支持 format pattern。
- 文档明确当前是有限模式，不是格式化模板。

## 类型提示与工具链同步

### 问题 15：LSP datetime 签名与 builtin 漂移

`xanalyzer_builtins_generated.h` 与 `datetime.c` 基本一致，但 `xlsp_stdlib.c` 存在明显漂移：

- `fromTimestamp` 在 LSP 中描述为毫秒，但 runtime 是秒。
- `timestamp` 在 LSP 中描述为毫秒，但 runtime 返回秒。
- LSP 把 DateTime 方法写成 `fn(dt: DateTime): int` 风格，而脚本实际使用 dot syntax。
- LSP 缺少 `createUTC`、`fromTimestampMs`、部分 method。

建议：

- LSP stdlib signature 从 builtin 声明统一生成。
- DateTime 方法和模块函数分开建模。

## 测试覆盖

现有覆盖：

- C-level 组件访问、weekday、比较、闰年、月天数、格式、offset。
- 脚本层 add/diff/timezone/比较/闰年/月天数/月加法 clamp。
- 格式化脚本回归。
- TOML 中也间接覆盖 datetime parse/toString。

缺口：

1. `create()` 非法日期：2 月 30 日、13 月、24 点、60 分。
2. `parse()` 非法日期是否返回 null。
3. 负 `fromTimestampMs()`。
4. timestamp 极值 add/diff overflow。
5. DST 边界和历史/未来 offset。
6. parse offset 后是否归一 UTC。
7. `toISOString()` local offset 是否根据目标时间点计算。
8. unknown unit 是否报错。
9. `format()` 长 pattern 和 C buffer truncation。
10. LSP/analyzer signature 同步。
11. 两套 DateTime method table 行为一致性。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 严重 | `fromTimestampMs()` 负值产生负 millisecond | 破坏 DateTime 不变量 | 使用 floor division，补测试 |
| 高 | `create/parse` 依赖 C 库归一化非法日期 | 错误输入静默变成其他日期 | 严格组件校验 |
| 高 | local offset 使用当前时间 offset | 历史/未来和 DST 显示错误 | 根据 timestamp 计算 offset |
| 高 | 两套 DateTime method table | AOT/JIT/VM 行为可能漂移 | 单一 method table 或生成同步 |
| 高 | LSP 签名与 runtime 漂移 | IDE 提示误导用户 | 从 builtin 声明生成 LSP |
| 中 | add/diff 缺少 overflow guard | 极端时间计算未定义 | checked arithmetic |
| 中 | unknown unit 写 stderr 并默认 seconds | typo 产生错误结果 | 返回 null 或标准错误 |
| 中 | parse format 参数语义不是真 format | API 名称误导 | 改名或支持 pattern parse |
| 中 | C format API 截断无错误 | C 调用者可能误用 | 返回所需长度或 heap API |
| 低 | header 暴露 GC/object layout | 模块边界重 | 隐藏结构体或稳定 C API |

## 后续实施建议

建议按 correctness 优先处理：

1. 修复 `fromTimestampMs()` 负值毫秒归一化。
2. 为 `create()` 和 `parse()` 增加严格组件校验。
3. 增加 `xr_datetime_local_offset_at(timestamp)`，修复 DST/历史 offset。
4. 收敛 DateTime method table 到单一来源。
5. 同步 LSP 与 builtin 签名。
6. 增加 checked arithmetic。
7. 明确 parse format 参数和 DateTime string/zone 模型。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
