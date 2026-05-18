# Runtime/Class 重构计划（`src/runtime/class/`）

**开发原则**：
- ⚡ **不考虑向后兼容**，Xray 无外部用户，直接采用最佳设计
- ✅ 避免临时 workaround 与兼容层，每一步落到"长期最优"
- ✅ 每个阶段结束必须 **`scripts/run_regression_tests.sh` 全绿** 才能合并
- ✅ 每阶段 commit 粒度控制在 1~3 次（功能性子步骤可拆 commit，但不留半成品状态）
- ✅ C 代码注释与 git commit 全部英文；本文档中文

---

## 当前基线（2026-04-19）

- **目录规模**：46 个文件，约 280 KB 源码
- **最大文件**：`xclass_builder.c`（34 KB，~900 行），`xclass.c`（33 KB，1010 行）
- **子模块耦合**：
  - 类系统核心：`xclass*` / `xinstance*` / `xmethod*` / `xenum*`
  - 反射系统：9 个 `xreflect_*` 文件
  - 容器 builtins：7 个 `x{array,map,set,slice,string,stringbuilder,enum}_builtins.*`（约 40 KB）
- **已知问题**：见 `docs/engineering/audit_baseline.md` 之外单独列出，核心 4 类：
  1. 未清零分配 → `xr_class_new_single` 4 个字段野指针
  2. `primary_supers` depth ≥ 8 时顺序错乱
  3. 内存所有权混乱（`xr_strdup` + 不配对 `xr_free`）
  4. `xr_class_build_itable` / `xclass_builder_generate_vtable` 内层 O(n·m) 可降 O(n)

---

## 关键设计决策（已确认）

| # | 主题 | 选择 | 关键影响 |
|---|------|------|---------|
| D1 | class/method/field name 所有权 | symbol table intern 只读指针 | 删除所有 `xr_strdup`，比较改指针相等 |
| D2 | 容器 builtins 归属 | 迁移到 `src/runtime/object/` | `class/` 目录规模减半，CMake 文件组重写 |
| D3 | `primary_supers` 策略 | 8 层数组 + secondary supers hash（HotSpot 方案） | `instanceof` 深层类也 O(1) |
| D4 | 反射缓存并发策略 | 类创建时 eager 构建 | 去掉懒初始化 + 锁，内存稍增 |
| D5 | `xr_class_set_super` 存废 | **删除**，合并到 builder | VM `OP_INHERIT` 改走 builder finalize |
| D6 | `XrClass` 字段可见性 | 内部字段迁入 `xclass_internal.h` | 外部头导出 ≤ 25 个符号 |

---

## 阶段总览

| # | 阶段 | 文件范围 | 风险 | 预估工时 | 关键收益 |
|---|------|---------|------|---------|---------|
| P1 | 核心正确性修复 | `xclass.c`、`xenum.c`、`xinstance.c` | 低 | 1 d | 野指针、primary_supers、字符串比较 |
| P2 | 死代码 & 未实现 API 清理 | `xclass.h`、`xmethod.c`、`xclass_builder.h`、`xclass_system.c` | 低 | 0.5 d | 删 ~250 行 dead code、消除重复注册 |
| P3 | 名字所有权重构（intern 指针） | 全模块 + 符号表 | 中 | 1 d | 零拷贝、消除泄漏、`strcmp`→指针比较 |
| P4 | `xr_class_set_super` 合并到 builder | `xclass.c`、`xclass_builder.c`、`xvm.c`、`xclass_from_descriptor.c` | 中 | 1 d | 删除 ~200 行重复方法扁平化逻辑 |
| P5 | 性能路径优化 | `xclass.c`、`xclass_builder.c`、`xclass_lookup.c` | 低 | 0.5 d | itable/vtable 构建 O(n²)→O(n)、op_flag switch |
| P6 | 结构封装 + 头文件收敛 | `xclass.h` → `xclass.h` + `xclass_internal.h` | 中 | 1 d | 外部头 148 行 → ≤ 80 行 |
| P7 | 反射系统 eager 化 + 文件合并 | 9 个 `xreflect_*` | 中 | 1 d | 并发安全 + 合 3 个小文件 |
| P8 | 容器 builtins 迁出 | 7 个 `*_builtins.*` → `runtime/object/builtins/` | 低 | 0.5 d | `class/` 目录聚焦类系统 |
| P9 | `xclass.c` 拆分 | → `xclass_core.c` + `xclass_itable.c` + `xclass_abstract.c` + `xclass_operator.c` | 低 | 0.5 d | 单文件 ≤ 500 行 |
| P10 | 深度增强（secondary supers hash + 字段 offset 升位宽） | `xclass.h`、`xclass_core.c` | 中 | 0.5 d | 支持无限继承深度 + 大 struct |
| P11 | 内存与错误路径加固 | 全模块 | 低 | 0.5 d | 分配失败回滚、`xr_log_*` 统一 |

