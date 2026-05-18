# 079 — Tuple First-Class 设计方案

**状态**：planned
**日期**：2026-05-16
**作者**：Cascade × @xuxinglei 联合讨论
**相关源码**：
- 类型系统：`src/runtime/value/xtype.h`、`src/runtime/value/xtype.c`
- 类型引用：`src/frontend/parser/xtype_ref.h`、`src/frontend/parser/xtype_ref.c`
- 解析器：`src/frontend/parser/xparse_type.c`、`src/frontend/parser/xparse_expr.c`
- 分析器:`src/frontend/analyzer/xanalyzer_builtins.c`、`src/frontend/analyzer/xanalyzer_native_types.c`
- stdlib 类型签名：`stdlib/types/map.xr`、`stdlib/types/json.xr`、`stdlib/types/channel.xr`
- runtime entries：`src/runtime/object/xmap.c::xr_map_entries`

---

## 1. 背景与动机

### 1.1 触发问题

`stdlib/types/map.xr:13` 与 `stdlib/types/json.xr:7` 写出的签名是：

```xray
class Map<K, V> {
    entries(): Array<[K, V]>
}
class Json {
    entries(): Array<[string, Json]>
}
```

`[K, V]` 在 xray 语法中**不存在**：

- 圆括号 `(T1, T2)` 是合法 tuple 类型注解（`xparse_type.c:226-238`）
- 方括号 `[N]T` 是合法 fixed-array 类型注解（`xparse_type.c:92-107`），N 必须为 literal int
- 方括号 `[T, U]` **既非 tuple 也非 fixed-array**，是 lightweight builtin parser 接受但 `parse_type_str` 不识别的"假签名"

而 runtime `xr_map_entries`（`src/runtime/object/xmap.c:574-592`）实际返回 `Array<Array<value>>`（嵌套数组）。即：**类型签名错、运行时与签名不一致**。

### 1.2 已尝试且否决的方案

| 方案 | 结果 | 原因 |
|------|------|------|
| `[K, V]→tuple` 解析分支 | 引入新 bug | 方括号在 xray 语法不合法；运行时是嵌套 array 不是 tuple，签名"撒谎" |
| 新增 `Pair<K, V>` builtin 类 | 否决 | 仅解决 entries 一处；3+ 元仍要 Triple/Quad；与 xray 已用 `(T, bool)` tuple type 形成机制分裂 |
| 改 `Array<Array<Json>>` 占位 | 否决 | 牺牲 K/V 类型精度，引入新技术债 |

### 1.3 关键观察 — xray 已经走在 tuple 路线上

```xray
// src/frontend/analyzer/xnative_type_defs.inc 中已有
class Channel<T> {
    tryRecv(): (T, bool)
}

// tests/regression/12_type_checking/1210_multi_return.xr
fn divide(a: int, b: int): (int, bool) {
    if (b == 0) { return 0, false }
    return a / b, true
}
```

xray **类型系统已经支持** `(T1, T2)` tuple type（用于多返回值类型注解），但 **没有 first-class 化**：
- 无 tuple 字面量（`xparse_expr.c:483` 显式禁止）
- 无 tuple 运行时对象（`XR_KIND_TUPLE` 仅是类型 tag）
- 无 tuple 字段访问（`.0`、`.1`）
- 多返回值通过 VM 多寄存器栈传递，是"特殊语法"

继续走 Pair 路线 = **xray 内部出现 tuple 与 Pair 双轨制**，最差选择。

### 1.4 决策

走 **tuple first-class** 路线，圆括号 `(a, b)` 字面量。Pair 类不引入。

---

## 2. 设计原则

1. **类型与值语法统一**：`(int, string)` 既是类型注解也是值字面量
2. **多返回值 = tuple 返回**：废除特殊语法，统一为 tuple
3. **解构语法统一**：`(x, y)` 模式在 var/let/param/match 共用
4. **零分配快速路径**：tuple 立即解构不分配；只有跨边界、被存储时 materialize
5. **结构性等价**：`(int, string)` 在任何位置等价
6. **完整性**：空 tuple `()`、一元 tuple `(a,)` 都合法可表达
7. **tuple 与 Array 严格分离**：不同运行时对象、不同语法、不互转

---

## 3. 语法规范

### 3.1 类型注解

```xray
// 二元及以上
let pair: (int, string) = (1, "hi")
let triple: (int, int, int) = (1, 2, 3)

// 嵌套
let nested: ((int, int), string) = ((1, 2), "name")

// 函数返回
fn divide(a: int, b: int): (int, bool) {
    if (b == 0) { return (0, false) }
    return (a / b, true)
}

// 容器内
let entries: Array<(string, int)> = m.entries()

// 一元（尾逗号必需）
let single: (int,) = (42,)

// 空（Unit type）— xray 完全移除 void，详见 4.5
fn log(msg: string): () { ... }
let nothing: () = ()
```

### 3.2 值字面量

```xray
let p = (1, "hi")           // 推断为 (int, string)
let t = (1,)                // 一元 tuple，推断为 (int,)
let u = ()                  // 空 tuple / unit value
let nested = ((1, 2), "a")  // 嵌套
let x = (1 + 2)             // 分组表达式，x: int = 3
let y = (1 + 2,)            // 一元 tuple，y: (int,)
```

### 3.3 字面量消歧表

| 形态 | 解析为 | 备注 |
|------|--------|------|
| `()` | unit value | 空 tuple |
| `(a)` | 分组表达式 | 求值结果 = `a` 的类型 |
| `(a,)` | 一元 tuple | 尾逗号强制 |
| `(a, b)` | 二元 tuple | |
| `(a, b,)` | 二元 tuple | 可选尾逗号 |
| `f(a, b)` | 函数调用 | 左侧是 callable |
| `(x, y) => expr` | lambda | `=>` 后置 token 决定 |

### 3.4 字段访问

```xray
let p = (1, "hi", true)
let a = p.0    // int = 1
let b = p.1    // string = "hi"
let c = p.2    // bool = true

// 编译期越界检查
let bad = p.3  // error: tuple (int, string, bool) has no field .3
```

**使用 `.N` 而非 `[N]`** —— 编译期已知字段（类似 struct field），与 array 动态索引区分。

### 3.5 解构（声明位置）

xray 控制流统一括号风格：`if (cond) {...}` / `for (i in 0..n) {...}` / `while (cond) {...}`。
for-in 内部，tuple pattern 直接进入解构变量位置。

