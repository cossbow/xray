# Comptime Feature Implementation Plan

> 日期：2026-05-05
> 状态：设计方案
> 范围：语言语法、parser、analyzer、typed AST canonicalizer、consteval、泛型单态化、Xi pipeline、VM/JIT/AOT 集成、formatter/LSP、测试

---

## 0. 目标

Xray 已经具备 typed scripting、泛型、class/struct/interface、协程、Xi typed SSA、VM/JIT/AOT 多后端，以及以 AOT native binary 为生产重点的演进方向。`comptime` 的目标不是给语言增加宏系统，而是把已经分散存在的常量折叠、枚举常量求值、泛型单态化、AOT 特化统一成一个明确、可验证、可预算的编译期语义层。

本方案的目标：

- 提供用户可见的编译期求值语法。
- 支持编译期断言和清晰诊断。
- 支持值级编译期参数，补齐类型泛型之外的特化能力。
- 支持编译期分支裁剪，让泛型函数按类型和值生成最小运行时代码。
- 保持 VM/JIT/AOT 后端语义一致：后端消费已经展开后的 Xi，不各自解释 `comptime`。
- 以确定性构建、安全沙箱、编译预算、代码体积可控为硬约束。

一句话定义：

```text
comptime 是 Xray 的编译期求值、特化和分支消除机制，不是 AST 宏系统。
```

---

## 1. 当前基线

### 1.1 已经具备的基础

Xray 现在已经有若干可以承载 `comptime` 的基础设施：

- 语言层已有 `const`、类型注解、泛型函数、泛型 class/struct、interface 约束、多返回值、module、class、struct、enum。
- parser 已能解析函数泛型参数，例如 `fn identity<T>(x: T): T`。
- analyzer 已有泛型约束、类型参数替换和单态化逻辑。
- 现有常量求值能力分散在 `xconst_fold.c`、编译器 scope 的 `ComptimeValue`、enum member 求值等位置。
- 当前主编译路径已经收敛到 Xi typed SSA，VM/JIT/AOT 都可以从同一语义来源派生。
- AOT 已有 intrinsic 和 method specialization 的现实需求，最容易从 `comptime` 获益。

### 1.2 当前缺口

当前缺口不在“能不能算常量”，而在“常量求值不是正式语言契约”：

- `const` 初始化是否可编译期求值没有统一规则。
- 编译期值类型没有统一表达。
- 泛型 key 主要覆盖类型参数，缺少值参数。
- 类型参数只能参与类型替换，不能稳定驱动 `comptime if`。
- 后端优化依赖局部特判，无法从 frontend 获得明确的“这段代码已在编译期决定”的事实。
- 缺少 deterministic build、side-effect ban、budget、diagnostic contract。

---

## 2. 用户可见设计

### 2.1 第一批语法

推荐第一批进入语言规范的语法：

```xray
const size = comptime 64 * 1024
```

```xray
comptime {
    compile_assert(size > 0, "size must be positive")
}
```

```xray
fn mask(comptime bits: int): int {
    compile_assert(bits > 0 && bits < 63, "invalid bit width")
    return (1 << bits) - 1
}
```

```xray
fn zero<T>(): T {
    comptime if (T == int) {
        return 0
    } else comptime if (T == float) {
        return 0.0
    } else comptime if (T == string) {
        return ""
    } else {
        compile_error("unsupported type")
    }
}
```

第一批能力包含：

- `comptime expr`
- `comptime { ... }`
- `compile_assert(condition, message)`
- `compile_error(message)`
- 函数参数修饰符 `comptime`
- `comptime if`

### 2.2 第二批语法

第二批可以增加：

```xray
comptime fn fib(n: int): int {
    if (n <= 1) return n
    return fib(n - 1) + fib(n - 2)
}
```

```xray
fn unroll(comptime n: int, value: int): int {
    let sum = 0
    comptime for (i in 0..n) {
        sum += value + i
    }
    return sum
}
```

```xray
struct Vec<T, comptime N: int> {
    data: Array<T>
}
```

