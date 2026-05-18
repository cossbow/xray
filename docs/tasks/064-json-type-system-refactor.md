# 064 — Json 类型系统重构

## 目标

将当前混合的 Json/JsonValue/结构类型 设计重构为清晰的两层模型：

1. **静态结构类型** (`type User = { name: string }`) — 编译期字段确定，不可扩展
2. **动态 Json 类型** (`Json`) — 单一自递归类型，代表任意 JSON 值，可扩展

同时将 `stdlib/json` 模块的核心能力提升为 `Json` 类型的内建方法，消除 `json`（模块）与 `Json`（类型）的混淆。

## 现状问题

### 问题 1：Json 和 JsonValue 双类型

- `Json` 仅代表 JSON 对象容器
- `JsonValue` 是单独的 union `(bool|int|float|string|Json|Array<unknown>)?`
- 两种访问路径返回不同类型：`data.name` → `Json?`，`data["name"]` → `JsonValue`
- `JsonValue` 不是自递归的：`Array<unknown>` 而非 `Array<JsonValue>`

### 问题 2：结构类型与动态 Json 共享 XR_KIND_JSON

- `type User = { name: string }` 内部是 `XR_KIND_JSON` + `allow_extension=false`
- 裸 `Json` 是 `XR_KIND_JSON` + `field_count=0`
- 导致：非 optional 字段也自动 nullable、静态类型和动态类型共享规则

### 问题 3：json 标准库与 Json 类型混淆

- `json.parse(str)`（stdlib 模块）vs `data.name`（Json 类型能力）
- 用户要 `import json` 才能做最基本的 JSON 解析
- `json.typeof()` 与语言的 `typeof/typename` 语义不同但名字相似

### 问题 4：unknown 传播过宽

- Analyzer 中约 124 处返回 `xr_type_new_unknown()`
- 导致 codegen 频繁退化到泛型指令
- VM 被迫做大量运行时类型分发

## 设计方案

### 最终用户心智模型

```
┌───────────────────────────────────────────────────────┐
│  type User = { name: string, age: int }               │
│  → 结构类型，编译期固定                                   │
│  → user.name : string                                 │
│  → user.foo  : ❌ 编译错误                              │
├───────────────────────────────────────────────────────┤
│  let data: Json = Json.parse(text)                    │
│  → 动态类型，运行时确定                                   │
│  → data.name : Json                                   │
│  → data.foo  : Json（可能是 null）                      │
├───────────────────────────────────────────────────────┤
│  let user: User = Json.decode<User>(data)             │
│  → 动态 → 静态的显式桥梁                                 │
└───────────────────────────────────────────────────────┘
```

### 设计决策

| 决策 | 选项 | 结论 |
|------|------|------|
| JsonValue 的命运 | 修复递归 vs 删除 | **删除**，`Json` 本身代表任意 JSON 值 |
| Json 的语义 | 仅对象 vs 任意 JSON 值 | **任意 JSON 值**（含 null/bool/int/float/string/array/object） |
| 结构类型的内部 kind | 继续用 XR_KIND_JSON vs 新增 XR_KIND_OBJECT | **新增 XR_KIND_OBJECT** |
| allow_extension 标志 | 保留 vs 删除 | **删除**，结构类型永远不可扩展，Json 永远可扩展 |
| json stdlib 模块 | 保留 vs 合并到 Json 类型 | **合并后废弃** |
| `let x = { a: 1 }` 无注解推断 | 推断为结构类型 vs Json | **结构类型**（强类型默认） |
| Json 字段访问返回类型 | Json? vs Json | **Json**（Json 本身包含 null） |
| `Json?` 写法 | 允许 vs 编译错误 | **编译错误**，Json 已包含 null，`Json?` 是语义重复 |

## 实施计划

### S0：统一 Json 字段访问返回类型（1-2 天）

**目标**：`data.foo` 和 `data["foo"]` 对裸 Json 返回相同类型。

**改动**：

| 文件 | 改动 |
|------|------|
| `src/frontend/analyzer/xanalyzer_visitor_expr.c` | `xa_visit_member_access()` 中，bare Json 的 member access 从返回 `Json?` 改为返回 `JsonValue` |

**不动的**：schema'd Json 的行为不变（有字段信息时返回对应类型）。

**测试**：
- `ctest --output-on-failure`
- `scripts/run_regression_tests.sh`
- 检查 `tests/regression/13_types/1302..1310_json*.xr` 是否需要更新期望输出