```xray
// 变量声明
let (a, b) = pair
let (x, y, z) = triple
let ((a, b), c) = nested        // 嵌套
let (a, _, c) = triple          // 忽略中间值
let (first, ...rest) = (1, 2, 3, 4)  // rest pattern, rest: (int, int, int)

// 函数参数
fn dist((x1, y1): (int, int), (x2, y2): (int, int)): float { ... }

// for-in 解构（tuple pattern 直接放在 for 括号内迭代变量位置）
for ((k, v) in m.entries()) {
    println(k, v)
}

// 直接迭代 Map（Map iter 自然产生 (K, V) tuple）
for ((k, v) in m) {
    println(k, v)
}

// 整体绑定为 tuple 变量
for (entry in m.entries()) {
    println(entry.0, entry.1)
}

// 多返回值（强制使用 tuple 解构，无特殊语法）
let (q, ok) = divide(10, 3)
```

**xray 现状的扁平 `for (k, v in m)` 语法将被废除**，统一到 tuple pattern `for ((k, v) in m)`。详见 6.4。

### 3.6 解构（模式匹配，未来 `match`）

```xray
match (coord) {
    (0, 0) => "origin",
    (x, 0) => "on x-axis",
    (0, y) => "on y-axis",
    (x, y) if (x == y) => "diagonal",
    _ => "general",
}
```

同一套 `(x, y)` 语法在所有解构上下文复用。

### 3.7 多返回值彻底废除特殊语法

按"无后向兼容、最佳设计"原则，**完全删除**原有多返回值特殊语法。**不保留任何语法糖**。

```xray
// 旧（彻底删除，不再合法）
fn f(): (int, bool) {
    return a, b              // ✗ syntax error
}
let x, y = f()               // ✗ syntax error

// 新（唯一合法形式）
fn f(): (int, bool) {
    return (a, b)            // 返回 tuple 值
}
let (x, y) = f()             // tuple 解构
```

**为什么彻底删而不是保留糖**：
- 双轨制是技术债：两种写法同时合法 → 团队风格永远分裂
- Parser 复杂度：要 desugar `return a, b` → `return (a, b)`，错误恢复也要处理两种形态
- 心智模型清晰：xray 函数永远只返回单个值；需要返回多个值就打包成 tuple — 单一概念，无特例
- 教学层面仍可叫"多返回值"作为概念称呼，但源码层只有 tuple

AST `AstReturnMulti` / `AstLetMulti` 节点全部删除。详见 9 节模块变化。

---

## 4. 类型系统

### 4.1 Type kind

`XR_KIND_TUPLE` 已存在（`src/runtime/value/xtype.h`）。完整化定义：

```c
struct XrType {
    XrTypeKind kind;
    ...
    union {
        ...
        struct {
            XrType **element_types;
            int element_count;     // 0 = unit, 1 = unary, 2+ = normal
        } tuple;
    };
};
```

### 4.2 等价规则

**结构性等价**：

```
(T1, T2, ..., Tn) ≡ (U1, U2, ..., Um)
iff n == m and ∀i. Ti ≡ Ui
```

不需要类型声明，元素数量和类型完全匹配即等价。

### 4.3 子类型规则（协变）

```
(T1, T2, ..., Tn) <: (U1, U2, ..., Un)
iff n == m and ∀i. Ti <: Ui
```

### 4.4 类型推断

| 字面量 | 推断结果 |
|--------|----------|
| `(1, "a")` | `(int, string)` |
| `(1,)` | `(int,)` |
| `()` | `()` (Unit) |
| `(1, "a", true)` | `(int, string, bool)` |

字面量直接确定 tuple 形态，无需类型注解。这是圆括号路线相对方括号路线的核心优势。

### 4.5 Unit type 与 void 完全移除

**决定：xray 完全移除 `void` 关键字与 `XR_KIND_VOID` kind，由 0 元 tuple即 Unit type `()` 完全取代。**

```xray
// 唯一合法形式
fn a(): () { ... }    // 显式 Unit 返回
fn b() { ... }        // 默认推断为 () 返回

// Unit value
let x: () = ()
let y = print("hi")   // y: ()

// 不再合法
fn c(): void { ... }  // ✗ syntax error: void is removed, use ()
```

#### 为什么完全移除 void

1. **Unit 是 void 的严格超集**。void 能表达的语义（“函数无返回值”）Unit 完全覆盖，且 Unit 还能作为类型一等公民（变量类型、泛型参数、作为值传递）
2. **消除特例**。void 在类型系统中是伪类型（不能作为变量类型、不能作为泛型参数、不能出现在表达式中）；移除后所有类型都是一等公民
3. **泛型一致**。`Channel<()>` 作为信号 channel 自然成立，不需要为 `Channel<void>` 做特判
4. **与主流现代语言对齐**。Rust/Kotlin/Scala/Haskell/F# 都只有 Unit 没有 void；仅 C/Java/JS/Go/TS 这些早期或动态语言保留 void
5. **与 xray 现有路线一致**。`Channel<T>.tryRecv(): (T, bool)`、多返回值都已使用 tuple type，把 0 元 tuple 升为 Unit 是自然延伸
6. **comptime 元编程**（task 071）需要类型完整性。如果 void 不是类型，元编程遇 void 返回值时必须特判

#### Unit / null 语义不同

| 项 | Unit `()` | null |
|----|-----------|------|
| 含义 | “无信息但确实有一个值” | “缺失，没有值” |
| 类型 | 自己的类型 `()` | 通常是 `T?`（optional）的一种状态 |
| 数学上 | “1”（单元素集合） | “None”／缺失标记 |
| 用途 | 副作用函数返回、信号 channel | 表示某个 T 值可能不存在 |
| 类型安全 | 完全安全 | 著名的 "billion dollar mistake" |

```xray
// Unit：函数做了事但没有有意义的返回值
fn print(s: string): () { ... }

// null：函数可能返回值也可能没结果
fn find(arr: Array<int>, target: int): int? {
    if (found) { return target }
    return null
}
```

#### IR 内部影响

xray IR 底层广泛使用 `type_void` 作为 side-effect 指令（STORE/SET_GLOBAL/SET_SHARED/INDEX_SET/ASSERT）的占位返回类型。这些使用与用户语言层 void **逆向只是名字重复**。移除后：

- `XR_KIND_VOID` 删除，由 `XR_KIND_TUPLE` (`element_count == 0`) 代替
- `g_type_void` 全局单例 → `g_type_unit`
- `xi_lower.h::type_void` → `type_unit`（仅重命名，语义零变化）
- IR dump / TFA 调试输出："void" → "()"
- xi_arc / xi_emit_slotmap 中近似 null 的 tag 处理逻辑按原语义保留，仅检查改为 `kind == XR_KIND_TUPLE && element_count == 0`

