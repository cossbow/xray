# DAP 模块重写实施文档（`src/app/dap`）

> **⚡ 开发原则：不考虑向后兼容性！**
>
> - Xray 没有外部用户，DAP 核心代码不到 4000 行，可以大胆重写
> - **不做兼容层**：不引入 bridge / adapter / shim，直接用正确 API
> - **不在错误接口上打补丁**：debug hook 不够用就重新设计，不强行适配
> - **不迁移死代码**：从未被调用的函数直接删除，在正确位置从头写
> - **不分 7 个 phase**：3 个 Phase 解决全部问题
> - capability 只宣称已实现的，未实现的不写进代码
>
> **本文定位**：把 `src/app/dap` 的源码审计结论收敛为可执行的 3-Phase 重写方案。

---

## 1. 背景

### 1.1 当前结构

```text
src/app/cli/xcmd_dap.c          # CLI 入口
src/app/dap/
├── xdap_protocol.c    (1018行)  # 协议 + 事件循环 + 所有 handler
├── xdap_controller.c  (366行)   # 会话状态 + 生命周期
├── xdap_debug.c       (1926行)  # 断点/变量/协程/反汇编/hook（大杂烩）
├── xdap_eval.c        (260行)   # 表达式求值
├── xdap_inspect.c     (241行)   # 栈/变量检查
└── xdap_transport.c   (592行)   # stdio/TCP 传输

src/runtime/xray_debug_hooks.[ch]  # VM 侧最小 hook 接口
src/vm/xvm.c                       # VM_DEBUG_CHECK 宏（调用 hook）
src/coro/xworker_sysmon.c          # debug resume 路径
```

### 1.2 当前测试

```text
tests/unit/dap/
├── test_dap_server.c       # JSON 消息格式
├── test_dap_breakpoints.c  # 断点数据结构
└── test_dap_transport.c    # transport framing
```

只覆盖 JSON / 数据结构 / transport 概念层，没有真实调试会话测试。

### 1.3 核心问题

当前 DAP 的主要矛盾不是"缺少功能"，而是 **写了功能但没接线**：

- VM 实际走的 hook 路径和 DAP 写好的逻辑**不是同一条路径**
- `xr_debug_on_line()` (1926行文件里最核心的函数) **从未被任何代码调用**
- `xdap_on_stopped()` **从未被任何代码调用**
- 停止状态存在 controller / debug state / protocol 三头管理
- `xray_free(isolate)` 在 `include/xray.h` 和 `src/api/xruntime.c` 里语义冲突
- capability 宣称 > 实际能力

---

## 2. 审计发现（按严重度排序）

### 2.1 致命：生命周期与接线

| ID | 问题 | 证据 |
|----|------|------|
| DAP-01 | isolate 析构用了语义冲突的 `xray_free()` | `xdap_controller.c` 3处调用；`include/xray.h` 声明为"free isolate"但 `src/api/xruntime.c` 实现为 GC no-op；真正析构是 `xray_isolate_delete()` |
| DAP-02 | `xr_debug_on_line()` 从未被调用 | 全局搜索 0 个调用点；其中包含 pause/logpoint/function bp/stop state 更新 |
| DAP-03 | `xdap_on_stopped()` 从未被调用 | 全局搜索 0 个调用点 |
| DAP-04 | `pause` 未接线 | controller 写 atomic 标志 → 消费者在 `xr_debug_on_line()` → 该函数无调用点 |
| DAP-05 | `stopOnEntry` 未接线 | launch 只设 `ctrl->step_mode` 意图字段，未调 `xr_debug_step_in()` 修改 VM 读取的真实 action |
| DAP-06 | 停止状态三头管理 | `ctrl->step_mode` vs `dbg->current_action`；`ctrl->stopped_*` vs `dbg->last_*`；`ctrl->stop_reason` vs 运行时真实原因 |

### 2.2 严重：协议契约漂移

