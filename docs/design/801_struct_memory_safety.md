# Struct 设计：C# 式值类型 + xray 特色

## 一、设计哲学

**一句话**：struct 就是值类型，赋值是拷贝，GC 自动管理，程序员不需要做任何内存决策。

借鉴 C# 的核心思想：
- struct 和 Json 对象语法几乎一样，唯一区别是**值语义 vs 引用语义**
- 程序员只需要知道一条规则：**struct 赋值会拷贝**
- 不需要 Arena、不需要 unsafe、不需要生命周期标注、不需要手动释放

## 二、C# 的 struct 设计（核心参考）

C# 是"简单性 vs 性能"平衡做得最好的语言：

```csharp
// struct = 值类型，赋值是拷贝
struct Vec3 { public float X, Y, Z; }
Vec3 a = new Vec3 { X = 1, Y = 2, Z = 3 };
Vec3 b = a;      // 拷贝
b.X = 10;        // 不影响 a

// ref/in 参数：零拷贝传递
void Normalize(ref Vec3 v) { ... }    // 可变引用
float Dot(in Vec3 a, in Vec3 b) { ... } // 只读引用
```

C# struct 的精髓：
- **程序员几乎不需要做任何决策** — 小数据用 struct，其余用 class
- **不需要想"在哪分配"、"谁释放"** — 栈变量自动释放，堆上由 GC 管
- **ref/in 极其简单** — 不需要生命周期标注，编译器保证安全
- **没有 Arena、没有 unsafe（虽然有但几乎不用）** — 保持简单

C# struct 的不足（xray 要避免）：
- 装箱陷阱：`object o = myStruct` 隐式堆分配
- 可变 struct 陷阱：从 List 读取 struct 是拷贝，修改拷贝不影响原始

## 三、xray struct 设计（简化版）

### 核心原则：像 C# 一样简单

**用户只需要知道一条规则：struct 赋值会拷贝，其余全自动。**

```xray
struct Vec3 { x: float; y: float; z: float }

let a = Vec3 { x: 1.0, y: 2.0, z: 3.0 }
let b = a        // 拷贝，b 和 a 独立
b.x = 10.0      // 不影响 a
```

不需要关心：
- 内存在哪？→ GC 自动管理（Immix 堆）
- 谁释放？→ GC 自动回收
- 跨协程怎么办？→ 自动 memcpy
- 大 struct 传参慢吗？→ 编译器自动优化为 `in`（只读引用）

### 3.1 完整语法

struct 是纯数据类型，**不支持方法**。操作 struct 用普通函数 + `ref`/`in` 参数。

```xray
// 基础 struct
struct Vec3 { x: float; y: float; z: float }

// 嵌套 struct（值内联，不是指针）
struct AABB { min: Vec3; max: Vec3 }

// 定长数组字段
struct Mat4x4 { data: [16]float }

// 操作 struct 用普通函数
fn vec3_length(v: in Vec3): float {
    return math.sqrt(v.x*v.x + v.y*v.y + v.z*v.z)
}

fn vec3_normalize(v: ref Vec3) {
    let len = vec3_length(v)
    v.x /= len; v.y /= len; v.z /= len
}
```

为什么不支持方法？
- struct 是纯数据载体，和 C 的 struct 定位一致
- 普通函数 + `ref`/`in` 完全够用，且更灵活（可以放在不同模块中）
- 减少语言复杂度，和 Json/class 的区分更清晰

### 3.2 值语义规则

```xray
// 1. 赋值 = 拷贝
let a = Vec3 { x: 1.0, y: 2.0, z: 3.0 }
let b = a       // memcpy
b.x = 10.0     // a.x 仍然是 1.0

// 2. 函数参数 = 拷贝（小 struct）
fn print_vec(v: Vec3) { print(v.x, v.y, v.z) }
print_vec(a)    // 拷贝一份传入

// 3. ref 参数 = 原地修改（零拷贝）
fn normalize(v: ref Vec3) {
    let len = math.sqrt(v.x*v.x + v.y*v.y + v.z*v.z)
    v.x /= len; v.y /= len; v.z /= len
}
normalize(ref a)   // 调用端也要写 ref，明确表示"我知道会被修改"

// 4. in 参数 = 只读引用（零拷贝，编译器禁止修改）
fn dot(a: in Vec3, b: in Vec3): float {
    return a.x*b.x + a.y*b.y + a.z*b.z
}

// 5. 跨协程 = 自动 memcpy
let ch = new Channel<Vec3>(1)
ch.send(a)      // 自动拷贝到目标协程

// 6. shared let = 系统堆 + 原子引用计数
shared let pos = Vec3 { x: 0.0, y: 0.0, z: 0.0 }  // 多协程共享
```

### 3.3 ref/in 的安全保证

ref/in 不需要 Rust 那样的生命周期标注，只有 **3 条简单的编译期规则**：

```xray
// 规则 1：不能跨协程
go fn(v: ref Vec3) { ... }()  // ❌ 编译错误

// 规则 2：不能逃逸（不能赋值给变量、存入容器、返回）
fn bad(v: ref Vec3): ref Vec3 { return v }   // ❌ 编译错误
let p = ref v                                 // ❌ 编译错误

// 规则 3：ref 只在函数调用期间有效（自动保证，无需标注）
normalize(ref v)  // ref 的生命周期 = normalize 的调用期间
```

这 3 条规则在编译期静态检查，零运行时开销。比 Rust 简单得多，因为 xray 的 ref 只能用于函数参数，不能存入数据结构。

### 3.4 struct 字段约束

```xray
struct Vec3 { x: float; y: float; z: float }           // ✅ 纯值类型
struct AABB { min: Vec3; max: Vec3 }                    // ✅ 嵌套值类型
struct Pixel { r: int; g: int; b: int; a: int }        // ✅ 整数字段
struct Mat4x4 { data: [16]float }                       // ✅ 定长数组

struct Bad1 { name: string; pos: Vec3 }                 // ❌ string 是引用类型
struct Bad2 { items: Array<int> }                       // ❌ Array 是引用类型
```

**为什么 struct 不能包含引用类型？**

这是整个设计的安全基石：
- **GC 不需要扫描 struct 内部** → 零 GC 开销（GC 只标记 struct 对象本身）
- **memcpy 永远安全** → 跨协程零开销、赋值零开销
- **内存布局和 C struct 完全一致** → FFI 零开销、AOT 直接映射
- **没有悬垂引用** → GC 引用被 memcpy 后可能指向已回收的对象，禁止这种情况

如果需要"既有值字段又有引用字段"，用 Json 对象（已有的功能，引用语义，GC 管理）。

### 3.5 嵌套 struct 一步偏移

编译器在编译时计算链式字段访问的总偏移量，生成**单条指令**：

```xray
let box = AABB { min: Vec3{x:0,y:0,z:0}, max: Vec3{x:1,y:1,z:1} }
let val = box.min.x    // 一条指令：OP_STRUCT_GETF R[dst], R[box], offset=0
box.max.z = 5.0        // 一条指令：OP_STRUCT_SETF R[box], offset=40, R[src]
```

内存布局（AABB，48 字节数据）：

```
+-------------------+
| GCHeader (16B)    |
| StructDef* (8B)   |
+-------------------+  data offset=0
| min.x  float (8B) |  0
| min.y  float (8B) |  8
| min.z  float (8B) |  16
| max.x  float (8B) |  24
| max.y  float (8B) |  32
| max.z  float (8B) |  40
+-------------------+
```

