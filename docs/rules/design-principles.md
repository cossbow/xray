# 设计原则

## 核心原则

- **不考虑向后兼容性**：全新语言，直接采用最佳设计
- **编译时检查优先**：尽可能把错误提前到编译期
- **编译通过 = 并发安全**：静态分析替代运行时锁
- **零隐式升级**：跨协程可见的值必须显式标注；编译器不偷偷改变分配位置

## 类型系统

- **Union 类型**：支持 `T | U | ...`，最多 6 个成员
  - 解析：`src/frontend/parser/xparse_type.c` (`TK_PIPE`)
  - 运行时：`XR_KIND_UNION`；扁平化 / 去重 / 规范排序 / `int | float → float` / `T | null → T?`
  - 超过 6 个成员时退化为 `unknown`（非 `any`）
- **可空类型**：`T?` 等价于 `T | null`，通过 `is_nullable` 标记实现
- **`any` 类型已移除**：不存在 `XR_KIND_ANY`；解析器对 `any` 标识符显式报错，提示改用具体类型或 `Json`
- **类型收窄**：`xr_type_filter` / `xr_type_exclude` / `xr_type_non_nullable`；`??` 对 `T?` 去 null 后联合右侧类型

---

## 并发共享模型（三条规则）

跨协程可见的值必须通过以下三种机制之一表达。**没有第四种**。

### 规则 1：`Channel` — 跨协程通信

```xray
const ch = Channel(10)        // Channel<T>，系统堆分配 + refcount
go producer(ch)               // Channel 作为参数传递（incref）
ch.send(value); ch.recv()     // 通信原语
```

- **唯一免写 `shared` 的例外**：`Channel` 对象本身就是跨协程通信设计
- 必须声明为 `const`（禁止 `let ch = Channel(...)` —— 避免重新赋值导致引用悬空）
- 允许通过参数传递给子协程，子协程 `incref` 共享同一个 Channel 实例

### 规则 2：`shared const` — 跨协程不可变共享

```xray
shared const CONFIG = load_config()
go fn() { use(CONFIG) }()     // 子协程读取全局堆上的同一份对象
```

- 分配在全局堆（带 refcount）
- 不可变 → 并发读安全，零拷贝
- 字面量也须显式写 `shared const`（编译器仍做常量内联优化；不暴露优化细节到语言规则）
- 不允许 `shared let` 作为闭包 upvalue（下文）

### 规则 3：函数参数 — 跨协程传值（深拷贝）

```xray
let data = compute()
go worker(data)               // data 被深拷贝到子协程堆
```

- 传值给 `go f(x)` 或 `ch.send(x)`：
  - 指针类型 → 深拷贝（`xr_deep_copy_to_coro`）
  - 值类型（int/float/bool/null）→ 直接拷贝
- 子协程获得独立副本，父子互不影响
- 无需任何标注

### `shared let` 的特殊地位

`shared let` 是规则 2 的可变变体，用于**生产者侧独占可变 + move 送出**：

```xray
shared let payload = build_large_object()
payload.finalize()               // 生产者还能改
ch.send(move payload)            // move 后 payload 不可再用
```

- 可变，但所有权独占
- 跨协程边界必须配合 `move`（`go task(move x)` / `ch.send(move x)`）
- **禁止**作为闭包 upvalue（会引入共享可变状态）

---

## 编译器行为总表

| 变量声明 | 本地使用 | `go` 闭包捕获 | `go f(x)` 传值 |
|---------|---------|--------------|---------------|
| `const x = 42`（字面量） | 内联字面量 | **编译错误**，要求 `shared const` 或传参 | 深拷贝（原语值免拷） |
| `const x = heap_obj` | 协程本地堆 | **编译错误**，要求 `shared const` 或传参 | 深拷贝 |
| `let x = ...` | 协程本地堆 | **编译错误**，要求传参 | 深拷贝 |
| `shared const x = ...` | 全局堆 | ✅ 允许（只读共享） | 深拷贝（很少这样用） |
| `shared let x = ...` | 全局堆 | **编译错误**（闭包捕获禁止） | 必须 `move x` |
| `const ch = Channel(n)` | 系统堆 | ✅ 允许 | ✅ 允许（incref） |

## `move` 语义

- 用途：将 `shared let` 的所有权从发送者转移到接收者
- 必须位置：`go f(move x)` / `ch.send(move x)` 的参数位
- 编译期检查（`src/frontend/analyzer/xanalyzer_visitor_expr.c`）：
  - 不能 `move` `const` 值
  - 不能 `move` `Channel`
  - 不能 `move` 值类型（无堆对象）
  - `move` 后再访问 → 编译错误 `"use of moved value"`

---

## 并发安全模型（证明）

```
Channel      → 通信原语（系统堆 + refcount）            → 运行时安全
shared const → 不可变，多协程零拷贝共享                  → 编译期安全
shared let   → 可变独占，Move 转移所有权                → 编译期安全
函数参数     → 深拷贝，协程间零共享                      → 隔离即安全
普通 let/const → 协程本地，不允许跨协程可见              → 编译期拒绝

结论：编译通过 ⇒ 无共享可变状态 ⇒ 无数据竞争（无需运行时锁）
```

## 设计权衡说明

- **为什么不保留 auto-escape？** 自动升级让用户看不到性能成本（全局堆 + atomic refcount 比 coro-local 慢 5-10 倍），且不符合"零意外"原则
- **为什么 Channel 免 `shared`？** Channel 本身就是为跨协程通信设计的；加 `shared` 只是噪声
- **为什么字面量也要 `shared const`？** 规则一致性优先于语法糖。编译器仍会做常量内联优化
- **为什么禁止 `shared let` 被闭包捕获？** 闭包捕获 = 共享可变 = 数据竞争。`shared let` 的唯一安全用法是"move 送出"