| ID | 问题 | 证据 |
|----|------|------|
| DAP-07 | 宣称 `supportsSteppingGranularity` 但不解析 `granularity` | `handle_initialize()` 设置了 flag，handler 中忽略参数 |
| DAP-08 | 宣称 `supportsExceptionFilterOptions/Options` 但不解析 | 同上 |
| DAP-09 | 异常断点没进 VM | `xr_debug_on_exception()` 只在 `xdap_debug.c` 定义，VM/运行时无调用 |
| DAP-10 | 函数断点/logpoint 没进 VM | 逻辑在 `xr_debug_on_line()` 中，而该函数未被调用 |
| DAP-11 | `exceptionInfo` 请求不存在 | 协议要求但未实现 |

### 2.3 重要：功能缺口

| ID | 问题 |
|----|------|
| DAP-12 | 协程枚举只覆盖 current + ready，缺 blocked/sleep/select/channel wait |
| DAP-13 | `xdap_find_coro()` 失败时静默回退主协程 |
| DAP-14 | `stackTrace` 不支持 `startFrame/levels` 分页 |
| DAP-15 | `scopes` 只有 Locals，缺 Globals/Closure/Upvalues |
| DAP-16 | 数组变量子项固定最多 100 个，无分页 |
| DAP-17 | `evaluate` 不支持全局/闭包/upvalue 查找 |
| DAP-18 | `setVariable` 只能改 Locals 的标量，不能改字段/索引 |
| DAP-19 | resume 结果用布尔值，debug break/yield/blocked 混为一类 |
| DAP-20 | TCP accept 阻塞；stdio 只设了 read_fd 非阻塞；写缓冲排空被动 |

### 2.4 架构

| ID | 问题 |
|----|------|
| DAP-21 | `xdap_debug.c` 1926 行，承担 10+ 种职责 |
| DAP-22 | `xdap_protocol.c` 1018 行，handler + run loop + dispatch 全在一起 |
| DAP-23 | `xray_debug_hooks.h` 接口太弱，做不了 pause/logpoint/function bp/exception |

---

## 3. 重写设计

### 3.1 新的 debug hook 接口

**现状问题**：当前 `xray_debug_hooks.h` 有 7 个零散回调，做不了 pause、logpoint、function breakpoint、exception breakpoint。DAP 只能自己另写一套 `xr_debug_on_line()` 但又没人调用。

**新设计**：VM 在安全点（行变化时）只调一次 `on_line`，由 hook 实现决定全部行为。

```c
/* src/runtime/xray_debug_hooks.h — complete redesign */

typedef enum {
    XR_DBG_ACTION_CONTINUE = 0,
    XR_DBG_ACTION_BREAK,
    XR_DBG_ACTION_STEP_IN,
    XR_DBG_ACTION_STEP_OUT,
    XR_DBG_ACTION_STEP_OVER
} XrDebugAction;

typedef struct XrDebugHooks {
    /* Called by VM at each line change (safe point).
     * Returns action: continue, break, or step variant.
     * This ONE callback replaces:
     *   check_breakpoint + on_breakpoint_hit + get_action +
     *   get_step_depth + set_last_line + get_last_line
     * Hook impl does: breakpoint check, step check, pause check,
     * logpoint output, function bp check — all in one call. */
    XrDebugAction (*on_line)(XrayIsolate *isolate, const char *path,
                             int line, XrClosure *closure,
                             XrBcCallFrame *frame);

    /* Called by VM/runtime when exception is thrown.
     * Returns: BREAK to stop, CONTINUE to propagate. */
    XrDebugAction (*on_exception)(XrayIsolate *isolate, const char *message,
                                  bool is_uncaught);

    /* Quick check — allows VM to skip on_line entirely when no debugger. */
    bool (*is_enabled)(XrayIsolate *isolate);
} XrDebugHooks;

XR_FUNC void xr_debug_register_hooks(XrayIsolate *isolate, XrDebugHooks *hooks);
XR_FUNC XrDebugHooks *xr_debug_get_hooks(XrayIsolate *isolate);
```

**好处**：
- VM 侧极简：`if (hooks && hooks->is_enabled(isolate)) action = hooks->on_line(...)`
- DAP 侧**完全控制**：breakpoint / step / pause / logpoint / function bp 全在 `on_line` 实现内决定
- 异常断点通过 `on_exception` 独立闭环
- 零开销原则不变：无 debugger 时 hooks 为 NULL

**VM 侧改动** (`src/vm/xvm.c`)：

