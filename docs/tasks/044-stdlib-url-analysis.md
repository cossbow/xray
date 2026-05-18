# stdlib/url 分析与优化建议

## 模块职责

`stdlib/url` 提供 URL 与 query string 的字符串处理能力：

- RFC 3986 percent encode/decode
- form encode/decode
- URL parse/format
- query parse/build
- relative URL resolve
- URL path join

该模块和 `encoding` 的边界应保持清晰：`url` 负责 URL 组件级 percent encoding，`encoding` 负责通用 hex/UTF/charset，`base64` 继续负责 base64/base64url。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/url/url.c` | C-level percent encoding、URL parser、query、resolve、loader |
| `stdlib/url/url.h` | C-level URL encode API 和 loader 声明 |
| `stdlib/ctxbuf.h` | format/buildQuery/resolve/join 动态字符串构造 |
| `tests/unit/encoding/test_url_encoding.c` | C-level encode/decode 单测 |
| `tests/regression/10_stdlib/1411_url_parse.xr` | 脚本层 parse/format/query/resolve/join 回归 |
| `src/frontend/analyzer/xanalyzer_builtins_generated.h` | url builtin 生成签名 |
| `src/app/lsp/xlsp_stdlib.c` | LSP url 手写提示 |

## 当前 API

| API | 当前语义 |
|---|---|
| `url.encode(s)` | RFC 3986 percent encode，保留 unreserved |
| `url.decode(s)` | percent decode，不把 `+` 当空格 |
| `url.encodeForm(s)` | form encode，空格变 `+` |
| `url.decodeForm(s)` | form decode，`+` 变空格 |
| `url.parse(url)` | 返回 Json：protocol、hostname、port、pathname、search、hash、username、password、host、origin |
| `url.format(obj)` | 从 Json 字段拼接 URL |
| `url.parseQuery(qs)` | query string 转 Json |
| `url.buildQuery(obj)` | Json 转 query string |
| `url.resolve(base, relative)` | 相对 URL 合并 |
| `url.join(...parts)` | URL path segment 拼接 |

## 依赖与架构边界

### 问题 1：`url.c` 依赖 runtime internal 和 symbol table

`url.c` include：

- `src/runtime/xisolate_internal.h`
- `src/runtime/symbol/xsymbol_table.h`
- `src/coro/xcoroutine.h`

其中 `buildQuery()` 直接访问 `X->symbol_table` 和 Json shape 字段。这样 stdlib/url 与 runtime object 内部表示绑定较深。

建议：

- 为 Json 对象提供稳定的 field iteration API。
- `buildQuery()` 不直接访问 isolate internal 和 symbol table。
- 统一 stdlib 对 Json/Map 的遍历方式。

### 问题 2：`url.h` 同时承担 C API 与 loader 声明

`url.h` 暴露 `xr_url_encode()` 等 C API，但也 include `xmodule.h` / `xvm.h`。

建议：

- 拆分轻量 URL encoding C API header。
- loader 声明交给 `xmodule_loaders.h` 或模块私有 header。
- C API header 只保留 `xdefs.h`、`stddef.h`、`stdbool.h` 等低层依赖。

### 问题 3：loader 定义缺少统一可见性修饰

`xr_load_module_url()` 定义没有 `XR_FUNC`。

建议和其他 stdlib loader 一起统一修复。

## 编码与解码语义

### 问题 4：C-level encode/decode 在 buffer 不足时静默截断

`xr_url_encode()` / `xr_url_decode()` 接收外部 buffer。如果 buffer 不足，当前逻辑直接停止写入并返回已写长度，没有错误信号。

影响：

- C 调用者可能把截断结果当完整 URL 使用。
- 脚本层当前按最大膨胀预分配，常规路径不触发，但 C API 对外不安全。

建议：

- C API 改为先计算所需长度，或返回 status + out_len。
- buffer 不足应返回错误，不应返回部分合法字符串。
- 增加 C 单测覆盖小 buffer 截断。

### 问题 5：长度计算缺少 overflow guard

脚本层 encode 使用：

- `s->length * 3 + 1`
- `src_len * 3`

极端输入会 `size_t` 溢出。

建议：

- 增加 `SIZE_MAX / 3` 上界检查。
- 统一使用 `XrCtxBuf` 逐步追加，避免预估乘法。

### 问题 6：invalid percent sequence 被原样保留

`url.decode("%ZZ")` 会保留 `%`，不是返回 null 或错误。

这可以作为 lenient decode 设计，但需要文档说明。否则用户可能以为 decode 会校验 percent encoding。

建议：

- 明确 `decode` 是 lenient passthrough。
- 如需严格校验，增加 `decodeStrict` 或 `isEncoded`。

### 问题 7：decode 可以生成任意 bytes string

percent decode 可生成非 UTF-8 bytes 或 NUL byte。当前直接 intern 为 string。

建议：

- 明确 Xray string 是否允许任意 bytes。
- 如果 string 应是 UTF-8 文本，decode 后需要 UTF-8 validate。
- 或增加 bytes API，例如 `decodeToBytes()`。

## URL parse/format 语义

### 问题 8：scheme parsing 只识别 `://`

