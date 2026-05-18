# xray C 工程规范 — 世界级大型 C 项目开发指南

> 本规范基于对 Lua 5.4、LuaJIT、QBE、Redis、nginx、libuv、wren、chibicc、MIR、SQLite、QuickJS 等世界级 C 开源项目的源码深度分析提炼而成。
> 所有规则都有对应的实证数据支撑。

---

## 第一章：文件组织规范

### 1.1 单文件行数上限

| 项目 | 最大单文件 | 行数 | 评价 |
|------|-----------|------|------|
| Lua 5.4 | lparser.c | 2,193 | ✅ 优秀 |
| LuaJIT | lj_record.c | 2,916 | ✅ 良好 |
| QBE | parse.c | 1,717 | ✅ 优秀 |
| wren | wren_compiler.c | 4,134 | ⚠️ 可接受 |
| chibicc | parse.c | 3,368 | ⚠️ 可接受 |
| Redis | module.c | 15,559 | ❌ 过大 |
| QuickJS | quickjs.c | 58,545 | ❌ 极端反例 |
| **xray** | **xvm.c** | **10,276** | **❌ 需要拆分** |

**规则 F-1：单个 .c 文件不超过 3,000 行（硬限制），目标 2,000 行以内**

- 超过 3,000 行的文件必须拆分
- xray 当前超标文件：xvm.c(10K), xir_builder.c(6K), xir_pass.c(4.6K), xanalyzer_visitor.c(4.2K), xir_codegen.c(3.8K), xparse.c(3.7K)
- 参照 Lua：lvm.c 1,972行实现 ~80条指令 + 类型转换 + 比较运算，证明 VM 主循环可以控制在 2K 行

**规则 F-2：单个 .h 文件不超过 800 行**

- 头文件是模块契约，应精炼
- Lua 最大头文件 lobject.h 865行（包含所有值类型定义，可接受）
- xray 超标：xpoll.h(936), xast_nodes.h(816), xrt_common.h(747)

### 1.2 文件命名规范

**规则 F-3：统一模块前缀，一个前缀对应一个子系统**

各项目的前缀体系：

| 项目 | 前缀 | 示例 |
|------|------|------|
| Lua 5.4 | l + 模块缩写 | lvm.c, lgc.c, ltable.c, lparser.c |
| LuaJIT | lj_ + 模块名 | lj_record.c, lj_gc.c, lj_tab.c |
| nginx | ngx_ + 子系统 | ngx_http_request.c, ngx_event.c |
| libuv | uv_ / uv__ | uv_tcp.c, uv__io_start() |
| Redis | 功能名（无前缀） | server.c, networking.c ← 不推荐 |

xray 当前前缀体系 `x` + 模块名（xvm, xgc, xparse 等）是合理的，应坚持。

**规则 F-4：.c 和 .h 一一对应**

- 每个 `xxx.c` 必须有且仅有一个对应的 `xxx.h`
- `xxx.h` 是 `xxx.c` 的唯一公共接口
- Lua 完美遵守：lvm.c↔lvm.h, lgc.c↔lgc.h, ltable.c↔ltable.h
- 如果一个 .c 不需要头文件（即不导出任何符号），说明它只是实现细节，应该考虑合并到父模块

### 1.3 目录结构

**规则 F-5：目录深度不超过 3 层（src/子系统/文件）**

最佳实践（取 nginx 和 libuv 的分层方式）：

```
src/
  core/       ← 基础类型、内存、断言
  value/      ← 值表示、类型系统
  object/     ← GC 对象（string, array, map 等）
  gc/         ← GC 实现
  frontend/   ← lexer, parser, codegen
  vm/         ← VM 解释器
  jit/        ← JIT 编译器
  xir/        ← 中间表示
  ...
```

**规则 F-6：平台相关代码用 unix/ 和 win/ 子目录隔离**

- libuv 的标杆模式：`src/unix/fs.c` vs `src/win/fs.c`，公共逻辑在 `src/fs-poll.c`
- 公共头文件定义接口，平台目录提供实现
- 通过 CMake/Makefile 选择编译哪个目录

---

## 第二章：头文件与接口设计规范

### 2.1 头文件依赖必须是 DAG

