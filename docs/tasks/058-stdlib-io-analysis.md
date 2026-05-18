# stdlib/io 分析与优化建议

## 模块职责

`stdlib/io` 是标准库的文件系统 I/O 模块，覆盖整文件读写、字节数组读写、目录遍历、文件状态、递归目录操作、符号链接、临时文件/目录、权限和当前工作目录管理。

它当前不是流式 I/O 抽象，也不是通用 descriptor/handle 层，而是以“同步便捷函数”为主：

- `readFile/readFileBytes/readLines` 一次性把内容读入内存。
- `writeFile/writeFileBytes/appendFile/copyFile` 执行整文件写入/拷贝。
- `exists/isFile/isDir/isSymlink/fileSize/stat` 查询文件状态。
- `mkdir/mkdirp/readDir/readDirRecursive/remove/removeAll/rename/chmod/touch` 操作路径。
- `symlink/readlink/realpath/tempFile/tempDir/cwd/chdir` 提供平台文件系统能力的薄封装。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/io/io.h` | io 模块 API 概览和加载入口 |
| `stdlib/io/io.c` | runtime 绑定、文件/目录/临时文件/符号链接实现、builtin declarations |
| `stdlib/common_io.h` | json/yaml/toml/xml/csv 等 parseFile/writeFile 共用同步整文件 helper |
| `src/os/os_fs.h` | 跨平台文件元数据、cwd、realpath、rename/remove/mkdir shim |
| `src/os/os_dir.h` | 跨平台目录迭代 shim |
| `src/os/unix/fs_unix.c` | POSIX 文件系统 shim 实现 |
| `src/os/unix/dir_unix.c` | POSIX 目录迭代 shim 实现 |
| `src/os/win/fs_win.c` | Windows 文件系统 shim 实现 |
| `src/os/win/dir_win.c` | Windows 目录迭代 shim 实现 |

## 当前脚本 API

`xr_load_module_io()` 实际导出 29 个函数：

| API | 当前签名 | 语义 |
|---|---|---|
| `readFile(path)` | `(string): string?` | 读取整文件为字符串 |
| `readFileBytes(path)` | `(string): Array<uint8>?` | 读取整文件为字节数组 |
| `writeFile(path, data)` | `(string, string): bool` | 截断写入字符串 |
| `writeFileBytes(path, data)` | `(string, Array<uint8>): bool` | 截断写入字节数组 |
| `appendFile(path, data)` | `(string, string): bool` | 追加字符串 |
| `exists(path)` | `(string): bool` | 路径是否存在 |
| `isFile(path)` | `(string): bool` | 是否普通文件 |
| `isDir(path)` | `(string): bool` | 是否目录 |
| `fileSize(path)` | `(string): int` | 文件大小，失败返回 `-1` |
| `remove(path)` | `(string): bool` | 删除文件或空目录 |
| `rename(old, new)` | `(string, string): bool` | 重命名/移动 |
| `mkdir(path)` | `(string): bool` | 创建单级目录 |
| `readDir(path)` | `(string): Array<string>?` | 读取目录 basename 列表 |
| `cwd()` | `(): string?` | 当前工作目录 |
| `chdir(path)` | `(string): bool` | 修改进程级 cwd |
| `copyFile(src, dst)` | `(string, string): bool` | 拷贝文件 |
| `readLines(path)` | `(string): Array<string>?` | 按行读取，去掉行尾换行 |
| `isSymlink(path)` | `(string): bool` | 是否符号链接 |
| `stat(path)` | `(string): FileStat?` | 文件状态 Json/handle shape |
| `mkdirp(path)` | `(string): bool` | 递归创建目录 |
| `removeAll(path)` | `(string): bool` | 递归删除目录树 |
| `chmod(path, mode)` | `(string, int): bool` | 修改权限 |
| `touch(path)` | `(string): bool` | 创建空文件或更新时间戳 |
| `symlink(target, link)` | `(string, string): bool` | 创建符号链接 |
| `readlink(path)` | `(string): string?` | 读取符号链接目标 |
| `realpath(path)` | `(string): string?` | 解析真实绝对路径 |
| `tempFile()` | `(): string?` | 创建临时文件并返回路径 |
| `tempDir()` | `(): string?` | 创建临时目录并返回路径 |
| `readDirRecursive(path)` | `(string): Array<string>?` | 递归读取相对路径列表 |

## 当前架构优点

- API 覆盖面完整，已经覆盖常见文件、目录、stat、临时路径、符号链接和递归操作。
- runtime 导出和 analyzer generated 表基本同步，`g_gen_io_functions` 覆盖 29 个函数，`FileStat` handle 字段也已生成。
- `stat()` 使用 per-isolate stdlib cache 缓存 shape，避免进程级可变全局和重复构造。
- `readFileBytes/writeFileBytes` 使用 typed `Array<uint8>`，适合二进制数据。
- `readDir` 通过 `xr_dir_*` 过滤 `.` 和 `..`，并封装 Unix/Windows 差异。
- `readDirRecursive` 明确不跟随 symlink，降低递归遍历逃逸风险。
- `removeAll` 使用 `FTW_PHYS`，避免跟随 symlink 删除树外目标。
- `mkdirp` 已拒绝空 path，并检查 `PATH_MAX` 截断风险。
- `copyFile` 在 macOS 使用 `fcopyfile`，Linux 使用 `sendfile`，常见平台上具备较好性能。
- `common_io.h` 已把多个解析模块的同步整文件读写集中到一个 helper，后续迁移 async I/O 有统一替换点。

## API 与工具链漂移

### 问题 1：LSP 的 io API 明显落后

LSP 当前列出 24 个 io symbol，缺少 runtime/analyzer 中已有的：

```text
readFileBytes
writeFileBytes
isSymlink
readlink
touch
```

同时多处签名不准确：

- `readFile` LSP 是 `string`，实际是 `string?`。
- `writeFile/appendFile/remove/rename/mkdir/mkdirp/chdir/copyFile/removeAll/chmod/symlink` LSP 未标返回 `bool`。
- `readDir/readDirRecursive/readLines` LSP 未体现失败时可能返回 `null`。
- `stat` LSP 是 `Json`，实际 analyzer 是 `FileStat?`。
- `realpath/tempFile/tempDir` LSP 是非空 string，实际是 `string?`。

影响：

- IDE 补全会漏函数。
- nullable 和 bool 返回值错误会误导用户写出未处理失败的代码。
- `FileStat` 字段提示不能和 analyzer 保持一致。

建议：

- 用 generated builtin metadata 统一生成 LSP stdlib symbols。
- 先手动补齐 io LSP，确保和 `XR_DEFINE_BUILTIN` 一致。
- 增加测试：runtime export、analyzer registry、LSP symbol 三者函数集合一致。

## 同步阻塞与调度边界

### 问题 2：所有 I/O 都是同步阻塞

`io.c` 文件头已说明当前调用会阻塞 worker thread。实际实现大量使用：

```text
fopen/fread/fwrite/fclose
fseek/ftell
copyfile/sendfile
opendir/readdir/stat/lstat
nftw
realpath/readlink/mkstemp/mkdtemp
```

影响：

- 在 coroutine worker 上读取大文件、慢磁盘、网络挂载目录时会阻塞整个 worker。
- `readDirRecursive/removeAll/copyFile` 对大目录树或大文件尤其明显。
- 与 `net` 模块的 coroutine-friendly I/O 语义不一致。

建议：

- 增加 async job pool，文件系统操作通过 yieldable C function 挂起当前 coroutine。
- 分层提供：`io.readFileSync` 与 `io.readFile`，或保留现有同步语义并新增 `io.readFileAsync`。
- 优先迁移长耗时操作：`readFile/readFileBytes/writeFile/writeFileBytes/copyFile/readDirRecursive/removeAll`。

### 问题 3：整文件读取缺少 streaming API

`readFile/readFileBytes/readLines` 都一次性读取全部内容。虽然有 `IO_MAX_READ_BYTES` 上限，但用户无法流式处理大文件。

建议：

- 引入 `File` handle：`open/read/write/seek/close`。
- 提供 iterator/channel 风格 `lines(path)`。
- 对大文件给出明确错误，而不是仅返回 `null`。

## 错误模型

### 问题 4：失败大多折叠成 `false/null/-1`

当前错误结果：

- 读取失败：`null`
- 写入失败：`false`
- `fileSize` 失败：`-1`
- `stat` 失败：`null`
- 大文件、权限不足、路径过长、类型错误、I/O 中断、磁盘满等无法区分。

影响：

- 用户无法做可靠错误处理。
- 测试只能验证成功/失败，不能验证失败原因。
- 调试体验差。

建议：

- 增加统一 `io.lastError()` 或返回 `{ ok, value, error }` 的扩展 API。
- C 层保留 errno/GetLastError，并映射到 `XrIoError`。
- `fileSize` 改为 `int?` 或新增 `stat(path).size` 作为推荐路径。

### 问题 5：写入没有检查 `fclose` 错误

`writeFile/writeFileBytes/appendFile` 只检查 `fwrite` 字节数，随后直接 `fclose`，没有把 close/flush 失败纳入返回值。磁盘满、NFS 延迟写失败可能只在 close 时暴露。

`common_io.h` 的 `xrs_file_write_all_sync` 已正确检查 `fclose`，但 `stdlib/io/io.c` 没有复用。

建议：

- `writeFile/writeFileBytes/appendFile` 复用 `xrs_file_write_all_sync` 或同等逻辑。
- 对 append 增加专用 helper，检查 `fclose`。

### 问题 6：读取没有严格检查 short read/ferror

`readFile/readFileBytes` 根据 `ftell` 得到 size 后调用 `fread`，但没有检查 `ferror` 或 `read_size == size`。如果读到一半出错，会返回截断内容。

`common_io.h` 的读 helper 已检查 `got != size || ferror(f)`。

建议：

- `readFile` 复用 `xrs_file_read_all_sync`。
- `readFileBytes` 增加 `ferror` 和 short read 检查。
- 对普通文件以外的路径返回明确错误。

## 路径与跨平台边界

### 问题 7：`io.c` 仍大量直接使用 POSIX API

虽然项目已有 `src/os/os_fs.h` 和 `src/os/os_dir.h`，但 `io.c` 仍直接包含/使用：

```text
unistd.h
fcntl.h
utime.h
ftw.h
copyfile.h / sendfile.h
remove/rename/mkdir/stat/lstat/chmod/symlink/readlink/realpath/nftw
```

影响：

- Windows 兼容性弱，`symlink/readlink/chmod/nftw/utime` 等都需要平台分支或 shim。
- 跨平台语义分散：一部分通过 `xr_fs_*`，一部分绕过 shim。
- 错误映射无法统一。

建议：

- 扩展 `os_fs` shim：copy、removeAll、chmod、touch、symlink/readlink、temp file/dir。
- `io.c` 尽量只调用 `xr_fs_*` / `xr_dir_*`。
- Windows UTF-8 路径最终应使用 wide-char API，而不是 ANSI `A` 系列。

### 问题 8：`chdir` 是进程级状态，和 isolate/coroutine 不隔离

`io.chdir()` 改的是进程当前工作目录，不是 isolate 局部状态。多个 isolate 或 coroutine 同时使用相对路径时，会互相影响。

影响：

- 并发测试可能不稳定。
- 服务端场景中，一个请求调用 `chdir` 会影响所有请求。

建议：

- 将 `chdir` 标记为危险 API，文档说明进程级副作用。
- 更推荐 `path.resolve(base, rel)` 或显式传绝对路径。
- 长期可引入 isolate-level virtual cwd，但需系统性改造所有相对路径解析。

### 问题 9：`PATH_MAX` 策略不统一

`mkdirp/readlink/realpath/tempFile/tempDir/readDirRecursive` 多处使用 `PATH_MAX` 栈 buffer。`readDir` 的底层 `XrDirEntry.name` 也限制 basename 长度为 260。

影响：

- Unix 上合法但超长路径可能失败或被跳过。
- `readDirRecursive` 对路径过长的 entry 静默跳过。
- Windows 260 限制与现代长路径模式不一致。

建议：

- 对外明确 path length limit。
- 内部逐步改为动态 buffer。
- 被跳过的 entry 应提供错误/计数/可选严格模式。

## 文件内容与内存边界

### 问题 10：`readFile` 使用 intern string，可能污染 intern table

`readFile` 和 `readLines` 都用 `xr_string_intern`。文件内容通常是高基数、大字符串，不适合进入 intern table。

影响：

- 大量读取不同文件会增加 intern table 压力。
- 日志/模板/用户上传内容等长字符串不应默认 intern。

建议：

- 文件内容创建普通 heap string，不默认 intern。
- 只有路径、字段名、短枚举等低基数字符串适合 intern。

### 问题 11：`readFile` 对文本编码没有定义

`readFile` 只是按字节读入并构造 Xray string，没有 UTF-8 验证、BOM 处理或编码转换。

建议：

- 文档明确 `readFile` 假定 UTF-8/opaque bytes。
- 推荐二进制数据使用 `readFileBytes`。
- 可新增 `readText(path, encoding?)`。

### 问题 12：临时文件只返回路径，不返回已打开 handle

`tempFile()` 使用 `mkstemp` 安全创建后立即关闭 fd，再返回路径。后续用户再 `writeFile` 会重新打开，存在 TOCTOU 窗口，虽然随机文件名降低风险。

建议：

- 短期文档说明 `tempFile` 返回的是已存在空文件路径。
- 长期提供 `createTempFile()` 返回 open File handle。

## 递归操作与安全边界

### 问题 13：`removeAll` 风险较高但没有保护栏

`removeAll(path)` 递归删除，没有拒绝根目录、空字符串以外的危险路径、当前工作目录或 home 目录等。当前只要用户传入合法路径就执行。

建议：

- 拒绝 `"/"`、`"."`、`".."`、空路径以及解析后过短/高风险路径。
- 增加 `force` 或 `allowRoot` 之类显式选项。
- 测试 symlink 不跟随、根路径拒绝、路径不存在行为。

### 问题 14：`readDirRecursive` 静默忽略错误

递归读取中：

- 打不开目录直接返回。
- 路径过长 entry 直接跳过。
- 深度超过 64 直接停止。
- `lstat` 失败不报错。

这对 best-effort 列表可接受，但不适合需要完整目录快照的场景。

建议：

- 提供 `readDirRecursiveDetailed`，返回 `{ entries, errors, truncated }`。
- 或增加 options：`strict`, `maxDepth`, `followSymlinks`。

## Analyzer 与类型系统

### 当前状态

analyzer generated 中 `io` 已注册：

- `GEN_IO_FUNCTION_COUNT = 29`
- `FileStat` handle 10 个字段：`size/mode/mtime/atime/ctime/uid/gid/isFile/isDir/isSymlink`

这比 LSP 更接近 runtime，是当前较好的单一事实来源。

### 问题 15：`readDir` builtin 类型未体现 nullable

`XR_DEFINE_BUILTIN(io_readDir, "readDir", "(path: string): Array<string>", ...)`，但实现失败返回 `null`。

类似问题包括：

- `readLines` 实现失败返回 `null`，builtin 是 `Array<string>`。
- `readDirRecursive` 实现失败可返回 `null`，builtin 是 `Array<string>`。
- `cwd` 实现可返回 `null`，builtin 是 `string`。

建议：

- builtin 签名改为 `Array<string>?` / `string?`。
- 或实现保证失败时返回空数组/空字符串，但这会隐藏错误，不推荐。

## 测试覆盖

已有测试：

| 文件 | 覆盖内容 |
|---|---|
| `1190_io_basic.xr` | tempFile、read/write、readLines、append、stat、fileSize、mkdirp、readDir、copy、rename、touch、cwd、realpath、removeAll |
| `1191_io_edge.xr` | 不存在路径、空文件、覆盖写、大字符串、readDirRecursive、remove 不存在、readLines 空/单行 |
| `1192_io_extended.xr` | 基础组合场景、chmod、chdir、recursive dir |

主要缺口：

1. `readFileBytes/writeFileBytes` 二进制往返。
2. `isSymlink/symlink/readlink` 符号链接行为。
3. `stat` 对 symlink 的 `isSymlink` 字段。
4. `readDir` 是否过滤 `.` 和 `..`。
5. `readDirRecursive` 不跟随 symlink。
6. `removeAll` 不跟随 symlink，且拒绝危险路径。
7. 写入时 `fclose` 失败/short write 的错误路径。
8. 读取时 short read/ferror 的错误路径。
9. `mkdirp` 对空 path、重复 path、trailing slash、过长 path。
10. `copyFile` 大文件、空文件、目标已存在、跨文件系统、权限失败。
11. `tempFile/tempDir` 环境变量 `TMPDIR/TMP/TEMP` 和路径过长。
12. `chdir` 并发副作用或至少恢复 cwd 的测试约束。
13. analyzer/LSP/runtime API 集合同步测试。
14. Windows path、Unicode path、separator、symlink privilege 行为。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | LSP 缺函数且签名错误 | IDE 误导用户 | 由 builtin metadata 生成 LSP |
| 高 | 同步 I/O 阻塞 worker | 大文件/慢磁盘影响调度 | 接入 async job pool 或提供 async API |
| 高 | 写入不检查 `fclose` | 磁盘满/NFS 错误漏报 | 复用 common write helper |
| 高 | 读取不检查 short read/ferror | 可能返回截断内容 | 复用 common read helper |
| 高 | `removeAll` 缺保护栏 | 误删风险 | 拒绝危险路径，增加 options |
| 中 | `io.c` 直接依赖 POSIX | Windows/跨平台弱 | 扩展 os_fs/os_dir shim |
| 中 | builtin nullable 不准确 | 静态分析误导 | 修正 `readDir/readLines/cwd` 等签名 |
| 中 | `chdir` 是进程级副作用 | 并发 isolate 相互影响 | 标记危险或引入 virtual cwd |
| 中 | 文件内容默认 intern | intern table 压力 | 改普通字符串 |
| 中 | 无 streaming API | 大文件不可扩展 | 增加 File handle/line iterator |
| 中 | `PATH_MAX`/260 basename 限制 | 合法长路径失败/跳过 | 动态 buffer 和明确错误 |
| 低 | `readFile` 编码未定义 | 文本/二进制边界模糊 | 文档化并新增 encoding API |
| 低 | `tempFile` 返回路径而非 handle | 有 TOCTOU 窗口 | 提供 open temp handle |

## 后续实施建议

建议优先顺序：

1. 修正 LSP io symbols，使其和 `XR_DEFINE_BUILTIN` / runtime 导出一致。
2. 修正 builtin nullable 签名：`readDir/readLines/readDirRecursive/cwd` 等。
3. 让 `readFile/writeFile/appendFile` 复用 `common_io.h` 的严格读写 helper，补齐 short read、ferror、fclose 检查。
4. 为 `readFileBytes/writeFileBytes` 增加同等严格错误检查。
5. 给 `removeAll` 增加危险路径保护和 symlink 安全测试。
6. 增加 `readFileBytes/writeFileBytes/symlink/readlink/isSymlink/readDirRecursive` 测试。
7. 将 POSIX-only 操作下沉到 `os_fs` shim，统一错误映射和 Windows 支持。
8. 明确 `chdir` 进程级副作用，并避免在并发测试中依赖相对路径。
9. 引入 `File` handle 和 streaming/async API，减少 worker 阻塞和整文件内存压力。
10. 把文件内容从 intern string 改为普通 string，避免 intern table 承载高基数数据。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