第二批能力包含：

- 显式 `comptime fn`
- `comptime for`
- class/struct/type alias 的值级编译期参数
- 受限类型反射 builtin

### 2.3 暂不支持的能力

第一版明确不支持：

- AST 宏。
- 任意代码生成。
- 编译期文件读写。
- 编译期网络访问。
- 编译期系统时间、环境变量、随机数。
- 编译期协程、channel、task、scope、select、await、yield。
- 编译期执行普通未标记函数。
- 编译期修改外层运行时变量。
- 编译期加载 native package 动态库。

这些限制不是临时缺陷，而是为了保证构建确定性、安全性和后端一致性。

---

## 3. 语义契约

### 3.1 `const` 与 `comptime` 的区别

`const` 表示不可重新赋值。

`comptime` 表示该表达式必须在编译期求值成功。

```xray
const a = 1 + 2
```

这可以被编译器优化成常量，但语言不强制它必须作为编译期表达式处理。

```xray
const a = comptime 1 + 2
```

这要求编译器在编译期得到结果；如果做不到，编译失败。

```xray
let runtime_value = readLine()
const x = comptime runtime_value.length
```

预期诊断：

```text
runtime value 'runtime_value' cannot be used in comptime expression
```

### 3.2 编译期值类型

第一批支持的编译期值：

| kind | 用途 |
|------|------|
| int | 容量、位宽、循环次数、枚举值 |
| float | 数学常量、表生成 |
| bool | `comptime if` 条件 |
| string | 静态名称、错误消息、schema key |
| null | 可选值占位 |
| type | 泛型类型参数、类型比较 |
| range | `comptime for` 的候选基础 |
| tuple | 多返回值或内部参数打包 |

第二批再考虑不可变 array/object。它们很有用，但会引入深拷贝、结构相等、hash key 和诊断展示问题。

### 3.3 可求值表达式

第一批允许：

- 字面量。
- 编译期已知的 `const`。
- `comptime` 参数。
- 泛型类型参数。
- 一元、二元、三元表达式。
- 字符串拼接。
- enum member。
- 受限 builtin：`compile_assert`、`compile_error`、`typeName`、`sameType`、`isNumericType` 等。

第一批禁止：

- 普通函数调用，除非函数被证明为 builtin consteval。
- 分配 class instance。
- 修改 module/global/shared 状态。
- 调用有 I/O、时间、随机、网络、协程、异常副作用的 API。
- 依赖运行时变量。

### 3.4 编译期函数

第二批支持 `comptime fn` 时，函数体限制如下：

- 参数和局部变量必须是编译期值。
- 允许局部 `let` 赋值和控制流。
- 允许调用其他 `comptime fn`。
- 允许递归，但有深度和执行预算。
- 允许抛出 compile-time diagnostic，不允许运行时异常逃出。
- 不允许闭包捕获运行时变量。
- 不允许协程、I/O、系统状态、native 调用。

### 3.5 编译期分支

`comptime if` 的条件必须是编译期 bool。

未选择分支不进入 Xi lowering，也不生成运行时代码。

未选择分支是否需要完整类型检查建议分两层：

- 语法必须合法，保证 formatter/LSP 可工作。
- 与当前实例无关的类型错误可以延迟到该分支被选择时再报。

这样可支持：

```xray
fn first<T>(x: T): int {
    comptime if (T == string) {
        return x.ord()
    } else comptime if (T == Array<int>) {
        return x[0]
    } else {
        compile_error("unsupported type")
    }
}
```

当 `T == string` 时，不要求 `x[0]` 在 `string` 分支外也成立。

---

## 4. 架构落点

### 4.1 编译管线位置

`comptime` 应在 analyzer 之后、Xi lowering 之前完成。

推荐管线：

```text
Source
  -> Parser
  -> Analyzer
  -> Typed AST Canonicalizer
  -> Comptime Elaboration
  -> Generic/Value Monomorphization
  -> XiRaw
  -> XiCanonical
  -> XiClosed
  -> XiOwned
  -> XiRepped
  -> XiBackend
  -> VM/JIT/AOT
```

