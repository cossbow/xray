# stdlib「加解密 / 压缩 / 正则」分组源码分析

> **分组**：`crypto`、`compress`、`regex`
> **代码规模**：约 9.4K 行 C（crypto 1150、compress 1271、regex 7 个文件共 ~7000）
> **定位**：三者都是「纯算法 + 少量 IO」，没有协程阻塞路径，天然更容易做 `xray` 风格的封装；但这组是**全仓库红线违反最严重、最不符合统一可见性规范**的一片。

---

## 0. 质量梯度（一眼结论）

| 模块 | 内存红线 | 可见性 | 并发安全 | 协议/规范完整度 | 实现质量 |
|------|---------|--------|----------|----------------|----------|
| `regex` | ✅ 全走 `xr_malloc`（`xr_re_alloc` 封装） | ❌ | ❌ DFA cache 有 data race | 中（缺 `\u`、`x` 标志、Unicode `.`） | 高（RE2 启发，DFA+NFA+OnePass） |
| `compress` | ❌ ~25 处 `malloc/free` | ❌ | ✅ 无状态 | 中（只有 fixed Huffman 编码；没有动态 Huffman） | 中高 |
| `crypto` | ❌ ~15 处 `malloc/free` | ❌ | ✅ 无状态 | 中（只有 CBC，无 GCM/CTR/AEAD） | 高（算法正确，wipe/constant-time 到位） |

三个模块共同的结构性问题：
1. **全部没有** `XRAY_API` / `XR_FUNC` 修饰符；
2. **crypto/compress 还在直接用 `malloc/free`**（红线硬违反）；
3. **错误全部静默返回 `null`/`false`/`0`**，脚本层无法拿到原因；
4. **全部没有流式 API**（大文件/大字符串只能一次性内存装载）；
5. **测试覆盖全部集中在「Happy path」**，边界/对抗/性能压测缺失。

---

## 1. `crypto` 模块（~1150 行）

### 1.1 文件结构

```
crypto.h          224 行   纯算法公共接口 + AES context + 模块入口
crypto.c         1150 行   MD5/SHA1/SHA256/SHA512/HMAC/AES-CBC/随机数/HEX/模块绑定
```

### 1.2 做得好的地方

- **算法是从零实现的纯 C**，无 OpenSSL 依赖，可移植性强；构造和常量表都和 FIPS/RFC 对齐。
- **`secure_wipe`** 做了平台分派（`memset_s` / `explicit_bzero` / `SecureZeroMemory` / `volatile` 回退），符合现代密码学最佳实践。
- **`crypto_decrypt` 的 PKCS7 去填充是常量时间实现**（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:1031-1043`），避免 padding oracle 攻击；这是整个仓库里密码学素养最高的一段。
- **`crypto_timing_safe_equal`**（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:1063-1080`）用 `volatile uint8_t diff` 防止编译器短路，思路正确。
- **随机数源**（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:755-777`）按平台选择 `BCryptGenRandom` / `arc4random_buf` / `getrandom` / `/dev/urandom`，都是 CSPRNG 级别。
- **`XR_DCHECK`** 在 update 入口做了参数校验（md5/sha256/aes_init/bytes_to_hex）。

### 1.3 问题清单

#### 🔴 硬红线

| # | 位置 | 问题 |
|---|------|------|
| C-1 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:480` | `hmac_compute` 用 `malloc(inner_len)` + `free(inner_buf)` —— 应统一 `xr_malloc/xr_free` |
| C-2 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:946,953,964` | `crypto_encrypt` 三处 `malloc`（padded/cipher/hex buffer） |
| C-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:966-980` | `crypto_encrypt` 五处 `free(...)`（分散、嵌套、容易漏清理） |
| C-4 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:1004,1021` | `crypto_decrypt` 两处 `malloc` |
| C-5 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:1008,1023,1047-1048,1057-1058` | `crypto_decrypt` 多分支 `free(...)` |
| C-6 | 全部公共 `xr_*` 函数 | 无 `XRAY_API`/`XR_FUNC` 修饰（`xr_md5`、`xr_sha256`、`xr_hmac_*`、`xr_aes_init`、`xr_aes_cbc_*`、`xr_random_bytes`、`xr_bytes_to_hex`、`xr_hex_to_bytes` —— 共 ~28 个） |

#### 🟠 严重（语义/安全）

