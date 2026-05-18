# stdlib/json 分析与优化建议

## 模块职责

`stdlib/json` 提供 Xray 值与 JSON 文本之间的双向转换能力：

- JSON parse：JSON string → Xray value / `Json` object
- JSON stringify：Xray value / `Array` / `Json` / `Map` / instance → JSON string
- JSON validate：快速合法性校验
- JSON type inspection：判断 JSON 文本或 Xray 值的 JSON 类型
- safe parse：返回 `{value, error}`
- object utility：`keys()` / `values()`

该模块同时处在三条边界上：

1. 纯 C JSON DOM parser：`src/base/xjson.*`
2. runtime `XrValue` / `XrJson` object model：`src/runtime/object/xjson.*`
3. stdlib 用户 API：`json.parse/stringify/isValid/typeof/tryParse/keys/values`

后续重构的重点应是让 parser、validator、runtime object bridge 和 serializer 使用单一语义源，避免边界漂移。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/json/json.c` | stdlib API、DOM→runtime bridge、runtime value serializer、validator |
| `stdlib/json/json.h` | loader 与 C API：`xr_json_parse_from_cstr` / `xr_json_stringify_to_cstr` |
| `src/base/xjson.c/h` | L0 纯 C JSON parser / builder / serializer |
| `src/runtime/object/xjson.c/h` | runtime `Json` hidden-class object |
| `stdlib/common_writer.h` | serializer buffer / indentation helper |
| `stdlib/common_parser.h` | shared depth limit、error-map helper |
| `tests/regression/10_stdlib/1000..1009_json*.xr` | stdlib JSON parser/stringifier/错误/roundtrip/fuzz/number 回归 |
| `tests/regression/13_types/1301..1310_json*.xr` | runtime Json 类型系统、overflow、动态访问等回归 |
| `tests/jit/018..020_json*.xr` | JIT Json 基础和字段访问回归 |

## 当前 API

| API | 当前语义 |
|---|---|
| `json.parse(s)` | 解析 JSON string，失败返回 null |
| `json.stringify(value, indent?)` | 序列化 Xray 值，indent clamp 到 0..8 |
| `json.isValid(s)` | 使用轻量 validator 判断是否合法 JSON |
| `json.typeof(value)` | 对 JSON 文本或 Xray 值返回 JSON 类型名 |
| `json.tryParse(s)` | 返回 `Json { value, error }` |
| `json.keys(obj)` | 返回 `Json/Map/Instance` 的 keys |
| `json.values(obj)` | 返回 `Json/Map/Instance` 的 values |
| `xr_json_parse_from_cstr(X, json, len)` | C API：JSON bytes → XrValue |
| `xr_json_stringify_to_cstr(X, value, out_len)` | C API：XrValue → `xr_malloc` buffer |

## 当前架构优点

- parse 逻辑已经委托给 `src/base/xjson`，避免 stdlib 内重复完整 parser。
- `xjson_parse()` 是 length-aware API，能够处理非 NUL-terminated 输入。
- `src/base/xjson` 已实现严格 JSON number、Unicode surrogate pair、trailing content 检查和 depth limit。
- stdlib serializer 已复用 `common_writer`，避免手写 buffer growth。
- JSON object 解析后映射到 runtime `XrJson`，可使用 dot field 访问。
- 数字解析保留 int64 路径，避免所有 JSON integer 都退化为 double。

## 依赖与架构边界

### 问题 1：stdlib/json 直接依赖 runtime internal

`json.c` include 并使用：

- `xisolate_internal.h`
- `xsymbol_table.h`
- runtime `XrJson` shape
- runtime `XrMap` node layout
- class/instance field layout

`stringify_json()`、`json_keys()`、`json_values()` 直接访问 `X->symbol_table`、`XrShape`、`XrMapNode`。

影响：

- stdlib/json 与 runtime object 内部实现强耦合。
- Json shape 或 Map storage 重构会影响 JSON stdlib。
- 其他 data format 模块也可能复制类似内部访问模式。

建议：

- runtime 提供稳定 object iteration API：`Json` fields、`Map` entries、instance fields。
- stdlib serializer 只依赖 iteration API，不直接读 shape/symbol table/node。
- `json.keys/values` 可复用 runtime `Json.keys/Json.values` 静态方法的底层实现。

### 问题 2：`json.h` 同时暴露 loader 和 C API

`json.h` include `xmodule.h` 和 `xvm.h`，但 `xr_json_parse_from_cstr()` / `xr_json_stringify_to_cstr()` 被 `http`、`csv` 等模块复用。

建议：

- 拆分轻量 C API header，减少对 module/vm 的传播。
- loader 声明放入统一 loader header 或模块私有 header。

### 问题 3：parser/validator 有两套实现

`json.parse()` 调用 `xjson_parse()`；`json.isValid()` 和 `json.typeof()` 使用 `stdlib/json/json.c` 内的 `JsonValidator`。

风险：

- validator 使用 NUL-terminated `char *` 语义，不使用 `XrString.length`。
- parser 使用 `xjson_parse(str->data, str->length)`。
- 两套 number/string/depth/whitespace 规则可能逐渐漂移。

建议：

- `isValid()` 直接调用 `xjson_parse()` 并立即释放 DOM，或在 `src/base/xjson` 提供 zero-DOM validator。
- `typeOf()` 使用同一个 parser/validator source。
- 所有 JSON 合法性语义只保留一套实现。

## JSON 字符串与二进制边界

### 问题 4：JSON 字符串包含 `\u0000` 时会被截断

`src/base/xjson` 的 `parse_string_content()` 会把 `\u0000` 解码成 `NUL` byte，并返回 NUL-terminated `char *`。

但 `dom_to_xrvalue()` 中：

```c
size_t len = strlen(v->as.string);
XrString *str = xr_string_intern(X, v->as.string, len, 0);
```

影响：

- `json.parse("\"a\\u0000b\"")` 可能变成长度 1 的字符串 `"a"`。
- object key 中的 NUL 也可能在 `xjson_object_set()` / symbol interning 时截断。
- JSON roundtrip 不满足 RFC 8259 字符串语义。

建议：

- `XrJsonValue` 的 string/member key 必须保存 byte length。
- parser string builder 返回 `{data, len}` 而非 C string。
- runtime string interning 使用显式长度。
- 增加 `\u0000`、embedded NUL key/value 测试。

### 问题 5：validator 使用 NUL termination，也会误判 embedded NUL

`JsonValidator` 用 `*v->ptr` 判断结束，不使用 `XrString.length`。如果 Xray string 可携带 NUL byte，则 `isValid()` / `typeof()` 会在 NUL 处提前停止。

建议：

- validator 改为 `{ptr, end}`。
- 不要对 runtime string 假设 NUL-terminated。

### 问题 6：stringify 不验证 UTF-8

`stringify_string()` 只转义控制字符、`"`、`\`，其他 bytes 原样输出。JSON 字符串按 RFC 8259 应为 Unicode 字符序列，实际编码通常要求 UTF-8。

