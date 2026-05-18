# stdlib 序列化 / 数据格式组 源码分析与优化建议

> 范围：`stdlib/json`（1239 行）、`stdlib/yaml`（1885 行，3 个 .c）、
> `stdlib/toml`（1583 行，2 个 .c）、`stdlib/xml`（1766 行，3 个 .c）、
> `stdlib/csv`（1424 行，2 个 .c），共 **7,897 行** C 代码。
>
> 严重程度：🔴 **严重**（违反硬性红线 / 正确性缺陷） · 🟠 **重要**（规范 / 鲁棒性）
> · 🟡 **建议**（可用性 / 设计） · 🟢 **优化**（性能 / 清理）
>
> 本文档的后续 PR 建议与"基础工具组"分析（`docs/analysis/stdlib_basic_tools.md`）
> 协同，推荐先合入 `stdlib/common.h`（PR-1）再统一解决本组问题。

---

## 1 · 总览

### 1.1 架构对比

| 模块 | 入口 .c | 主体拆分 | 设计风格 |
|------|--------|---------|----------|
| `json` | 单文件 | parse + serialize + validator 三合一 | 递归下降 + 栈 depth 计数 |
| `yaml` | `yaml.c` + `yaml_parser.c` + `yaml_emitter.c` + `yaml_scanner.c` | 扫描器 + 状态机 + 单独 emitter | SIMD + SWAR 加速 |
| `toml` | `toml.c` + `toml_parser.c` | 状态机 + writer | 表驱动状态 + SWAR |
| `xml` | `xml.c` + `xml_parser.c` + `xml_node.c` | 状态机 + 中间 XmlNode 树 → 转成 XrMap | 两阶段（C 树 + GC 映射）|
| `csv` | `csv.c` + `csv_parser.c` | 状态机 + writer | SIMD + SWAR，支持 TSV/auto-detect |

### 1.2 做得好的地方

- **JSON**：零拷贝快速路径（无转义直接 intern）、UTF-16 代理对、**independent validator** 无 GC 分配；`count_object_keys` 预扫描 shape capacity。
- **YAML**：SIMD `find_newline/find_any` 加速、block + flow 双风格、anchor/alias、多文档 `parseAll`、literal/folded block scalar。
- **TOML**：状态机完整、SWAR 数字解析、支持 inline table 的 dotted key、hex/oct/bin、array-of-tables、parseStrict 返回错误数组 + meta。
- **XML**：XmlNode 中间树 `xr_malloc` 管理（不压 GC），最后一步转 XrMap；SIMD 文本扫描；完整 entity UTF-8 输出；`XmlKeys` 缓存 intern 过的键。
- **CSV**：状态机严谨（5 个状态），BOM 剥离、auto-detect delim、dynamicTyping、header→Map 映射、relaxQuotes/relaxColumns 容错、comment skip、skipRows/maxRows。

### 1.3 系统性问题（跨模块）

| # | 严重度 | 问题 | 受影响模块 |
|---|-------|------|------------|
| S1 | 🔴 | `malloc/realloc/free` 而非 `xr_malloc/xr_free` | `json`、`yaml`（3 文件全部） |
| S2 | 🔴 | 公共 C API 非 static 却无 `XRAY_API`/`XR_FUNC` | **全部** |
| S3 | 🟠 | 约 5 份重复的 writer/CtxBuf 样板（~500 行） | `json`(`JsonWriter`)、`yaml`(`YamlEmitter`)、`toml`(`TomlWriter`)、`xml`(`XmlWriter`)、`csv`(`CsvWriter`) |
| S4 | 🟠 | `realloc` 失败后仍尝试写入 → UB | `json`(L449-452,L470-473)、`yaml_parser`（6+ 处）、`yaml_emitter`(L58-60)、`toml`/`xml`/`csv`(silent return, 下一次写越界) |
| S5 | 🟠 | 浮点输出 `%.15g`（或 `%g`）无法 IEEE 754 往返，应 `%.17g` | `json`、`yaml`、`toml`、`csv` |
| S6 | 🟠 | 同步 `fopen/fread/fwrite` 阻塞 worker | `yaml`/`toml`/`xml`/`csv` 的 `parseFile/writeFile` |
| S7 | 🟠 | 嵌套深度上限不统一（JSON=512、YAML=64 可配、TOML=**无**、XML=`XML_MAX_DEPTH`） | **全部** |
| S8 | 🟡 | 错误路径反复 `xr_string_intern("type"/"line"/...)` 常量键 | `toml_parser.c:add_error`、`csv_parser.c:add_error`、`xml_parser.c:add_error`、`yaml parseStrict` |
| S9 | 🟡 | 重复键（duplicate keys）都是"静默覆盖"，config 字段 `allow_duplicate_keys` 没实际使用 | `json`、`yaml`、`toml`（部分）|
| S10 | 🟡 | `Map` 键只支持 string / int，其他类型静默忽略 | `json.stringify_map`、`yaml.emit_map`、`toml.write_value`、`csv.write_row` |