| # | 位置 | 问题 | 建议 |
|---|------|------|------|
| C-7 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:927-982` `crypto.encrypt` | **密钥派生用单轮 SHA-256**，离字典攻击只差一次 GPU hash；同时**没有 MAC / 认证标签**，属于"加密但未认证"，对 bit-flipping 攻击毫无抵抗。业界 10 年前已经不推荐 | 1）引入 PBKDF2/scrypt/argon2 做 KDF；2）改用 **AES-256-GCM**（或至少 Encrypt-then-MAC：AES-CBC + HMAC-SHA256）；3）旧 API 保留但打 deprecation warning |
| C-8 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:894-903` `randomBytes` | 硬上限 **1024 字节**；超过 1 次系统调用就得不到。对生成密钥对、token、nonce 有限制 | 调到 64KB 或做流式接口 |
| C-9 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:844-882` `crypto_hmac` | 算法分派用 4 次 `strcmp` —— 性能其次，真正问题是**大小写敏感**、不支持 `"SHA256"` | 用小 enum 或 `xr_hash_bytes64` 查 2 字节枚举表 |
| C-10 | 整个模块 | **只暴露 AES-CBC，没有 AES-GCM / CTR / ECB** —— 现代协议（TLS 1.3、AGE、libsodium-secretbox）都强依赖 AEAD | 实现 AES-GCM；同时把 block cipher 原语单独导出 |
| C-11 | 整个模块 | 无**流式 Hash API 到脚本层**（C 有 `xr_sha256_init/update/final`，但 binding 不暴露） | 暴露 `crypto.Sha256()` / `.update(chunk)` / `.hex()` |
| C-12 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:641-709` `aes_encrypt_block` / `aes_decrypt_block` | **软件 S-box 查表 → cache-timing 旁路可见**。服务器场景共享机器被敌手在同一 CPU 上做 FLUSH+RELOAD 即可回收密钥。纯 C 没法根治，但至少**文档要警告** | 文档里加 "Not recommended for server-side bulk encryption against side-channel adversaries"；未来检测 AES-NI 走硬件路径 |

#### 🟡 中等

| # | 位置 | 问题 | 建议 |
|---|------|------|------|
| C-13 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:279`、`388` | `xr_sha1_update`、`xr_sha512_update` 没有 `XR_DCHECK`（和 `md5_update`/`sha256_update` 不一致） | 补上 |
| C-14 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:766-774` `/dev/urandom` 读取 | **不处理 `EINTR`**；在信号密集场景失败返回 `-1` | `n < 0 && errno == EINTR` 时继续 |
| C-15 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:905-919` `crypto_uuid` | 失败静默返回 `null`；用户区分不出"平台随机数坏了"和"参数错误" | 抛出 runtime error，或设置错误字段 |
| C-16 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:593-631` `xr_aes_init` | `default: memset(ctx, 0, ...); return;` 前面已 `XR_DCHECK` key_bits，release 模式走 default 导致 `ctx->rounds=0`，后续 `aes_encrypt_block` 会跑 0 轮，悄悄返回明文 | 返回错误码 或 `return -1` 让调用者判空 |
| C-17 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/crypto/crypto.c:1063-1080` `timing_safe_equal` | 循环次数 = `min_len`，对长度不等的输入比对耗时和 `min_len` 成正比 → **泄漏较短字符串的长度** | 若严格要求恒定时间，应填充到固定大小或只允许等长比较 |
| C-18 | `xr_hex_to_bytes` | 解码速率不是常量时间（`hex_digit` 分支）；解码 secret material（如 AES key）时有泄漏风险 | 改成查表常量时间版本 |

### 1.4 测试覆盖

- ✅ `1400_crypto_hash.xr` / `1401_crypto_hmac.xr` / `1402_crypto_random.xr` / `1403_crypto_sha512.xr` / `1404_crypto_aes.xr` / `1405_crypto_utils.xr` 覆盖基本路径；
- ✅ `tests/unit/stdlib/test_crypto.c` 单元测试。

缺口：
- ❌ 官方 NIST 测试向量（SHA、AES-CAVP）对照
- ❌ AES-CBC 多块 + IV 对齐 边界
- ❌ HMAC RFC 4231 向量
- ❌ 10⁶ 次 PKCS7 正确/错误 padding 的统计学侧信道测试
- ❌ UUID v4 RFC 4122 variant 字段验证
- ❌ 大输入（>4GB）的 `xr_md5_update`（`count[0]` 回绕）

---

## 2. `compress` 模块（~1271 行）

### 2.1 文件结构

```
compress.h        127 行   错误码 + deflate/inflate/gzip/zlib/CRC/adler API + alloc 版本
compress.c       1271 行   CRC32 表、Adler32、deflate/inflate、LZ77 + Fixed Huffman、gzip/zlib 包装、脚本绑定
```

### 2.2 做得好的地方

- **预计算的 Fixed Huffman 编码表**（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:461-498`）—— 直接 output bits，比运行期构建 canonical code 快得多。
- **长度/距离码用二分查找**（`find_len_code` / `find_dist_code`，`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:514-533`）—— O(5) vs O(29) 线性扫描。
- **CRC32 表 + Adler32 NMAX 5552 分块** 实现规范、高效。
- **gzip 可选字段** FEXTRA/FNAME/FCOMMENT/FHCRC 全部解析。
- **zlib CMF+FLG % 31 == 0 校验、FDICT 跳过** 都对齐了 RFC。
- **`xr_is_zlib` / `xr_is_gzip`** 对外暴露，让用户能做格式嗅探。
- **`xr_gunzip_alloc` 读 trailer ISIZE 做 buffer 预分配**，大多数场景一次性命中。