```c
/* Replace VM_DEBUG_CHECK macro — much simpler */
#define VM_DEBUG_CHECK() do { \
    XrDebugHooks *_hooks = (XrDebugHooks *)isolate->debug_hooks; \
    if (_hooks && _hooks->is_enabled && _hooks->is_enabled(isolate)) { \
        int _line = /* ... get line from pc ... */; \
        const char *_path = /* ... get source path ... */; \
        if (_line > 0) { \
            XrDebugAction _act = _hooks->on_line(isolate, _path, _line, cl, ci); \
            if (_act == XR_DBG_ACTION_BREAK) { \
                ci->pc = pc - 1; \
                return XR_VM_DEBUG_BREAK; \
            } \
        } \
    } \
} while(0)
```

### 3.2 单一 stop state

删除三头管理，引入唯一权威停止状态：

```c
/* In xdap_controller.h */
typedef enum {
    XDAP_STOP_NONE = 0,
    XDAP_STOP_BREAKPOINT,
    XDAP_STOP_STEP,
    XDAP_STOP_PAUSE,
    XDAP_STOP_ENTRY,
    XDAP_STOP_EXCEPTION,
    XDAP_STOP_FUNCTION_BREAKPOINT,
    XDAP_STOP_GOTO
} XdapStopReason;

typedef struct XdapStopState {
    XdapStopReason reason;
    XrCoroutine *coro;
    int thread_id;
    const char *path;
    int line;
    bool exception_is_uncaught;
    char *exception_message;
} XdapStopState;
```

**规则**：
- 只有一个 helper 写入：`xdap_record_stop()` / `xdap_clear_stop()`
- `stopped` 事件只读这份状态
- `stackTrace/variables/evaluate` 只用这份状态里的 coro/thread_id
- 不再有 `ctrl->stopped_*` vs `dbg->last_*` 双源

### 3.3 resume 分类

```c
typedef enum {
    XDAP_RESUME_STOPPED,     /* debug break — send stopped event */
    XDAP_RESUME_YIELDED,     /* coroutine yield — stay running */
    XDAP_RESUME_BLOCKED,     /* channel/timer/select — stay running */
    XDAP_RESUME_TERMINATED,  /* program ended */
    XDAP_RESUME_ERROR        /* runtime error */
} XdapResumeResult;
```

不再用 `bool still_running` 压平所有状态。

### 3.4 `xray_free` 修正

全局搜索确认范围极小（4 处调用，2 个文件）：

- `include/xray.h` 中 `xray_free()` 声明：**删除**（用 `xray_isolate_delete()` 替代）
- `src/api/xruntime.c` 中 `xray_free()` 实现：**删除**
- `src/app/dap/xdap_controller.c` 中 3 处调用：**改为 `xray_isolate_delete()`**

一次性修完，不留兼容层。

### 3.5 新文件结构

```text
src/app/dap/
├── xdap_protocol.c             # 薄入口：dispatch table + run loop
├── xdap_protocol.h
├── xdap_session.c              # initialize/launch/configurationDone/restart/terminate/disconnect
├── xdap_session.h
├── xdap_execution.c            # continue/step/pause + stop state + resume classification
├── xdap_execution.h
├── xdap_breakpoints.c          # line bp / conditional / function bp / logpoint / exception bp
├── xdap_breakpoints.h
├── xdap_threads.c              # coroutine 枚举 + threadId 映射
├── xdap_threads.h
├── xdap_variables.c            # scopes / variables / paging / setVariable
├── xdap_variables.h
├── xdap_eval.c                 # 表达式求值（保留，增强）
├── xdap_inspect.c              # stackTrace / disassemble
├── xdap_inspect.h
├── xdap_transport.c            # stdio/TCP transport（保留，修非阻塞）
└── xdap_transport.h
```

**关键决策**：
- 不引入 `xdap_runtime_bridge`——DAP 在 L8，直接 include 低层内部头是架构允许的
- 通过 include 纪律保证边界：每个 `.c` 的 include 列表有明确职责分组
- `xdap_debug.c` 和 `xdap_controller.c` 不保留——职责完全拆散到新文件

### 3.6 删除清单

直接删除，不迁移：

