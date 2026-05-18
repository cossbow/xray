# stdlib 基础工具组源码分析与优化建议

> 范围：`stdlib/os`、`stdlib/path`、`stdlib/io`、`stdlib/time`、`stdlib/datetime`、
> `stdlib/url`、`stdlib/math`、`stdlib/encoding`、`stdlib/log`、`stdlib/gc`、
> `stdlib/test_yield` 共 11 个模块，约 8.2K 行 C 代码。
>
> 严重程度标注：🔴 **严重**（违反硬性红线 / 安全隐患） · 🟠 **重要**（行为缺陷 /
> 鲁棒性问题） · 🟡 **建议**（可用性 / 设计改进） · 🟢 **优化**（性能 / 清理）

---

## 1 · 总览与横向一致性

### 1.1 做得好的地方

- **`math`**：用 `xr_random_bytes` 做 PRNG，`randomInt` 用拒绝采样避免取模偏置；`abs(INT64_MIN)` 溢出正确处理。
- **`url`**：全面使用 `xr_malloc`/`xr_free`；RFC 3986 `remove_dot_segments` 正确实现；`UrlParts` 零拷贝解析。
- **`path/normalize`**：显式避免 `strtok`，线程安全；用偏移量数组代替中间字符串拷贝。
- **`io/stat`**：使用 `XrJson + XrShape` 做结构化返回，是 L0 级 Shape 的优秀示范。
- **`gc`**：每协程 GC 控制设计干净；`gc.info()` 一次返回全量状态。
- **`encoding`**：UTF-16 代理对完整处理；`xr_utf8_*` 复用 core 不重复。
- **`datetime`**：用 native type + 方法表，`mod->loaded` 之外的设计整洁。

### 1.2 系统性问题（跨模块）

| 严重度 | 问题 | 影响模块 |
|-------|------|---------|
| 🔴 | 直接使用 `malloc/free/realloc`，违反"统一 `xr_malloc/xr_free`"硬性红线 | `os`, `io`, `log`, `test_yield` |
| 🔴 | 非 static 函数未带 `XRAY_API`/`XR_FUNC` 修饰符 | `time`, `log`, `url`, `encoding`, `datetime`, `io`, `gc`, `os`, `path`, `test_yield`（**全部**） |
| 🔴 | 文件作用域可变全局变量未受控 | `log`（`g_default_logger`、`g_async_queue`）、`test_yield`（`g_counter`）、`io`（`shape_stat_result`） |
| 🟠 | 同步阻塞系统调用未让出协程 | `os.sleep`、`os.exec`、`io.*File/*Dir`、`time.sleep` fallback |
| 🟠 | `xr_realloc` 使用时未用 `XR_REALLOC` 中转，OOM 会泄漏原指针 | `os`, `log` |
| 🟠 | 固定栈缓冲截断字符串（`buf[256]`/`[4096]` 等）而无错误返回 | `url`, `datetime`, `log`, `path`, `io` |
| 🟠 | `mod->loaded = true` 遗漏 | `gc`, `datetime`, `test_yield` |
| 🟡 | 每次函数调用都重复 intern 常量键字符串（如 `"stdout"`、`"mtime"`） | `path.parse/format`, `url.parse`, `os.exec`, `gc.info`, `log` |
| 🟡 | 错误信息丢失：统一返回 `bool`/`null`，吞掉 `errno` | `os`, `io` 几乎所有文件系统 API |
| 🟡 | 断言密度远低于 50–80 行/次红线 | 几乎所有模块 |
| 🟢 | 重复样板：`make_string(X, "...")`、`EXPORT_CFUNC` 宏在 11 个 `.c` 各写一遍 | 所有模块 |

### 1.3 推荐总体方向

1. **统一一层 thin-wrapper helper**：把 `get_string_arg`/`make_string`/`EXPORT_CFUNC` 抽到 `stdlib/common.h`（纯 inline，不破坏 DAG），彻底消除 11 份重复副本。
2. **把所有 `malloc/free` 替换为 `xr_malloc/xr_free`**：这是红线必须改。
3. **非 static 声明统一加 `XR_FUNC`（多编译单元使用）或降级为 `static`（仅文件内用）**：配合 amalgamated build 才能让内联生效。
4. **阻塞 syscall 通过 `XrAsyncPool` 卸载**：`io` 已有半成品 (`xr_io_read_on_async`)，但结果无回传路径，是"写了一半"的代码。要么删除要么补完。
5. **固定栈缓冲统一换 `CtxBuf` 或 `xr_malloc` 扩展**：不再静默截断。

