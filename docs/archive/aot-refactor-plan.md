# AOT 重构计划：从 VM 加速层到独立编译目标

> Date: 2026-04-22 | Status: DRAFT
> Principle: **不考虑向后兼容——每一步都选择最佳设计**

---

## 零、当前状态 vs 目标状态

```
当前:  .xr → bytecode + XIR → .c fragments → embed in VM → VM执行module_init, AOT加速叶子函数
目标:  .xr → XIR → .c → cc → standalone binary (零VM依赖)
```

### 核心病因

所有妥协都源于一个根本决策：`main()` 创建 VM Isolate 并调用 `xr_execute()`。
这导致了 8 个连锁问题（详见审计报告），消除它需要 **一条关键路径**：

```
Step 1: 纯 C main() + module_init 序列   ← 消除 VM Isolate
Step 2: __module_init AOT 化              ← 消除字节码嵌入
Step 3: Class → typed C struct            ← 消除 xrt_arc_alloc 原始字节
Step 4: ARC retain/release codegen        ← 消除 bump-alloc-only
Step 5: 简化 tag 编号 + 删除 compat.h    ← 消除 VM 兼容层
Step 6: GETSHARED → 具名 C 全局变量       ← 消除 xrt_shared[] 平面数组
```

每一步独立可验证，可中间停顿。Step 1-2 是解除 VM 依赖的核心路径。

---

## Step 1: 纯 C main() 生成

### 目标
生成不依赖 VM 的 `main()`，直接调用 AOT 函数。

### 当前代码
```c
// xcmd_build.c:784-848 — 生成 VM bootstrap main()
int main(...) {
    XrayIsolate *X = xray_isolate_new(&params);  // VM
    struct XrProto *proto = xr_bytecode_load(X, entry->bc, entry->size);  // 字节码
    register_aot_in_proto(proto, aot_entries, 5); // JIT thunk
    int result = xr_execute(X, proto);            // VM 解释执行
}
```

### 目标代码
```c
int main(int argc, char **argv) {
    xrt_arc_init();           // ARC bump allocator
    xrt_shared_init(argc, argv);   // 全局状态
    xr___module_init(NULL);   // 直接 C 调用
    xrt_bump_destroy();
    return 0;
}
```

### 修改范围
- `xcmd_build.c:784-848` — 替换 `main()` 生成逻辑
- 删除 thunk 生成代码 (`xcmd_build.c:674-731`)
- 删除 `register_aot_in_proto` 生成
- 删除 bytecode bundle 嵌入 (`xcmd_build.c:661-664`)
- 删除 `xray_isolate.h` include

### 风险
- **高**: `__module_init` 必须先 AOT 化 (Step 2)，否则无法调用
- **中**: 失去 VM fallback — 未 AOT 化的字节码直接丢弃

### 测试策略
- 从最简测试开始：`hello.xr` (仅 println，无 class)
- 逐步启用更复杂测试
- 旧测试暂时保持 VM 模式（`--native-legacy` flag）

### 依赖
- 需要 Step 2 同步进行（module_init AOT 化）

---

## Step 2: `__module_init` AOT 化

### 目标
顶层模块代码也能 AOT 编译，不跳过 "has child closures" 的函数。

### 当前阻塞原因
```c
// xcmd_build.c:602-607
if (p->protos.count > 0) {
    printf("skip AOT hot path (creates child closures)\n");
}
```

顶层函数包含子 proto（类方法、闭包），被跳过 AOT 热路径注册。
但实际上 C 代码已经生成了——只是没有注册为 JIT thunk。

### 关键子任务

#### 2a: 消除 CLASS_FROM_DESC 对 VM 的依赖

当前 `__module_init` 生成：
```c
v13 = INT64_C(67436544);                  // CLASS_FROM_DESC → 魔术数字
xrt_shared[0] = xrt_box_int(v12);         // SETSHARED → 存储到 shared
v18 = ((XrValue (*)(...))((...)->fn))(...); // 间接闭包调用 → 构造实例
```