**总计**：约 7~8 个有效开发日。链式依赖：P1 → P2 → P3 → P4 → P5 → P6 → P7 → P8 → P9 → P10 → P11。

> P3（name intern）影响面广，完成前其余阶段能做但会接触相同代码，冲突成本高；建议严格串行到 P4。P5 之后并行度大幅提高。

---

## P1：核心正确性修复

### 动机
4 个潜在缺陷会在边界情况下触发崩溃或错误结果，必须先止血：

1. `xr_class_new_single` 用 `XR_ALLOCATE`（= `xr_malloc`，不清零），且遗漏 `field_default_values`、`struct_layout`、`reflect_cache`、`type_metadata` 4 个字段。后续 `xr_class_free` 会对野指针 `xr_free`。
2. @/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/class/xclass.c:201-208 在 depth ≥ 8 时反向填充 `primary_supers`，破坏 O(1) `instanceof`。
3. @/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/class/xenum.c:216-217 用指针相等比较字符串，未 intern 字符串会漏匹配。
4. @/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/class/xinstance.c:113-149 直接 `fprintf(stderr, ...)`，违反日志约定。

### 目标修改

**步骤 1.1** — `xr_class_new_single` 改用 `memset(cls, 0, sizeof(*cls))` 起手，后续字段赋值保留做文档用途
```c
static XrClass* xr_class_new_single(XrayIsolate *X, const char *name) {
    (void)X;
    XR_DCHECK(name != NULL, "Class name must not be NULL");
    XrClass *cls = XR_ALLOCATE(XrClass);
    if (!cls) return NULL;
    memset(cls, 0, sizeof(*cls));              // <-- 新增
    xr_gc_header_init_type(&cls->gc, XR_TCLASS);
    cls->name = xr_strdup(name);               // P3 会改掉
    cls->depth = 0;
    cls->primary_supers[0] = cls;
    cls->instance_size = sizeof(XrGCHeader);
    return cls;                                 // 删除所有 `= NULL` / `= 0` 冗余
}
```
**同时** `xr_class_builder_finalize` 已经有 memset，这一步统一两条路径。

**步骤 1.2** — 修复 `primary_supers` 深度 ≥ 8 的填充顺序（`xclass.c:201-208` + `xclass_builder.c:549-556` 两处都改）：
```c
// 正确做法：链式收集后按 [Object..子] 顺序填前 8 层
XrClass *chain[256];
int n = 0;
for (XrClass *c = subclass; c != NULL && n < 256; c = c->super) chain[n++] = c;
// chain[n-1] = Object, chain[0] = subclass
int base = n - 1;
for (int i = 0; i < 8 && base - i >= 0; i++) {
    subclass->primary_supers[i] = chain[base - i];
}
// 其余 (i >= 8) 由 P10 的 secondary supers hash 处理
```

**步骤 1.3** — `xr_enum_from_value` Tier 3 字符串比较改为 `xr_string_equal()`（或自己写长度+memcmp）。