编译器偏移计算过程（`box.max.z`）：

```
1. AABB 字段 "max" → type=Vec3, offset=24
2. Vec3 字段 "z"   → type=float, offset=16
3. 总偏移 = 24 + 16 = 40
4. 生成: OP_STRUCT_GETF R[dst], R[box], 40, TYPE_F64
```

这是**编译期完成的**，运行时只有一次内存读取。

### 3.6 定长数组 `[N]T` 字段

```xray
struct Mat4x4 { data: [16]float }
let m = Mat4x4 { data: [1,0,0,0, 0,1,0,0, 0,0,1,0, 0,0,0,1] }

m.data[3]       // 常量索引 → 编译为静态偏移 3*8=24
m.data[i]       // 变量索引 → OP_STRUCT_GETFI + 运行时边界检查
```

编译器对常量索引优化为静态偏移，和嵌套字段访问复用同一条 `OP_STRUCT_GETF` 指令。

### 3.7 补充语法（实现栈分配对等所需）

以下语法是实现 JIT/AOT 纯栈分配、和 C 完全一致所需的少量补充。不增加用户心智负担。

#### 零值初始化

```xray
let v = Vec3{}                     // 所有字段初始化为零值 (0.0, 0.0, 0.0)
let box = AABB{}                   // 嵌套 struct 也全部零初始化
```

AOT 映射：`Vec3 v = {0};` — 标准 C 零初始化，栈变量必备。

#### struct 作为返回值

```xray
fn make_vec3(x: float, y: float, z: float): Vec3 {
    return Vec3 { x: x, y: y, z: z }
}

let v = make_vec3(1.0, 2.0, 3.0)  // 值返回，AOT 中 C 编译器做 RVO
```

AOT 映射：`Vec3 make_vec3(double x, double y, double z) { return (Vec3){x,y,z}; }`

#### 定长数组变量

struct 字段中已有 `[N]T`，但也可以作为独立变量使用：

```xray
let points: [100]Vec3              // 100 个 Vec3，全部零初始化
points[0] = Vec3 { x: 1.0, y: 2.0, z: 3.0 }
points[i].x = 10.0                 // 直接修改
```

AOT 映射：`Vec3 points[100] = {0};` — 纯栈上数组，和 C 完全一致。

与 `Array<Vec3>` 的区别：

| 特性 | `[N]Vec3` | `Array<Vec3>` |
|------|-----------|---------------|
| 长度 | 编译时固定 | 运行时可变 |
| 分配位置 | 栈（AOT/JIT）| 堆（GC 管理）|
| 越界检查 | 编译时（常量索引）+ 运行时 | 运行时 |
| 适用场景 | 矩阵、粒子缓冲、小批量 | 动态集合 |

## 四、性能分析：堆分配 vs C# 栈分配

### 操作级对比

| 操作 | C# (栈分配) | xray (Immix 堆) | 差距 |
|------|------------|-----------------|------|
| 创建 Vec3 | ~0 ns (编译时确定栈帧) | ~5-10 ns (bump pointer + GC header) | 存在但小 |
| 字段访问 | ~1 ns (偏移) | ~1 ns (偏移) | **相同** |
| 赋值拷贝 | ~2 ns (memcpy 24B) | ~2 ns (memcpy 24B) | **相同** |
| ref/in 参数 | ~0 ns (传指针) | ~0 ns (传指针) | **相同** |
| 销毁 | ~0 ns (栈帧弹出) | ~3 ns 平摊 (GC 标记但不扫描) | 存在但小 |

Immix 快速路径（bump allocator）只有 3-5 条指令：

```c
size = (size + 7) & ~7;          // align
char *result = heap->cursor;      // read pointer
char *new_cursor = result + size; // compute new position
if (new_cursor <= heap->limit) {  // boundary check
    heap->cursor = new_cursor;    // move pointer
    return result;
}
```

### 实际场景估算

| 场景 | C# | xray | 说明 |
|------|-----|------|------|
| 每帧创建 100 个 Vec3 | ~0 μs | ~0.5-1 μs | 几乎无感 |
| 每秒创建 100 万个 Vec3 | ~0 ms | ~5-10 ms + GC | 脚本语言可接受 |
| 大量 Vec3 字段读写 | 基准 | **相同** | 偏移访问完全一致 |

**结论**：差距主要在分配和 GC，字段访问完全一样。对比 Python (~100ns/对象) 和 JavaScript (~20ns/对象)，xray 的 5-10 ns 已经很快。

### 三模式统一设计：解释器 / JIT / AOT

当前语法设计已经完备，无需额外语法即可实现纯栈分配。关键：**让编译器知道「这是 struct」**。

#### 三模式表示

```
              字节码解释器         JIT              AOT (transpile-to-C)
              ──────────         ────             ──────────────────
分配方式      Immix 堆+GCHeader  标量替换→寄存器   C 栈变量 (不逃逸)
              (必须，XrValue     (不逃逸)          ARC 堆   (逃逸)
               寄存器只有16B)     Immix 堆(逃逸)

字段访问      base + offset      寄存器直读        v.field / v->field
ref/in        传指针             传寄存器/指针      T* / const T*
GC 开销       标记但不扫描       零(标量替换后)     零(栈上/ARC)
```

#### 为什么语法不需要改变？

用户写的代码在三种模式下语义完全一致：

```xray
let v = Vec3 { x: 1.0, y: 2.0, z: 3.0 }
v.x = 10.0
normalize(ref v)
```

- **解释器**：`v` 在 Immix 堆上，XrValue 存指针，`v.x` 用 `OP_STRUCT_GETF`
- **JIT（不逃逸）**：`v` 被标量替换为 3 个寄存器 `v_x, v_y, v_z`，`v.x` 直读 `v_x`
- **AOT**：`v` 是 C 栈变量 `Vec3 v = {1.0, 2.0, 3.0};`，`v.x` 直读

**用户无感知，编译器自动选择。**

#### JIT 已有基础设施

xray 的 JIT 已实现逃逸分析 + 标量替换（`xir_pass_escape_analysis`）：

1. 找到 `XIR_ALLOC`（堆分配指令）
2. 分析逃逸条件：是否 RET、是否传给 CALL、是否跨 block、是否存入容器
3. 不逃逸 → 每个字段创建独立 vreg，STORE_FIELD/LOAD_FIELD 替换为 MOV
4. ALLOC → NOP（DCE 清除）

struct 只需复用这条路径：`OP_STRUCT_NEW` → `XIR_ALLOC` → 逃逸分析 → 标量替换。
由于 struct 只有值类型字段，逃逸分析更简单（不需要 write barrier）。

#### AOT 映射

```
xray                              C (AOT 生成)
─────────────                    ─────────────
struct Vec3 { x: float ... }     typedef struct { double x,y,z; } Vec3;
let v = Vec3{x:1, y:2, z:3}     Vec3 v = {1.0, 2.0, 3.0};  // 栈！

fn f(v: Vec3)                    void f(Vec3 v)              // 值传递
fn f(v: ref Vec3)                void f(Vec3 *v)             // 可变指针
fn f(v: in Vec3)                 void f(const Vec3 *v)       // const 指针
fn f(): Vec3                     Vec3 f(void)                // RVO

v.x                              v.x                         // 直接字段
box.min.x                        box.min.x                   // 嵌套直接访问
arr[i].x                         arr.data[i].x               // 连续存储
```

