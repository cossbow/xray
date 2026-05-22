---
id: spec.7_scoping_and_name_resolution
order: 008
---

<!-- xr-spec:cn -->
---

## 7. 作用域与名字解析 (Scoping)

> 真值源：`src/frontend/analyzer/xanalyzer_scope.c`、`src/frontend/analyzer/xanalyzer_capture.c`。

### 7.1 词法作用域与提升

Xray 采用**词法作用域**：名字的可见性由源代码结构决定。

**作用域类型**：

| 作用域 | 触发 | 示例 |
|--|--|--|
| 模块 | 每个 `.xr` 文件 | 顶层 `let` `fn` `class` |
| 函数 / 闭包 | `fn` / 箭头函数进入 | 参数 + 函数体 |
| 块 | `{...}` | `if` `while` `for` `match` 分支体 |
| `scope` 块 | `scope { ... }` 关键字 | 显式词法作用域 + 结构化并发（见 §10.7） |
| `for` 头 | `for (let i=0; ...)` | `i` 仅循环体可见 |
| `catch` 参数 | `catch (e)` | `e` 仅 catch 体可见 |
| 类体 | `class` 定义 | 字段、方法 |

**提升规则**：

- 顶层 `fn` `class` `struct` `interface` `enum` `type` **提升**至当前作用域顶部——可在定义前引用。
- `let` / `const` **不提升**——必须在定义后使用。
- 同名重复声明：同作用域内 2 个同名变量 → 编译错误（嵌套作用域可 shadow）。

```xray
main()                    // OK：使用提升后的 fn
fn main() { ... }

let y = x                 // 错误：x 未声明
let x = 10
```

#### Shadow 规则

嵌套块可 shadow 外层同名变量：

```xray
let x = 1
{
    let x = "hello"           // shadow：OK
    print(x)                 // "hello"
}
print(x)                     // 1
```

### 7.2 闭包捕获语义

闭包捕获外层作用域的变量为 **upvalue**。

#### 普通同步闭包

默认按 **引用捕获**：

```xray
fn make_counter() -> (() -> int) {
    let count = 0
    return fn() -> int {
        count += 1                  // 修改外层 count
        return count
    }
}

let c = make_counter()
print(c())      // 1
print(c())      // 2
```

- 闭包与原变量**共享**。
- 外层作用域退出后，被闭包引用的变量会被 GC 保活（提升到堆）。

#### 闭包优化

编译器会分析 upvalue：
- 仅读 → 可能隐式复制（避免闭包转换）。
- 读写 → 提升为闭包 box。
- 详见 §17.5。

### 7.3 所有权与 move

Xray **不**是全面 ownership/borrow checker 语言（不像 Rust）。但在**跨协程数据传递**中使用 move 语义：

```xray
shared let big_buffer = new Bytes(1024 * 1024)

let t = go fn(b: Bytes) -> int {
    return process(b)
}(big_buffer)             // 编译错误：shared let 不能直接传递，必须 move

let t2 = go fn(b: Bytes) -> int {
    return process(b)
}(move big_buffer)        // OK：所有权转移

print(big_buffer.length)  // 编译错误：move 后访问
```

**move 使用场景**：`move` 作为**实参前缀**出现在调用位置（参见 §10.8）：

- `go f(move x)`、`go fn(...){...}(move x)`：把所有权转给协程。
- `ch.send(move data)`：跨协程发送时转移所有权（避免拷贝）。
- 普通函数调用 `f(move x)`：把所有权传入函数（被调函数独占）。

### 7.4 协程数据传递规则（避免数据竞争）

"保证编译期消除数据竞争"是 xray 并发模型的核心设计原则。

`go` 启动的协程**不能直接捕获**外层作用域的可变变量；数据必须通过**参数传递**进入协程。普通变量自动深拷贝；shared 变量按下表区分：

| 变量种类 | 跨协程传递规则 |
|---|---|
| 普通 `let` / `const`（局部） | 作为实参传递时**自动深拷贝**；不能被闭包捕获修改 |
| 函数参数 | ✅ 完全自由（已经是拷贝 / move 进来的） |
| `shared const` | ✅ 跨协程零拷贝只读共享（可被闭包捕获） |
| `shared let` | ⚠️ 必须用 `move` 实参前缀转移所有权；move 后原变量在编译期不可访问 |
| `Channel<T>` | ✅ 可被闭包捕获（生命周期由 channel 自身管理） |
| `this` / 闭包 upvalue（可变） | ❌ 不能跨协程；必须通过参数显式传递 |
| 全局 `import` 的函数/类 | ✅ 不可变定义，可自由引用 |

```xray
let local = 0
go { local += 1 }                        // ❌ 编译错误：不能捕获可变局部变量
```

#### 正确姿势

```xray
// 方法 1：作为参数传值（普通变量自动深拷贝）
let arr = [1, 2, 3]
let t = go fn(data: Array<int>) -> int {
    data.push(4)            // 拷贝上修改，不影响原值
    return data.length
}(arr)
print(arr)                  // [1, 2, 3] 未变

// 方法 2：shared const 零拷贝只读（可被捕获）
shared const config = { rate: 100 }
let t2 = go fn(c: Json) -> int {
    return c.rate
}(config)

// 方法 3：move 转移所有权
shared let big = new Bytes(1024)
let t3 = go fn(b: Bytes) -> int {
    return process(b)
}(move big)
// big 在此处不可访问

// 方法 4：Channel 通信（可被捕获）
shared const ch = new Channel<int>(10)
let t4 = go fn(c: Channel<int>) -> int {
    return c.recv()
}(ch)
ch.send(42)
```

### 7.5 GC 与对象生命周期

Xray 采用多层内存管理：

| 存储 | 机制 | 释放时机 |
|--|--|--|
| 全局堆（`shared const`） | refcount | refcount 变 0 |
| 局部堆（一般对象） | Mark-Sweep GC | 不可达时 |
| 栈（`struct` 值、本地） | RAII | 作用域退出 |
| Arena（底层临时分配） | 批量释放 | arena 结束 |

**GC 观察点**：
- 默认 incremental Mark-Sweep。
- Mark 阶段从根集（全局、栈、寄存器）遍历。
- Sweep 阶段释放未标记对象。
- GC 需要 GC-safepoint；指令列表中成为 safepoint 的点包括函数调用、后向跳转、显式 `gc.collect()`。

**写屏障**与**代际 GC** 设计：见 `docs/rules/gc-memory.md`。
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 7. Scoping and Name Resolution

Each block creates a lexical scope. Declarations are resolved through lexical parent scopes and then module/prelude scopes as appropriate.

Functions may capture outer variables. Coroutines started with `go` use stricter capture rules to avoid accidental data races:

- `shared const` may be captured directly.
- Ordinary local variables cannot be captured directly by `go` closures.
- Mutable shared state must be explicit and often transferred or synchronized.
- Channels are the preferred communication mechanism.

Modules are private by default. Only exported declarations are visible to importers.

Circular module dependencies are rejected.
<!-- /xr-spec:en -->
