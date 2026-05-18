# 消灭 Cold Path 层：VM 分发架构重构

## 背景

Cold path 分层最初为优化 VM I-cache（将低频代码标记 XR_NOINLINE，移入独立编译单元）。
现在 VM 性能不是首要目标（AOT 是未来），cold path 带来的复杂度成本超过收益：

- **5 个专属文件**（3433 行），独立编译单元导致编译器无法内联
- **VM_COLD_* 返回码体系**：函数返回 int，由 `VM_DISPATCH_COLD` 宏逐个 if 翻译为 vmbreak/goto/return
- **多路径分歧根源**：相同语义在 cold path + dispatch + JIT 三处独立实现（channel deep copy 已证明此类 bug）
- **参数爆炸**：每个函数 6-9 个参数（isolate, vm_ctx, instr/receiver, base, a, frame, pc）

## 目标

1. **消灭 cold path 作为独立编译单元** — 所有 VM 辅助函数与 `xvm.c` 在同一翻译单元
2. **消灭 XR_NOINLINE** — 编译器决定是否内联
3. **简单函数直接内联进 vmcase** — 消除函数调用开销和返回码翻译
4. **保留必要的辅助函数** — 复杂逻辑（>50 行）仍为 static 函数，但在同一 TU
5. **与 unified-class-dispatch 方案协同** — property dispatch if-else 链在该方案中消灭

## 当前文件清单

| 文件 | 行数 | 函数 | 归宿 |
|------|------|------|------|
| `xvm_cold_paths.h` | 170 | 声明 + 共享宏/helpers | 合入 `xvm_internal.h` |
| `xvm_cold_chan.c` | 329 | 3 函数 | → `xvm_dispatch_chan.inc.c` |
| `xvm_cold_call.c` | 693 | 6 函数 | → `xvm_dispatch_invoke.inc.c` |
| `xvm_cold_object.c` | 1039 | 5 函数 | → `xvm_dispatch_object.inc.c` |
| `xvm_cold_coro.c` | 1203 | 9 函数 | → `xvm_dispatch_coro.inc.c` |
| **合计** | **3433** | **23 函数** | |

## 函数逐个归宿分析

### xvm_cold_chan.c → xvm_dispatch_chan.inc.c

| 函数 | 行数 | 处理方式 | 理由 |
|------|------|----------|------|
| `vm_select_block` | 104 | **保留为 static 函数** | 复杂逻辑（内存分配、timer、dist hooks），但不需 NOINLINE |
| `vm_chan_send_timeout` | 86 | **保留为 static 函数** | 含超时+轮询逻辑 |
| `vm_chan_recv_timeout` | 92 | **保留为 static 函数** | 含超时+轮询逻辑 |

**操作**：
1. 将 3 个函数体移入 `xvm_dispatch_chan.inc.c` 开头（dispatch switch 之前的文件级 scope，由 xvm.c 在 `run()` 之前 include）
2. 不对——.inc.c 是在 run() 内部 include 的。需要新的 include 位置。
3. **方案**：创建 `xvm_helpers.inc.c`，在 xvm.c 的 `run()` 函数定义之前 `#include`。此文件包含所有从 cold path 迁移来的 static 函数。
4. 删除 `xvm_cold_chan.c`。

### xvm_cold_call.c → xvm_dispatch_invoke.inc.c

| 函数 | 行数 | 处理方式 | 理由 |
|------|------|----------|------|
| `vm_invoke_channel` | 136 | **保留为 static 函数** | 处理 trySend/tryRecv/send/recv/close/isClosed 等多分支，含阻塞语义 |
| `vm_invoke_task_handle` | 63 | **保留为 static 函数** | wait/cancel/toString 分支 |
| `vm_invoke_coro_handle` | 18 | **直接内联进 vmcase** | 仅 3 个分支（toString/isCancelled/isFinished），极简 |
| `vm_invoke_enum` | 67 | **保留为 static 函数** | values/fromValue/toString 等分支 |
| `vm_invoke_class` | 131 | **保留为 static 函数** | 构造器 + static method + frame push |
| `vm_superinvoke` | 116 | **保留为 static 函数** | super.method() + constructor :super() |

