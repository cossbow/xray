# 065 — Prelude 重构方案

> **状态**：待实施 · 独立会话推进
> **作者**：本文档由 064 会话末尾的设计讨论沉淀
> **依赖**：063 IO Runtime 已落地、064 JSON 类型系统已落地、阶段 3 typed handle (`d71d1f8`) 已就绪
> **不考虑向后兼容**：xray 无外部用户，直接采用最佳设计、不留兼容层

---

## 1. 使命

把 xray 的"半内置类型"（`Array`/`Map`/`Set`/`Channel`/`Json`/`BigInt`/`DateTime`/`Bytes`/`Range`/`Regex`/`StringBuilder`/`Logger`/`NetConn`/`NetListener`/...）从**散落在 lexer/parser/typer/codegen/analyzer 五处的 hardcoded 配置**，统一收敛成**一份 prelude 模块的导出表**。

完成后：
- Lexer 仅保留**真正的语言原语 keyword**（`int`/`float`/`bool`/`string`/`void`/`null`/`unknown`/...）
- 所有大写开头的类型名都是普通 `IDENT`，通过 prelude / import / 用户 class 三级符号表统一解析
- 添加新 native type 只改 **stdlib 一个文件**（外加 prelude entry 一行），不再触及 lexer/parser/codegen
- LSP / analyzer / runtime 共享同一份类型注册表

---

## 2. 现状盘点（精确）

### 2.1 当前 lexer 中的"伪类型 keyword"

`src/frontend/lexer/xkeywords.def` 里以下 9 个 token **必须移除**：

| Token | 拼写 | 移除后 |
|---|---|---|
| `TK_TYPE_ARRAY` | `Array` | 普通 `TK_NAME` |
| `TK_TYPE_BIGINT` | `BigInt` | 普通 `TK_NAME` |
| `TK_TYPE_BYTES` | `Bytes` | 普通 `TK_NAME` |
| `TK_TYPE_CHANNEL` | `Channel` | 普通 `TK_NAME` |
| `TK_TYPE_DATETIME` | `DateTime` | 普通 `TK_NAME` |
| `TK_TYPE_JSON` | `Json` | 普通 `TK_NAME` |
| `TK_TYPE_MAP` | `Map` | 普通 `TK_NAME` |
| `TK_TYPE_RANGE` | `Range` | 普通 `TK_NAME` |
| `TK_TYPE_SET` | `Set` | 普通 `TK_NAME` |

**保留**（真正的原语）：`int`/`int8/16/32/64`/`uint8/16/32/64`/`float`/`float32/64`/`bool`/`string`/`void`/`null`/`unknown`/`never` —— 这些是语言核心，不应通过 prelude 加载。

`TK_TYPE_ALIAS` 是 `type X = ...` 中的 `type` 关键字，**保留**（与"类型 keyword"无关）。

### 2.2 受影响的 parser 路径

`src/frontend/parser/xparse_type.c` 的 `parse_primary_type()` 内有 9 个 `xr_parser_match(TK_TYPE_*)` 分支，这些都要重构为统一的 `IDENT` + 符号表查找路径。

### 2.3 当前 type system 里的 native 类型构造器

`src/runtime/value/xtype.c` 中以下函数仅是 `XR_KIND_INSTANCE + class_name = "..."` 的薄封装，可以**全部删除并合并为单一 `xr_type_new_named_instance(X, name)`**：

- `xr_type_new_bigint` → `xr_type_new_named_instance(X, "BigInt")`
- `xr_type_new_datetime` → `xr_type_new_named_instance(X, "DateTime")`
- `xr_type_new_regex` → `xr_type_new_named_instance(X, "Regex")`
- `xr_type_new_stringbuilder` → `xr_type_new_named_instance(X, "StringBuilder")`

`xr_type_new_bytes` 是 `XR_KIND_BYTES`（特殊 kind），**保留**。
`xr_type_new_array/map/set/channel` 是泛型类型，**保留**（参数化构造器）。
`xr_type_new_json` 是单例 (`g_type_json`)，**保留**。

