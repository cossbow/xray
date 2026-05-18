# stdlib/base64 分析与优化建议

## 模块职责

`stdlib/base64` 提供 RFC 4648 Base64 编解码能力：

- 标准 Base64 encode/decode
- URL-safe Base64 encode/decode
- 字节数组 encode/decode
- Base64 字符串合法性检查
- C-level API，供 `http` / `ws` 等模块复用

该模块同时承担“脚本层 stdlib 模块”和“C 内部工具库”两种职责，因此头文件依赖边界和 C API 契约需要比普通模块更清晰。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/base64/base64.c` | 编解码核心、脚本 binding、loader |
| `stdlib/base64/base64.h` | C-level API 和 loader 声明 |
| `tests/unit/encoding/test_base64.c` | C-level API 单测 |
| `tests/regression/10_stdlib/1180_base64_basic.xr` | 脚本层回归测试 |
| `stdlib/ws/ws.c` | 使用 `xr_base64_encode()` 生成 WebSocket key |
| `stdlib/http/http_proxy.c` | 使用 `xr_base64_encode()` 生成 proxy auth header |

## 当前 API

### C-level API

| 函数 | 当前语义 |
|---|---|
| `xr_base64_encode(data, len, out_len)` | 标准 Base64，带 padding |
| `xr_base64_encode_url(data, len, out_len)` | URL-safe Base64，不带 padding |
| `xr_base64_decode(data, len, out_len)` | 解码，接受标准和 URL-safe 字符 |
| `xr_base64_decode_url(data, len, out_len)` | 直接复用标准 decode |
| `xr_base64_is_valid(data, len)` | 检查长度和字符是否合法 |

### 脚本层 API

| API | 当前语义 |
|---|---|
| `base64.encode(str)` | 字符串编码为标准 Base64 |
| `base64.decode(str)` | Base64 解码为字符串，失败返回 null |
| `base64.encodeUrl(str)` | 字符串编码为 URL-safe Base64，无 padding |
| `base64.decodeUrl(str)` | URL-safe Base64 解码为字符串，实际等同 decode |
| `base64.encodeBytes(bytes)` | 字节数组编码为标准 Base64 |
| `base64.decodeToBytes(str)` | Base64 解码为 `Bytes`，失败返回 null |
| `base64.isValid(str)` | 判断是否可被当前 decoder 接受 |

## 已有测试覆盖

### C-level 单测

`tests/unit/encoding/test_base64.c` 覆盖：

- RFC 4648 基础 encode 向量
- decode 基础向量
- roundtrip
- URL-safe encode/roundtrip
- validation
- 0..255 binary data roundtrip

### 脚本层回归

`1180_base64_basic.xr` 覆盖：

- encode/decode 基础与空串
- 中文字符串 roundtrip
- URL-safe encode/decode
- invalid decode 返回 null
- len%4 边界
- no-padding decode
- encodeBytes/decodeToBytes
- 长字符串 roundtrip

整体覆盖优于不少小型 stdlib 模块。

## 依赖与架构边界

### 问题 1：C API 没有 `XR_FUNC` 修饰

`base64.h` 暴露多个非 static C 函数，但声明没有 `XR_FUNC`：

- `xr_base64_encode`
- `xr_base64_encode_url`
- `xr_base64_decode`
- `xr_base64_decode_url`
- `xr_base64_is_valid`

`base64.c` 中定义也没有 `XR_FUNC`。

影响：

- 违反项目非 static 函数可见性规范。
- 未来动态库符号导出和静态检查可能失败。

建议：

- C-level API 声明和定义统一加 `XR_FUNC`。
- loader `xr_load_module_base64()` 也统一加 `XR_FUNC`。

### 问题 2：`base64.h` 同时承担 core API 与 module loader，依赖过重

`base64.h` 前半部分只是 C-level API，但后半部分 include 了：

- `xmodule.h`
- `xvm.h`
- `xvalue.h`
- `xstring.h`
- `xarray.h`
- `xgc.h`
- `xmalloc.h`

这导致 `ws.c`、`http_proxy.c` 只想使用 `xr_base64_encode()`，却被迫引入 VM/runtime/module 依赖。单测甚至选择 forward declare C API 来避免 include 该 header。

建议：

- 拆分为轻量 C API header 和模块 loader header。
- C API header 只依赖 `xdefs.h`、`stddef.h`、`stdbool.h`。
- loader 声明由 `xmodule_loaders.h` 或模块私有 header 管理。

### 问题 3：`base64.c` include VM internal 头

`base64.c` include `src/vm/xvm_internal.h`，但当前实现没有直接使用 VM internal 类型。

建议移除，降低 stdlib 与 VM internal 耦合。

## 内存与生命周期

### 当前正确点

- 编解码临时 buffer 都使用 `xr_malloc/xr_free`。
- 脚本层返回字符串时使用 `xrs_string_value_n()`，随后释放临时 buffer。
- decodeToBytes 在数组创建失败时会释放 decoded buffer。

### 问题 4：长度计算缺少 overflow 检查

encode 长度计算：

```c
size_t encoded_len = ((len + 2) / 3) * 4;
```

当 `len` 接近 `SIZE_MAX` 时，`len + 2` 和后续乘法可能溢出。

decode 也有：

```c
unsigned char *output = xr_malloc(decoded_len + 1);
```

`decoded_len + 1` 也缺少上界检查。

建议：

- 增加 `SIZE_MAX` 上界检查。
- 对过大输入返回 NULL/null。
- 对 `decodeToBytes` 还要检查 `out_len <= INT32_MAX`。

### 问题 5：C API 对 NULL 输入的契约不明确

当前 C API 不检查：

- `data == NULL && len > 0`
- `out_len` 在失败时是否写 0

`len == 0` 时即使 `data == NULL` 可能不会崩溃，但契约没有写清。

建议：

- 定义 `data == NULL && len == 0` 是否合法。
- `data == NULL && len > 0` 应返回 NULL。
- 所有失败路径将 `*out_len = 0`。

### 问题 6：`decodeToBytes` 长度强转为 int32

```c
XrArray *arr = xr_array_bytes_new(xr_current_coro(X), (int32_t) out_len);
```

如果 decoded 输出超过 `INT32_MAX`，强转会截断。

建议：

- 在创建 Bytes 前检查 `out_len <= INT32_MAX`。
- 对超大输入返回 null 或错误。

## 编解码语义问题

### 问题 7：standard decode 和 URL-safe decode 实际没有区分

decode table 同时接受：

- 标准 `+` / `/`
- URL-safe `-` / `_`

因此：

- `xr_base64_decode()` 接受 URL-safe 输入。
- `xr_base64_decode_url()` 直接复用标准 decode。
- `isValid()` 也接受两种 alphabet。

这可能是有意的 lenient decoder，但当前 API 名称让人以为 standard 和 URL-safe 是两个模式。

建议二选一：

- 明确当前是 lenient decoder，文档写清两种 alphabet 都接受。
- 或拆成 strict standard / strict URL-safe / lenient 三种策略。

### 问题 8：padding 校验偏宽松

当前允许：

- 无 padding 输入，如 `QQ`
- 非 canonical padding，如 `Zg=`

单测里也注释说明 `Zg=` 被接受。

这对兼容性友好，但如果 `isValid()` 被用户理解为 RFC canonical validation，就会误判。

建议：

- 明确 `isValid()` 是“当前 decoder 可接受”还是“canonical Base64”。
- 如需严格验证，增加 `isCanonical()` 或 strict 参数。

### 问题 9：没有校验尾部 unused bits

对于 tail 长度为 2 或 3 的输入，Base64 canonical encoding 要求未使用的低位为 0。当前 decoder 没有检查这些 pad bits。

影响：

- 多个非 canonical 字符串可能解码为同一字节序列。
- 如果用于签名、token 或安全敏感校验，canonical ambiguity 可能成为问题。

建议：

- 如果保留 lenient decode，至少 strict validation 应检查 pad bits。
- 文档明确当前不做 canonical 校验。

### 问题 10：whitespace 处理未定义

当前 decoder 不接受空白字符。MIME Base64 常见输入会包含换行。

建议：

- 明确 stdlib/base64 是否只实现 RFC 4648 raw form。
- 如需 MIME 兼容，增加 `decodeMime()` 或配置项，而不是让默认 decode 隐式忽略空白。

## 字符串与二进制边界

### 问题 11：`decode()` 返回 string，但 Base64 可表示任意 bytes

`base64.decode()` 把 decoded bytes 用 `xrs_string_value_n()` 转为字符串。这可以保存长度，但语义上存在问题：

- decoded bytes 可能包含 `\0`。
- decoded bytes 可能不是有效 UTF-8。
- 如果 Xray string 被语言层假定为文本，这会破坏文本语义。

当前已有 `decodeToBytes()`，它更适合任意二进制数据。

建议：

- 明确 `decode()` 只适用于文本 Base64。
- 推荐二进制使用 `decodeToBytes()`。
- 增加测试：`base64.decode("AA==")` 的长度和行为。
- 如果语言 string 必须 UTF-8，`decode()` 应验证 UTF-8 或废弃。

### 问题 12：`encodeBytes()` 对非 Bytes 数组静默截断/填 0

`encodeBytes()` fast path 支持 `XR_ELEM_U8`。slow path 对普通数组：

- int 强转为 unsigned char
- 非 int 当 0
- 超出 0..255 的 int 被截断

但 signature 是 `Array<uint8>`。

建议：

- 如果 API 只支持 Bytes，就拒绝非 `XR_ELEM_U8`。
- 如果支持普通数组，则校验每个元素是 0..255 的 int。
- 非法元素应返回 null 或错误，不应静默变 0。

## 测试缺口

已有测试覆盖较好，但还缺：

1. C API NULL 输入契约。
2. encode/decode 超大长度 overflow 防护。
3. `decodeToBytes` 超过 int32 长度的保护。
4. strict vs lenient padding 行为：`Zg=`, `Zg===`, `QQ`, `QQ==`。
5. pad bits canonical 校验。
6. standard decoder 是否应接受 `-` / `_`。
7. URL-safe decoder 是否应接受 `+` / `/`。
8. whitespace 输入是否被拒绝。
9. `decode()` 返回含 NUL bytes 的字符串行为。
10. `encodeBytes()` 非 Bytes 数组、负数、>255、非 int 元素。
11. header self-contained 编译，尤其是 C API header 不应拉入 VM。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | C API 与 loader 缺少 `XR_FUNC` | 违反可见性规范 | 声明和定义统一修饰 |
| 高 | `base64.h` 依赖 VM/runtime 过重 | C 复用者被迫引入高层依赖 | 拆分轻量 C API header |
| 高 | 长度计算缺少 overflow 检查 | 极端输入可能分配错误 | 增加 size 上界检查 |
| 中 | standard/url decoder 没有区分 | API 语义不清 | 定义 lenient/strict 策略 |
| 中 | padding 与 pad bits 校验偏宽松 | canonical validation 不可靠 | 增加 strict validation 或文档化 |
| 中 | `decode()` 返回任意 bytes string | 文本/二进制边界模糊 | 推荐 bytes API，定义 UTF-8 策略 |
| 中 | `encodeBytes()` slow path 静默截断 | 数据损坏不报错 | 拒绝非法元素 |
| 中 | `decodeToBytes` int32 截断风险 | 大输入可能错误分配 | 检查 out_len 上限 |
| 低 | `base64.c` include VM internal | 不必要耦合 | 删除 include |
| 低 | C API 失败时 out_len 未清零 | 调用者易误用旧值 | 失败路径写 0 |

## 后续实施建议

建议把 `base64` 作为“C API 与 stdlib binding 拆分”的试点：

1. 拆分轻量 C API header，保留 `xr_base64_*` 给 `ws/http` 使用。
2. C API 和 loader 统一加 `XR_FUNC`。
3. 增加 NULL、overflow、out_len 失败契约。
4. 明确 lenient vs strict decode/validation 策略。
5. 收紧 `encodeBytes()` 参数校验。
6. 为 `decode()` 文本语义与 `decodeToBytes()` 二进制语义补文档和测试。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