AOT 当前已有 `xcgen_struct.c`（Json→C struct promotion），原生 struct 关键字让这条路径更直接——不需要从 Shape 推断，直接生成 C typedef。

#### 逃逸条件（三模式共用）

| 条件 | 逃逸？ | 说明 |
|------|--------|------|
| 局部变量，只在当前函数使用 | ❌ | 可以栈/寄存器 |
| 作为函数参数（值/ref/in） | ❌ | 传值是 memcpy，ref/in 是指针 |
| 函数返回值 | ❌ | AOT: C 编译器做 RVO |
| 赋值给另一个局部变量 | ❌ | memcpy 到栈上新位置 |
| 存入 `Array<Struct>` | ✅ | Array 在堆上，但 struct 数据是 memcpy 进去 |
| 通过 Channel 发送 | ✅ | memcpy 到目标协程 |
| 赋值给 `shared let` | ✅ | 拷贝到系统堆 |
| 被闭包捕获 | ✅ | 拷贝到闭包 upvalue |

注意：即使"逃逸"，struct 也是 **memcpy 进去**，不是共享指针。所以逃逸不影响安全性，只影响分配位置。

#### 纯值字段约束：栈分配的安全基石

struct 只包含值类型字段，这个看似"限制"的约束，恰恰是让栈分配变得简单安全的关键：

| 效果 | 原因 |
|------|------|
| 不需要 GC root 注册 | 栈上 struct 没有 GC 指针，不需要告诉 GC |
| memcpy 永远安全 | 没有指针字段，不会产生悬垂引用 |
| 标量替换更彻底 | 所有字段都是 native 类型（int64/double） |
| AOT 直接映射 C struct | 不需要 GC 集成代码 |
| ref 安全性自动保证 | ref 的生命周期 = 函数调用栈帧，不会悬垂 |

C# 和 Java 的逃逸分析更复杂，是因为它们的对象可以包含引用类型。
xray 的纯值字段约束让 JIT/AOT 的优化路径大幅简化。

## 五、Array&lt;Struct&gt;：连续存储

没有容器支持，struct 的使用场景非常有限。`Array<Struct>` 应该是 P0 特性。

### 问题：指针方式存储 struct 没有意义

如果 Array 存的是 struct 指针（XrValue），每个元素都是独立 GC 对象，无法发挥 struct 内联的优势。

### 解决方案：连续内存存储

```xray
let particles = Array<Vec3>(1000)
particles[0] = Vec3 { x: 1.0, y: 2.0, z: 3.0 }
particles[i].x = 10.0               // 直接修改元素字段
```

底层内存布局：

```
Array<Vec3> (1 个 GC 对象，连续内存):
┌──────────────────────────────────────┐
│ GCHeader (16B)                       │
│ StructDef* / length / capacity       │
│ data:                                │
│   [0] x=1.0, y=2.0, z=3.0  (24B)   │
│   [1] x=..., y=..., z=...  (24B)   │
│   ...                                │
└──────────────────────────────────────┘
```

这和已有的 Bytes（`Array<uint8>`）机制完全一致，只是元素大小从 1 字节变为 `sizeof(struct)` 字节。

### 语义

```xray
// 读取返回拷贝（值语义）
let v = arr[0]     // v 是 arr[0] 的拷贝
v.x = 10.0         // 不影响 arr[0]

// 直接修改元素字段
arr[0].x = 10.0    // 编译为: *(float*)(arr.data + 0*24 + 0) = 10.0

// 跨协程：整个数组一次 memcpy
ch.send(arr)       // memcpy(dst, arr.data, 1000 * 24)
```

`arr[i].x = 10.0` 编译为一次偏移计算：`base + i * sizeof(Vec3) + offsetof(Vec3, x)`。

## 六、内存安全与 C# 对比

**没有 Arena。没有 unsafe。没有手动释放。没有泄漏。**

| 对象 | 管理方式 | 泄漏风险 |
|------|---------|--------|
| struct 实例 | GC 自动回收 | 零 |
| Array&lt;Struct&gt; | GC 自动回收 | 零 |
| shared struct | 系统堆 + 原子引用计数 | 零 |

### 与 C# 的差异

| 特性 | C# | xray |
|------|-----|------|
| 默认分配位置 | 栈 | GC 堆（后续可优化到栈） |
| 装箱 | 有（性能陷阱） | 无（统一 XrValue 表示） |
| ref/in | 同 | 同 + 不能跨协程约束 |
| struct 方法 | 支持 | 不支持（用普通函数） |
| `List<Vec3>` | 支持 | `Array<Vec3>` 连续存储 |
| GC 扫描 | 需要扫描 struct 引用字段 | 不需要（禁止引用字段） |
| 跨线程 | 需手动同步 | shared let 自动 memcpy |

### FFI

struct 的 C ABI 兼容布局使 FFI 更直接。xray 已有 `extern fn` + Bytes 机制，如果未来需要裸指针操作，可按需添加 unsafe 模块。

## 七、类型体系总览

```
xray 类型体系
├── 值类型（赋值拷贝，无 GC 扫描）
│   ├── int / float / bool
│   ├── struct（用户定义的值类型）
│   └── enum（值枚举，后续支持）
│
├── 引用类型（赋值共享，GC 管理）
│   ├── string（不可变，特殊处理）
│   ├── Array / Map / Set
│   ├── Json（动态对象）
│   ├── class instance
│   ├── Closure
│   └── Channel
│
└── 特殊
    └── null / undefined
```

## 八、实现路线

### P0：核心 struct（脚本层完整可用）

| 步骤 | 内容 | 估计工作量 |
|------|------|-----------|
| 1 | Parser: `struct` 声明 + `[N]T` + `ref`/`in` + `Vec3{}` 零值初始化 | ~250 行 |
| 2 | AST: `AST_STRUCT_DECL` + `AST_STRUCT_INIT` 节点 | ~80 行 |
| 3 | Type System: `XR_KIND_STRUCT` + 字段类型检查 + 返回值 | ~180 行 |
| 4 | Runtime: `XR_TSTRUCT` + `XrStructDef` + 内存布局计算 | ~200 行 |
| 5 | Compiler: struct 初始化 + 嵌套一步偏移 + ref/in 检查 | ~300 行 |
| 6 | VM: `OP_STRUCT_NEW/GETF/SETF/GETFI/SETFI` | ~150 行 |
| 7 | Deep copy: `XR_TSTRUCT` → memcpy 快速路径 | ~30 行 |
| 8 | `Array<Struct>`: 连续存储 + `arr[i].field` 编译 | ~200 行 |
| 9 | 定长数组变量: `let arr: [N]Vec3` 独立变量支持 | ~80 行 |

**P0 总计约 1470 行。**

### P1：JIT/AOT 栈分配（透明优化，不改变语义）

| 步骤 | 内容 | 估计工作量 |
|------|------|-----------|
| 10 | XIR Builder: `OP_STRUCT_NEW` → `XIR_ALLOC` + struct 元数据 | ~100 行 |
| 11 | 逃逸分析: 复用 `xir_pass_escape_analysis`，struct 路径 | ~50 行 |
| 12 | AOT: `xcgen_struct.c` 扩展，原生 struct → C typedef | ~150 行 |
| 13 | AOT: 不逃逸 struct → C 栈变量（去掉 `xrt_arc_alloc`） | ~100 行 |
| 14 | AOT: ref/in → `T*` / `const T*` 参数、返回值 RVO | ~80 行 |