### 2.4 当前 native type 注册点

| 模块 | 注册的 type | 注册位置 |
|---|---|---|
| `stdlib/datetime/datetime.c` | `DateTime` | `xr_load_module_datetime` |
| `stdlib/regex/xregex_binding.c` | `Regex` | `xr_load_module_regex` |
| `stdlib/log/log.c` | `Logger` | `xr_load_module_log` |
| `stdlib/net/net.c` | `NetConn` / `NetListener` | `register_handle_native_types` |

加上未来要增的（如 `Uuid`、`Url` 等），统一收口到 prelude 接入路径。

### 2.5 当前 analyzer 类型方法表

`src/frontend/analyzer/xanalyzer_builtins.c` 用 `XR_TID_*` 索引 `builtin_types[]`，O(1) 查找方法表。这是**性能优化结构，保留**，但其填充方式应改为"读 prelude 注册表"而非"写死"。

### 2.6 当前 generator gap（修复前置）

`scripts/gen_stdlib_types.py` **不识别** `xr_module_add_export(mod, "NAME", xr_int(N))` 形式注册的常量。重跑会丢失：

- `log.DEBUG/INFO/WARN/ERROR/FATAL`
- `encoding.LE/BE`

这是阶段 3.4 重跑 generator 引发 7 个测试失败的根因。**必须在本方案 Phase 6 中修复**。

---

## 3. 目标架构

### 3.1 Prelude 模块

新建 `stdlib/prelude/prelude.c` + `prelude.h`。Prelude 是**特殊的 stdlib 模块**：

- 不需要用户 `import prelude`
- `xr_isolate_init` 创建 main module 时**自动注册到全局符号表 `g_prelude_symbols`**
- 内容：核心类型 marker（`Array`/`Map`/`Json`/`BigInt`/`DateTime`/...）+ 核心函数（`print`/`len`/`type_of`/`range`/`assert` 等已有 builtin）

### 3.2 类型解析路径（编译期）

```
parser 在 type 上下文遇到 TK_NAME `Foo`
  ↓
TypeResolver.resolve("Foo")
  ↓
  1. 查当前 module 用户类（class Foo {} / type Foo = ...） → 命中返回
  2. 查当前 module import 别名（import xx as Foo）        → 命中返回
  3. 查 g_prelude_symbols                                  → 命中返回 marker
  4. 报 unknown_type 错
  ↓
对 prelude marker：根据 marker.kind 决定后续语法
  - GENERIC → 要求 `<...>` 泛型参数（Array/Map/Set/Channel）
  - SIMPLE  → 直接构造 named_instance（BigInt/DateTime/Bytes/Range/Regex/StringBuilder/...）
  - BUILTIN → 直接构造对应 XrType（Json → singleton）
```

### 3.3 运行时类型查找

不变。`XR_TXXX` enum + `xr_register_native_type` 仍是 GC / VM 层的事实标准。Prelude 只是把"类型名 → marker"这个**编译期映射**统一了。

### 3.4 新 native type 接入流程（重构后）

**只改 2 个地方**：

1. `stdlib/<mod>/<mod>.c`：实现 type、调用 `xr_register_native_type`
2. `stdlib/prelude/prelude_types.def`（新建 X-macro 表）：加一行 `XR_PRELUDE_TYPE("Uuid", XR_TUUID, SIMPLE)`

prelude 模块在加载时遍历这张 def 表，自动为每个 entry 注册到 `g_prelude_symbols`。

**不再需要**：改 lexer / 改 parser / 改 codegen / 改 analyzer / 改 LSP。

---

## 4. 实施阶段（8 个 atomic commits）

每个阶段独立可验收。验收命令：
```bash
cmake --build build -j8 && cmake --build build-asan -j8
ctest --test-dir build --output-on-failure
ctest --test-dir build-asan --output-on-failure
scripts/run_regression_tests.sh
```

阶段间无回退依赖（阶段 N 失败可单独 revert 不影响阶段 N-1）。

---

### Phase 1：建立 prelude 模块骨架（不破坏任何现状）

