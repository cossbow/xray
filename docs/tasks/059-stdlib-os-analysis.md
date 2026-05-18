# stdlib/os 分析与优化建议

## 模块职责

`stdlib/os` 是标准库的操作系统接口模块，提供环境变量、进程信息、当前工作目录、主机/用户信息、系统资源信息、信号、睡眠、命令执行和平台常量。

它和 `stdlib/io` 的边界不同：

- `io` 主要面向文件系统路径和文件内容。
- `os` 主要面向进程级状态、宿主 OS 信息和进程控制。

当前 `os` 模块是高权限模块，很多 API 都有进程级副作用：

- `exit()` 直接终止整个进程。
- `setenv/unsetenv/environ` 操作进程环境。
- `chdir` 修改进程当前工作目录。
- `kill` 可向任意 pid 发送信号。
- `exec` 通过 shell 执行命令。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/os/os.h` | os 模块职责说明和加载入口 |
| `stdlib/os/os.c` | runtime 绑定、平台分支、builtin declarations、模块常量导出 |
| `src/os/os_fs.h` | cwd/chdir 等文件系统 shim，被 `os.getcwd/chdir` 使用 |
| `src/module/xmodule_loaders.h` | os 模块注册在 filesystem 模块组 |
| `src/module/xmodule.c` | `stdlib_filesystem` 中注册 `os` loader |
| `src/frontend/analyzer/xanalyzer_builtins_generated.h` | os builtin 函数静态分析表 |
| `src/app/lsp/xlsp_stdlib.c` | LSP os symbol 手写表 |

## 当前脚本 API

`xr_load_module_os()` 实际导出 24 个函数和 4 个常量。

### 函数

| API | 当前签名 | 语义 |
|---|---|---|
| `getenv(name)` | `(string): string?` | 获取环境变量 |
| `setenv(name, value)` | `(string, string): bool` | 设置环境变量，覆盖已有值 |
| `unsetenv(name)` | `(string): bool` | 删除环境变量 |
| `environ()` | `(): Map<string, string>` | 复制当前环境变量到 Map |
| `exit(code?)` | `(int?): void` | 终止整个进程 |
| `getpid()` | `(): int` | 当前进程 pid |
| `getcwd()` | `(): string?` | 当前工作目录 |
| `chdir(path)` | `(string): bool` | 修改进程当前工作目录 |
| `hostname()` | `(): string?` | 主机名 |
| `tmpdir()` | `(): string` | 临时目录候选路径 |
| `username()` | `(): string?` | 当前用户名称 |
| `homedir()` | `(): string?` | 当前用户 home 目录 |
| `uid()` | `(): int` | 用户 id；Windows 当前返回 0 |
| `gid()` | `(): int` | group id；Windows 当前返回 0 |
| `cpuCount()` | `(): int` | 在线 CPU 数 |
| `totalMemory()` | `(): int` | 总内存字节数；不支持平台返回 0 |
| `freeMemory()` | `(): int` | 可用内存字节数；不支持平台返回 0 |
| `uptime()` | `(): float` | 系统启动时长秒数 |
| `loadavg()` | `(): Array<float>` | 1/5/15 分钟负载 |
| `ppid()` | `(): int` | 父进程 pid；Windows 当前返回 0 |
| `kill(pid, signal?)` | `(int, int?): bool` | 发送信号；Windows 当前返回 false |
| `sleep(ms)` | `(int): void` | coroutine-friendly timer sleep |
| `clock()` | `(): float` | 进程 CPU 时间 |
| `exec(cmd)` | `(string): Map<string, any>?` | shell 执行命令，返回 stdout/stderr/exitCode |

### 常量

| 常量 | 语义 |
|---|---|
| `platform` | `darwin/linux/windows/freebsd/openbsd/unknown` 中的一个 |
| `arch` | `arm64/x64/x86/arm/unknown` 中的一个 |
| `sep` | 路径分隔符 |
| `eol` | 行尾符 |

## 当前架构优点

- API 覆盖了 OS 模块常见能力：env、process、cwd、user、sysinfo、exec、sleep、platform constants。
- `sleep(ms)` 已经是 yieldable C function，使用 timer wheel，不会阻塞 worker thread。
- `getcwd/chdir` 复用了 `xr_fs_getcwd/xr_fs_chdir`，避免直接散落平台分支。
- analyzer generated 表覆盖 24 个 runtime 函数，并注册在 builtin module registry 中。
- 测试已有基础/扩展覆盖，包括 env、platform、cwd、sysinfo、exec stdout/stderr/exitCode。
- macOS/Linux/Windows 都有一定平台分支，基础信息 API 有降级返回。

## API 与工具链漂移

### 问题 1：LSP 的 os API 严重落后

LSP 当前只列：

```text
getenv setenv exec exit args platform
```

实际 runtime 有：

```text
getenv setenv unsetenv environ exit getpid getcwd chdir hostname tmpdir
username homedir uid gid cpuCount totalMemory freeMemory uptime loadavg
ppid kill sleep clock exec
```

LSP 缺失 20 多个函数和常量：

```text
unsetenv environ getpid getcwd chdir hostname tmpdir username homedir
uid gid cpuCount totalMemory freeMemory uptime loadavg ppid kill sleep clock
arch sep eol
```

同时 LSP 有一个 `args` 变量，但 runtime `xr_load_module_os()` 没有导出 `args`。

签名漂移：

- `setenv` LSP 无返回值，实际返回 `bool`。
- `exec` LSP 是 `fn(command: string): int`，实际返回 `Map<string, any>?`。
- `exit` LSP 要求 `code: int`，实际 code 可选。
- `platform` LSP 只有常量本身，未覆盖 `arch/sep/eol`。

建议：

- 用 generated builtin metadata 统一生成 LSP。
- 如果要提供 `os.args`，必须在 runtime/analyzer 中真实导出；否则从 LSP 删除。
- 增加 runtime/analyzer/LSP API 集合同步测试。

### 问题 2：analyzer 未覆盖模块常量

analyzer generated 表只包含 `XR_DEFINE_BUILTIN` 生成的函数，`platform/arch/sep/eol` 是 runtime 中用 `xr_module_add_export` 添加的常量，不在当前 `g_gen_os_functions` 中。

影响：

- 静态分析可能无法准确识别 `os.platform/os.arch/os.sep/os.eol`。
- LSP 与 analyzer 的常量来源不统一。

建议：

- builtin declaration 机制增加 constants 支持。
- 或为常量提供统一的 `XR_DEFINE_BUILTIN_CONST`，由 analyzer/LSP 共同消费。

## `os.exec` 风险分析

### 问题 3：Unix 实现顺序读取 stdout/stderr，存在管道死锁

Unix 路径中：

1. 创建 stdout pipe 和 stderr pipe。
2. fork 子进程。
3. 父进程先 `read_fd_to_string(stdout_pipe[0])`。
4. 再 `read_fd_to_string(stderr_pipe[0])`。
5. 最后 `waitpid`。

如果子进程向 stderr 输出大量数据并填满 stderr pipe，同时 stdout 没有关闭，子进程会阻塞在写 stderr；父进程则还在等 stdout EOF，双方死锁。

典型触发：

```bash
python -c 'import sys; sys.stderr.write("x" * 1000000)'
```

影响：

- `os.exec()` 可永久卡住 worker thread。
- 回归测试只覆盖小 stdout/stderr，无法发现该问题。

建议：

- 使用 non-blocking pipe + poll/select 同时 drain stdout/stderr。
- 或启动两个 reader job/coroutine 并发读取。
- 增加 max output limit，超限后 kill 子进程并返回错误。

### 问题 4：`os.exec` 是同步阻塞调用

`exec` 当前是普通 `XRS_EXPORT`，执行期间会阻塞当前 worker thread：

- fork/exec/popen 阻塞。
- 读取 stdout/stderr 阻塞。
- `waitpid(pid, &status, 0)` 阻塞。

与 `os.sleep` 已 yieldable 的设计不一致。

建议：

- 将 `exec` 改为 yieldable C function，通过 async process job 或 netpoll/readable fd 驱动。
- 至少提供 `execSync` / `exec` 的明确语义区分。
- 增加 `timeoutMs` 参数。

### 问题 5：`os.exec(cmd)` 固定走 shell

Unix 使用：

```text
/bin/sh -c cmd
```

Windows 使用 `_popen(cmd, "r")`。

这符合 shell convenience API，但安全边界必须明确：

- 任何未转义用户输入都会造成命令注入。
- 环境变量、cwd、PATH 都影响执行结果。
- 返回 Map 不携带启动失败/timeout/signal 的结构化错误。

建议：

- 保留 `exec(cmd)` 作为 shell API，但文档明确不接受未信任输入。
- 新增 `spawn(program, args, options)`，不经 shell。
- 支持 `cwd/env/stdin/timeout/maxOutput`。

### 问题 6：输出无上限，可能耗尽内存

`read_fd_to_string` 会持续 `xr_realloc` 扩容直到 EOF，没有 output cap。Windows `_popen` 路径也一样。

建议：

- 默认限制 stdout/stderr 总字节数。
- 超限返回 `{stdout, stderr, exitCode, truncated: true}` 或错误。
- 测试大输出场景。

### 问题 7：Windows `exec` 语义弱于 Unix

Windows 路径：

- 只捕获 stdout。
- stderr 固定为空字符串。
- exit code 解析较粗糙。
- 没有和 Unix 相同的 `stderr` 测试能力。

建议：

- 统一实现为 spawn + pipe stdout/stderr。
- Windows 使用 `CreateProcessW` 和匿名管道。
- 所有平台都返回相同结构。

## 进程级副作用边界

### 问题 8：环境变量是进程全局状态

`setenv/unsetenv/environ/getenv` 操作的是整个进程环境，不是 isolate 局部状态。

影响：

- 多 isolate 或多 coroutine 并发读写 env 会互相影响。
- 测试间共享环境变量，容易产生顺序依赖。
- `environ()` 遍历时若其他线程同时 set/unset，可能看到不一致状态。

建议：

- 文档明确环境变量 API 是进程级副作用。
- 测试使用唯一前缀并清理。
- 长期可提供 `ProcessOptions.env` 给 `spawn`，避免修改全局 env。

### 问题 9：`chdir` 是进程全局状态

`os.chdir()` 与 `io.chdir()` 一样修改进程 cwd。所有相对路径解析都会受影响。

影响：

- 并发服务中某个 coroutine 调用 `chdir` 会影响其他 coroutine。
- 测试如果失败未恢复 cwd，会污染后续测试。

建议：

- 标记为危险 API，文档强调进程级副作用。
- 测试使用 `defer` 或等价清理策略恢复 cwd。
- 推荐业务代码用绝对路径和 `path.resolve`。

### 问题 10：`exit` 和 `kill` 需要更强安全边界

`os.exit()` 直接终止整个宿主进程，`os.kill(pid, sig)` 可以向任意进程发送信号。

风险：

- embedding 场景中脚本可终止宿主应用。
- 测试误用 `kill` 可能杀掉无关进程。
- `kill(0, sig)`、负 pid 等 POSIX 特殊语义没有防护。

建议：

- 模块能力分级：默认禁用危险 API，或由 isolate policy 显式开启。
- `kill` 拒绝 `pid <= 0`，除非显式 allowProcessGroup。
- 测试中只对当前进程使用 signal 0 这类无害检查，或 mock。

## 平台与系统信息边界

### 问题 11：部分平台返回值具有误导性

当前 Windows 路径：

- `uid()` 返回 0。
- `gid()` 返回 0。
- `ppid()` 返回 0。
- `kill()` 返回 false。
- `loadavg()` 返回 `[0, 0, 0]`。

这些是降级实现，但脚本层无法区分“不支持”和“真实值为 0”。

建议：

- 不支持平台返回 `null` 或增加 `isSupported(name)`。
- 对 `uid/gid/ppid` 使用 nullable 签名。
- `loadavg` 在 Windows 返回 `null` 或明确标记 unsupported。

### 问题 12：`get_platform()` 有重复 BSD 条件

`get_platform()` 中连续两个分支都判断 `XR_OS_BSD`：

```text
XR_OS_BSD -> freebsd
XR_OS_BSD -> openbsd
```

第二个分支不可达。若需要区分 FreeBSD/OpenBSD，应使用不同平台宏。

建议：

- 使用明确宏：`XR_OS_FREEBSD`、`XR_OS_OPENBSD`。
- 测试 platform 常量允许这些值。

### 问题 13：`get_arch()` 覆盖不足

当前支持：`arm64/x64/x86/arm/unknown`。未覆盖 riscv64、ppc64、s390x、wasm 等潜在目标。

建议：

- 补齐平台宏映射。
- analyzer/LSP 文档说明可能返回 `unknown`。

### 问题 14：`hostname()` 可能缺少截断处理

POSIX `gethostname(buf, sizeof(buf))` 在某些平台上截断时不保证 NUL 终止。当前直接 `xrs_string_value_c(X, buf)`。

建议：

- 初始化 buffer 或强制 `buf[sizeof(buf)-1] = '\0'`。
- 截断时返回错误或动态扩容。

### 问题 15：系统资源信息没有错误区分

`totalMemory/freeMemory/uptime/loadavg` 失败时返回 0 或 `[0,0,0]`，无法区分真实值与失败。

建议：

- 使用 nullable 返回或 result object。
- `loadavg` 检查 `getloadavg` 返回值。
- `totalMemory` macOS `sysctlbyname` 应检查返回值。

## 内存与字符串边界

### 问题 16：`environ()` 对 key 使用 intern

环境变量 key 数量较小，intern key 可以接受。但 value 使用普通 `xrs_string_value_c`，这是合理的。需要确认 `xrs_string_value_c` 是否 intern，如果也 intern，则大量动态 env value 可能污染 intern table。

建议：

- key 可 intern，value 不建议 intern。
- 在 string API 层明确 `xrs_string_value_c` 的 intern/heap 行为。

### 问题 17：`read_fd_to_string` 对 read error 处理不足

`read_fd_to_string` 只在 `read > 0` 时追加；`read < 0` 直接退出循环并返回已有 buffer，没有区分 EINTR/EAGAIN/真实错误。

建议：

- `EINTR` 重试。
- 真实错误返回失败并释放 buffer。
- 对 nonblocking fd 时处理 EAGAIN。

## 测试覆盖

已有测试：

| 文件 | 覆盖内容 |
|---|---|
| `1160_os_basic.xr` | platform/arch/sep/eol、getenv/setenv/unsetenv/environ、getpid/getcwd/hostname/tmpdir/chdir |
| `1161_os_extended.xr` | username/homedir/uid/gid、cpu/memory/uptime/loadavg、ppid/clock、exec stdout/stderr/exitCode、env 返回值、invalid chdir |

主要缺口：

1. `os.sleep` 确认不阻塞其他 coroutine。
2. `os.exec` 大 stdout、大 stderr、stdout+stderr 混合输出，覆盖管道死锁风险。
3. `os.exec` timeout/长运行命令，目前没有 API，至少应记录缺口。
4. `os.exec` 启动失败、signal termination、命令不存在、空命令。
5. Windows exec stderr/exitCode 行为。
6. `kill` 安全行为：无效 pid、signal 0、自身 pid、负 pid 拒绝。
7. `exit` 不应在普通回归中直接调用，需要 subprocess 测试。
8. `environ` 在 setenv/unsetenv 后快照一致性。
9. `chdir` 测试失败时恢复 cwd 的可靠性。
10. `tmpdir` 返回路径是否存在、是否可写。
11. `hostname` 长主机名/截断处理。
12. `platform/arch` 新架构值。
13. analyzer/LSP/runtime API 集合同步。
14. 模块常量在 analyzer 中的类型识别。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 高 | LSP os API 严重缺失且 `args` 是幽灵符号 | IDE 误导用户 | 由 builtin metadata 生成 LSP，删除或实现 `args` |
| 高 | `exec` 顺序读 stdout/stderr 可能死锁 | 命令输出稍大即可卡住 | poll/select 同时 drain 两个 pipe |
| 高 | `exec` 同步阻塞 worker | 长命令影响调度 | 改 yieldable 或提供 async/spawn API |
| 高 | `exec` 通过 shell 执行 | 命令注入风险 | 增加不经 shell 的 `spawn(program,args)` |
| 高 | `exec` 输出无上限 | 内存耗尽风险 | 增加 max output limit |
| 高 | `exit/kill/chdir/setenv` 是进程级副作用 | embedding/并发不安全 | isolate capability policy |
| 中 | analyzer 不覆盖常量 | 静态分析不完整 | builtin const declaration |
| 中 | Windows 多个 API 返回 0/false 占位 | 无法区分 unsupported | nullable 或 support query |
| 中 | `get_platform` BSD 分支不可达 | 平台识别错误 | 使用具体 BSD 宏 |
| 中 | `hostname` 截断/NUL 处理不足 | 潜在越界读/错误值 | 强制 NUL 或动态 buffer |
| 中 | `read_fd_to_string` 不处理 read error | 截断输出被当成功 | EINTR 重试，错误返回 |
| 低 | `tmpdir` 不验证存在/可写 | 后续临时文件失败 | 返回 candidate 或 validate 版本 |
| 低 | `arch` 覆盖不足 | 新架构显示 unknown | 补宏映射 |

## 后续实施建议

建议优先顺序：

1. 修正 LSP os symbols：补齐 24 个函数和 `arch/sep/eol`，删除或实现 `args`。
2. 为 builtin declaration 增加模块常量，使 analyzer/LSP/runtime 共享同一事实来源。
3. 修复 `os.exec`：同时读取 stdout/stderr，处理 EINTR/read error，检查 waitpid EINTR。
4. 给 `os.exec` 增加 `timeoutMs/maxOutputBytes/cwd/env` options。
5. 新增 `spawn(program, args, options)`，避免 shell 注入。
6. 把 `exec` 改成 yieldable，避免阻塞 worker。
7. 引入 isolate/module capability policy，控制 `exit/kill/exec/chdir/setenv` 等危险 API。
8. 将 Windows unsupported API 改成 nullable 或显式 unsupported error。
9. 修复 `hostname/platform/arch/loadavg/totalMemory` 的错误处理和平台覆盖。
10. 增加 os 模块并发、副作用、exec 大输出和 LSP/analyzer 同步测试。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