**P1 总计约 480 行。**

### 实现顺序

```
P0: Parser → AST → StructDef → Type System
        → Compiler → VM opcodes → Deep copy
        → Array<Struct> → 定长数组变量

P1: XIR Builder（struct 路径）→ 逃逸分析复用
        → AOT struct typedef → 栈分配 → ref/in ABI
```

P0 完成后 struct 在脚本层完整可用。P1 是透明优化，JIT/AOT 自动栈分配，用户代码不需要任何改变。

## 九、关键设计决策

### Q: 为什么默认 GC 堆而不是栈？

脚本语言中变量生命周期难以静态确定（闭包捕获、容器存储等）。GC 堆是最安全的默认选择。编译器优化可以在确认不逃逸时将 struct 放到栈上，但这是透明的优化，不改变语义。

### Q: 为什么禁止 struct 包含引用类型？

这是核心约束，带来三大好处：

1. **GC 零开销**：不需要扫描 struct 内部
2. **memcpy 安全**：任何时候都可以安全拷贝
3. **C ABI 兼容**：内存布局和 C struct 一致

如果需要"值字段 + 引用字段"的混合类型，用 Json 对象或 class。

### Q: struct 为什么不支持方法？

struct 是纯数据载体。操作 struct 用普通函数 + `ref`/`in`：

- 更简单，减少语言复杂度
- 普通函数可以放在不同模块中，更灵活
- 和 Json/class 的定位区分更清晰（class 有方法，struct 没有）
- C 的 struct 也不支持方法，这是被验证过的简洁设计

### Q: 递归数据结构（树、图、链表）怎么办？

`struct TreeNode { left: TreeNode }` 大小无限递归——必须有间接引用。

#### 核心设计：`Ptr<T>` — 受管理的 struct 指针

```xray
struct TreeNode {
    key: int
    value: float
    left: Ptr<TreeNode>?       // 可空的 struct 指针
    right: Ptr<TreeNode>?
}

// 创建堆上 struct
let root = Ptr.new(TreeNode { key: 42, value: 1.0, left: null, right: null })
root.left = Ptr.new(TreeNode { key: 10, value: 2.0, left: null, right: null })

// 访问（自动解引用，不需要 -> 或 *）
root.key              // 42
root.left.key         // 10

// 遍历——和 C 代码一样直观
fn inorder(node: Ptr<TreeNode>?) {
    if node == null { return }
    inorder(node.left)
    print(node.key)
    inorder(node.right)
}

// 插入
fn insert(node: Ptr<TreeNode>?, key: int): Ptr<TreeNode> {
    if node == null {
        return Ptr.new(TreeNode { key: key, value: 0.0, left: null, right: null })
    }
    if key < node.key {
        node.left = insert(node.left, key)
    } else {
        node.right = insert(node.right, key)
    }
    return node
}
```

#### Ptr 的语义

| 特性 | 说明 |
|------|------|
| 约束 | `Ptr<T>` 的 T 必须是 struct 类型 |
| 赋值 | 浅拷贝（拷贝指针，和 C 的 `T*` 赋值一致） |
| 解引用 | 自动（`node.key`，不需要 `->` 或 `*`） |
| 可空 | `Ptr<T>?` 表示可空，`null` 表示空指针 |
| 内存管理 | 解释器: GC / AOT: ARC — 自动管理，不泄漏 |

#### AOT 映射——和 C 完全一致

```
xray                              C (AOT 生成)
─────────────                    ─────────────
Ptr<TreeNode>                    TreeNode*
Ptr<TreeNode>?                   TreeNode*           // NULL = null
Ptr.new(TreeNode{...})           xrt_arc_alloc(sizeof(TreeNode))

node.key                         node->key
node.left                        node->left
node.left.key                    node->left->key

fn f(n: Ptr<TreeNode>)           void f(TreeNode *n)
fn f(n: Ptr<TreeNode>?)          void f(TreeNode *n)  // n can be NULL
```

**完全是标准 C 代码。**

#### 更多场景覆盖

```xray
// 链表
struct ListNode {
    data: int
    next: Ptr<ListNode>?
}

// 混合：值字段 + 指针字段
struct Particle {
    pos: Vec3                   // 内联（值，24B）
    vel: Vec3                   // 内联（值，24B）
    next: Ptr<Particle>?        // 链表指针（8B）
}

// 异构树
struct ASTNode {
    kind: int
    children: Array<Ptr<ASTNode>>
}

// 图（邻接表）
struct GraphNode {
    id: int
    neighbors: Array<Ptr<GraphNode>>
}
```

AOT 全部直接映射为 C 的 `struct + T*`。

#### 两类 struct（编译器自动区分，用户无感知）

| 特性 | 纯值 struct | 含 Ptr struct |
|------|------------|--------------|
| 示例 | `Vec3, AABB, Mat4x4` | `TreeNode, ListNode` |
| 字段 | int/float/bool/struct/[N]T | + `Ptr<Struct>` |
| 赋值 | 深拷贝（memcpy 全部字段）| memcpy（含指针值拷贝）|
| 栈分配 | ✅ 完全可以 | struct 本身可以，Ptr 指向堆 |
| GC 扫描 | 不需要 | 扫描 Ptr 字段 |
| JIT 标量替换 | ✅ 完全展开 | 非 Ptr 字段可展开 |
| AOT 映射 | C struct（纯值）| C struct（含 `T*`）|
| ref/in 参数 | ✅ | ✅ |

编译器看到 struct 定义时自动分析：无 Ptr 字段 → 纯值 struct；有 Ptr 字段 → 含指针 struct。
**用户不需要标记，不需要关心。**

#### 和 `Arena<T>` 的关系

Ptr 和 Arena 不互斥，而是互补：

```xray
// 方式 1: Ptr（通用，优雅）
let root = Ptr.new(TreeNode { ... })
root.left = Ptr.new(TreeNode { ... })

// 方式 2: Arena + Handle（高性能，批量，缓存友好）
let arena = Arena<TreeNode>(1024)
let root = arena.alloc(TreeNode { ... })
arena[root].left = arena.alloc(TreeNode { ... })
```

| 场景 | 推荐 |
|------|------|
| 通用树/图/链表 | `Ptr<T>` — 代码最优雅 |
| ECS / 粒子系统 / 编译器 IR | `Arena<T>` — 连续内存，批量释放 |
| 快速原型 | `class` — 已有机制 |

#### 安全性

| 风险 | 如何保证 |
|------|---------|
| 悬垂指针 | GC/ARC 自动追踪，对象在所有 Ptr 释放后才回收 |
| 内存泄漏 | GC 标记清除（解释器）/ ARC refcount（AOT）|
| 循环引用 | GC 可处理；AOT 后续可引入 `WeakPtr<T>` |
| 空指针解引用 | `Ptr<T>?` 需要 null 检查，编译器可静态检查 |

#### 为什么不用 `Box<T>`？

Box 需要所有权系统（move/clone），引入后会走向 Rust 化（Rc、Arc、RefCell...）。
Ptr 更简单：**赋值就是拷贝指针，内存由 GC/ARC 管理，不需要所有权概念。**