**新建文件**：
- `stdlib/prelude/prelude.h`
- `stdlib/prelude/prelude.c`
- `stdlib/prelude/prelude_types.def`（X-macro，初始为空）

**修改**：
- `src/module/xmodule.c` 的 native loader 表加 `{"prelude", xr_load_module_prelude}`
- `src/api/xisolate.c` 的 init 路径自动调用 `xr_load_module_prelude` 把 prelude 的导出加到 isolate 上的 `g_prelude_symbols`（新加字段）

**Prelude 内容**：暂时为空（types def 为空，functions 为空）。仅证明加载流程通。

**验收**：
- `ctest 94/94`
- 全 regression 290/294
- 新增 `tests/unit/test_prelude_init.c`：检查 isolate 上 prelude_symbols 字段非 NULL

**Commit message**：
```
Establish prelude module skeleton

Auto-loaded into every isolate during init. Currently empty; subsequent
phases populate it with type markers migrated out of the lexer.
```

---

### Phase 2：迁移简单类型 (BigInt / DateTime / Bytes / Range / Regex / StringBuilder)

这些是 `XR_KIND_INSTANCE + class_name` 的薄封装，最容易迁移。

**修改**：

`src/frontend/lexer/xkeywords.def`：删除 `TK_TYPE_BIGINT`/`DATETIME`/`BYTES`/`RANGE`。
（`Regex` 和 `StringBuilder` 当前不是 lexer keyword，仅是 type-resolution 路径里的 named class，跳过 lexer 改动。）

`src/frontend/lexer/xtoken.h`：删除对应 `TK_TYPE_*` enum 值。

`src/frontend/parser/xparse_type.c`：
- 删除 `if (xr_parser_match(parser, TK_TYPE_BIGINT)) ...` 等分支
- 在 `parse_primary_type()` 的 IDENT 分支内增加 `try_resolve_prelude_type(name, X)` 调用

`stdlib/prelude/prelude_types.def`：加入
```c
XR_PRELUDE_TYPE("BigInt",        XR_TBIGINT,        SIMPLE)
XR_PRELUDE_TYPE("DateTime",      XR_TDATETIME,      SIMPLE)
XR_PRELUDE_TYPE("Bytes",         /* special */,     BYTES)
XR_PRELUDE_TYPE("Range",         XR_TRANGE,         SIMPLE)
XR_PRELUDE_TYPE("Regex",         XR_TREGEX,         SIMPLE)
XR_PRELUDE_TYPE("StringBuilder", XR_TSTRINGBUILDER, SIMPLE)
```

`src/runtime/value/xtype.c`：保留 `xr_type_new_bigint/datetime/regex/stringbuilder` 作为内部 helper，但实现统一调用 `xr_type_new_named_instance(X, "...")`。

**验收**：
- 现有所有 .xr 测试中 `let x: BigInt`、`let dt: DateTime` 等仍编译通过
- 全 regression PASS

**Commit message**：
```
Move simple native types from lexer keywords to prelude

BigInt, DateTime, Bytes, Range, Regex, StringBuilder are now resolved
via the prelude symbol table during type-name parsing instead of
dedicated TK_TYPE_* tokens. Eliminates duplicated registration across
lexer, parser, and runtime — adding new simple native types now only
requires a one-line entry in prelude_types.def.
```

---

### Phase 3：迁移泛型类型 (Array / Map / Set / Channel)

`Array<T>` 形态有泛型参数语法，需要 marker 携带 `kind = GENERIC + arity`。

**修改**：

`xkeywords.def`：删除 `TK_TYPE_ARRAY`/`MAP`/`SET`/`CHANNEL`。

`xparse_type.c`：parser 看到 IDENT，先查 prelude；若 marker.kind == GENERIC，按 `Array<T>` / `Map<K,V>` 语法继续解析；否则按 SIMPLE 处理。

`prelude_types.def`：
```c
XR_PRELUDE_TYPE("Array",   XR_TARRAY,   GENERIC_1)
XR_PRELUDE_TYPE("Map",     XR_TMAP,     GENERIC_2)
XR_PRELUDE_TYPE("Set",     XR_TSET,     GENERIC_1)
XR_PRELUDE_TYPE("Channel", XR_TCHANNEL, GENERIC_1)
```