影响：

- 如果 Xray string 可以包含任意 bytes，`json.stringify()` 可输出非法 UTF-8 JSON。

建议：

- 明确 Xray string 是否是 UTF-8 文本。
- 若 string 允许 bytes，JSON stringify 应 validate UTF-8，非法序列转 `U+FFFD` 或报错。
- 与 `encoding`、`base64` 中的 string/Bytes 边界一起统一。

## 数字语义

### 问题 7：整数/浮点启发式需要文档化为非 JSON 标准语义

JSON number 没有 int/float 区分。当前策略：

- 无小数/指数且 `strtoll` 不溢出 → `int`
- 其他 → `float`

这对 Xray 很实用，但需要文档化。尤其：

- `1` 是 int
- `1.0` 是 float
- `1e0` 是 float
- int64 溢出后变 float

建议：

- 文档明确 number mapping。
- 对超大整数提供可选 decimal/string 模式，避免精度损失。

### 问题 8：`strtod` overflow 接受 Infinity

测试中 `1e309` 当前期望解析为 Infinity。严格 JSON 语法允许 number token，但 JSON 语义不定义 IEEE Infinity；很多 JSON 库会把数值范围错误作为 parse error 或保存为 decimal token。

建议：

- 决定是否接受 overflow-to-infinity。
- 如果 stringify 已把 NaN/Infinity 输出为 null，parse 接受 Infinity 会造成不对称。
- 推荐默认 strict parse 拒绝 non-finite number，或文档化当前行为。

