# stdlib/crypto 分析与优化建议

## 模块职责

`stdlib/crypto` 提供哈希、HMAC、安全随机数、UUID、AES-CBC 加解密和常量时间比较能力：

- hash：`md5()` / `sha1()` / `sha256()` / `sha512()`
- HMAC：`hmac(algo, key, data)`
- random：`randomBytes(n)` / `uuid()`
- encryption：`encrypt(key, plaintext)` / `decrypt(key, ciphertext)`
- comparison：`timingSafeEqual(a, b)`

当前实现是纯 C 算法实现，不依赖 OpenSSL/libsodium。随机数通过 `src/os/os_random` 走系统 CSPRNG。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/crypto/crypto.h` | C API、context struct、脚本层 API 文档 |
| `stdlib/crypto/crypto.c` | MD5/SHA1/SHA256/SHA512、HMAC、AES-CBC、hex、module binding |
| `src/os/os_random.h` | cross-platform CSPRNG API |
| `src/os/unix/random_unix.c` | macOS/BSD `arc4random_buf`，Linux `getrandom` + `/dev/urandom` fallback |
| `src/os/win/random_win.c` | Windows `BCryptGenRandom` |
| `tests/unit/stdlib/test_crypto.c` | C 层 hash/HMAC/AES/random/hex 单元测试 |
| `tests/regression/10_stdlib/1400_crypto_hash.xr` | 脚本层 MD5/SHA1/SHA256 测试 |
| `tests/regression/10_stdlib/1401_crypto_hmac.xr` | 脚本层 HMAC 测试 |
| `tests/regression/10_stdlib/1402_crypto_random.xr` | randomBytes / UUID 测试 |
| `tests/regression/10_stdlib/1403_crypto_sha512.xr` | SHA512 / HMAC-SHA512 测试 |
| `tests/regression/10_stdlib/1404_crypto_aes.xr` | AES-CBC encrypt/decrypt 测试 |
| `tests/regression/10_stdlib/1405_crypto_utils.xr` | timingSafeEqual 和 hash length 测试 |

## 当前 API

| API | 当前签名 | 返回 |
|---|---|---|
| `md5(data)` | `(data: string): string` | 32 chars lowercase hex |
| `sha1(data)` | `(data: string): string` | 40 chars lowercase hex |
| `sha256(data)` | `(data: string): string` | 64 chars lowercase hex |
| `sha512(data)` | `(data: string): string` | 128 chars lowercase hex |
| `hmac(algo, key, data)` | `(algo: string, key: string, data: string): string` | lowercase hex |
| `randomBytes(n)` | `(n: int): string` | n bytes 的 lowercase hex，长度为 `n * 2` |
| `uuid()` | `(): string` | UUID v4 string |
| `encrypt(key, plaintext)` | `(key: string, plaintext: string): string` | hex(iv + ciphertext) |
| `decrypt(key, ciphertext)` | `(key: string, ciphertext: string): string?` | plaintext 或 null |
| `timingSafeEqual(a, b)` | `(a: string, b: string): bool` | constant-time-ish equality |

## 当前实现特点

- hash 和 HMAC 都是纯 C 实现。
- hash 输出固定为 lowercase hex，不支持 raw bytes/base64 输出。
- HMAC 支持 `md5` / `sha1` / `sha256` / `sha512`。
- `randomBytes(n)` 使用系统 CSPRNG，但脚本层返回 hex string，不是 raw bytes。
- `uuid()` 使用 16 字节 CSPRNG，正确设置 v4 和 variant bits。
- `encrypt()` 使用 AES-256-CBC + PKCS7 padding，key 为 `SHA256(user_key)`，IV 随机并 prepended。
- `decrypt()` 做 hex decode、AES-CBC decrypt、PKCS7 padding 检查。
- `xr_secure_wipe()` 用平台 API 或 volatile fallback 清理 key/context 等敏感栈数据。

## 架构优点

- 随机源统一走 `xr_random_bytes()`，且失败时 abort，避免继续使用低熵/部分随机数据。
- hash/HMAC 覆盖常见标准向量。
- AES-CBC 基础 roundtrip 和 padding 边界有回归测试。
- key schedule、AES context、derived key 在 encrypt/decrypt 后有 wipe。
- `timingSafeEqual()` 提供 HMAC 校验的基本工具。
- 所有脚本层 hash/HMAC 输出采用明确的 lowercase hex 表示。

## 关键问题

### 问题 1：LSP crypto 符号严重落后

analyzer builtin 已声明 10 个函数：

```text
md5 sha1 sha256 sha512 hmac randomBytes uuid encrypt decrypt timingSafeEqual
```

但 LSP 只暴露：

```text
md5 sha1 sha256 randomBytes
```

同时 LSP 将 `randomBytes` 返回类型标为 `Bytes`，而 analyzer 和实现返回 `string`。脚本层测试也确认返回 hex string，长度为 `2 * n`。

影响：

- 补全/hover 不完整。
- 静态分析与 LSP 类型不一致。
- 用户会误以为 `randomBytes()` 返回 raw bytes。

建议：

- 同步 LSP：补齐 `sha512/hmac/uuid/encrypt/decrypt/timingSafeEqual`。
- 将 `randomBytes` 说明改为 hex string，或实际引入 Bytes 类型。

### 问题 2：`test_crypto.c` 的 `xr_random_bytes` 前向声明过期

`src/os/os_random.h` 当前声明：

```c
XR_FUNC void xr_random_bytes(unsigned char *buf, size_t len);
```

但单元测试中前向声明为：

```c
int xr_random_bytes(uint8_t *buffer, size_t len);
```

并且测试断言返回 0。

影响：

- C 调用约定上返回值未定义，测试可能读取垃圾寄存器。
- 该测试无法可靠验证 CSPRNG API。

建议：

- 测试包含 `src/os/os_random.h`，不要手写过期 prototype。
- `xr_random_bytes()` 是 abort-on-failure API，测试应只调用并检查输出 buffer 变化。

### 问题 3：公开 C 函数缺少 `XR_FUNC` 修饰符

`crypto.h` 中大量非 static 函数声明未带 `XR_FUNC`，例如：

```c
void xr_md5_init(...);
void xr_sha256(...);
void xr_hmac_sha256(...);
void xr_aes_init(...);
```

这不符合项目可见性规则。

建议：

- 给所有需要跨编译单元调用的 `xr_*` crypto C API 添加 `XR_FUNC`。
- 不需要外部调用的 helper 保持 `static`。

### 问题 4：MD5/SHA1 仍以普通 API 暴露

MD5 和 SHA1 已不适合安全用途。虽然 header 有提示，但脚本层 API/hover 不一定会提示风险。

建议：

- 文档和 LSP 明确标注 “compatibility only”。
- 新 API 推荐 `sha256/sha512`。
- 如有包管理/签名用途，禁止使用 MD5/SHA1。

### 问题 5：AES-CBC 无认证，不适合作为默认安全加密 API

`crypto.encrypt()` 当前是：

```text
AES-256-CBC + PKCS7 + random IV + hex(iv+ciphertext)
```

但没有 HMAC 或 AEAD tag。

风险：

- CBC ciphertext 可被篡改。
- decrypt 根据 padding 返回 null，若暴露给远端，可形成 padding oracle 侧信道。
- 用户会误以为 `encrypt()` 是现代安全加密。

建议：

- 新增 authenticated encryption API：AES-GCM 或 ChaCha20-Poly1305。
- 如果继续使用 CBC，应采用 encrypt-then-MAC：HMAC-SHA256(iv+ciphertext)。
- 将当前 `encrypt/decrypt` 标注为 compatibility/educational，或改名为 `encryptCbcUnsafe`。

### 问题 6：key derivation 只是 SHA-256(key)

`encrypt()` 将用户 key 直接 SHA-256 得到 AES-256 key，没有 salt，也没有 PBKDF2/scrypt/Argon2。

影响：

- 用户传入 password 时抗暴力破解能力弱。
- 相同 password 永远派生同一 key。

建议：

- 提供 KDF：PBKDF2-HMAC-SHA256 至少应支持；更推荐 Argon2id/scrypt。
- ciphertext 格式包含 KDF 参数、salt、iv、tag。

### 问题 7：`randomBytes()` 返回 hex string 且最大 1024 字节

实现限制：

```text
n <= 0 或 n > 1024 返回 null
```

输出是 hex string，不是 raw bytes。

建议：

- 文档明确 `randomBytes(n)` 返回 hex 编码。
- 如果引入 Bytes 类型，提供 `randomBytesRaw(n): Bytes`。
- 支持 `randomHex(n)` 命名更准确。
- 对 `n > 1024` 的限制给出错误或文档说明。

### 问题 8：`hmac()` 使用 `strcmp` 比较算法名，依赖 NUL-terminated string

脚本层 string 是 length-aware，但 `XR_STRING_CHARS(algo)` 传给 `strcmp()`。如果算法名包含 embedded NUL，比较行为会截断。

影响：

- `"sha256\0x"` 可能被当作 `sha256`。
- 和其他 length-aware API 语义不一致。

建议：

- 算法名比较使用 length-aware match。
- 对未知算法返回 detailed error 或 null。

### 问题 9：`xr_hex_to_bytes()` 使用 `strlen()`

C API `xr_hex_to_bytes(const char *hex, ...)` 使用 NUL-terminated 输入。脚本层 decrypt 在传入 `XR_STRING_CHARS(cipher_hex_str)` 前已用 `cipher_hex_str->length` 做长度检查，但 decode 阶段仍可能因 embedded NUL 截断。

建议：

- 增加 `xr_hex_to_bytes_n(hex, hex_len, output, max_len)`。
- 脚本层 decrypt 使用 length-aware decode。

### 问题 10：`timingSafeEqual()` 长度不同仍泄漏长度相关时间

实现长度不等时只比较 `min_len`，循环次数仍取决于较短长度。

如果长度本身是 secret，这不是完全常量时间。

建议：

- 文档说明长度不同会返回 false，比较时间与较短长度相关。
- 对 HMAC hex 字符串这种固定长度用途是可接受的。
- 若要严格 constant-time，提供固定长度 buffer 比较 API。

### 问题 11：HMAC 中间 buffer 分配失败时返回全零 digest

`hmac_compute()` 在 `xr_malloc(inner_len)` 失败时：

```c
memset(digest, 0, digest_size);
return;
```

脚本层无法区分真实 digest 与内存失败。虽然全零 HMAC 极不可能，但错误应该可观测。

建议：

- HMAC C API 返回 bool/error code。
- 脚本层内存失败返回 null 或 abort。

### 问题 12：敏感 heap buffer 释放前未全部 wipe

`encrypt()` 会 wipe `aes_key` 和 ctx，但 heap 上的 padded plaintext、ciphertext、hex output 在 free 前没有 wipe。`decrypt()` 的 raw/cipher/plain buffer 也没有全部 wipe。

影响：

- plaintext/key-adjacent material 可能留在 heap。

建议：

- 对 padded plaintext、decrypted plaintext临时 buffer、raw ciphertext buffer 在释放前 wipe。
- 返回给用户的 plaintext/ciphertext string 不应 wipe，但中间 buffer 应 wipe。

## 测试覆盖

现有覆盖：

- MD5/SHA1/SHA256/SHA512 known vectors。
- HMAC-MD5/SHA1/SHA256/SHA512 基本向量和长度。
- AES-CBC encrypt/decrypt roundtrip、empty、long、exact block、wrong key、random IV。
- randomBytes 长度、唯一性、UUID 格式和唯一性。
- timingSafeEqual equal/different/different length/HMAC verify。
- C 层 AES-CBC roundtrip、hash vectors、random、hex。

主要缺口：

1. `randomBytes(0)`、负数、`>1024` 的脚本层边界。
2. `uuid()` version nibble 和 variant bits 显式断言。
3. HMAC RFC 4231 多组向量，尤其 long key、key length > block size。
4. `hmac()` unknown algorithm、大小写算法名、embedded NUL 算法名。
5. `decrypt()` invalid hex、odd length、truncated IV、tampered ciphertext、tampered padding。
6. AES-CBC malleability风险缺少文档/测试。
7. `timingSafeEqual()` 固定长度 HMAC 场景之外的语义说明。
8. C API visibility 检查。
9. `xr_hex_to_bytes()` length-aware variant。
10. LSP/analyzer signature 同步测试。
11. CSPRNG failure 无法单元测试，但 prototype 应至少正确。
12. 需要交叉验证 hash/HMAC/AES 与成熟库输出。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | AES-CBC 无认证 | ciphertext 可篡改，可能 padding oracle | AEAD 或 encrypt-then-MAC |
| 高 | test 中 `xr_random_bytes` 原型错误 | 测试结果未定义 | 包含正确 header，移除返回值断言 |
| 高 | LSP crypto 只暴露部分 API | 补全和 hover 严重缺失 | 同步全部 builtin |
| 中 | `randomBytes` 实际返回 hex string，LSP 标 Bytes | 类型漂移 | 改 LSP 或引入 Bytes |
| 中 | C API 缺少 `XR_FUNC` | 违反可见性规则 | 给导出 C API 加修饰符 |
| 中 | key derivation 仅 SHA256(password) | password 加密抗暴力弱 | PBKDF2/scrypt/Argon2id |
| 中 | length-unaware `strcmp/strlen` | embedded NUL 语义漂移 | 使用 length-aware helpers |
| 中 | HMAC 分配失败返回全零 digest | 错误不可观测 | 返回 error/null |
| 中 | 中间敏感 heap buffer 未 wipe | 明文残留风险 | free 前 secure wipe |
| 低 | MD5/SHA1 普通暴露 | 用户误用 | LSP/文档标 compatibility only |

## 后续实施建议

建议优先顺序：

1. 修复 `test_crypto.c` 中 `xr_random_bytes` 原型和测试方式。
2. 同步 LSP crypto 符号和 `randomBytes` 类型描述。
3. 给 crypto C API 补 `XR_FUNC`。
4. 增加 length-aware `hmac` algorithm match 和 hex decode。
5. 给 `decrypt()` 增加更多 invalid/tampered 测试。
6. 将当前 AES-CBC API 标注为 unauthenticated，并新增 AEAD 或 encrypt-then-MAC API。
7. 为 `randomBytes()` 明确命名/返回格式，规划 Bytes 类型。
8. 对中间敏感 heap buffer 做 secure wipe。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