**步骤 1.4** — `xinstance.c` 内全部 `fprintf(stderr, ...)` 换成 `xr_log_warning("class", ...)`。同步清理 `xclass.c:176` 等遗留。

### 验收
- 单元测试 `tests/regress/class_inheritance.xr` 增加 10 层继承用例（新增）
- AddressSanitizer 跑 `scripts/run_regression_tests.sh` 无 heap-use-after-free
- `ctest --output-on-failure` 全绿

### Commit
1. `fix(class): zero-init XrClass in xr_class_new_single`
2. `fix(class): correct primary_supers order at depth >= 8`
3. `fix(enum): use string compare instead of pointer equality`
4. `refactor(class): replace fprintf with xr_log_warning`

---

## P2：死代码 & 未实现 API 清理

### 清理清单

| 位置 | 内容 | 操作 |
|------|------|------|
| `xclass.h:279,281` | `xr_class_get_method` / `xr_class_get_static_field` 仅声明无实现无调用 | 删除声明 |
| `xmethod.c:21-73` | 5 个 `xr_method_from_*` 半成品构造器（未设 name/symbol/param_count） | 删除；builder 已直接填字段 |
| `xclass_builder.h:60-61` | `XrClassBuilder` 的 `XrGCHeader gc` 字段 | 删除（builder 临时对象，无需 GC 扫描） |
| `xclass.c:31-32` | `#define DEBUG_VTABLE 0` 无用宏 | 删除 |
| `xclass.c:153-165` | `xr_class_new` 里调 `xr_registry_register_class` 与 `xr_core_init` 的第二次调用重复 | 只保留 `xr_core_init` 的显式注册；`xr_class_new` 里移除 |
| `xclass.c:490-542` | `xr_class_lookup_method` fallback 路径（symbol_to_index 总存在，死路径） | 删除 fallback，保留 fast-path |
| `xreflect_method_stubs.c` 整文件 | 空 stub 文件 | 评估后删除或合并 |

### 验收
- `git grep` 确认无任何对已删函数的引用
- `scripts/run_regression_tests.sh` 全绿
- `cloc src/runtime/class` 行数下降 ~300 行

### Commit
1. `refactor(class): remove unimplemented xr_class_get_method/get_static_field`
2. `refactor(class): remove half-baked xr_method_from_* constructors`
3. `refactor(class): drop XrGCHeader from XrClassBuilder`
4. `refactor(class): remove duplicate registry_register_class in xr_class_new`
5. `refactor(class): remove dead fallback path in xr_class_lookup_method`

---

## P3：名字所有权重构（intern 指针）

### 动机
当前 `class->name`、`field->name`、`method->name` 全部走 `xr_strdup`，但 `xr_class_free` 只释放 `field_default_values`/`vtable` 等，三处 name 都没 `xr_free` → 内存泄漏。一次性改走 symbol table intern 既修复泄漏又提速比较（strcmp → 指针相等）。

### 目标设计

**规则**：类系统里所有 "名字" 字段均为 symbol table 中的只读指针（字符串与类同生命周期）。

```c
// src/runtime/symbol/xsymbol_table.h  （已存在，扩展 API）
XR_FUNC const char *xr_symbol_intern(XrSymbolTable *tab, const char *s, size_t len);
// 已有 xr_symbol_register_in_table 返回 SymbolId，需要新增返回 const char* 的版本
// 或复用：先 register 拿 id，再 xr_symbol_get_name_in_table(tab, id)
```

**改动点**：
1. `XrClass.name`、`XrFieldDescriptor.name`、`XrMethod` 所有 name 字段改为 `const char *`，由 isolate 的 symbol table 拥有
2. 所有 `xr_strdup(name)` 替换为 `xr_isolate_intern(X, name)`（新 helper，包装 symbol table 调用）
3. 所有 `strcmp(a->name, b->name)` 替换为 `a->name == b->name`（指针相等，命中 intern 保证）
4. `xr_class_free` 移除对 `name` / `fields[i].name` / `methods[i].name` 的考虑（isolate 统一回收）

