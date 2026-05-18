# stdlib/toml 分析与优化建议

## 模块职责

`stdlib/toml` 提供 TOML 配置文件解析与序列化：

- `toml.parse(data)`：解析 TOML string
- `toml.parseStrict(data)`：解析并返回 `{data, errors, meta}`
- `toml.stringify(value, options?)`：Map/Json → TOML string
- `toml.parseFile(path)`：同步读取并解析 TOML 文件
- `toml.writeFile(path, value)`：序列化并同步写文件

该模块用于配置文件场景，正确性优先级高于宽松容错。后续重构应优先保证普通 parse、strict parse、C API、工具链签名和测试语义一致。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/toml/toml.h` | C API、loader、模块文档 |
| `stdlib/toml/toml.c` | stdlib binding、DOM bridge、serializer、file API |
| `stdlib/toml/toml_parser.h` | runtime parser config/result/context |
| `stdlib/toml/toml_parser.c` | runtime parser，主要用于 `parseStrict()` |
| `src/base/xtoml.h/c` | L0 pure C TOML parser，主要用于普通 `parse()` |
| `stdlib/datetime/datetime.h/c` | TOML datetime bridge |
| `stdlib/common_parser.h` | error map helper |
| `stdlib/common_writer.h` | serializer buffer helper |
| `stdlib/common_io.h` | sync file read/write helper |
| `tests/regression/10_stdlib/1306_toml_basic.xr` | TOML 基础测试 |
| `tests/regression/10_stdlib/1307_toml_advanced.xr` | TOML 高级字符串、array table、datetime、strict 测试 |

## 当前 API

| API | 当前语义 |
|---|---|
| `toml.parse(data)` | 普通解析，失败返回空 Map |
| `toml.parseStrict(data)` | 返回 Map：`{data, errors, meta}` |
| `toml.stringify(value, options?)` | 仅当 value 为 Map 时输出 TOML，否则空字符串 |
| `toml.parseFile(path)` | 同步读文件，失败返回空 Map |
| `toml.writeFile(path, value)` | 同步写文件，返回 bool |
| `xr_toml_parse(X, data, len)` | C API：普通 parse |
| `xr_toml_stringify(X, value, indent)` | C API：Map → TOML string |

## 当前支持范围

普通 parser `src/base/xtoml.c` 注释声明支持：

- basic / multiline / literal string
- integer：decimal/hex/octal/bin with underscores
- float：inf/nan
- boolean
- datetime：在 L0 DOM 里存 string，stdlib bridge 尝试转 DateTime
- array
- inline table
- standard table `[t]`
- array table `[[t]]`
- dotted keys

runtime strict parser `stdlib/toml_parser.c` 也实现类似能力，但与 `xtoml` 是独立实现。

## 当前架构优点

- 普通 parse 使用 L0 pure C parser `xtoml`，这一点比 runtime-coupled parser 更利于复用。
- `parseStrict()` 能收集结构化 errors，错误 schema 复用 `xrs_error_push()`。
- serializer 复用 `common_writer`，并针对 TOML string/control char 做 escape。
- duplicate key 和 duplicate table header 已有错误检测路径。
- datetime 已桥接到 runtime `DateTime`，不是只保留 string。
- array of tables 和 nested table stringify 已有基本支持。

## 依赖与架构边界

### 问题 1：普通 parse 与 strict parse 使用两套 parser

普通 parse：

```c
XrTomlValue *root = xtoml_parse(data, len);
```

strict parse：

```c
toml_parser_init(...);
TomlResult parse_result = toml_parser_parse(&parser);
```

影响：

- `toml.parse()` 与 `toml.parseStrict().data` 可能对同一输入产生不同结果。
- bug 修复需要改两份 parser。
- 数字、datetime、duplicate key、inline table、array table 行为容易漂移。
- 测试覆盖越多，维护成本越高。

建议：

- 单一 parser source。
- 推荐把 `src/base/xtoml` 扩展为返回 diagnostics/meta 的 parser。
- `parse()` 与 `parseStrict()` 都调用同一 parser，只是错误暴露策略不同。

### 问题 2：strict parser 直接依赖 runtime internal

`toml_parser.h` include `xisolate_internal.h`、`xarray.h`、`xmap.h`、`xjson.h`，解析过程中直接构造 `XrMap/XrArray/XrString`。

建议：

- 淘汰 runtime parser 或改成 bridge 层。
- 让 L0 `xtoml` 输出 DOM + diagnostics，stdlib bridge 负责 runtime object 构造。

### 问题 3：serializer 直接读取 Map/Json internal

`toml.c` 的 serializer 直接遍历 `XrMapNode` 和 `XrJson` shape/symbol table。

影响：

- runtime Map/Json storage 改动会影响 TOML serializer。
- 与 JSON/YAML/XML serializer 存在相同耦合。

建议：

- 增加 runtime object iterator API。
- data format serializer 全部通过统一 iterator 访问 Map/Json/Instance。

## API 类型与工具链同步

### 问题 4：builtin/LSP 声明为 `Json?`，实现返回 `Map`

builtin：

```text
parse(data: string): Json?
parseStrict(data: string): Json?
parseFile(path: string): Json?
```

实现：

- `toml.parse()` 返回 `xr_value_from_map(...)`
- `parseStrict()` 返回 `xr_value_from_map(result)`
- `parseFile()` 返回 Map 或空 Map

影响：

- analyzer/LSP 误导用户。
- Json dot access 与 Map index access 语义不一致。
- 与 YAML/CSV 有同类问题。

建议：

- 统一数据格式模块对象表示。
- 若返回 Map，声明改为 `Map<string, any>` 或 `any`。
- 若标准化为 Json，则 bridge 生成 `XrJson`。

### 问题 5：`parseStrict()` 名称语义不准确

`parseStrict()` 返回 `{data, errors, meta}`，即使 errors 非空也仍返回 data。它更像 `parseDetailed()`，不是严格拒绝错误。

建议：

- 改名为 `parseDetailed()`。
- 或实现真正 strict：errors 非空时 `data=null` 或抛标准错误。
- 与 YAML/CSV/XML 的 detailed/strict 命名统一。

### 问题 6：`stringify` 类型声明与实现不一致

生成签名里 `stringify(value: unknown): string`，LSP 中是 `fn(obj: Json): string`，实现只接受 Map，否则返回空字符串；同时实现接受 options/int indent，但 indent 当前 reserved。

建议：

- 统一声明：`(value: Map<string, any>, options?: Json|int): string` 的等价形式。
- 如果 indent 不生效，不要对外暗示格式化能力。

## Parse 错误模型

### 问题 7：普通 parse 失败返回空 Map

`xr_toml_parse()` 中：

```c
if (!root) return xr_value_from_map(xr_map_new(...));
```

`toml_parse_file()` 读失败也返回空 Map。

影响：

- 合法空 TOML、parse error、file read error 无法区分。
- 配置文件错误可能被静默忽略。

建议：

- `parse()` 失败返回 null，或返回 Result。
- 推荐用户使用 `parseStrict/parseDetailed`，但 `parse()` 文档必须明确错误行为。
- file I/O error 应进入 structured error。

### 问题 8：strict config 字段未明显改变 parser 行为

`TomlConfig.strict` 被设置，但当前看到的 parser 主要通过 errors 收集，不明显因为 strict=true 改变容错策略。

建议：

- 如果 strict 只是 detailed result，移除 strict flag。
- 如果 strict 要更严格，应对 recoverable syntax、duplicate key、invalid number 等直接 fail。

### 问题 9：invalid number 记录 error 后仍可能返回数值

runtime parser 中，遇到 signed hex/oct/bin 会 add error，但仍继续解析并返回负值。TOML spec 禁止带符号的 non-decimal integer。

建议：

- strict 模式下 invalid number 应返回 null/失败，不应继续产生看似合法的值。
- ordinary parse 也应与 spec 保持一致。

## TOML 语义问题

### 问题 10：datetime type mapping 文档过时

`toml.h` 文档写：

```text
Datetime -> string (ISO 8601 format)
```

但 `dom_to_xrvalue()` 会尝试：

```c
XrDateTime *dt = xr_datetime_parse(...);
if (dt) return xr_datetime_value(dt);
```

测试中也调用 `toString()` 检查 DateTime。

建议：

- 更新文档和 builtin/LSP 类型描述。
- 明确 TOML local date、local time、local datetime、offset datetime 的映射。
- 如果无法完整表示 TOML local date/time，应保留 string 或引入专门类型。

### 问题 11：DateTime bridge 继承 datetime 模块的边界问题

TOML datetime 解析依赖 `xr_datetime_parse()`，因此受 datetime 模块限制影响：

- local offset 语义
- invalid date normalize 风险
- local date/time 子集支持不足
- fractional seconds 精度可能只保留 milliseconds

建议：

- TOML datetime parser 不应只复用通用 DateTime parse。
- 对 TOML v1.0 的四类 date/time 类型分别建模或明确降级策略。

### 问题 12：array 类型一致性未检查

TOML v1.0 允许 array 元素类型不同，但很多配置语义希望同质。当前 parser 没有额外限制，这符合 TOML 1.0，但文档应说明。

### 问题 13：inline table 与 normal table mutation 边界需要测试

TOML 对 inline table 有封闭性要求：inline table 里定义的 key/table 不能之后再追加子键。当前 parser 的 `set_nested_value()` 和 table creation 策略需要专门测试。

建议：

- 增加 inline table 后追加 dotted key 的测试。
- 增加 normal table 与 dotted key 冲突测试。

### 问题 14：duplicate key/table 语义普通 parse 与 strict parse 可能不同

`xtoml` 的 `table_set()` 是 replace 语义；runtime parser duplicate key 保留 first 并记录 error。

这意味着：

- `toml.parse("a=1\na=2")` 可能返回 `a=2` 或 parse fail，取决于 xtoml 后续逻辑。
- `toml.parseStrict(...).data` 保留 `a=1` 并记录 error。

建议：

- duplicate 策略在唯一 parser 中统一。
- TOML spec 下重复定义应是 parse error。

## Serializer 语义问题

### 问题 15：`indent` 参数是 reserved，不生效

`toml.h` 和 `TomlWriter` 注释都说明 indent 不生效，writer emits flat TOML。

建议：

- 从用户文档中移除格式化暗示。
- 或实现 nested table value indentation。

### 问题 16：null 没有 TOML 表示，当前输出空字符串

`write_value()` 对 null 输出 `""`。注释说 table context skip null，但实际 `write_table()` 不跳过 null，会输出：

```toml
key = ""
```

影响：

- null roundtrip 变 empty string。
- 用户可能误以为 TOML 支持 null。

建议：

- stringifier strict 模式下遇到 null 返回错误。
- 默认模式可 skip null，但必须文档化。
- 不应注释与实现不一致。

### 问题 17：Map/Json non-string keys 可能输出非法 TOML

inline table branch 和 table branch对非 string key 会跳过写 key，但仍可能输出 ` = value`。

建议：

- serializer 限制 key 必须 string。
- 非 string key 返回错误或 stringify key。
- 增加测试。

### 问题 18：Map 输出顺序不稳定

Map 遍历基于 hash node 顺序。配置文件 stringify 如果用于生成文件，顺序稳定性很重要。

建议：

- 增加 key sorting option。
- 或使用 ordered map / insertion order。

### 问题 19：array-of-tables 检测只看第一个元素

`is_array_of_tables()` 只检查 array 第一项是否 Map。若后续元素不是 Map，会被忽略或产生混合语义。

建议：

- 检查所有元素。
- 非 Map 元素应报错或作为 ordinary array 处理。

### 问题 20：cycle guard 不是 cycle detection

serializer depth guard 到达上限时输出空字符串，避免栈溢出但隐藏循环。

建议：

- 增加 visited set。
- strict stringify 遇到 cycle 返回错误。

## 文件 I/O 与阻塞

### 问题 21：parseFile/writeFile 同步全量 I/O

`parseFile()` 全量读入文件；`writeFile()` 先完整 stringify 再写。与 CSV/YAML 相同，有阻塞 worker 和大文件内存风险。

建议：

- 标注为 blocking API。
- 增加 detailed file API 返回 I/O error。
- 后续统一迁移到 yieldable I/O。

### 问题 22：writeFile 不透传 stringify options

`toml_write_file()` 固定：

```c
xr_toml_stringify(X, args[1], 0)
```

即使 `toml.stringify()` 支持第二参数，`writeFile()` 也不支持。

建议：

- `writeFile(path, value, options?)` 与 `stringify(value, options?)` 对齐。

## 测试覆盖

现有覆盖：

- 基础 key-value。
- table、nested table。
- array、inline table。
- hex/octal/bin number。
- multiline basic/literal string。
- array of tables。
- escape/unicode escape。
- dotted keys、inline table dotted keys。
- special floats、negative numbers。
- stringify roundtrip、nested roundtrip。
- parseStrict 基础和 duplicate key。
- comments、bool EOF。
- datetime、space separator datetime。
- underscore number。

主要缺口：

1. 普通 parse 与 parseStrict 同输入一致性。
2. parse error vs empty TOML vs file read error 区分。
3. invalid number 不应继续产生值。
4. duplicate key/table 在 ordinary parse 和 strict parse 的统一行为。
5. inline table 封闭性。
6. array-of-tables 混合元素 stringify。
7. TOML local date / local time / offset datetime 精确映射。
8. DateTime fractional seconds 精度。
9. null stringify 策略。
10. non-string Map key stringify。
11. indent reserved 行为。
12. sorted/stable output。
13. parseFile/writeFile error detail。
14. Unicode invalid escape 和 surrogate 边界。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 严重 | `parse()` 与 `parseStrict()` 使用两套 parser | 语义漂移、维护成本高 | 单一 parser + diagnostics |
| 高 | 声明 `Json?` 但实现返回 Map | 静态类型与运行时不一致 | 统一对象表示或修正声明 |
| 高 | parse/file failure 返回空 Map | 错误被静默忽略 | 返回 null/Result/detailed error |
| 高 | TOML datetime 文档与实现不一致 | 用户误解类型 | 更新文档并明确 TOML date/time 映射 |
| 高 | invalid number 可能记录 error 后仍返回值 | strict 不严格 | errors 非空时 fail 或返回 null |
| 中 | duplicate key 策略两 parser 可能不同 | data 不一致 | 统一 duplicate handling |
| 中 | null stringify 变 empty string | 数据语义丢失 | skip/null-error 策略 |
| 中 | serializer 直接读 runtime internals | runtime 改动影响 stdlib | object iterator API |
| 中 | non-string key 可能输出非法 TOML | 输出不合法 | key validation |
| 中 | sync full-file I/O | 阻塞和内存风险 | yieldable/streaming I/O |
| 低 | indent 参数不生效 | 用户配置误导 | 隐藏或实现 |
| 低 | Map 输出顺序不稳定 | 配置 diff 不稳定 | sort/order option |

## 后续实施建议

建议先做 parser 单一化：

1. 扩展 `src/base/xtoml` 支持 diagnostics/meta。
2. 删除或降级 `stdlib/toml_parser.c` 为兼容 bridge，不再独立解析。
3. 统一 `parse()` 与 `parseStrict/parseDetailed()` 返回类型与错误策略。
4. 明确 TOML datetime 四类类型映射。
5. 修正 builtin/LSP 签名。
6. 定义 stringify 对 null、non-string key、cycle 的策略。
7. 增加 strict conformance tests，覆盖 duplicate、inline table closure、invalid number、date/time。
8. 标注 file API blocking 并规划 yieldable I/O。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
