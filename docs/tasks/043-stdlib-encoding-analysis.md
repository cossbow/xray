# stdlib/encoding 分析与优化建议

## 模块职责

`stdlib/encoding` 当前提供三类能力：

- hex 编解码
- UTF-8 校验、字符数、字节长度
- UTF-16 LE/BE 编解码和 BOM 处理

该模块的合理边界是“字符编码与二进制表示转换”。它不应重复 `base64` 模块职责，也不应扩展为完整 Unicode 属性库；Unicode 属性和字符分类当前属于 `src/base/xunicode.*` 与 regex 等底层能力。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/encoding/encoding.c` | hex、UTF-16 C API、脚本 binding、loader |
| `stdlib/encoding/encoding.h` | C-level API、UTF-16 endian enum、loader 声明 |
| `src/runtime/object/xutf8.h` / `xutf8.c` | UTF-8 decode/encode/strlen/validate 底层实现 |
| `src/base/xsimd.h` / `xsimd.c` | hex char 到 nibble 的查表能力 |
| `tests/unit/encoding/test_hex_encoding.c` | C-level hex 单测 |
| `tests/unit/encoding/test_unicode.c` | Unicode 属性底层单测，不直接测 stdlib/encoding |
| `tests/regression/10_stdlib/1300_encoding.xr` | encoding 脚本层回归测试 |

## 当前 API

### C-level API

| 函数 | 当前语义 |
|---|---|
| `xr_hex_encode(data, len, output)` | bytes 转 lowercase hex，返回输出长度 |
| `xr_hex_decode(hex, len, output)` | hex 转 bytes，失败返回 -1 |
| `xr_hex_valid(hex, len)` | 校验 hex 字符串和偶数长度 |
| `xr_utf16_encode(utf8, utf8_len, output, out_cap, endian)` | UTF-8 转 UTF-16 bytes |
| `xr_utf16_decode(utf16, utf16_len, output, out_cap, endian)` | UTF-16 bytes 转 UTF-8 |
| `xr_utf16_encoded_len(utf8, utf8_len)` | 计算 UTF-16 输出字节数 |
| `xr_utf16_to_utf8_len(utf16, utf16_len, endian)` | 计算 UTF-16 解码为 UTF-8 的字节数 |

### 脚本层 API

| API | 当前语义 |
|---|---|
| `encoding.hexEncode(data: string)` | string bytes 转 hex string |
| `encoding.hexDecode(hex: string)` | hex 转 `Array<uint8>?` |
| `encoding.hexDecodeString(hex: string)` | hex 转 string? |
| `encoding.hexValid(hex: string)` | 判断 hex 是否有效 |
| `encoding.utf8Valid(data: string)` | 调用 core UTF-8 校验 |
| `encoding.utf8Count(data: string)` | 返回 UTF-8 codepoint 数 |
| `encoding.utf8ByteLength(data: string)` | 返回底层 byte length |
| `encoding.utf16Encode(data: string, endian?: int)` | string 转 UTF-16 bytes |
| `encoding.utf16Decode(data: any, endian?: int, stripBom?: bool)` | string/Bytes 转 UTF-16 解码 string? |
| `encoding.LE` / `encoding.BE` | endian 常量 |

## 依赖与架构边界

### 问题 1：`encoding.h` 同时承担 C API 与 loader，依赖偏重

`encoding.h` 暴露 C-level API，但同时 include：

- `xmodule.h`
- `xvm.h`
- `xstring.h`
- `xutf8.h`
- `xsimd.h`

这让只想使用 `xr_hex_encode()` 或 `xr_utf16_encode()` 的 C 模块被迫引入 module/vm/runtime object 依赖。

建议：

- 拆分轻量 C API header，例如只依赖 `xdefs.h`、`stdint.h`、`stddef.h`、`stdbool.h`。
- loader 声明改由模块私有 header 或 `xmodule_loaders.h` 维护。
- `xutf8.h` 只在实现文件或专用 Unicode API header 中使用。

### 问题 2：loader 定义缺少统一可见性修饰

`xr_load_module_encoding()` 的声明使用普通函数声明，定义处也没有 `XR_FUNC`。

建议和其他 stdlib loader 一起统一修复。

### 问题 3：与 `base64` 模块的边界需要明确

`base64` 和 `encoding` 都涉及“bytes/string 的文本表示转换”。当前划分是：

- `base64` 独立模块处理 base64/base64url。
- `encoding` 处理 hex 和 UTF-16。

这个划分可以保留，但需要在文档与 API 命名上明确。否则后续容易把 base64、percent encoding、charset transcoding 都混入 `encoding`。

建议：

- `encoding` 聚焦 charset/hex。
- `base64` 保持独立，因为已有 C API 被 `http/ws` 使用。
- percent encoding 归 `url`。

## API 签名漂移

### 问题 4：builtin 声明与 analyzer 生成结果不一致

`encoding.c` 中 builtin 声明：

- `utf16Decode(data: any, endian?: int, stripBom?: bool): string?`

但 `src/frontend/analyzer/xanalyzer_builtins_generated.h` 中仍显示：

- `utf16Decode(data: Bytes, endian?: int): string?`

这意味着脚本层实际支持 string/Bytes 和 `stripBom`，但静态分析器可能仍按旧签名检查。

影响：

- 使用第三个参数可能被 analyzer 拒绝。
- 传 string 到 `utf16Decode` 的实现可行，但类型系统可能不认。
- stdlib 类型生成流程存在同步缺口。

建议：

- 重新生成 builtin 类型声明。
- 将生成结果纳入验证流程。
- 对 `any` 参数是否合理单独评审，优先考虑 `Bytes|string` 不可表达时的文档策略。

### 问题 5：LSP stdlib signature 也存在漂移

`src/app/lsp/xlsp_stdlib.c` 中 encoding 签名与当前 builtin 不一致，例如：

- `hexEncode` 被描述为 `fn(data: Bytes): string`，但 builtin 是 string。
- `utf16Decode` 没有 `stripBom` 参数。

影响：

- IDE 提示与编译器/运行时不一致。
- 用户按 LSP 提示写代码可能遇到 analyzer 或运行时差异。

建议：

- LSP 不应手写 stdlib signature，或至少从同一生成源同步。
- 在 stdlib 类型生成后自动更新 analyzer 与 LSP。

## 内存与生命周期

### 当前正确点

- 临时 buffer 都使用 `xr_malloc/xr_free`。
- 脚本层 hex/UTF-16 转换后都会释放临时 buffer。
- `make_bytes()` 复制数据到 GC 管理的 typed array，临时 buffer 生命周期清晰。

### 问题 6：长度返回值使用 `int`，超大输入会溢出

C API 返回长度使用 `int`：

- `xr_hex_encode()` 返回 `int`，内部把 `len * 2` 转 int。
- `xr_hex_decode()` 返回 `int`。
- `xr_utf16_encoded_len()` 返回 `int`。
- `xr_utf16_to_utf8_len()` 返回 `int`。

绑定层也用 int 创建 bytes：

- `make_bytes(X, output, out_len)`
- `xr_array_with_capacity_typed(coro, len, XR_ELEM_U8)`

影响：

- 大输入可能长度截断。
- `len * 2 + 1`、`out_len + 2`、`out_len + 1` 缺少 size 上界检查。

建议：

- C API 长度计算使用 `size_t` 输出参数，错误用 bool/int status 表达。
- 脚本层创建 Bytes 前检查 `len <= INT32_MAX`。
- 所有乘法/加法加 overflow guard。

### 问题 7：`xr_hex_encode()` 对空输入和 NULL 输出语义混在一起

`xr_hex_encode()` 当前遇到 `!data || !output` 返回 0。空输入也返回 0。

影响：

- 调用者无法区分合法空输入和错误。
- 这类 API 被外部 C 模块复用时容易误用。

建议：

- C API 统一返回 bool/status，长度通过 out param 返回。
- 或约定 NULL + len 0 合法，NULL + len > 0 错误，并文档化。

### 问题 8：`xr_utf16_to_utf8_len()` 预扫描不完整校验 surrogate pair

长度预扫描遇到 high surrogate 时只跳过后续两个字节并按 supplementary codepoint 计算长度，没有验证后一个 unit 是否为 low surrogate。实际 decode 阶段会校验并失败。

影响：

- 错误输入会先分配 output，再在 decode 阶段失败。
- 预扫描和 decode 的合法性判断不一致。

建议：

- 长度预扫描复用 decode 验证逻辑。
- 或把 UTF-16 decode 改成一次 pass + dynamic buffer，避免双 pass 漂移。

## 字符串与二进制边界

### 问题 9：`hexEncode` 只接受 string，但文档和 LSP 暗示 Bytes

脚本层 `hexEncode(data: string)` 把字符串底层 bytes 编码为 hex。对于任意二进制，应该支持 Bytes。

当前已有 `base64.encodeBytes()`，但 encoding 没有对应 `hexEncodeBytes()`。

建议：

- 明确 `hexEncode` 是文本 bytes 还是通用 bytes。
- 增加 `hexEncodeBytes(bytes)` 或让 `hexEncode` 接受 string/Bytes。
- 同步 builtin/analyzer/LSP 签名。

### 问题 10：`hexDecodeString` 可能返回非 UTF-8 字符串

hex 可表示任意 bytes。`hexDecodeString()` 直接 `xrs_string_value_n()`，不会验证 UTF-8。

影响：

- 如果语言 string 语义应为 UTF-8 文本，该 API 会制造非法 string。
- 如果 string 是 byte string，则命名应更明确。

建议：

- 明确 Xray string 是否允许任意 bytes。
- 若 string 必须 UTF-8，则 `hexDecodeString` 应 validate UTF-8，失败返回 null。
- 推荐二进制场景使用 `hexDecode()` 返回 Bytes。

### 问题 11：`utf16Decode` 接受 string 作为 bytes，语义容易混淆

`utf16Decode(data: any)` 支持 string 和 `Array<uint8>`。传 string 时按 string 底层 bytes 当 UTF-16 数据处理。

建议：

- 脚本层优先只接受 Bytes。
- 如果保留 string 输入，需要文档明确这是“二进制字符串兼容模式”。
- analyzer/LSP 签名必须与实际行为一致。

## UTF-8 / Unicode 语义

### 问题 12：`utf8Valid` 对脚本 string 的价值有限

如果 Xray string 构造时已保证 UTF-8，则 `utf8Valid(str)` 永远 true；如果 string 可包含任意 bytes，则它有价值。

当前 `base64.decode()` 与 `hexDecodeString()` 都可能构造任意 bytes string，因此 `utf8Valid()` 是有用的，但这也说明 string 文本/bytes 语义尚未统一。

建议：

- 明确 string 是否是 UTF-8 文本类型。
- 若不是，考虑引入 Bytes 为二进制唯一载体，逐步减少任意 bytes string。

### 问题 13：UTF-8 count 是 codepoint count，不是 grapheme cluster count

`utf8Count()` 调用 `xr_utf8_strlen()`，统计 Unicode codepoint 数，不处理组合字符和 grapheme cluster。

建议：

- 文档明确是 codepoint count。
- 不建议短期引入 grapheme cluster，避免标准库体量膨胀。

## 测试覆盖

现有覆盖：

- C-level hex encode/decode/valid/roundtrip。
- 底层 Unicode 属性测试。
- 脚本层 hex、UTF-8 基本、UTF-16 LE/BE、CJK、空串、endian 常量。

缺口：

1. UTF-16 surrogate pair，例如 emoji。
2. UTF-16 invalid high surrogate / low surrogate。
3. UTF-16 odd byte length。
4. BOM auto-detect：LE BOM、BE BOM、stripBom=false。
5. `utf16Decode` 第三个参数的 analyzer 与运行时一致性。
6. `hexDecodeString` 对非 UTF-8 bytes 的行为。
7. `hexEncode` 是否应支持 Bytes。
8. C API NULL 输入契约。
9. 长度 overflow 和 `INT32_MAX` 上界。
10. LSP/analyzer signature 同步检查。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | analyzer generated signature 与 builtin 不一致 | 静态检查和运行时行为漂移 | 重新生成并纳入验证 |
| 高 | LSP signature 与 builtin 不一致 | IDE 提示误导用户 | LSP 从同一来源生成或同步 |
| 高 | 长度 API 使用 int 且缺少 overflow guard | 大输入可能截断或分配错误 | 改 size_t/status，补上界检查 |
| 高 | `encoding.h` C API 与 loader 混在一起 | C 复用者被迫引入高层依赖 | 拆分轻量 header |
| 中 | `hexEncode` string/Bytes 语义不统一 | 二进制编码 API 不完整 | 增加 Bytes 入口或统一参数 |
| 中 | `hexDecodeString` 可生成非 UTF-8 string | 文本/二进制边界模糊 | 验证 UTF-8 或推荐 Bytes |
| 中 | `utf16Decode` 接受 string bytes | API 易误解 | 优先接受 Bytes，兼容行为文档化 |
| 中 | UTF-16 length 预扫描校验不完整 | 错误输入会先分配再失败 | 统一预扫描与 decode 校验 |
| 低 | UTF-8 count 名称可能被误解 | 用户可能期待 grapheme count | 文档注明 codepoint count |
| 低 | C API NULL 空输入契约不清 | 调用者难判断错误 | 统一 status/out_len 约定 |

## 后续实施建议

建议把 `encoding` 的后续优化分成三类：

1. 类型同步：修正 builtin/analyzer/LSP 三处签名漂移，建立单一生成源。
2. C API 边界：拆分轻量 header，改进长度与错误契约。
3. 文本/二进制语义：明确 string 是否允许任意 bytes，收敛 hex/UTF-16 API 的 Bytes 使用方式。

补测优先级：

- UTF-16 surrogate pair 与 invalid surrogate。
- BOM auto-detect 与 stripBom。
- analyzer 是否接受 `utf16Decode(bytes, encoding.LE, false)`。
- hex 非 UTF-8 bytes 的脚本行为。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