### 2.3 问题清单

#### 🔴 硬红线

| # | 位置 | 问题 |
|---|------|------|
| Z-1 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:673-728` | `xr_deflate` LZ77 哈希表 `malloc/free`（`hash_table`、`prev_chain` 共 ~132KB） |
| Z-2 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:930-962` | `xr_gzip_alloc` / `xr_gunzip_alloc` 直接 `malloc/free` —— 而且**把 `malloc` 出来的指针往外 export 给调用者 `free`**，双重违反 |
| Z-3 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:1016-1163` | 6 个 binding 函数各自 `malloc`/`free`，且 `inflate`/`zlibDecompress` 的**重试循环**每次 `cap *= 2` 都 `malloc+free` 一轮，非常浪费 |
| Z-4 | 全部公共 `xr_*` 函数 | 无 `XRAY_API`/`XR_FUNC` 修饰（15+ 个） |
| Z-5 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:345-395` `xr_inflate` | **`build_huffman_table` 返回值全部没检查**（line 345、350、369、394、395）—— 非法 Huffman 表仍继续执行，`decode_huffman` 读未初始化的 `counts`/`symbols` → UB |

#### 🟠 严重

| # | 位置 | 问题 | 建议 |
|---|------|------|------|
| Z-6 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:670-732` `xr_deflate` | **只生成 BTYPE=01（Fixed Huffman），没有 BTYPE=10（Dynamic Huffman）**。对英文文本压缩比比 zlib 低 ~15-20%；对稀疏二进制数据无任何优势 | 至少实现 dynamic Huffman 的构建（code-length 编码、HLIT/HDIST/HCLEN） |
| Z-7 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:566-600` `lz77_find_match` | `max_chain = 128` 硬编码，**不随 `level` 变化**。所以 `level=1` 和 `level=9` 产出的压缩数据**完全一样**，level 参数名存实亡 | 按 zlib 策略映射（level 1 → chain 4，level 6 → 128，level 9 → 4096），并加 lazy matching |
| Z-8 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:559` | 函数签名用 `int`：`int *hash_table`、`int *prev_chain`、`int match_pos`、`int *match_dist` —— 输入 > 2GB 时 UB | 换 `size_t` / `int32_t` 并加长度上限校验 |
| Z-9 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:782-786,838-841` gzip trailer | ISIZE 只写 **低 32 位**，`xr_gunzip` 用它做严格相等校验 → 大文件（>4GB）**永远校验失败** | 按 RFC 1952 就是 `mod 2³²`，但解压端校验应该 mask：`stored_size != (*out_len & 0xFFFFFFFFu)` 已是 mask 写法，OK。问题在**文档要讲清楚限制** |
| Z-10 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:1081-1101` | `inflate` 重试上限 **8 次**，每次 `cap *= 2`，最终 ~256x 输入；对压缩比 > 1000:1 的 bomb（42.zip 之类）会**拒绝解压合法内容**；对恶意 zip bomb 又没真正的防护 | 让调用方传入 `max_output_size`，显式拒绝超过的；内部重试改成"估计-检查"而不是瞎翻倍 |
| Z-11 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:610-614` `xr_deflate_bound` | 注释"Worst case: uncompressed storage"，但 fixed Huffman 最坏也可能略超（每字节 9bit + EOB 7bit + 字头 3bit）。当前估算**只适用于 level=0**，压缩路径下如果输出恰好挤不下会返回 `BUFFER` | 按 zlib 公式：`in_len + (in_len >> 12) + (in_len >> 14) + 11`；至少加 32 字节 slack |
| Z-12 | 所有 binding | **错误静默返回 `null`**，脚本拿不到 `XrCompressError` 枚举；`compress.gunzip` 失败既可能是"不是 gzip"也可能是"CRC 错"也可能是"OOM"，没法区分 | 暴露 `compress.gunzipEx(data)` 返回 `{ok: bool, data?, error?}` 或 throw |
| Z-13 | 整个模块 | **无流式 API** —— compress/decompress 只能一次性内存装载；压 1GB 日志必须 1GB buffer | 参考 zlib `deflateInit/deflate/deflateEnd`，暴露 `compress.Gzip()` / `.write(chunk)` / `.finish()` |
| Z-14 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:191-201` `br_align` | "put back unconsumed bytes" 逻辑正确，但读完最后一个字节后如果 `bits_in_buf` 含 < 8 bit 剩余，也被清零 —— 不影响正确性，但边界条件微妙，**需要测试覆盖** | 加 fuzz 测试 |

#### 🟡 中等

| # | 位置 | 问题 |
|---|------|------|
| Z-15 | 全体 | **无 zstd / brotli / lz4** —— 现代 web 生态广泛使用。至少列 roadmap |
| Z-16 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:736-738` `xr_is_gzip` | 要求 `len >= 10`，但 2 字节魔数就足够判断"是不是 gzip"。用户想做 sniff 时多 2 秒的 IO |
| Z-17 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:746-790` `xr_gzip` | MTIME 恒为 0，不符合 RFC 推荐；OS 字段写 0xFF（unknown）还行 |
| Z-18 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:372` 动态 Huffman 代码长度解码 | `lengths[288 + 32]` 在栈上 1280 字节 —— OK。但 `if (i == 0) return ERR` 防守对 sym=16 的重复（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/compress/compress.c:381`），逻辑对 |

### 2.4 测试覆盖

- ✅ `1301_compress.xr` + `tests/unit/stdlib/test_compress.c` 覆盖基础 gzip/deflate/zlib。

缺口：
- ❌ **对真实 zlib / gzip 产物的互操作测试**（用 `/usr/bin/gzip` 压一个文件，用 xray 解；反向亦同）—— 这是这种模块的首要测试
- ❌ zip bomb 防御（很小输入 → 很大输出）
- ❌ 截断 / 损坏 CRC / 损坏 length header 的 fuzz
- ❌ 不同 `level` 产出的压缩比对比（验证 Z-7）
- ❌ 4GB 以上数据的 ISIZE 回绕

---

## 3. `regex` 模块（7 个文件，~7K 行）

### 3.1 文件结构

```
xregex.h             289 行   公共 API
xregex_internal.h    416 行   AST / Prog / DFA / SparseSet 结构
xregex.c             768 行   顶层 compile + match/find/replace/split
xregex_parse.c       969 行   AST 解析
xregex_compile.c    1268 行   AST → Prog (Thompson 构造 + ByteMap + 优化分析)
xregex_nfa.c         945 行   NFA 引擎（match / search / OnePass）
xregex_dfa.c         458 行   DFA 按需构建 + 缓存
xregex_binding.c     548 行   脚本绑定 + 匹配对象构造
```

### 3.2 做得好的地方（这是全仓库工程品质最高的一个大模块）

- **RE2 启发的三引擎架构**：DFA（首选）+ NFA（带 capture / 零宽断言 / Unicode）+ OnePass（无歧义 fast path）。
- **线性时间保证**，没有回溯语义，天然抗 ReDoS。
- **AST 用 Arena 分配**（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_parse.c:872-873`），compile 完 `xr_arena_destroy` 一键清空。
- **ByteMap 字节类压缩**（`xr_prog_compute_bytemap`，`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_compile.c:945-998`）—— 把 DFA 的 transition table 从 256 压到常见 ~20。
- **Literal fast path**（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_nfa.c:752-763`）直接 `memmem`，比通用 NFA 快 10-100x。
- **Prefix 优化**：AST 分析抽固定前缀，搜索时先 `memmem` 跳到前缀位置。
- **SparseSet** `O(1)` clear/contains/insert（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_internal.h:288-323`）。
- **TLS NFA context**（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_nfa.c:111-146`）避免 per-match 的 malloc/free。
- **Unicode 属性类** `\p{L}`、`\p{Han}` 支持，走 `xunicode` 模块。
- **指令位压缩**：8 字节 / inst，op 4 位、out 27 位，ALT out1 复用 union。
- **GC 管理的 regex 对象**（`xregex_binding.c:116-142`），用户丢弃后能自动回收。
- **命名捕获组** `(?P<name>...)` 和 `(?<name>...)` 双语法兼容。
- **内联 flag 组** `(?i:...)` 带作用域保存/恢复（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_parse.c:587-600`）。