### 落实顺序

1. 在 `xisolate_api.h` 暴露 `xr_isolate_intern_name(XrayIsolate *X, const char *s)`
2. 批量替换 `xclass.c`、`xclass_builder.c`、`xenum.c`、`xclass_descriptor.c` 中的 `xr_strdup`
3. 批量替换 `xclass.c:xr_class_lookup_field_by_name`、`xclass.c:xr_class_implements_interface` 中的 `strcmp` 为指针相等（注意：外部传入的名字也必须先 intern）
4. 删除 `xr_class_free` 中对 `cls->name` 的任何处理；同理不释放字段/方法名
5. `xr_class_descriptor_print` 等调试工具依然能打印（intern 指针也是合法 C 字符串）

### 验收
- AddressSanitizer + LeakSanitizer 零泄漏
- 基准测试 `bench/class_lookup.xr`：按名字查字段比当前快 ≥ 2×

### Commit
1. `feat(isolate): expose xr_isolate_intern_name helper`
2. `refactor(class): use symbol-interned names (no xr_strdup)`
3. `refactor(class): pointer equality for name compare (no strcmp)`

---

## P4：`xr_class_set_super` 合并到 builder

### 动机
`xr_class_set_super` 存在两个"姿态"：
1. **空类态**（`xr_class_new` 调用）：只设 super + 更新 primary_supers
2. **非空态**（VM `OP_INHERIT` 调用）：执行完整的方法扁平化、vtable 重建、field 重排

姿态 2 的逻辑与 `xr_class_builder_finalize` 的 flatten 过程**几乎完全重复**（@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/class/xclass.c:262-373 vs @/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/class/xclass_builder.c:682-777）。

### 目标设计

```
方案：彻底删除 xr_class_set_super，所有类都通过 builder 创建
  ├─ xr_class_new(X, name, super)  内部改为 wrap builder（空 builder + finalize）
  ├─ VM OP_INHERIT 改为编译期就决定 super，运行期不再动态设 super
  └─ 编译器侧：若源代码有继承关系，AST → descriptor 时就带 super_name
```

### 前置验证（需要完成）
- 检索 `OP_INHERIT` 产生条件：确认所有 AST 到字节码路径都能在**编译期**确定 super
- 若存在"late-binding"场景（如 cross-module forward reference），使用 descriptor 的 `super_global_index` 机制延迟解析，**解析后一次 finalize**

### 落实顺序
1. 梳理所有 `xr_class_set_super` 调用：
   - `xr_class_new` → 改为 builder wrap
   - `OP_INHERIT` → 改为在 class descriptor 里提前带 super，或走 builder
2. 删除 `xr_class_set_super` 实现与头声明（150+ 行）
3. 删除 `OP_INHERIT` 字节码（及相关 emit/jit 代码），或保留为 no-op 但标记 deprecated
4. 抽取公共 flatten helper（如果 builder 自身还需要）：
   ```c
   // src/runtime/class/xclass_internal.h
   void xr_class_flatten_parent_members(XrClass *cls, XrClass *super);
   ```

### 验收
- `git grep xr_class_set_super` 零结果
- VM 测试 `tests/regress/inheritance*.xr` 全绿
- `xclass.c` 减少 ~200 行

### Commit
1. `refactor(vm): resolve super at compile time, drop runtime binding`
2. `refactor(class): delete xr_class_set_super, unify via builder`

---

## P5：性能路径优化

### 5.1 `xr_class_build_itable` 降阶
@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/class/xclass.c:780-788：对每个接口方法线性扫类方法。改用 `method_symbol_to_index` 做 O(1) 查询。