---

## 2 · 逐模块问题清单

### 2.1 `stdlib/os`

**🔴 严重**

1. **直接 `malloc`/`realloc`/`free`**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/os/os.c:440-457`（`read_fd_to_string`）、`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/os/os.c:473-483`（Windows 分支的 `_popen` 读取）。必须换成 `xr_malloc/xr_realloc/xr_free`。
2. **`realloc` 不中转即赋值回原指针**：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/os/os.c:478` `output = (char*)realloc(output, cap);` 失败直接泄漏。应用 `XR_REALLOC(output, cap)` 宏。

**🟠 重要**

3. **`os_sleep` 阻塞 worker 线程**：`nanosleep` 会挂起整个 OS 线程，直接破坏协程并发。应改为调用 `xr_coro_sleep`/`xr_netpoll_timer_yield` 一类协程级挂起。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/os/os.c:410-428`。
4. **`os_exec` 同步 `fork+exec+waitpid`**：当前实现会阻塞 worker 直到子进程结束，且在多线程协程环境下 `fork` 是经典雷区（锁继承、信号处理器继承）。长期应转成 `posix_spawn` + async pipe IO；短期至少在 `XrAsyncPool` 里跑。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/os/os.c:461-541`。
5. **Windows `gethostname` 缺少 `WSAStartup`**：Windows 下不先初始化 Winsock 即调用 `gethostname` 返回 `WSANOTINITIALISED`。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/os/os.c:198`。
6. **Windows `_pclose` 返回值误用作 `exitCode`**：`_pclose` 返回与 `_wait`/`system` 相同的编码格式，需要 `WEXITSTATUS`-类解码才能得到真正 exit code。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/os/os.c:483,488`。
7. **`os_exec` Unix：`read_fd_to_string` 返回 `NULL` 时跳过 `waitpid`？** 检查：实际上 NULL 仍继续往下跑，使用 `?:` 兜底 `""`。但如果 stdout 返回 NULL，`stderr_pipe` 里的数据读取仍保留句柄，**不是 bug**；但 `waitpid` 没指定 `WNOHANG`，子进程若悬挂会导致永久阻塞。建议加入 pid 超时或 `waitpid(…, WNOHANG)` 轮询。

**🟡 建议**

8. `environ` 在 macOS 下按进程环境安全，但 iterating 过程中另一线程调用 `setenv` 会 UB。标注线程安全限制或加全局锁。
9. 常量 `sep`/`eol` 与 `path` 模块已有的 `sep`/`delimiter` 重复；考虑只在一处暴露。

---

### 2.2 `stdlib/path`

**🟠 重要**