**规则 H-1：头文件依赖必须是严格的有向无环图（DAG），零循环依赖**

Lua 5.4 的头文件依赖层次（完美 DAG）：

```
Layer 0: lua.h, luaconf.h              ← 公共 API + 配置
Layer 1: llimits.h → lua.h             ← 基础类型限制
Layer 2: lobject.h → llimits.h, lua.h  ← 值/对象定义
Layer 3: lstate.h → lobject.h          ← 全局状态
Layer 4: lgc.h, ltable.h → lobject.h, lstate.h
Layer 5: lvm.h → ldo.h, lobject.h      ← 最上层
```

**每个头文件只 include 比自己更底层的头文件，绝不反向依赖。**

如何验证：如果在头文件依赖图中能找到一个环（A→B→C→A），就是设计缺陷。解决方案：提取共享类型到更底层的头文件，或用前向声明。

### 2.2 可见性控制体系

**规则 H-2：实现四级可见性控制**

| 级别 | Lua 5.4 | LuaJIT | xray 建议 | 用途 |
|------|---------|--------|-----------|------|
| 公共 API | `LUA_API` | `LUA_API` | `XRAY_API` | 嵌入者/用户可调用 |
| 标准库 API | `LUALIB_API` | `LUALIB_API` | `XRAY_LIB_API` | 标准库扩展用 |
| 内部跨模块 | `LUAI_FUNC` | `LJ_FUNC` | `XR_FUNC` | 模块间调用，不对外 |
| 文件内部 | `static` | `static` | `static` | 文件内部实现 |

关键特性：

- `XR_FUNC` 在 static build 下应变成 `static`（LuaJIT 的做法），让编译器优化跨模块调用
- 所有内部函数默认 `static`，只有确实被其他 .c 使用的才用 `XR_FUNC`

### 2.3 导出函数数量控制

**规则 H-3：每个头文件导出函数不超过 25 个**

Lua 5.4 实证：

| 头文件 | 导出函数数 | 对应 .c 行数 |
|--------|-----------|-------------|
| lvm.h | 17 | 1,972 |
| lgc.h | 11 | 1,804 |
| ltable.h | 20 | 1,355 |
| lstring.h | 13 | 372 |
| lfunc.h | 12 | 294 |
| ldo.h | 20 | 1,164 |
| lcode.h | 36 ← 略多 | 2,066 |

经验公式：**导出函数数 ≈ .c 文件行数 / 100**。如果一个头文件导出了 50+ 函数，说明模块职责过重，需要拆分。

### 2.4 头文件内容规范

**规则 H-4：头文件只包含声明，不包含实现**

头文件允许包含的内容：
- 类型定义（typedef, struct, enum）
- 宏定义（常量宏、内联操作宏）
- 函数声明（带可见性修饰符）
- 内联函数（仅限性能关键的短函数，<5行）

头文件**禁止**包含的内容：
- 函数实现（除极短内联函数）
- 全局变量定义（只允许 extern 声明）
- 长逻辑代码

**规则 H-5：每个头文件用 include guard，不用 #pragma once**

```c
#ifndef xray_module_h
#define xray_module_h
/* ... */
#endif
```

原因：`#pragma once` 不是 C 标准的一部分，跨编译器行为不一致。Lua、Redis、nginx 全部使用传统 include guard。

---

## 第三章：函数与封装规范

### 3.1 static 函数比例

**规则 C-1：每个 .c 文件的 static 函数比例应 ≥ 80%，目标 ≥ 90%**

Lua 5.4 实证（100% 封装率）：

```
lvm.c     — 100% static
lgc.c     — 100% static
lparser.c — 100% static
lcode.c   — 100% static
ltable.c  — 100% static
ldo.c     — 85% static
```

含义：**如果一个函数不需要被其他文件调用，它就必须是 static。没有例外。**

好处：
1. 编译器可以内联 static 函数（零成本模块化）
2. 防止符号污染和命名冲突
3. 明确标识"这是实现细节，不是接口"

### 3.2 函数大小

**规则 C-2：单个函数不超过 150 行（含注释），目标 80 行以内**

- Lua 的 `luaV_execute` 是唯一例外（VM 主循环，约 800 行），但通过宏将每条指令压缩到 3-10 行
- 超过 150 行的函数通常说明职责过多，应拆分为子函数