### 3.3 问题清单

#### 🔴 硬红线 / 严重 Bug

| # | 位置 | 问题 |
|---|------|------|
| R-1 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex.c:189-196` 注释 + `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_dfa.c:387` | **注释"created at compile time, thread-safe" 是错的**。DFA cache 是 **lazy 填充** 的：`state->next[byte_class] = next_state` 是写操作。多个协程分布在不同 OS 线程上 `regex.test` 同一个正则时，这行有 data race → 轻则漏匹配、重则野指针。这是**最高优先级 bug** |
| R-2 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex.c:110-111` | `xr_re_alloc(sizeof(XrRegex))` / `xr_re_strdup(pattern)` **返回值没 NULL 检查**，OOM 时下一行 `re->pattern = ...` 立即 NPE |
| R-3 | 全部公共 `xr_regex_*` 函数 | 无 `XRAY_API`/`XR_FUNC` 修饰（30+ 个） |

#### 🟠 严重（正确性 / 性能 / 文档 vs 实现不符）

| # | 位置 | 问题 | 建议 |
|---|------|------|------|
| R-4 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_parse.c:241-402` `parse_escape` | **没有 `case 'u':` —— `\u0000` 转义完全不支持**。但 xregex_parse.c:22 的文件注释写 "Escapes \n, \t, \r, \x00, **\u0000** etc." | 加 `\uHHHH` 和 `\u{HHHHHH}` |
| R-5 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_parse.c:722-731` `parse_atom` 和 `parse_cc_char` | **多字节 UTF-8 字符被按字节拆成多个 literal AST 节点**。后果：UTF-8 模式下 `(.)` 只匹配 1 字节，`[中文]` 只按原始字节序比较（而非 codepoint），`XR_RE_UNICODE` flag 定义了但根本没实现 | `XR_RE_UNICODE` 模式下用 `decode_utf8` 读完整 codepoint，生成 `XR_OP_UNICODE_RANGE` |
| R-6 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_nfa.c:112,116-124` | `__thread g_nfa_ctx` **线程退出时不释放**。每个 worker OS 线程终生持有一份 context（至少 `256 * sizeof(XrThread) ≈ 132KB`）；动态创建的线程池随规模线性增长 | 注册 `pthread_key_create` + destructor；或者放 `XrayIsolate->thread_local_arena` |
| R-7 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_nfa.c:225-269` `add_thread` | **递归展开 epsilon 链** —— 对深层嵌套 `((((a))))` 或长链 `(?:a\|b\|c\|...)` 栈深可能 > 几千。没有 depth guard | 改迭代（显式 stack）或设 `XR_RE_MAX_NESTED_DEPTH=100` 硬限制 |
| R-8 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_compile.c:886-901` | `prog->start_unanchored` 产出但**全仓库无任何读取点**；`xr_nfa_match`/`xr_nfa_search` 都用 `prog->start`。是**死代码 + 浪费 2-3 条指令槽**（ALT + ANY + 可能的 NOP） | 删掉字段和生成代码，把搜索用 seed 驱动的现有方式继续用 |
| R-9 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_dfa.c:253` | DFA cache hash table **初始大小 1024 且永不 rehash**；内存预算还有 8MB 时就因哈希满了返回 NULL → 全量 fallback NFA。对复杂正则（字符类多、状态集大）浪费严重 | 负载因子 > 0.75 时 `resize`（重新插入） |
| R-10 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_compile.c:342-391` `expand_ranges_ignorecase` | 输出缓冲上限 `max_out = 128`，若 char class 有多个跨越大小写边界的范围 → **静默截断**，匹配漏字符 | 改动态 `xr_malloc` 或在上层精确计算所需容量 |
| R-11 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_compile.c:1186` 提取 prefix | **IGNORECASE 模式下不提取 prefix** —— 前缀优化失效，无谓的 10-100x 性能损失 | 提取大小写折叠后的 prefix（预计算 upper/lower 两份做 2 次 `memmem` 取最小），或用 `strcasestr` 风格 |
| R-12 | 全局 | **`XR_RE_EXTENDED`（x 标志）**在 `xregex.h:44` 声明、但 `parse_group`（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_parse.c:561-585`）没有解析 x 也没有在 `parse_atom` 忽略空白/注释 —— 公开 API 承诺的功能不存在 | 要么实现，要么从 enum 删掉并更新文档 |
| R-13 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex.h:75-80` | `XR_RE_MAX_CAPTURES = 32` 写死在头文件；超过时 `captures_to_match` 静默截断（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex.c:179`），没有 compile-time 报错 | parser 检测到 `capture_index > 31` 时返回 `XR_RE_ERR_TOO_MANY_CAPTURES` |
| R-14 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_binding.c:182-196` `regex_compile` | 通过 `value_to_cstring` 拿到 `XrString.data`，但 `xr_regex_parse` 内部用 `strlen(pattern)` 找 end —— **模式包含 NUL 字节会被截断**。XrString 本身有 length，应透传 | 在 `xr_regex_compile_ex` 加 `size_t pattern_len` 参数，用 `pattern + pattern_len` 初始化 `parser->end` |
| R-15 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex.c:115-117` DFA 懒创建 | DFA 在 compile 结束时**就创建了**（如果 safe），但多数场景（只做 capture）永远用不到。对正则字面量密集的脚本浪费 ~几 KB/个 | 改懒创建：第一次 `xr_regex_test` / `xr_regex_count` 等无 capture 路径调用时才 `xr_dfa_new` |
| R-16 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_binding.c:69-103` `create_match_object` | `match->groups[i].start == NULL`（未捕获）时 `start`/`end` 填 `0` —— 和"起点 0 的空匹配"语义冲突，脚本侧无法区分 | 未捕获组填 `-1` 或 `null`，和 Python `re` 对齐 |
| R-17 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_binding.c:49-61` `parse_flags` | **`u`, `x`, `U` 标志静默忽略**；脚本写 `regex.compile(p, "iu")` 和 `"i"` 行为完全一样 | 至少 log warning，或增设 strict 模式 |
| R-18 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_dfa.c:100-126` `dfa_cache_lookup` | **hash 冲突用线性探测**，命中率/冲突率未监测；最坏退化到 O(n) 查 | 改 robin hood / cuckoo，或至少加 metric |
| R-19 | `@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex.c:456-461` | `XR_REALLOC_OR_ABORT` 在 find_all 分配失败时 **直接 abort 整个进程** —— 对长期运行的服务不友好 | 改返回 `NULL` + `xr_regex_find_all` 把 `out_count=0` 作为"内存不够"信号（已有），但实际使用 abort 宏 |