1. **`path_resolve` 不使用 `IS_SEP`**：只按 `/` 判断绝对/分隔，Windows 下 `\` 路径失效。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/path/path.c:326,332`。
2. **`path_resolve` 固定 `result[4096]`**：`strncpy`/`strncat` 切断长路径且无报错；macOS 下 `PATH_MAX` 本身就 1024，但拼接多段可轻易超过。建议改用 `CtxBuf`。
3. **`path_format` 硬编码 `"/"` 分隔符**：忽略 `PATH_SEP`，在 Windows 会生成混合分隔路径。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/path/path.c:506`。
4. **`path_parse` 忽略 `args[0]` 不是字符串的情况**：`path = ""` 后继续 call `path_basename(args, 1)` 会读非 string 的 args → `get_string_arg` 返回 NULL → basename 返回 `""`。但中途创建的 map 被污染；应尽早返回空对象。

**🟡 建议**

5. `path_parse` / `path_format` 每次都 `make_string(X, "dir")` intern 6 个键；可预先 intern 一次并缓存 per-isolate。
6. `path_extname` 对 `foo.tar.gz` 只返回 `.gz`（符合 POSIX 惯例，与 Node 一致），但对 `..hidden` 的行为值得在 doc 里显式声明。

---

### 2.3 `stdlib/io`

**🔴 严重**

1. **全程 `malloc/free`**：`io_async_read_invoke` / `io_async_write_invoke` / `xr_io_read_on_async` / `xr_io_write_on_async` 的任务数据与内容缓冲都用 `malloc`。违反红线。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/io/io.c:698-823`。
2. **`shape_stat_result` 是进程级全局**：`static XrShape *shape_stat_result = NULL;` 仅在 `xr_load_module_io` 里重建一次。多 Isolate 场景下 `SymbolId` 每个 Isolate 独立，跨 Isolate 复用此 shape 会命中错误的符号表。注释也写明"Must rebuild every time because SymbolIds are per-isolate"，但代码不满足此要求。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/io/io.c:416-440`。**正确做法**：把 `shape_stat_result` 放进 `XrayIsolate` 的 per-isolate stdlib cache。

**🟠 重要**

3. **`xr_io_read_on_async` / `xr_io_write_on_async` 是半成品**：提交 async job 后直接返回 true，但 **调用者永远拿不到 `d->content/d->success` 结果**；任务完成后分配的 buffer/struct 也**没人释放**。当前代码如果有任何使用者，就一定在泄漏 + 丢结果。建议：① 整段删除直到真正需要；② 或者配合 continuation：在 `xr_async_job_create` 时注册 resume hook，把内容塞回协程栈。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/io/io.c:727-824`。
4. **所有同步文件 IO 阻塞 worker**：`readFile`/`writeFile`/`readDir`/`removeAll` 等都直接调用 `fopen/fread/fwrite/nftw`。在 worker 线程跑时会挂住所有 N_coro。至少要提供一套 "async" 版本，或默认走 async pool。
5. **`io_readFile` / `io_readFileBytes` 大文件截断**：`ftell` 返回 `long`，`(int32_t)size` 强转到 `XrArray` 长度字段，>2GB 文件会溢出成负数并 `malloc` 分配极小 buffer 随后 `fread` 缓冲区越界。必须先校验 `size <= INT32_MAX` 再分配。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/io/io.c:74-78,103-107`。
6. **`io_mkdirp` `len == 0` 访问 `tmp[-1]`**：UB。入口应校验 `len > 0`。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/io/io.c:487-488`。
7. **Linux `sendfile` 不循环**：大文件一次 `sendfile` 可能只返回部分字节数，也可能被信号中断。需要循环直到 `st.st_size` 字节全部写入。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/io/io.c:340-344`。
8. **`io_tempFile`/`io_tempDir` 仅 Unix**：硬编码 `/tmp/xray_XXXXXX`；Windows 下 `mkstemp` / `mkdtemp` 不存在。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/io/io.c:585-601`。
9. **`io_readDirRecursive` 路径拼接缺溢出检查**：`snprintf(fullpath, PATH_MAX, "%s/%s", path, name)` 返回值未检查；超长路径被静默截断。还缺 inode 去环（硬链接 / bind mount 能产生循环）。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/io/io.c:615-643`。

**🟡 建议**

10. 所有返回 `bool` 的文件 API（remove/rename/mkdir…）把 `errno` 吞掉。可以提供 `io.lastError() → string` 或改成 `{ok: bool, error: string}` 结果结构。
11. `io_stat` 路径用同一个 `stat` 后再调 `lstat` 检测 symlink，其实一次 `lstat` + 条件 `stat(目标)` 更省；而且当前实现对 symlink 只上报是否是 link，不暴露 `S_ISCHR`/`S_ISBLK`/`S_ISFIFO`。
12. `io_readLines` 用 `getline` 的 `free(line)` 用的是 system free（来自 libc），与 `xr_free` 不一致但是这个是 `getline` 内部 alloc 的，确实只能 `free`；保持原样，但需在注释说明。

---

### 2.4 `stdlib/time`

**🔴 严重**

1. **非 static 函数缺可见性修饰**：`xr_time_now`/`xr_time_clock`/`xr_time_monotonic`/`xr_time_nanos`/`xr_time_micros`/`xr_time_sleep` 都是 non-static，但 `.h` 中裸声明 `XrValue xr_time_now(...)`。应加 `XR_FUNC` 或若仅 `.c` 使用则全部 `static`（本模块其实只在自己 `.c` 用，建议统一 `static`）。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/time/time.h:37-78`。

