---
id: spec.9_generics
order: 010
---

<!-- xr-spec:cn -->
---

## 9. 泛型 (Generics)

> 真值源：`src/frontend/analyzer/xanalyzer_generic.c`、`src/frontend/analyzer/xanalyzer_subtype.c`。

### 9.1 类型参数语法 `<T>`

```ebnf
TypeParams ::= '<' TypeParam (',' TypeParam)* '>'
TypeParam  ::= Identifier (':' ConstraintList)?
ConstraintList ::= Type ('&' Type)*               // 交叉约束用 '&' 连接
TypeArgs   ::= '<' Type (',' Type)* '>'
```

```xray
// 泛型函数
fn identity<T>(x: T) -> T {
    return x
}

let a = identity<int>(42)
let b = identity("hello")               // 推断 T=string

// 泛型类
class Box<T> {
    value: T
    constructor(v: T) { this.value = v }
    get() -> T { return this.value }
}

let b1 = new Box<int>(42)
let b2 = new Box<string>("hi")

// 多参数泛型
class Pair<K, V> {
    key: K
    value: V
    constructor(k: K, v: V) {
        this.key = k; this.value = v
    }
}

// 泛型接口
interface Comparable<T> {
    compareTo(other: T) -> int
}
```

### 9.2 类型约束：`<T: Constraint>` 与交叉约束 `&`

xray 的约束语法统一用冒号 `:`，多个约束用 `&` 连接（读作“同时满足”）。**不使用** Java/TS 的 `extends` / `implements` 作为约束关键字。

```xray
// 单一约束
fn first<T: Comparable>(a: T, b: T) -> T {
    return a
}

// 多个约束（交叉）——T 必须同时满足 Comparable、Hashable、Stringable
fn passThrough<T: Comparable & Hashable & Stringable>(x: T) -> T {
    return x
}

// 多个类型参数，每个独立约束
fn pickValue<K: Hashable, V>(k: K, v: V) -> V {
    return v
}
```

**内置约束接口**（详见 §14.14）：

| 接口 | 含义 |
|---|---|
| `Comparable` | 可用 `<` `<=` `>` `>=` 比较；int/float/string/Comparable 实现者 |
| `Hashable` | 可作为 `Map` / `Set` 的键；int/float/string/bool/enum/Hashable 实现者 |
| `Stringable` | 可调 `.toString()`；几乎所有内置类型默认实现 |
| `Iterable<T>` | 可被 `for-in` 遭历；Array、Map、Json、string、Range、enum、自定义 `iterator()` |

**当前限制**：
- 约束只能位于类型参数后，不支持 where 子句。
- 不支持**高阶类型**（`F<_>` 作为参数）。
- 不支持默认类型参数（`<T = int>`）。
- 接口实现仍需**显式 `implements`**（在类声明位置，不是约束位置，详见 §5.4）。

### 9.3 类型推断与显式实例化

#### 类型推断

```xray
identity(42)                    // T 推断为 int
new Box("hello")                // T 推断为 string
new Pair("key", 100)            // K=string, V=int
```

推断算法是**双向推断**：
- 从参数推断（调用位置实参类型 → 类型参数）。
- 从返回值推断（上下文期望类型 → 类型参数）。

#### 显式实例化

在推断失败或需要明确时：

```xray
let empty = new Array<int>()              // 无元素可推
let m = new Map<string, int>()
let result = identity<float>(0)            // 0 默认 int，强制 float
```

### 9.4 特化与 monomorphization

**实现策略**：编译期 monomorphization（单态化）。

- 编译器收集所有泛型函数/类的具体类型实例化点，为每个类型组合生成专用 AST 克隆并编译为独立字节码。
- 名字修饰（name mangling）：`identity<int>` → `identity$i64`，`Pair<string, int>` → `Pair$str$i64`。
- 按表示分组共享（rep-sharing）：同为指针表示的类型共用同一份特化（最多 3 版本：I64 / F64 / PTR）。
- 编译期严格类型检查保证安全；运行时保留具体类型参数信息供 `Reflect.typeOf` 使用。

> 真值源：`src/frontend/analyzer/xanalyzer_mono.c`（单态化 pass）、`xanalyzer_mono.h`（API）。

**性能影响**：
- 单态化的泛型函数可被 JIT / AOT 直接优化为原生类型操作（无装箱）。
- 内置特化容器（`Array<int>`、`Bytes`）进一步避免装箱开销。
- 编译体积随实例化组合数线性增长；上限 `XR_MONO_MAX_INSTANCES` 防止爆炸。

### 9.5 协议（duck typing）与名义类型

#### 名义类型为主

xray 的接口实现需**显式 `implements`**——这与 Go 的"隐式接口实现"不同。

```xray
interface Drawable { draw() -> () }

class Square implements Drawable {        // 必须显式 implements
    draw() -> () { ... }
}

class Wrong {
    draw() -> () { ... }
}

fn render(d: Drawable) { ... }
render(new Square())     // OK
render(new Wrong())      // 编译错误：Wrong 不是 Drawable
```

#### 结构化对象

仅`object literal` 与 `type T = {...}` 是结构化匹配：

```xray
type Point = { x: float, y: float }

fn describe(p: Point) { ... }

describe({ x: 1.0, y: 2.0 })   // OK：字面量结构匹配
describe({ x: 1.0, y: 2.0, z: 3.0 })  // 编译错误：sealed 类型多了字段
```

### 9.6 方差（Variance）

当前不支持显式方差标注（`out T` / `in T`）。默认行为：
- 容器类型：**不变**（`Array<Dog>` 不是 `Array<Animal>` 的子类型）。
- 函数类型：参数逆变、返回值协变（标准规则）。

### 9.7 泛型与运行时反射

由于 monomorphization，每个具体实例化在运行时都有独立的类/函数定义，且保留了类型参数信息：

```xray
class Container<T> {
    items: Array<T>
}
let c = new Container<int>()
print(Reflect.typeOf(c))       // "Container<int>"
```

对具体值的类型检查使用 `is` / `as`。
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 9. Generics

Generic functions, classes, structs, interfaces, and ADT enums use type parameters:

```xray
fn id<T>(x: T) -> T { return x }
class Box<T> { value: T }
enum Option<T> { Some(T) None }
```

Constraints use `:`:

```xray
fn max<T: Comparable>(a: T, b: T) -> T {
    return a > b ? a : b
}
```

Multiple constraints use `&`:

```xray
fn f<T: Comparable & Stringable>(x: T) -> string {
    return x.toString()
}
```

Built-in constraint interfaces include:

| Constraint | Meaning |
|--|--|
| `Comparable` | supports ordering comparisons |
| `Hashable` | valid as a Map/Set key |
| `Stringable` | has string conversion |
| `Iterable<T>` | can be used in `for-in` |
<!-- /xr-spec:en -->