关键原则：

```text
Xi 和 backend 不直接执行 comptime。
```

后端只看到普通 Xi 函数、普通常量、普通分支、已经单态化的函数实例。

### 4.2 Frontend 数据结构

建议新增统一编译期值：

```c
typedef enum XrCtValueKind {
    XR_CT_INVALID,
    XR_CT_INT,
    XR_CT_FLOAT,
    XR_CT_BOOL,
    XR_CT_STRING,
    XR_CT_NULL,
    XR_CT_TYPE,
    XR_CT_RANGE,
    XR_CT_TUPLE,
} XrCtValueKind;
```

建议新增：

```text
XrCtValue
  kind
  type
  payload
  source_node_id

XrCtEnv
  symbol_id -> XrCtValue
  parent
  budget

XaComptimeFact
  node_id
  status
  value
  diagnostic_id

XaInstanceKey
  callee_symbol_id
  type_args[]
  comptime_args[]
```

### 4.3 AST 改动

建议新增 AST 节点：

```text
AST_COMPTIME_EXPR
AST_COMPTIME_BLOCK
AST_COMPTIME_IF
```

建议修改已有节点：

```text
XrParamNode
  is_comptime

FunctionDeclNode
  is_comptime_fn

ClassDeclNode / struct decl
  generic params support value params later
```

值级泛型参数建议不要塞进 `XrGenericParam` 的现有结构里。现有结构明显偏类型参数，应引入统一的参数模型：

```text
XrGenericParam
  kind = type | value
  name
  type_constraint
  value_type
  default_value
```

或者新增 `XrCompileParam`，由 function/class/struct 共用。

### 4.4 Parser 改动

需要增加 token：

```text
TK_COMPTIME
```

parser 需要支持：

- statement 起始位置的 `comptime`。
- expression prefix 的 `comptime`。
- function parameter modifier 的 `comptime`。
- `comptime if`。
- 后续支持 `comptime fn`、`comptime for`。

歧义规则：

```xray
comptime if (cond) { ... }
```

解析为 `AST_COMPTIME_IF`。

```xray
comptime { ... }
```

解析为 `AST_COMPTIME_BLOCK`。

```xray
comptime expr
```

解析为 `AST_COMPTIME_EXPR`。

### 4.5 Analyzer 改动

Analyzer 负责：

- 判断哪些 symbol 是编译期可用。
- 判断哪些表达式可 consteval。
- 检查 `comptime` 参数实参必须编译期已知。
- 检查 `comptime if` 条件必须是编译期 bool。
- 记录 `XaComptimeFact`。
- 为 value specialization 生成 instance key。
- 禁止运行时值流入编译期表达式。
- 禁止有副作用调用进入 consteval。

错误必须指向用户源位置，而不是 lowerer 或 backend internal error。

### 4.6 Comptime Elaboration

Elaboration 是把包含 `comptime` 的 typed AST 改写为普通 typed AST 的过程。

它负责：

- 把 `comptime expr` 替换成 literal 或 typed const node。
- 执行 `compile_assert` 和 `compile_error`。
- 删除 `comptime block` 中不会进入运行时的语句。
- 对 `comptime if` 只保留被选择分支。
- 对 `comptime` 参数创建函数实例。
- 记录 debug map，方便诊断指回原始源码。

### 4.7 Xi 集成

第一批不新增 `XI_COMPTIME`。

原因：

- `comptime` 是 frontend elaboration，不是运行时语义。
- Xi verifier 不需要理解编译期执行。
- VM/JIT/AOT 不需要分支实现。
- 避免三后端分别补语义。

Xi 只需要接收：

- 已替换的常量。
- 已裁剪的分支。
- 已单态化的函数。
- 已生成的普通 AST 或直接 XiRaw。

后续如需保留调试信息，可在 Xi metadata 里记录：

```text
XiDebugMap
  generated_node_id -> source_node_id
  generated_function -> generic_origin + comptime args
```