| 文件/函数 | 原因 |
|-----------|------|
| `xr_debug_on_line()` | 从未被调用，1000+ 行；逻辑在新 hook `on_line` 实现中从头写 |
| `xdap_on_stopped()` | 从未被调用；逻辑被 `xdap_record_stop()` 取代 |
| `ctrl->step_mode` | 与 debug state 重复的意图字段 |
| `ctrl->stopped_*` 系列 | 被 `XdapStopState` 统一取代 |
| `xray_free()` API | 语义冲突，全局删除 |
| `xdap_debug.c` (整个文件) | 1926 行大杂烩，职责拆散到新文件 |
| `xdap_controller.c` (整个文件) | 职责拆散到 `xdap_session.c` + `xdap_execution.c` |
| 旧 `xray_debug_hooks.h` 的 7 个零散回调 | 被新的 3 回调接口取代 |

---

## 4. 分阶段实施

### 总览

| Phase | 目标 | 预估 | 核心动作 |
|-------|------|------|----------|
| **A** | 基础设施 | 1-2 天 | 修 `xray_free`；重写 debug hook 接口；建回归测试框架 |
| **B** | 核心重写 | 3-5 天 | 删除死代码；按新结构实现全部核心功能；capability 只宣称已实现的 |
| **C** | 高级能力 | 2-3 天 | 异常/函数断点/logpoint 闭环；evaluate/setVariable 扩展；完整协程枚举 |

---

### Phase A — 基础设施

#### A.1 全局修正 `xray_free` 语义冲突

涉及文件：

- `include/xray.h`
- `src/api/xruntime.c`
- `src/app/dap/xdap_controller.c`

改动：

1. 从 `include/xray.h` 删除 `xray_free()` 声明
2. 从 `src/api/xruntime.c` 删除 `xray_free()` 实现
3. `xdap_controller.c` 中 3 处 `xray_free(ctrl->isolate)` / `xray_free(old_isolate)` → `xray_isolate_delete()`
4. 确认全局无其他调用点（已验证：只有这 4 处）

#### A.2 重写 `xray_debug_hooks.h`

涉及文件：

- `src/runtime/xray_debug_hooks.h` — 全部重写为 3.1 节新接口
- `src/runtime/xray_debug_hooks.c` — 对应实现更新
- `src/vm/xvm.c` — `VM_DEBUG_CHECK` 宏简化为单次 `on_line` 调用

要求：

- 旧的 7 个回调全部删除，不保留
- `on_line` 单回调承载 breakpoint + step + pause + logpoint + function bp
- `on_exception` 回调承载 caught/uncaught exception breakpoint
- `is_enabled` 保留作为快速跳过检查
- VM 侧 `VM_DEBUG_CHECK` 宏行数从 ~40 行缩减到 ~10 行

#### A.3 建立 DAP 回归测试框架

涉及文件：

- 新增 `scripts/run_dap_regression_tests.sh`
- 新增 `tests/regression/dap/` 目录
- 修改 `tests/unit/CMakeLists.txt`

方案：**stdio transcript driver**

- 启动 DAP server 子进程
- 发送标准 JSON-RPC 请求
- 校验响应和事件的结构与顺序

Phase A 至少覆盖：

- `initialize` → capability snapshot
- `launch` → `configurationDone` → `stopOnEntry`
- `continue` → 正常退出
- `disconnect`

#### A.4 验收标准

- [ ] `xray_free()` 从全代码库消失
- [ ] `xray_debug_hooks.h` 只有 3 个回调
- [ ] `VM_DEBUG_CHECK` 宏调用 `on_line`
- [ ] DAP 回归框架可运行至少 4 个基础 transcript
- [ ] `cmake --build build -j8 && ctest --output-on-failure --test-dir build` 通过
- [ ] `scripts/run_regression_tests.sh` 通过

---

### Phase B — 核心重写

#### B.1 删除旧文件

- 删除 `xdap_debug.c`（1926 行）
- 删除 `xdap_controller.c`（366 行）
- 保留 `xdap_eval.c`、`xdap_inspect.c`、`xdap_transport.c`（后续增强）

#### B.2 新建核心文件

按 3.5 节结构创建新文件，从头实现：