**🟠 重要**

2. **`xr_time_sleep` fallback 阻塞**：`xr_sleep_ms` 是 OS sleep。注释说编译期会翻译成 `OP_SLEEP`，但 dynamic dispatch / import-as-value 路径仍会命中这个函数。至少应在协程上下文中调 `xr_coro_timer_yield`。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/time/time.c:103-127`。
3. **`get_monotonic_ns` Windows 路径溢出**：`counter.QuadPart * 1000000000LL / freq.QuadPart`。当进程已跑几天 + freq=10MHz 时乘法会溢出。改成 `sec = q/freq; ns = (q%freq)*1e9/freq; return sec*1e9 + ns;`。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/time/time.c:40-47`。

**🟡 建议**

4. `xr_time_clock` 用 `clock()` 在 32-bit `clock_t` 系统上易溢出；macOS/Linux 上都是 `CLOCKS_PER_SEC` = 1e6 的 clock_t，约 70 分钟溢出。建议改用 `getrusage(RUSAGE_SELF, …)` 的 `ru_utime+ru_stime`。

---

### 2.5 `stdlib/datetime`

**🟠 重要**

1. **`datetime_alloc` 不判 NULL**：`xr_gc_alloc` OOM 会返回 NULL，紧接着就写 `dt->timestamp = 0` 崩溃。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/datetime/datetime.c:34-42`。
2. **`xr_datetime_format` 静默截断**：`char temp[512]` 装不下就停写且不报错；再 memcpy 到 caller buf。`char buf[256]` 的 caller `dt_format` 同理更容易截断。建议用动态 buffer 或让 caller 传入大小。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/datetime/datetime.c:264-304`、`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/datetime/datetime.c:605-615`。
3. **`xr_datetime_to_utc` / `to_local` 用 `memcpy` 跳过 `XrGCHeader` 拷贝字段**：与 GC header 内部布局强耦合，一旦 header 加字段就静默出错。直接字段赋值更安全。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/datetime/datetime.c:501-531`。
4. **`xr_datetime_parse` 时区探测脆弱**：对 `"2024-01-15"` 这类短串回退到 `strrchr(str, '-')` 命中日期分隔符，靠 `p > str + 10` 启发式规避。建议先锚定 `'T'`/' ' 把日期、时间、时区三段拆分再各自解析。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/datetime/datetime.c:195-217`。
5. **负数时间戳毫秒归一化依赖 C99 整除语义**：`total_ms % 1000` 在 `total_ms` 为负时 C99 保证截断除法，但早期编译器实现不一。已做 `< 0` 修正，但最好显式用 `floor_div`/`floor_mod`。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/datetime/datetime.c:413-420`。
6. **`mktime` 返回 `-1` 视为 0 epoch**：`datetime_mktime` 直接把 `time_t(-1)` 塞进 `timestamp`，后续格式化出 `"1969-12-31T23:59:59"`；无法区分"真的就是 -1 秒"和"解析失败"。应让 caller 拿到 NULL / 失败指示。

**🟡 建议**

7. 每个组件 getter (`dt_year` 等) 都 `localtime_r(timestamp)` 一次；循环格式化场景有 10 倍性能损失。可以引入 `XrDateTime.cached_tm` 字段或提供 `dt.components() → Map`。
8. 缺 `dt.setYear`/`setMonth`/… 类不可变"更新"方法，只能靠 `add`。
9. `%lld` 应改用 `PRId64`。

---

### 2.6 `stdlib/url`

**🟠 重要**

1. **`url_parse_fn` `host_buf[512]`/`origin_buf[1024]` 固定**：`host = hostname:port`，真实 IPv6 + 端口可达 50 字节，但 `origin` 前缀再加 scheme 拼超长 URL 会截断。建议用 `CtxBuf`。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/url/url.c:392-419`。
2. **`url_build_query_fn` / `url_format_fn` `buf[4096]`**：同样静默截断。真实 OAuth scope、JWT-in-URL 用例常超 4K。
3. **`url_parse_query_fn` OOM 静默丢 key**：`xr_malloc(dk + 1)` 失败时 `continue`，调用者拿到部分解析的结果。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/url/url.c:528-544`。
4. **`url_resolve_fn` 未按 RFC 3986 §5.3 处理 base 的 query/hash**：`resolve("http://a/p?x=1", "b")` 当前结果拼成 `http://a/b`，正确结果应是 `http://a/b`（去除 base query/hash），正巧一样；但 `resolve("http://a/p?x", "#frag")` 的正确行为是保留 base path 并 clear query → 当前实现会产出错误结果（把 rel 直接 append 到 `/`）。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/url/url.c:636-722`。