### 4.8 VM/JIT/AOT 集成

VM：

- 不执行 `comptime`。
- 用作语义参考后端，验证 elaboration 后的代码行为。

JIT：

- 不执行 `comptime`。
- 只看到更少的分支和更具体的函数实例。
- IC feedback 仍按运行时调用工作。

AOT：

- 最大受益者。
- 值级特化可减少动态 dispatch。
- `comptime if` 可消除类型分支。
- 编译期表和常量可直接进入 C const 或 static readonly 数据。

---

## 5. 实施里程碑

### 5.1 里程碑一：语法与基础诊断

内容：

- lexer 增加 `comptime` keyword。
- parser 支持 `comptime expr`、`comptime {}`、`comptime if`。
- AST 增加对应节点。
- formatter 支持新语法。
- analyzer 对新节点给出临时诊断，不进入 lowerer。

验收：

- parser 单测覆盖三种基本语法。
- formatter 输入输出稳定。
- 非法位置有清晰错误。

### 5.2 里程碑二：统一 consteval core

内容：

- 新增 `XrCtValue` 和 `XrCtEnv`。
- 把现有常量折叠能力迁移到 consteval core。
- 支持 literal、unary、binary、ternary、string concat、enum member。
- 支持 `compile_assert` 和 `compile_error`。
- 支持 `comptime expr` 替换为 literal。

验收：

- `const x = comptime 1 + 2` 生成常量 `3`。
- `compile_assert(false, "...")` 编译失败。
- 运行时变量进入 `comptime` 必须报错。
- VM/JIT/AOT 执行结果一致。

### 5.3 里程碑三：编译期参数和值级特化

内容：

- `XrParamNode` 支持 `is_comptime`。
- analyzer 检查实参必须 consteval 成功。
- 单态化 key 加入 comptime args。
- elaboration 时把 comptime 参数引用替换为编译期值。
- 生成实例名和 debug map。

验收：

- `mask(8)` 和 `mask(16)` 生成不同实例。
- 实例数量超预算时报错。
- 同一参数重复调用复用实例。
- AOT 输出中不保留 `bits` 运行时参数。

### 5.4 里程碑四：类型值和 `comptime if`

内容：

- 支持 `XR_CT_TYPE`。
- 泛型类型参数可作为编译期值参与比较。
- 支持 `T == int`、`T == Array<int>` 等类型相等判断。
- `comptime if` 只保留被选分支。
- 未选分支只做语法检查和延迟语义检查。

验收：

- `zero<int>()` 只保留 int 分支。
- `zero<string>()` 只保留 string 分支。
- 不支持的类型触发 `compile_error`。
- Xi dump 不包含未选分支。

### 5.5 里程碑五：受限 `comptime fn` 与 `comptime for`

内容：

- 支持显式 `comptime fn`。
- 支持局部变量、if、while、return。
- 支持递归预算。
- 支持 `comptime for` 基于 range 展开。
- 禁止 I/O、协程、运行时分配和外部副作用。

验收：

- `fib(10)` 编译期求值成功。
- 递归超限给出可读诊断。
- `comptime for` 产生展开后的普通语句。
- 编译预算可配置并可测试。

### 5.6 里程碑六：类型反射 builtin

内容：

- `typeName<T>()`
- `sameType<T, U>()`
- `isIntType<T>()`
- `isFloatType<T>()`
- `isStringType<T>()`
- `fieldCount<T>()`
- `fieldName<T>(index)`
- `fieldType<T>(index)`

第一版只读类型信息，不生成 AST，不访问运行时对象。

验收：

- struct schema 生成可在编译期完成。
- 字段越界在编译期报错。
- private 字段是否暴露遵循 analyzer 访问控制。

---

## 6. 详细演示效果案例

### 6.1 编译期数值计算

用户代码：

```xray
const KB = comptime 1024
const BUFFER_SIZE = comptime KB * 64

fn main(): void {
    print(BUFFER_SIZE)
}
```

