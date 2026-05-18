# stdlib/csv 分析与优化建议

## 模块职责

`stdlib/csv` 提供 CSV/TSV 文本解析、自动分隔符识别、CSV 序列化和文件读写能力：

- `parse()`：CSV string → rows
- `parseDetailed()`：CSV string → `{data, errors, meta}`
- `parseTsv()`：TSV shortcut
- `parseAuto()`：自动检测 delimiter 后解析
- `stringify()`：Array/Map rows → CSV string
- `parseFile()` / `writeFile()`：同步文件读写

该模块属于数据格式层，但它同时涉及：

- parser 状态机
- runtime `Array/Map/Json` 构造
- shared parser error schema
- JSON stringify 复用
- 同步文件 I/O

后续重构应重点保证 API 返回类型、错误暴露、配置语义和阻塞 I/O 边界一致。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/csv/csv.c` | stdlib binding、stringify、parseFile/writeFile、loader |
| `stdlib/csv/csv_parser.c` | CSV 状态机 parser、动态类型、delimiter 检测 |
| `stdlib/csv/csv_parser.h` | parser config/result/context 定义 |
| `stdlib/csv/csv.h` | 模块 API 注释和 loader 声明 |
| `stdlib/common_parser.h` | error map helper、配置读取 helper |
| `stdlib/common_writer.h` | serializer buffer helper |
| `stdlib/common_io.h` | 同步 file read/write helper |
| `stdlib/json/json.h` | nested container stringify 时复用 JSON C API |
| `tests/regression/10_stdlib/1305_csv_basic.xr` | CSV 基础、quoted、CRLF、config、dynamicTyping、roundtrip 测试 |

## 当前 API

| API | 当前语义 |
|---|---|
| `csv.parse(data, options?)` | 返回 rows；无效输入返回空数组；解析错误不直接暴露 |
| `csv.parseDetailed(data, options?)` | 返回 `{ data, errors, meta }` |
| `csv.parseTsv(data)` | delimiter 强制为 tab |
| `csv.parseAuto(data)` | 从前 4KB 自动选择 delimiter |
| `csv.stringify(data, options?)` | 支持 array rows 和 map rows |
| `csv.parseFile(path, options?)` | 同步读文件后 parse，失败返回空数组 |
| `csv.writeFile(path, data, options?)` | stringify 后同步写文件，返回 bool |

配置项包括：`delimiter`、`quoteChar`、`escapeChar`、`header`、`columns`、`dynamicTyping`、`trimFields`、`skipEmptyLines`、`comments`、`skipRows`、`maxRows`、`relaxQuotes`、`relaxColumns`、`linebreak`、`nullStrings`。

## 当前架构优点

- parser 是单 pass 状态机，能处理 quoted field、escaped quote、CRLF、mixed linebreak。
- 解析时默认 zero-copy field slice，只有 escape 场景使用 temp buffer。
- dynamic typing 提供 int/float/bool/null 自动转换。
- `parseDetailed()` 已提供 errors/meta。
- `common_parser` 统一 error map schema。
- `common_writer` 统一 serializer buffer。
- stringifier 对 nested Array/Map/Json 会输出 JSON 文本，而不是静默空字段。
- `comments` 配置已经复制进 fixed buffer，避免悬垂指针。

## 依赖与架构边界

### 问题 1：parser header 依赖 runtime internal

`csv_parser.h` include：

- `xisolate_internal.h`
- `xarray.h`
- `xmap.h`
- `xjson.h`

parser 结构直接保存 `XrayIsolate*`，解析过程中直接构造 runtime `XrArray/Map/String`。

影响：

- parser 无法作为纯 C CSV parser 复用。
- CSV 状态机与 runtime allocation 强绑定。
- 如果未来需要 streaming parser 或 CLI 工具复用，需要重构。

建议：

- 将 tokenizer/state machine 与 runtime builder 分离。
- parser core 输出 row/field callbacks，stdlib 层负责构造 XrValue。
- 参考 `json` 的 L0 DOM parser 与 runtime bridge 分层。

### 问题 2：`csv.h` loader 和 API 文档不一致

`csv.h` 注释中列出 `parseFile/writeFile` 等 API，但 header 只声明 loader。它 include `xmodule.h/xvm.h`，没有暴露轻量 C parser API。

建议：

- 如果 CSV C API 不打算外部使用，header 应只保留 loader。
- 如果要外部复用 parser，应提供轻量 parser header，避免 VM/module 依赖。

### 问题 3：loader 定义缺少统一可见性修饰

`xr_load_module_csv()` 定义处没有 `XR_FUNC`。

建议和其他 stdlib loader 一起统一修复。

## API 类型与工具链同步

### 问题 4：`parseDetailed()` 返回 `Map`，但 builtin 声明是 `Json`

实现中：

```c
XrMap *result = xr_map_new(...);
return xr_value_from_map(result);
```

builtin 声明：

```text
(data: string, options?: Json): Json
```

影响：

- 静态类型和运行时类型不一致。
- 用户按 Json field 访问可能与 Map 访问规则不同。
- `result["meta"]` 测试可过，但 `result.meta` 不一定符合声明。

建议：

- 统一返回 `Json`，或修改类型声明为 `Map<string, any>` / `any`。
- 数据格式模块的 detailed result 建议统一返回 `Json`。

### 问题 5：header 模式返回 `Array<Map>`，但 parse 声明为 `Array<Array<string>>`

`csv.parse(data, { header: true })` 返回 row map，而 builtin 仍是：

```text
Array<Array<string>>
```

同时 dynamicTyping 也会让值不是 string。

建议：

- 如果类型系统不支持 union，可声明为 `Array<any>`。
- 文档明确：无 header 为 rows array；header 为 object rows；dynamicTyping 改变 value 类型。
- LSP/analyzer 同步。

### 问题 6：LSP CSV 签名缺少 options

`xlsp_stdlib.c` 中 CSV 签名多为 `fn(text: string)`，没有 options 参数；`stringify` 也简化为 `Array`。

建议：

- 从 builtin 声明生成 LSP signature。
- 对 header/dynamicTyping 这种返回类型变化增加文档提示。

## 错误处理语义

### 问题 7：`parse()` 隐藏 errors

`csv_parse()` 始终返回 `parser.result.data`，即使 parser.result.errors 非空也不提示。

影响：

- 字段数量不匹配、非法 quote 等错误可能被用户忽略。
- 用户必须知道要调用 `parseDetailed()` 才能看到错误。

建议：

- 文档明确 `parse()` 是 lenient data-only API。
- 提供 `parseStrict()`，遇到 errors 返回 null/抛错。
- 或 `parse()` 在严重错误时返回 null。

### 问题 8：maxRows 使用 ERROR 状态终止，但 meta.aborted 未设置

`finish_row()` 达到 `maxRows` 时：

```c
meta.truncated = true;
state = CSV_STATE_ERROR;
```

但 `meta.aborted` 从未设置。`CSV_STATE_ERROR` 同时用于错误和正常截断，语义混淆。

建议：

- 增加独立 `CSV_STATE_DONE` 或 `parser->done`。
- maxRows 设置 `truncated=true`，不要进入 error state。
- 真正错误才设置 `aborted=true`。

### 问题 9：字段 mismatch 记录 error 但仍加入 row

当列数不一致且 `relaxColumns=false` 时，parser 添加 error，但仍继续把 row 加入结果。

这是可接受的 lenient 策略，但必须文档化。

建议：

- `parseDetailed()` 暴露 errors。
- `parseStrict()` 或 strict option 控制是否丢弃/中止。

### 问题 10：file API 失败返回空数组/false，缺少错误原因

`parseFile()` 读失败返回空数组，与合法空文件 parse 结果无法区分。

建议：

- 增加 `parseFileDetailed()` 或让 `parseDetailed()` 支持 file path mode。
- 返回 `{data, errors, meta}`，包含 I/O error。
- 或抛标准 I/O 错误。

## CSV 格式语义

### 问题 11：RFC 4180 默认 linebreak 不一致

parser 兼容 `\n`、`\r\n`、`\r`，stringify 默认 `\n`。RFC 4180 标准行尾是 CRLF。

这不是 bug，但文档中如果宣称 RFC 4180，应说明默认是 Unix newline，用户可通过 `linebreak: "\r\n"` 获取 RFC 风格输出。

### 问题 12：custom quote/escape/delimiter 配置缺少完整验证

`xrs_cfg_get_char()` 只取字符，当前看不到对以下冲突的验证：

- delimiter == quoteChar
- delimiter == newline
- quoteChar == `\0`
- escapeChar == delimiter

建议：

- 配置层做合法性校验。
- invalid config 返回 detailed error，不要默默产生怪异解析。

### 问题 13：auto delimiter 只按出现次数选择

`csv_detect_delimiter()` 扫描前 4KB，忽略 quote 内字符，选择出现次数最多的 `, \t ; |`。

风险：

- 自然文本或第一行注释可能误导。
- 不检查每行列数一致性。

建议：

- 按候选 delimiter 计算多行 field count variance。
- 优先选择行间列数稳定的 delimiter。
- 将检测结果和 confidence 写入 meta。

### 问题 14：comment 检测发生在 field parse 后

comment line 被先解析成 row，再检查第一个字段是否以 prefix 开头。

影响：

- quoted `"# not comment"` 是否应被当注释需要明确。
- 带前导空格的 comment 是否生效取决于 `trimFields`。

建议：

- 文档化 comment 识别规则。
- 如需 RFC-like dialect，comment 应只在行首 raw prefix 识别。

## 动态类型语义

### 问题 15：dynamicTyping 的 nullStrings 语义与注释/实现可能不一致

注释说默认 `nullStrings` 是 `"", "null", "NULL"`，但 `csv_config_init()` 中 `null_strings = NULL`。实际 `finish_field()` 在没有 `nullStrings` 时调用 `csv_convert_value()`，其中空串和 `null` 都转 null，且 `TRUE/FALSE/NULL` 也大小写不敏感转换。

影响：

- 文档、注释和实现语义容易漂移。
- `nullStrings: []` 是否能保留空串取决于实现路径；当前有 `ns` 时不匹配会继续 `csv_convert_value()`，空串仍会变 null。

建议：

- 明确 null coercion 规则。
- 如果 `nullStrings: []` 表示禁用 null detection，应绕过 `csv_convert_value()` 的空/null 分支或拆分 number/bool conversion。
- 增加 `nullStrings: []` 回归测试。

### 问题 16：dynamicTyping 接受 `+1`、`.5` 等非 JSON 数字

CSV 自身没有标准类型系统，这可以接受。但需要说明 dynamicTyping 是宽松文本类型推断，不是 JSON number parser。

### 问题 17：NaN/Infinity roundtrip 策略未明确

stringify float 可输出 `inf`、`-inf`、`nan`。dynamicTyping 的 `strtod` fallback 可能读回这些值，取决于 libc 行为。

建议：

- 明确是否支持非有限 float。
- 跨平台测试 `nan/inf`。

## 序列化语义

### 问题 18：Map array stringify 只取第一行 keys

`csv.stringify()` 如果第一行是 Map，会用第一行 keys 作为 headers，后续 Map 中额外 key 会被丢弃；缺失 key 输出 null/空字段。

建议：

- 文档说明 header selection 策略。
- 允许 `columns` 配置指定输出列顺序。
- 默认扫描所有 rows 合并 keys，或提供 strict 模式。

### 问题 19：Map key 顺序可能不稳定

headers 来自 `xr_map_keys()`，如果 Map 是 hash table，顺序不一定是 insertion order。

建议：

- stringifier 支持 `columns` 控制顺序。
- 文档说明默认顺序不稳定。

### 问题 20：nested container 输出 JSON 字符串，但 parse 不会自动转回结构

stringify nested Array/Map/Json 时输出 JSON 文本字段。`csv.parse()` 即使 dynamicTyping=true 也只会转数字/布尔/null，不会 parse JSON object/array。

建议：

- 文档说明 nested container roundtrip 只保留 JSON text，不恢复结构。
- 如需要，增加 `dynamicTyping: "json"` 或 `parseJsonFields` 选项。

## 阻塞 I/O 与大文件

### 问题 21：parseFile/writeFile 是同步全量读写

`parseFile()` 使用 `xrs_file_read_all_sync()`，将整个文件读入内存。`writeFile()` 先 stringify 成完整 string，再写文件。

影响：

- 大文件内存占用高。
- 文件 I/O 会阻塞 coroutine worker。

建议：

- 标注为 blocking API。
- 增加 streaming parser/writer 或 chunked file API。
- 与 `io/fs` 的阻塞策略统一。

### 问题 22：parser 结果全量 materialize

即使 `maxRows` 可以截断，常规 parse 会把所有 rows 全部构造成 runtime arrays/maps。

建议：

- 提供 iterator/callback/streaming API。
- 或允许 row limit + projection。

## 测试覆盖

现有覆盖：

- 基础 parse/header/quoted/escaped quote。
- stringify 基础和 roundtrip。
- TSV、auto delimiter。
- 空数据、空字段、连续 delimiter。
- CRLF 和 mixed linebreak。
- skipRows/maxRows/trimFields/comments/relaxColumns。
- dynamicTyping 常见值。
- semicolon/pipe delimiter。

主要缺口：

1. `parseDetailed().errors` 的具体内容断言。
2. unterminated quote / invalid escape / field mismatch strict 行为。
3. `meta.truncated` 与 `meta.aborted`。
4. `nullStrings: []` 是否保留空串和 literal null。
5. custom `quoteChar/escapeChar/delimiter` 冲突配置。
6. `linebreak: "\r\n"` stringify 输出。
7. Map stringify header order 和 extra/missing keys。
8. nested container stringify + parse roundtrip 语义。
9. `parseFile()` 失败与空文件区分。
10. 大文件 / maxRows 内存与性能。
11. comment raw 行首 vs trim 后行为。
12. non-finite float stringify/parse 跨平台。
13. auto delimiter confidence/误判场景。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | `parseDetailed()` 返回 Map 但声明 Json | 静态类型与运行时不一致 | 改实现或改声明 |
| 高 | header/dynamicTyping 改变 parse 返回类型但声明固定 | analyzer/LSP 误导 | 声明为 `Array<any>` 或拆 API |
| 高 | `parse()` 隐藏解析错误 | 数据错误被静默接受 | 提供 strict parse 或文档化 |
| 高 | `nullStrings: []` 语义疑似不能禁用空/null 转换 | 数据类型不符合用户配置 | 拆分 null coercion 与类型转换 |
| 中 | maxRows 用 ERROR 状态但不设置 aborted | meta 语义混乱 | 增加 done/truncated 状态 |
| 中 | file API 同步全量读写 | 阻塞 worker 且大文件内存高 | 标 blocking，增加 streaming API |
| 中 | auto delimiter 只按频次 | 容易误判 | 基于多行列数稳定性检测 |
| 中 | Map stringify header 顺序不稳定且丢 extra keys | 输出不可预测/数据丢失 | columns 配置和 strict 模式 |
| 中 | config 缺少冲突校验 | 解析语义异常 | validate config 并返回 detailed error |
| 低 | LSP 签名缺 options | IDE 提示不完整 | 从 builtin 生成 LSP |
| 低 | nested container 只变 JSON text | roundtrip 不恢复结构 | 文档化或新增 parseJsonFields |

## 后续实施建议

建议优先处理类型和错误语义：

1. 统一 `parseDetailed()` 返回类型，优先改为 `Json`。
2. 更新 `parse/stringify` 的 builtin/LSP 签名，反映 header/dynamicTyping/options。
3. 增加 `parseStrict()` 或 strict option。
4. 修复 `nullStrings: []` 的禁用语义并补测试。
5. 将 maxRows 截断从 ERROR 状态中拆出。
6. 增加 config validation。
7. 为 Map stringify 增加 `columns` 输出顺序支持。
8. 规划 streaming parser/writer，避免大文件全量 materialize。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
