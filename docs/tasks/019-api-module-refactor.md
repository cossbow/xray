# src/api 模块重构实施文档

> **原则：不考虑向后兼容性，直接采用最佳设计。**

## 概览

src/api 是 Xray 运行时的对外入口层，当前 15 个文件 ~75K，存在 P0 级 bug、
可见性违规、内存安全缺陷、死代码等问题。本文档按 4 个阶段逐步清理，
每个阶段独立可验证。

---

## Phase 1 — 紧急 bug 修复（P0）

> 不改接口、不改结构，只修 bug。每个修完跑 `ctest --output-on-failure`。

### 1.1 REPL 死循环 — `xrepl.c:103`

**问题**：`ctx->shared_var_capacity == 0` 时 `new_capacity = 0 * 2 = 0`，while 永不退出。

```c
// Before
int new_capacity = ctx->shared_var_capacity * 2;

// After
int new_capacity = ctx->shared_var_capacity < 8 ? 8 : ctx->shared_var_capacity * 2;
```

### 1.2 malloc 未检 NULL — `xtest_runner.c`

4 处 `xr_malloc` 后直接 `memset`/`memcpy`，NULL 时段错误。

| 位置 | 修复方式 |
|------|---------|
| `xr_test_runner_new` (L51) | 加 `if (!runner) return NULL;` |
| `xr_test_discover` (L78) | 加 `if (!suite) return NULL;` |
| `xr_test_runner_add_failure` (L146-148) | 每个 `xr_malloc` 后加 NULL 检查 |

### 1.3 malloc 未检 NULL — `xray_runtime.c`

| 位置 | 修复方式 |
|------|---------|
| `xrt_str_concat` (L40) | `if (!r) return XRT_NULL;` |
| `xrt_string_slice` (L171) | `if (!r) return XRT_NULL;` |

### 1.4 不安全 realloc — `xtest_runner.c`

两处直接赋值覆盖原指针：

```c
// Before (L28)
suite->tests = (XrTestFunc *)xr_realloc(suite->tests, ...);

// After — 使用项目宏 XR_REALLOC_OR_ABORT
XR_REALLOC_OR_ABORT(suite->tests,
    sizeof(XrTestFunc) * suite->test_capacity,
    "suite_add_test: tests grow");
```

`suite_add_hook` (L39) 同理。

---

## Phase 2 — 代码规范对齐（P1）

> 统一可见性、消除矛盾断言、对齐命名。

### 2.1 可见性修饰符补全

**规则**：公共 API 函数 → `XRAY_API`，内部跨模块函数 → `XR_FUNC`。

| 文件 | 函数 | 修饰符 |
|------|------|--------|
| `xisolate_tls.c` | `xray_isolate_enter`, `xray_isolate_exit` | `XRAY_API` |
| `xisolate_scripting.c` | `xray_isolate_dostring`, `xray_isolate_dofile`, `xray_isolate_dofile_debug`, `xray_isolate_set_script_info` | `XRAY_API` |
| `xisolate_full.c` | `xray_isolate_setup_full` | `XRAY_API` |
| `xisolate.c` | `xr_isolate_get_symbol_table` | `XR_FUNC` + 参数改为 `XrayIsolate*` |
| `xruntime.c` | `xray_alloc`, `xray_realloc`, `xray_free` | `XRAY_API` |
| `xray_runtime.c` | 所有 `xrt_*` 函数 | `XRAY_API` |

### 2.2 消除矛盾断言模式

`xglobal_object.c:27-31` — `XR_DCHECK` 紧接 `if (!ptr) return`。

**修复策略**：公共函数入口统一用 `xray_api_checkr` / `xray_api_check`
（内含 DCHECK + early return），去掉手动 if + log。

```c
// Before
XR_DCHECK(isolate != NULL, "global_object_create: NULL isolate");
if (isolate == NULL) {
    xr_log_warning("global", "global_object_create: isolate is NULL");
    return NULL;
}

// After
xray_api_checkr(isolate != NULL, "global_object_create: NULL isolate", NULL);
```

### 2.3 硬编码字符串 → 常量

在 `xtype_names.h`（或合适位置）新增：
```c
#define CLASS_NAME_STRING_BUILDER  "StringBuilder"
```

`xglobal_object.c:127` 改用常量。

### 2.4 `xr_isolate_get_symbol_table` 签名修正

```c
// Before
void* xr_isolate_get_symbol_table(void *isolate);

// After
XR_FUNC void* xr_isolate_get_symbol_table(XrayIsolate *isolate);
```

---

## Phase 3 — 健壮性提升（P2）

### 3.1 `isolate_init_full` 统一 fail-fast

当前：部分子系统分配失败静默跳过，部分返回 -1。

**修复**：所有关键子系统失败统一 return -1。

```c
// 修改原则：
isolate->config = xr_malloc(sizeof(XrayConfig));
if (!isolate->config) return -1;    // 不再静默跳过
xr_config_init((XrayConfig*)isolate->config);

isolate->analyzer_pool = xr_type_pool_new();
if (!isolate->analyzer_pool) return -1;

isolate->symbol_table = xr_symbol_table_create();
if (!isolate->symbol_table) return -1;

isolate->global_object = xr_global_object_create(isolate);
if (!isolate->global_object) return -1;
```