## 对象语义

### 问题 9：duplicate key 策略为 last wins，但未暴露

`xjson_object_set()` 对重复 key 做覆盖，释放旧 value。因此 `{ "a": 1, "a": 2 }` 结果为 `a=2`。

建议：

- 文档明确 duplicate key 处理策略。
- `tryParse` 可选返回 warning 或 strict duplicate-key error。

### 问题 10：`Json` object field count 使用 `uint16_t`，超大 object 风险

`dom_to_xrvalue()` 中：

```c
uint16_t cap = v->as.object.count > 4 ? (uint16_t) v->as.object.count : 4;
```

如果 JSON object field count 超过 `UINT16_MAX`，会截断 capacity。runtime `XrJson` shape 也使用 `uint16_t field_count`。

建议：

- parser bridge 检查 object field count 上限。
- 超出 runtime `Json` 能力时返回 error，不要截断。
- 大对象可以考虑映射为 `Map` 或提供 streaming API。

### 问题 11：Map stringify 跳过部分 key 类型

`stringify_map()` 只输出 string 和 int key，跳过 float/array/map/instance key。这个策略比输出占位 key 更安全，但用户可能不知道数据被丢弃。

建议：

- 文档化 Map key policy。
- 提供 strict stringify 模式：遇到不可表示 key 返回错误。

### 问题 12：Instance stringify 可能暴露私有字段或内部字段

当前代码注释里提到 private field 可跳过，但实际没有跳过 private field，只跳过 static field。

建议：

- 明确 class/struct instance JSON 序列化策略。
- 默认跳过 private/internal field。
- 或只序列化显式标注字段。

## 序列化递归与循环

### 问题 13：depth guard 不是循环检测

`stringify_value()` 用 `w->depth >= JSON_MAX_DEPTH` 输出 null。它能防止无限递归最终栈溢出，但不能准确报告循环结构。

影响：

- 循环 Array/Map/Json 会输出深层 null，而不是错误。
- 用户难以定位问题。

建议：

- 增加 visited set 检测 object identity。
- strict 模式下遇到循环返回错误。
- 默认模式可输出 null，但应文档化。

### 问题 14：parse 和 stringify depth limit 使用同名值但语义不同

`xjson_parse()` 使用 `XJSON_MAX_DEPTH = 256`；stdlib serializer 使用 `XR_STDLIB_MAX_DEPTH = 256`。目前数值一致，但来源不同。

建议：

- depth limit 单一配置源。
- `json.parse`、`json.stringify`、`json.isValid` 都对用户暴露一致限制。

## API 命名与声明同步

### 问题 15：header 文档写 `typeOf`，实现导出 `typeof`

`json.h` 的 module exports 注释写：

- `typeOf(value)`

但实际 builtin/export 是：

- `typeof`

建议：

- 统一命名。考虑避免与语言内置 `typeof` 混淆。
- 如果保留 `typeof`，修正文档和 LSP/知识库。
- 如果改为 `typeOf`，保留 alias 或一次性迁移。

### 问题 16：`isValid` 实现支持 opts，但 builtin 类型未声明

`json_is_valid()` 支持第二参数：bool 或 Json `{strict}`。但 builtin 声明为：

```text
(s: string): bool
```

影响：

- analyzer/LSP 可能不允许第二参数。
- 用户无法发现 strict 模式。

建议：

- 更新 builtin 类型声明。
- 或移除隐藏参数，改显式 `isValidStrict()`。

### 问题 17：`tryParse` 错误信息过粗

`tryParse()` 失败时只返回 `"Invalid JSON"`，没有 line/column、error type、附近 token。

建议：

- `src/base/xjson` parser 返回错误位置与错误码。
- `tryParse` 返回结构化 error：`{ type, line, column, message }`。
- 复用 `common_parser.h` error-map 机制。

## C API 语义