##### `xdap_session.c` — 会话生命周期

- `handle_initialize()` — capability 只写真实支持的项
- `handle_launch()` — isolate 创建、debug hook 注册、`stopOnEntry` 通过新 hook 真实 arm
- `handle_configuration_done()` — 触发首次执行
- `handle_restart()` — `xray_isolate_delete()` 旧实例 → 创建新实例
- `handle_terminate()` / `handle_disconnect()` — 干净退出

##### `xdap_execution.c` — 执行控制与 stop state

- `XdapStopState` 唯一权威源
- `xdap_record_stop()` / `xdap_clear_stop()` — 单一写入点
- `handle_continue()` / `handle_next()` / `handle_step_in()` / `handle_step_out()` — 直接修改 debug state
- `handle_pause()` — 设置 pause 标志（由 `on_line` hook 消费）
- `XdapResumeResult` 分类 — 替代布尔值

##### `xdap_breakpoints.c` — 断点管理

- 行断点 hash table（保留原有 O(1) 查找设计）
- 条件断点 + 命中计数
- logpoint（通过 `on_line` hook 在命中时发 `output` event，不中断）
- 函数断点（通过函数名匹配）
- 异常断点配置（caught / uncaught filters）

##### `xdap_threads.c` — 协程/线程映射

- `threadId ↔ coroutine` 双向映射
- 枚举当前 + ready + blocked/sleep/select 等待态
- 查找失败返回错误，**不再静默回退主协程**

##### `xdap_variables.c` — 变量与 scope

- `Locals` / `Globals` / `Closure/Upvalues` 三种 scope
- varRef 元数据：`{ kind, coro, frame_idx, parent, ... }`
- `start/count` 分页（不再硬截断 100 项）
- `namedVariables` / `indexedVariables` 正确设置
- `setVariable` 支持 Locals + 对象字段 + 数组索引

##### `xdap_protocol.c` — 精简

- 只保留：消息解包 + dispatch table + run loop + 事件发送
- 所有 handler 实现移到对应模块
- capability 生成由 `xdap_session.c` 负责

#### B.3 hook `on_line` 实现

这是整个重写的核心——把所有 line-level 调试逻辑收敛到一个函数中：

```c
/* In xdap_execution.c (or xdap_breakpoints.c) */
static XrDebugAction dap_on_line(XrayIsolate *isolate, const char *path,
                                  int line, XrClosure *closure,
                                  XrBcCallFrame *frame) {
    XdapController *ctrl = get_controller(isolate);
    XrDebugState *dbg = get_debug_state(isolate);

    /* 1. Pause check (highest priority) */
    if (atomic_load(&ctrl->pause_requested)) {
        atomic_store(&ctrl->pause_requested, false);
        xdap_record_stop(ctrl, XDAP_STOP_PAUSE, ...);
        return XR_DBG_ACTION_BREAK;
    }

    /* 2. Breakpoint check */
    XdapBreakpoint *bp = xdap_find_breakpoint(dbg, path, line);
    if (bp) {
        if (bp->log_message) {
            /* Logpoint: send output event, don't break */
            xdap_send_output_event(ctrl, expand_logpoint(bp, frame));
            return XR_DBG_ACTION_CONTINUE;
        }
        if (evaluate_condition(bp, frame)) {
            xdap_record_stop(ctrl, XDAP_STOP_BREAKPOINT, ...);
            return XR_DBG_ACTION_BREAK;
        }
    }

    /* 3. Function breakpoint check */
    if (xdap_check_function_breakpoint(dbg, closure)) {
        xdap_record_stop(ctrl, XDAP_STOP_FUNCTION_BREAKPOINT, ...);
        return XR_DBG_ACTION_BREAK;
    }

    /* 4. Step check */
    XrDebugAction step_result = xdap_check_step(dbg, path, line, frame);
    if (step_result == XR_DBG_ACTION_BREAK) {
        xdap_record_stop(ctrl, XDAP_STOP_STEP, ...);
        return XR_DBG_ACTION_BREAK;
    }

    return XR_DBG_ACTION_CONTINUE;
}
```