```c
for (int j = 0; j < iface->method_count; j++) {
    XrMethod *iface_method = &iface->methods[j];
    XrMethod *impl = NULL;
    // 原：for (int k = 0; k < cls->method_count; k++) if (...symbol==...) impl = ...
    // 新：
    if (iface_method->symbol >= 0 && iface_method->symbol < cls->method_map_capacity) {
        int idx = cls->method_symbol_to_index[iface_method->symbol];
        if (idx >= 0 && idx < cls->method_count) impl = &cls->methods[idx];
    }
    entry->methods[j] = impl;
}
```

### 5.2 `xclass_builder_generate_vtable` `find_method_in_parent_vtable` 降阶
@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/class/xclass_builder.c:426-438：改用父类的 `method_symbol_to_index`（与 P4 后的 flatten helper 保持一致）。

### 5.3 `xr_symbol_to_op_flag` 改 switch
@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/class/xclass.c:37-65：22 个 if-else 改 `switch`，编译器会生成跳转表。

### 5.4 `xr_class_lookup_interface_method_by_symbol` 加 per-entry 符号映射
`XrItableEntry` 加 `int *method_symbol_to_index; int method_map_capacity;`，build_itable 时一次性生成。

```c
typedef struct XrItableEntry {
    struct XrClass *interface;
    XrMethod **methods;
    uint16_t method_count;
    int *method_symbol_to_index;       // <-- 新增
    int method_map_capacity;            // <-- 新增
} XrItableEntry;
```

### 5.5 `xr_class_lookup_field_by_name` 走符号表
```c
int xr_class_lookup_field_by_name(XrClass *cls, const char *name) {
    XrSymbolTable *tab = xr_isolate_get_symbol_table(/* … */);
    int sym = xr_symbol_lookup_in_table(tab, name);
    if (sym < 0) return -1;
    return xr_class_lookup_field(cls, sym);
}
```

### 验收
- `bench/class_lookup.xr` 字段/方法查找 p50 下降 ≥ 20%
- `bench/class_dispatch.xr` 接口方法分发 p50 下降 ≥ 30%

### Commit
1. `perf(class): build_itable uses symbol_to_index (O(n·m)→O(n))`
2. `perf(builder): parent vtable lookup via symbol map`
3. `perf(class): xr_symbol_to_op_flag as switch`
4. `perf(class): per-entry method symbol map in ITable`
5. `perf(class): lookup_field_by_name via symbol table`

---

## P6：结构封装 + 头文件收敛

### 目标
`xclass.h` 从 148 行、40+ 导出符号缩到 ≤ 80 行、≤ 25 个符号。所有 vtable / symbol map / itable 的内部字段迁到新文件 `xclass_internal.h`。

### 拆分方案

`xclass.h`（外部稳定 API）：
```c
// 对外暴露的 opaque 句柄
typedef struct XrClass XrClass;
typedef struct XrFieldDescriptor XrFieldDescriptor;
typedef struct XrItableEntry XrItableEntry;

// 仅导出：name 访问、创建 API、instanceof、反射桥接
const char *xr_class_get_name(XrClass *cls);
bool xr_class_instanceof(const XrClass *cls, const XrClass *target);
XrClass *xr_class_new(XrayIsolate *X, const char *name, XrClass *super);
// ……（约 20 个公共 API）
```

`xclass_internal.h`（仅 class/ 模块内部使用）：
```c
// XrClass 完整结构（所有字段）
struct XrClass { … };

// 内部 helper
void xr_class_flatten_parent_members(XrClass *cls, XrClass *super);
void xr_class_build_vtable(XrClass *cls);
// ……
```

### 同步改动
- `xclass_info.h`（分析器用的描述符）保持独立，位置不变
- `xclass_builder.h` 对外只保留 builder API，`XrClassBuilder` 结构也迁到 `xclass_builder_internal.h`
- 所有 `class/` 内部 `.c` 改 include `xclass_internal.h`
- 外部调用方（`vm/`, `jit/`, `api/`) 继续 include `xclass.h`，验证编译通过

### 验收
- `xclass.h` 导出符号数 ≤ 25
- `scripts/check_header_exports.sh xclass.h` 通过（若该脚本存在，否则手工 grep）
- 全量构建通过

