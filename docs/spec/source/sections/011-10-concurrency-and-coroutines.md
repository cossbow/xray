---
id: spec.10_concurrency_and_coroutines
order: 011
---

<!-- xr-spec:cn -->
---

## 10. 并发与协程 (Concurrency)

> 真值源：`src/runtime/coro/xcoro_*.c`、`src/runtime/sync/xchannel.c`、`src/runtime/sync/xscope.c`、`docs/rules/design-principles.md`。

xray 的并发是**协程 (goroutine 风格) + Channel + 强静态约束**。设计目标：写 `go { ... }` 就和写普通函数一样简单，但**编译期保证不发生数据竞争**。

### 10.1 协程模型

| 维度 | 选择 |
|--|--|
| 调度模型 | M:N（用户态协程 + 多 OS 线程） |
| 调度策略 | 协作式（GC-safepoint）+ work-stealing |
| 栈模型 | Segmented stack（按需扩展） |
| 创建开销 | ~微秒级，KB 级初始栈 |
| 上下文切换 | 用户态切栈，无 syscall |

协程默认分布在多个 worker 线程上；运行时根据 CPU 核数自动设置 GOMAXPROCS 风格的并行度。

### 10.2 `go` — 启动协程

```ebnf
GoExpr ::= 'go' (Block | CallExpr | LambdaExpr CallArgs?)
```

`go` 是**表达式**，返回 `Task<T>` 句柄。三种形式都合法：

```xray
// 形式 1：调用一个已声明的函数
let t1 = go worker(0, channel)

// 形式 2：调用一个 lambda 字面量（用于内联逻辑+捕获参数）
let t2 = go fn(d: Json) -> int {
    return d.value * 2
}(payload)

// 形式 3：块形式（隐式包装为零参 lambda）
let t3 = go {
    return compute()
}
```

**move 在参数位置**：跨协程转移所有权通过参数前缀 `move` 实现，**不是** `go` 的选项：

```xray
shared let data = { value: 10 }
let task = go fn(d: Json) -> int {
    return d.value + 1
}(move data)        // 把 data 的所有权移交给协程；之后 data 不可访问
```

**语义**：
- 每个 `go` 表达式都返回一个 `Task<T>`，其中 `T` 是被调函数的返回类型；返回 `()` 的函数对应 `Task<null>`。
- 协程在闲置 worker 线程中调度（M:N）。
- 协程内**未捕获**异常存在 `Task` 中，由 `await` 时重抛。
- 普通局部变量（非 `shared`、非 `move`）传给 `go` 时**自动深拷贝**；`shared const` 零拷贝共享；`shared let` 必须 `move`。

### 10.3 `await` — 等待结果

```ebnf
AwaitExpr ::= 'await' Expression
           |  'await' 'all' Expression       // 等待全部完成
           |  'await' 'any' Expression       // 等待任一完成
```

```xray
// 单 task
let task = go fetch("https://example.com")
let result = await task                    // 让出当前协程直到 task 完成

// await all：等待全部完成，返回结果数组（与输入顺序一致）
let t1 = go compute(2)
let t2 = go compute(3)
let t3 = go compute(4)
let results: Array<int> = await all [t1, t2, t3]
// 也可直接对变量使用，无需中括号
let tasks = [t1, t2, t3]
let results2: Array<int> = await all tasks

// await any：等待任一完成，返回该任务结果；其他任务继续运行
let first = await any [t1, t2, t3]

// await anySuccess：跳过失败任务，等待第一个成功的任务
let firstOk = await anySuccess [t1, t2, t3]
```

**语义**：

- `await` 仅作用于 `Task<T>` 类型；其他类型为编译错误。
- 当前协程**让出**直到目标完成（不阻塞 OS 线程）。
- 异常传播：
  - `await t` 重抛 t 抛出的异常。
  - `await all` 中任一任务抛异常即整体抛异常（其余任务会被取消）。
  - `await any` 仅当**全部失败**时抛异常；只要有一个完成，返回该任务结果。
  - `await anySuccess` 类似 `await any`，但**跳过**抛异常的任务，只等成功完成的。
- `all` / `any` / `anySuccess` 在 `await` 后面是**上下文关键字**，仅在此位置生效。

### 10.4 `Task<T>` 句柄

`go expr` 返回 `Task<T>`，其中 `T` 为被调函数的返回类型。Task 句柄支持：