> 注意：`isolate_init_full` 返回 -1 后，`xisolate.c` 的 `fail_after_vm:`
> 分支会执行 cleanup，再由 `isolate_cleanup_full`（通过 `cleanup_extra`）
> 释放已初始化的子系统。需确认 cleanup 能安全处理半初始化状态（当前已用
> NULL 检查保护，无需额外修改）。

### 3.2 `xr_global_register_all_core_classes` 检查返回值

```c
// 方案：宏简化，任一注册失败即返回 false
#define REGISTER_OR_FAIL(name, klass) \
    if (!xr_global_register_class(global, (name), (klass))) return false

REGISTER_OR_FAIL(CLASS_NAME_OBJECT, core->objectClass);
REGISTER_OR_FAIL(TYPE_NAME_STRING, core->stringClass);
// ...

#undef REGISTER_OR_FAIL
```

### 3.3 断言密度补全

按 "50-80 行 ≥ 1 个断言" 规则，需重点补充：

| 文件 | 当前断言数 | 需补至少 |
|------|-----------|---------|
| `xray_runtime.c` (205行) | 0 | 3-4 个 |
| `xrepl.c` (251行) | 0 | 4-5 个 |
| `xglobal_object.c` (137行) | 1 | 2 个 |
| `xvm_compile.c` (327行) | 6 | 已达标 |

重点位置：
- `xrepl.c` — `xr_repl_compile` 入口、编译结果校验、seed/collect 入口
- `xray_runtime.c` — 各算术函数入口的 tag 合法性检查
- `xglobal_object.c` — `register_all_core_classes` 中 `core` 指针校验

### 3.4 `xray_isolate_get_stats` gc_count 硬编码

```c
// Before
if (gc_count) *gc_count = 0;

// After
if (gc_count) *gc_count = isolate->gc.gc_count;  // 从 GC state 获取
```
> 需确认 `XrGC` 结构体是否有 `gc_count` 字段，若无则新增。

---

## Phase 4 — 结构优化（P3）

### 4.1 删除死代码：`xruntime.h` / `xruntime.c`

`xruntime.h` 声明了 ~60 个 API（类型检查、值构造、容器操作等），
`xruntime.c` 仅实现 3 个且标注 "currently unused"。

**决策**：
- **删除** `xruntime.c`（3 个未使用的 wrapper 函数）
- **精简** `xruntime.h`：仅保留已有实现对应的声明（或降级为内部文档）
- 真正的 Runtime API 由 `xray.h` + `xray_isolate.h` 承担

### 4.2 AOT 运行时内存管理

`xray_runtime.c` 中 `xrt_str_concat` / `xrt_string_slice` 每次
`xr_malloc` 新字符串，无释放路径 → 累积泄漏。

**方案**：引入轻量 bump allocator：

```c
// 新增 xrt_arena.h
typedef struct {
    char *base;
    size_t used;
    size_t capacity;
} XrtArena;

void  xrt_arena_init(XrtArena *a, size_t cap);
void *xrt_arena_alloc(XrtArena *a, size_t size);
void  xrt_arena_reset(XrtArena *a);   // 批量释放
void  xrt_arena_free(XrtArena *a);
```

所有 `xrt_*` 字符串操作从 arena 分配，调用者在合适时机 reset。
> 考虑到 AOT 模块尚处早期阶段，此项可标记为 TODO 暂缓。

### 4.3 `xrt_div` 浮点除零处理

```c
// After
double fb = (b.tag == XRT_TAG_I64) ? (double)b.i : b.f;
if (fb == 0.0) {
    fprintf(stderr, "xrt_div: division by zero\n");
    return xrt_box_float(0.0);  // 或 NaN，取决于语言语义
}
return xrt_box_float(fa / fb);
```
> 需确认 Xray 语言规范对浮点除零的定义。

### 4.4 移除多余 `#include <stdlib.h>`

以下文件使用 `xr_malloc/xr_free` 且不直接调用 `stdlib.h` 中任何函数：
- `xglobal_object.c`（无 `abort`/`exit`/`atoi` 等）
- `xisolate.c`（同上）

### 4.5 平台兼容性

`xisolate_scripting.c` 中 `#include <unistd.h>`（`getcwd`）和
`<limits.h>`（`PATH_MAX`）是 POSIX-only。若需支持 Windows：

```c
#ifdef _WIN32
#include <direct.h>
#define getcwd _getcwd
#define PATH_MAX _MAX_PATH
#else
#include <unistd.h>
#include <limits.h>
#endif
```
> 当前阶段不需要 Windows 支持则可忽略。

---

## 实施顺序 & 检查点

```
Phase 1 (P0 bug fix)     ──→ ctest 全过 ──→ commit
Phase 2 (规范对齐)        ──→ ctest 全过 ──→ commit
Phase 3 (健壮性)          ──→ ctest 全过 ──→ commit
Phase 4 (结构优化)        ──→ ctest 全过 ──→ commit
```

每个 Phase 单独 commit，message 格式：
```
refactor(api): Phase N — <一句话摘要>
```

## 预期效果

| 维度 | Before | After |
|------|--------|-------|
| P0 bug | 6 处 | 0 |
| 可见性违规 | ~20 处 | 0 |
| 不安全 realloc | 2 处 | 0 |
| 矛盾断言 | 1 处 | 0 |
| 死代码行数 | ~270 行 | 0 |
| 断言密度不足文件 | 3 个 | 0 |
| malloc 未检 NULL | 6 处 | 0 |