### Commit
1. `refactor(class): extract XrClass internals to xclass_internal.h`
2. `refactor(class): extract XrClassBuilder internals to xclass_builder_internal.h`
3. `refactor(class): shrink xclass.h public surface (148→≤80 lines)`

---

## P7：反射系统 eager 化 + 文件合并

### 7.1 eager 构建反射缓存（D4）
当前 `xr_reflect_cache` 懒创建（`get_or_create_cache`），多协程并发访问同一类时可能双重创建。

**改动**：`xr_class_builder_finalize` 最后一步直接调 `xr_reflect_cache_create`，`klass->reflect_cache` 永远非 NULL。
```c
// xclass_builder.c: finalize 末尾
cls->reflect_cache = xr_reflect_cache_create(builder->isolate, cls);
if (!cls->reflect_cache) { /* 失败清理 */ }
```
然后：
- 删除 `xreflect_type.c` 的 `get_or_create_cache` helper
- `xr_type_getFields` / `xr_type_getMethods` 等直接 `klass->reflect_cache->field_wrappers[i]`
- `xr_class_free` 在最前面释放 `reflect_cache`

### 7.2 `type_metadata` 同策略
`xr_registry_register_class` 在 class 创建时立刻调用（P2 已经确认单点注册）。从而 `klass->type_metadata` 也永远非 NULL，消除 `xr_registry_find_type_by_class` 的懒赋值竞态。

### 7.3 文件合并
- `xreflect_constructor.c`（2KB） + `xreflect_method_stubs.c`（2.7KB） + `xreflect_field.c`（5.8KB） → `xreflect_members.c`
- `xreflect_api.c` + `xreflect_api.h`：可拆分 `xreflect_dispatch.c`（`xr_reflect_*` 静态方法）和 `xreflect_wrapper.c`（`xr_create_*_object`、metadata getter）— 但这是可选项，暂不做

### 验收
- 并发测试 `tests/concurrent/reflect_race.xr`（新增）：10 个协程同时反射同一个类 1000 次，无崩溃、无重复 cache
- 文件数从 46 减到 43

### Commit
1. `refactor(reflect): eager-build per-class cache at finalize`
2. `refactor(reflect): eager-register type metadata at class creation`
3. `refactor(reflect): merge constructor/method_stubs/field into members.c`

---

## P8：容器 builtins 迁出 `class/`

### 目标
将容器类型的内建方法从 `class/` 搬到 `runtime/object/builtins/`，让 `class/` 只负责"类系统本身"。

### 迁移清单

| 旧路径 | 新路径 |
|--------|--------|
| `class/xarray_builtins.{c,h}` | `object/builtins/xarray_builtins.{c,h}` |
| `class/xmap_builtins.{c,h}` | `object/builtins/xmap_builtins.{c,h}` |
| `class/xset_builtins.{c,h}` | `object/builtins/xset_builtins.{c,h}` |
| `class/xslice_builtins.{c,h}` | `object/builtins/xslice_builtins.{c,h}` |
| `class/xstring_builtins.{c,h}` | `object/builtins/xstring_builtins.{c,h}` |
| `class/xstringbuilder_builtins.{c,h}` | `object/builtins/xstringbuilder_builtins.{c,h}` |
| `class/xenum_builtins.{c,h}` | 保留在 `class/`（Enum 是类系统一部分） |
| `class/xclass_json_api.{c,h}` | `object/builtins/xjson_builtins.{c,h}`（同步改名） |

### 步骤
1. `git mv` 移动文件（保留 history）
2. 修正所有 `#include "xarray_builtins.h"` → `"../object/builtins/xarray_builtins.h"`（或通过 CMake include path 保持原样）
3. 更新根 `CMakeLists.txt` 的 `add_library` 源文件列表
4. 如果新增目录 `runtime/object/builtins/`，加入 CMake `target_include_directories`