**验收标准**：`data.foo` 和 `data["foo"]` 对同一个 bare Json 变量返回相同类型。

---

### S1：Json 语义扩展为"任意 JSON 值"，删除 JsonValue（1 周）

**目标**：`Json` 成为唯一的动态 JSON 类型，概念上代表 `null | bool | int | float | string | Array<Json> | JsonObject`。

#### S1.1 删除 JsonValue 类型构造

| 文件 | 改动 |
|------|------|
| `src/runtime/value/xtype.c` | 删除 `xr_type_new_json_value()` 函数（约 25 行） |
| `src/runtime/value/xtype.c` | 删除 `xr_type_is_json_value()` 函数 |
| `src/runtime/value/xtype.h` | 删除 `xr_type_new_json_value()` 和 `xr_type_is_json_value()` 声明 |
| `src/runtime/xisolate_internal.h` | 删除 `json_value_type` 缓存字段 |

#### S1.2 所有原先返回 JsonValue 的地方改为返回 Json

| 文件 | 位置 | 改动 |
|------|------|------|
| `src/frontend/analyzer/xanalyzer_visitor_expr.c` | `xa_visit_member_access()` | bare Json → 返回 `xr_type_new_json()` |
| `src/frontend/analyzer/xanalyzer_visitor_expr.c` | `xa_visit_index_get()` | Json subscript → 返回 `xr_type_new_json()` |
| `src/frontend/parser/xparse_type.c` | `JsonValue` 关键字解析 | 改为等同于 `Json`（兼容期），或直接报 deprecation warning |

#### S1.3 更新 Json 相关类型兼容性逻辑

| 文件 | 改动 |
|------|------|
| `src/runtime/value/xtype.h` | `xr_is_json_coercion()` — 去掉 `xr_type_is_json_value` 分支 |
| `src/runtime/value/xtype.c` | `xr_type_assignable()` — 简化 JsonValue 相关分支 |
| `src/vm/xvm_dispatch_object.inc.c` | 检查是否有 JsonValue 特殊处理（当前 1 处引用） |

#### S1.4 禁止 `Json?` 写法

因为 `Json` 本身语义包含 null（null 是合法 JSON 值），`Json?` 是语义重复。编译器在遇到 `Json?` 类型注解时应产生编译错误：

```
error: 'Json?' is not allowed — Json already includes null as a valid value.
       Use 'Json' instead.
```

**实现**：

| 文件 | 改动 |
|------|------|
| `src/frontend/analyzer/xanalyzer_visitor_decl.c` | 变量声明类型检查：如果 base type 是 Json 且标记 nullable，报错 |
| `src/frontend/parser/xparse_type.c` | 类型解析时：`Json?` → 编译错误 |
| `src/frontend/analyzer/xanalyzer_visitor_expr.c` | 函数签名返回类型检查 |

#### S1.5 Json 字段访问统一返回 `Json`（不再是 `Json?`）

字段访问不需要额外的 nullable 包装：

| 场景 | 旧返回 | 新返回 |
|------|--------|--------|
| `data.name`（bare Json） | `Json?` | `Json` |
| `data["name"]`（bare Json） | `JsonValue` | `Json` |
| `data.name`（schema'd Json, field exists） | `T?` | `T?`（保持） |
| `data.name`（schema'd Json, field unknown） | `Json?` | `Json` |

**测试**：
- `ctest --output-on-failure`
- `scripts/run_regression_tests.sh`
- 重点关注：`tests/regression/13_types/1301..1310_json*.xr`
- 重点关注：`tests/compile_errors/json/` 目录
- 重点关注：`tests/jit/018..020_json*.xr`

**验收标准**：
- 语言中不再存在 `JsonValue` 类型
- 所有 Json 字段访问统一返回 `Json`
- 全量回归通过

---

### S2：新增 XR_KIND_OBJECT，拆分结构类型（2-4 周）

**目标**：`type User = { name: string, age: int }` 使用独立的 `XR_KIND_OBJECT`，与 `XR_KIND_JSON` 彻底分离。

#### S2.1 类型系统层