### Q: `Ptr<T>` 总是堆分配吗？和 C 比性能如何？

**C 的 struct 指针不一定是堆分配。** C 有 4 种分配方式：

```c
// 1. 栈分配 + 取地址 — O(0)，最快，函数返回自动回收
void build_tree() {
    TreeNode a = {42, 1.0, NULL, NULL};   // 栈上
    TreeNode b = {10, 2.0, NULL, NULL};   // 栈上
    a.left = &b;                           // 指向栈上的 b
    process(&a);                           // 传栈地址
}   // a, b 随栈帧消失

// 2. 堆分配 — malloc，需手动 free
TreeNode *c = malloc(sizeof(TreeNode));
c->left = malloc(sizeof(TreeNode));

// 3. Arena 批量分配 — 连续内存，一次释放
TreeNode *pool = malloc(1000 * sizeof(TreeNode));
pool[0].left = &pool[1];
free(pool);  // 一次释放所有

// 4. 全局/静态
static TreeNode global = {0, 0.0, NULL, NULL};
```

**C 快的根本原因不是"堆分配快"，而是很多时候根本不需要堆分配：**

| 因素 | C | xray Ptr.new() (未优化) |
|------|---|------------------------|
| 栈分配 | O(0)，移动栈指针 | ❌ 总是堆分配 |
| 对象 header | 0 字节 | GC header / ARC header |
| 指针解引用 | 一条 MOV | 同 C（AOT 后） |
| GC 开销 | 零 | 标记/扫描/引用计数 |

**但编译器可以消除这个差距：**

```c
// xray 源码：
let p = Ptr.new(TreeNode { key: 42, left: null, right: null })
process(p)

// 如果编译器证明 p 不逃逸，AOT 可以生成：
TreeNode _p = {42, NULL, NULL};   // 栈上！零堆分配
process(&_p);                      // 传栈地址

// 而不是：
TreeNode *p = xrt_arc_alloc(sizeof(TreeNode));  // 堆上
```

**三模式优化（和之前的纯值 struct 一样）：**

| 模式 | 不逃逸的 Ptr.new() | 逃逸的 Ptr.new() |
|------|-------------------|-----------------|
| 解释器 | GC 堆分配 | GC 堆分配 |
| JIT | 标量替换 → 寄存器 | GC 堆分配 |
| AOT | 栈变量 + 取地址 | ARC 堆分配 |

**结论：`Ptr.new()` 的性能取决于逃逸分析。不逃逸时和 C 完全一致（零堆分配）。逃逸时有 GC/ARC 管理开销，但换来了内存安全。** 这就是 xray 的核心权衡：**安全是默认的，性能通过编译器优化透明获得。**

### Q: unsafe 如何避免 C 的忘记释放和双重释放等陷阱？

C 的四大内存陷阱：

1. **忘记释放**（内存泄漏）：malloc 后没有 free
2. **双重释放**：同一块内存 free 两次 → undefined behavior
3. **释放后使用**（use-after-free）：free 后继续用指针 → 段错误或数据损坏
4. **悬垂指针**：指向已释放或已失效内存的指针

各语言的解决方案：

| 语言 | 方案 | 保证级别 | 代价 |
|------|------|---------|------|
| Rust | 所有权 + 借用检查器 | 编译期 100% | 高学习成本 |
| Zig | Allocator 参数化 + debug 检测 | Debug 运行期 | 中 |
| Swift | ARC（自动引用计数）| 运行期自动 | 引用计数开销 |
| C# | GC + unsafe 隔离 | 安全区 100%，unsafe 无保护 | GC 开销 |

**xray 的分层安全设计：**

```
┌─────────────────────────────────────────────────┐
│  第一层：Ptr<T>（99% 代码）                       │
│  GC/ARC 自动管理，不可能有内存错误                  │
│  忘记释放 → 不可能（GC/ARC 自动回收）               │
│  双重释放 → 不可能（GC/ARC 追踪引用）               │
│  use-after-free → 不可能（有引用就不会被回收）       │
├─────────────────────────────────────────────────┤
│  第二层：unsafe.Ptr<T>（1% 极端场景）              │
│  import unsafe 才能使用                            │
│  unsafe.alloc<T>() / unsafe.free() = malloc/free  │
│  编译器禁止 unsafe 代码存储 GC 对象引用              │
├─────────────────────────────────────────────────┤
│  第三层：Runtime Guards（debug 模式安全网）          │
│  双重释放检测：free 时检查 canary 标记              │
│  use-after-free：free 后填充 poison pattern        │
│  泄漏检测：程序退出时报告未 free 的 alloc           │
│  边界检查：Ptr 索引访问时检查越界                    │
│  （release 模式下全部关闭，零开销）                  │
└─────────────────────────────────────────────────┘
```

**核心洞察：xray 和 C 的根本区别是 `Ptr<T>` 不是裸指针。**

```
C 的指针          →  裸指针，零保护，全靠程序员
xray Ptr<T>      →  GC/ARC 管理，自动安全，不可能出错
xray unsafe.Ptr  →  裸指针，但有 debug 检测作为安全网
```

99% 的用户只用 `Ptr<T>`，永远不会遇到内存管理问题。
1% 需要极致性能的用户用 `unsafe.Ptr<T>`，有 debug runtime guards 作为安全网。

**unsafe 的 Runtime Guards 示例（debug 模式）：**

```xray
import unsafe

fn example() {
    let p = unsafe.alloc<int>(10)    // malloc(10 * 8)

    unsafe.free(p)                    // 正常释放
    unsafe.free(p)                    // ❌ debug 模式: "double free detected at 0x..."

    p[0] = 42                         // ❌ debug 模式: "use-after-free at 0x..."
}
// 如果忘记 free，程序退出时：
// ⚠️ "memory leak: 80 bytes allocated at example():2 never freed"
```

这些检测在 debug 模式下自动开启（类似 ASan），release 模式下零开销。
用户不需要做任何额外操作，只要用 debug build 就自动获得安全保护。

**详细设计见 `docs/design/500_unsafe_and_lowlevel_design.md`。**

### Q: 能否合并 Ptr<T> 和 unsafe.Ptr<T> 为一套机制？

**可以，而且应该这样做。**

两套指针类型的问题：
- 学习成本：用户需要理解两种指针的区别和适用场景
- 概念负担：什么时候用 `Ptr<T>`？什么时候用 `unsafe.Ptr<T>`？
- 代码风格分裂：安全代码和不安全代码使用不同的指针类型

#### 各语言的"统一指针"方案对比

| 语言 | 指针模型 | 内存安全保证 | 学习成本 | 用户心智 |
|------|---------|------------|---------|---------|
| **Rust** | `&T` + 所有权 + 生命周期 | 编译期 100% | **极高** | 必须理解 borrow checker |
| **Swift** | ARC 自动管理 | 运行期自动 | **极低** | 不需要思考内存 |
| **Go** | `*T` + GC | 运行期自动 | **极低** | 不需要思考内存 |
| **Zig** | `*T` + allocator + defer | 编译期辅助 | 中 | 需要理解 allocator |
| **C#** | struct(值) + class(GC引用) | 值类型编译期 / 引用类型GC | 低 | 两类型分清即可 |

#### xray 的最优选择：Swift 式统一 ARC