**🟡 建议**

5. `is_unreserved` 用 `isalnum`（locale 相关）：在土耳其语 locale 下 `I/i` 可能被当非 alnum。改成显式 ASCII 范围检查。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/url/url.c:51-53`。
6. `url_parse_query_fn` 先 `xr_malloc` 再立即 `xr_free` key_copy，仅为得到 `\0` 结尾字符串传给 `xr_json_set_by_key`。可以给 `xr_json_set_by_key_n(key, keylen, val)` 版本，零拷贝。
7. `port` 未做数值校验，允许 `":abc"` 过关。

---

### 2.7 `stdlib/math`

**🟡 建议**

1. **`get_number` 非数值静默返回 0**：`math.sqrt("foo")` → `sqrt(0) = 0`。应返回 NaN 或在 caller 提前 `XR_DCHECK(XR_IS_NUMBER(v))`。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/math/math.c:27-31`。
2. **`M_PI`/`M_E` 依赖非 C99 扩展**：`<math.h>` 里的 `M_*` 宏需要 `_GNU_SOURCE`/`_XOPEN_SOURCE`。macOS 默认有，Linux glibc 有，但 MSVC 需要 `_USE_MATH_DEFINES`。Windows port 会编译失败。建议在本模块顶部显式定义：
   ```c
   #ifndef M_PI
   #define M_PI 3.14159265358979323846
   #endif
   ```
3. **无 seeded PRNG**：`xr_random_bytes` 是密码学级，跑 Monte-Carlo 无法复现。建议新增 `math.seed(n)` + 独立 xorshift/pcg 状态（per-isolate）。
4. **缺 `math.gcd/lcm/isprime`**：常用工具函数，拖到上层每次重写。
5. `math_isNaN` 只接受 float，传 int 直接 false（对）；但 `math_isFinite` 接受 int 路径经 `get_number`，int 永远 finite，OK。
6. `math_random` 用 53 bits → double 精度正确，但每次调用都进一次 `xr_random_bytes` 系统调用，热循环下有明显成本；可以批量取 8KB 缓存。

---

### 2.8 `stdlib/encoding`

**🟡 建议**

1. 所有 xray 绑定都是 `xr_malloc → 填充 → xr_string_intern → xr_free`，double-copy。如果 `xr_string_intern` 支持"预占位+填写"接口可省一次拷贝。
2. `xr_utf16_encode/decode` 内层 endian if 在热路径：可拆成 `_le` 和 `_be` 两个函数，caller 分发一次，小数据无影响，大文件省 10-15%。
3. `xr_utf8_decoded_len` 命名易误会：它算的是 "UTF-16 解码后得到的 UTF-8 字节数"。改名 `xr_utf16_to_utf8_len` 更清晰。
4. BOM (`0xFEFF`) 没有可选剥离；对从文件读入的 UTF-16 尤其常见。加一个 `stripBom=true` 可选参数即可。
5. **缺** Base16 大写变体、Base32、Ascii85/Base85。如果以后要支持，应与 `stdlib/base64` 合并成单一 `encoding` 模块。

---

### 2.9 `stdlib/log`

**🔴 严重**