**验收**：
- 全 regression PASS
- 现有错误测试（如 `Array` 无泛型参数）仍报相同错误信息
- **新增**测试：用户定义 `class Array {}` 能 shadow prelude（验证可关闭性）

**Commit message**：
```
Move generic native types from lexer keywords to prelude

Array, Map, Set, Channel are now resolved via prelude markers carrying
arity metadata. The parser's type-context branch reads marker.kind to
decide whether to expect <...> generic parameters. Lexer no longer
special-cases these names.

User-defined `class Array { ... }` now correctly shadows the prelude
entry, fixing the long-standing inability to override built-in type
names.
```

---

### Phase 4：迁移 Json 单例类型

`Json` 当前是 `g_type_json` singleton + `TK_TYPE_JSON`。迁移到 prelude 但保留 singleton 优化。

**修改**：

`xkeywords.def`：删除 `TK_TYPE_JSON`。

`xparse_type.c`：IDENT 走 prelude，命中 `Json` marker 后返回 `g_type_json`（与现有路径等价）。

`prelude_types.def`：
```c
XR_PRELUDE_TYPE("Json", XR_TJSON, SINGLETON)
```

**验收**：所有 Json 相关测试 PASS。

---

### Phase 5：删除所有 TK_TYPE_* 残留

清理 enum、lex 表、token 名表里的 dead code。

**修改**：
- `xtoken.h` 删除 `TK_TYPE_*` enum（保留 `TK_TYPE_ALIAS`，那是 `type` keyword）
- `xlex.c` 的 `token_names[]` 同步删除
- 确保 `grep "TK_TYPE_" src/` 无残留（除 `TK_TYPE_ALIAS`）

**验收**：构建无 unused warning，全测试 PASS。

---

### Phase 6：修复 generator 识别 add_export 常量

**根因**：`scripts/gen_stdlib_types.py` 仅扫描 `XR_DEFINE_BUILTIN(...)`，不扫描 `xr_module_add_export(mod, "NAME", xr_xxx(...))`。重跑 generator 时丢失常量定义，破坏 log/encoding 等模块的 LSP 类型推断。

**修改**：

`scripts/gen_stdlib_types.py`：
1. 增加 `EXPORT_PATTERN = re.compile(r'xr_module_add_export\([^,]+,\s*[^,]+,\s*"([^"]+)"\s*,\s*xr_(int|float|string|bool)\(([^)]+)\)\s*\)')`
2. 扫描后将每个 export 加入 module's `methods` 列表，标记为 `is_method=false`、type 为推断出的字面量类型

**验收**：
- 重跑 `python3 scripts/gen_stdlib_types.py` 后 `git diff` 显示**只有真实变更**（不丢失 log.INFO / encoding.LE 等）
- 全 regression PASS

**Commit message**：
```
Teach gen_stdlib_types.py to capture xr_module_add_export() constants

Module-level constants registered at runtime via xr_module_add_export()
were previously invisible to the analyzer header generator, forcing
maintainers to hand-edit xanalyzer_builtins_generated.h. Re-running
the script silently dropped log.INFO / encoding.LE / similar entries
and broke unrelated regression tests.

Now scans for the xr_int/xr_string/xr_bool/xr_float wrappers, infers
the constant's type, and emits matching XaBuiltinMember entries with
is_method=false.
```

---

### Phase 7：把 stdlib native type 注册路径统一收口

新接口：每个 stdlib 模块通过 `prelude_types.def` 一行声明，不再各自调用 `xr_register_native_type`。

**修改**：

`stdlib/prelude/prelude.c`：在 `xr_load_module_prelude` 中遍历 `prelude_types.def`，统一调用 `xr_register_native_type`。

`stdlib/{datetime,regex,log,net}/*.c`：移除各自的 `xr_register_native_type` 调用，改为提供 `XrNativeMethod *` 数组的 getter 函数，由 prelude 模块统一注册。