| 方法 / 属性 | 类型 | 说明 |
|--|--|--|
| `t.done` | `bool`（属性） | 任务是否已完成（成功、失败或取消） |
| `t.cancelled` | `bool`（属性） | 任务是否被取消过 |
| `t.result` | `Json`（属性） | 任务返回值；未完成或失败时为 `null` |
| `t.error` | `string?`（属性） | 任务的异常消息；未失败时为 `null` |
| `t.cancel()` | `() -> ()` | 请求取消任务（合作式） |

```xray
let t = go fetch(url)
if (!t.done) { /* 还在跑 */ }
let r = await t

// 直接读取属性（无需 await）
print(t.done, t.cancelled, t.result, t.error)
```

**取消语义**：`cancel()` 设置取消标志；协程在下一个 safepoint（GC 检查点、Channel 操作、`await`、`yield`）检测到标志后抛出取消异常。`await` 已取消的 task 返回 `null`，`t.cancelled` 变为 `true`。

### 10.5 Channel

```ebnf
ChannelType ::= 'Channel' '<' Type '>'
ChannelNew  ::= 'new' 'Channel' ('<' Type '>')? '(' Expression ')'
```

Channel 通常以 `shared const` 声明（生命周期跨协程，引用语义）：

```xray
shared const ch  = new Channel<int>(10)    // 有缓冲，capacity = 10
shared const ch0 = new Channel<int>(0)     // 无缓冲（同步握手）
shared const cha = new Channel(3)          // 元素类型从首次 send 推断
```

**API**（注意全部为 **camelCase**）：

| 方法 | 签名 | 行为 |
|--|--|--|
| `send(v)` | `(T) -> ()` | 阻塞发送；满则等待消费者；channel 已关闭时抛异常 |
| `recv()` | `() -> T?` | 阻塞接收；空则等待生产者；channel 已关闭且缓冲为空时返回 `null` |
| `trySend(v)` | `(T) -> bool` | 非阻塞：成功返回 `true`，满或已关闭返回 `false` |
| `tryRecv()` | `() -> (T, bool)` | 非阻塞，返回**多值**：`(value, ok)`；空或已关闭时 `ok=false` |
| `sendTimeout(v, ms)` | `(T, int) -> bool` | 带超时发送；超时返回 `false` |
| `recvTimeout(ms)` | `(int) -> (T, bool)` | 带超时接收；超时返回 `ok=false` |
| `close()` | `() -> ()` | 关闭 channel；幂等 |
| `isClosed` | `bool`（属性） | channel 是否已关闭 |

```xray
shared const ch = new Channel<int>(2)
ch.send(10)
let v = ch.recv()                       // 10

let ok = ch.trySend(20)                 // true / false
let (val, ok2) = ch.tryRecv()           // 元组解构：值 + 成功标志
if (ok2) { print(val) }

ch.close()
```

**send/recv 与 `move`**：发送大对象时用 `ch.send(move payload)` 转移所有权，避免拷贝；接收方独占。

**语义**：
- **MPMC**（多生产者多消费者）。
- 有缓冲 ch：满则发送方挂起，空则接收方挂起。
- 无缓冲 ch：发送/接收必须同时握手（rendezvous）。
- 关闭后：`send` 抛异常；`recv` 返回剩余值，剩余取完后返回 `null`；`tryRecv` 返回 `(zero, false)`。

### 10.6 `select`

`select` 在多个 channel 操作中多路复用；非阻塞分支用 `_` 占位。

```ebnf
SelectStmt ::= 'select' '{' SelectArm+ '}'
SelectArm  ::= RecvArm | SendArm | TimeoutArm | DefaultArm
RecvArm    ::= Identifier 'from' Expression '->' Block
SendArm    ::= Expression 'to' Expression '->' Block
TimeoutArm ::= 'after' Expression '->' Block
DefaultArm ::= '_' '->' Block
```

```xray
shared const ch1 = new Channel<int>(2)
shared const ch2 = new Channel<int>(2)

select {
    msg from ch1 -> { print("got from ch1:", msg) }      // 接收分支
    msg from ch2 -> { print("got from ch2:", msg) }      // 接收分支
    100  to   ch1 -> { print("sent 100 to ch1") }        // 发送分支
    _ -> { print("no channel ready") }                   // 默认分支（非阻塞）
}
```