#### 🟡 中等

| # | 问题 | 建议 |
|---|------|------|
| R-20 | 错误位置 `parser_error` 用**字节偏移**（`p - pattern`）—— Unicode 模式下用户看不懂 | 可选提供字符偏移（代价是 parse 完后走一次 UTF-8 扫描） |
| R-21 | `xr_regex_iter_next` 每次都从 `iter->pos` 重新跑完整搜索 —— 长文本 + 大量 match 性能差（每次 O(n)） | 保留 DFA state 做增量式匹配 |
| R-22 | `xr_regex_is_valid` 不接受 flags 参数（`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/regex/xregex_binding.c:409-417`） | 加 flags 参数 |
| R-23 | 替换不支持 function callback（类似 JS `String.replace(re, fn)`） | 加 `regex.replace(re, text, (match) => ...)` |
| R-24 | `xr_regex_dump` / `xr_prog_dump` 用 `printf` 直接输出 | 改走 `xray` log 模块 |
| R-25 | `compile_repeat` 展开 `{n}` 会把 child 节点 compile n 次（字面复制指令）。`a{1000}` 生成数千条指令，触发 `XR_RE_MAX_PROG_INST = 10000` 上限并不宽松 | 对大 n 改用循环 ALT 结构（但会失去 OnePass），或者按 log2(n) 折半分解 |
| R-26 | `_GNU_SOURCE` for memmem | 移到 CMake 层统一定义 |