### 1.4 推荐总体方向（与基础工具组 PR-1/PR-2 联动）

1. **统一 `stdlib/common_writer.h`**（从本组 5 份 writer 抽出，配合基础工具组的 `CtxBuf`）— 单点修复 OOM、精度、UTF-8 转义。
2. **JSON / YAML `malloc` 全改 `xr_malloc`**，`realloc` 全改 `XR_REALLOC` 中转宏。
3. **`parseFile/writeFile` 统一走 `XrAsyncPool`**（参见基础工具组的 io 建议）。
4. **每模块共享 `parseStrict` error-map builder**：把 7 次 `xr_string_intern("type"/"line"/"column"/"message"/...)` 抽成一个 inline helper。
5. **统一 `XR_STDLIB_MAX_DEPTH`**（建议 256），每个 parser 的 config 里可以覆盖但默认一致。

---

## 2 · 逐模块问题清单

### 2.1 `stdlib/json`（1,239 行）

**🔴 严重**

1. **全程 `malloc`/`realloc`/`free`** — 违反硬红线。位置：
   - `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/json/json.c:107-109`（`STR_ENSURE` 宏扩展）
   - `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/json/json.c:442`（`writer_init`）
   - `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/json/json.c:449-452`（`writer_append` realloc 无失败处理）
   - `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/json/json.c:470-473`（`writer_newline`）
   - `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/json/json.c:785-813`（`xr_json_parse_from_cstr` malloc/free）

2. **`xr_json_stringify_to_cstr` / `xr_json_parse_from_cstr` 未加 `XRAY_API`/`XR_FUNC`** — `.h` 裸声明，违反可见性规范。
   `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/json/json.h:49,52`。

**🟠 重要**

3. **`%.15g` 精度不够**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/json/json.c:691` — `stringify(3.14159265358979)` 会输出 `3.1415926535898`，再 parse 回来丢位。应改 `%.17g`。

4. **`writer_append` / `writer_newline` OOM 后继续写**：两处 `realloc` 结果未检查，内存紧张时变 segfault。换 `XR_REALLOC`。

5. **`count_object_keys` 对残缺输入可能死循环或误算**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/json/json.c:298` 循环条件 `*s != '\0' && !(depth == 0 && *s == '}')`。如果字符串未关闭（`{"a": "x`）并且前面没碰到 `"`，会一直扫到 `\0`。扫描函数仅用于 pre-allocation 容量估计，最终由真正的 `parse_object` 报错，影响有限但值得加显式输入长度参数。

6. **`parse` 错误无法区分 `"null"`**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/json/json.c:732-742` — 任何错误都返回 `xr_null()`，无法与合法 `null` 值区分。建议把错误路径改走 `parser.error` 并通过 VM 错误机制上抛；或显式推荐 `tryParse`。

7. **`stringify_map` 不保证插入顺序**：按哈希表 slot 顺序输出，而 `XrJson` 保留插入顺序。API 不一致会让 Map→JSON→Map 往返时键顺序变化。

8. **`json_type_of` 对字符串输入只看首字符**：`typeOf("truefalse")` 返回 `"boolean"`。应先 `json_is_valid` 或扫到下一个结构边界。`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/json/json.c:977-984`。

**🟡 建议**

9. `JSON_MAX_DEPTH = 512` 与基础工具组 `JsonValidator` 共用，OK；但 YAML/TOML 不同，待统一。
10. 验证器 `validate_string` 允许控制字符（RFC 8259 要求 `0x00–0x1F` 必须转义）：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/json/json.c:834` 的 while 条件只排除 `"`/`\`，没排除 `< 0x20`。strict 模式下应拒绝。
11. 重复键静默覆盖；spec 允许 undefined，但应提供 `strict: true` 开关。
12. 缺流式 API（入/出均一次性整串）。大文档场景待增强。

---

### 2.2 `stdlib/yaml`（1,885 行，3 个 .c）

**🔴 严重**

1. **全程 `malloc/realloc/free`** — 违反硬红线。密集位置：
   - `yaml_parser.c:127,137,165,242,662,723,760`（buffer 扩容）
   - `yaml_emitter.c:44,58-60,77-80,410`
   所有 `malloc`/`realloc` 应替换为 `xr_malloc`/`XR_REALLOC`。

2. **公共 API 无 `XRAY_API`/`XR_FUNC`**：`xr_yaml_parse`/`xr_yaml_parse_all`/`xr_yaml_stringify`、`yaml_parser_init` 等全部裸声明。

**🟠 重要**

3. **`\x` 转义产生非法 UTF-8**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/yaml/yaml_parser.c:191-203` — 对 `\xAB` 只写一字节 `0xAB`。YAML 1.2 规范要求 `\x` 是 Unicode code point 的 8-bit 表示，>= 0x80 时应 UTF-8 编码为两字节 `0xC2 0xAB`。否则字符串 `\xC3` 就变成非法 UTF-8 字节序列。