目标：
```c
// CLASS_FROM_DESC → 无操作（class 在编译期已知）
// SETSHARED[0] = class → 无操作（构造函数直接调用）
// Call Vec2(3, 4) → 直接 C 构造
XrValue v18;
{ XrValue _inst = {.ptr = xrt_arc_alloc(32), .tag = XRT_TAG_PTR};
  xr_constructor(NULL, _inst, xrt_box_float(3), xrt_box_float(4));
  v18 = _inst; }
```

需要：
- XIR builder 的 OP_CLASS_CREATE_FROM_DESCRIPTOR → 标记为 no-op / elide
- OP_CALL on class value → 检测 callee_proto 是 constructor，走 emit_call_known 路径
- 当前 `scale`/`add` 中的构造函数调用逻辑已经实现，需要推广到 module_init

#### 2b: 消除 closure creation 对 VM 的依赖

当前：
```c
v1 = xrt_closure_new((void*)xr_constructor, 0);  // 创建闭包对象
```

这些闭包只用于存入 class descriptor，然后通过 INVOKE_METHOD 查找。
在 AOT 中完全不需要——方法由 CHA devirtualization 直接解析。

需要：
- OP_CLOSURE → 如果 callee 已知且不逃逸，标记为 dead code
- DCE 会自动消除

#### 2c: println / 运行时调用

module_init 中的 `println(v.x)` 等调用需要 AOT 运行时支持。
当前 `xrt_method.h` 已有 `xrt_println`，应可直接使用。

### 修改范围
- `xir_builder_object.c` — OP_CLASS_CREATE_FROM_DESCRIPTOR AOT 处理
- `xir_builder_call.c` — OP_CALL 对 class value 的处理（推广构造函数逻辑）
- `xcmd_build.c:602-607` — 移除 "has child closures" 跳过逻辑
- `xcgen_call.c` — CALL_DIRECT 闭包调用 fallback（暂时保留？）

### 风险
- **高**: 大量字节码模式需要在 XIR builder 中处理
- **中**: 某些动态模式（元编程、eval）无法 AOT 化

### 测试策略
- `hello.xr` → `println` 直接调用
- `arithmetic.xr` → 基本算术 + 输出
- `class_methods.xr` → 类构造 + 方法调用 + 输出
- 每个测试：`xray build --native test.xr -o test && ./test`
- 对比 `xray run test.xr` 输出

### 验收标准
- `hello.xr` 生成的二进制不包含字节码 blob
- `ldd` / `otool -L` 不显示 libxray 依赖

---

## Step 3: Class → Typed C Struct

### 目标
从 `xrt_arc_alloc(N)` 原始字节 → `xrt_obj_alloc(TYPE_ID, sizeof(XrtObj_T))` 有类型实例。

### 当前代码
```c
// 实例分配
XrValue _inst = {.ptr = xrt_arc_alloc(32), .tag = XRT_TAG_PTR};
// 字段访问
*(double*)((char*)v0.ptr + 0)      // this.x
*(double*)((char*)v0.ptr + 16)     // this.y
```

### 目标代码
```c
// 生成的 struct
typedef struct {
    XrtObjHeader hdr;
    double x;
    double y;
} XrtObj_Vec2;

// 实例分配
XrtObj_Vec2 *_inst = (XrtObj_Vec2 *)xrt_obj_alloc(TYPE_ID_VEC2, sizeof(XrtObj_Vec2));
// 字段访问
_inst->x
_inst->y
```

### 关键子任务

#### 3a: Class descriptor → C struct typedef 生成

从 `XrClassDescriptor` 中提取：
- 类名
- 字段名 + 类型（从 TFIELD_GET/SET 的 float bitmap 推断）
- 继承关系（parent descriptor）

生成到 `XCGEN_SEC_TYPES` section。

#### 3b: 全局类型表注册

在 module_init 开头调用：
```c
static XrtMethodFn vec2_vtable[] = { ... };
uint16_t TYPE_ID_VEC2 = xrt_type_register("Vec2", 0, vec2_vtable, N, NULL, sizeof(XrtObj_Vec2));
```

#### 3c: vtable 生成

从 class descriptor 的 `instance_methods[]` 生成 vtable 数组。
CHA devirtualization 后大部分是直接调用，vtable 仅用于多态场景。

#### 3d: 字段访问 codegen 更新