**这一个函数取代了旧的**：
- VM 里 ~40 行 `VM_DEBUG_CHECK` 宏中的逻辑
- `xr_debug_on_line()` 中 ~200 行从未被调用的逻辑
- `check_breakpoint` / `on_breakpoint_hit` / `get_action` / `get_step_depth` 四个分散回调

#### B.4 `on_exception` 实现

```c
static XrDebugAction dap_on_exception(XrayIsolate *isolate, const char *message,
                                       bool is_uncaught) {
    XdapController *ctrl = get_controller(isolate);
    XrDebugState *dbg = get_debug_state(isolate);

    if ((is_uncaught && dbg->break_on_uncaught) ||
        (!is_uncaught && dbg->break_on_caught)) {
        xdap_record_stop(ctrl, XDAP_STOP_EXCEPTION, ...);
        /* Save exception info for exceptionInfo request */
        ctrl->stop.exception_message = xr_strdup(message);
        ctrl->stop.exception_is_uncaught = is_uncaught;
        return XR_DBG_ACTION_BREAK;
    }
    return XR_DBG_ACTION_CONTINUE;
}
```

#### B.5 transport 修正

保留 `xdap_transport.c`，修以下问题：

- TCP server `accept()` → 非阻塞 + poll
- stdio `write_fd` → 也设非阻塞
- 写缓冲 → 主循环每 tick 主动 drain，不只靠 `write()` / `try_read()` 触发

#### B.6 capability 诚实

Phase B 完成后 `initialize` response 只包含：

```text
✅ supportsConfigurationDoneRequest
✅ supportsConditionalBreakpoints
✅ supportsHitConditionalBreakpoints
✅ supportsSetVariable          (Locals + 字段/索引)
✅ supportsTerminateRequest
✅ supportsRestartRequest
✅ supportsDisassembleRequest
✅ supportsPauseRequest         (新：真正接线)
✅ supportsStopOnEntry          (新：真正接线)
```

Phase B 中**不宣称**（等 Phase C 闭环后再开启）：

```text
❌ supportsFunctionBreakpoints  → Phase C
❌ supportsLogPoints            → Phase C
❌ supportsExceptionFilterOptions → Phase C
❌ supportsExceptionOptions     → Phase C
❌ supportsSteppingGranularity  → Phase C (if needed)
❌ exceptionInfo request        → Phase C
```

#### B.7 验收标准

- [ ] `xdap_debug.c` 和 `xdap_controller.c` 不再存在
- [ ] 新文件结构按 3.5 节落地
- [ ] `stopOnEntry` 真实生效
- [ ] `pause` 在运行态下能在安全点停住
- [ ] `continue/next/stepIn/stepOut` 的 stopped 事件原因正确
- [ ] `restart/disconnect/terminate` 无 crash/leak（ASan 验证）
- [ ] `stackTrace/scopes/variables` 基本功能正常
- [ ] resume 使用 `XdapResumeResult` 分类
- [ ] transport TCP accept 非阻塞
- [ ] 所有现有回归测试通过
- [ ] 补充 DAP transcript 回归覆盖基础流

---

### Phase C — 高级能力

#### C.1 异常断点闭环

涉及文件：

- VM/运行时异常抛出路径 — 调用 `hooks->on_exception()`
- `xdap_breakpoints.c` — `on_exception` 实现
- `xdap_session.c` — `handle_set_exception_breakpoints()`
- `xdap_protocol.c` — 新增 `exceptionInfo` handler

要求：

- caught / uncaught 可分别配置
- `exceptionInfo` 返回权威 stop snapshot 中的异常信息
- capability: 开启 `supportsExceptionFilterOptions`

#### C.2 函数断点闭环

涉及文件：

- `xdap_breakpoints.c` — 函数名匹配逻辑
- `xdap_execution.c` — `on_line` 中 function bp 检查

要求：

- 能按函数名设置断点
- 在函数首行触发
- capability: 开启 `supportsFunctionBreakpoints`

#### C.3 logpoint 闭环

涉及文件：

- `xdap_breakpoints.c` — logpoint 模板解析与展开
- `xdap_execution.c` — `on_line` 中 logpoint 触发 `output` event

要求：

- logpoint 命中时发送 `output` event，不中断执行
- 支持 `{expression}` 模板语法
- capability: 开启 `supportsLogPoints`