| 文件 | 改动 |
|------|------|
| `src/runtime/value/xtype.h` | `XrTypeKind` 枚举新增 `XR_KIND_OBJECT` |
| `src/runtime/value/xtype.h` | `XrObjectType` 结构体删除 `allow_extension` 字段 |
| `src/runtime/value/xtype.h` | 新增 `XR_TYPE_IS_OBJECT(t)` 宏 |
| `src/runtime/value/xtype.h` | `xr_kind_is_object_like()` 加入 `XR_KIND_OBJECT` |
| `src/runtime/value/xtype.c` | 新增 `xr_type_new_object()` 构造函数（复用现有签名但 kind 改为 OBJECT） |
| `src/runtime/value/xtype.c` | `xr_type_assignable()` 新增 OBJECT 兼容规则 |

#### S2.2 Parser 层

| 文件 | 改动 |
|------|------|
| `src/frontend/parser/xparse_type.c` | `type X = { ... }` 解析时设置 kind 为 `XR_KIND_OBJECT` 而非 `XR_KIND_JSON` |

#### S2.3 Analyzer 层

| 文件 | 改动 |
|------|------|
| `src/frontend/analyzer/xanalyzer_visitor_expr.c` | `xa_visit_member_access()` — OBJECT 类型：已知字段返回 `T`（非 optional 不 nullable），未知字段编译错误 |
| `src/frontend/analyzer/xanalyzer_visitor_expr.c` | `xa_visit_index_get()` — OBJECT 类型：不允许下标访问（编译错误） |
| `src/frontend/analyzer/xanalyzer_visitor_decl.c` | `xa_infer_return_json_type()` — 推断的对象字面量类型改为 OBJECT |

#### S2.4 Codegen 层

| 文件 | 改动 |
|------|------|
| `src/frontend/codegen/xexpr_object.c` | OBJECT 类型的字面量编译（运行时仍生成 XrJson，但类型系统层面区分） |
| `src/frontend/codegen/xstmt_assignment.c` | OBJECT 类型的赋值检查（禁止扩展字段） |
| `src/frontend/codegen/xstmt_typed.c` | OBJECT 类型声明 |
| `src/frontend/codegen/xexpr_collection.c` | OBJECT 类型的 index 访问处理 |

#### S2.5 运行时布局（不变）

**关键决策：运行时不需要新增对象类型。** `XR_KIND_OBJECT` 在运行时仍用 `XrJson` + Shape 表示。区分只在编译期类型系统中。VM 对 OBJECT 的操作和对 Json 的操作走同一套 opcode（`OP_GETPROP`, `OP_JSON_GET` 等）。

#### S2.6 字段访问规则

| 类型 | `obj.name`（name 存在） | `obj.name`（name 不存在） | `obj["name"]` | 运行时扩展 |
|------|------------------------|-------------------------|---------------|-----------|
| 结构类型（required field） | `T` | 编译错误 | 编译错误 | 编译错误 |
| 结构类型（optional field `name?`） | `T?` | 编译错误 | 编译错误 | 编译错误 |
| Json | `Json` | `Json` | `Json` | 允许 |

**测试**：
- 新增 `tests/regression/13_types/1311_object_type.xr` — 结构类型基本行为
- 新增 `tests/regression/13_types/1312_object_vs_json.xr` — 结构类型与 Json 的区别
- 新增 `tests/compile_errors/json/object_extension_error.xr` — 结构类型禁止扩展
- 全量 ctest + regression + AOT

**验收标准**：
- `type User = { name: string }` 的 `user.name` 返回 `string`（非 nullable）
- `user.foo = 1` 对未声明字段产生编译错误
- 所有现有 Json 测试不受影响

---

### S3：json 标准库合并到 Json 类型（1-2 周）

**目标**：删除 `import json` 依赖，所有 JSON 操作通过 `Json.xxx()` 或 `data.xxx()` 完成。

#### S3.1 新增 Json 静态方法

| 方法 | 签名 | 来源 |
|------|------|------|
| `Json.parse(str)` | `(string): Json` | 替代 `json.parse()` |
| `Json.stringify(val, indent?)` | `(Json, int?): string` | 替代 `json.stringify()` |
| `Json.tryParse(str)` | `(string): Json?` | 替代 `json.tryParse()`，简化返回值 |
| `Json.isValid(str)` | `(string): bool` | 替代 `json.isValid()` |

**实现方式**：

编译器识别 `Json.parse(...)` 等静态调用模式，生成对应 opcode 或 builtin call。

具体路径有两种选择：

- **方案 A**：在 analyzer/codegen 中 hardcode `Json` 类型的静态方法分发
  - 优点：简单直接
  - 缺点：增加 hardcode 路径