### 3.4 测试覆盖

- ✅ `1100_regex.xr` / `1110_regex_unicode.xr` / `1115_regex_advanced.xr` / `1120_regex_re2_compat.xr` / `1130_regex_literal.xr`
- ✅ `tests/unit/regex/test_regex.c`
- ✅ benchmark `regexredux.xr`

缺口：
- ❌ **多线程并发 `regex.test`/`find` 同一 regex**（能稳定触发 R-1 DFA race）
- ❌ **OOM 注入**（`xr_malloc` 返回 NULL 时 R-2/R-9/R-10/R-19 能否优雅失败）
- ❌ **字节精确对照 RE2/PCRE**（差异化测试发现 R-4/R-5）
- ❌ **极深嵌套** `((((((((((a))))))))))` → 测 R-7
- ❌ **含 NUL 的 pattern / text** → 测 R-14
- ❌ **`{1000}` 之类大量展开** → 测 R-25 + 指令数爆炸
- ❌ **Fuzz**（afl / libfuzzer）—— 正则 parser 是最值得 fuzz 的子系统之一
- ❌ **DFA cache 填满后的 fallback 路径**（构造高状态数正则）

---

## 4. 横向对比 / 系统性建议

### 4.1 共有问题 → 共有重构

| 问题 | 涉及模块 | 建议统一做法 |
|------|----------|-------------|
| **`malloc/free` 红线违反** | crypto (~15 处)、compress (~25 处) | 一次性 sed 式替换为 `xr_malloc/xr_free`；binding 一层的动态缓冲改用 `stdlib/common.h` 里的 `XrDynBuf` 或 `XR_REALLOC_OR_ABORT` |
| **`XRAY_API`/`XR_FUNC` 缺失** | 三者共 ~75 个公共函数 | 全部补齐；配套改 `Bindings.md` 规范 |
| **错误码丢失** | 三者 binding 层都 `return xr_null()` | 统一引入 `crypto.Result` / `compress.Result` / `regex.Result`（复用 `stdlib/serialization` 文档里已提议的 `{ok, data, error}` 三元组），保留原 API 做快捷方式 |
| **流式 API 缺失** | 三者都只有 one-shot | 抽通用 `XrStream` 接口（看能否和 `stdlib/io` 复用），分别暴露 `crypto.Sha256Stream`、`compress.GzipStream`、`regex.MatchIter`（regex 已有 iter，但不增量） |
| **OOM → abort** | regex 用 `XR_REALLOC_OR_ABORT` | `XR_REALLOC_OR_ABORT` 适合 VM 内部，不适合 stdlib 对外 API。应改返回错误码 |
| **恶意输入防护** | compress（zip bomb）、regex（爆炸正则）、crypto（超长输入） | 加 `max_output_size` / `max_instructions` / `max_hash_input` 配置 |