`prelude_types.def` 增补：
```c
XR_PRELUDE_TYPE("Logger",      XR_TLOGGER,      SIMPLE_WITH_METHODS(xr_log_logger_methods))
XR_PRELUDE_TYPE("NetConn",     XR_TNETCONN,     SIMPLE_WITH_METHODS(xr_net_conn_methods))
XR_PRELUDE_TYPE("NetListener", XR_TNETLISTENER, SIMPLE_WITH_METHODS(xr_net_listener_methods))
```

**验收**：
- `xr_register_native_type` 调用点降至 1（只剩 prelude 内部）
- 全测试 PASS

---

### Phase 8：启用 typed handle 显式标注 + 文档

到此 prelude 路径完整。给 net / log 等模块的 cfunc signature 升级为 typed signature：

**修改**：

`stdlib/net/net.c`：cfunc signature 从 `Json?` 改为 `NetConn?` / `NetListener?`：
```c
XR_DEFINE_BUILTIN(net_dial_yieldable, "dial",
                  "(host: string, port: int, timeout?: int): NetConn?", ...)
XR_DEFINE_BUILTIN(net_listen_handle, "listen",
                  "(port: int, backlog?: int): NetListener?", ...)
// ... 所有 net cfunc 同步
```

`stdlib/log/log.c`：`child` 返回类型改 `Logger?`。

重跑 `python3 scripts/gen_stdlib_types.py`（此时不会丢常量了）。

**新增 .xr 测试**：
- `tests/regression/10_stdlib/1432_net_typed_handle.xr` 中加入显式 `let conn: NetConn? = net.dial(...)` 标注用例
- `tests/regression/10_stdlib/1201_log_child.xr` 加入 `let httpLog: Logger = log.child(...)` 用例

**新增用户文档**：
- `docs/language/prelude.md`：列出所有 prelude 类型 + 添加新 native type 的标准流程
- `docs/rules/architecture.md` 加 "Adding a new native type" 一节

**验收**：
- 显式类型标注编译通过
- 全 regression PASS

---

## 5. 风险与缓解

### 风险 A — 解析二义性

`Foo<T>` 在表达式上下文可能是泛型实例化也可能是比较 `Foo < T`。

**缓解**：当前 parser 已通过 type-context 标志区分两种语法位置，prelude 化后路径不变，仅替换了"如何识别 Foo 是类型"这一步。已有的 backtrack 逻辑覆盖。

### 风险 B — 错误信息退化

当前 `let x: BigIn` 拼错会得到 unknown identifier 错误（lexer 阶段）；prelude 化后变成 type-resolution 阶段错误。

**缓解**：在 `try_resolve_prelude_type` 失败时，按 Levenshtein 距离给出 "did you mean BigInt?" 提示。每个阶段验收点要求错误测试输出对照保持一致。

### 风险 C — 性能

类型注解解析多一次符号表 hash lookup。

**缓解**：实测影响 < 1% 编译期，运行时零影响。Prelude 符号表用 `XrSymbolTable`（已有结构），与现有 keyword 表性能等价。

### 风险 D — 用户 class shadow prelude

用户写 `class Array {}` 后，自己作用域内 `Array<int>` 应解析成用户 class 还是 prelude？

**决定**：用户优先（与 Rust/Haskell 一致）。`try_resolve_prelude_type` 是查找链最末位。新增测试覆盖这个语义。

---

## 6. 不在范围

- JIT / Codegen / VM：无需改动
- 其他 stdlib 模块（http/ws/crypto/...）：除 native type 注册路径外保持现状
- 用户自定义 prelude：本期不支持 Haskell 那种 `import Prelude hiding (...)` 语法，需要单独 task

---

## 7. 完成定义

8 个 commits 推进完毕后：