将 `*(double*)((char*)v0.ptr + offset)` 替换为 `((XrtObj_Vec2*)v0.ptr)->x`。
需要在 XIR LOAD_FIELD/STORE_FIELD 中传递 struct 类型信息。

### 修改范围
- `xcgen.c` — 新增 struct typedef 生成
- `xcgen_call.c:emit_call_known` — 构造函数用 `xrt_obj_alloc`
- `xcgen_expr.c` — LOAD_FIELD/STORE_FIELD 用 struct 成员名
- `xrt_class.h` — 已实现，直接使用
- `xcmd_build.c` — 传递 class descriptor 信息到 codegen

### 风险
- **中**: 需要从字节码/descriptor 恢复字段名（可能只有 symbol ID）
- **低**: struct layout 必须与 VM 的 field offset 兼容（但按设计不需要兼容）

### 测试策略
- `class_basic.xr` → 基本 class 字段读写
- `class_methods.xr` → 方法中创建新实例
- 验证 `sizeof(XrtObj_Vec2)` 正确
- 验证 `instanceof` 通过 type_id 工作

---

## Step 4: ARC Codegen

### 目标
在生成的 C 函数中插入 `xrt_retain/xrt_release`。

### 当前状态
- `xrt_arc.h` 有完整 retain/release 实现（含 bump alloc 快路径）
- `xrt_class.h` 有 `xrt_obj_retain/release`
- **零** `retain/release` 调用在生成代码中

### 关键子任务

#### 4a: 函数参数 retain / scope-end release

```c
static XrValue xr_scale(XrtContext ctx, XrValue self, XrValue factor) {
    xrt_arc_retain_val(self);     // retain params
    xrt_arc_retain_val(factor);
    // ... body ...
    xrt_arc_release_val(factor);  // release at scope end
    xrt_arc_release_val(self);
    return result;                // ownership transferred
}
```

#### 4b: Last-use 优化 (move semantics)

XIR SSA 天然支持 last-use 分析：如果一个 vreg 只有一个使用点且是最后使用，
跳过 retain 并将 release sink 到使用点之后。

这是 Nim ARC 的核心优化，直接在 codegen 层实现：
- 扫描每个 vreg 的使用计数
- 只有 1 次使用的 ptr-tagged vreg → 不插入 retain，移动所有权

#### 4c: 统一 xrt_arc vs xrt_class ARC

当前有两套 ARC：
- `xrt_arc.h`: `XrtArcHdr` (位于用户数据前)
- `xrt_class.h`: `XrtObjHeader` (嵌入在 struct 开头)

**决策**: 统一为 `XrtObjHeader` 模型（与设计文档一致）：
- 删除 `XrtArcHdr` 前缀模型
- 所有堆对象 (string, array, map, class instance) 以 `XrtObjHeader` 开头
- `xrt_obj_alloc` 为唯一分配入口
- bump allocator 保留为 `xrt_obj_alloc` 的快路径

### 修改范围
- `xcgen.c` — 函数序言/收尾插入 retain/release
- `xcgen_expr.c` — 赋值时 retain 新值 + release 旧值
- `xrt_arc.h` — 简化为 `xrt_obj_alloc` 的 bump 快路径
- `xrt_class.h` — 成为唯一 ARC 入口
- 删除 `XrtArcHdr` 分裂

### 风险
- **高**: 错误的 retain/release 配对导致 use-after-free 或泄漏
- **中**: 性能回退（过多 retain/release 调用）

### 测试策略
- AddressSanitizer 构建
- 计数器验证：`retain_count == release_count` (program exit)
- Valgrind leak check

---

## Step 5: Tag 编号简化 + 删除 compat.h

### 目标
- Tag 编号简化为设计文档的 `NULL=0, BOOL=1, I64=2, F64=3, PTR=4`
- 删除 `xrt_compat.h`（XrValue 别名、XR_TAG_* 宏）
- 全部用 `XrtValue` + `XRT_TAG_*`

### 当前阻塞原因
Tag 值必须与 VM `xvalue.h` 一致（因为 JIT thunk 在两者之间传递值）。
Step 1-2 消除 VM 依赖后，此约束消失。

### 修改范围
- `xrt_value.h` — 修改 tag 定义
- 删除 `xrt_compat.h`
- `xcgen*.c` — 所有生成 `XrValue` → `XrtValue`
- `xrt_*.h` — 移除 XR_ 前缀引用