**语义**：
- 接收分支 `name from ch -> body`：等价于 `name = ch.recv()`，但仅在 ch 有数据时被选中。
- 发送分支 `value to ch -> body`：等价于 `ch.send(value)`，但仅在 ch 有空间时被选中。
- 默认分支 `_ -> body`：当前无任何分支就绪时立即执行；**省略默认分支**会让 select 阻塞直到某个分支就绪。
- 多个分支同时就绪时**随机**选择一个（与 Go 一致）。

### 10.7 `scope` 块（结构化并发 / 词法作用域）

`scope` 是**语句关键字**，建立一个新的词法作用域块。它服务两个目的：

1. **纯词法作用域**：与 C/Rust `{ ... }` 局部块一致，块内 `let` 不影响外层同名变量。
2. **结构化并发**（语义增强）：在 `scope` 块内 `go` 启动的协程，块退出前**自动等待**全部完成或取消。

```ebnf
ScopeStmt          ::= 'scope' Block
LinkedScopeStmt    ::= 'linked' 'scope' Block          // 兄弟失败 → 取消所有 + 重抛
SupervisorScopeExpr ::= 'supervisor' 'scope' Block     // 收集所有错误，返回 Array<string>
```

```xray
// 词法作用域用途
let x = 1
scope {
    let x = 10            // shadow 外层 x，块内有效
    print(x)              // 10
}
print(x)                  // 1

// 结构化并发用途（与 go 配合）
scope {
    go worker_a()
    go worker_b()
    // scope 块退出前，等待 a/b 全部完成；任一抛异常不影响兄弟
}
```

**三种 scope 变体**：

| 形式 | 子协程抛异常时的行为 | 返回值 |
|---|---|---|
| `scope { ... }` | 不取消兄弟；异常不向外传播（每个 task 独立） | 无（语句形式） |
| `linked scope { ... }` | **取消所有兄弟**协程，并向外**重抛**最先抛出的异常 | 无 |
| `supervisor scope { ... }` | **收集**所有失败子协程的异常消息，子协程之间互不影响 | `Array<string>`（错误列表；可为空表示全部成功） |

```xray
// linked scope：失败传播
try {
    linked scope {
        go ok_worker()
        go failing_worker()         // 抛异常
    }
} catch (e) {
    print("caught:", e)              // 命中此分支
}

// supervisor scope：收集错误
let errors = supervisor scope {
    go failing("error1")
    go failing("error2")
    go ok()
}
print(errors.length)                 // 2（只统计失败的）
```

**通用语义**：
- `scope` 不是函数调用，也不需要 import；是关键字块语句。
- 三种形式都在块退出前等待所有 `go` 启动的子协程完成。

### 10.8 `move` — 跨协程所有权转移

```ebnf
MoveExpr ::= 'move' Identifier        // 仅出现在调用参数位置
```

`move` 是**实参修饰前缀**（不是 `go` 的选项）。它把 `shared let` 变量的所有权从当前作用域转移到被调函数（包括 `go` 启动的协程、`ch.send()` 等）。move 后原变量在编译期被标记为**已 moved**，再次引用是编译错误。

```xray
shared let buf = new Bytes(1024 * 1024)

// 移交给协程
let t = go fn(b: Bytes) -> int {
    return process(b)
}(move buf)
// 编译错误：buf has been moved
// print(buf.length)

// 移交给 channel
shared const ch = new Channel<Bytes>(1)
shared let payload = new Bytes(4096)
ch.send(move payload)
// 编译错误：payload has been moved
```

详见 §7.3、§7.4 关于 shared 变量的捕获规则。

### 10.9 同步原语

xray 的默认并发模型偏向**消息传递 + 不可变共享**——通过 `shared const`、`Channel`、`move`、`scope` 已能在编译期消除大部分数据竞争，因此**不**鼓励使用裸 Mutex/锁。

如确需互斥锁/原子操作，运行时层面提供：

| 原语 | 形态 | 说明 |
|---|---|---|
| Channel(1) | 单元素 channel | 互斥的最佳实践（通过 send/recv 模拟 lock/unlock） |
| `shared let` + `move` | 编译期独占 | 跨协程独占，无运行时开销 |

> **设计说明**：xray 不在标准库中暴露 `Mutex`/`RwLock`/`Atomic*` 等通用并发原语；如未来引入，将作为 unstable 模块单独开放（参见 `docs/known_bugs.md` 与未来设计 RFC）。


### 10.10 `yield` — 让出 CPU

```ebnf
YieldStmt ::= 'yield'
```