**操作**：
1. `vm_invoke_coro_handle`（18 行）内联进 OP_INVOKE 的 coro 分支。
2. 其余 5 个函数移入 `xvm_helpers.inc.c`，改 `XR_NOINLINE` 为 `static`。
3. 删除 `xvm_cold_call.c`。

### xvm_cold_object.c → xvm_dispatch_object.inc.c

| 函数 | 行数 | 处理方式 | 理由 |
|------|------|----------|------|
| `vm_setprop_type_dispatch` | 193 | **保留，标记待删除** | 等 unified-class-dispatch 完成后用 XrClass property setter 替代 |
| `vm_setprop_instance_setter` | 61 | **保留为 static 函数** | setter 查找 + 可能调 closure |
| `vm_getprop_type_dispatch` | 523 | **保留，标记待删除** | 最大的 if-else 链，等 XrClass property getter 注册后替代 |
| `vm_getprop_instance_getter` | 66 | **保留为 static 函数** | getter 查找 + 可能调 closure |
| `vm_invoke_module` | 140 | **保留为 static 函数** | module export 查找 + frame push |

**操作**：
1. 全部移入 `xvm_helpers.inc.c`，改 `XR_NOINLINE` 为 `static`。
2. `vm_getprop_type_dispatch` 和 `vm_setprop_type_dispatch` 头部加注释 `/* TODO: replace with XrClass property dispatch */`。
3. 删除 `xvm_cold_object.c`。

### xvm_cold_coro.c → xvm_dispatch_coro.inc.c

| 函数 | 行数 | 处理方式 | 理由 |
|------|------|----------|------|
| `vm_coro_ctrl` | 483 | **拆分** | 内含 10+ sub-ops（sleep/yield/collect/etc），按子操作拆为 3-4 个小函数 |
| `vm_go` | 137 | **保留为 static 函数** | goroutine 创建 + frame push |
| `vm_await_recycle_coro` | 8 | **移为 static inline** | 极小 helper |
| `vm_task_consume_result` | 16 | **移为 static inline** | 极小 helper |
| `vm_await_read_result` | 12 | **移为 static inline** | 极小 helper |
| `vm_await` | 97 | **保留为 static 函数** | 含 CAS + blocking + 多类型支持 |
| `vm_await_timeout` | 100 | **保留为 static 函数** | await + timer 逻辑 |
| `vm_await_all` | 114 | **保留为 static 函数** | 数组遍历 + batch await |
| `vm_await_any` | 107 | **保留为 static 函数** | 数组遍历 + race 语义 |

**操作**：
1. `vm_coro_ctrl` (483 行) 超过 150 行限制。拆分为：
   - `vm_coro_ctrl_sleep(...)` — sleep/usleep
   - `vm_coro_ctrl_yield(...)` — yield/reduce
   - `vm_coro_ctrl_debug(...)` — collect/debug/coroutines introspection
   - `vm_coro_ctrl` 变为 switch 分派到上面 3 个
2. 小 helper (`vm_await_recycle_coro`, `vm_task_consume_result`, `vm_await_read_result`) 移为 static inline。
3. 其余保留为 static 函数，移入 `xvm_helpers.inc.c`。
4. 删除 `xvm_cold_coro.c`。

### xvm_cold_paths.h → 拆解

| 内容 | 行数 | 归宿 |
|------|------|------|
| includes (25 行) | 25 | **删除** — xvm.c 已有全部 includes |
| likely/unlikely 宏 | 7 | **删除** — xvm_internal.h 已有 |
| VM_INTERN / VM_INTERN_KEY | 2 | → `xvm_internal.h` |
| GC_SAFE_ENTER/LEAVE | 2 | → `xvm_internal.h` |
| COLD_CORO 宏 | 1 | **删除** — 改用 VM_CURRENT_CORO |
| VM_COLD_* 返回码 | 7 | → `xvm_internal.h`（重命名为 `VM_HELPER_*`） |
| VM_COLD_THROW 宏 | 7 | → `xvm_internal.h`（重命名为 `VM_HELPER_THROW`） |
| vm_cold_get_coro() | 6 | → `xvm_internal.h`（重命名为 `vm_get_current_coro`） |
| vm_chan_copy_send/recv | 8 | → `xvm_internal.h`（保持原名） |
| VmCoroEntry typedef | 4 | → `xvm_helpers.inc.c`（仅 coro helpers 使用） |
| 函数声明（25 行） | 25 | **删除** — 函数变 static，声明在同文件 |