### 4.6 一元 tuple

```xray
let t: (int,) = (42,)
let n: int = t.0     // 提取

// (int,) 和 int 不是同一类型
let bad: int = t     // error: type (int,) is not assignable to int
```

一元 tuple 在 generic 参数化、宏展开等场景有价值。日常少用但完整性必要。

### 4.7 Tuple 与 Array 严格分离

```xray
let t = (1, 2, 3)         // (int, int, int) tuple
let a = [1, 2, 3]         // Array<int>

// 不可隐式转换
let bad1: Array<int> = t           // error
let bad2: (int, int, int) = a      // error

// 显式转换 API
let arr_from_tuple = Array.from(t)
let tuple_from_arr: (int, int, int) = a.toTuple()  // 运行时校验长度
```

### 4.8 Optional vs tuple of optionals

```xray
let a: (int, string)?     // 整个 tuple 可空
let b: (int?, string?)    // 元素各自可空
// 两者不等价
```

---

## 5. 运行时表示

### 5.1 双模式设计

xray 多返回值已经是"VM 多寄存器栈传递"，是 zero-alloc tuple 的天然形态。Tuple first-class 保留这个优化，按使用场景选择形态：

| 场景 | 表示 | 分配 |
|------|------|------|
| 函数返回值 + 立即解构 | VM 寄存器/栈 | 零分配 |
| 立即解构表达式 `let (x, y) = (a, b)` | constant fold | 零分配 |
| 存到变量 `let t = (1, 2)` | heap `XrTuple` | 一次 |
| 放进 array/map/struct field | heap `XrTuple` | 一次 |
| 通过 Channel 传输 | heap `XrTuple` + deep copy | 一次 + 深拷贝 |
| 作为函数参数（非 spread） | heap `XrTuple` | 一次 |

判定时机：编译期决定，IR 优化阶段做 escape analysis lite。

### 5.2 Heap-allocated XrTuple

新建 `src/runtime/object/xtuple.{h,c}`：

```c
struct XrTuple {
    XrGCHeader header;       // 统一 GC 头
    uint16_t element_count;  // 编译期已知
    XrValue elements[];      // 柔性数组，连续存储
};

XR_FUNC XrTuple *xr_tuple_new(struct XrCoroutine *coro, uint16_t element_count);
XR_FUNC XrValue xr_tuple_get(XrTuple *t, uint16_t index);   // 编译期已校验下标
XR_FUNC bool xr_tuple_equals(XrTuple *a, XrTuple *b);
XR_FUNC uint64_t xr_tuple_hash(XrTuple *t);
```

特点：
- 单次分配（header + N 个 XrValue 连续）
- 不需要 hash table 或符号查找
- `.N` 索引访问编译期 lower 为 `tuple->elements[N]`，O(1)
- GC mark 直接扫描 elements 数组

### 5.3 寄存器模式（多返回值零分配路径）

xray VM 当前的多返回机制：

```
return (a, b, c)
↓
push a
push b
push c
return N (caller pops N values)
```

**保持不变**。仅在"返回值需要被存储为 tuple 对象"时 materialize：

```xray
let t = divide(10, 3)        // 需要 materialize -> heap XrTuple
let (q, ok) = divide(10, 3)  // 直接消费栈值 -> 零分配
```

### 5.4 Materialize 时机

IR 阶段插入 `MATERIALIZE_TUPLE` 指令：

```
// 立即解构：无 materialize
let (q, ok) = f()
↓
%a, %b = call f
let q = %a
let ok = %b

// 存储为变量：插入 materialize
let t = f()
↓
%a, %b = call f
%t = alloc_tuple 2
%t[0] = %a
%t[1] = %b
let t = %t
```

### 5.5 类型 ID 与 tag

新增 `XR_TTUPLE` 到 `XrObjType`：

```c
enum XrObjType {
    ...
    XR_TTUPLE,
    ...
};

#define XR_IS_TUPLE(v)   (XR_IS_PTR(v) && XR_GC_GET_TYPE(XR_TO_PTR(v)) == XR_TTUPLE)
#define XR_TO_TUPLE(v)   ((XrTuple *) XR_TO_PTR(v))
```

### 5.6 GC 集成

- Mark phase：扫描 `tuple->elements[i]` 引用
- Sweep phase：标准回收
- Write barrier：tuple element 写入按现有规则触发（tuple 一旦构造通常 immutable，写屏障路径冷）

---

## 6. 与现有特性整合

### 6.1 多返回值

完全统一：
- AST：`AstReturnMulti(values)` 删除，改为 `AstReturn(AstTupleExpr(values))`
- 类型检查：`(int, bool)` 单一类型
- VM：保留多寄存器实现作为 tuple 的"未 materialize"形态

### 6.2 Channel<T>

现状（已用 tuple type）：
```xray
class Channel<T> {
    tryRecv(): (T, bool)
}
```

零改动，自然能用：
```xray
let (value, ok) = ch.tryRecv()
if (!ok) { return }
```

### 6.3 Map.entries / Json.entries

签名改：

```xray
class Map<K, V> {
    entries(): Array<(K, V)>            // 替换 Array<[K, V]>
    forEach(fn: fn(key: K, value: V): ()): ()
}

class Json {
    entries(): Array<(string, Json)>
}
```

Runtime 改：`xr_map_entries` 创建 `XrTuple` 对象数组（替换嵌套 `XrArray`）：

```c
// 旧
XrArray *pair = xr_array_with_capacity(coro, 2);
xr_array_push(pair, n->key);
xr_array_push(pair, n->value);
xr_array_push(arr, xr_value_from_array(pair));

// 新
XrTuple *pair = xr_tuple_new(coro, 2);
pair->elements[0] = n->key;
pair->elements[1] = n->value;
xr_array_push(arr, xr_value_from_tuple(pair));
```

使用方：

```xray
for ((k, v) in m.entries()) {
    println(k + "=" + v.toString())
}

let pairs = m.entries()
let first_key = pairs[0].0   // tuple 字段访问
```

### 6.4 for-in 解构

xray for-in 的括号包裹与 tuple pattern 合并：

```xray
// 普通迭代（单变量）
for (item in arr) { ... }                   // arr: Array<T>, item: T

// tuple pattern 解构迭代
for ((k, v) in m.entries()) { ... }         // m.entries(): Array<(K, V)>
for ((k, v) in m) { ... }                   // Map 直接迭代产生 (K, V) tuple
for ((i, x) in arr.enumerate()) { ... }     // arr.enumerate(): Array<(int, T)>

// 整体绑定为 tuple
for (entry in m.entries()) {                // entry: (K, V)
    println(entry.0, entry.1)
}
```