1. **几乎全程 `malloc/free`**：`ctxbuf_*` / `create_child_logger` / `xr_log_reset` / `xr_gc_destroy_logger` / `async_queue` 条目。违反红线。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/log/log.c:242-280,781-805,922-931`。
2. **`ctxbuf_ensure` `realloc` 失败泄漏**：失败时 `b->data` 已是原指针，下一行可能写越界（外层并没有重检 `data`）。应用 `XR_REALLOC` 并把失败往上传。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/log/log.c:249-254`。
3. **文件作用域可变全局**：`g_default_logger`、`g_async_queue`、`g_async_initialized` 是硬性红线的反例。至少应封在 `XrayRuntime` 或 `XrayIsolate` 里，或加 mutex 保护并在注释里显式声明例外。

**🟠 重要**

4. **`set_output`/`set_level`/`set_format` 无锁**：多协程并发调用 `log.info` 时，另一协程调 `setOutput` 会让正在写的 thread `fclose` 已在写入的 FILE*。
5. **`write_json_string_buf` 不校验 UTF-8**：raw 高位字节直接塞进 JSON 字符串，产生非法 JSON（严格解析器会拒）。应复用 `encoding.utf8Valid` / `xr_utf8_decode`，遇非法 byte 输出 `\uFFFD`。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/log/log.c:319-339`。
6. **`value_to_string` `vbuf[64]` 不够**：long 字符串 key 或 float `%.17g` 会截断。
7. **async queue 大小硬编码 256**：满时 producer 阻塞等 `cond_wait`，把调用点直接卡死，却没 drop/sample 策略。高吞吐日志会变成 latency 毒药。
8. **`parent` 字段声明了但从未在 `xr_log_write_ex` 里走链**：要么拆了字段，要么让 level/output/format 支持回溯 parent。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/log/log.h:56`。
9. **`%.6g` 精度丢失**：float roundtrip 要 `%.17g`。

**🟡 建议**

10. 缺 log 轮转、彩色输出、`WithContext` 绑定协程 trace id。
11. `xr_log_fatal` 同时 `async_queue_flush` + `exit(1)`，但没有 `async_queue_stop`，pthread 资源在 atexit 前泄漏（小影响）。

---

### 2.10 `stdlib/gc`

**🟠 重要**

1. **`xr_load_module_gc` 结尾无 `mod->loaded = true`**：与所有其他 stdlib 模块不一致。虽然调用链可能不依赖这一位，但形成"约定违背"。
2. **`gc_setpause`/`gc_setstepmul` 无边界**：只检查 `> 0`，上界无约束；极大值会导致分代触发周期异常。

**🟡 建议**

3. **`gc.info()` Map 键大小写不一致**：`totalbytes`、`gccount`、`freeblocks` 是全小写；但 `totalKB`、`gctimeMs`、`lastgctimeUs` 是 camelCase。统一成 camelCase（或全 snake_case）。
4. 缺 `gc.onCollect(cb)` 回调钩子（Lua 的 `__gc` 风格），调试内存问题非常有用。

---

### 2.11 `stdlib/test_yield`

**🔴 严重**

1. **全程 `malloc/free`**：10 个 state 结构体全用系统 `malloc`。由于这是 yield 函数的 C-侧 continuation state，不能放 GC 堆（协程挂起期间 GC 能跑），但仍应用 `xr_malloc`（对应系统 `malloc` 的桥接）。
2. **`g_counter` 文件作用域可变全局，无原子/锁**：所谓"并发计数"测试在多 worker 下会自带 data race，ASAN/TSAN 必报。应改成 `_Atomic int64_t`。文件：`@/Users/xuxinglei/workspace/xray-lang/xray/stdlib/test_yield/test_yield.c:41`。
3. **`malloc` 返回 NULL 未检查**：所有 `malloc(sizeof(...))` 之后立刻写字段。OOM 会 SEGV。
4. **`mod->loaded = true` 缺失**：同 gc。

**🟡 建议**

5. 当前 module 无条件导出到生产构建；建议 `#ifdef XRAY_ENABLE_TEST_STDLIB` 包住注册函数，release 里去掉。
6. 可将 `test_yield.counter_inc/get/reset` 移到单独 `test_concurrency` 测试夹具，不与 yield 测试混在一个 state machine 文件里。

---

## 3 · 推荐的重构顺序（按 ROI）