### 问题 18：`xr_json_parse_from_cstr()` 空输入与 JSON null 无法区分

C API 返回 `xr_null()` 表示 parse error；合法 JSON `null` 也返回 `xr_null()`。

脚本层同样有这个问题：`json.parse("null") == null`，`json.parse("invalid") == null`。

建议：

- C API 返回 status + out value。
- 脚本层建议用户使用 `tryParse()` 区分 null 与错误。
- 文档明确 `parse()` 无法区分 JSON null 和 parse error。

### 问题 19：C API stringify 对 OOM 没有显式错误传播

`xr_json_stringify_to_cstr()` 使用 writer 构造后 steal。底层 `XrCtxBuf` OOM 策略需要和所有 stdlib serializer 统一。如果 OOM abort 是约定，应文档化。

建议：

- C API 文档明确 OOM 策略。
- 如果要可恢复，返回 NULL 并携带 error。

## 测试覆盖

现有覆盖很丰富：

- 基础类型、object、array、nested parse。
- string escape、Unicode、special chars。
- invalid JSON、trailing comma、leading zero、bare word。
- roundtrip。
- fuzz-style nested/width 测试。
- number boundary、int64、大浮点、overflow/underflow。
- runtime `Json` 类型、overflow property、动态访问、computed property、for-in、type coercion。
- JIT Json 基础和字段写入。

主要缺口：

1. `\u0000` 和 embedded NUL string/key。
2. `isValid()` 与 `parse()` 对含 NUL string 的一致性。
3. `isValid(strict)` 第二参数 analyzer 是否接受。
4. `typeof` / `typeOf` 命名一致性。
5. duplicate key 策略。
6. object field count 超过 `uint16_t` 的安全失败。
7. string 非 UTF-8 bytes 的 stringify 行为。
8. circular Array/Map/Json stringify。
9. private field instance stringify。
10. Map 非 string/int key 的丢弃策略。
11. parse error line/column。
12. `parse("null")` 与 parse error 的 API 区分。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 严重 | JSON string embedded NUL 被 `strlen` 截断 | 数据损坏，roundtrip 错误 | DOM string/key 保存长度 |
| 高 | `isValid/typeOf` validator 与 parser 双实现漂移 | 合法性判断不一致 | 统一到 `xjson` validator/parser |
| 高 | C/script `parse()` 用 null 表示错误且 JSON null 也为 null | 无法区分合法 null 和错误 | 使用 status/out 或推荐 tryParse |
| 高 | `isValid` 支持 opts 但类型声明未同步 | analyzer/LSP 漂移 | 更新 builtin 或改显式 API |
| 高 | 直接依赖 runtime internal shape/symbol/map layout | runtime 重构影响 stdlib | 提供统一 iteration API |
| 中 | `typeOf` 文档与 `typeof` 实现不一致 | 用户和工具链困惑 | 统一命名 |
| 中 | 超大 object field count 截断风险 | 内存/shape 行为异常 | bridge 上限检查 |
| 中 | stringify 无 UTF-8 校验 | 可输出非法 JSON 文本 | validate 或明确 string 语义 |
| 中 | duplicate key last wins 未文档化 | 用户可能误解数据保留 | 文档化或 strict duplicate 检查 |
| 中 | stringify 循环结构只靠 depth guard | 输出不透明 null | visited set / strict error |
| 低 | tryParse 错误信息过粗 | 排错困难 | parser 返回 line/column/error code |
| 低 | Map/Instance 序列化策略未明 | 数据可能静默丢失或泄露 | 定义 strict/default 策略 |

## 后续实施建议

建议优先处理数据正确性与语义单一化：

1. 修改 `src/base/xjson` DOM string/member key 表示，保存显式长度。
2. 让 `isValid()` / `typeof()` 复用 `xjson` 的同一套 length-aware validator。
3. 同步 `isValid` strict 参数和 `typeof/typeOf` 命名。
4. 为 C API 增加 status/out 形式，脚本文档强调 `tryParse()`。
5. 为 runtime object traversal 增加统一 iterator API。
6. 增加 stringify cycle detection 和 strict mode。
7. 明确 number、duplicate key、Map key、Instance field 序列化策略。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