**废除 xray 现有的扁平语法 `for (k, v in m)`**。原语法是“括号内同时出现多变量与 `in`”的特例，与 tuple pattern 不统一。废除后统一使用 `for ((k, v) in collection)` 是全局 tuple pattern 在迭代场景的自然延伸。

### 6.5 lambda / 闭包

```xray
let pair_maker = fn(x: int): (int, int) { return (x, x * 2) }

arr.map(fn((k, v): (string, int)): string { return k + "=" + v.toString() })
```

### 6.6 错误处理（未来 Result 类型）

Tuple first-class 让 Go 风格错误处理自然：

```xray
type Result<T, E> = (T?, E?)

fn parse(s: string): Result<int, string> {
    if (valid(s)) { return (toInt(s), null) }
    return (null, "invalid: " + s)
}

let (value, err) = parse("42")
if (err != null) { throw new Exception(err) }
```

### 6.7 跨协程边界

```xray
let ch: Channel<(int, string)> = Channel.new()
go fn() { ch.send((1, "hi")) }()
let (n, s) = ch.recv()
```

跨协程 tuple 必须 deep-copy（与现有协程数据共享规则一致）：
- tuple 本身可序列化
- 内部 elements 递归 deep-copy
- 若 element 含可变引用（Array/Map），按现有 channel 规则处理

### 6.8 JSON 序列化

```xray
let t = (1, "hi", true)
let json = Json.stringify(t)        // "[1, \"hi\", true]" — JSON array

let parsed: (int, string, bool) = Json.parse("[1, \"hi\", true]")
```

tuple ↔ JSON array 双向自动映射。

---

## 7. 边界条件

### 7.1 空 tuple `()` 与 Unit

**Unit type 是 xray 唯一的“无返回值”表达**。void 关键字、`XR_KIND_VOID` kind 均已完全移除。详见 4.5。

Unit 作为类型系统一等公民：可作为 generic 参数、可作为 channel 载荷、可赋值、可参与表达式。

### 7.2 一元 tuple `(a,)`

强制尾逗号，与 Python/Rust/Haskell 一致。

### 7.3 嵌套与平坦化

```xray
let nested: ((int, int), int) = ((1, 2), 3)
let flat: (int, int, int) = (1, 2, 3)
// 不可隐式互转
```

不引入自动 flatten。

### 7.4 命名 tuple 字段（不引入）

Swift 风格 `(x: int, y: int)` 不采纳。理由：
- 命名字段是 class 的本职工作，重复机制
- `coord.0/.1` 已足够
- 真要命名字段，定义 `class Coord { x: int; y: int }`

保持 tuple 纯粹位置型。

### 7.5 可变 tuple（不允许）

```xray
let t: (int, int) = (1, 2)
t.0 = 3       // error: tuple is immutable
```

可变需求请用 class。

### 7.6 自动 method

Tuple 自动获得：
- `.toString()` —— `(1, "hi", true).toString() == "(1, \"hi\", true)"`
- `==` 结构相等（按元素逐一比较）
- 哈希值（按元素哈希组合）—— 可作 Set/Map key

不引入 `.swap()` / `.map()` 等复合操作（归 stdlib 自由函数）。

### 7.7 Spread `...t`（不引入，未来扩展）

```xray
// 一阶段：不支持
f(...t)              // error
let m = (...a, ...b) // error

// 未来阶段
fn f(a: int, b: int) { ... }
f(...(1, 2))         // 自动展开
```

减少首版复杂度。

---

## 8. IR / VM 整合

### 8.1 新 IR 节点

```c
XI_TUPLE_NEW,         // 构造 tuple 值，N 个 element
XI_TUPLE_GET,         // 取 tuple.N 字段
XI_TUPLE_DESTRUCTURE, // 解构 tuple 为 N 个寄存器（编译期已知 N）
```

`XI_RETURN_MULTI` 保留（作为 tuple 的"未 materialize"形态）。

### 8.2 IR 优化：消除冗余 materialize

```
let (x, y) = (a, b)
↓ 朴素 lowering
%t = tuple_new 2
%t.0 = a
%t.1 = b
%x = %t.0
%y = %t.1
↓ peephole 优化
%x = a
%y = b
```

对 `TUPLE_NEW` 紧接 `TUPLE_DESTRUCTURE` 的模式直接 copy。

### 8.3 函数边界返回模式

```c
typedef enum {
    XR_CALL_RET_SINGLE,    // 单值返回
    XR_CALL_RET_TUPLE_REG, // tuple 通过多寄存器（未 materialize）
    XR_CALL_RET_TUPLE_BOX, // tuple 已 materialize 为 XrTuple
} XrCallReturnMode;
```

VM `call` 指令根据调用方上下文决定接收方式。

---

## 9. 模块/编译器变化总览