- **方案 B**：让 `Json` 成为一个 builtin class/module hybrid（类似 `Math`）
  - 优点：统一机制
  - 缺点：需要新的 builtin class 基础设施

建议 **方案 A 先行**，后续如果有更多 builtin 类型需要静态方法，再抽象为通用机制。

#### S3.2 新增 Json 实例方法

| 方法 | 签名 | 说明 |
|------|------|------|
| `data.keys()` | `(): Array<string>` | 替代 `json.keys(data)` |
| `data.values()` | `(): Array<Json>` | 替代 `json.values(data)` |
| `data.has(key)` | `(string): bool` | 新增 |
| `data.isNull()` | `(): bool` | 判断 Json 值是否为 null |
| `data.isInt()` | `(): bool` | 判断 Json 值是否为 int |
| `data.isFloat()` | `(): bool` | 判断 Json 值是否为 float |
| `data.isString()` | `(): bool` | 判断 Json 值是否为 string |
| `data.isBool()` | `(): bool` | 判断 Json 值是否为 bool |
| `data.isArray()` | `(): bool` | 判断 Json 值是否为 Array |
| `data.isObject()` | `(): bool` | 判断 Json 值是否为 JSON object |

**实现文件**：

| 文件 | 改动 |
|------|------|
| `src/runtime/object/xjson_methods.c` | 扩展 method table，新增 keys/values/has 方法体 |
| `src/runtime/object/xjson_methods.h` | 更新声明 |
| `src/runtime/symbol/xsymbol_table.h` | 确认 SYMBOL_KEYS / SYMBOL_VALUES / SYMBOL_HAS 已注册 |

#### S3.3 删除 json 模块

Xray 是新语言，没有历史包袱，不需要过渡期。直接删除 `json` 模块。

| 文件 | 改动 |
|------|------|
| `stdlib/json/json.c` | 删除 module loader `xr_load_module_json()`，移除用户 API 函数 |
| `stdlib/json/json.h` | C API `xr_json_parse_from_cstr` / `xr_json_stringify_to_cstr` 保留（供 C embedding 使用），迁移到 `src/runtime/object/xjson.c` |
| `src/module/xmodule_loader.c` | 从 module registry 中删除 `"json"` 条目 |

`import json` 直接产生编译错误：`module 'json' has been removed — use Json.parse(), Json.stringify() etc. directly`。

#### S3.4 删除 json.typeof

`json.typeof()` 与语言级 `typeof`/`typename` 语义不同（使用 JSON 规范名称），且使用场景极窄。不迁移，随模块一起废弃。

如果以后用户需要 JSON 规范类型名，可新增 `data.jsonType()` 方法，但优先级低。

**测试**：
- 修改 `tests/regression/10_stdlib/1000..1009_json*.xr` 从 `import json; json.parse(...)` 改为 `Json.parse(...)`
- 新增 `tests/regression/13_types/1313_json_static_methods.xr`
- 新增 `tests/regression/13_types/1314_json_instance_methods.xr`
- 新增 `tests/regression/13_types/1315_json_type_checks.xr` — isNull/isInt/isString/isArray/isObject 测试
- 确认 `import json` 产生编译错误

**验收标准**：
- `Json.parse(text)` 无需 import 即可使用
- `data.keys()` / `data.values()` / `data.has("key")` 作为实例方法工作
- `data.isNull()` / `data.isInt()` / `data.isString()` / `data.isArray()` / `data.isObject()` 实例方法工作
- `import json` 产生编译错误
- `json.typeof` 不迁移

---

### S4：收窄 unknown 传播（持续优化，2-4 周）

**目标**：减少 analyzer 中 `xr_type_new_unknown()` 的返回点，让更多操作获得精确类型，使编译器生成更多 typed opcodes。

#### S4.1 审计 unknown 返回点

当前约 124 处返回 `unknown`，按文件分布：

| 文件 | 次数 | 备注 |
|------|------|------|
| `xanalyzer_visitor_expr.c` | ~54 | 表达式类型推导 |
| `xanalyzer_visitor.c` | ~19 | 语句级推导 |
| `xanalyzer_builtins.c` | ~18 | builtin 方法返回类型 |
| `xanalyzer_visitor_decl.c` | ~10 | 声明级推导 |
| `xanalyzer.c` | ~7 | 顶层 |
| `xanalyzer_visitor_call.c` | ~6 | 调用推导 |
| 其他 | ~10 | |

#### S4.2 分类处理