**核心决策：`Ptr<T>` 是唯一的 struct 指针类型，不引入 `unsafe.Ptr<T>`。**

```
Ptr<T> = 唯一的 struct 间接引用
├── 编译期保证（零开销）：
│   ├── 不逃逸 → 栈分配 + 作用域释放
│   ├── 单一所有者 → move 语义 + 作用域释放
│   └── struct 类型检查 → 不能放 GC 对象
├── 运行时保证（低开销，用户无感）：
│   ├── 解释器 → GC 管理（处理循环引用）
│   └── AOT → ARC（编译器自动插入 retain/release）
└── 编译器优化（消除运行时开销）：
    ├── 连续 retain+release 对 → 消除
    ├── 创建后直接赋值 → 跳过 retain
    └── 函数内唯一引用 → 不需要原子操作
```

#### 编译器自动选择策略（用户完全无感）

```xray
// 用户写的代码始终一样：
let root = Ptr.new(TreeNode { key: 42, left: null, right: null })
root.left = Ptr.new(TreeNode { key: 10, left: null, right: null })
```

编译器根据使用方式自动选择最优策略：

| 场景 | 编译器策略 | 运行时开销 | 和 C 的差距 |
|------|-----------|----------|-----------|
| 不逃逸（局部使用） | 栈分配 + 作用域释放 | **零** | **零差距** |
| 单一所有者（赋值后不再引用） | move + 作用域释放 | **零** | **零差距** |
| 共享引用（多处持有） | ARC retain/release | 引用计数 | 略有开销 |
| 解释器模式 | GC 管理 | GC 标记 | 有开销 |

**90% 的 Ptr 使用场景是不逃逸或单一所有者，编译器可以完全消除运行时开销。**

#### AOT 生成的 C 代码示例

```xray
fn build_tree(): Ptr<TreeNode> {
    let root = Ptr.new(TreeNode { key: 42, left: null, right: null })
    let left = Ptr.new(TreeNode { key: 10, left: null, right: null })
    root.left = left
    return root
}
```

AOT 编译器分析：
- `left` 创建后赋值给 `root.left` → **move**，不需要 retain/release
- `root` 通过 return 传递给调用者 → **ownership 转移**，不需要 release

```c
// AOT 生成（优化后）
TreeNode* xr_build_tree(void) {
    TreeNode *root = xrt_arc_alloc(sizeof(TreeNode));
    root->key = 42; root->left = NULL; root->right = NULL;

    TreeNode *left = xrt_arc_alloc(sizeof(TreeNode));
    left->key = 10; left->left = NULL; left->right = NULL;

    root->left = left;   // move, no retain needed
    return root;          // ownership transfer, no release
}
```

**零冗余 ARC 操作！和手写 C 代码完全一致。**

如果不逃逸：

```xray
fn process_local() {
    let node = Ptr.new(TreeNode { key: 42, left: null, right: null })
    print(node.key)
}   // node 在这里自动释放
```

```c
// AOT 生成（不逃逸优化）
void xr_process_local(void) {
    TreeNode _node = {42, NULL, NULL};  // 栈分配！
    printf("%lld\n", _node.key);
}   // 栈帧回收，零 free 调用
```

#### 循环引用：唯一需要关注的问题

ARC 的唯一弱点是循环引用。但在 struct 场景中极少出现：

```
树结构      parent → children（单向）       → 无循环 ✅
链表        prev → next（单向）             → 无循环 ✅
图结构      node ↔ neighbors（双向）        → 可能循环 ⚠️
双向链表    prev ↔ next（双向）             → 可能循环 ⚠️
```

**处理方案（编译期）：**

编译器分析 struct 定义的依赖图，检测潜在循环：

```xray
struct Parent {
    children: Array<Ptr<Child>>
}

struct Child {
    parent: Ptr<Parent>?    // ⚠️ 编译器警告: Parent → Child → Parent 潜在循环
}
```

**三种解决方式：**

1. **解释器模式**：GC 天然处理循环引用，无需任何操作
2. **AOT 模式**：编译器检测 + `weak` 修饰符

```xray
struct Child {
    weak parent: Ptr<Parent>?   // weak 表示弱引用，不增加引用计数
}
```

3. **Arena<T> 模式**：图结构用 Arena + Handle，无引用计数问题

```xray
let arena = Arena<GraphNode>(1024)
let a = arena.alloc(GraphNode { id: 1, neighbors: [] })
let b = arena.alloc(GraphNode { id: 2, neighbors: [] })
arena[a].neighbors.push(b)   // Handle 是索引，不是引用
arena[b].neighbors.push(a)   // 无循环引用问题
```

**注意：`weak` 不是新的指针类型，只是 Ptr 字段上的修饰符。**
`Ptr<T>` 仍然是唯一的指针类型，`weak` 只是告诉编译器"这个引用不拥有对象"。

#### 为什么不需要 unsafe.Ptr？

| 原本需要 unsafe.Ptr 的场景 | 用 Ptr<T> 如何覆盖 |
|---------------------------|-------------------|
| 递归数据结构 | `Ptr<T>` 直接覆盖 |
| 高性能批量分配 | `Arena<T>` + `Handle<T>` |
| FFI 互操作 | 推迟到需要时再引入（语言扩展） |
| 自定义分配器 | 推迟到需要时再引入 |

**第一版只需要 `Ptr<T>` + `Arena<T>`，覆盖 99% 场景。**
`unsafe.Ptr<T>` 作为未来语言扩展，不影响现有代码，需要 FFI 时再引入。

#### 最终统一设计

```
用户学习的概念（只有 2 个）：
├── struct：值类型，栈分配，赋值拷贝
└── Ptr<T>：struct 指针，自动管理

内存管理（编译器自动，用户无感）：
├── 不逃逸 → 栈分配（零开销）
├── 单一所有者 → move（零开销）
├── 共享引用 → ARC/GC（自动）
└── 循环引用 → 编译器检测 + weak 修饰符

补充工具（可选）：
├── Arena<T>：批量分配，连续内存
├── weak 修饰符：解决循环引用
└── unsafe.Ptr<T>：未来扩展（FFI 场景）
```

**这就是 xray 的内存管理哲学：一套机制，编译器搞定一切，用户不需要思考内存。**

和 Swift 的对比：

| 特性 | Swift | xray |
|------|-------|------|
| 引用类型管理 | ARC (class) | ARC (Ptr<Struct>) |
| 值类型 | struct (栈) | struct (栈) |
| 弱引用 | `weak var` | `weak` 字段修饰符 |
| 编译器优化 | ARC 优化 pass | 逃逸分析 + ARC 消除 |
| 循环引用 | 运行期泄漏 | 编译期检测 + 警告 |
| 学习成本 | 极低 | 极低 |

**xray 比 Swift 更好的一点：编译器可以静态检测 struct 间的循环引用**（因为 struct 字段类型在编译期完全已知），而 Swift 的 class 可以有任意引用，无法完全静态检测。

### Q: C# 在极致性能底层场景下是如何管理内存和指针的？

C# 从 .NET Core 开始，引入了大量高性能底层特性，形成了完整的"零分配"工具链。
这些设计对 xray 有重要参考价值。

#### 1. Span\<T\> — 栈上的零拷贝内存视图

C# 最重要的高性能创新。Span 是一个指向连续内存的"安全切片"，可以指向数组、栈内存、
甚至非托管内存，**但编译器保证它不会逃逸**。