```
Parser
  ├─ xparse_type.c: () / (T,) / (T1, T2, ...) 类型注解放宽尾逗号、加 unit
  ├─ xparse_type.c: 删除 `void` 关键字识别，所有 void 错误提示“use () instead”
  ├─ xparse_expr.c: 解禁 (a, b) 表达式，处理 (a) vs (a,) 消歧
  ├─ xparse_stmt.c: 解构 let/for/fn-param 统一 (x, y) 模式、废除 `for (k, v in m)` 扁平语法
  ├─ xparse_stmt.c: 废除 `return a, b` / `let x, y = ...` 多返回特殊语法，提示 “use tuple instead”
  ├─ xparse_error.c: 为 3 类废除语法提供友好错误提示（见 9.1）
  └─ 新增: tuple.0 / .1 字段语法

Type system
  ├─ xtype.h / xtype.c: XR_KIND_TUPLE 完整启用
  ├─ 结构等价、协变规则
  ├─ 删除 XR_KIND_VOID kind、g_type_void 单例、XR_TYPE_IS_VOID 宏
  ├─ g_type_void → g_type_unit（0 元 tuple）
  └─ Type inference: 字面量直接推断

IR（void 移除影响）
  ├─ xi_lower.h::type_void → type_unit（仅重命名，语义零变化）
  ├─ xi_dump.c / xm_tfa.c：kind 名称输出从 "void" → "()"
  ├─ xi_arc.c 里 XR_KIND_VOID 分支跳转为检查 0 元 tuple
  └─ xi_emit_slotmap.c 里 XR_TAG_NULL 占位逻辑同步调整

Analyzer
  ├─ xanalyzer_*: tuple 字面量类型检查
  ├─ .N 字段访问编译期越界检查
  ├─ 解构模式 vs tuple type 匹配验证
  └─ 内置签名: Map.entries / Json.entries / Channel.tryRecv 跟随

IR
  ├─ xi_*: XI_TUPLE_NEW / XI_TUPLE_GET / XI_TUPLE_DESTRUCTURE
  ├─ 消除冗余 materialize 优化
  └─ 多返回值 lowering 统一为 tuple

Runtime
  ├─ src/runtime/object/xtuple.{h,c}: XrTuple 对象
  ├─ src/runtime/value/xtype.h: XR_TTUPLE type id
  ├─ src/runtime/closure: tuple 作为函数参数/返回的处理
  └─ src/runtime/gc: mark/sweep 集成

VM
  ├─ TUPLE_NEW / TUPLE_GET / TUPLE_DESTRUCTURE opcode
  ├─ 多返回值保留为"未 materialize tuple"路径
  └─ 调用约定: caller 选择接收方式

JIT / AOT
  ├─ tuple field access 内联展开
  ├─ const tuple 折叠
  └─ escape analysis（中长期）

Stdlib
  ├─ stdlib/types/map.xr: entries(): Array<(K, V)>
  ├─ stdlib/types/json.xr: entries(): Array<(string, Json)>
  ├─ stdlib/types/channel.xr: tryRecv(): (T, bool) (零改动)
  ├─ Array<T> 加 .enumerate(): Array<(int, T)>
  ├─ 全部 stdlib .xr 文件：`void` → `()` 批量替换
  └─ 删除 Pair / [K,V] 等占位写法

Tests / Demos
  ├─ tests/regression/**/*.xr：`: void` → `: ()` 批量替换、`return a, b` → `return (a, b)`
  ├─ demos/**/*.xr：同步迁移
  ├─ docs/**/*.md：文档示例同步
  ├─ 新增 regression/12_type_checking/: tuple 类型推断、解构、嵌套、unit
  ├─ 新增 regression/06_collections/: Map.entries 返回 tuple、for-in 解构
  ├─ 新增 compile_errors/type/: tuple 类型错误、长度不匹配、字段越界、void 使用报错
  └─ 新增 unit/analyzer/: tuple 解构边界
```

### 9.1 废除语法的友好错误提示

废除特殊语法虽然让心智模型统一，但来自 Go/Python/Lua/JS 的用户会习惯性写出旧形式。Parser 必须在**精确定位** + **一行 help 提示**两个维度做友好诊断，把学习曲线压到最低。

#### 9.1.1 多返回值废除

```
error[E0301]: multi-return special syntax was removed
 --> example.xr:5:14
  |
5 |     return a, b
  |              ^ help: wrap returns in a tuple: `return (a, b)`
  |
  = note: xray functions return a single value; use a tuple type `(T1, T2)` for multiple values
```

**触发条件**：parser 在 `return` 后看到 `expr , expr` 模式（顶层逗号）
**修复建议**：自动构造 `return (a, b)` 形式作为 quick-fix（LSP code action）

#### 9.1.2 多变量绑定废除

```
error[E0302]: comma-separated binding requires tuple pattern
 --> example.xr:7:9
  |
7 |     let q, ok = divide(10, 3)
  |         ^^^^^ help: use tuple destructuring: `let (q, ok) = divide(10, 3)`
  |
  = note: bindings must be a single name or a tuple pattern
```

**触发条件**：parser 在 `let` 后看到 `ident , ident` 模式
**修复建议**：自动加 `( )` 包裹的 quick-fix

#### 9.1.3 for-in 扁平语法废除

```
error[E0303]: for-in flat binding syntax was removed
 --> example.xr:9:10
  |
9 |     for (k, v in m) { ... }
  |          ^^^^ help: use tuple pattern: `for ((k, v) in m) { ... }`
  |
  = note: Map directly iterates as (K, V) tuples; tuple pattern is required for destructuring
```

**触发条件**：parser 在 `for (` 后看到 `ident , ident in` 模式（变量列表后面接 `in`）
**修复建议**：自动用 `( )` 包裹 binding 部分

#### 9.1.4 void 关键字废除

```
error[E0304]: `void` keyword was removed
 --> example.xr:3:18
  |
3 |     fn log(s: string): void { ... }
  |                        ^^^^ help: use Unit type instead: `()`
  |
  = note: xray uses 0-tuple `()` as Unit; void has been removed
```

**触发条件**：parser/lexer 遇到 `void` token
**修复建议**：自动替换 `void` → `()` 的 quick-fix

#### 9.1.5 实现要求

- 所有 4 类诊断必须有**专属 error code**（E0301-E0304），便于文档化与搜索
- 必须给出 **`help:` 行** 包含可直接复制粘贴的修复后代码
- 必须给出 **`note:` 行** 解释**为什么**这样设计（用户教育）
- LSP **code action** 必须实现，IDE 一键修复
- 错误恢复：报告诊断后**继续解析**（按 quick-fix 推断的形式构造 AST），让后续 stage 仍能给出类型错误，避免级联失败

---

## 10. 实施依赖图（按依赖排序）

> 不谈工作量，仅按依赖梳理顺序。每段都有可独立验证的产物。

### 段 0：void 移除 / Unit 升为一等公民（前置）

0.1 `XR_KIND_VOID` 删除，0 元 tuple（`XR_KIND_TUPLE` + `element_count == 0`）代替
0.2 `g_type_void` → `g_type_unit`，`XR_TYPE_IS_VOID` → `XR_TYPE_IS_UNIT`
0.3 IR 内部 `type_void` 重命名为 `type_unit`，全位置迭代
0.4 Parser 删除 `void` 关键字，所有 void 位置报 **E0304** 诊断（见 9.1.4）并提供 `void` → `()` 的 quick-fix
0.5 stdlib / tests / demos / docs 全位置批量替换 `void` → `()`
0.6 验收：现有 ctest 全部通过（语义零变化，仅名字改）、E0304 在 `tests/compile_errors/` 下有专属用例验证 help/note 输出

### 段 1：基础设施

1.1 `XrTuple` runtime 对象 + GC 集成
1.2 `XR_TTUPLE` type id + value cast macros
1.3 `XR_KIND_TUPLE` type system 完整化（`xtype.{h,c}`）
1.4 验收：单元测试构造/访问/GC tuple 对象

### 段 2：编译器前端

2.1 Parser: `(a, b)` 字面量、`(a,)` 一元、`()` unit、`tuple.N` 访问
2.2 Parser: 解构模式 `(x, y)` 在 let/for/fn-param
2.3 Type inference: 字面量直接推断为 tuple
2.4 Analyzer: 类型检查、字段越界
2.5 验收：xray 源码能写 `let p = (1, "hi"); let (a, b) = p` 并 type check 正确