`url_parse_internal()` 只有在 `scheme://` 存在时才识别 protocol。因此：

- `mailto:user@example.com`
- `urn:example:animal:ferret:nose`
- `data:text/plain,abc`

这类 RFC 3986 URI 不会按 scheme 解析。

建议：

- 如果模块目标是 URL 而非泛 URI，应文档明确只支持 authority URL。
- 如果宣称 RFC 3986，应支持 generic URI scheme。

### 问题 9：无效 port 字段处理不一致

`parse()` 会验证 port 是否为 0..65535 的数字。无效时 `port` 字段输出空字符串。

但 `host` 和 `origin` 的派生逻辑仍使用原始 `parts.port`：

- `url.parse("http://h:abc/").port == ""`
- `host` 仍可能是 `h:abc`
- `origin` 仍可能是 `http://h:abc`

建议：

- 派生 `host/origin` 时使用 validated port。
- 或保留 rawPort 字段，把合法 port 和原始 port 区分。

### 问题 10：userinfo 使用第一个 `@`

userinfo 查找第一个 `@`。实际 URL 中 userinfo 可能包含 percent-encoded `@`，raw `@` 情况也更接近使用最后一个 `@` 分割 authority。

建议：

- 明确 raw `@` 不被允许，要求 percent-encode。
- 或使用最后一个 `@` 作为 host 分隔。
- 增加 userinfo 边界测试。

### 问题 11：hostname 不做 IDNA/punycode/大小写规范化

当前 hostname 只做字符串切片，不处理：

- Unicode domain
- punycode
- host lowercase normalization
- percent-encoded host

短期可以接受，但需文档说明非目标。

### 问题 12：format 不做组件编码

`url.format()` 直接拼接 Json 字段，不对 username/password/path/search/hash 做 percent encoding。

建议：

- 明确 format 输入字段必须已编码。
- 或按组件语义自动编码。
- 如果自动编码，需要避免 parse/format roundtrip 双重编码。

## resolve 与路径处理

### 问题 13：dot-segment 处理作用于整个 URI buffer

`url_resolve_fn()` 先拼出完整 URI，再调用 `remove_dot_segments(result.data, result.len)`。

RFC 3986 的 dot-segment removal 应作用于 path component，而不是整个 URI。当前对整个 URI 做 in-place 处理，遇到 `https://host/../x` 这类路径时，回退逻辑可能跨过 path 边界影响 authority。

影响：

- authority 可能被错误裁剪。
- query/hash 中如果包含类似路径片段，也可能被错误处理。

建议：

- resolve 时先分别构造 scheme/authority/path/query/fragment。
- 只对 path component 调用 dot-segment removal。
- 增加测试：`url.resolve("https://h/a", "../x")`、`url.resolve("https://h/a", "/../x")`、query/hash 中包含 `/../`。

### 问题 14：absolute URI 判断只接受 `scheme://`

`resolve()` 对 relative 是否 absolute 的判断也是 `scheme://`。这与 RFC generic URI 不一致。

建议同 parse：要么明确只支持 hierarchical URL，要么实现 scheme 通用判断。

### 问题 15：url.join 与 path.join 语义可能混淆