```csharp
// Span 可以指向不同来源的内存
void ProcessData() {
    // 1. 指向数组的切片（零拷贝）
    int[] array = new int[1000];
    Span<int> slice = array.AsSpan(100, 200);  // 从 index 100 开始，取 200 个

    // 2. 指向栈分配的内存
    Span<byte> stackBuf = stackalloc byte[256];

    // 3. 指向非托管内存
    IntPtr ptr = Marshal.AllocHGlobal(1024);
    Span<byte> nativeBuf = new Span<byte>((void*)ptr, 1024);

    // 统一的安全 API 操作所有三种内存
    stackBuf.Fill(0);
    stackBuf[0] = 42;
    stackBuf.Slice(0, 10).CopyTo(nativeBuf);
}
```

**关键设计：Span 是 `ref struct`（只能在栈上）：**

```csharp
ref struct Span<T> {   // ref struct = 编译器保证不逃逸
    ref T _reference;   // 内部是一个 managed pointer
    int _length;
}

// 编译器禁止：
class Foo {
    Span<int> field;           // ❌ 不能作为 class 字段
}
async Task Bar() {
    Span<int> s = stackalloc int[10];  // ❌ 不能在 async 方法中
}
Span<int> Escape() {
    Span<int> s = stackalloc int[10];
    return s;                  // ❌ 不能返回栈上的 Span
}
```

**xray 对应设计启发：**
- xray 的 `[N]T` 定长数组 + 未来的 slice 视图 = C# 的 Span
- struct 天然不逃逸（值类型） = C# 的 ref struct 约束
- 编译器逃逸分析自动保证安全

#### 2. stackalloc — 显式栈分配

```csharp
void FastSort() {
    // 栈上分配临时缓冲区（不走 GC）
    Span<int> temp = stackalloc int[128];

    // 可以安全使用，函数返回自动回收
    for (int i = 0; i < 128; i++) temp[i] = i;

    // 有边界检查，越界会抛异常
    // temp[200] = 0;  // ❌ IndexOutOfRangeException
}

// 条件栈分配：小数据栈上，大数据堆上
void AdaptiveBuffer(int size) {
    Span<byte> buf = size <= 256
        ? stackalloc byte[size]
        : new byte[size];
    ProcessBuffer(buf);
}
```

**xray 对应设计启发：**
- xray 的 `[N]T` 定长数组天然就是栈分配
- 编译器逃逸分析自动决定栈/堆，不需要 stackalloc 关键字
- AOT 模式下不逃逸的 struct/Ptr 自动栈分配

#### 3. ref / in / ref readonly — 零拷贝参数传递

```csharp
struct Matrix4x4 {  // 64 字节的大 struct
    float M11, M12, M13, M14;
    float M21, M22, M23, M24;
    float M31, M32, M33, M34;
    float M41, M42, M43, M44;
}

// in = readonly reference，不拷贝，不能修改
static float Determinant(in Matrix4x4 m) {
    return m.M11 * m.M22 - m.M12 * m.M21;  // 直接读取，零拷贝
    // m.M11 = 0;  // ❌ 编译错误：in 参数不能修改
}

// ref = 可修改的引用
static void Scale(ref Matrix4x4 m, float factor) {
    m.M11 *= factor;  // 原地修改
    m.M22 *= factor;
}

// ref return = 返回引用（避免拷贝大 struct）
static ref Matrix4x4 GetFirst(Matrix4x4[] array) {
    return ref array[0];  // 返回引用，不是拷贝
}

// 使用
Matrix4x4 m = new();
float det = Determinant(in m);     // 零拷贝读取
Scale(ref m, 2.0f);                // 原地修改
ref var first = ref GetFirst(arr); // 引用返回
```

**xray 已有对应设计：**
- `in` 参数 = C# 的 `in`（只读引用）
- `ref` 参数 = C# 的 `ref`（可修改引用）
- xray 的设计和 C# 完全一致！

#### 4. ArrayPool\<T\> — 对象池避免 GC

```csharp
void ProcessLargeData() {
    // 从池中租借数组（不触发 GC）
    byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
    try {
        // 使用 buffer...
        ReadData(buffer);
    } finally {
        // 归还到池中（不触发 GC）
        ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
    }
}
```

**xray 对应设计：**
- `Arena<T>` = 类似概念（批量分配，一次释放）
- xray 的 Arena 更强大：连续内存 + Handle 索引

#### 5. unsafe + fixed — C# 的底层指针操作

```csharp
// unsafe 块：允许使用裸指针
unsafe void DirectMemoryAccess() {
    int[] array = new int[100];

    // fixed 固定数组，防止 GC 移动
    fixed (int* ptr = array) {
        // 指针算术
        *(ptr + 10) = 42;

        // 批量内存操作
        Buffer.MemoryCopy(ptr, destPtr, 400, 400);
    }
}

// 类型双关（type punning）
unsafe float IntBitsToFloat(int bits) {
    return *(float*)&bits;
}

// 非托管内存分配
unsafe void NativeAlloc() {
    // .NET 6+ 的 NativeMemory API
    void* ptr = NativeMemory.Alloc(1024);
    NativeMemory.Clear(ptr, 1024);

    // 对齐分配（SIMD 需要）
    void* aligned = NativeMemory.AlignedAlloc(1024, 32);

    NativeMemory.Free(ptr);
    NativeMemory.AlignedFree(aligned);
}

// 结构体内存布局控制
[StructLayout(LayoutKind.Explicit)]
struct FloatIntUnion {
    [FieldOffset(0)] public float FloatValue;
    [FieldOffset(0)] public int IntValue;    // union：和 float 共享内存
}

// Unsafe 类：无边界检查的操作
void FastCopy<T>(T[] src, T[] dst) where T : struct {
    ref T srcRef = ref MemoryMarshal.GetArrayDataReference(src);
    ref T dstRef = ref MemoryMarshal.GetArrayDataReference(dst);
    Unsafe.CopyBlock(
        ref Unsafe.As<T, byte>(ref dstRef),
        ref Unsafe.As<T, byte>(ref srcRef),
        (uint)(src.Length * Unsafe.SizeOf<T>()));
}
```

**xray 对应设计：**
- unsafe 推迟到 FFI 场景再引入
- `Arena<T>` 的连续内存已覆盖大部分批量操作需求
- struct 的 `[N]T` 定长数组提供固定大小的连续内存

#### 6. C# 高性能实战：零分配 JSON 解析器

```csharp
// System.Text.Json 的高性能设计
void ParseJson(ReadOnlySpan<byte> utf8Json) {
    var reader = new Utf8JsonReader(utf8Json);  // 栈上，零分配

    while (reader.Read()) {
        switch (reader.TokenType) {
            case JsonTokenType.PropertyName:
                // GetString() 返回 string（堆分配）
                // 但高性能路径用 ValueSpan 避免分配
                ReadOnlySpan<byte> name = reader.ValueSpan;
                if (name.SequenceEqual("id"u8)) {
                    reader.Read();
                    int id = reader.GetInt32();
                }
                break;
        }
    }
}

// PipelineReader：零拷贝网络 IO
async Task ProcessPipeline(PipeReader reader) {
    while (true) {
        ReadResult result = await reader.ReadAsync();
        ReadOnlySequence<byte> buffer = result.Buffer;

        // 直接操作内核缓冲区，零拷贝
        ProcessBuffer(buffer);

        // 告诉管道已消费多少
        reader.AdvanceTo(buffer.End);
    }
}
```