### 段 3：IR / VM

3.1 IR opcodes: `TUPLE_NEW` / `TUPLE_GET` / `TUPLE_DESTRUCTURE`
3.2 VM bytecode 实现
3.3 优化: 即时解构消除 materialize
3.4 多返回值 lowering 统一为 tuple
3.5 验收：tuple 表达式运行正确，多返回值零分配路径保留

### 段 4：stdlib 整合

4.1 `Map.entries` / `Json.entries` 签名改 `Array<(K, V)>`
4.2 Runtime `xr_map_entries` / `xr_json_entries` 改用 `XrTuple`
4.3 删除之前的 `[K, V]` 占位语法
4.4 验收：`for ((k, v) in m.entries())` 完整链路通

### 段 5：多返回值彻底废除特殊语法

5.1 Parser: 删除 `return a, b` / `let x, y = f()` 识别路径，发现后报 **E0301 / E0302** 诊断（见 9.1.1 / 9.1.2）
5.2 Parser: 删除 `for (k, v in m)` 识别路径，发现后报 **E0303** 诊断（见 9.1.3）
5.3 LSP code action：为 E0301 / E0302 / E0303 提供一键修复
5.4 删除 AST `ReturnMulti` / `LetMulti` 节点定义
5.5 现有 .xr 代码（stdlib/tests/demos）批量迁移为 tuple 形式
5.6 验收：
  - 不再存在 `return a, b` / `let x, y = ...` / `for (k, v in m)` 写法
  - `tests/compile_errors/` 下有 E0301-E0303 的专属用例，验证错误输出格式、行列位置、help/note 内容
  - LSP code action 集成测试验证 quick-fix 输出的代码能重新编译通过

### 段 6：高级特性（可选 / 后续）

6.1 Spread: `f(...t)`、`(...a, ...b)`
6.2 Match 模式匹配中 tuple pattern
6.3 Escape analysis 优化
6.4 Generic specialization for tuples

---

## 11. 关键设计决策汇总

| 决策点 | 选择 | 备选 | 理由 |
|--------|------|------|------|
| 字面量符号 | `(a, b)` | `[a, b]` (TS) | tuple vs array 视觉分明；数学传统、新语言主流 |
| 类型注解 | `(T1, T2)` | `[T1, T2]` | 与值字面量统一；xray 现有签名已是圆括号 |
| 一元 tuple | `(a,)` 强制尾逗号 | 禁止一元 | 完整性优先；泛型/宏场景必要 |
| void 关键字 | 完全移除 | 作为 () 别名 / 保留 | Unit 是 void 严格超集；消除语言特例；Rust/Kotlin/Scala 路线 |
| 空 tuple | `()` 是 Unit type，唯一“无值”表达 | 单独 unit 类型 | 与 tuple 统一 |
| 字段访问 | `.N` | `[N]` | 编译期已知 vs 动态索引；与 array 区分 |
| 命名字段 | 不支持 | Swift 命名 tuple | 重复 class 机制 |
| 多返回值 | 彻底废除特殊语法 | 保留为语法糖 / 独立机制 | 避免双轨制技术债；心智模型单一 |
| tuple vs array | 严格分离 | 隐式互转 | 避免类型谎言；显式 API |
| Spread | 初版不支持 | 直接支持 | 减少首版复杂度 |
| Match pattern | 同一套 `(x, y)` 语法 | 单独 pattern | 统一性 |
| 运行时表示 | 双模式（reg + heap） | 仅 heap | 保留多返回零分配 |
| Materialize 时机 | 编译期决定 | 运行时判定 | 性能可预测 |
| 结构等价 | 同 N 同类型即等价 | 名义等价 | 与 Rust/Swift/Scala 一致 |
| 可变性 | 不可变 | 可变 | 简化语义；可变需求用 class |

---

## 12. 验证场景（设计自洽性 sanity check）

### 12.1 Map.entries 全链路

```xray
let m: Map<string, int> = #{"a": 1, "b": 2}

// 直接迭代解构
for ((k, v) in m.entries()) {
    println(k + "=" + v.toString())
}

// 数组操作
let entries = m.entries()           // Array<(string, int)>
let first = entries[0]               // (string, int)
let key = first.0                    // string
let val = first.1                    // int

// 排序
entries.sort(fn(a: (string, int), b: (string, int)): int {
    return a.1 - b.1
})

// 转换
let strs = entries.map(fn((k, v): (string, int)): string {
    return k + "=" + v.toString()
})
```

### 12.2 多返回值 + 错误处理

```xray
fn parse_int(s: string): (int, bool) {
    return (n, true)
}

// 即时解构（零分配）
let (n, ok) = parse_int("42")
if (!ok) { return }

// 完整 tuple 值
let result = parse_int("42")        // result: (int, bool)
if (result.1) { use(result.0) }
```

### 12.3 协程间通信

```xray
let ch: Channel<(string, int)> = Channel.new()

go fn() {
    for ((k, v) in m.entries()) {
        ch.send((k, v))      // 跨协程 deep copy
    }
    ch.close()
}()

for {
    let (val, ok) = ch.tryRecv()
    if (!ok) { break }
    let (k, v) = val
    println(k, v)
}
```

---

## 13. 风险与开放问题

### 13.1 Parser caveat

1. **`(a, b) => expr` lambda 与 tuple 区分**：靠 `=>` 后置 token 决定。Parser 需 lookahead。
2. **`(a)` 永远是 grouping**：一元 tuple **必须**写 `(a,)`。文档+错误提示需明确教育。
3. **跨语言习惯迁移成本**：来自 Go / Python / Lua / JS / 旧版 xray 的用户会习惯性写 `return a, b` / `let x, y = ...` / `for (k, v in m)` / `: void`。parser 必须为所有 4 类写法提供精确、友好的诊断与一键修复，**详细规范见 9.1 节**。这是把方案 A（完全无糖）的学习成本压到最低的关键保障。

### 13.2 开放问题（建议先不决定）

1. **Tuple 的 method**：是否给 tuple 加 `.swap()`、`.toArray()`、`.map()` 等？倾向：仅 `.toString()`/`.equals` 自动，复杂操作用 stdlib 自由函数。
2. **泛型推断与 tuple**：`zip(a, b)` 自动推断 tuple 元素类型 — 标准做法，无问题。
3. **Generic specialization**：能否 `class Container<T>` 用 tuple 作为 T？应该可以，跟其他类型一样。

### 13.3 与其他后续任务的关系