`url.join()` 只是 URL path 拼接，不做 percent encoding，也不保留 scheme/authority 特殊语义。

例如传入 `"https://example.com"` 与 path segment 时，可能得到用户不预期的结果。

建议：

- 文档明确它是 path segment join，不是 URL resolve。
- 或改名为 `joinPath`。
- URL 合并推荐 `resolve()`。

## query 语义

### 问题 16：parseQuery 不支持重复 key

`parseQuery("a=1&a=2")` 会后写覆盖前写，无法表达数组或多值 query。

建议：

- 明确重复 key 策略：last wins / first wins / array。
- 如果使用 Json，支持 array value 更符合常见 query 语义。

### 问题 17：buildQuery 只真正处理 string 值

`buildQuery()` 对 string 值输出 `key=value`；对非 null 非 string 值输出 `key=`，不会把 int/bool/float 转字符串。

影响：

- Json 中常见数字/布尔 query 值无法正确输出。

建议：

- 明确只支持 string value。
- 或调用统一 value-to-string writer。
- 增加 int/bool/null 测试。

### 问题 18：parseQuery OOM 策略不一致

`dec_buf` 分配失败时返回空 Json；key buffer 分配失败时使用 `XR_CHECK` 终止。

建议：

- 标准库 native 函数的 OOM 策略统一。
- 不能静默返回部分或空结果。

## 测试覆盖

现有覆盖：

- C-level RFC encode/decode/form encode/form decode。
- 脚本层 parse、userinfo、IPv6、format roundtrip。
- parseQuery/buildQuery 基础。
- encodeForm/decodeForm。
- resolve 基础绝对 path、相对 path、absolute URL。
- join 基础。

缺口：

1. C API buffer 不足截断。
2. encode length overflow 防护。
3. invalid percent sequence 严格/宽松策略。
4. decode 生成 NUL/非 UTF-8 bytes。
5. non-authority URI scheme，例如 mailto/data/urn。
6. invalid port 对 port/host/origin 的一致性。
7. userinfo 中多个 `@`。
8. IDNA/punycode 非目标说明或测试。
9. resolve 只对 path 做 dot-segment removal。
10. query/hash 中包含 `/../` 的 resolve。
11. parseQuery 重复 key。
12. buildQuery 非 string value。
13. format 是否编码组件。
14. `join()` 对 absolute URL 参数的行为。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 严重 | `resolve()` 对完整 URI 做 dot-segment removal | 可能破坏 authority/query/hash | 只处理 path component |
| 高 | C API buffer 不足静默截断 | C 调用者可能使用不完整 URL | 返回错误或所需长度 |
| 高 | 无效 port 在 `port` 与 `host/origin` 中不一致 | parse 结果自相矛盾 | 派生字段使用 validated port |
| 高 | `url.c` 依赖 isolate internal/symbol table | stdlib 与 runtime 内部耦合 | Json field iterator API |
| 中 | `scheme://` 以外 URI 不支持 | RFC 3986 声明过宽 | 限定目标或补 generic URI |
| 中 | query 重复 key 不支持 | 常见 query 数据丢失 | 支持 array 或定义覆盖策略 |
| 中 | buildQuery 非 string 值输出为空值 | 数字/布尔 query 不可用 | 统一 value stringify |
| 中 | decode invalid percent 宽松但未文档化 | 用户误以为会校验 | 增加 strict API 或文档 |
| 中 | decode 可生成任意 bytes string | 文本/二进制边界模糊 | 定义 string/Bytes 语义 |
| 低 | loader/header 依赖需收敛 | 模块边界不清 | 拆分 C API header，loader 加修饰 |

## 后续实施建议

建议优先修 correctness 与 API 自洽问题：

1. 修复 `resolve()`，只对 path component 执行 dot-segment removal。
2. 修复 invalid port 派生字段不一致。
3. 为 C-level encode/decode 增加所需长度或错误状态。
4. 明确 URL 模块支持的是 authority URL 还是 generic URI。
5. 收敛 query 多值、非 string value、format 编码策略。
6. 通过 Json iterator API 去掉对 runtime internal/symbol table 的直接依赖。
7. 拆分轻量 C API header。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
