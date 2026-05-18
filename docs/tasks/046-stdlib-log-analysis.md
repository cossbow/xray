# stdlib/log 分析与优化建议

## 模块职责

`stdlib/log` 提供结构化日志能力：

- 多级别日志：debug/info/warn/error/fatal
- text/json 两种输出格式
- 默认 logger 与 child logger
- source location 输出
- stdout/stderr/file 输出目标
- async logging 和 flush

它不是普通纯函数库，而是带 I/O、后台线程、per-isolate 状态、native type、GC destroy hook 的运行时能力模块。后续重构需要重点保证并发安全、资源生命周期和错误模型一致。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/log/log.c` | logger 状态、格式化、异步队列、binding、native type 注册 |
| `stdlib/log/log.h` | level/format/logger 结构和 loader 声明 |
| `stdlib/stdlib_cache.h` | per-isolate log state 存储 |
| `stdlib/ctxbuf.h` | 日志行动态 buffer |
| `src/os/os_thread.h` | async queue 线程、mutex、cond |
| `tests/regression/10_stdlib/1200_log_basic.xr` | 基础 level、输出、format 回归 |
| `tests/regression/10_stdlib/1201_log_child.xr` | child logger 回归 |
| `tests/regression/10_stdlib/1202_log_source.xr` | source location 回归 |
| `tests/regression/10_stdlib/1203_log_async.xr` | async/flush 回归 |

## 当前 API

| API | 当前语义 |
|---|---|
| `debug/info/warn/error/fatal(...args)` | 输出日志，fatal 会 flush 后 `exit(1)` |
| `setLevel(level)` | 设置默认 logger level，支持 int 或 string |
| `setFormat(format)` | 设置 text/json |
| `setOutput(path)` | 设置 stdout/stderr 或 append 文件 |
| `isEnabled(level)` | 检查默认 logger 是否启用给定 level |
| `enableSource(enabled)` | 开启 source location |
| `enableAsync(enabled)` | 开启/关闭异步队列 |
| `flush()` | 等待 async queue drain |
| `child(...fields)` | 创建 child logger，继承上下文 |

常量：`DEBUG=10`、`INFO=20`、`WARN=30`、`ERROR=40`、`FATAL=50`。

## 当前架构优点

- mutable log state 已收敛到 `XrStdlibCache::log_state`，不是进程全局变量。
- async queue 使用 per-isolate 状态，避免多个 isolate 直接共享 ring buffer。
- async producer 满队列时采用 drop-newest，不会阻塞 coroutine worker。
- `log_state_destroy()` 挂在 stdlib cache cleanup，能停止 async thread 并关闭自定义文件。
- JSON string escape 会处理 invalid UTF-8，避免输出非法 JSON 文本。

## 依赖与架构边界

### 问题 1：`log.c` 依赖 VM/isolate internal

`log.c` include 并使用：

- `xvm.h`
- `xisolate_internal.h`
- `xisolate_api.h`
- `xalloc_unified.h`
- `xnative_type.h`

source location 直接遍历 `isolate->vm.frames`。这让 stdlib/log 与 VM frame layout 强耦合。

建议：

- VM 层提供稳定的 current source location API。
- log 模块只调用 `xr_vm_current_source_location()` 一类接口。
- `XrLoggerRef` 分配和 native type 注册也应通过稳定 runtime API。

### 问题 2：`log.h` 暴露实现结构体

`log.h` 暴露 `XrLogger`、`XrLoggerRef`、context buffer、FILE*、async flags 等内部布局。

影响：

- 外部代码可能依赖内部字段。
- 后续改 async queue 或 context representation 会扩大影响面。

建议：

- 对外只暴露 opaque `XrLogger` / `XrLoggerRef`。
- loader 声明与内部结构分离。
- GC destroy hook 不需要通过 stdlib public header 暴露结构。

### 问题 3：loader 和 GC destroy 可见性需要统一确认

`xr_load_module_log()` 定义没有 `XR_FUNC`。`xr_gc_destroy_logger()` 注释说由 runtime header 声明，但定义处也没有修饰。

建议：

- 与 `xgc_internal.h` 声明对齐，确保定义带同样可见性修饰。
- loader 统一加 `XR_FUNC`。

## 并发与生命周期

### 问题 4：同步写日志没有持有 logger mutex

`setLevel/setFormat/setOutput/enableAsync` 会持有 `ls->mutex` 修改 logger 字段。但 `xr_log_write_ex()` 读取：

- `logger->level`
- `logger->format`
- `logger->output`
- `logger->add_source`
- `logger->async_mode`
- context 指针

没有统一加锁。

影响：

- 并发写日志和配置变更存在 data race。
- `setOutput()` 关闭旧 FILE* 时，另一个线程可能刚读取旧指针并执行 `fwrite()`。

建议：

- 日志写入开始时在锁内 snapshot 配置和 FILE*。
- 对 FILE* 生命周期做引用计数或延迟关闭。
- 或规定 log 配置只允许初始化阶段变更，并在运行时禁止并发 set。

### 问题 5：`setOutput()` 的旧文件关闭策略仍不安全

当前逻辑是 swap 后关闭旧文件，并注释说明让 in-flight log 继续写。但没有引用计数或锁保护，无法保证没有 in-flight writer 正在使用旧 FILE*。

建议：

- 在同一 mutex 下完成 snapshot/write/close，或引入 output sink refcount。
- async queue 的 `q->output` 也需要跟随更新或独立管理。

### 问题 6：async queue output 与 logger output 可能不同步

`enableAsync(true)` 初始化 queue 时传入当时的 `logger->output`。后续 `setOutput()` 只修改 `logger->output`，不会更新 `async_queue->output`。

影响：

- async 开启后切换输出目标可能不生效。
- flush/stop 期间的 output 生命周期更复杂。

建议：

- `setOutput()` 如果 async 已启用，应同步更新 queue output。
- 或切换输出前自动 flush + stop + restart queue。

### 问题 7：child logger 配置是创建时 snapshot

`create_child_logger()` 复制 parent 的 level/format/output/add_source/async_mode。父 logger 后续 `setLevel/setFormat/setOutput/enableAsync` 不会影响已创建 child。

这可能和“inherit context”的直觉不同。

建议：

- 明确 child logger 继承策略：snapshot 还是 live inheritance。
- 如果期望 live inheritance，child 应引用共享 config，只保存 context。
- 如果保留 snapshot，文档说明。

### 问题 8：child logger 和 async queue 生命周期有交叉风险

child logger 保存 `output` FILE* 指针和 `async_mode` snapshot。如果默认 logger 后续关闭文件或停止 async queue，child logger 仍可能持有旧指针或旧 async_mode。

建议：

- child logger 不直接保存 FILE*，而是保存 sink 引用。
- async state 应位于 shared state，不应按 logger snapshot 存 bool。

## 错误模型

### 问题 9：配置错误直接写 stderr 并返回 null

`setLevel/setFormat/setOutput` 对错误参数或打开失败直接 `fprintf(stderr, ...)`。

影响：

- 标准库错误模型不统一。
- 脚本无法捕获失败。
- `setOutput()` 打开文件失败会回退 stderr，但调用方不知道。

建议：

- 返回 bool 表示是否成功，或抛标准错误。
- `setOutput()` 应暴露失败原因。
- 不要在标准库内部直接写 stderr 作为错误通道。

### 问题 10：`fatal()` 直接 `exit(1)`

`log.fatal()` 和 child logger fatal 会直接退出进程。

影响：

- 嵌入场景不可接受。
- 测试无法安全覆盖。
- isolate 内部脚本 fatal 影响整个宿主进程。

建议：

- fatal 只输出日志，不退出；退出行为交给 CLI/app 层。
- 或提供可配置 panic/exit 策略，默认不退出嵌入宿主。

### 问题 11：奇数个 attributes 静默丢弃最后一个

`extract_log_args()` 使用 `(nargs - 1) / 2`，child logger 也类似。奇数个 key/value 会静默忽略最后一个。

建议：

- 明确 attrs 必须成对。
- 奇数参数返回错误或把最后一个值记为 null。
- key 必须 string 或可 stringify 的规则也需要明确。

## 格式化语义

### 问题 12：JSON attribute key 未转义

JSON 输出里 key 通过：

```c
ctxbuf_printf(&b, ",\"%s\":", key_str);
```

如果 key 包含 `"`、`\`、控制字符，JSON 会非法或被注入字段结构。child logger JSON context 也有类似风险。