- **可以补全的**：builtin 方法返回类型未注册（`xanalyzer_builtins.c` 的 18 处）→ 补全签名
- **错误恢复的**：analyzer 遇错后的 unknown 返回 → 保留，这些是正确行为
- **推导不完整的**：某些表达式路径缺少类型传播 → 逐个修复
- **泛型未实例化的**：泛型参数未绑定时返回 unknown → 保留为 unknown 但添加 warning

#### S4.3 添加 unknown 计数指标

在编译器 stats 中添加 `unknown_type_count` 指标，每次编译报告有多少变量/表达式仍是 unknown。作为质量度量长期跟踪。

**验收标准**：
- unknown 返回点从 ~124 减少到 <80（短期目标）
- 无新增回归

---

### S5：Json.decode\<T\>() 编译器自动生成类型化解码

**目标**：提供从 `Json` 到结构类型的类型安全转换，编译器自动生成 decode 函数，零样板代码。

Xray 是新语言，没有历史包袱，直接采用最佳方案。

**前置条件**：
- S2 完成（XR_KIND_OBJECT 就绪）

#### 设计

编译器在遇到 `Json.decode<T>(data)` 时，自动为结构类型 `T` 生成 decode 函数：

```xray
type User = { name: string, age: int, email?: string }

// 用户写：
let user: User? = Json.decode<User>(data)

// 编译器自动生成等价于：
// fn __User_decode(data: Json): User? {
//     if !data.isObject() { return null }
//     if !data.has("name") || !data.name.isString() { return null }
//     if !data.has("age") || !data.age.isInt() { return null }
//     let email: string? = null
//     if data.has("email") && data.email.isString() {
//         email = data.email as string
//     }
//     return { name: data.name as string, age: data.age as int, email: email }
// }
```

#### 规则

| 场景 | 行为 |
|------|------|
| required field 缺失或类型不匹配 | 返回 null |
| optional field 缺失 | 设为 null |
| optional field 存在但类型不匹配 | 返回 null |
| 多余字段 | 忽略（只取声明的字段） |
| 嵌套结构类型 | 递归生成 decode |
| 返回类型 | `T?`（decode 可能失败） |

#### 实现

| 文件 | 改动 |
|------|------|
| `src/frontend/analyzer/xanalyzer_visitor_call.c` | 识别 `Json.decode<T>()` 调用，验证 T 是结构类型 |
| `src/frontend/codegen/` | 新增 `xcodegen_json_decode.c`，为每个目标结构类型生成 decode 函数的 bytecode |
| `src/frontend/codegen/xexpr_call.c` | `Json.decode<T>(data)` 调用替换为生成的 decode 函数调用 |

#### 编译器生成策略

- 每个结构类型最多生成一个 decode 函数（去重）
- 生成的函数作为 hidden proto 挂在编译单元上
- 嵌套类型递归生成，循环引用通过前向声明处理

**测试**：
- 新增 `tests/regression/13_types/1316_json_decode.xr` — 基本 decode
- 新增 `tests/regression/13_types/1317_json_decode_nested.xr` — 嵌套结构 decode
- 新增 `tests/regression/13_types/1318_json_decode_optional.xr` — optional 字段 decode
- 新增 `tests/regression/13_types/1319_json_decode_error.xr` — decode 失败返回 null

**验收标准**：
- `Json.decode<User>(data)` 无需手写任何转换代码
- required field 缺失返回 null（不 crash）
- optional field 正确处理
- 嵌套结构类型递归 decode 工作

## 影响面汇总

### 涉及文件（按改动量排序）

| 文件 | 阶段 | 改动量 |
|------|------|--------|
| `src/frontend/analyzer/xanalyzer_visitor_expr.c` | S0, S1, S2 | 大 |
| `src/runtime/value/xtype.c` | S1, S2 | 大 |
| `src/runtime/value/xtype.h` | S1, S2 | 中 |
| `src/runtime/object/xjson_methods.c` | S3 | 中 |
| `src/frontend/parser/xparse_type.c` | S1, S2 | 中 |
| `src/frontend/codegen/xexpr_object.c` | S2 | 中 |
| `src/frontend/codegen/xstmt_assignment.c` | S2 | 小 |
| `src/frontend/codegen/xstmt_typed.c` | S2 | 小 |
| `src/frontend/codegen/xexpr_collection.c` | S2 | 小 |
| `src/frontend/analyzer/xanalyzer_visitor_decl.c` | S2 | 小 |
| `src/frontend/analyzer/xanalyzer_builtins.c` | S4 | 中 |
| `src/runtime/xisolate_internal.h` | S1 | 小 |
| `src/vm/xvm_dispatch_object.inc.c` | S1 | 小 |
| `stdlib/json/json.c` | S3 | 小（标记 deprecated） |

