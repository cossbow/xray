# stdlib/yaml 分析与优化建议

## 模块职责

`stdlib/yaml` 提供 YAML 文本与 Xray 值之间的转换：

- 单文档解析：`parse()`
- 带 errors/meta 的解析：`parseStrict()`
- 多文档解析：`parseAll()`
- 序列化：`stringify()`
- 文件读写：`parseFile()` / `writeFile()`

当前实现目标是“YAML 1.2 常用子集”，而不是完整 YAML 规范实现。它支持 block/flow sequence、block/flow mapping、quoted string、block scalar、简单 anchor/alias、部分 scalar type 推断和 emitter。

后续重构应明确 YAML 支持范围，避免用户误以为这是完整 YAML 1.2 parser。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/yaml/yaml.h` | C API、loader、模块文档 |
| `stdlib/yaml/yaml.c` | stdlib binding、parse/stringify/file API |
| `stdlib/yaml/yaml_parser.h` | parser config/result/context/anchor 表 |
| `stdlib/yaml/yaml_parser.c` | 手写 YAML 子集 parser |
| `stdlib/yaml/yaml_emitter.c` | YAML emitter |
| `stdlib/yaml/yaml_scanner.c` | scanner 辅助实现 |
| `stdlib/common_parser.h` | shared depth limit 和 error map helper |
| `stdlib/common_writer.h` | emitter buffer helper |
| `stdlib/common_io.h` | 同步 file read/write helper |
| `tests/regression/10_stdlib/1302_yaml_basic.xr` | YAML 基础测试 |
| `tests/regression/10_stdlib/1304_yaml_advanced.xr` | YAML block scalar、多文档、hex/octal、特殊 float 测试 |
| `tests/regression/10_stdlib/1008_json_yaml_fuzz.xr` | JSON/YAML fuzz-style 回归 |

## 当前 API

| API | 当前语义 |
|---|---|
| `yaml.parse(data, config?)` | 解析单文档 YAML，失败或空输入返回 null |
| `yaml.parseStrict(data)` | 返回 `{data, errors, meta}` |
| `yaml.parseAll(data)` | 返回 Array，每个元素为文档 |
| `yaml.stringify(value, config?)` | 序列化为 YAML，支持 indent/flowLevel/lineWidth 参数 |
| `yaml.parseFile(path)` | 同步读文件后 parse |
| `yaml.writeFile(path, value)` | stringify 后同步写文件 |
| `xr_yaml_parse()` | C API：单文档 parse |
| `xr_yaml_parse_all()` | C API：多文档 parse |
| `xr_yaml_stringify()` | C API：serialize |

## 当前支持的 YAML 子集

解析侧目前支持：

- plain scalar
- single-quoted scalar
- double-quoted scalar 和常见 escape
- literal block scalar：`|`，支持 chomping
- folded block scalar：`>`，支持 chomping
- flow sequence：`[a, b]`
- flow mapping：`{a: 1, b: 2}`
- block sequence：`- item`
- block mapping：`key: value`
- simple explicit key：`? key`
- anchor：`&name value`
- alias：`*name`
- document start：`---`
- document end：`...`
- scalar typing：null/bool/int/float/hex/octal/.inf/.nan

不明显支持或支持有限：

- tags / custom tags
- merge key `<<`
- full YAML directives `%YAML`
- complex multi-line quoted scalar folding
- advanced flow syntax and trailing comma rules
- comments attached to values
- full duplicate key policy in flow mapping
- anchor graph deep copy / alias identity semantics

## 当前架构优点

- parser 是 length-aware，使用 `{ptr, end}`，比 NUL-terminated parser 安全。
- depth limit 与 stdlib-wide `XR_STDLIB_MAX_DEPTH` 对齐。
- parser errors 使用 shared `xrs_error_push()` schema。
- anchor table 已从固定 64 条改为动态增长。
- emitter 复用 `common_writer`，避免重复 buffer 实现。
- emitter 会主动 quote 可能被 YAML 1.1/1.2 解析成 bool/null/number 的字符串，提高 roundtrip 稳定性。
- emitter 对递归深度有 guard，避免循环结构直接栈溢出。

## 依赖与架构边界

### 问题 1：parser 直接构造 runtime object

`yaml_parser.h` include runtime internal，并且 parser 中直接创建：

- `XrArray`
- `XrMap`
- `XrString`
- `XrValue`

影响：

- parser core 不能作为 L0/L1 纯 parser 复用。
- YAML 状态机和 runtime allocator/GC 强耦合。
- 后续如果需要 CLI、LSP、package metadata 使用 YAML，需要引入 runtime。

建议：

- 抽出 L0 YAML event/parser 或 DOM。
- stdlib 层只负责 DOM/event → runtime value bridge。
- 与 `json` 的 base parser 分层保持一致。

### 问题 2：emitter 直接读取 runtime internals

`yaml_emitter.c` 直接读取：

- `X->symbol_table`
- `XrShape`
- `XrMapNode`
- `xr_map_sizenode()` / `xr_map_node()`

影响与 `json` 类似：runtime object storage 改动会影响 YAML。

建议：

- 引入 runtime object iterator API。
- data format emitters 统一使用 iterator，而不是读内部结构。

### 问题 3：`yaml.h` 同时暴露 C API 和 loader

`yaml.h` include `xmodule.h` / `xvm.h`，但其中 `xr_yaml_parse/stringify` 可能被其他 C 模块复用。

建议：

- 拆分轻量 C API header 和 module loader header。
- C API 只依赖 `XrValue` 和 `XrayIsolate` forward declaration。

## API 类型与工具链同步

### 问题 4：`parseStrict()` 名称误导

`yaml_parse_strict()` 设置 `config.safe = true`，调用 `yaml_parser_parse_strict()`。但 `yaml_parser_parse_strict()` 只是：

```c
parser->result.data = yaml_parser_parse(parser);
return parser->result;
```

它并不会在 errors 非空时拒绝输出，也不会比 parse 多做结构校验。

建议：

- 改名为 `parseDetailed()`，或真正实现 strict：errors 非空时 data=null/返回失败。
- 与 CSV/XML/TOML 的 detailed/strict 命名统一。

### 问题 5：返回类型声明为 Json，但实现多为 Map

builtin：

```text
parse(data: string): Json?
parseStrict(data: string): Json?
parseAll(data: string): Array<Json>
```

但 parser 实际对 mapping 返回 `XrMap`，不是 runtime `XrJson`。

影响：

- 静态类型与运行时不一致。
- 用户可能按 Json dot access 使用时遇到不一致。

建议：

- 统一返回 `Json`，或把声明改成 `Map<string, any>?` / `any`。
- data format 模块建议统一对象表示，避免 JSON/YAML/TOML/XML 混用 Map/Json。

### 问题 6：`stringify(value, config?)` 声明过窄

builtin 声明是：

```text
(value: Json): string
```

实际支持 `null/bool/int/float/string/Array/Map/Json`，第二参数也支持 int 或 Json options。

建议：

- 声明改为 `(value: any, options?: Json|int): string` 的等价形式。
- 如果类型系统不支持 union，可用 `(value: any, options?: any): string`。

### 问题 7：`lineWidth` 参数存在但未实现

`yaml.h` 和 `yaml_emitter.c` 明确说明 `lineWidth` reserved，不会 wrap。

建议：

- 文档标注 `lineWidth` 当前无效果。
- 或暂时不暴露该选项，避免用户误用。

## Parser 语义问题

### 问题 8：`parseAll()` 多文档分割不完整

`parseAll()` 循环调用 `parse_document()`，只特殊跳过 `...`。`parse_document()` 能跳过开头 `---`，但解析第一个文档时并不会在下一个 `---` 前停止；它主要按当前 value 的 parser 消耗输入。

测试也只是 `docs.length >= 1`，没有严格断言三文档。

建议：

- 明确实现 document boundary scanner。
- `parse_document()` 应在 `---` / `...` boundary 停止。
- 测试 `---\na:1\n---\nb:2` 应返回两个文档。

### 问题 9：flow mapping duplicate key 不检查

`parse_flow_mapping()` 里有注释说 duplicate key check，但实际是 `(void)0`，直接 `xr_map_set()` 覆盖。

block mapping 已检查 duplicate key 并记录 error。

建议：

- flow/block mapping duplicate-key 策略一致。
- `allowDuplicateKeys=false` 时都记录 error 或拒绝覆盖。

### 问题 10：safe 配置未真正影响 parser

`YamlConfig.safe` 默认 true，`parseStrict()` 也设置 true。但当前 parser 看不到 tag 或危险构造处理，因此 safe 实际没有明显行为。

建议：

- 如果没有 tags/exec/object construction，移除或文档化 no-op。
- 如果未来支持 tags，safe 必须阻止非 core schema tags。

### 问题 11：alias 未找到时返回 null，无法区分合法 null

`find_anchor()` 找不到返回 `xr_null()`。这和 YAML null 值无法区分。

建议：

- 未定义 alias 应记录 error。
- strict 模式应失败。

### 问题 12：anchor alias 保留 object identity，未深拷贝

`find_anchor()` 返回保存的 `XrValue`。如果 anchor 值是 Array/Map，alias 与 anchor 指向同一 runtime object。

这可能符合 YAML graph 的 identity 语义，但 Xray 用户对解析结果的修改会影响多个位置。

建议：

- 文档明确 alias identity。
- 如果希望 JSON-like tree，应提供 deep-copy alias mode。

### 问题 13：anchor name 被截断到 63 bytes

`save_anchor()` 和 parser stack buffer 都截断 anchor name。

建议：

- 超长 anchor name 应报错或动态分配。
- 不应静默截断导致 name collision。

### 问题 14：double-quoted Unicode escape 校验不足

`parse_double_quoted()` 对 `\x`/`\u`/`\U` 读取 hex digits，但没有严格确认 digit 数量和合法 hex。遇到非法字符时仍会累积当前 cp。

建议：

- 非法 escape 记录 error。
- 严格校验 code point 范围、surrogate、不完整 escape。

### 问题 15：plain scalar number detection 较宽松

当前 number detection 允许字符 `x/X/o/O` 出现在非 hex/octal 上下文。最终 `strtod/strtoll` 通常会失败并回退字符串，但逻辑复杂。

建议：

- 拆分 YAML scalar resolver：bool/null/int/float/hex/octal/special float/string。
- 用明确 grammar 替代宽松字符扫描。

## Emitter 语义问题

### 问题 16：循环结构只靠 depth guard，不是真正 cycle detection

`emit_value()` 在 `level >= XR_STDLIB_MAX_DEPTH` 时输出 `null`。这能防止崩溃，但无法准确报告循环。

建议：

- 增加 visited set 检测 Array/Map/Json identity。
- strict stringify 遇到 cycle 返回错误。
- 默认输出 null 可以保留但必须文档化。

### 问题 17：Map key 支持不完整，且可能输出非法 mapping

`emit_map()` 对非 string/int key 的处理是空输出 key 后仍输出 `": "` 和 value。

影响：

- float/bool/object key 可能生成非法或不可预期 YAML。

建议：

- 明确支持的 key 类型。
- 不支持 key 应报错或 stringify key。
- 不应静默输出空 key。

### 问题 18：emitter 输出顺序对 Map 可能不稳定

Map 遍历基于 hash table node index，默认顺序可能不是插入顺序。

建议：

- 文档说明 Map 输出顺序不稳定。
- 对配置文件场景提供 ordered map 或排序 key 选项。

### 问题 19：block scalar emit 与 parse 的 roundtrip 需要更多测试

emitter 对含 newline 的 string 使用 `|` literal block。parser 支持 literal/folded block 和 chomping，但复杂情况如结尾多 newline、空行、缩进等需要系统测试。

建议：

- 对 multiline string roundtrip 增加表驱动测试。
- 覆盖 clip/strip/keep chomping。

## 文件 I/O 与阻塞

### 问题 20：parseFile/writeFile 是同步全量 I/O

`parseFile()` 使用 `xrs_file_read_all_sync()`，`writeFile()` 先生成完整 YAML string 再写文件。

影响：

- 大文件内存占用高。
- 文件读写阻塞 coroutine worker。

建议：

- 标注为 blocking API。
- 后续迁移到 yieldable I/O 或 streaming parser/emitter。
- 与 CSV/JSON/TOML/XML 的 file API 统一。

### 问题 21：writeFile 忽略 stringify config

`yaml_write_file()` 固定调用：

```c
xr_yaml_stringify(X, args[1], 2)
```

不支持传入 indent/flowLevel/lineWidth options，而 `stringify()` 支持。

建议：

- `writeFile(path, value, options?)` 与 `stringify(value, options?)` 参数一致。
- 更新 builtin/LSP 签名。

## 测试覆盖

现有覆盖：

- 基础 key-value、nested mapping。
- flow sequence/mapping。
- quoted string。
- stringify smoke test。
- literal/folded block scalar。
- parseAll 基础。
- hex/octal number。
- `.inf/.nan`。
- JSON/YAML fuzz-style block/flow mixed、roundtrip、nested sequence/mapping。

主要缺口：

1. `parseAll()` 对多个 `---` 文档的准确分割。
2. `parseStrict()` errors 内容和 strict 行为。
3. block vs flow duplicate key 一致性。
4. undefined alias error。
5. alias identity/deep-copy 语义。
6. anchor name 超长和 collision。
7. invalid escape、incomplete Unicode escape。
8. maxDepth error 和返回值。
9. Map non-string/int key stringify。
10. lineWidth 不生效的文档或测试。
11. writeFile options。
12. file read failure 与 parse failure 区分。
13. multiline string block scalar roundtrip 的 trailing newline/chomping。
14. private/instance/unsupported value stringify 行为。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | 返回类型声明 Json，但实现返回 Map | 静态类型与运行时不一致 | 统一对象表示或修正声明 |
| 高 | `parseStrict()` 名称误导 | 用户以为会拒绝错误输入 | 改名 parseDetailed 或真正 strict |
| 高 | `parseAll()` 文档边界不完整 | 多文档 YAML 解析错误 | 实现 boundary-aware parser |
| 高 | flow duplicate key 不检查 | 与 block mapping 行为不一致 | 统一 duplicate policy |
| 中 | `safe` 配置近似 no-op | 安全语义误导 | 移除或实现 tag 安全策略 |
| 中 | undefined alias 返回 null | 错误与合法 null 混淆 | 记录 error，strict fail |
| 中 | anchor name 静默截断 | name collision / 错误引用 | 动态分配或报错 |
| 中 | Unicode escape 校验不足 | 非法输入被接受 | 严格 escape parser |
| 中 | emitter 对非 string/int map key 不安全 | 生成非法 YAML | key 策略和 strict error |
| 中 | file API 阻塞全量 I/O | 大文件和 coroutine worker 风险 | streaming/yieldable I/O |
| 低 | lineWidth 参数未实现 | 用户配置无效果 | 文档隐藏或实现 wrap |
| 低 | Map 输出顺序不稳定 | 配置 diff 不稳定 | sort/order option |

## 后续实施建议

建议按语义一致性优先：

1. 明确 YAML 模块只支持 common subset，并写入用户文档。
2. 统一 parse/parseStrict/parseAll 返回对象类型，优先返回 `Json` 或修正声明。
3. 重命名 `parseStrict()` 为 `parseDetailed()`，或实现真正 strict。
4. 修复 `parseAll()` 文档边界处理。
5. 统一 flow/block duplicate key 检查。
6. 完善 alias/anchor 错误语义。
7. 为 emitter 增加 key validation 和 cycle detection。
8. 让 `writeFile()` 支持 options，并标注 blocking。
9. 抽象 runtime object iterator，减少 emitter 对 Map/Json internals 的依赖。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