- **Pattern matching / `match`**：tuple pattern 是 match 的子集，本任务先做解构，match 留作后续单独任务
- **Spread `...t`**：可独立后续任务
- **Result/Option type**：tuple-based `(T?, E?)` 自然成立，但单独抽象类型可后续任务
- **Escape analysis**：tuple materialize 优化，长线优化任务

---

## 14. 与 `[N]T` 的关系

`[N]T` fixed-array 类型是另一个独立问题，已在主任务讨论中决定走"收紧到 struct field 上下文"方案。该方案与 tuple first-class **无依赖**，可并行执行。本文档只关注 tuple。

---

## 15. 落地优先级

按用户当前关注度排序：

1. **优先**：段 4 stdlib 整合的"语法层占位"先用回 `Array<[K, V]>` + TODO 注释（已在前置 commit `78a4c35` 实现），不阻塞 tuple 主线
2. **本任务**：段 1-5 完整实施，落地 tuple first-class
3. **后续任务**：段 6 高级特性单独立项

---

## 附录 A：与主流语言对比

| 语言 | 字面量 | 类型注解 | 字段访问 | 一元 | 空 | 多返回 |
|------|--------|----------|----------|------|----|----|
| Rust | `(a, b)` | `(T1, T2)` | `.0` `.1` | `(a,)` | `()` Unit | tuple |
| Swift | `(a, b)` | `(T1, T2)` | `.0` 或命名 | (单值) | `()` Void | tuple |
| Scala | `(a, b)` | `(T1, T2)` | `._1` `._2` | 不支持 | `Unit` | tuple |
| Python | `(a, b)` | `Tuple[T1,T2]` | `[0]` `[1]` | `(a,)` | `()` | tuple |
| Haskell | `(a, b)` | `(T1, T2)` | `fst`/`snd` | (单值) | `()` Unit | tuple |
| OCaml | `(a, b)` | `T1 * T2` | pattern only | (单值) | `unit` | tuple |
| TypeScript | `[a, b]` | `[T1, T2]` | `[0]` `[1]` | `[a]` | `[]` | tuple |
| **xray (本设计)** | `(a, b)` | `(T1, T2)` | `.0` `.1` | `(a,)` | `()` Unit | tuple |

xray 选择与 Rust/Swift/Scala/Python/Haskell 一致的圆括号路线。

---

## 附录 B：术语表

| 术语 | 定义 |
|------|------|
| Tuple | 异构有限有序值组，如 `(1, "a", true)` |
| Tuple type | 类型注解 `(T1, T2, ..., Tn)` |
| Unit type | 0 元 tuple `()`，等价 `void` |
| Unary tuple | 1 元 tuple `(a,)`，需尾逗号 |
| Materialize | 将"寄存器/栈上的 tuple 形态"转为 heap `XrTuple` 对象 |
| Destructure | 解构，将 tuple 拆为多个绑定，如 `let (a, b) = t` |
| Structural equality | 结构性等价，按元素逐一比较类型/值 |
| Escape analysis | 编译期分析 tuple 是否需要 materialize 的优化 |

---

## 附录 C：与已存废 Pair 方案的对比

| 项 | Pair 类方案 | Tuple first-class（本方案） |
|----|-------------|------------------------------|
| 解决场景 | 仅 Map.entries 类 API | 多返回值、解构、模式匹配、Map.entries、Channel、错误处理 |
| 与 xray 现有 `(T, bool)` 的关系 | 制造分裂 | 自然延续 |
| 性能 | 必然 heap 分配 | 多寄存器零分配 + 按需 materialize |
| N>2 元场景 | 需新增 Triple/Quad/Quint 类 | 一种语法搞定 |
| 模式匹配契合 | 通过 `.key`/`.value` 访问 | tuple pattern 自然 |
| 字段命名 | 固定 `.key`/`.value` | 位置 `.0`/`.1`（命名场景用 class） |
| 实施面 | 局部 patch | 基础设施升级 |
| 长期演进 | 静态，每场景再加类 | 共享一套机制，所有 tuple 场景受益 |

---

## 附录 D：重要设计决策的论证

本节记录焦点设计决策的多角度对比与论证过程。该部分与主体设计互补。

### D.1 为什么使用 `.N` 而非 `[N]`

核心区别：tuple 元素数是编译期固定的，每个槽类型不同；array 是运行期可变、元素同构。

**`.N` 路线优势**：

1. **编译期类型精度**：`p.0` 推断为 `T1`，`p.1` 推断为 `T2`——每个访问返回精确类型
2. **编译期越界检查**：`p.5` 在 N=3 时编译期报错，不需运行期检查
3. **与 struct field 语义对齐**：tuple 本质是“匿名 struct”，`obj.field` 与 `tuple.N` 同为编译期字段访问
4. **解析简单无歧**：`p.0` 是 member access，`a[0]` 是 indexing，两套独立语法无 disambiguation

**`[N]` 路线的核心漏洞**：
```xray
let i = some_int()
let x = t[i]   // i 是动态变量，类型推断必须降级为 union: int|string|bool
```
用户能写出 `t[i]` 这种动态索引，类型推断丢失 tuple 精度。

**主流静态类型语言选择**：
| 语言 | 字面量 | 类型注解 | 字段访问 | 一元 | 空 | 多返回 |
|------|--------|----------|----------|------|----|----|
| Rust | `(a, b)` | `(T1, T2)` | `.0` `.1` | `(a,)` | `()` Unit | tuple |
| Swift | `(a, b)` | `(T1, T2)` | `.0` 或命名 | (单值) | `()` Void | tuple |
| Scala | `(a, b)` | `(T1, T2)` | `._1` `._2` | 不支持 | `Unit` | tuple |
| Python | `(a, b)` | `Tuple[T1,T2]` | `[0]` `[1]` | `(a,)` | `()` | tuple |
| Haskell | `(a, b)` | `(T1, T2)` | `fst`/`snd` | (单值) | `()` Unit | tuple |
| OCaml | `(a, b)` | `T1 * T2` | pattern only | (单值) | `unit` | tuple |
| TypeScript | `[a, b]` | `[T1, T2]` | `[0]` `[1]` | `[a]` | `[]` | tuple |
| **xray (本设计)** | `(a, b)` | `(T1, T2)` | `.0` `.1` | `(a,)` | `()` Unit | tuple |

xray 选择与 Rust/Swift/Scala/Python/Haskell 一致的圆括号路线。

### D.2 为什么多返回值彻底废除特殊语法而非保留为语法糖