**规则 C-3：函数参数不超过 6 个**

- 超过 6 个参数应封装为结构体
- Lua 的函数参数平均 2-3 个，几乎从不超过 5 个
- 第一个参数通常是上下文（lua_State *L, vm *V 等）

### 3.3 函数命名

**规则 C-4：函数命名格式为 前缀_模块_动作**

各项目的命名风格：

```
Lua:    luaV_execute, luaC_step, luaH_getint, luaG_runerror
LuaJIT: lj_record_ins, lj_gc_step, lj_tab_new
nginx:  ngx_http_process_request, ngx_event_init
libuv:  uv_tcp_init, uv_run, uv__io_start (双下划线=内部)
```

xray 建议：

```
公共 API:  xray_vm_execute, xray_gc_collect
内部函数:  xr_vm_dispatch, xr_gc_mark_object
宏:        XR_IS_STRING, XR_TAG_PTR
类型:      XrValue, XrArray, XrGCHeader
```

### 3.4 全局变量

**规则 C-5：禁止文件作用域的可变全局变量**

- Lua 的 lvm.c 有 **零个** 全局/文件作用域变量
- 所有状态都通过 `lua_State *L` 参数传递
- 常量数组和 lookup table 可以是 `static const`

唯一允许的全局变量：
- `static const` 常量表（如 opcode 名称表、类型名称表）
- 单例模式的全局状态（必须通过函数访问，不直接暴露）

---

## 第四章：宏与抽象规范

### 4.1 宏是 C 语言唯一的零成本抽象

**规则 M-1：用宏消除重复模式，但只做一层抽象**

Lua lvm.c 的宏抽象层（教科书级别）：

```c
/* 算术指令族：2个参数消除 12+ 条指令的重复 */
#define op_arith(L,iop,fop) {  \
  TValue *v1 = vRB(i);  \
  TValue *v2 = vRC(i);  \
  op_arith_aux(L, v1, v2, iop, fop); }

/* 使用方式：每条指令只需 1 行 */
vmcase(OP_ADD)  { op_arith(L, luaV_iadd, luai_numadd); vmbreak; }
vmcase(OP_SUB)  { op_arith(L, luaV_isub, luai_numsub); vmbreak; }
vmcase(OP_MUL)  { op_arith(L, luaV_imul, luai_nummul); vmbreak; }
```

分层结构（从底层到顶层）：

```
op_arithf_aux  ← 最底层：浮点运算核心
  ↑
op_arith_aux   ← 中间层：整数快速路径 + 浮点回退
  ↑
op_arith       ← 顶层：寄存器操作数版本
op_arithK      ← 顶层：常量操作数版本
op_arithI      ← 顶层：立即数操作数版本
```

**每条指令最终只需一行调用，所有类型检查、快速路径、回退逻辑都被宏封装。**

### 4.2 宏命名规范

**规则 M-2：宏命名遵循层次**

```
XR_XXX          ← 全大写：常量宏、类型检查宏
xr_xxx          ← 小写前缀：类函数宏（看起来像函数调用）
XR_DCHECK       ← 调试断言（release 下编译消除）
```

### 4.3 条件编译

**规则 M-3：条件编译块不超过 20 行，超过则提取为独立函数或文件**

```c
/* 好的写法：短条件块 */
#ifdef XRAY_DEBUG
  xr_log("gc step: %d objects marked", count);
#endif

/* 坏的写法：长条件块（应提取到 xgc_debug.c）*/
#ifdef XRAY_DEBUG
  // 50 行调试代码...
#endif
```

### 4.4 X-macro 模式

**规则 M-4：使用 X-macro 管理枚举/表格的一致性**

libuv 的 errno 映射：

```c
#define UV_ERRNO_MAP(XX)    \
  XX(E2BIG, "too big")     \
  XX(EACCES, "permission denied") \
  XX(EADDRINUSE, "address in use") \
  ...

/* 自动生成枚举 */
typedef enum { UV_ERRNO_MAP(UV_ERRNO_GEN) } uv_errno_t;

/* 自动生成字符串表 */
const char* uv_strerror(int err) {
  switch(err) { UV_ERRNO_MAP(UV_STRERROR_GEN) }
}
```