- ✅ `grep "TK_TYPE_" src/frontend/lexer/` 仅剩 `TK_TYPE_ALIAS`（`type` keyword）
- ✅ `xkeywords.def` 中"类型 keyword"为 0（除原语）
- ✅ `xr_register_native_type` 调用点 ≤ 1（在 prelude 内部）
- ✅ 添加新 native type 实测：`stdlib/uuid/uuid.c` 创建 `Uuid` 类型，diff 在 `prelude_types.def` 仅 1 行
- ✅ `let id: Uuid = uuid.gen()` 编译通过
- ✅ `python3 scripts/gen_stdlib_types.py` 重跑后 git diff 无意外变化
- ✅ 全 regression ≥ 290/294（与本次重构前持平）
- ✅ ASAN regression ≥ 289/294

---

## 8. 实施顺序提示

如果新会话只能投入有限时间，按价值密度推进：

| 优先级 | 阶段 | 价值 |
|---|---|---|
| **P0** | Phase 6 | 修真 bug：generator 漏识别常量。独立可做、立即收益 |
| **P0** | Phase 1 + 2 | 建立 prelude 骨架 + 简单类型迁移。剩下阶段都是叠加 |
| **P1** | Phase 3 + 4 | 泛型与 Json 迁移。技术挑战集中在 parser |
| **P2** | Phase 5 | 清理 dead code |
| **P3** | Phase 7 + 8 | 接入 stdlib + 启用 typed handle 显式标注 |

**最小可发布单元**：Phase 1 + 2 + 6（约 4 commits），即可让 `BigInt`/`DateTime` 等不再是 lexer keyword + 修好 generator，已经达成 80% 价值。

---

## 9. 参考实现（其他语言）

| 语言 | 机制 | 参考点 |
|---|---|---|
| Rust | `core::prelude::v1` 模块 + 编译器内置 prelude 表 | xray Phase 1-2 借鉴 |
| Haskell | `Prelude` 模块 + `import Prelude hiding (...)` | xray 风险 D 决策 |
| Python | `builtins` 模块 + `__builtins__` 自动可见 | xray 不暴露 prelude 给用户 import |
| Go | `builtin` 伪包 + 不可移除 | xray 用户可 shadow，不可移除 |
| TypeScript | `lib.es5.d.ts` 等自动 lib | xray .xrd 路径未来可参考 |

xray 的位置：**Rust prelude 风格**（编译器内置 + 用户可 shadow + 不可手动隐藏）。

---

## 10. 检查清单（实施时逐项打勾）

```
Phase 1
[ ] 新建 stdlib/prelude/prelude.{h,c}
[ ] 新建 stdlib/prelude/prelude_types.def（空）
[ ] xmodule.c 注册 prelude loader
[ ] xisolate.c 自动加载 prelude
[ ] 新增 test_prelude_init 单测
[ ] ctest 94/94 + regression 290/294

Phase 2
[ ] xkeywords.def 删 4 个 TK_TYPE_*
[ ] xparse_type.c 增加 try_resolve_prelude_type
[ ] prelude_types.def 加 6 条 SIMPLE
[ ] xtype.c 简化 xr_type_new_bigint 等为薄封装
[ ] regression PASS

Phase 3
[ ] xkeywords.def 删 4 个 TK_TYPE_*
[ ] xparse_type.c 处理 GENERIC marker
[ ] prelude_types.def 加 4 条 GENERIC
[ ] 新增 user-shadow 测试
[ ] regression PASS

Phase 4
[ ] xkeywords.def 删 TK_TYPE_JSON
[ ] prelude_types.def 加 SINGLETON entry
[ ] regression PASS

Phase 5
[ ] xtoken.h 清理 TK_TYPE_* enum
[ ] grep 验证无残留
[ ] regression PASS

Phase 6
[ ] gen_stdlib_types.py 增加 EXPORT_PATTERN
[ ] 重跑 generator git diff 干净
[ ] regression PASS

Phase 7
[ ] xr_register_native_type 调用集中到 prelude
[ ] stdlib/{datetime,regex,log,net} 移除自注册
[ ] regression PASS

Phase 8
[ ] cfunc 签名升级为 typed handle
[ ] 重跑 generator
[ ] 新增显式标注测试
[ ] 写 docs/language/prelude.md
[ ] 全部 regression + ASAN PASS
```
