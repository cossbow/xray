# stdlib/regex 分析与优化建议

## 模块职责

`stdlib/regex` 是 Xray 的正则标准库模块，同时承接语言级正则字面量 `/pattern/flags` 的运行时语义。它不是系统库封装，而是自研 RE2-style 引擎，核心目标是：

- 将 pattern 解析为 AST，再编译为 regex program。
- 使用 NFA 执行语义，必要时用 DFA cache 加速。
- 支持捕获组、命名捕获、Unicode property、flags、替换、分割和对象方法。
- 避免 backtracking-only 特性，降低 ReDoS 风险。
- 通过 `XR_TREGEX` 让编译后的 Regex 由 GC 管理。

当前模块同时承担三类职责：C regex 引擎、脚本层 `regex` 模块函数、Regex 对象实例方法。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/regex/xregex.h` | C public API、flags、error、match/capture 结构 |
| `stdlib/regex/xregex_internal.h` | AST、program、NFA/DFA 内部结构和限制常量 |
| `stdlib/regex/xregex_parse.c` | pattern parser、inline flags、Unicode property、捕获组、语法错误 |
| `stdlib/regex/xregex_compile.c` | AST 到 program 的编译、prefix/literal/onepass/bytemap 优化 |
| `stdlib/regex/xregex_nfa.c` | NFA 执行 |
| `stdlib/regex/xregex_dfa.c` | DFA state cache、memory budget、DFA 搜索 |
| `stdlib/regex/xregex.c` | compile/test/match/findAll/replace/split/escape 等 public API |
| `stdlib/regex/xregex_binding.c` | 模块导出、Regex GC object、literal bridge、native type 注册 |
| `stdlib/regex/regex_methods.c` | `XrMethodSlot` 风格的 Regex 实例方法 |
| `src/vm/xvm_dispatch_assert.inc.c` | `OP_REGEX_COMPILE` 调用 `xr_regex_compile_literal` |
| `src/runtime/gc/xgc.c` | `XR_TREGEX` destructor 注册 |
| `src/runtime/value/xmethod_table.c` | `XR_TID_REGEX` method table 注册 |

## 当前脚本 API

`xr_load_module_regex()` 实际导出 11 个模块函数：

| API | 当前语义 | 返回值 |
|---|---|---|
| `compile(pattern, flags?)` | 编译 pattern，flags 支持 `i/m/s` | Regex 或 `null` |
| `test(re, text)` | 搜索是否存在匹配 | `bool` |
| `find(re, text, offset?)` | 从 byte offset 搜索第一个 match | `Json?` |
| `fullFind(re, text)` | 整串匹配 | `Json?` |
| `count(re, text)` | 统计 match 数量 | `int` |
| `findAll(re, text, limit?)` | 返回 match object 列表 | `Array<Json>` |
| `replace(re, text, replacement)` | 替换第一个 match | `string?` |
| `replaceAll(re, text, replacement)` | 替换所有 match | `string?` |
| `split(re, text, limit?)` | 按 pattern 分割 | `Array<string>` |
| `escape(text)` | 转义 regex 元字符 | `string?` |
| `isValid(pattern)` | 检查 pattern 是否可编译 | `bool` |

Match object 当前包含：`start`、`end`、`text`、`groups`。`groups[0]` 是完整匹配，后续元素是捕获组；未匹配的捕获组返回 `null`。

Regex object 当前支持实例方法：`test`、`find`、`findAll`、`replace`、`replaceAll`、`split`，并提供 `pattern` getter。

## 已有优势

- **安全方向正确**：没有实现 backreference/lookaround 这类 backtracking-only 特性，符合 RE2-style 的可控复杂度目标。
- **限制常量明确**：已有 program instruction、repeat、nested depth、capture count、DFA memory 等上限。
- **生命周期清晰**：Regex object 由 GC 管理，底层 `XrRegex` 由 `regex_object_destroy` 释放。
- **字面量复用同一入口**：`/pattern/flags` 通过 `OP_REGEX_COMPILE` 进入 `xr_regex_compile_literal()`，避免与 `regex.compile()` 完全分裂。
- **Unicode 覆盖较好**：支持 `\p{Han}`、`\p{L}`、`\p{N}`、`\P{...}` 等 property。
- **测试基础较完整**：已有基础、Unicode、advanced flags、RE2 compatibility、regex literal 回归测试。
- **空匹配处理有防护**：`findAll/count/replace/split` 对零长度 match 有推进逻辑，避免直接无限循环。

## 主要问题

### 1. LSP 声明严重缺失

LSP 中 `regex_symbols` 只列出 `test/find/findAll/replace/split`，缺失模块函数：

- `compile`
- `fullFind`
- `count`
- `replaceAll`
- `escape`
- `isValid`

也缺失 Regex 实例能力：

- `pattern`
- `replaceAll`
- `find(text, offset?)`
- `findAll(text, limit?)`
- `split(text, limit?)`

并且 LSP 把 `find` 描述为返回 `string?`，但 runtime 返回 match object。补全、hover 和诊断会误导用户。

### 2. analyzer generated 与 runtime 语义漂移

`xanalyzer_builtins_generated.h` 中的 regex 元数据与 runtime 不一致：

| API | runtime 实际 | analyzer 当前 |
|---|---|---|
| `compile` | 失败可返回 `null` | `Regex` |
| `find` | `Json?` | `string?` |
| `fullFind` | `Json?` | `Array<string>?` |
| `findAll` | `Array<Json>` | `Array<string>` |
| `split` | 支持 `limit?` | 无 `limit?` |
| `replace/replaceAll` | 参数错误或 OOM 可返回 `null` | `string` |

这会导致静态分析、补全和真实运行结果不一致。建议把 Regex、Match 类型先显式建模，再统一生成 analyzer/LSP 声明。

### 3. `XR_DEFINE_BUILTIN` 自身也不够准确

`xregex_binding.c` 里的 declaration 为了绕过类型系统使用了大量 `any`，例如 `compile` 返回 `any`、pattern 参数为 `any`。这能降低接入成本，但会隐藏真实 API 约束。

建议：

- 引入 `Regex` 和 `RegexMatch` 的标准类型声明。
- `compile` 明确为 `Regex?`，或者在 compile error 抛异常后返回 `Regex`。
- `find/fullFind` 明确为 `RegexMatch?`。
- `findAll` 明确为 `Array<RegexMatch>`。

### 4. 模块函数与实例方法存在双轨实现

当前存在两套实例方法注册路径：

- `xregex_binding.c` 中的 `XrNativeMethod regex_methods`。
- `regex_methods.c` 中的 `xr_regex_method_table`，由 `xmethod_table.c` 注册到 `XR_TID_REGEX`。

两者的职责边界不清晰，容易出现方法数量、签名、错误处理不一致。建议保留单一权威路径，另一条只做兼容桥接或删除。

### 5. `regex_compile_literal()` 与 `regex.compile()` 错误语义不同

`regex.compile()` 在失败时调用 `xr_runtime_error()` 并返回 `null`；`xr_regex_compile_literal()` 失败时静默返回 `null`。此外 literal flags 对未知字符是静默忽略，以保持旧行为；而 `parse_flags()` 也基本忽略未知 flags。

风险：

- 字面量编译失败可能在后续调用处才表现为 `null` 错误。
- 错拼 flags 不易发现。
- 同一个 pattern 在模块函数和 literal 下错误体验不同。

建议：

- 统一 compile error 策略：要么全部抛 `RegexError`，要么全部返回 `Result/Regex?` 并可读取错误信息。
- 对未知 flags 给出诊断；如果考虑兼容，至少在 analyzer/LSP 层提示。

### 6. DFA cache 可能有共享可变状态问题

`XrRegex` 编译后持有 `dfa`，DFA 内部有 `state_cache`、`start_state`、work queue、`mem_used` 等可变字段。匹配时会 lazy 创建和写入 cache。

如果同一个 Regex object 被多协程共享，DFA cache 不是只读结构，当前未看到 mutex/atomic 保护。虽然语言层普通对象跨协程有隔离规则，但 `shared const` 或原生对象未来若支持跨协程共享，需要明确 Regex 是否可共享。

建议：

- 明确 Regex object 的并发模型：不可跨协程共享、共享时 clone DFA、或让 DFA cache per-coro/per-match。
- 如果允许共享，DFA cache 必须加锁或改为不可变/线程本地缓存。
- 文档中声明 compiled regex 是否可复用、是否线程安全。

### 7. `split` 结果数量存在隐式 256 上限

binding 层根据 `limit` 分配 parts：无 limit 时 `max_parts = 256`。因此 `regex.split(re, text)` 默认最多返回 256 段，这不是常见语言的默认行为，也未在声明中体现。

风险：长文本分割被静默截断。

建议：

- 语义上明确默认 unlimited，内部动态扩容。
- 或文档明确默认 cap，并提供显式 `limit`。
- analyzer/LSP 必须暴露 `limit?` 参数。

### 8. `findAll` 默认也有上限语义

底层 `xr_regex_find_all()` 默认初始 capacity 为 16，并在未指定 limit 时最多扩容到 1024；指定 limit 可降低上限。当前脚本 API 没有明显文档说明默认最多返回 1024 个 match。

建议：

- 明确 `limit` 的语义和默认上限。
- 如果 API 名为 `findAll`，默认静默 1024 cap 可能不符合直觉，应考虑改名或改为动态增长到内存上限。

### 9. replacement 不支持捕获组引用

`replace/replaceAll` 当前把 replacement 当作普通字符串拼接，未看到 `$1`、`${name}` 或 `\1` 这类捕获组展开语义。已有测试只覆盖固定替换字符串。

建议：

- 明确标准库是否支持 capture interpolation。
- 如果不支持，文档说明 replacement 是 literal。
- 如果支持，增加 `$0/$1/${name}` 规则、转义规则和缺失捕获行为。

### 10. named capture 未暴露到脚本 match object

parser 支持命名捕获并保存 capture names，但 binding 返回的 match object 只有 `groups` 数组，没有 `namedGroups` 或按名称访问能力。

建议：

- 在 `RegexMatch` 中增加 `namedGroups: Json` 或 `Map<string, string?>`。
- 增加 named capture 的 parser、runtime、JSON shape 测试。

### 11. offset 是 byte offset，不是字符 offset

`find(re, text, offset?)` 直接按 C 字符串指针偏移，语义更接近 byte offset。UTF-8 文本中如果 offset 落在 codepoint 中间，行为需要明确。

建议：

- 文档明确 offset 单位。
- 如果脚本层字符串以 Unicode 字符为主要抽象，应改为 character offset 或至少校验 UTF-8 boundary。

### 12. 错误与 OOM 语义不统一

当前多处参数错误返回空值、空数组或 `false`；compile error 报 runtime error；replace OOM 可能返回 `null`；escape 分配失败可能返回原始输入。

建议形成统一策略：

- 参数类型错误：runtime error。
- pattern 语法错误：`RegexError` 或 `Regex? + lastError`，不能两套混用。
- OOM：统一 runtime OOM error，避免返回看似合法的旧值。
- `isValid()` 只负责布尔检查，不吞掉其他系统性错误。

## 性能与 ReDoS 边界

当前引擎总体方向正确，但需要把边界变成可验证契约。

已存在的有利因素：

- parser 限制 repeat 上限、nested depth、capture count。
- compiler 限制 program instruction 数量。
- DFA 有 memory budget 和 max state count。
- NFA/DFA 都避免递归 backtracking。
- 空匹配推进避免无限循环。

仍需明确的边界：

- 最坏情况下 DFA 失败后 NFA 的时间复杂度上界。
- Unicode property 在字符类组合中的开销上界。
- `replaceAll/findAll/split` 对超长文本和超多 match 的内存策略。
- DFA cache 写入失败后的 fallback 策略是否稳定可预测。
- pattern 编译时 AST arena、capture name、program instruction 的内存上限是否对外可诊断。

建议增加 microbench/regression：

- `(a+)+$`、`(a|aa)*b`、`([a-z]?){N}` 等 ReDoS 常见样例。
- 超长文本无匹配、有大量空匹配、大量短匹配。
- 超大 alternation、深层嵌套、超大 repeat、过多捕获组。
- DFA memory budget 命中后的 fallback 结果一致性。

## 测试覆盖现状

已有测试：

- `1100_regex.xr`：基础 compile/test/find/fullFind/replace/split/escape/isValid。
- `1110_regex_unicode.xr`：Unicode property。
- `1115_regex_advanced.xr`：lazy、ignorecase、multiline、dotall。
- `1120_regex_re2_compat.xr`：大量 RE2 兼容基础语义。
- `1130_regex_literal.xr`：正则字面量。

主要缺口：

- LSP/analyzer 与 runtime API 的生成一致性测试。
- Regex 实例方法完整测试，特别是 `re.replaceAll()`、`re.pattern`、`limit/offset`。
- named capture 运行时暴露测试。
- replacement 捕获组引用语义测试或“不支持”测试。
- `split` 超过 256 段的行为测试。
- `findAll` 超过 1024 个 match 的行为测试。
- invalid flags、invalid offset、参数类型错误的诊断测试。
- UTF-8 offset 落在字符中间的测试。
- compile limits：repeat、nested depth、capture count、program instruction。
- DFA budget/fallback 的压力测试。
- regex literal 编译失败的错误路径测试。

## 优化建议

### 优先修正 API 真相源

1. 设计并声明 `Regex`、`RegexMatch` 类型。
2. 让 `XR_DEFINE_BUILTIN`、analyzer、LSP 由同一份 metadata 生成。
3. LSP 同时支持 module function 补全和 Regex object method 补全。
4. 修正 `find/fullFind/findAll/compile/split/replace` 的返回类型和可空性。

### 统一错误语义

建议选择一种主策略：

- **异常策略**：`regex.compile()` 和 regex literal 编译失败都抛 `RegexError`，成功返回 `Regex`。
- **可空策略**：编译失败都返回 `null`，并提供 `regex.lastError()` 或 `regex.compileResult()`。

当前混合策略不利于 analyzer、LSP 和用户代码。

### 明确集合 API 的上限

`findAll` 和 `split` 应该显式定义：

- 默认是否无限。
- 默认上限是多少。
- `limit <= 0` 的意义。
- 达到上限时是否截断、返回剩余文本、还是报错。

如果继续保留默认 cap，API 文档和测试必须明确。

### 清理实例方法注册路径

建议保留 `XR_TID_REGEX` method table 作为实例方法的唯一权威入口；`xregex_binding.c` 只负责模块函数、object wrapper 和 native type 注册所需最小信息。这样能减少双轨漂移。

### 明确并发模型

Regex 编译结果看似不可变，但 DFA cache 是 lazy mutable。建议至少在文档中声明：

- Regex object 是否允许跨协程共享。
- DFA cache 是否属于 Regex object 状态。
- 多协程共享时是否需要 clone 或锁。

如果未来支持共享 compiled regex，优先考虑 per-coro DFA cache 或锁保护。

## 建议验收清单

- `regex` 模块函数、Regex 实例方法、analyzer、LSP 四者 API 完全一致。
- `RegexMatch` 形状有稳定文档和类型声明。
- compile/literal/isValid 的错误语义一致。
- `findAll/split` 的默认上限和 `limit` 语义有测试。
- named capture 要么暴露并测试，要么从对外能力中移除或标为内部能力。
- replacement capture interpolation 要么实现并测试，要么明确 replacement 是 literal。
- ReDoS 压力样例不会出现指数级耗时。
- DFA budget 命中后结果仍与 NFA 一致。

## 结论

`stdlib/regex` 的引擎方向和基础能力较成熟，是当前标准库中技术含量较高的模块。核心风险不在基础匹配功能，而在“脚本 API 类型真相源不统一”和“隐藏上限/错误语义未文档化”。

后续重构应优先统一 `Regex`/`RegexMatch` 类型、修正 analyzer/LSP 漂移、明确 `findAll/split/replace` 语义，再处理 DFA cache 并发模型和 ReDoS 压力测试。