建议：

- key 也必须走 JSON string escaping。
- 最好限制 key 为 string，并校验合法字段名。

### 问题 13：Text format value quoting 不转义内部引号

Text format 对需要 quote 的值输出：

```text
key="value"
```

但没有转义内部 `"`、换行等字符，机器解析不稳定。

建议：

- 定义 text log 的 escaping 规则。
- 或明确 text format 仅人读，机器读用 JSON。

### 问题 14：timestamp 使用 localtime，缺少 timezone 标识

`get_timestamp()` 使用 localtime，格式为 `YYYY-MM-DDTHH:MM:SS.mmm`，没有 `Z` 或 offset。

建议：

- JSON 格式默认使用 UTC ISO timestamp。
- Text 格式也可以包含 offset 或明确是 local time。

## 性能与阻塞

### 问题 15：sync logging 是阻塞 I/O，但 builtin 没有慢调用标记

同步模式下 `fwrite()` / `fflush()` / `fopen()` 都可能阻塞 worker。

建议：

- 明确 log API 是否属于 blocking stdlib API。
- 默认启用 async 或把 sync logging 标为 slow/yieldable。
- `setOutput()` 打开文件也应进入 blocking/safe path。

### 问题 16：async flush 会阻塞当前 worker

`flush()` 使用 condvar 等待 queue drain。这个行为合理，但需要文档说明它是阻塞操作。