### 验收
- `ls src/runtime/class/` 减少约 12 个文件
- 构建通过，所有 builtin 方法测试绿
- 依赖图检查：`object/builtins/` 不依赖 `class/` 以外的更高层（维持 L3 约束）

### Commit
1. `refactor(object): move container builtins out of class/`

---

## P9：`xclass.c` 拆分

### 目标
`xclass.c` 1010 行 → 4 个 ≤ 400 行文件。

### 拆分方案

| 新文件 | 内容 | 预估行数 |
|--------|------|---------|
| `xclass_core.c` | `xr_class_new_single`、`xr_class_new`、`xr_class_free`、`xr_class_lookup_*`、`xr_class_instanceof`（如果改 non-inline） | ~400 |
| `xclass_itable.c` | `xr_class_build_itable`、`xr_class_lookup_interface_method*`、`xr_class_implements_interface*`、`xr_class_verify_interface` | ~200 |
| `xclass_abstract.c` | `xr_class_mark_abstract`、`xr_class_add_abstract_method`、`xr_class_can_instantiate`、`xr_class_inherit_abstract_methods`、`xr_class_is_abstract_method` | ~100 |
| `xclass_operator.c` | `xr_symbol_to_op_flag`、`xr_class_compute_operator_flags`、op 相关 helper | ~100 |

### 验收
- 每个新文件 ≤ 500 行
- 所有原测试绿

### Commit
1. `refactor(class): split xclass.c into core/itable/abstract/operator`

---

## P10：深度增强

### 10.1 secondary supers hash（D3）
在 `primary_supers[8]` 之外，为 depth ≥ 8 的继承层级提供 O(1) 查找。

```c
struct XrClass {
    …
    XrClass *primary_supers[8];
    uint8_t depth;
    // 新增：深度 ≥ 8 时构建
    XrClass **secondary_supers_hash;   // 开放寻址 hash
    uint16_t secondary_supers_capacity; // 必为 2 的幂
};

// instanceof 热路径（xclass.h inline）：
static inline bool xr_class_instanceof(const XrClass *cls, const XrClass *target) {
    if (cls == NULL || target == NULL) return false;
    if (cls == target) return true;
    if (target->depth < 8) {
        return cls->primary_supers[target->depth] == target;
    }
    if (!cls->secondary_supers_hash) return false;
    uint32_t mask = cls->secondary_supers_capacity - 1;
    uint32_t h = xr_ptr_hash(target) & mask;
    for (;;) {
        XrClass *slot = cls->secondary_supers_hash[h];
        if (slot == target) return true;
        if (slot == NULL) return false;
        h = (h + 1) & mask;
    }
}
```

构建时机：`xr_class_builder_finalize`，当 `depth >= 8` 时分配并填充祖先。

### 10.2 Field offset 升位宽
`XrFieldDescriptor.offset` 从 `uint16_t` 升为 `uint32_t`，解除 64KB 实例尺寸上限。评估 `XrFieldDescriptor` 的 cache 影响（从 8B+对齐变 12B+对齐 → 可能增大到 16B/24B；需测是否值得）。

**决策点**：如果 stride 增长带来基准回退 > 2%，回退此项。

### 10.3 `field_map_capacity` 自适应
当 `max_symbol > 2 × own_field_count`（稀疏），改用小 robin-hood hashmap 代替直接映射。保持 `xr_class_lookup_field(sym)` 接口不变，内部分支。

### 验收
- `tests/regress/deep_inheritance.xr`（10 层继承 + instanceof）全绿
- 大 struct 测试（`instance_size > 64KB`）通过
- 微基准：hashmap 模式下查找 p50 比当前 ≤ 30% 慢（可接受）

### Commit
1. `feat(class): secondary supers hash for deep inheritance`
2. `feat(class): uint32 field offset for large structs`
3. `perf(class): adaptive field symbol map (array or hashmap)`

---

## P11：内存与错误路径加固

### 11.1 `xr_class_build_itable` 失败清理
@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/class/xclass.c:775 失败直接返回，前面已分配的 `entries` 泄漏。改为 `goto cleanup` 集中释放。