4. **`allow_duplicate_keys` 配置不生效**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/yaml/yaml_parser.c:531` 明确注释 `(void)p->config.allow_duplicate_keys`。`parseStrict` 永远不会在重复键时报错。

5. **`YAML_MAX_ANCHORS = 64` 硬编码** + 超过静默丢弃：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/yaml/yaml_parser.c:90`。大型配置文件（kubernetes manifest、helm chart）常超 64 个 anchor。应改成 `xr_malloc` 动态增长；且超限应 push 到 errors。

6. **`parseStrict.errors` 永远为空**：parser 从不往 `parser->result.errors` 推错误，`yaml_parser_parse_strict` 只是把 data 塞进去返回。"strict" 形同虚设。

7. **`emit.lineWidth` 参数接受但从不使用**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/yaml/yaml_emitter.c:28,400` — 公开 API 承诺的功能不存在。应删除或实现。

8. **Emitter 不支持 anchor/alias 输出**：循环引用的数据会无限递归直到 stack overflow。至少应加深度保护。

9. **`needs_quote` 过于简陋**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/yaml/yaml_emitter.c:87-113` — 不识别如下场景必须加引号：
   - 看起来像数字的字符串（`"123"` 被写成 `123` 然后 reparse 成 int）
   - YAML 1.1 truthy/falsy 词（`yes/no/on/off/y/n/True`）— 很多工具仍按 1.1 parse
   - 前导/尾随空格
   - 内部包含 `: ` 会破坏 block mapping
   推荐参考 [ruamel.yaml](https://yaml.readthedocs.io/en/latest/detail/) 的 `quote_required` 判定。

10. **`emit_string` 选择 `|` literal 的阈值 `length > 40` 是魔数**：应按是否包含 `\n` 决定，配合 emit_string 的 line_width config。

**🟡 建议**

11. `find_anchor` 用 `strlen(p->anchors[i].name)` 每次线性扫，能换成 `XrHashMap` 或在 `YamlAnchor` 里存 `name_len`。
12. `yaml_skip_to_eol` SIMD 路径未更新 `p->col`（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/yaml/yaml_scanner.c:64-74`），后续错误定位列号会偏。
13. Tab 用于缩进不报错（YAML 1.2 禁止）— 可能导致不可见歧义。
14. Scalar 识别里 `strncasecmp(".inf", 4)` 未区分 `.Inf` / `.INF` 大小写（其实 strncasecmp 会匹配），但 `.NaN` / `.NAN` 等大小写变体被正确处理。OK。

---

### 2.3 `stdlib/toml`（1,583 行，2 个 .c）

**🔴 严重**

1. **公共 API 缺 `XRAY_API`/`XR_FUNC`**：`xr_toml_parse`, `xr_toml_stringify`, `toml_parser_init/parse/cleanup`, `toml_config_*`。

**🟠 重要**

2. **`parse_datetime` 返回字符串而非 `XrDateTime`**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/toml/toml_parser.c:510-539` — TOML 的 datetime 是一等类型（spec §Local/Offset Date-Time），当前实现退化为字符串。应调用 `stdlib/datetime` 的 `xr_datetime_parse` 产出 `XrDateTime`。这是规范正确性问题。

3. **重复键不报错**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/toml/toml_parser.c:751-758` — 虽然 `add_error` 被调用，但紧接着 `xr_map_set` 仍然覆盖。TOML 规范强制要求 "You cannot define any key or table more than once"，应跳过赋值或让调用者终止。

4. **`[a]` 重定义不报错**：`get_or_create_table` 遇到已存在的 Map 直接复用。TOML 允许 "dotted keys 创建隐式表后显式 `[a.b]` 可继续"，但 `[a]` `[a]` 直接重复是错误，当前完全没检测。

5. **负数 hex/oct/bin 接受**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/toml/toml_parser.c:417-420,487-494` — `-0xFF` 会被 parse 成 -255。TOML spec 只允许 10 进制数字前加 `+/-`。

6. **下划线位置不校验**：`_1`、`1_`、`1__2` 都会被静默吃掉（parser 直接 `continue`），与 spec 冲突。

7. **Writer 不处理 `XrDateTime`、`XrJson`**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/toml/toml.c:152-207` — 只覆盖 `null/bool/int/float/string/array/map`。`XrJson` 遇到走默认 `null` 输出 `""`（甚至还不是）。

8. **控制字符 < 0x20 不转义**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/toml/toml.c:117-128` — `\b`/`\f` 直接按原文出，多行模式更是全都原样输出。spec 要求 basic string 不能含有 `U+0000`–`U+0008` 等非法字符，需 `\uXXXX` 转义。

9. **`make_prefix` 用 `sprintf` 不是 `snprintf`**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/toml/toml.c:216` — 虽然紧前面 `xr_malloc(plen + key->length + 2)` 足够，但习惯上永远用 `snprintf`。

10. **`indent` 参数接受但从不用**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/toml/toml.c:372-386` — API 承诺缩进控制，实际全部 flat。

**🟡 建议**

11. `TomlWriter.tw_str(NULL)` 会 UB — 所有 `tw_str` 应在宏里 `if (s) ...`。
12. 错误消息无细节（`"Expected ']'"` 对长文件定位困难），至少加一个 `snprintf("got '%c'", PEEK())` 片段。
13. 缺 `stringify` 上深度限制 — 循环引用死循环。

---

### 2.4 `stdlib/xml`（1,766 行，3 个 .c）

**🟠 重要**

1. **`validate_entities` 配置不生效**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/xml/xml_parser.c:316-317` 读入但 `decode_entities_inplace` / `decode_entity` 完全不看此标志。未知实体 `&nbsp;` 等 HTML 常见实体静默原样通过。strict 模式下应该报错。

2. **仅支持 5 个 named entities**（`lt/gt/amp/quot/apos`）— 对解析浏览器导出 HTML/XHTML 有严重限制。建议做一个 HTML5 entities 子表或至少 `&nbsp;`, `&copy;`, `&reg;`。

3. **Processing Instruction 内容被丢弃**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/xml/xml_parser.c:627-636` — `<?xml-stylesheet href="a.xsl"?>` 整条被吃掉。应作为 node 保留或通过 config 可选。

4. **`xml_node_set_attr` 重复属性 O(n)**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/xml/xml_node.c:102-110` — 线性扫描所有已存在属性比对。对属性 >50 的大节点会退化成 O(n²)。改小 hash 或用有序数组 + binary search。

5. **不处理 namespace (`xmlns:`)**：名字空间和 prefixed names 全当成字符串拼接。用 xray 处理 SOAP/SVG 会难受。至少要在 node 里记录一下命名空间 URI。

6. **`ensure_buf_cap` realloc 失败无反应**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/xml/xml_parser.c:59-65` — OOM 后 `*buf = NULL`，下一次 `buf_append_char` 写空指针。

7. **`extract_write_config` 存 `encoding` 指针**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/xml/xml.c:88` — `config->encoding = XR_TO_STRING(val)->data`。由于 intern 字符串 GC 期间不移动/不释放，目前安全；但跨 GC 运行时若 string 所属 XrJson 被 collect，data 可能悬空。把它拷贝到栈 char[32] 更稳。

8. **`xml_keys_init` 每次调用重做 `xr_string_intern`**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/xml/xml.c:53-64` 在 `xml_parse_fn` / `xml_stringify_fn` / `xml_element_fn` 等每个 binding 里都会被调用，而 intern 本身是 O(hash) 但仍有开销。应改 per-isolate 缓存。

**🟡 建议**

9. Self-closing `<br/>` 在 `XML_STATE_TAG_NAME` 路径创建元素但不更新 `parser->current`，OK。但 `XML_STATE_TAG_SPACE` 路径调用 `parser_close_element` 后如果 `parser->current == NULL`，depth 计数仍然 `--`（虽然里面有 `if (depth > 0)`），逻辑对但读起来别扭。
10. `XML_STATE_DOCTYPE` 整体吞掉，内部实体引用完全忽略（不展开 `<!ENTITY foo "bar">`）。这对多数 stdlib 用途足够，但应在文档说明。
11. `XmlNode` 对 XPath-like 查询没有 API（`querySelector`/`findAll`）— 用户得自己遍历 map tree。这是可以增值的方向。

---

### 2.5 `stdlib/csv`（1,424 行，2 个 .c）

**🟠 重要**

1. **`finish_row` 空行检查越界**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/csv/csv_parser.c:470-486` — 当 `col_count == 0` 时仍然执行 `xr_array_get(current_row, 0)`（取第 0 个元素），对空数组行为取决于 `xr_array_get` 实现，多数会返回 null 但这是 debug build 触发 assert 的经典边界。应加 `if (col_count == 0) { ... return; }` 快速分支。

2. **`ensure_temp_cap` realloc 失败无传递**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/csv/csv_parser.c:216-220` — OOM 时 temp_buf 不变，紧接着 `memcpy(parser->temp_buf + parser->temp_len, ...)` 仍写入旧 buf 尾端。会越界。

3. **SIMD QUOTED fast-path 可能跳过 escape**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/csv/csv_parser.c:653-664` — `xr_simd_find_char` 只找 `quote`；若用户将 `escape_char` 设为非 `quote`（比如 `\`），SIMD 会把 `\` 当普通字符跳过，之后 state machine 碰不到它，导致 `\"` 这种 escape 被当成字段终止。只有默认 `escape == quote == '"'` 时安全。应在 `escape != quote` 时禁用 SIMD 路径（或同时找两个字符 = `find_any`）。

4. **`%g` 精度不足**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/csv/csv.c:117` — default 6 位精度，round-trip 不保证。换 `%.17g`。

**🟡 建议**

5. **dynamic_typing 下 null 字符串无法保留**：`"null"` 单独字段被转成 `xr_null()`；若原始数据字段字面是 "null"，信息丢失。给个 `nullStrings: []` config。

6. `csv_detect_delimiter` 只扫前 4KB：文件头若是大段 header 包含 `,`（比如 CSV 内嵌 metadata），自动检测会偏向 `,`。文档说明这个限制即可。

7. 缺 `parseStream` 风格的增量接口。对 > 100MB 的日志 CSV，当前实现必须一次性读入。

8. `write_row` 处理 `XR_IS_NULL` 输出空字段（OK），处理 Map 时没设计（直接忽略）。数组元素如果是 Map，建议用 `json.stringify` 嵌入单字段。

---

## 3 · 代码量与技术债估算

### 3.1 重复代码

| 抽象 | 当前副本数 | 行数 × 副本 | 可删减 |
|------|----------|-------------|--------|
| Writer (init/ensure/append/char/str/indent/newline) | 5 | ~70 × 5 = 350 | ~280 |
| `add_error` 构造错误 Map（反复 `xr_string_intern`）| 3（`toml`/`csv`/`xml`）| ~30 × 3 = 90 | ~60 |
| `parseFile`/`writeFile` 骨架 | 4 | ~25 × 4 = 100 | ~70 |
| `extract_config` 从 Json 读取 bool/int/str | 5 | ~30 × 5 = 150 | ~110 |
| 数字解析兜底（`strtoll`/`strtod`/SWAR） | 3 | ~20 × 3 = 60 | ~20 |
| **合计可压缩** | — | **~750** | **~540** |

如果配合基础工具组 PR-1 的 `stdlib/common.h` 一起做，序列化组再加一个 `stdlib/common_parser.h` + `stdlib/common_writer.h`，可以减掉约 10% 代码。

### 3.2 malloc 迁移量

| 文件 | `malloc/realloc/free` 出现次数 |
|------|------------------------------|
| `json/json.c` | 7 |
| `yaml/yaml_parser.c` | 12 |
| `yaml/yaml_emitter.c` | 4 |
| `yaml/yaml.c` | 0（已改 xr_* ✓） |
| `toml/*` | 0 ✓ |
| `xml/*` | 0 ✓ |
| `csv/*` | 0 ✓ |
| **合计** | **23 处** 需要改 |

---

## 4 · 推荐的 PR 顺序

| # | 动作 | 受益范围 | 工作量 |
|---|------|---------|-------|
| P1 | 合入基础工具组 `stdlib/common.h`（`xrs_string_arg`/`xrs_string_value_n`/`XRS_EXPORT`） | 全组 | 已规划 |
| P2 | 新增 `stdlib/common_writer.h`，把 5 份 `XxWriter` 替换为 `StdlibBuffer`（用 `xr_malloc` + `XR_REALLOC`） | `json`/`yaml`/`toml`/`xml`/`csv` | S |
| P3 | `json` + `yaml` 的 `malloc/realloc/free` 全量替换；JSON/YAML `%.15g` → `%.17g` | `json`/`yaml` | S |
| P4 | 新增 `stdlib/common_parser.h`：统一 `MAX_DEPTH`、`add_error_map`、`extract_config_*`；`parseStrict` 错误 builder 收敛 | 全组 | M |
| P5 | `toml.parse_datetime` 返回 `XrDateTime`；Writer 支持 `XrJson`、`XrDateTime` | `toml` | M |
| P6 | `yaml` 补齐：`\x` UTF-8 修复、`needs_quote` 补全、`allow_duplicate_keys` 真实生效、`parseStrict` errors 真实写入 | `yaml` | M |
| P7 | `csv` 修复：`finish_row` 空行越界、SIMD escape/quote 不同时正确性、`%g` → `%.17g` | `csv` | S |
| P8 | `xml` 补齐：HTML5 entities 子表、PI 保留（选配）、`xml_keys` per-isolate 缓存、namespace 基础支持 | `xml` | M |
| P9 | `parseFile/writeFile` 全部走 `XrAsyncPool` | 全组 | L（依赖 io 模块改造） |
| P10 | 公开 C API 全加 `XR_FUNC`（或判定仅文件内用而改 static） | 全组 | S |

S ≈ 半天 · M ≈ 1-2 天 · L ≈ 3-5 天

---

## 5 · 测试覆盖空白

建议在 `tests/stdlib/` 增加以下用例：

### JSON
- `parse("null")` 与 parse 失败的区分（目前不可区分）
- `stringify(3.141592653589793)` → roundtrip 相等
- 深度 600 嵌套触发 `JSON_MAX_DEPTH`
- 重复键策略（当前后者胜出，应有测试锁定）
- `isValid("\"\x01\"")` 严格模式拒绝

### YAML
- `"\xC3"` double-quoted string → roundtrip 是 UTF-8 两字节
- 65+ anchor 数据
- `allow_duplicate_keys: false` 下出错
- `yes/no/on/off/True` 作为值需要被识别为字符串而非 bool（取决于 YAML 1.2 vs 1.1 语义）
- Flow / block 混合嵌套

### TOML
- `parse("dt = 2024-01-15T10:00:00Z")` 返回值是 `XrDateTime`
- `[a.b]` 后 `[a.b]` 再定义 → 报错
- `-0xFF` 拒绝
- `_1`、`1_`、`1__2` 拒绝
- `stringify(XrDateTime.now())` 产出合法 TOML datetime

### XML
- 大量 attribute（>100）性能回归
- `&copy;` 等未知实体在 `validate_entities: true` 下报错
- Namespace `<x:root xmlns:x="...">` 解析
- Processing Instruction 保留（若支持）
- 自闭合 tag + siblings

### CSV
- 空行在 `skip_empty_lines: false` 时保留
- escape ≠ quote 时 SIMD 正确性
- BOM + UTF-8 数据往返
- 超长字段（>256KB）
- `"`嵌套在无引号字段中（relax mode）

---

## 6 · 结语

本组的质量梯度清晰：

| 模块 | 红线合规 | 实现完整度 | 规范对齐 | 综合评分 |
|------|---------|----------|---------|---------|
| `xml` | ✅ | 高 | 中（namespace、entities 欠缺）| ⭐⭐⭐⭐ |
| `csv` | ✅ | 高（TSV/auto/header/relax）| 高 | ⭐⭐⭐⭐ |
| `toml` | ✅ | 中（datetime 退化、dup key 未检测）| **中低** | ⭐⭐⭐ |
| `json` | ❌（malloc）| 高 | 高 | ⭐⭐⭐ |
| `yaml` | ❌（malloc）| 中（strict/line_width/anchor output 欠缺）| **中低** | ⭐⭐ |

**下一步最紧迫**：

1. `json` + `yaml` 的 `malloc` 改造（红线，半天工作量）。
2. `yaml.\x` UTF-8 bug（影响非 ASCII 用户，属语义正确性）。
3. `toml` datetime 退化（影响生态互通）。

以上完成后，本组就达到与 `xml`/`csv` 同一水平线，可以作为后续"加解密/压缩/正则"组的参考样板。