### 风险
- **低**: 纯文本替换，可自动化

### 测试策略
- 全套 AOT 测试重新运行

---

## Step 6: GETSHARED → 具名 C 全局变量

### 目标
```c
// 当前
static XrValue xrt_shared[1];
xrt_shared[0] = ...;

// 目标
static XrtValue mod_main__Vec2_class;   // 类对象（或直接 elide）
static XrtValue mod_main__v1;           // 顶层 let 绑定
extern XrtValue mod_math__pi;           // 跨模块 import
```

### 关键子任务

#### 6a: shared index → 具名变量映射

从 proto 的 shared variable table 提取变量名。
`SETSHARED[idx]` → `mod_<module>__<name> = expr;`

#### 6b: 跨模块 extern 声明

import 的模块变量 → `extern XrtValue mod_<dep>__<name>;`
在多模块编译时（Phase D），由 linker 解析。

### 修改范围
- `xcmd_build.c:build_shared_proto_map` — 提取变量名
- `xcgen.c` — 声明具名全局变量
- `xcgen_call.c` — GETSHARED/SETSHARED codegen 使用变量名

### 风险
- **中**: 共享变量表可能不含名字（仅有索引）
- 需要验证 proto->shared_names 是否存在

### 测试策略
- `shared_vars.xr` → 模块内共享变量
- `modules/export_const.xr` → 跨模块 import

---

## 执行顺序 & 里程碑

```
Week 1:  Step 1 + Step 2a (纯 C main + CLASS_FROM_DESC elide)
         里程碑: hello.xr 无 VM 依赖独立运行

Week 2:  Step 2b-2c (closure elide + println)
         里程碑: class_methods.xr 无 VM 独立运行

Week 3:  Step 3a-3c (class → C struct + vtable)
         里程碑: instanceof 测试通过

Week 4:  Step 4a-4b (ARC retain/release)
         里程碑: AddressSanitizer clean

Week 5:  Step 5 + Step 6 (tag 简化 + 具名全局变量)
         里程碑: 设计文档 100% 符合

Week 6:  Step 4c (ARC 统一) + 回归测试 + 文档更新
         里程碑: Phase A+B 完成
```

### 每步验收标准

| Step | 验收 |
|------|------|
| 1 | `main()` 不含 `XrayIsolate`、`xr_execute`、`xr_bytecode_load` |
| 2 | 生成 .c 不含 `xr_app_mod0_bc[]` 字节码 blob |
| 3 | 生成 .c 包含 `typedef struct { XrtObjHeader hdr; ... } XrtObj_*;` |
| 4 | `grep -c 'xrt_.*retain\|xrt_.*release' output.c` > 0 |
| 5 | `grep XR_TAG_ output.c` = 0; `grep xrt_compat output.c` = 0 |
| 6 | `grep xrt_shared output.c` = 0 |

---

## 降级策略

如果某个 Step 遇到阻塞（如某些字节码模式无法 AOT 化），可以：

1. **标记为 runtime fallback** — 生成 `xrt_call_dynamic(...)` 调用运行时分发
2. **保留该测试为 skip** — 在测试脚本中标记 `SKIP_AOT_STANDALONE`
3. **不引入 VM 回退** — 宁可报错，不走"半 VM 半 AOT"

---

## 废弃物清单（完成后删除）

| 文件/代码 | 所属 Step | 说明 |
|-----------|----------|------|
| `xcmd_build.c` thunk 生成 (line 674-731) | Step 1 | JIT calling convention adapters |
| `xcmd_build.c` bytecode bundle embed | Step 2 | `xr_app_mod0_bc[]` |
| `xcmd_build.c` `register_aot_in_proto` | Step 1 | proto tree traversal |
| `xrt_compat.h` | Step 5 | VM type aliases |
| `XrtArcHdr` in `xrt_arc.h` | Step 4c | 统一为 XrtObjHeader |
| `xrt_shared[]` flat array | Step 6 | 替换为具名全局变量 |
| `xcgen_bridge.h` JIT runtime forward decls | Step 1 | AOT 不再需要 JIT helpers |