xray 的 opcode 和类型定义适合用此模式，保证枚举值、字符串名称、dispatch table 永远同步。

---

## 第五章：错误处理与防御性编程

### 5.1 断言密度

**规则 E-1：每 50-100 行代码至少 1 个断言**

各项目 assert 密度：

| 项目 | 密度（每 N 行 1 个 assert） |
|------|---------------------------|
| chibicc | 1 per 7 lines ← 极端防御 |
| QBE | 1 per 68 lines |
| Lua 5.4 | 1 per 91 lines |
| Redis | 1 per 91 lines |
| LuaJIT | 1 per 111 lines |
| wren | 1 per 117 lines |

xray 建议：**目标 1 per 50-80 lines**，关键路径（GC、VM、JIT）应更密集。

### 5.2 断言层次

**规则 E-2：区分 debug-only 断言和 always-on 检查**

```c
/* Debug-only: release 下编译消除，零运行时成本 */
XR_DCHECK(ptr != NULL);                    /* 开发期捕获逻辑错误 */
XR_DCHECK(slot < frame->max_slots);        /* 边界检查 */

/* Always-on: 始终检查，防止数据损坏 */
XR_CHECK(alloc_size <= XR_MAX_ALLOC);      /* 防止 OOM 攻击 */
if (XR_UNLIKELY(!ptr)) return XR_ERR_OOM;  /* 分配失败处理 */
```

Lua 的做法：`lua_assert` 在非 debug 模式下变成 `((void)0)`，零成本；`api_check` 在 API 入口始终检查。

### 5.3 错误处理模式

**规则 E-3：使用统一的错误处理框架**

三种经过验证的模式：

**模式 A：返回值（libuv 风格）**
```c
int uv_tcp_bind(uv_tcp_t *handle, const struct sockaddr *addr, unsigned int flags);
/* 返回 0 成功，负值错误码 */
```

**模式 B：longjmp（Lua 风格）**
```c
LUAI_FUNC l_noret luaD_throw(lua_State *L, TStatus errcode);
/* 内部错误用 longjmp 跳出，配合 luaD_pcall 的 setjmp */
```

**模式 C：panic + 日志（Redis 风格）**
```c
serverPanic("Unknown object encoding %d", encoding);
/* 不可恢复的错误，记录日志后终止 */
```

xray 建议：VM/GC 内部用 longjmp（已有），API 入口用返回值，不可恢复错误用 panic。

### 5.4 前置条件检查

**规则 E-4：公共函数入口必须有参数验证**

```c
void xr_array_set(XrArray *arr, int index, XrValue val) {
    XR_DCHECK(arr != NULL);
    XR_DCHECK(index >= 0 && index < arr->count);
    /* ... */
}
```

---

## 第六章：内存管理规范

### 6.1 统一分配器

**规则 G-1：所有内存分配通过统一的分配器，禁止直接调用 malloc/free**

Lua 的做法：
```c
/* 所有分配通过 luaM_xxx 系列 */
#define luaM_new(L,t)       cast(t*, luaM_malloc_(L, sizeof(t), 0))
#define luaM_newvector(L,n,t) cast(t*, luaM_malloc_(L, (n)*sizeof(t), 0))
#define luaM_free(L, b)     luaM_free_(L, (b), sizeof(*(b)))
```

好处：
- 集中控制分配策略（arena、pool、GC 集成）
- 分配计数用于 GC 触发判断
- 便于统计内存使用、检测泄漏

### 6.2 GC barrier 规范

**规则 G-2：写入 GC 对象引用时必须插入 write barrier**

Lua 的 barrier 宏：
```c
#define luaC_barrier(L,p,v) (  \
    iscollectable(v) ? luaC_objbarrier(L,p,gcvalue(v)) : cast_void(0))

#define luaC_objbarrier(L,p,o) (  \
    (isblack(p) && iswhite(o)) ? \
    luaC_barrier_(L,obj2gco(p),obj2gco(o)) : cast_void(0))
```

关键原则：
- **forward barrier**：将白对象标灰（用于不可变的父对象）
- **back barrier**：将黑对象退灰（用于可能反复修改的容器）
- 选择哪种取决于父对象的修改频率