#### C.4 evaluate 扩展

涉及文件：

- `xdap_eval.c`

要求：

- 支持 locals → upvalues → globals 的查找链
- `watch` / `repl` / `hover` context 区分
- 继续禁止带副作用的函数调用（短期）

#### C.5 setVariable 扩展

涉及文件：

- `xdap_variables.c`

要求：

- 支持对象字段修改
- 支持数组元素修改
- 支持 map 项修改
- 不支持的场景返回清晰错误

#### C.6 协程完整枚举

涉及文件：

- `xdap_threads.c`

要求：

- 枚举覆盖 blocked / sleep / timer / select / channel wait 等待态
- 每种状态有明确标签
- 非主协程 stackTrace / variables 正确

#### C.7 stackTrace / variables 分页

涉及文件：

- `xdap_inspect.c`
- `xdap_variables.c`

要求：

- `stackTrace(startFrame, levels)` 正确分页
- `variables(start, count)` 正确分页
- `namedVariables` / `indexedVariables` 正确设置

#### C.8 steppingGranularity (可选)

若有需求：

- 解析 `stepIn/next/stepOut` 请求中的 `granularity`
- 支持 `statement` / `line` / `instruction` 粒度
- capability: 开启 `supportsSteppingGranularity`

若无明确需求，不实现。

#### C.9 验收标准

- [ ] caught exception breakpoint 命中时停住
- [ ] uncaught exception breakpoint 命中时停住
- [ ] `exceptionInfo` 返回异常消息和 uncaught 标志
- [ ] function breakpoint 在函数首行触发
- [ ] logpoint 输出到 `output` event 且不中断
- [ ] evaluate 能访问 globals 和 upvalues
- [ ] setVariable 能修改对象字段和数组元素
- [ ] thread list 显示 blocked/sleep 等等待态
- [ ] stackTrace / variables 分页正确
- [ ] 每个新能力开启后对应 capability 加入 initialize response
- [ ] 每个新能力有 transcript 回归测试

---

## 5. 文件变更总览

### 5.1 删除

| 文件 | Phase | 原因 |
|------|-------|------|
| `src/app/dap/xdap_debug.c` | B | 1926 行大杂烩，职责拆散到新文件 |
| `src/app/dap/xdap_debug.h` | B | 配合 .c 删除 |
| `src/app/dap/xdap_controller.c` | B | 职责拆散到 session + execution |
| `src/app/dap/xdap_controller.h` | B | 配合 .c 删除（公共类型移到新头文件） |
| `include/xray.h` 中 `xray_free()` | A | 语义冲突 API |
| `src/api/xruntime.c` 中 `xray_free()` | A | 语义冲突实现 |

### 5.2 重写

| 文件 | Phase | 改动 |
|------|-------|------|
| `src/runtime/xray_debug_hooks.h` | A | 7 回调 → 3 回调 |
| `src/runtime/xray_debug_hooks.c` | A | 对应实现 |
| `src/vm/xvm.c` | A | `VM_DEBUG_CHECK` 简化 |
| `src/app/dap/xdap_protocol.c` | B | 只保留 dispatch + run loop |
| `src/app/dap/xdap_transport.c` | B | 修非阻塞 |

### 5.3 新增

| 文件 | Phase | 说明 |
|------|-------|------|
| `src/app/dap/xdap_session.[ch]` | B | 会话生命周期 |
| `src/app/dap/xdap_execution.[ch]` | B | 执行控制 + stop state |
| `src/app/dap/xdap_breakpoints.[ch]` | B | 所有断点类型 |
| `src/app/dap/xdap_threads.[ch]` | B | 协程枚举 + 线程映射 |
| `src/app/dap/xdap_variables.[ch]` | B | scopes + variables + paging |
| `scripts/run_dap_regression_tests.sh` | A | 回归入口 |
| `tests/regression/dap/` | A-C | transcript 回归 |
| `tests/unit/dap/test_dap_execution.c` | B | stop state / resume |
| `tests/unit/dap/test_dap_threads.c` | B/C | 协程映射 |
| `tests/unit/dap/test_dap_variables.c` | B/C | 分页 / setVariable |
| `tests/unit/dap/test_dap_exception.c` | C | 异常断点 |

