# stdlib/xml 分析与优化建议

## 模块职责

`stdlib/xml` 提供 XML 文本解析、XML node Map 表示、XML 序列化、文件读写和节点构造辅助函数：

- `xml.parse(data, options?)`：XML string → node Map
- `xml.parseDetailed(data)`：XML string → `{ doc, errors }`
- `xml.parseFile(path, options?)`：同步读文件后 parse
- `xml.stringify(node, options?)`：node Map → XML string
- `xml.writeFile(path, node, options?)`：同步写文件
- `xml.document()` / `xml.element()` / `xml.text()` / `xml.comment()` / `xml.cdata()`：构造 node Map

当前实现是 `src/base/xxml` 的纯 C DOM parser 加 `stdlib/xml/xml.c` 的 runtime Map bridge。总体方向比 YAML/CSV 的 runtime-coupled parser 更清晰，但 API 声明、错误模型和安全边界仍需统一。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/xml/xml.h` | 模块 API 注释与 loader 声明 |
| `stdlib/xml/xml.c` | stdlib binding、DOM→Map bridge、Map→XML serializer、node builders |
| `src/base/xxml.h` | 纯 C XML DOM、parser config、error/result 类型 |
| `src/base/xxml.c` | XML state-machine parser、entity decode、DOM lifecycle |
| `stdlib/common_parser.h` | parseDetailed error map helper |
| `stdlib/common_writer.h` | serializer buffer helper |
| `stdlib/common_io.h` | sync file read/write helper |
| `stdlib/stdlib_cache.h` | per-isolate XML intern key cache |
| `tests/regression/10_stdlib/1308_xml_basic.xr` | XML 基础 parse/stringify/node 构造回归 |
| `tests/regression/10_stdlib/1309_xml_advanced.xr` | entity、CDATA、PI、DOCTYPE、mixed content、roundtrip 回归 |
| `tests/unit/stdlib/test_xml_node.c` | base XML node 构造、append、attr、cleanup 单元测试 |

## 当前 API

| API | 当前语义 |
|---|---|
| `parse(data, options?)` | 返回 root element 的 Map；失败返回 null |
| `parseDetailed(data)` | 返回 Map：`{ doc, errors }` |
| `parseFile(path, options?)` | 同步读文件，失败返回 null |
| `stringify(node, options?)` | 输入必须是 Map，否则返回空字符串 |
| `writeFile(path, node, options?)` | stringify 后同步写文件，返回 bool |
| `document()` | 构造 `{type:"document", children:[]}` |
| `element(tag, attrs?)` | 构造 element node Map |
| `text(content)` | 构造 text node Map |
| `comment(content)` | 构造 comment node Map |
| `cdata(content)` | 构造 cdata node Map |

parse config：

- `preserveWhitespace`
- `preserveComments`
- `preserveCData`
- `validateEntities`

write config：

- `indent`
- `declaration`
- `encoding`

## Runtime 表示

元素节点 Map：

```text
{
  type: "element",
  tag: "root",
  attrs: Map<string, string>,
  namespaces?: Map<string, string>,
  children: Array<Map>
}
```

文本/注释/CDATA 节点 Map：

```text
{
  type: "text" | "comment" | "cdata",
  text: string
}
```

当前 `parse()` 返回 root element，不返回 document wrapper；但 `document()` 构造的是 document node，serializer 目前没有完整处理 document node。

## 当前架构优点

- `src/base/xxml` 是纯 C DOM parser，没有 runtime/GC 依赖。
- parse result 有 C error array，`parseDetailed()` 能桥接为 runtime error maps。
- DOM node 保存显式 length，文本、属性、CDATA 可以承载 embedded NUL 风险较低。
- stdlib bridge 用 per-isolate cache 缓存常用 key string，降低重复 interning 成本。
- entity decode 支持 XML predefined entities、部分常见 HTML entities、numeric decimal/hex unicode。
- `preserveComments/preserveCData/preserveWhitespace` 提供实用控制。
- parser 与 serializer 均有 depth guard。

## 支持范围

当前 parser 支持：

- XML declaration / processing instruction 跳过或作为 PI node
- element start/end tag
- self-closing tag
- attributes with single/double quotes
- text node
- comment
- CDATA
- DOCTYPE 跳过，包括内部 subset 的 bracket depth
- predefined entities：`lt/gt/amp/quot/apos`
- numeric entities：decimal 和 hex
- 部分 HTML-like named entities
- basic namespace declaration extraction：`xmlns` / `xmlns:prefix`

不完整或未严格支持：

- XML namespace QName 解析与 namespace URI 绑定
- XML Name grammar 完整规则
- DTD/entity declaration 解析
- external entity resolution
- validation
- processing instruction 保留配置
- XML declaration version/encoding/standalone 解析
- multiple root element strict rejection
- unclosed root/end-of-input strict detection

## 依赖与架构边界

### 问题 1：stdlib 返回 Map，但 builtin/LSP 声明为 `Json?`

builtin：

```text
parse(data: string, options?: Json): Json?
parseDetailed(data: string): Json?
stringify(node: Json, options?: Json): string
```

实现实际返回 `XrMap`，并且 tests 使用 `doc["tag"]` 访问。

影响：

- 静态类型与运行时不一致。
- LSP 提示用户以 Json 使用，但实际是 Map node model。
- 与 TOML/YAML 的同类问题一致。

建议：

- 若标准化为 `Json`，bridge 应构造 `XrJson`。
- 若保留 Map，签名改为 `Map<string, any>` / `any`。
- data format 模块统一对象表示策略。

### 问题 2：`parseDetailed()` 不接收 options，但实现可接收

`xml_parse_detailed()` 实现中如果 `argc >= 2` 会读取 config；builtin 声明却是：

```text
(data: string): Json?
```

建议：

- 更新声明为 `(data: string, options?: Json): ...`。
- LSP 同步 options 参数。

### 问题 3：serializer 对 Json 输入不支持

签名声称 `node: Json`，但实现要求：

```c
if (argc < 1 || !XR_IS_MAP(args[0])) return "";
```

建议：

- 声明改为 Map node。
- 或 serializer 支持 Json node object。

### 问题 4：`document()` 构造的 document node 不能被 serializer 完整输出

`serialize_map_node()` 只处理 element/text/comment/cdata。`document()` 返回 `{type:"document", children:[]}`，但 serializer 没有 document 分支。

建议：

- serializer 支持 document node：输出 declaration + root children。
- 或移除/弱化 document builder，明确 stringify 只接受 element node。

## Parser 正确性与安全

### 问题 5：parse 错误不影响 `parse()` 返回 root

底层 parser 记录 errors，但 `xml.parse()` 只要有 `result.doc->root` 就返回 node Map。

影响：

- mismatched tag、invalid entity 等错误可能被用户忽略。
- 用户必须知道调用 `parseDetailed()` 才能看到 errors。

建议：

- 文档明确 `parse()` 是 lenient data-only API。
- 增加 `parseStrict()`：errors 非空时返回 null/抛错。
- 或让 `parse()` 对 fatal error 返回 null。

### 问题 6：unclosed element EOF 检测不足

在主循环结束后只调用 `flush_text()`，没有显式检查 `p->current != NULL` 或 `depth > 0` 并记录 unclosed tag。

影响：

- `<root><a>` 可能返回部分 DOM 且 errors 为空或不完整。

建议：

- EOF 后若 depth > 0，记录 `UnclosedTag`。
- `parseStrict()` 应拒绝。

### 问题 7：multiple root element 未严格拒绝

当 `p->current == NULL` 时新元素会设置 `p->doc->root = elem`。如果输入有多个 top-level elements，后一个 root 可能覆盖前一个 root。

建议：

- root 已存在且遇到第二个 top-level element 时记录 `MultipleRoots`。
- strict parse 返回错误。

### 问题 8：mismatched closing tag 后仍继续 close 当前元素

`XXML_STATE_END_TAG` 中检测不匹配后记录 error，但仍调用 `parser_close_element()`。

这便于 recovery，但会产生修复过的 DOM。需要文档化 lenient 行为，并在 strict 模式下 fail。

### 问题 9：entity validation 只记录错误，不拒绝解析

`validateEntities=true` 时 unknown entity 会记录 `InvalidEntity`，但原文本保留并继续解析。

建议：

- `parse()` 可继续 lenient。
- strict API 应 fail。
- `parseDetailed()` 应测试 unknown entity errors。

### 问题 10：DOCTYPE 跳过策略需要安全声明

当前 DOCTYPE 被跳过，不解析 entity declaration，也不加载 external subset。这避免 XXE，但也意味着 DTD-defined entity 不可用。

建议：

- 明确 XML parser 不解析 DTD、不解析外部实体。
- 对 `<!ENTITY ...>` 内部 entity 不支持，unknown entity 应报错。
- 增加 XXE regression：确保不发起文件/网络访问。

### 问题 11：XML depth limit 数值与注释不一致

`xxml.c` 中：

```c
#define XR_XML_MAX_DEPTH 64
```

注释写 “matches the stdlib-wide cap”，但 `XR_STDLIB_MAX_DEPTH` 当前是 256。

stdlib bridge 的 `NODE_TO_MAP_MAX_DEPTH` 和 serializer `SERIALIZE_MAX_DEPTH` 使用 256。

建议：

- 统一 parse/bridge/stringify depth limit。
- 如果 XML 特意更低，应改注释并暴露配置。

### 问题 12：XML Name grammar 不完整

tag start 只允许 `isalpha` 或 `_`，后续允许 alnum `_ - : .`；attribute name 类似，但不允许 `.`。XML Name 规则更复杂，支持 Unicode name chars。

建议：

- 文档声明 ASCII subset。
- 或实现 XML Name grammar。

## Namespace 语义

### 问题 13：namespaces 只是拆出 xmlns 声明，不做 namespace resolution

`node_to_map_r()` 将 `xmlns` / `xmlns:prefix` 放入 `namespaces` Map，但：

- `tag` 仍是原始 QName。
- attr key 仍保留 `prefix:name`。
- 不继承父级 namespace。
- 不提供 `{localName, prefix, namespaceURI}`。

建议：

- 文档明确当前只是 declaration extraction。
- 如果要 namespace-aware API，增加字段：`prefix/localName/nsURI`。
- serializer 应避免重复/丢失 namespace 声明。

## Serializer 语义

### 问题 14：serializer 只支持 Map node model

`stringify()` 不支持 `Json`，也不支持直接序列化 `document`。

建议：

- 签名和实现统一。
- 对 unsupported node type 返回错误或空 string，并在 detailed stringify 暴露错误。

### 问题 15：CDATA 内容未校验 `]]>`

`serialize_map_node()` 对 cdata 直接输出：

```xml
<![CDATA[...]]>
```

如果文本包含 `]]>`，会生成非法 XML。

建议：

- 拆分 CDATA：`]]]]><![CDATA[>`。
- 或 strict mode 报错。
- 增加测试。

### 问题 16：comment 内容未校验 `--` 和结尾 `-`

XML comment 不允许包含 `--`，也不应以 `-` 结尾。当前直接输出。

建议：

- strict stringify 检查 comment text。
- 默认模式可替换或报错。

### 问题 17：tag/attr name 未校验

`xml.element(tag)` 接受任意 string，serializer 直接输出 tag；attrs key 也直接输出。

影响：

- 用户可构造非法或注入式 XML。

建议：

- builder 或 serializer 校验 XML name。
- invalid name 返回 null/error。

### 问题 18：attribute order 不稳定

attrs 使用 Map，serializer 通过 `xr_map_keys()` 输出。顺序可能不稳定。

建议：

- 文档说明属性顺序不稳定。
- 如果需要稳定输出，提供 sorted option。

### 问题 19：mixed content pretty print 可能改变 whitespace

当 `indent > 0`，serializer 对多子节点插入换行/缩进。对于 mixed content，pretty print 可能改变文本语义。

建议：

- mixed content 默认 compact 或 preserve mode。
- 检测 text+element 混合时不插入格式化 whitespace。

## 文件 I/O 与错误模型

### 问题 20：parseFile/writeFile 是同步全量 I/O

`parseFile()` 读完整文件到内存；`writeFile()` 先完整 stringify 再写。

影响：

- 大 XML 文件内存占用高。
- 同步 I/O 阻塞 coroutine worker。

建议：

- 标注为 blocking API。
- 增加 streaming parser/SAX API 或 chunked writer。
- file API 提供 detailed error，不把 read failure 与 parse failure 混淆。

### 问题 21：parseFile 没有 detailed variant

文件读取失败和 XML parse failure 都可能只返回 null。

建议：

- `parseFileDetailed(path, options?)` 返回 `{doc, errors}`，含 I/O error。
- 或让 `parseDetailed()` 支持 path mode。

## 测试覆盖

现有覆盖：

- 基础 element/children/text/attrs。
- deep nesting、multiple children、empty element。
- preserve comments。
- compact stringify。
- named/numeric/hex/unicode entity decode。
- attribute entity decode。
- CDATA preserve/as text。
- self-closing attributes。
- processing instruction skip。
- node builders。
- stringify roundtrip。
- parseDetailed valid path。
- whitespace default strip/preserve。
- DOCTYPE skip 和 internal subset skip。
- text escaping。
- mixed content parse。
- base XML node 单元测试。

主要缺口：

1. `parseDetailed()` 对 invalid XML 的 errors 内容断言。
2. unclosed tag EOF。
3. multiple roots。
4. mismatched tag strict/lenient 行为。
5. unknown entity with `validateEntities=true/false`。
6. DOCTYPE/XXE 安全测试。
7. namespace resolution 或 declaration extraction 测试。
8. `document()` stringify。
9. CDATA 中 `]]>` stringify。
10. comment 中 `--` stringify。
11. invalid tag/attr name builder/stringify。
12. pretty print mixed content whitespace。
13. attr order stability。
14. file read/write error detail。
15. XML depth limit边界。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | 声明 `Json?` 但实现返回 Map | 类型系统与运行时不一致 | 统一对象表示或修正签名 |
| 高 | `parse()` 忽略 errors 返回部分 DOM | 错误 XML 被静默接受 | 增加 strict parse / 文档化 lenient |
| 高 | EOF 未检查 unclosed tag | 非法 XML 可无错误返回 | EOF depth check |
| 高 | multiple roots 可能覆盖 root | 数据丢失/错误 DOM | 检测 MultipleRoots |
| 中 | XML depth limit 64 与 stdlib cap 注释不一致 | 安全边界混乱 | 统一或明确配置 |
| 中 | namespace 只是拆声明不解析 | 用户误解 namespace 支持 | 文档化或实现 ns-aware model |
| 中 | document node 无法 stringify | builder/API 不闭环 | serializer 支持 document |
| 中 | CDATA/comment/tag/attr stringify 未校验 | 可生成非法 XML | strict validation |
| 中 | sync full-file I/O | 阻塞和大文件内存风险 | streaming/yieldable I/O |
| 低 | parseDetailed 签名缺 options | 工具链漂移 | 更新 builtin/LSP |
| 低 | attr order 不稳定 | diff 不稳定 | sorted option |

## 后续实施建议

建议优先处理 API 和错误模型：

1. 统一 XML node 的公开类型声明，修正 `Json?`/Map 漂移。
2. 为 `parse()`/`parseDetailed()`/未来 `parseStrict()` 明确 lenient vs strict 语义。
3. 补 EOF unclosed tag、multiple roots、unknown entity errors。
4. 统一 parse/bridge/stringify depth limit。
5. 支持 document node stringify，或调整 builder API。
6. 增加 serializer validation：CDATA/comment/name。
7. 明确 namespace 支持范围。
8. 标注 file API blocking，并规划 streaming parser/SAX API。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