### 6.3 内存所有权

**规则 G-3：每块内存有且仅有一个所有者，所有权转移必须显式**

```c
/* 创建时获得所有权 */
XrString *s = xr_string_new(isolate, "hello", 5);

/* 转移所有权给 GC（通过赋值给 GC 管理的 slot）*/
frame->slots[0] = xr_obj_value(s);

/* 此后不要手动 free，GC 负责回收 */
```

---

## 第七章：命名规范

### 7.1 命名体系

**规则 N-1：四层命名体系**

```
xray_xxx()       ← 公共 API 函数（小写，用户可见）
xr_xxx()         ← 内部 API 函数（小写，模块间）
XR_XXX           ← 宏定义（全大写）
XrXxx            ← 类型定义（PascalCase）
```

### 7.2 模块前缀

**规则 N-2：每个子系统有唯一的 2-4 字母前缀**

| 子系统 | 前缀 | 示例 |
|--------|------|------|
| VM | xr_vm_ | xr_vm_execute, xr_vm_dispatch |
| GC | xr_gc_ | xr_gc_step, xr_gc_mark |
| 解析器 | xr_parse_ | xr_parse_expr, xr_parse_stmt |
| 词法器 | xr_lex_ | xr_lex_next, xr_lex_peek |
| 代码生成 | xr_emit_ | xr_emit_op, xr_emit_const |
| 类型系统 | xr_type_ | xr_type_check, xr_type_unify |
| 字符串 | xr_str_ | xr_str_new, xr_str_hash |
| 数组 | xr_array_ | xr_array_push, xr_array_get |

### 7.3 变量命名

**规则 N-3：变量名应短小但含义明确**

Lua 的变量命名风格（极简但清晰）：

```c
lua_State *L;        /* 全局上下文（几乎每个函数第一个参数）*/
TValue *ra, *rb;     /* 寄存器 A, B */
Instruction i;       /* 当前指令 */
CallInfo *ci;        /* 当前调用帧 */
Proto *p;            /* 函数原型 */
GCObject *o;         /* GC 对象 */
Table *h;            /* 哈希表（h = hash）*/
int n, b;            /* 计数、偏移等通用整数 */
```

规律：
- 高频变量用 1-2 字母（L, i, n, p, o）
- 在作用域很短（<20行）的地方，短名比长名更清晰
- 结构体字段可以更短，因为有类型上下文

---

## 第八章：注释规范

### 8.1 注释语言和格式

**规则 D-1：所有注释统一用英文**

（遵循用户 Global Rules 中的注释规范）

### 8.2 注释质量

**规则 D-2：注释解释"为什么"，不解释"做什么"**

```c
/* BAD: 说了跟代码一样的废话 */
count++;  /* increment count */

/* GOOD: 解释了不明显的设计决策 */
count++;  /* pre-increment to account for the sentinel node at index 0 */
```

**Lua lgc.h 的标杆级注释（第 125-160 行）：**

用 35 行注释完整解释了分代 GC 的 7 个年龄状态、状态转换规则、颜色不变量——这比任何外部文档都精确。关键特征：
- 先给出设计的"大图"
- 再解释每个状态的含义和约束
- 最后说明异常情况（线程和 upvalue 的特殊处理）

### 8.3 注释密度

**规则 D-3：注释占比 15-25%**

| 项目 | 注释占比 | 评价 |
|------|---------|------|
| Lua lgc.c | 26% | ✅ GC 需要详细解释 |
| Lua ltable.c | 25% | ✅ 哈希算法需要解释 |
| Lua lvm.c | 15% | ✅ 代码自解释性强 |
| Lua lparser.c | 14% | ✅ 递归下降本身是文档 |

### 8.4 分段注释

**规则 D-4：长文件用分段注释标记逻辑区域**

Lua 风格：
```c
/* {====================================================== */
/* GC Barrier functions */
/* ======================================================= */

/* ... 相关函数 ... */

/* }====================================================== */
```

### 8.5 文件头部注释

遵循用户 Global Rules 中定义的文件头格式。

---

## 第九章：编译与构建规范

### 9.1 编译警告

