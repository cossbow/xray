---
id: spec.16_runtime_model
order: 017
---

<!-- xr-spec:cn -->
---

## 16. 运行时模型 (Runtime Model)

> 真值源：`src/runtime/`、`src/vm/`、`docs/rules/gc-memory.md`、`docs/rules/architecture.md`。

### 16.1 值表示

xray 值统一用 `xray_value_t` 表示。布局策略：

- **NaN-boxing**（在 64 位平台）：double 编码用未使用的 NaN 表示空间存放小整数、bool、指针标记。
- **指针标记**：低位 tag 区分对象类型。
- **对象引用**：堆对象通过 tagged pointer 引用；当前 GC 不移动对象。

| 值类型 | 内部表示 |
|--|--|
| `int` | 53-bit immediate（NaN-box） |
| `float` | 双精度直接存放 |
| `bool` | tag |
| `null` | 单一全局值 |
| `string` | 堆对象 + 短字符串内联（≤ 7 字节） |
| `Bytes` | 堆对象 + capacity/length |
| 其他对象 | 堆指针 |

### 16.2 内存分配

| 区域 | 用途 |
|--|--|
| **系统堆** | C `malloc/free`，用于 native 数据结构 |
| **全局堆** | `shared const` / `shared let`，refcount GC |
| **协程堆** | 每协程独立的 Mark-Sweep GC 堆 |
| **栈** | `struct` 值、局部 immediate、函数帧 |
| **Arena** | parser 临时分配、frame allocation |

### 16.3 GC 模型

- 默认 **per-coroutine Mark-Sweep / Immix Mark-Region**。
- **三色标记**：white（未访问）/ gray（待扫）/ black（已扫）。
- **写屏障**：对象字段、模块字段、静态字段、容器写入 GC 值时触发；Sticky Immix 使用后向屏障维护 young/old 关系。
- **GC-safepoint**：函数调用、后向跳转、显式 `gc.collect()`。

详见 `docs/rules/gc-memory.md`。

### 16.4 协程调度

- M:N 调度（M OS 线程 × N 协程）。
- **work-stealing**：空闲 worker 从其他 worker 队列偷任务。
- **协作式抢占**：协程在 safepoint 让出（非强制抢占）。
- **栈管理**：segmented stack 按需扩展。

详见 `src/runtime/coro/`。

### 16.5 进程级全局访问

- `process`（全局内置，无需 import）：进程自身信息。
- `os`（需 `import os`）：操作系统、环境、进程控制。

```xray
// 进程自身信息 — 全局对象
process.file              // 当前脚本路径（与 __file__ 等价）
process.args              // Array<string>，进程命令行参数
process.dir               // 脚本所在目录（与 __dir__ 等价）

// OS / 环境 — 需 import
import os
os.getenv("PATH")         // 读取环境变量 -> string?
os.environ()              // 获取全部环境变量 -> Map<string, string>
os.exit(0)                // 退出进程
os.getpid()               // 进程 ID
os.getcwd()               // 当前工作目录
os.hostname()             // 主机名
os.tmpdir()               // 临时目录
os.platform               // 常量："darwin" / "linux" / "windows"
os.arch                   // 常量："arm64" / "x64" / "x86"
os.sep                    // 常量：路径分隔符
os.eol                    // 常量：行尾
os.sleep(100)             // 休眠毫秒数（与 `time.sleep` 等价）
```

> **命名约定**：`os.*` 以 POSIX 函数名为主（`getenv` / `getcwd` / `getpid`）；不随 Node.js。Node 风格的 `process.env` 映射不提供，请用 `os.getenv(name)` / `os.environ()`。

详见 `stdlib/os/`。

### 16.6 异常运行时

内置 `Exception` 类是 prelude 类型（声明：`stdlib/types/exception.xr`），用户可直接 `new` 或继承：

```xray
@native
class Exception {
    message: string             // 人类可读消息
    stack: Array<string>        // 自动 capture 的调用栈，每帧一行格式化字符串
    cause: Exception?           // 链式 cause
    code: int                   // 错误码（从 "E0xxx: ..." 前缀自动解析，默认 0）
    data: Json?                 // throw 非异常值时原始值被包装在此

    constructor(message: string = "", cause: Exception? = null)
    fn toString() -> string
}
```

`throw` 表达式的操作数静态类型必须是 `Exception` 或其派生类（见 §8.1.1）；其它类型在编译期被拒绝（错误码 `E0370`）。VM 抛出的运行时错误也使用此 `Exception` 类型。

栈展开：VM `xvm_unwind_stack()` 按 try-table 查找 catch handler，逐帧释放局部、执行 finally / defer，到达 catch 后跳转。详见 §8。

### 16.7 Result 运行时

`Result<T, E>` 是 prelude 的 ADT enum（见 §8.2 / §5.6.2）。运行时表示：tag (1 字节) + payload。`Result.Ok(v)` 与 `Result.Err(e)` 是值对象，`match` 解构在 IR 层 lower 为 tag-test + payload-extract。

由于 `Result` 在异常零路径上无成本（不抛、不展开栈），使用 Result 的代码路径性能等同于 tagged-union。
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 16. Runtime Model

Xray values are represented by tagged runtime values and GC-managed heap objects. The runtime includes class descriptors, native body descriptors, arrays, maps, sets, tuples, strings, BigInts, Json objects, exceptions, tasks, coroutines, channels, and native handles.

Memory regions include:

- System/native heap.
- Global/shared heap.
- Per-coroutine heap.
- VM frames and stacks.
- Parser/compiler arenas.

Garbage collection is based on per-coroutine collection and shared-object handling. Safepoints include calls, backward branches, channel operations, `await`, `yield`, and explicit GC operations.

Coroutines are scheduled M:N over worker threads with cooperative safepoints and work stealing.
<!-- /xr-spec:en -->