编译期展开后的等价代码：

```xray
const KB = 1024
const BUFFER_SIZE = 65536

fn main(): void {
    print(65536)
}
```

运行输出：

```text
65536
```

预期后端效果：

- VM bytecode 直接加载整数常量。
- JIT 不需要为乘法生成机器码。
- AOT C 输出中直接出现 `65536`。

### 6.2 编译期断言成功

用户代码：

```xray
const CAP = comptime 256

comptime {
    compile_assert(CAP > 0, "capacity must be positive")
    compile_assert((CAP & (CAP - 1)) == 0, "capacity must be power of two")
}

fn main(): void {
    print(CAP)
}
```

运行输出：

```text
256
```

编译期效果：

- 两个断言在编译期执行。
- `comptime` block 不生成运行时代码。
- `main` 只保留 `print(256)`。

### 6.3 编译期断言失败

用户代码：

```xray
const CAP = comptime 300

comptime {
    compile_assert((CAP & (CAP - 1)) == 0, "capacity must be power of two")
}
```

预期诊断：

```text
error: capacity must be power of two
  at example.xr:4:5
```

预期行为：

- 编译失败。
- 不产生 Xi。
- 不进入 VM/JIT/AOT 后端。

### 6.4 禁止运行时值进入编译期

用户代码：

```xray
fn main(): void {
    let n = readLine().toInt()
    const size = comptime n * 2
    print(size)
}
```

预期诊断：

```text
error: runtime value 'n' cannot be used in comptime expression
  at example.xr:3:27
```

说明：

- `n` 来自运行时 I/O。
- `comptime` 表达式必须在编译期封闭。
- 编译器不能把运行时输入提升到编译期。

### 6.5 值级编译期参数

用户代码：

```xray
fn mask(comptime bits: int): int {
    compile_assert(bits > 0 && bits < 63, "invalid bit width")
    return (1 << bits) - 1
}

fn main(): void {
    print(mask(8))
    print(mask(16))
}
```

编译期生成两个实例：

```text
mask<bits=8>()
mask<bits=16>()
```

展开后的等价效果：

```xray
fn mask_bits_8(): int {
    return 255
}

fn mask_bits_16(): int {
    return 65535
}

fn main(): void {
    print(255)
    print(65535)
}
```

运行输出：

```text
255
65535
```

预期 AOT 效果：

- `bits` 不作为运行时参数传递。
- 左移和减法可以在编译期完成。
- 如果调用点直接使用结果，可以进一步内联为常量。

### 6.6 重复特化复用实例

用户代码：

```xray
fn scale(comptime factor: int, x: int): int {
    return x * factor
}

fn main(): void {
    print(scale(4, 10))
    print(scale(4, 20))
    print(scale(8, 10))
}
```

实例缓存：

```text
scale<factor=4>(x: int): int
scale<factor=8>(x: int): int
```

不会生成三个实例，只生成两个。

运行输出：

```text
40
80
80
```

预期优化效果：

- `factor=4` 的两个调用共享同一个特化函数。
- AOT 可把乘以 4 优化为移位或 immediate multiply。
- JIT 看到的是普通函数调用或内联候选。

### 6.7 类型驱动默认值

用户代码：

```xray
fn defaultValue<T>(): T {
    comptime if (T == int) {
        return 0
    } else comptime if (T == float) {
        return 0.0
    } else comptime if (T == bool) {
        return false
    } else comptime if (T == string) {
        return ""
    } else {
        compile_error("unsupported default type")
    }
}

fn main(): void {
    print(defaultValue<int>())
    print(defaultValue<string>())
}
```

`defaultValue<int>()` 展开后：

```xray
fn defaultValue_int(): int {
    return 0
}
```

`defaultValue<string>()` 展开后：

```xray
fn defaultValue_string(): string {
    return ""
}
```

运行输出：

```text
0

```

说明：

- 未选择分支不进入 Xi lowering。
- `compile_error` 分支只在没有匹配类型时触发。
- 这比运行时 `typeof` 分支更适合 AOT。