### 4.2 按 ROI 排序的 PR 建议

| 序号 | 工作量 | 收益 | 建议 |
|------|-------|------|------|
| **P1** | S | 🔴🔴🔴 | **修 DFA race（R-1）** —— 要么给 DFA 加一把细粒度锁，要么把 DFA cache 变成只在 compile 时同步填充（类似 RE2 的 StartStates），或者每个 isolate/coroutine 一份 DFA。当前行为是潜在 crash/漏匹配 |
| **P2** | S | 🔴🔴 | **crypto + compress 的 `malloc/free` 全量换 `xr_malloc/xr_free`** —— 纯机械工作，风险低 |
| **P3** | S | 🔴🔴 | **三模块全部补 `XRAY_API`/`XR_FUNC`** —— 配合 P2 一起提 |
| **P4** | S | 🔴 | **修 R-2**（`xr_regex_compile_ex` NULL 检查）和 **Z-5**（`xr_inflate` 的 `build_huffman_table` 返回值） |
| **P5** | M | 🟠🟠 | **crypto 加 AES-GCM 和 PBKDF2**，把 `crypto.encrypt` 迁到 authenticated encryption；旧 API deprecate 一个版本再移除（C-7） |
| **P6** | M | 🟠🟠 | **regex 实现真正的 UTF-8 模式**（R-4 `\u` 转义 + R-5 `XR_RE_UNICODE` 模式下 `.` / char class 基于 codepoint）。这是 xray 语言层面"unicode 友好"的基础 |
| **P7** | M | 🟠 | **DFA cache rehash**（R-9）+ **IGNORECASE 下 prefix 提取**（R-11）—— 性能 2-10x |
| **P8** | M | 🟠 | **compress 引入 dynamic Huffman**（Z-6）—— 压缩比提升 10-20%，对日志/JSON 场景明显 |
| **P9** | M | 🟠 | **删除 `start_unanchored` 死代码**（R-8），顺带把 `XR_RE_EXTENDED` 从头文件移除或实现（R-12） |
| **P10** | L | 🟠 | **三模块流式 API**（`Sha256Stream` / `GzipStream` / `RegexIncrementalMatcher`）—— 大文件场景必须 |
| **P11** | L | 🟡 | **统一 stdlib 错误对象** —— 把"返回 `null` = 错"改成"返回 `{ok, data/error}`"，可以和 serialization 组的建议一起做 |