**保留糖的代价**：
- 双轨制技术债：`return a, b` 和 `return (a, b)` 都合法→ 团队风格永远分裂
- 学习成本：用户必须学两种写法
- Parser 调试多一重：`return a, b` vs `return a; b` 错误恢复要区分
- 错误信息不准确：error 时“单返回还是多返回”存歧义
- 文档负担：每个示例要选一种，自动制造“哪种更地道”争论

**废除后**：
- 源码层只有 tuple 一种合法形式
- 心智模型单一：xray 函数永远返回单个值，多个值就打包为 tuple
- 错误信息全局清晰：`expected (int, bool), got int`、`return a, b is no longer valid syntax, use return (a, b)`
- 概念称呼上仍可叫“多返回值”，作为用户认知 / 教学项

**Rust/Swift 的做法与 xray 一致**：语言层只有 tuple，“多返回值”只是概念上的描述。

### D.3 为什么 Unit type 等价 void 且完全移除 void

**void 本质**是 C/Java/JS 中的伪类型（不能作变量类型、不能作泛型参数、不能在表达式中出现）。其本质是语法层标记，不是类型系统一等公民。

**Unit type `()`** 是包含唯一一个值 `()` 的真正类型：
- 可作变量类型：`let x: () = ()`
- 可作泛型参数：`Channel<()>` 信号 channel
- 可表达式中出现：`if (cond) { print() } else { () }`

**Unit / null 语义不同**：两者是完全不同的概念。Unit 有唯一值（"无信息但有值"），null 是"值缺失"。Unit 是安全的，null 是著名的"billion dollar mistake"。在 xray 中，null 属于 optional `T?` 类型的一种状态。

**主流语言选择**：

| 语言 | Unit 表达 | 是否同时保留 void |
|------|----------|-------------------|
| Rust | `()` | ✗ |
| Kotlin | `Unit` | ✗ |
| Scala | `Unit` | ✗ |
| Haskell | `()` | ✗ |
| F# | `unit` | ✗ |
| Swift | `Void` (alias `()`) | ✗ |
| C / Java / JS / Go / TS | 仅有 void | — |

**所有现代静态类型语言都选择 Unit，没有保留 void**。xray 为新设计语言，选同样路线。

### D.4 一元 tuple `(a,)` 的价值

一元 tuple 在日常应用代码中几乎用不到，但在以下场景**必不可少**：

#### D.4.1 generic 一致性

```xray
fn box<T>(x: T): (T,) { return (x,) }

let a = box(42)        // (int,)
let b = box((1, 2))    // ((int, int),)  — 嵌套清晰，外层"box 包装"语义保留
```

没有一元 tuple，N=1 case 包装语义丢失（`box(42) → 42` 跟没包一样）。

#### D.4.2 模式匹配统一性

```xray
fn count_args<T>(args: T): string {
    match (args) {
        () => "0 args",
        (a,) => "1 arg",          // 必须这种语法
        (a, b) => "2 args",
        (a, b, c) => "3 args",
    }
}
```

没有一元 tuple，"1 arg" 这个 case 无法表达（`(a)` 是分组而非 pattern）。

#### D.4.3 comptime 元编程

xray 已规划 comptime（task 071）。元编程遇 N=1 case：

```xray
@comptime
fn make_tuple<...Ts>(...args: Ts): (Ts...,) {
    return (args...,)
}

make_tuple(1)        // 期望: (int,)
make_tuple(1, 2)     // 期望: (int, int)
make_tuple()         // 期望: ()
```

一元 tuple 不存在则元编程必须特判 N=1，破坏统一性。

#### D.4.4 函数签名变换

```xray
// uncurry: 把多参函数变成"接受 tuple 的单参函数"
fn uncurry<A, B>(f: fn(A) -> B): fn((A,)) -> B { ... }
```

N=1 case 必须能表达。

#### D.4.5 主流语言两派

| 支持派（完整性优先） | 禁止派（实用性优先） |
|--------------------------|-------------------------|
| Rust（`(a,)`）、Python（`(a,)`） | Scala（禁止）、Swift（`(a)≡a`）、OCaml（`(a)≡a`） |

**xray 走 comptime 路线**（task 071），需要完整性。尾逗号是 Python/Rust 验证 30 年的成熟方案，认知成本低。缺失的代价是永久的语言瑕疵。**xray 选支持派**。

### D.5 为什么选圆括号 `(a, b)` 而非方括号 `[a, b]`

**关键区别**：tuple 和 array 是不同概念，应该有不同语法。

**`(a, b)` 路线**：
- tuple 和 array 字面量视觉分明：`(1, 2, 3)` 是 tuple，`[1, 2, 3]` 是 array
- 与 xray 现有 `(K, V)` 类型注解、`Channel.tryRecv(): (T, bool)` 返回类型一致
- 数学传统：`(a, b)` 是 ordered pair / 坐标
- Rust/Swift/Scala/Python/Haskell/OCaml 主流选择
- 提供一元 tuple `(a,)` 表达能力

**`[a, b]` 路线（TS 妥协方案）**：
- 与 array 字面量同形，靠类型推断区分
- 核心漏洞：`let x = [1, 2]` 推断为 array 还是 tuple？必须显式注解 `[number, string]` 才能推为 tuple
- 一元 tuple `[a]` 与单元素 array 完全无法区分
- 仅 TypeScript 使用（为了跨 JS 实践的妥协）

**xray 是新设计语言**，没有"必须复用 JS Array"约束，选 `(a, b)` 是设计纯粹性优先于实现便利。详细对比见本任务讨论历史。

### D.6 为什么 tuple 与 array 严格分离

xray tuple 和 array 是**不同运行时对象**、**不同类型**、**不同语法形态**：

| 项 | Tuple `(...)` | Array `[...]` |
|----|---------------|---------------|
| 类型 | 异构、长度编译期已知 | 同构、运行时可变 |
| 运行时 | `XrTuple` 连续 element 存储 | `XrArray` 动态序列 |
| 访问 | `.N` 编译期字段 | `[i]` 运行期索引 |
| 可变性 | 不可变 | 可变 |
| 迭代语义 | tuple pattern 解构 | for-each 元素 |

**隐式转换会造成"类型谎言"**——语义间隔会被模糊，调试、错误信息、性能优化都会被抠破。**所以严格分离**——需要转换时提供显式 API（`Array.from(tuple)` / `array.toTuple()`）。

Rust/Swift/Scala 都是这么做。Python 不分离是动态语言设计，不适合 xray 静态类型推断。

```xray
// tuple → array
let arr = Array.from((1, 2, 3))  // [1, 2, 3]

// array → tuple（运行时校验长度匹配）
let tup: (int, int, int) = arr.toTuple()  // (1, 2, 3)
```