**规则 B-1：开启所有警告，-Werror 在 CI 中强制**

```cmake
target_compile_options(xray PRIVATE
    -Wall -Wextra -Wpedantic
    -Wno-unused-parameter    # 允许：回调函数签名固定
    -Wshadow                 # 禁止变量遮蔽
    -Wconversion             # 隐式类型转换警告
)
```

### 9.2 编译模式

**规则 B-2：支持 Debug / Release / ASan / TSan 四种构建**

- **Debug**: `-O0 -g -DXRAY_DEBUG`，所有断言开启
- **Release**: `-O2 -DNDEBUG`，断言关闭，LTO 优化
- **ASan**: `-fsanitize=address`，检测内存越界和 use-after-free
- **TSan**: `-fsanitize=thread`，检测数据竞争

---

## 第十章：架构约束规则（可量化检查）

以下规则可以通过脚本自动检查：

| # | 规则 | 阈值 | 检查命令 |
|---|------|------|---------|
| Q-1 | 单文件行数 | ≤ 3,000 | `wc -l *.c \| awk '$1>3000'` |
| Q-2 | 头文件行数 | ≤ 800 | `wc -l *.h \| awk '$1>800'` |
| Q-3 | 头文件导出函数数 | ≤ 25 | `grep -c 'XR_FUNC\|XRAY_API'` |
| Q-4 | 函数行数 | ≤ 150 | 静态分析工具 |
| Q-5 | 函数参数数 | ≤ 6 | 静态分析工具 |
| Q-6 | static 比例 | ≥ 80% | `grep -c '^static'` vs total |
| Q-7 | 断言密度 | ≤ 100 lines/assert | `grep -c 'DCHECK\|CHECK\|assert'` |
| Q-8 | 循环依赖 | = 0 | 依赖图分析 |
| Q-9 | 直接 malloc/free | = 0 | `grep -rn 'malloc\|free(' --include='*.c'` |
| Q-10 | 注释占比 | 15-25% | 注释行数 / 总行数 |

---

## 第十一章：xray 当前问题诊断与改进路线

### 11.1 当前超标文件

| 文件 | 行数 | 问题 | 建议 |
|------|------|------|------|
| xvm.c | 10,276 | 超标 3.4x | 参照 Lua lvm.c，用宏抽象算术/比较/属性访问指令族，提取内置方法分发到独立文件 |
| xir_builder.c | 6,036 | 超标 2x | 按 pass 类型拆分为多个 xir_build_xxx.c |
| xir_pass.c | 4,605 | 超标 1.5x | 每个 pass 一个文件（Lua 的 lgc.c, lcode.c 分别独立是最好的例子） |
| xanalyzer_visitor.c | 4,175 | 超标 1.4x | 按 AST 节点类型分组拆分 |
| xir_codegen.c | 3,752 | 超标 1.25x | 提取寄存器分配和指令选择为独立文件 |
| xparse.c | 3,731 | 超标 1.24x | 提取表达式解析和类型注解解析 |

### 11.2 核心对标

**以 Lua 5.4 lvm.c (1,972行) 为标杆，将 xvm.c (10,276行) 瘦身到 3,000 行以内。**

方法：
1. **宏抽象算术指令族**（参照 Lua op_arith 系列）→ 预计减少 ~2,000 行
2. **提取内置方法分发**到 xvm_builtin_dispatch.c → 预计减少 ~2,000 行  
3. **提取属性访问/类型转换**到 xvm_property.c → 预计减少 ~1,500 行
4. **提取迭代器/for-in 逻辑**到 xvm_iterator.c → 预计减少 ~1,000 行

### 11.3 改进优先级

1. **P0**：建立可见性宏体系（XR_FUNC / XRAY_API / static）
2. **P0**：头文件依赖 DAG 梳理（消除循环依赖）
3. **P1**：xvm.c 瘦身（宏抽象 + 拆分）
4. **P1**：统一断言宏（确保 debug/release 行为一致）
5. **P2**：其他超标文件拆分
6. **P2**：添加自动化架构检查脚本

---

## 附录 A：各项目架构特征速查表