建议：

- 对 `flush()` 增加超时或返回 dropped count。
- 测试覆盖 flush 在空队列、有队列和 stopped queue 的行为。

### 问题 17：async 队列大小固定且策略不可配置

`ASYNC_QUEUE_SIZE = 256`，满时 drop-newest。策略不错，但调用者无法观测当前 dropped 总数，只能看到后台 synthetic marker。

建议：

- 增加 `log.stats()` 返回 queue size、dropped count、async enabled。
- 允许配置 queue size 或 drop strategy。

## 测试覆盖

现有回归覆盖：

- level 常量和 setLevel/isEnabled 基础。
- debug/info/warn/error 不崩溃。
- text/json format 切换。
- child logger 创建、nested child、child methods。
- source location 开关。
- async enable/flush/disable。

缺口：

1. 输出内容断言：当前多数测试只验证不崩溃。
2. JSON 格式合法性和 key/value escaping。
3. odd attributes 行为。
4. setOutput 到临时文件，检查内容和关闭时机。
5. async setOutput 后是否写到新目标。
6. 并发多 coroutine logging 是否交错、崩溃或 data race。
7. async queue full 后 dropped marker 和统计。
8. child logger 是否跟随父 logger 后续配置变化。
9. fatal 行为是否应该进程退出。
10. source location 行号准确性。
11. log_state cleanup 是否停止线程并释放 child logger/context。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 严重 | `setOutput` 与同步写存在 FILE* 生命周期竞态 | 并发写可能使用已关闭文件 | output sink refcount 或写入锁保护 |
| 高 | JSON key 未转义 | 可产生非法 JSON 或字段注入 | key/value 都走 JSON escape |
| 高 | `fatal()` 直接 `exit(1)` | 嵌入宿主被脚本终止 | fatal 只记录或可配置策略 |
| 高 | async queue output 与 logger output 不同步 | async 下 setOutput 可能失效 | setOutput 同步 queue 或重启 queue |
| 高 | log write 读取配置无锁 | 多 coroutine 配置/写日志 data race | snapshot config 或共享 immutable config |
| 中 | child logger 配置 snapshot 语义不明确 | 父配置变更不影响 child | 文档化或改 live inheritance |
| 中 | blocking I/O 未标记 | worker 可能被日志阻塞 | 标 slow/yieldable 或默认 async |
| 中 | 配置错误写 stderr 且不可捕获 | 错误模型不统一 | 返回 bool 或标准错误 |
| 中 | attrs 奇数参数静默丢弃 | 用户错误被隐藏 | 校验成对参数 |
| 低 | source location 依赖 VM frame internal | VM layout 改动会破坏 log | VM 提供稳定 source API |
| 低 | 注释存在临时阶段编号 | 规则脚本可能拦截后续提交 | 清理源码注释 |

## 后续实施建议

建议先处理会导致数据损坏或进程级副作用的问题：

1. 修复 JSON key escaping。
2. 重构 output sink 生命周期，解决 `setOutput` 与并发写竞态。
3. 明确并修改 `fatal()` 的进程退出策略。
4. 统一 async queue output 与 logger output。
5. 明确 child logger config 继承语义。
6. 为 log API 增加输出内容测试和并发测试。
7. 将 source location 获取封装到 VM 稳定 API。
8. 清理源码中临时阶段编号式注释。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