**操作**：
1. 将共享宏/helpers 移入 `xvm_internal.h`。
2. 删除 `xvm_cold_paths.h`。

## 新文件结构

```
src/vm/
├── xvm.c                          (843 行，不变)
│   #include "xvm_internal.h"
│   #include "xvm_helpers.inc.c"   ← NEW: static helper functions (file scope)
│   XrVMResult run(...) {
│       #include "xvm_dispatch_*.inc.c"  ← 现有 dispatch files (inside switch)
│   }
├── xvm_internal.h                 (现有 + 从 cold_paths.h 迁入的宏)
├── xvm_helpers.inc.c              ← NEW: ~1800 行，所有辅助函数
├── xvm_dispatch_chan.inc.c        (现有，调用 helpers)
├── xvm_dispatch_invoke.inc.c      (现有，调用 helpers)
├── xvm_dispatch_object.inc.c      (现有，调用 helpers)
├── xvm_dispatch_coro.inc.c        (现有，调用 helpers)
└── ... 其余 dispatch files ...

删除：
  xvm_cold_paths.h
  xvm_cold_chan.c
  xvm_cold_call.c
  xvm_cold_object.c
  xvm_cold_coro.c
```

## xvm_helpers.inc.c 组织

```c
/*
 * xvm_helpers.inc.c — VM dispatch helper functions
 *
 * Included at file scope in xvm.c before run().
 * All functions are static (same TU, compiler decides inlining).
 * NOT a standalone translation unit.
 */

/* ===== Shared inline helpers ===== */
static inline void vm_await_recycle_coro(XrCoroutine *coro) { ... }
static inline XrValue vm_task_consume_result(...) { ... }
static inline XrValue vm_await_read_result(...) { ... }

/* ===== Channel helpers ===== */
static int vm_select_block(...) { ... }
static int vm_chan_send_timeout(...) { ... }
static int vm_chan_recv_timeout(...) { ... }

/* ===== Invoke helpers ===== */
static int vm_invoke_channel(...) { ... }
static int vm_invoke_task_handle(...) { ... }
static int vm_invoke_enum(...) { ... }
static int vm_invoke_class(...) { ... }
static int vm_superinvoke(...) { ... }
static int vm_invoke_module(...) { ... }

/* ===== Property helpers (temporary — will be replaced by XrClass dispatch) ===== */
static int vm_getprop_type_dispatch(...) { ... }
static int vm_getprop_instance_getter(...) { ... }
static int vm_setprop_type_dispatch(...) { ... }
static int vm_setprop_instance_setter(...) { ... }

/* ===== Coroutine helpers ===== */
static int vm_coro_ctrl_sleep(...) { ... }
static int vm_coro_ctrl_yield(...) { ... }
static int vm_coro_ctrl_debug(...) { ... }
static int vm_coro_ctrl(...) { ... }   /* switch to sub-functions */
static int vm_go(...) { ... }
static int vm_await(...) { ... }
static int vm_await_timeout(...) { ... }
static int vm_await_all(...) { ... }
static int vm_await_any(...) { ... }
static int vm_collect_all_coros(...) { ... }
```

**预估行数**：~1800 行（原 3433 减去 includes/declarations/NOINLINE boilerplate + vm_invoke_coro_handle 内联）。

## 返回码重命名

cold path 概念消除后，返回码仅表示"辅助函数请求的控制流动作"：