```xray
for (i in 0..1000) {
    do_chunk(i)
    yield                       // 主动 safepoint，让其他协程有机会跑
}
```

**当前实现**：作为语句使用，等价 Go 的 `runtime.Gosched()`；不支持带值 `yield`。

### 10.11 并发安全模型

xray 通过类型系统**编译期消除大部分数据竞争**：

| 规则 | 强制 |
|--|--|
| `go` 闭包不能捕获普通 `let` 局部变量 | ✅ |
| `shared const` 跨协程零拷贝只读 | ✅ |
| `shared let` 必须 `move` 才能跨协程 | ✅ |
| Channel 跨协程传值 | ✅ |
| 共享可变状态必须显式 Mutex | 文档约定 |

**仍可能存在数据竞争**（运行时检测，非编译期）：
- 在 Channel 中发送可变 class 引用（接收方可能与发送方同时修改）— 建议总是发送 `shared const` / `Bytes` / 不可变对象 / `move` 移交。
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 10. Concurrency and Coroutines

### 10.1 `go`

```xray
let task = go compute()
```

`go` starts a coroutine and returns a `Task<T>` handle.

### 10.2 `await`

```xray
let r = await task
let results = await all [t1, t2, t3]
let first = await any [t1, t2, t3]
let ok = await anySuccess [t1, t2, t3]
```

`await` propagates exceptions from the awaited task. `await all` fails when any task fails. `await anySuccess` skips failing tasks until one succeeds or all fail.

### 10.3 `Task<T>`

Task members:

| Member | Meaning |
|--|--|
| `done` | task completed |
| `cancelled` | task was cancelled |
| `result` | completed result or null |
| `error` | error message/value or null |
| `cancel()` | request cooperative cancellation |

### 10.4 `Channel<T>`

```xray
shared const ch = new Channel<int>(2)
ch.send(10)
let v = ch.recv()
let ok = ch.trySend(20)
let (value, got) = ch.tryRecv()
```

Channel methods:

| Method | Meaning |
|--|--|
| `send(v)` | blocking send; throws on closed channel |
| `recv()` | blocking receive; returns buffered values, then null when closed and empty |
| `trySend(v)` | non-blocking send |
| `tryRecv()` | non-blocking receive returning `(value, ok)` |
| `sendTimeout(v, ms)` | send with timeout |
| `recvTimeout(ms)` | receive with timeout |
| `close()` | close the channel |
| `isClosed` / `isClosed()` | closed state |

### 10.5 `select`

`select` multiplexes multiple channel operations. The non-blocking default branch uses `_`.

```ebnf
SelectStmt ::= 'select' '{' SelectArm+ '}'
SelectArm  ::= RecvArm | SendArm | TimeoutArm | DefaultArm
RecvArm    ::= Identifier 'from' Expression '->' Block
SendArm    ::= Expression 'to' Expression '->' Block
TimeoutArm ::= 'after' Expression '->' Block
DefaultArm ::= '_' '->' Block
```

```xray
shared const ch1 = new Channel<int>(2)
shared const ch2 = new Channel<int>(2)

select {
    msg from ch1 -> { print("got from ch1:", msg) }
    msg from ch2 -> { print("got from ch2:", msg) }
    100 to ch1 -> { print("sent 100 to ch1") }
    after 1000 -> { print("timeout") }
    _ -> { print("no channel ready") }
}
```

Semantics:

- Receive arm `name from ch -> body`: equivalent to `name = ch.recv()`, but selected only when `ch` has data.
- Send arm `value to ch -> body`: equivalent to `ch.send(value)`, but selected only when `ch` has capacity.
- Timeout arm `after ms -> body`: selected when no channel arm becomes ready before the timeout.
- Default arm `_ -> body`: runs immediately when no arm is ready; omitting it makes `select` block until an arm becomes ready.
- When multiple arms are ready at the same time, one is selected randomly, matching Go.

### 10.6 `scope`

```xray
scope { ... }
linked scope { ... }
supervisor scope { ... }
```

A scope controls the lifetime of child coroutines. Linked and supervisor scopes provide different cancellation/failure propagation policies.

### 10.7 Safety Model

The language design makes cross-coroutine sharing explicit. Ordinary locals are isolated. Shared immutable data is marked `shared const`. Mutable sharing is explicit and constrained. Channels are the preferred way to communicate mutable values.
<!-- /xr-spec:en -->
