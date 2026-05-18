# stdlib/compress 分析与优化建议

## 模块职责

`stdlib/compress` 提供压缩、解压和 checksum 能力：

- gzip：`gzip()` / `gunzip()` / `isGzip()`
- raw deflate：`deflate()` / `inflate()`
- zlib wrapper：`zlibCompress()` / `zlibDecompress()` / `isZlib()`
- checksum：`crc32()` / `adler32()`
- C 层 stream API：用于 HTTP/WS 复用 zlib-backed streaming 与 flush 控制

该模块当前既是 xray 脚本层的压缩 API，也是 HTTP/WS 内部压缩能力的底层入口。后续重构应把“二进制数据类型、错误可观测性、解压输出上限、stream API 暴露策略”作为核心问题处理。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/compress/compress.h` | C API、模块声明、压缩格式和 stream API 声明 |
| `stdlib/compress/compress.c` | 纯 C deflate/inflate、gzip/zlib wrapper、checksum、xray binding |
| `stdlib/compress/compress_zlib.c` | system zlib wrapper、stateful stream、HTTP/WS helper |
| `stdlib/ws/ws_deflate.c` | WS permessage-deflate thin wrapper |
| `stdlib/http/http_client.c` | HTTP Content-Encoding 解压调用方 |
| `tests/regression/10_stdlib/1301_compress.xr` | checksum、gzip/deflate/zlib roundtrip、level、empty/large data 测试 |

## 当前 API

脚本层 builtin：

| API | 签名 | 当前语义 |
|---|---|---|
| `gzip(data, level?)` | `(data: string, level?: int): string?` | gzip compress，返回二进制 string |
| `gunzip(data)` | `(data: string): string?` | gzip decompress |
| `isGzip(data)` | `(data: string): bool` | 仅检查 gzip magic header |
| `deflate(data, level?)` | `(data: string, level?: int): string?` | raw deflate compress |
| `inflate(data)` | `(data: string): string?` | raw deflate decompress |
| `zlibCompress(data, level?)` | `(data: string, level?: int): string?` | zlib wrapper compress |
| `zlibDecompress(data)` | `(data: string): string?` | zlib wrapper decompress |
| `isZlib(data)` | `(data: string): bool` | 检查 zlib header |
| `crc32(data)` | `(data: string): int` | CRC-32 checksum |
| `adler32(data)` | `(data: string): int` | Adler-32 checksum |

C 层额外 API：

- caller-buffer API：`xr_deflate()` / `xr_inflate()` / `xr_gzip()` / `xr_gunzip()` / `xr_zlib_compress()` / `xr_zlib_decompress()`
- allocating API：`xr_gzip_alloc()` / `xr_gunzip_alloc()`
- zlib stream API：`xr_zlib_stream_new_deflate()` / `xr_zlib_stream_new_inflate()` / `xr_zlib_stream_process()` / `xr_zlib_stream_free()`
- WS helper：`xr_deflate_sync_flush()` / `xr_inflate_bounded()`
- HTTP helper：`xr_zlib_gzip_compress()` / `xr_zlib_gzip_decompress()` / `xr_zlib_deflate_compress()` / `xr_zlib_deflate_decompress()`
- content encoding detection：`xr_detect_content_encoding()`

## 当前架构优点

- 纯 C deflate/inflate 不依赖系统 zlib，可用于基础 gzip/zlib wrapper。
- checksum 自实现，覆盖 CRC32 和 Adler32 known value 测试。
- zlib-backed stream API 已统一 WS permessage-deflate 的 `Z_SYNC_FLUSH` 和 HTTP gzip/deflate 的 `Z_FINISH`。
- WS 解压路径通过 `xr_inflate_bounded()` 提供 `max_out`，有 zip-bomb guard。
- 所有 heap 分配使用 `xr_malloc/xr_free/xr_calloc/xr_realloc`。
- 脚本层使用 `xrs_string_value_n()`，可以保存 embedded NUL 的二进制结果。
- gzip 解压会验证 CRC32 和 original size。
- zlib 解压会验证 Adler32。

## 关键边界

### 纯 C codec 与 zlib-backed stream 并存

`compress.c` 实现自研 deflate/inflate 和 gzip/zlib wrapper；`compress_zlib.c` 通过 system zlib 提供 stream 和 HTTP/WS helper。

当前结果是：

- 脚本层 `compress.gzip/deflate/zlibCompress` 走纯 C codec。
- HTTP/WS 内部更偏向 zlib-backed API。
- 两套实现的格式兼容、压缩比、错误码和安全边界需要分别测试。

建议：

- 明确脚本层默认走纯 C 还是 system zlib。
- 若 system zlib 可用，考虑脚本层也复用 zlib-backed API，纯 C codec 作为 fallback。
- 增加交叉兼容测试：纯 C compress → zlib decompress，zlib compress → 纯 C decompress。

## API 类型与工具链同步

### 问题 1：脚本层用 `string` 表示二进制数据

压缩输出是任意 bytes，但 API 暴露为 `string?`。这依赖 Xray string 的 length-aware 能力。

影响：

- 用户容易把压缩结果当 UTF-8 文本处理。
- LSP 中 compress 声明为 `Bytes`，analyzer builtin 是 `string`，二者不一致。
- 如果后续增加真正 Bytes 类型，需要迁移。

建议：

- 短期：文档明确 compress string 是 binary-safe string。
- 中期：引入 `Bytes` 类型或 `Buffer` 类型。
- 同步 analyzer/LSP 签名，避免一个是 `string` 一个是 `Bytes`。

### 问题 2：脚本层错误不可观测

压缩/解压失败统一返回 `null`。底层 `XrCompressError` 有丰富错误码，但脚本层丢失：

- memory
- invalid data
- buffer too small
- invalid header
- checksum mismatch
- stream error

建议：

- 增加 `gunzipDetailed()` / `inflateDetailed()` / `zlibDecompressDetailed()`。
- 或统一 stdlib Result 格式：`{ok, data, error}`。
- checksum mismatch 应该能被用户区分。

### 问题 3：level 参数静默 clamp

脚本层 level 小于 0 改为 0，大于 9 改为 9。

建议：

- 文档明确 clamp 行为。
- strict API 可对非法 level 返回错误。

## 解压安全与内存

### 问题 4：脚本层解压缺少用户可配置 output cap

`inflate()` 和 `zlibDecompress()` 采用：

```text
cap = input_len * 8 + 1024
最多扩容 8 次
```

`gunzip()` 通过 gzip trailer 的 original size 估算，最多扩容 4 次。

风险：

- 对不可信输入，仍可能分配很大内存。
- `gunzip()` 依赖 trailer ISIZE，攻击者可以伪造较大值导致初始分配过大，或伪造较小值导致多次重试。
- 没有用户级 `maxOutputBytes` 参数。

建议：

- 脚本层增加 `maxOutputBytes` option。
- C allocating API 增加 bounded variant：`xr_gunzip_alloc_bounded()` / `xr_inflate_alloc_bounded()` / `xr_zlib_decompress_alloc_bounded()`。
- 默认 cap 应有全局安全上限。

### 问题 5：HTTP zlib decompress helper 没有 output cap

`xr_zlib_gzip_decompress()` 和 `xr_zlib_deflate_decompress()` 动态扩容直到 stream end，没有 `max_out` 参数。

相比之下 WS 路径已经通过 `xr_inflate_bounded()` 使用 `max_message_size` 防 zip bomb。

建议：

- HTTP decompress helper 增加 bounded variant。
- HTTP client 根据响应 body limit 或配置传入 max_out。
- 对 gzip/deflate Content-Encoding 统一使用 bounded API。

### 问题 6：`xr_gzip_original_size()` 只取 32-bit ISIZE

gzip trailer ISIZE 本来就是 mod 2^32。对超过 4GiB 的内容，无法表示真实大小。

建议：

- 文档明确限制。
- 大文件场景使用 streaming 解压，不依赖 ISIZE 一次性分配。

## 格式语义与兼容性

### 问题 7：`isGzip()` 只检查 magic bytes

当前 `xr_is_gzip()` 只判断：

```text
len >= 10 && data[0] == 0x1F && data[1] == 0x8B
```

未检查 method、flags、header 合法性。

建议：

- 保留快速 `isGzip()` 语义但文档说明“looks like gzip”。
- 增加 `validateGzip()` 或让 `isGzip()` 做更完整 header parse。

### 问题 8：`isZlib()` 只检查 header 基本合法性

`isZlib()` 检查 CMF/FLG mod 31 和 compression method，但不检查完整流和 checksum。

建议同 gzip：区分快速探测与完整验证。

### 问题 9：zlib FDICT 被跳过但没有 dictionary 支持

`xr_zlib_decompress()` 看到 FDICT 后跳过 DICTID，然后继续 inflate。

TOML zlib 语义中 FDICT 表示需要 preset dictionary；直接跳过可能导致 inflate 失败或语义不清。

建议：

- 若不支持 preset dictionary，应返回明确错误。
- 不应静默跳过后尝试普通 inflate。

### 问题 10：纯 C deflate 压缩只使用 fixed Huffman

自研 `xr_deflate()` 对压缩输出使用 fixed Huffman，未生成 dynamic Huffman。它能解 dynamic Huffman，但压缩比可能明显不如 zlib。

建议：

- 文档说明压缩比取舍。
- 如果性能/压缩比重要，脚本层可默认改用 zlib-backed compress。
- 或后续实现 dynamic Huffman encoder。

## Stream API 暴露策略

### 问题 11：C 层 stream API 未暴露给脚本层

`compress_zlib.c` 提供 stateful stream，但 xray module 只暴露 one-shot API。

影响：

- 大文件或网络流只能一次性读入。
- 用户无法控制 flush mode。
- WS/HTTP 内部能力比用户 API 更强。

建议：

- 增加脚本层 streaming API 或 Reader/Writer adapter。
- 至少提供 chunked one-shot helpers：`gzipStream()` / `inflateChunks()` 等。
- 明确 flush modes 的安全可用范围。

### 问题 12：header 注释中的历史迁移描述应收敛

`compress.h` 中已有“canonical compression entry point”的长说明，包含迁移计划。当前 WS 已经是 thin wrapper；HTTP 也部分复用 `compress_zlib`。

建议：

- 长期文档应描述当前事实，不保留过期迁移过程。
- 把设计意图压缩到 API doc comment。

## 测试覆盖

现有覆盖：

- CRC32 deterministic 和 known value。
- Adler32 known value。
- gzip/deflate/zlib roundtrip。
- empty data roundtrip。
- large string roundtrip。
- compression level 0..9。
- invalid data detection for `isGzip/isZlib`。
- repeated pattern compression ratio。

主要缺口：

1. gzip checksum mismatch 返回 null / detailed error。
2. zlib Adler32 mismatch。
3. invalid gzip header variants：bad method、bad flags、truncated optional fields。
4. invalid zlib FDICT dictionary case。
5. corrupt deflate stream fuzz cases。
6. high compression ratio zip-bomb cap。
7. script API binary string with embedded NUL。
8. C stream API unit tests：Z_SYNC_FLUSH、Z_FINISH、多 chunk feed。
9. HTTP gzip/deflate bounded decompression。
10. WS permessage-deflate trailer strip/append edge cases。
11. pure C codec 与 system zlib 交叉兼容。
12. level clamp 边界：negative、>9。
13. 4GiB ISIZE wrap 行为文档或测试。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | 脚本层解压没有 output cap | 不可信输入可导致大内存分配 | 增加 maxOutputBytes / bounded API |
| 高 | HTTP decompress helper 无 cap | HTTP Content-Encoding 有 zip-bomb 风险 | bounded helper + response limit |
| 高 | 错误码被脚本层吞掉 | 用户无法区分 checksum/header/data error | detailed Result API |
| 中 | `string` 承载二进制与 LSP `Bytes` 不一致 | 类型和用户认知漂移 | 引入 Bytes 或同步声明 |
| 中 | FDICT 被跳过但不支持 dictionary | zlib 语义不明确 | 明确返回 unsupported dictionary |
| 中 | `isGzip/isZlib` 是浅检测 | 用户误以为完整验证 | 文档化或增加 validate API |
| 中 | 纯 C 与 zlib-backed 两套 codec | 行为/性能/安全边界漂移 | 统一默认路径或交叉测试 |
| 中 | stream API 未暴露脚本层 | 大文件只能 one-shot | chunk/stream API |
| 低 | level 静默 clamp | 参数错误不可见 | strict validation 或文档化 |
| 低 | long migration note 可能过期 | 维护者误解当前事实 | 收敛为事实性 API 注释 |

## 后续实施建议

建议优先做安全与可观测性：

1. 为所有脚本层解压 API 增加 `maxOutputBytes` options。
2. 增加 detailed API 暴露 `XrCompressError`。
3. HTTP gzip/deflate 解压改用 bounded helper。
4. 明确 `string` 是 binary-safe，并同步 analyzer/LSP 类型。
5. 增加 pure C 与 zlib-backed 交叉兼容测试。
6. 为 C stream API 增加单元测试。
7. 明确 FDICT、gzip header validation、`isGzip/isZlib` 浅检测语义。
8. 评估脚本层是否应默认使用 system zlib 以获得更好压缩比和兼容性。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