```c
/* xvm_internal.h */
#define VM_HELPER_BREAK      0   /* vmbreak — next instruction */
#define VM_HELPER_CONTINUE  -1   /* caller handles next step */
#define VM_HELPER_STARTFUNC  1   /* goto startfunc — enter new frame */
#define VM_HELPER_BLOCKED    2   /* return XR_VM_BLOCKED */
#define VM_HELPER_YIELD      3   /* return XR_VM_YIELD */
#define VM_HELPER_ERROR      4   /* exception thrown, unwind */
#define VM_HELPER_FATAL      5   /* unrecoverable error */
#define VM_HELPER_GO_CHILD   6   /* return XR_VM_GO_CHILD */
```

`VM_DISPATCH_COLD` 宏重命名为 `VM_DISPATCH_HELPER`，逻辑不变。

## 与 unified-class-dispatch 的关系

本方案是 **前置步骤**，unified-class-dispatch 是 **后续步骤**：

```
本方案（消灭 cold path 层）         unified-class-dispatch 方案
─────────────────────────        ────────────────────────────
消灭独立编译单元                    统一 XrMethodFn 调用约定
消灭 XR_NOINLINE                   XrClass 注册所有 builtin 方法
合并到同一 TU                       重写 OP_INVOKE（~150 行替代 ~770 行）
保留 vm_getprop_type_dispatch ──→   用 XrClass property dispatch 替代 ──→ 删除
保留 vm_invoke_channel ──→         Channel 方法注册到 XrClass ──→ 可能简化
保留 OP_INVOKE_BUILTIN ──→         删除 OP_INVOKE_BUILTIN + XrICBuiltin
```

## 外部依赖检查

| 外部文件 | 引用内容 | 处理 |
|----------|----------|------|
| `src/coro/xtask.h` L36 | 注释 `xvm_cold_paths.c` | 更新注释 |
| `src/coro/xtask.c` L26 | 注释 `xvm_cold_paths.c` | 更新注释 |
| `src/jit/xm_jit_runtime_coro.c` L1131 | 注释 `xvm_cold_paths.c` | 更新注释 |
| `CMakeLists.txt` L531 | `xvm_cold_paths.c` fno-lto | 删除（文件不存在） |

**无代码依赖**。所有外部引用仅为注释。

## 执行顺序

1. **xvm_internal.h**：添加从 `xvm_cold_paths.h` 迁入的宏/helpers
2. **创建 `xvm_helpers.inc.c`**：移入所有 static 函数
3. **xvm.c**：`#include "xvm_helpers.inc.c"` 在 `run()` 之前；删除 `#include "xvm_cold_paths.h"`
4. **vm_invoke_coro_handle**：内联进 `xvm_dispatch_invoke.inc.c`
5. **vm_coro_ctrl**：拆分为 3 个子函数
6. **删除**：`xvm_cold_paths.h`, `xvm_cold_chan.c`, `xvm_cold_call.c`, `xvm_cold_object.c`, `xvm_cold_coro.c`
7. **更新**：CMakeLists.txt 删除 stale fno-lto 引用
8. **更新注释**：xtask.h, xtask.c, xm_jit_runtime_coro.c
9. **编译 + 全量测试**

## 风险评估

| 风险 | 可能性 | 缓解 |
|------|--------|------|
| xvm_helpers.inc.c 超 3000 行 | 低（预估 ~1800） | vm_coro_ctrl 拆分已压缩 |
| static 函数符号冲突 | 无 | 全部 static，仅 xvm.c TU 可见 |
| 编译时间增加（大 TU） | 低 | xvm.c 原来就包含 10+ .inc.c |
| 语义变更 | 无 | 纯机械重构，不改逻辑 |

## 预期收益

- **删除 5 个文件**，净减 ~1600 行（3433 原始 - ~1800 合并后）
- **消灭 XR_NOINLINE**：编译器可自由内联 hot helper
- **同一 TU**：编译器可做跨函数优化（const propagation, dead arg elimination）
- **命名一致**：不再有 "cold" 概念，只有 "helper"
- **为 unified-class-dispatch 铺路**：property dispatch 函数已标记待替换