### 5.4 修改（非重写）

| 文件 | Phase | 改动 |
|------|-------|------|
| `src/app/dap/xdap_eval.c` | C | 增加 globals/upvalues 查找 |
| `src/app/dap/xdap_inspect.c` | B/C | stackTrace 分页 |
| `src/coro/xworker_sysmon.c` | B | resume 结果适配 XdapResumeResult |
| `tests/unit/CMakeLists.txt` | A-C | 新测试目标 |
| `CMakeLists.txt` | B | 新源文件 |
| `docs/engineering/README.md` | now | 入口已添加 |

---

## 6. 验证基线

每个 Phase 完成后执行：

```bash
cmake --build build -j8
ctest --output-on-failure --test-dir build
scripts/run_regression_tests.sh
scripts/check_architecture.sh
scripts/run_dap_regression_tests.sh
```

Phase A 和 B 建议增加 ASan 验证（lifecycle 和 stop/resume 路径）：

```bash
cmake -B build-asan -DCMAKE_BUILD_TYPE=Debug -DENABLE_ASAN=ON -DBUILD_TESTS=ON
cmake --build build-asan -j8
ctest --output-on-failure --test-dir build-asan
```

---

## 7. 风险与缓解

| 风险 | 缓解 |
|------|------|
| debug hook 重写影响 VM 性能 | `is_enabled` 快速跳过；无 debugger 时 hooks=NULL，与原来一样零开销 |
| `xray_free` 删除可能有遗漏调用点 | 已验证只有 4 处；删除后编译即可发现遗漏 |
| `on_exception` hook 需要 VM/运行时异常路径配合 | Phase B 先注册回调；Phase C 接入真实异常路径 |
| 新文件结构的 CMake 接入 | 一次性配置，风险低 |
| transcript 测试框架的初始投入 | 框架本身很薄（shell + expect/diff），核心是 JSON fixture |

---

## 8. 推荐实施顺序

```text
Phase A (1-2 天)
  ├─ A.1  删除 xray_free()（全局 4 处）
  ├─ A.2  重写 xray_debug_hooks.h + VM_DEBUG_CHECK
  ├─ A.3  建立 transcript 测试框架 + 4 个基础 fixture
  └─ A.4  验收：构建 + 测试 + ASan

Phase B (3-5 天)
  ├─ B.1  删除 xdap_debug.c + xdap_controller.c
  ├─ B.2  新建 session / execution / breakpoints / threads / variables
  ├─ B.3  实现 on_line hook（核心！）
  ├─ B.4  实现 on_exception hook（桩）
  ├─ B.5  修 transport 非阻塞
  ├─ B.6  capability 诚实化
  └─ B.7  验收：构建 + 全量测试 + ASan + transcript

Phase C (2-3 天)
  ├─ C.1  异常断点接入 VM 异常路径
  ├─ C.2  函数断点闭环
  ├─ C.3  logpoint 闭环
  ├─ C.4  evaluate 扩展
  ├─ C.5  setVariable 扩展
  ├─ C.6  协程完整枚举
  ├─ C.7  stackTrace/variables 分页
  └─ C.8  验收：构建 + 全量测试 + ASan + transcript
```

---

## 9. 结论

`src/app/dap` 当前的核心问题是 **写了功能但没接线**，而不是缺少功能。

按 Xray 的开发原则：

- **不在错误 hook 上打补丁** → 重新设计 `xray_debug_hooks.h`
- **不迁移死代码** → 删除 `xr_debug_on_line()` 和 `xdap_on_stopped()`，从头写
- **不做兼容层** → 删除 `xray_free()`，不引入 bridge
- **不分 7 个 phase** → 3 个 Phase，每个都选最佳设计

3 个 Phase 完成后，DAP 模块将是：

- ✅ 生命周期正确（`xray_isolate_delete()` 唯一析构路径）
- ✅ 停止链路闭环（单一 `XdapStopState`，单一 `on_line` hook）
- ✅ 协议契约诚实（只宣称已实现的 capability）
- ✅ 代码结构清晰（每文件单一职责，无 1900+ 行大杂烩）
- ✅ 有回归测试保护（transcript + unit + ASan）