### 6.8 类型不支持时的编译期错误

用户代码：

```xray
class User {
    name: string
}

fn defaultValue<T>(): T {
    comptime if (T == int) {
        return 0
    } else {
        compile_error("unsupported default type")
    }
}

const u = defaultValue<User>()
```

预期诊断：

```text
error: unsupported default type
  while instantiating defaultValue<User>
  at example.xr:13:11
```

说明：

- 诊断应包含实例化栈。
- 用户能看到是哪个泛型实例触发错误。

### 6.9 编译期分支消除运行时成本

用户代码：

```xray
fn parseNumber<T>(s: string): T {
    comptime if (T == int) {
        return s.toInt() as T
    } else comptime if (T == float) {
        return s.toFloat() as T
    } else {
        compile_error("parseNumber only supports int and float")
    }
}

fn main(): void {
    print(parseNumber<int>("42"))
    print(parseNumber<float>("3.5"))
}
```

运行输出：

```text
42
3.5
```

AOT 期望：

- `parseNumber<int>` 只包含 `toInt` 路径。
- `parseNumber<float>` 只包含 `toFloat` 路径。
- 没有运行时类型判断。

### 6.10 固定容量类型

建议第二批支持的用户代码：

```xray
struct FixedArray<T, comptime N: int> {
    data: Array<T>

    constructor(init: T) {
        compile_assert(N > 0, "N must be positive")
        this.data = Array<T>(N, init)
    }

    length(): int {
        return N
    }
}

fn main(): void {
    let a = FixedArray<int, 4>(0)
    print(a.length())
}
```

展开效果：

```text
FixedArray<int, 4> 是独立类型实例。
N 在该实例中始终等于 4。
length() 可直接返回 4。
```

运行输出：

```text
4
```

AOT 期望：

- `length()` 可以内联为常量。
- 如果后续引入更底层的 fixed buffer layout，`N` 可直接决定 struct layout。

### 6.11 编译期循环展开

建议第二批支持的用户代码：

```xray
fn weightedSum(comptime N: int, xs: Array<int>): int {
    compile_assert(N > 0, "N must be positive")
    let sum = 0
    comptime for (i in 0..N) {
        sum += xs[i] * (i + 1)
    }
    return sum
}

fn main(): void {
    print(weightedSum(3, [10, 20, 30]))
}
```

展开后的等价效果：

```xray
fn weightedSum_N_3(xs: Array<int>): int {
    let sum = 0
    sum += xs[0] * 1
    sum += xs[1] * 2
    sum += xs[2] * 3
    return sum
}
```

运行输出：

```text
140
```

约束：

- `N` 必须是编译期 int。
- 展开语句数量必须受预算限制。
- 过大展开应提示改用运行时循环。

### 6.12 编译期生成查找表

建议第二批支持的用户代码：

```xray
comptime fn squareTable(comptime N: int): Array<int> {
    let out = []
    for (i in 0..N) {
        out.push(i * i)
    }
    return out
}

const TABLE = comptime squareTable(5)

fn main(): void {
    print(TABLE[0])
    print(TABLE[4])
}
```

编译期结果：

```xray
const TABLE = [0, 1, 4, 9, 16]
```

运行输出：

```text
0
16
```

实现注意：

- 这需要编译期不可变 array 值。
- AOT 可把表放到静态只读数据区。
- VM 可放入常量池。

### 6.13 类型反射生成 schema

建议第二批或后续支持的用户代码：

```xray
struct User {
    id: int
    name: string
    active: bool
}

comptime fn schemaOf<T>(): string {
    let s = "{"
    comptime for (i in 0..fieldCount<T>()) {
        if (i > 0) s += ","
        s += fieldName<T>(i)
        s += ":"
        s += typeName<fieldType<T>(i)>()
    }
    s += "}"
    return s
}

const USER_SCHEMA = comptime schemaOf<User>()

fn main(): void {
    print(USER_SCHEMA)
}
```

编译期结果：