| # | 动作 | 预期收益 | 工作量 |
|---|------|---------|--------|
| 1 | 抽 `stdlib/common.h`（inline helpers: `get_string_arg`、`make_string`、`EXPORT_CFUNC`、`EXPORT_CFUNC_YIELDABLE`、`register_cached_key`）| 去除 1500+ 行重复代码；一处修复=全员受益 | S |
| 2 | 全局 sed：`malloc/calloc/realloc/free` → `xr_malloc/...`；`realloc` → `XR_REALLOC` | 合规 + debug 模式能捕获泄漏 | S |
| 3 | 所有 stdlib `.h` 非 static 函数加 `XR_FUNC` 修饰（或改 static） | amalgamated build 能内联；合规 | S |
| 4 | 把固定 `char buf[4096]` 全换 `CtxBuf`（从 log 模块抽出成 `stdlib/ctxbuf.h`）| 根除静默截断 | M |
| 5 | `io.readFile/writeFile/readDir/removeAll` 走 `XrAsyncPool`，或补全 `xr_io_*_on_async` 的 continuation 回传 | 消除 worker 阻塞 | L |
| 6 | `os.exec` 改 `posix_spawn` + async pipe；或者单独跑后台线程 | 同 5 | M |
| 7 | `os.sleep` / `time.sleep` 走协程 timer | 同 5 | S |
| 8 | `io.stat` 的 `shape_stat_result` 改 per-isolate | 修复多 Isolate 崩溃 | S |
| 9 | `log` 全局 logger 加 mutex，或迁移到 Isolate | 线程安全 | M |
| 10 | `url.buildQuery` / `url.format` / `datetime.format` 改动态缓冲 | 同 4 | S |

> S ≈ 半天 · M ≈ 1-2 天 · L ≈ 3-5 天

---

## 4 · 断言与可见性清单

### 4.1 各模块可见性违规明细

| 模块 | 非 static 但无修饰符的符号 |
|------|-----------------------|
| `time` | `xr_time_now/clock/monotonic/nanos/micros/sleep` |
| `log` | `xr_log_default/reset/write_ex/write/debug/info/warn/error/fatal/set_level/set_format/set_output/child/is_enabled/enable_source/enable_async/flush`、`xr_logger_*`、`xr_gc_destroy_logger`、`xr_log_level_name/parse` |
| `url` | `xr_url_encode/decode/encode_form/decode_form` |
| `encoding` | `xr_hex_encode/decode/valid`、`xr_utf16_encode/decode`、`xr_utf16_encoded_len`、`xr_utf8_decoded_len` |
| `datetime` | `xr_datetime_*`（全部 15 个） |
| `io` | `xr_io_read_on_async`、`xr_io_write_on_async` |

一次性修复脚本（伪代码）：

```bash
# 文件内仅本 .c 使用的改 static；.h 导出的改 XR_FUNC。
# 具体符号表见上表。
```

### 4.2 断言密度抽样

| 文件 | 行数 | 应有断言 | 实际 | 差距 |
|------|------|---------|------|------|
| `os.c` | 685 | 8-13 | 0 | 🔴 |
| `io.c` | 927 | 12-18 | 0 | 🔴 |
| `path.c` | 565 | 7-11 | 0 | 🔴 |
| `url.c` | 790 | 10-16 | 0 | 🔴 |
| `math.c` | 507 | 6-10 | 0 | 🔴 |
| `datetime.c` | 851 | 11-17 | 0 | 🔴 |
| `log.c` | 1022 | 13-20 | 1（`xr_log_write_ex`） | 🔴 |
| `encoding.c` | 467 | 6-9 | 0 | 🔴 |
| `time.c` | 185 | 2-3 | 0 | 🟠 |
| `gc.c` | 341 | 4-7 | 0 | 🟠 |
| `test_yield.c` | 509 | 6-10 | 0 | 🔴 |

**建议**：至少在每个模块入口（模块加载、公共 API 入口）加 `XR_DCHECK(isolate != NULL)` / `XR_DCHECK(args != NULL)` / `XR_DCHECK(argc >= N)`；关键不变式（GC 指针非空、buffer 容量足够）加 `XR_CHECK_BOUNDS`。

---

## 5 · 测试覆盖空白（观察到的）

以下场景在 `tests/` 中应验证但容易被忽略，建议补齐：

