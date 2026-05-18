# C 代码规范

## 文件规模

| 约束 | 限制 | 目标 |
|------|------|------|
| .c 文件 | ≤ 3,000 行 | ≤ 2,000 行 |
| .h 文件 | ≤ 800 行 | |
| 单函数 | ≤ 150 行 | ≤ 80 行 |
| 函数参数 | ≤ 6 个 | 超过用结构体 |
| .h 导出函数 | ≤ 25 个 | |
| static 函数比例 | ≥ 80% | ≥ 90% |

## 可见性（四级，定义在 `src/base/xdefs.h`）

```c
XRAY_API    // 公共 API（嵌入者可调用）
XR_FUNC     // 内部跨模块函数（amalg build 下变 static）
XR_DATA     // 内部跨模块数据（amalg build 下变 static）
static      // 文件内部（默认选择）
// 无修饰符的非 static 函数 = BUG
```

## 命名

```c
xray_xxx()    // 公共 API
xr_xxx()      // 内部 API
XR_XXX        // 宏
XrValue       // 类型：Xr + PascalCase
xr_vm_xxx()   // 模块前缀：xr_ + 模块名 + 动作
```

## 文件命名

- `src/` 下 `.c/.h` 统一 `x` 前缀（`xlex.h`, `xarray.c`）
- `include/` 下公共头文件 `xray_` 前缀（`xray.h`）
- Include guard: `#ifndef XLEX_H` / `#endif // XLEX_H`（`//` 风格，禁止 `/* */`）

## 文件头注释模板

```c
/*
 * xray - Lightweight typed scripting with native concurrency
 * https://www.xray-lang.org
 *
 * Copyright (c) 2026 Xinglei Xu <xingleixu@gmail.com>
 * Licensed under the MIT License
 *
 * <文件名> - <简短英文描述>
 *
 * KEY CONCEPT:
 *   <核心概念，1-3句话>
 */
```

## 内存安全（定义在 `src/base/xmalloc.h`）

- 所有分配通过 `xr_malloc/xr_free`，**禁止直接 malloc/free**
- `xr_malloc/xr_calloc` 后**必须检查 NULL**
- `xr_realloc` **禁止直接赋值给原指针**：

```c
// 错误: ptr = xr_realloc(ptr, new_size);
// 正确:
T *tmp = (T *)xr_realloc(ptr, new_size);
if (!tmp) return;
ptr = tmp;
```

- 分配失败必须正确清理已分配资源
- 检测工具: `python3 scripts/static_check.py -c memory`

## 断言（定义在 `src/base/xchecks.h`）

- 密度目标：每 50-80 行至少 1 个
- `XR_DCHECK(cond, msg)` — debug-only，release 消除
- `XR_CHECK(cond, msg)` — always-on
- `XR_CHECK_BOUNDS(index, limit, msg)` — 数组边界检查
- `XR_DCHECK_EQ/NE/LT/LE/GT/GE(a, b, msg)` — 比较断言
- 公共函数入口必须有参数验证
- **禁止** `XR_DCHECK(ptr != NULL)` 紧接 `if (!ptr) return`（矛盾模式）

## 注释

- **所有注释用英文**
- 单行 `//`，多行 `/* ... */`
- 注释解释"为什么"，不解释"做什么"
- 长文件用 `/* === Section Name === */` 分段

### 注释铁律（绝对禁止 / 同样适用于 git commit message）

文档会迁移、改名、归档；阶段是实施期间的临时坐标，做完就过时。**永久代码与历史里只允许出现自包含的事实**。

**❌ 禁止引用 `.md` 文档路径**

```c
// BAD: 当文档被归档/重命名时立即失效
// See docs/tasks/001-vm-refactor.md for details.
// Per docs/engineering/jit_known_limitations.md §3.
```

**❌ 禁止任何"阶段说辞"**

```c
// BAD: 阶段是实施期间的临时坐标
// Phase 0 of vm refactor: unified entry contract.
// TODO: fix in Phase 5.
// P0 priority — see refactor plan.
// Round 2 fix.
// 本次重构后改为 ...
```

**✅ 正确做法**

直接陈述**事实**和**原因**；要长期保留的设计意图写到对应函数 / 类型 / 模块的 doc comment 里。

```c
// GOOD: 自包含的事实 + 不变量 + 原因
// Single authoritative ctx resolver. Concurrent isolates would race
// on the previous TLS-based lookup, so this MUST be called only from
// the owning worker thread (XR_DCHECK below enforces this).
XR_FUNC XrExecCtx *xr_vm_current_ctx(void);
```

同样规则适用于 **git commit message**：

```
BAD:  refactor(vm): Phase 0 of vm_refactor_plan
GOOD: refactor(vm): unify entry contract via xr_vm_current_ctx
```

## 宏

- 用宏消除重复，但只做**一层抽象**
- 条件编译块 ≤ 20 行，超过提取为函数/文件
- 使用 X-macro 保证枚举/字符串表/dispatch table 同步

## 错误处理

- VM/GC 内部：longjmp
- API 入口：返回错误码
- 不可恢复错误：panic + 日志