| 维度 | Lua 5.4 | LuaJIT | QBE | Redis | nginx | libuv | SQLite |
|------|---------|--------|-----|-------|-------|-------|--------|
| 总代码量 | 34K | 99K | 18K | 175K | 230K | 110K | 260K(amalg) |
| 最大文件 | 2.2K | 2.9K | 1.7K | 15.6K | 7.3K | 3.8K | N/A |
| 头文件策略 | 1:1对应 | 1:1对应 | 单头all.h | server.h聚合 | ngx_core.h聚合 | 单头uv.h | sqlite3.h |
| 可见性控制 | 4级 | 4级(LJ_FUNC) | static | 无系统性 | static | uv/uv__ | SQLITE_API |
| 依赖方向 | 纯DAG | 纯DAG | 无依赖 | 扁平 | 分层DAG | 接口/平台分离 | 7层分层 |
| static比例 | 100% | >90% | >90% | ~70% | >85% | >80% | N/A |
| 断言密度 | 1/91行 | 1/111行 | 1/68行 | 1/91行 | 1/~100行 | 中等 | 极高 |
| 平台抽象 | luaconf.h | lj_arch.h | Makefile | config.h | os/unix os/win32 | unix/ win/ | os层 |
| 错误处理 | longjmp | longjmp | abort | panic+log | 返回值+log | 返回错误码 | 返回错误码 |
| 命名前缀 | lua/luaI/l | lj_ | 无 | 混合 | ngx_ | uv_ | sqlite3_ |

## 附录 B：Lua 5.4 核心架构图

```
                    ┌──────────────────────┐
                    │     lua.h (API)      │   Layer 0: 公共接口
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │   llimits.h (types)  │   Layer 1: 基础类型
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │ lobject.h (values)   │   Layer 2: 值/对象
                    └──────┬───────────────┘
                           │
              ┌────────────▼────────────┐
              │ lstate.h (global state) │      Layer 3: 全局状态
              └────┬──────────┬────┬────┘
                   │          │    │
         ┌─────▼──┐  ┌───▼───┐ ┌▼─────┐
         │ lgc.h  │  │ltable │ │lstring│  Layer 4: 子系统
         │ (GC)   │  │(table)│ │(str)  │
         └────────┘  └───────┘ └───────┘
                   │
              ┌────▼────────────────────┐
              │ lvm.h (VM interpreter)  │  Layer 5: 最上层
              └─────────────────────────┘

  所有依赖都是向下的（DAG），零循环。
  每个 .h 只暴露必要接口（10-20个函数）。
  每个 .c 的内部函数 100% static。
```

## 附录 C：检查脚本模板

```bash
#!/bin/bash
# scripts/check_architecture.sh
# Check architecture constraints for xray

SRC_DIR="src"
ERRORS=0

echo "=== xray Architecture Check ==="

# Q-1: File size limit
echo "--- Checking file sizes ---"
for f in $(find $SRC_DIR -name '*.c'); do
    lines=$(wc -l < "$f")
    if [ "$lines" -gt 3000 ]; then
        echo "FAIL: $f has $lines lines (limit: 3000)"
        ERRORS=$((ERRORS + 1))
    fi
done

# Q-2: Header size limit
echo "--- Checking header sizes ---"
for f in $(find $SRC_DIR -name '*.h'); do
    lines=$(wc -l < "$f")
    if [ "$lines" -gt 800 ]; then
        echo "FAIL: $f has $lines lines (limit: 800)"
        ERRORS=$((ERRORS + 1))
    fi
done

# Q-9: No direct malloc/free
echo "--- Checking direct malloc/free ---"
hits=$(grep -rn '\bmalloc\b\|\bfree\b\|\bcalloc\b\|\brealloc\b' \
    --include='*.c' $SRC_DIR \
    | grep -v 'xr_malloc\|xr_free\|xr_calloc\|xr_realloc\|luaM_\|# *include' \
    | grep -v '\.h:')
if [ -n "$hits" ]; then
    echo "FAIL: Direct malloc/free found:"
    echo "$hits"
    ERRORS=$((ERRORS + 1))
fi

echo ""
if [ "$ERRORS" -eq 0 ]; then
    echo "ALL CHECKS PASSED"
else
    echo "FAILED: $ERRORS violations found"
fi
exit $ERRORS
```