- `io.readFile` 对 > 2GB 文件的行为
- `io.readFileBytes` 返回 `Array<uint8>` 时 `length` 正确性（大文件）
- `io.copyFile` Linux 路径下 `sendfile` 截断（大文件 + O_APPEND dst）
- `io.removeAll` 对含符号链接的目录（不应跟随 symlink 删到外面）
- `os.exec` 长输出（> 64KB）、stderr 与 stdout 都很大时的死锁可能
- `path.normalize("/../../foo")` 绝对路径 `..` 超出根
- `path.resolve` 多段拼接超过 4096 字节
- `url.parse` IPv6 带 zone id：`http://[fe80::1%25eth0]:80/`
- `url.resolve` base 含 query：`resolve("http://a/p?x=1", "b")` → 应为 `http://a/b`
- `datetime.parse` 闰秒 / 闰月末 / 跨夏令时边界
- `log` async 模式下子协程调 `fatal` 是否正确 flush
- `encoding.utf16Decode` 孤立 surrogate、截断在 surrogate pair 中间
- `math.random` 统计均匀性（Chi-squared）
- `gc.disable()` 嵌套（`disable+disable+enable` 应还处于 disabled）
- `test_yield.counter_inc` 1000 协程并发下的一致性（需 `_Atomic`）

---

## 6 · 快速开始：3 个最值得先动手的 PR

### PR-1 · `stdlib/common.h` + `malloc` 合规化

**目标**：去重 + 合规。**diff 预估**：-1200 行 / +200 行。

```c
// stdlib/common.h
#ifndef XR_STDLIB_COMMON_H
#define XR_STDLIB_COMMON_H

#include "../src/runtime/value/xvalue.h"
#include "../src/runtime/object/xstring.h"
#include "../src/module/xmodule.h"
#include "../src/base/xmalloc.h"

static inline const char* xrs_string_arg(XrValue v, size_t *out_len) {
    if (!XR_IS_STRING(v)) return NULL;
    XrString *s = XR_TO_STRING(v);
    if (out_len) *out_len = s->length;
    return s->data;
}

static inline XrValue xrs_string_value_n(XrayIsolate *X, const char *s, size_t len) {
    if (!s) return xr_null();
    return xr_string_value(xr_string_intern(X, s, len, 0));
}

static inline XrValue xrs_string_value_c(XrayIsolate *X, const char *s) {
    if (!s) return xr_null();
    return xrs_string_value_n(X, s, strlen(s));
}

#define XRS_EXPORT(mod, isolate, name_str, func_ptr) do { \
    XrCFunction *_cf = xr_vm_cfunction_new((isolate), (func_ptr), (name_str)); \
    xr_module_add_export((isolate), (mod), (name_str), xr_value_from_cfunction(_cf)); \
} while (0)

#endif
```

然后把所有模块里的 `get_string_arg`/`make_string`/`EXPORT_CFUNC` 替换成 `xrs_*` / `XRS_EXPORT`。

### PR-2 · `stdlib/ctxbuf.h`（从 log 抽公共动态缓冲）

把 `log.c` 里 `CtxBuf` 整块抽出成 `.h inline`，用 `xr_malloc`/`XR_REALLOC`，然后把 `url`、`path.resolve`、`datetime.format`、`io.readDirRecursive` 里所有 `char buf[4096]` 替换。

### PR-3 · `io.stat` shape per-isolate

在 `XrayIsolate` 加 `XrShape *stdlib_shapes[N]` 数组（或独立 `stdlib_cache` 结构），`io_ensure_stat_shape` 改为基于当前 isolate 查询/创建。

---

## 7 · 结语

基础工具组整体设计清爽、分层合理，是读源码的好切入点。当前主要技术债集中在三点：

1. **内存与可见性合规**（项目红线层面，必须补）；
2. **阻塞 syscall 未协程化**（与 xray "native concurrency" 的核心卖点矛盾）；
3. **固定缓冲/静默截断**（埋在输出层的隐蔽 bug）。

这三类问题改完，这个组可以作为其他 stdlib 组（序列化、加解密、net、http）的模板参考，后续分析 conversation 里可以少花时间在同类问题上。