```xray
const USER_SCHEMA = "{id:int,name:string,active:bool}"
```

运行输出：

```text
{id:int,name:string,active:bool}
```

边界：

- 只读类型信息。
- 不生成 AST。
- private 字段是否暴露由 analyzer 访问规则决定。

### 6.14 按目标平台选择实现

建议后续支持的用户代码：

```xray
fn pathSeparator(): string {
    comptime if (target.os == "windows") {
        return "\\"
    } else {
        return "/"
    }
}
```

macOS/Linux 展开后：

```xray
fn pathSeparator(): string {
    return "/"
}
```

Windows 展开后：

```xray
fn pathSeparator(): string {
    return "\\"
}
```

约束：

- `target` 是编译器注入的只读编译期对象。
- 不允许读取环境变量来改变结果。
- cross compile 时以目标平台为准，不以宿主平台为准。

### 6.15 禁止编译期协程

用户代码：

```xray
comptime {
    let task = go fn() { return 1 }
    let x = await task
}
```

预期诊断：

```text
error: coroutine operations are not allowed in comptime context
  at example.xr:2:16
```

原因：

- 编译期不运行调度器。
- `go`、`await`、`select`、`yield` 属于运行时并发语义。
- 保持 consteval deterministic。

### 6.16 编译期预算超限

用户代码：

```xray
comptime fn loopForever(): int {
    while (true) {}
    return 0
}

const x = comptime loopForever()
```

预期诊断：

```text
error: comptime evaluation exceeded instruction budget
  while evaluating loopForever()
  at example.xr:6:20
```

预算建议：

| 项 | 默认值 |
|----|--------|
| eval instruction budget | 100000 |
| recursion depth | 128 |
| generated AST nodes | 50000 |
| generated function instances | 1024 |
| string bytes | 1 MiB |
| array/object elements | 65536 |

这些值应允许通过编译参数调整，但默认值必须保护普通用户。

---

## 7. 诊断设计

### 7.1 必须包含的信息

`comptime` 诊断至少包含：

- 原始源位置。
- 当前求值表达式。
- 泛型实例化栈。
- 调用栈，限长展示。
- 预算项名称。
- 如果是运行时值泄漏，指出具体 symbol。

### 7.2 示例：实例化栈

```text
error: unsupported default type
  at lib/default.xr:9:9

instantiation stack:
  defaultValue<User>() at app.xr:12:15
  makeUserDefault() at app.xr:20:12
```

### 7.3 示例：运行时值泄漏

```text
error: runtime value 'n' cannot be used in comptime expression
  at app.xr:3:27

note: 'n' is defined here as a runtime local
  at app.xr:2:9
```

---

## 8. 测试方案

### 8.1 Unit tests

新增测试桶：

```text
tests/unit/frontend/test_comptime_parse.c
tests/unit/frontend/test_comptime_eval.c
tests/unit/frontend/test_comptime_diag.c
tests/unit/frontend/test_comptime_mono.c
```

覆盖：

- token。
- AST 节点。
- formatter。
- consteval primitive。
- compile_assert。
- runtime value rejection。
- value specialization key。
- type value comparison。
- budget error。

### 8.2 Xi compare tests

新增 `test_xi_compare` 用例：

- compile-time arithmetic。
- compile_assert success。
- value param specialization。
- `comptime if` type branch。
- repeated instance reuse。
- illegal branch not selected。

要求：

- legacy-free Xi pipeline 下 VM 执行结果正确。
- Xi dump 不包含未选分支。
- multi-backend diff 为零。

### 8.3 Regression tests

新增回归文件：

```text
tests/regression/comptime/001_basic_expr.xr
tests/regression/comptime/002_compile_assert.xr
tests/regression/comptime/003_value_param.xr
tests/regression/comptime/004_comptime_if_type.xr
tests/regression/comptime/005_invalid_runtime_value.xr
tests/regression/comptime/006_budget_limit.xr
tests/regression/comptime/007_instance_reuse.xr
tests/regression/comptime/008_aot_constants.xr
```