#### 7. C# 高性能工具总览

| C# 特性 | 功能 | 分配位置 | GC 压力 |
|---------|------|---------|---------|
| `struct` | 值类型 | 栈/内联 | 零 |
| `Span<T>` | 内存视图 | 栈（ref struct） | 零 |
| `stackalloc` | 显式栈分配 | 栈 | 零 |
| `in/ref` | 零拷贝传参 | 引用 | 零 |
| `ArrayPool<T>` | 对象池 | 池化堆 | 极低 |
| `ref return` | 返回引用 | 引用 | 零 |
| `unsafe/fixed` | 裸指针 | 手动 | 零 |
| `NativeMemory` | 非托管内存 | 手动堆 | 零 |
| `StructLayout` | 内存布局控制 | N/A | N/A |
| `Unsafe` 类 | 无检查操作 | N/A | N/A |

**C# 的设计哲学：默认安全（GC），但提供完整的零分配工具链让开发者按需降级。**

#### 8. xray 与 C# 的对照和启发

| C# 特性 | xray 对应 | 状态 |
|---------|----------|------|
| `struct` (值类型) | `struct` (值类型) | ✅ 已设计 |
| `in/ref` 参数 | `in/ref` 参数 | ✅ 已设计 |
| `Span<T>` (安全视图) | `ArraySlice`（零拷贝）+ `[N]T` | ✅ 已实现 |
| `stackalloc` | 编译器自动栈分配 | ✅ 已设计（更优雅） |
| `ArrayPool<T>` | `Arena<T>` | ✅ 已设计 |
| `ref return` | 返回值 struct (RVO) | ✅ 已设计 |
| `unsafe/fixed` | 推迟到 FFI | 🔜 未来扩展 |
| `NativeMemory` | 推迟到 FFI | 🔜 未来扩展 |
| `StructLayout` | struct 自动布局 | ✅ 编译器决定 |
| `ref struct` 约束 | struct 天然值类型 | ✅ 不需要额外关键字 |

**关键洞察：**

1. **xray 已有 ArraySlice，功能上覆盖 Span\<T\>**：xray 的 `arr[start:end]` 语法已实现零拷贝数组视图（ArraySlice），支持所有 12 种元素类型（包括 uint8 替代 Bytes）。区别在于 C# 的 Span 是栈上 ref struct（零分配），而 xray 的 ArraySlice 是 GC 堆对象。但编译器逃逸分析可以将不逃逸的 ArraySlice 优化为栈分配，自动达到 Span 的零分配效果。

2. **xray 不需要 stackalloc**：因为编译器自动做栈分配。C# 需要 stackalloc 是因为默认是 GC 堆分配，需要显式指定栈分配。xray 的编译器自动选择最优策略，更优雅。

3. **xray 不需要 ref struct**：因为 struct 天然只能包含值类型字段，不需要额外约束。C# 需要 ref struct 是因为 struct 可以包含 class 引用，需要额外标记来禁止逃逸。

4. **xray 已有 ArraySlice**：`arr[start:end]` 语法已实现零拷贝数组视图。未来可考虑将 slice 扩展到 struct 数组（`Array<Struct>` 的 slice 视图），以及将不逃逸的 ArraySlice 通过编译器优化为栈分配。

**C# 的经验对 xray 设计的三大启发：**

```
启发 1：值类型 + 零拷贝传参 = 消除 90% 的 GC 压力
   → xray 已有：struct + ref/in

启发 2：编译器保证不逃逸 > 运行时追踪
   → xray 已有：逃逸分析自动栈分配

启发 3：分层设计 = 默认安全 + 按需降级
   → xray 的 Ptr<T>（自动管理）+ 未来 unsafe（按需）
```

**结论：xray 当前的 struct + Ptr\<T\> + Arena\<T\> 设计已经覆盖了 C# 高性能工具链的核心场景，
而且在很多方面更简洁（不需要 Span/stackalloc/ref struct 等额外概念）。
未来如果需要 slice 视图和 unsafe 操作，可以渐进式引入。**

## 十、语法设计决策（802 demo 验证后更新）

基于 802 demo 验证和详细讨论，确定以下语法决策（详见 803 文档）：

### 决策 1：`*T` + `new` 替代 `Ptr<T>` + `Ptr.new()`

```xray
// 旧：Ptr<TreeNode>? + Ptr.new(...)
// 新：*TreeNode? + new TreeNode{...}
struct TreeNode {
    key: int
    left: *TreeNode?
    right: *TreeNode?
}
let root = new TreeNode { key: 42, left: null, right: null }
```

- `*T` = 受管理指针（GC/ARC），`new T{...}` = 堆分配
- `T{...}`（无 new）= 值，栈分配
- `*T?` = nullable，`weak *T` = 弱引用（ARC 打破循环）

### 决策 2：保持 `Array<T>` 不变

不引入 `[]T` 语法。原因：Array 有构造函数（`Array(10)`）、3 个静态方法
（`Array.from/range/withCapacity`）、运行时类型名（`Type.Array`、`x is Array`），
`[]T` 无法完全替代这些用法。

### 决策 3：`[N]T` 作为独立值类型

`[N]T` 和 `Array<T>` 概念上同属数组类型族，但语法分开：

| 类型 | 语义 | 分配 | Resize | struct 字段 |
|------|------|------|--------|------------|
| `[N]T` | 值类型 | 栈/内嵌 | ❌ | ✅ |
| `Array<T>` | 引用类型 | GC 堆 | ✅ | ❌ |

`[N]T` 可用于：struct 字段、局部变量、函数参数（in/ref）、返回值。

---

## 十一、总结

> **struct = 值类型。`*T` = struct 指针。`new` = 堆分配。内存自动管理。**
> **一套机制，编译器搞定一切，用户不需要思考内存。**

### 用户视角

xray 程序员只需要知道：

1. `struct` 是值类型，赋值会拷贝
2. `*T` 是 struct 指针，`new T{...}` 堆分配，自动管理不需要手动释放
3. `ref` 原地修改，`in` 只读引用
4. `Array<T>` 动态数组（引用类型），`[N]T` 定长数组（值类型）
5. 不需要关心内存分配和释放

### 编译器视角

| 模式 | struct 命运 | `*T` 命运 | 和 C 的差距 |
|------|------------|-----------|-----------|
| 解释器 | Immix 堆 + GC | GC 管理 | 有（堆分配） |
| JIT（不逃逸）| 标量替换 → 寄存器 | 标量替换 | **零差距** |
| AOT（不逃逸）| C 栈变量 | C 栈变量 + 取地址 | **零差距** |
| AOT（逃逸）| C struct | ARC (retain/release) | 引用计数开销 |

### 设计的独特优势

- **一套指针类型**：`*T` 统一覆盖所有间接引用场景，没有 Rust 的 Box/Rc/Arc/RefCell 分裂
- **编译期循环引用检测**：struct 字段类型完全已知，编译器可以静态分析依赖图
- **三模式透明切换**：同一份代码，解释器/JIT/AOT 自动选择最优策略
- **渐进式扩展**：unsafe.Ptr 可作为未来语言扩展，不影响现有设计
- **已有基础设施**：JIT 逃逸分析 + AOT struct promotion + ARC 已实现