---

## 5. 测试覆盖补齐清单（可直接作 `tests/regression/10_stdlib/` 的 backlog）

### crypto
1. NIST SHA-{1,256,512} / AES CAVP 向量批量比对
2. HMAC RFC 4231 向量
3. `crypto.encrypt` / `crypto.decrypt` 往返（binary-safe，含 `\0`）
4. PKCS7 padding 故障注入 + 1000 次统计时序
5. `randomBytes(1024)` 熵值测试（chi-square / monobit）
6. UUID v4 RFC 4122 variant/version 字段
7. `xr_md5_update` 在 2⁶¹ 字节上的 count carry
8. `/dev/urandom` read 返回 0/EINTR 的恢复

### compress
1. **`gzip` / `zlib` 与系统 `gunzip`/`zcat` 的互操作**（最重要）
2. Fuzz：随机 1KB 二进制 → `xr_gzip` → `xr_gunzip` → 一致性
3. 截断 gzip / 损坏 CRC / 损坏 NLEN / 非法 Huffman 表
4. zip bomb 防护（1KB → 1GB 的测试样本）
5. level 0/1/6/9 产出大小对比（发现 Z-7）
6. 空输入 / 单字节输入 / 4GB+ 输入

### regex
1. **并发 test**：100 协程同时 `regex.test(r, text)` 同一个 r，长时间跑（触发 R-1）
2. RE2/PCRE 差异化对比（同一测试集）
3. Unicode 综合：`\p{Han}`、UTF-8 内 `.`、`[\u4E00-\u9FFF]`（`\u` 要先修好）
4. `{1000}` 型展开（R-25）+ 指令上限
5. 1000 层嵌套 capture（R-7 栈深）
6. Pattern 含 NUL（R-14）
7. DFA 状态爆炸（10000+ 状态触发 R-9 + fallback）
8. OOM 注入（用 `xr_malloc` mock）
9. `replaceAll` 内 `$` 各种形式（`$0`、`$&`、`${name}`、`$$`、`${99}`）

---

## 6. 附：代码量与技术债粗算

| 模块 | 行数 | 红线违反 | Bug 风险点 | 重构工作量估算 |
|------|------|---------|-----------|---------------|
| crypto | 1150 | 15+ malloc/free | 8 `🟠` + 6 `🟡` | 2-3 PD（仅 P2 替换 0.5 PD） |
| compress | 1271 | 25+ malloc/free | 9 `🟠` + 4 `🟡` | 3-5 PD（P8 dynamic Huffman 占 2 PD） |
| regex | 7000 | 0 | 16 `🟠` + 7 `🟡` | 5-8 PD（P1 DFA 锁 + P6 真 UTF-8 + P7 cache rehash 各 1.5-2 PD） |
| **合计** | **~9400** | **~40 处** | **~50 bug 条目** | **~10-16 PD** |

> 结论：这一组的**技术债集中在 crypto/compress 的机械违规**（一次性就能清完）和 **regex 的架构级遗留**（DFA 并发、UTF-8 支持）。如果只做 P1~P4，一个工作日可以把"红线 + 数据竞争 + NPE"全部清掉，收益非常可观。