### 8.4 AOT tests

AOT 重点验证：

- 编译期常量进入 C 输出。
- `comptime if` 未选分支不出现在 C 输出。
- value param specialization 不把 comptime 参数作为运行时参数传递。
- VM-AOT 输出一致。

### 8.5 验证命令

实现涉及 C 代码后必须运行：

```bash
ctest --output-on-failure
```

完整回归：

```bash
scripts/run_regression_tests.sh
```

AOT 验证：

```bash
tests/aot/run_aot_tests.sh
```

如涉及内存安全或 consteval 分配，额外运行 ASAN 构建。

---

## 9. 风险与约束

### 9.1 编译时间

风险：复杂 consteval、递归和展开会拖慢编译。

约束：

- 每个 eval context 必须有 instruction budget。
- 每个 module 必须有 generated node budget。
- 每个 generic declaration 必须有 instance budget。
- 超限是编译错误或明确降级诊断，不允许卡死。

### 9.2 二进制体积

风险：值级参数导致实例爆炸。

约束：

- instance key 去重。
- 相同 comptime args 复用实例。
- 提供 stats 输出实例数量。
- 对高维 value specialization 给出 warning。

### 9.3 构建确定性

风险：编译期访问外部状态导致同一源码不同结果。

约束：

- 默认禁止 I/O、网络、时间、随机、环境变量。
- `target` 等编译期输入必须来自编译器显式参数。
- consteval builtin 必须是 deterministic。

### 9.4 语义分裂

风险：VM/JIT/AOT 各自实现 comptime 导致漂移。

约束：

- `comptime` 必须在 frontend elaboration 处理。
- Xi 不保留 `XI_COMPTIME`。
- 后端 verifier 不接受未展开的 comptime 节点。

### 9.5 诊断复杂度

风险：泛型实例化和 consteval 调用栈导致错误难懂。

约束：

- 记录 source map。
- 记录 instantiation stack。
- 控制调用栈展示长度。
- 对运行时值泄漏给出 definition note。

---

## 10. 与 Xray 长期架构的关系

`comptime` 应作为 compiler architecture refactor 的 frontend elaboration 能力，而不是新后端能力。

它依赖这些长期架构契约：

- Analyzer 输出稳定 binding/scope/type facts。
- Typed AST canonicalizer 固定求值顺序。
- Generic monomorphization 有统一 instance registry。
- Xi lowering 不做名字解析、不猜类型、不静默降级。
- Xi verifier 确保 backend 输入已经没有高层 frontend-only 节点。

推荐在 compiler 架构重构中把 `Comptime Elaboration` 定义为正式 pass：

```text
Analyzer facts
  -> Typed AST Canonicalizer
  -> Comptime Elaboration
  -> Monomorphization
  -> XiRaw lowering
```

---

## 11. 推荐第一版边界

第一版只做：

- `comptime expr`
- `comptime { compile_assert(...) }`
- `compile_error(...)`
- 函数参数 `comptime name: Type`
- `comptime if`
- int/float/bool/string/null/type 的编译期值
- type equality 和少量类型 predicate builtin

第一版不做：

- `comptime fn`
- `comptime for`
- array/object 编译期值
- type reflection fields API
- AST 宏
- 编译期 I/O
- 编译期协程

这样可以把收益最大的部分先落地，同时避免引入过大的解释器、沙箱和工具链复杂度。

---

## 12. 最小可交付定义

最小可交付版本必须满足：

- `comptime` token、parser、formatter 完整。
- `comptime expr` 可求值并替换为 literal。
- `compile_assert` 可成功或失败。
- 运行时值进入 `comptime` 有明确诊断。
- `comptime` 参数参与函数特化。
- `comptime if` 可按 bool/type 条件裁剪分支。
- VM/JIT/AOT 对同一测试输出一致。
- consteval 有预算，不会卡死编译器。
- regression 和 AOT 均有覆盖。

如果只实现这些，Xray 就已经拥有可用、可解释、对 AOT 有实际价值的 `comptime` 特性。