### 11.2 builder finalize 失败回滚
`xr_class_builder_finalize` 中若 vtable / itable / reflect_cache 任一失败，已分配的其他部分需释放（当前有部分泄漏）。

### 11.3 `xr_reflect_cache_create` 改 goto cleanup
@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/class/xreflect_cache.c:41-96 逐级 `xr_free` 很长且易漏。改：
```c
XrReflectCache *cache = XR_ALLOCATE(XrReflectCache);
if (!cache) return NULL;
memset(cache, 0, sizeof(*cache));
// …分配各项…
if (!field_wrappers) goto fail;
// …
return cache;
fail:
    xr_reflect_cache_free(cache);   // 统一清理
    return NULL;
```

### 11.4 `FieldWrapper` / `MethodWrapper` 清零
所有 wrapper 分配后 `memset(wrapper, 0, sizeof(*wrapper))`（与 P1 保持一致）。

### 11.5 日志统一
全模块搜索 `fprintf(stderr,` 替换为 `xr_log_error` / `xr_log_warning`。保留 `xr_class_print` 这种调试打印用的 `printf`。

### 验收
- 模糊测试 / ASAN `scripts/run_regression_tests.sh --asan` 无泄漏
- `git grep "fprintf(stderr" src/runtime/class` 仅剩调试打印

### Commit
1. `fix(class): cleanup partial allocations on build_itable failure`
2. `fix(class): rollback partial state on builder finalize failure`
3. `refactor(reflect): unified error cleanup via goto`
4. `refactor(class): unify error logging via xr_log_*`

---

## 风险管理

| 阶段 | 风险 | 缓解 |
|------|------|------|
| P3 | intern 化后某处漏改，出现双份字符串，指针比较失败 | 增加断言 `XR_DCHECK(name == xr_isolate_intern_name(X, name))`，ASAN 兜底 |
| P4 | 删除 `OP_INHERIT` 后，若分析器仍生成此字节码，VM panic | 先在 VM 改为 no-op 观察一个回归周期，再彻底删字节码 |
| P6 | 外部 `#include "xclass.h"` 打破（如 `api/`、`jit/`） | CI 前置：构建 `--target all` 而非只 `app/cli` |
| P7 | eager 构建使启动慢 | 用 `bench/startup.xr` 测量，若启动 > 5% 回退到懒创建 + atomic CAS |
| P8 | 容器 builtins 移动后依赖层级违反 | 完成后跑 `docs/rules/architecture.md` 依赖检查脚本 |
| P10.2 | `uint32_t offset` 增大 `XrFieldDescriptor` cache footprint | 基准测试守门 |

### 回滚策略
- 每阶段独立分支 `refactor/class-p{N}`；通过回归测试后 merge 到 `refactor/class-main`
- `refactor/class-main` 每完成 2~3 阶段 rebase 合并到 `main`
- 任何单阶段在基准回退超阈值时，立即 revert 该阶段，不影响后续阶段推进（除链式依赖）

---

## 验收总标准

合并到 `main` 前必须满足：

- [ ] `scripts/run_regression_tests.sh` 全绿
- [ ] `scripts/run_regression_tests.sh --asan` 无 leak / use-after-free
- [ ] `cloc src/runtime/class` 总行数下降 ≥ 20%（预期从 ~10000 行到 ~8000 行）
- [ ] `xclass.h` 导出符号 ≤ 25
- [ ] 目录文件数 ≤ 35（当前 46）
- [ ] 最大单文件 ≤ 800 行
- [ ] `bench/class_*.xr` p50 无回退，关键路径提速 ≥ 20%
- [ ] `docs/rules/architecture.md` 依赖层级检查通过

---

## 后续（Out of Scope，本计划不包含）

- JIT 侧对 class 的特化（`jit_known_limitations.md` 有记录）
- Generic / 参数化类型（`type_system_v2`）
- 序列化与热重载