### 涉及测试

| 测试组 | 阶段 | 预期影响 |
|--------|------|----------|
| `tests/regression/13_types/1301..1310_json*.xr` | S0, S1, S2 | 需要更新期望输出 |
| `tests/regression/10_stdlib/1000..1009_json*.xr` | S3 | 从 `import json` 迁移到 `Json.xxx()` |
| `tests/compile_errors/json/` | S1, S2 | 可能需要更新 |
| `tests/jit/018..020_json*.xr` | S1 | 检查是否受 JsonValue 删除影响 |
| 新增：`1311_object_type.xr` | S2 | 新测试 |
| 新增：`1312_object_vs_json.xr` | S2 | 新测试 |
| 新增：`1313_json_static_methods.xr` | S3 | 新测试 |
| 新增：`1314_json_instance_methods.xr` | S3 | 新测试 |

### 不涉及的

- **VM 运行时分发逻辑**：`xvm_dispatch_*.inc.c` 中的 Json fast path 保持不变
- **XrJson 运行时对象**：`src/runtime/object/xjson.c` 不改
- **JIT / AOT 后端**：不受类型系统重构影响（它们读取的是 opcode，不是 XrTypeKind）
- **Shape / IC 机制**：保持不变
- **GC**：不涉及

## 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| S1 删 JsonValue 导致大量编译错误 | 用户代码中使用了 `JsonValue` 类型注解 | 直接删除，编译错误信息明确告知用 `Json` 替代。Xray 是新语言，不需要过渡期 |
| S2 XR_KIND_OBJECT 引入后 switch 遗漏 | 现有 `switch (type->kind)` 未处理 OBJECT | 编译器 `-Wswitch` 自动检测；搜索所有 `XR_KIND_JSON` case 逐个检查是否需要加 `XR_KIND_OBJECT` |
| S3 静态方法分发机制不成熟 | `Json.parse()` 的编译路径可能 hack | S3 先用 hardcode 路径，记录技术债，后续统一 |
| 回归测试覆盖不足 | 改动面广但测试只覆盖 happy path | 每个阶段必须全量 ctest + regression + AOT |

## 里程碑与验收

| 阶段 | 预估时间 | 验收命令 | 关键指标 |
|------|----------|----------|----------|
| S0 | 1-2 天 | `ctest && scripts/run_regression_tests.sh` | 字段访问返回类型统一 |
| S1 | 1 周 | `ctest && scripts/run_regression_tests.sh` | JsonValue 完全删除，grep 结果为 0 |
| S2 | 2-4 周 | `ctest && scripts/run_regression_tests.sh && tests/aot/run_aot_tests.sh` | `user.name` 返回 `string`（非 nullable） |
| S3 | 1-2 周 | `ctest && scripts/run_regression_tests.sh` | `Json.parse()` 无需 import |
| S4 | 持续 | `ctest` | unknown 计数 < 80 |
| S5 | 2-3 周 | `ctest && scripts/run_regression_tests.sh` | Json.decode\<T\>() 可用，嵌套 decode 工作 |

## 语言规范更新

S1 完成后需更新 `docs/rules/language-spec.md`：

1. 删除 `JsonValue` 的定义和所有引用
2. 更新 `Json` 的定义为"任意 JSON 值"
3. 更新 Json 字段访问的返回类型规则
4. 明确 `Json?` 是编译错误（Json 已包含 null）

S2 完成后需更新：

1. 明确 `type X = { ... }` 是"结构类型"，字段编译期固定
2. 明确结构类型的非 optional 字段不是 nullable
3. 添加结构类型与 Json 的对比说明

S3 完成后需更新：

1. 添加 `Json.parse()` / `Json.stringify()` 等静态方法文档
2. 添加 `data.keys()` / `data.values()` / `data.has()` 实例方法文档
3. 添加 `data.isNull()` / `data.isInt()` / `data.isString()` / `data.isArray()` / `data.isObject()` 实例方法文档
4. 删除 `import json` 的所有文档引用

S5 完成后需更新：

1. 添加 `Json.decode<T>()` 文档
2. 说明编译器自动生成 decode 函数的规则
