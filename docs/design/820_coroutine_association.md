# 820 协程关联机制详细实施方案

## 实现状态

| 功能 | 状态 | Commit |
|------|------|--------|
| Phase 1: 前缀修饰符语法 (linked/monitored/supervisor) | ✅ 完成 | `1c2b3a4a` |
| Phase 2A: linked/supervisor scope 运行时错误传播 | ✅ 完成 | `67778ca1` |
| Phase 2B: linked go 表达式语法 + Task parent-child | ✅ 完成 | `18fa7c0b` |
| Phase 2C: task.monitor() API + CompletionNode CHANNEL | ✅ 完成 | `694cedc2` |
| Phase 3B: linked scope 兄弟取消 | ✅ 完成 | `5131bfdd` |
| Phase 4A: supervisor scope errors[] 收集 | ✅ 完成 | `bfa9e467` |
| Phase 4B: task.link() / task.unlink() API | ✅ 完成 | `4351ec3a` |
| Phase 4C: JIT 适配验证 | ✅ 完成 | JIT 不涉及协程指令，262/266 无新回退 |
| 审计修复: GC/枚举/monitor/monitored go 清理 | ✅ 完成 | `6e19215a`..`248f7e3b` |

**实际实现与设计文档的差异**：
- linked go 的关联信息存储在 `XrTask.link_mode` 而非 `XrCoroExt.links` 链表
- 兄弟取消通过 `ScopeContext.first_child` 链表实现（非 link 链表）
- `task.monitor()` 返回 buffered(1) Channel，事件为 task 本身（接收方用 .done/.result/.error 检查）
- scope 内子协程通过 scope 机制传播错误，不走 Task parent-child 层级
- `supervisor scope` 可作为表达式使用，返回 errors[] 数组
- `task.link()` 使用 `XrTaskLink` 链表（非 `XrCoroExt.links`），支持 already-failed 检测
- `monitored go` 前缀已删除，用 `go` + `task.monitor()` 替代（更简洁）
- `XrLinkMode` / `XrScopeMode` 枚举定义在 `xtask.h`（替代魔法数字）
- `coro->result/error` 保留：executor 级结果用于 vm_await deep copy，与 task->result/error 互补

## 1. 设计目标

为 xray 协程系统添加显式的关联机制（link / monitor），实现：

- **默认独立**：`go fn()` 行为不变，协程之间零耦合
- **显式关联**：通过前缀修饰符声明关联策略
- **Channel 统一**：monitor 通知通过 Channel 投递，复用现有 select 语法
- **cluster 一致**：本地 monitor 和远程 monitor 语义完全相同
- **零开销**：不使用关联功能时，无任何运行时成本

## 2. 语法规范

### 2.1 go 前缀修饰符

```
go_stmt := link_modifier? 'go' go_options? (call_expr | block)
link_modifier := 'linked' | 'monitored'
go_options := '(' option_list ')'
option_list := option (',' option)*
option := IDENT ':' expr          // name: "xxx", priority: .high
```

**示例**：

```javascript
go fn()                                       // 独立（默认）
go(name: "w1") fn()                           // 独立，带名字
go { block }                                  // 独立，匿名块

linked go fn()                                // 双向关联
monitored go fn()                             // 单向监控

linked go(name: "db", priority: .high) query(sql)   // 完整组合
```

**语义定义**：

| 修饰符 | 语义 | 实现 |
|--------|------|------|
| (无) | 独立，fire-and-forget | 现有行为不变 |
| `linked` | 双向错误传播：子 panic → 父收到 LinkError；父 cancel → 子 cancel | XrCoroutine.links 链表 |
| `monitored` | 单向完成通知：子完成/失败 → 通过 Channel 通知父 | 复用 xcoro_registry 的 monitor 机制 |

### 2.2 scope 前缀修饰符

```
scope_stmt := scope_modifier? 'scope' scope_options? block
scope_modifier := 'linked' | 'supervisor'
scope_options := '(' option_list ')'
```

**示例**：

```javascript
scope { ... }                                 // 等待屏障（现有行为）
linked scope { ... }                          // 子失败 → 取消全部 → scope 抛错
supervisor scope { ... }                      // 子失败 → 不影响兄弟 → 收集错误
```

**语义定义**：

| 修饰符 | scope 内 go 的默认行为 | 子失败时 | scope 出口 |
|--------|----------------------|---------|-----------|
| (无) | 独立 | 静默 | 等待全部完成 |
| `linked` | 自动 linked | 取消所有兄弟 + scope 抛错 | 等待全部完成或首个失败 |
| `supervisor` | 自动 monitored | 不影响兄弟 | 等待全部完成，收集错误数组 |

### 2.3 运行时 API（Phase 2）

```javascript
// Task 实例方法
let ch = task.monitor()           // 返回通知 Channel
task.link(other)                  // 双向关联两个 task
task.unlink(other)                // 解除关联

// 当前协程
Task.self.monitor(someTask)       // 当前协程监控某 task
Task.self.link(someTask)          // 当前协程与某 task 双向关联
```

### 2.4 select 语法兼容

**不需要任何 select 语法改动**。monitor 返回 Channel，用现有语法接收：

```javascript
let t = go compute(42)
let mon = t.monitor()

select {
    event from mon => {               // 复用现有 "msg from ch" 语法
        if (event.ok) {
            print("result: " + event.value)
        } else {
            print("error: " + event.error)
        }
    }
    msg from data_ch => { ... }       // 其他 channel 照常
    after 5000 => { ... }             // 超时照常
}
```

## 3. 数据结构设计

### 3.1 关联模式枚举

```c
// xcoroutine.h
typedef enum {
    XR_LINK_NONE      = 0,  // 默认独立
    XR_LINK_LINKED    = 1,  // 双向错误传播
    XR_LINK_MONITORED = 2,  // 单向完成通知
} XrLinkMode;

typedef enum {
    XR_SCOPE_WAIT       = 0,  // 等待屏障（现有行为）
    XR_SCOPE_LINKED     = 1,  // 子失败 → 取消全部 → scope 抛错
    XR_SCOPE_SUPERVISOR = 2,  // 子失败 → 不影响兄弟 → 收集错误
} XrScopeMode;
```

### 3.2 XrCoroutine 新增字段

```c
// xcoroutine.h — XrCoroExt（按需分配，零开销）
typedef struct XrCoroExt {
    // ... 现有字段 ...
    struct XrCoroMonitor *watched_by;    // [已有] 生命周期监控列表

    /* === Link Association (Phase 1) === */
    struct XrLinkEntry *links;           // 双向关联链表头
    XrLinkMode spawn_link_mode;          // spawn 时声明的关联模式
    struct XrCoroutine *link_parent;     // linked go 的父协程（快捷指针）
} XrCoroExt;
```

### 3.3 Link 链表节点

```c
// xcoro_link.h (新文件)
typedef struct XrLinkEntry {
    struct XrCoroutine *peer;     // 关联对端
    struct XrLinkEntry *next;     // 链表
} XrLinkEntry;
```

### 3.4 XrScopeContext 扩展

```c
// xcoroutine.h — 现有 XrScopeContext
typedef struct XrScopeContext {
    _Atomic int count;                // [已有] 子协程计数
    struct XrScopeContext *parent;    // [已有] 嵌套 scope 链
    uint8_t mode;                     // [新增] XrScopeMode
    _Atomic bool cancel_requested;    // [新增] linked scope: 子失败时设置
    XrValue first_error;              // [新增] linked scope: 首个错误
    struct XrArray *errors;           // [新增] supervisor scope: 错误收集数组
} XrScopeContext;
```

### 3.5 AST 节点扩展

```c
// xast_nodes.h
typedef struct GoExprNode {
    AstNode *expr;
    const char *name;
    AstNode *priority;
    uint8_t link_mode;          // [新增] 0=NONE, 1=LINKED, 2=MONITORED
} GoExprNode;

typedef struct ScopeBlockNode {
    AstNode *body;
    uint8_t scope_mode;         // [新增] 0=WAIT, 1=LINKED, 2=SUPERVISOR
} ScopeBlockNode;
```

## 4. 分阶段实施计划

### Phase 1：前端 + 字节码（spawn 时关联）

只涉及编译管线，不改 VM 运行时。目标：语法可用，字节码携带关联信息。

#### Step 1.1：上下文关键字识别

**文件**：`src/frontend/parser/xparse_coroutine.c`

**改动**：在 `xr_parse_go_expr` 之前，调用处需要识别前缀修饰符。
实际入口在主解析器的语句分派处。

**逻辑**（伪代码）：

```
// 在主解析器的 statement dispatch 中:
if (current_token == TK_NAME) {
    // 检查是否是 "linked" 或 "monitored" 后跟 "go" 或 "scope"
    if (text == "linked" && peek_next() ∈ {TK_GO, "scope"}) {
        link_mode = XR_LINK_LINKED;
        advance();  // 消费 "linked"
        if (check(TK_GO)) {
            parse_go_expr_with_link_mode(link_mode);
        } else {
            parse_scope_block_with_mode(XR_SCOPE_LINKED);
        }
    }
    else if (text == "monitored" && peek_next() == TK_GO) {
        link_mode = XR_LINK_MONITORED;
        advance();  // 消费 "monitored"
        parse_go_expr_with_link_mode(link_mode);
    }
    else if (text == "supervisor" && peek_next() == "scope") {
        advance();  // 消费 "supervisor"
        parse_scope_block_with_mode(XR_SCOPE_SUPERVISOR);
    }
}
```

**关键点**：`linked`/`monitored`/`supervisor` 是**上下文关键字**（只在 go/scope 前面才是修饰符，其他位置作为普通标识符）。Parser 需要前瞻 1 个 token 来判断。

#### Step 1.2：AST 节点新增字段

**文件**：`src/frontend/parser/xast_nodes.h`

```c
// GoExprNode 新增:
uint8_t link_mode;      // XR_LINK_NONE / XR_LINK_LINKED / XR_LINK_MONITORED

// ScopeBlockNode 新增:
uint8_t scope_mode;     // XR_SCOPE_WAIT / XR_SCOPE_LINKED / XR_SCOPE_SUPERVISOR
```

**文件**：`src/frontend/parser/xast.c`

```c
// xr_ast_go_expr 签名扩展:
AstNode *xr_ast_go_expr(XrayIsolate *X, AstNode *expr,
                         const char *name, AstNode *priority,
                         uint8_t link_mode, int line);

// xr_ast_scope_block 签名扩展:
AstNode *xr_ast_scope_block(XrayIsolate *X, AstNode *body,
                              uint8_t scope_mode, int line);
```

#### Step 1.3：字节码编码

**文件**：`src/frontend/codegen/xstmt_coroutine.c`

在 `compile_go_expr` 中，如果 `link_mode != 0`，在 OP_GO 后追加一条 OP_NOP 编码：

```c
// 现有: OP_NOP A=1 Bx=name_idx  (名字)
// 现有: OP_NOP A=2 Bx=priority  (优先级)
// 新增: OP_NOP A=3 Bx=link_mode (关联模式)
if (node->link_mode != 0) {
    emit_aBx(c->emitter, OP_NOP, 3, node->link_mode);
}
```

在 `compile_scope_block` 中，修改 `OP_SCOPE_ENTER` 的 A 操作数携带 scope_mode：

```c
// 现有: emit_abc(c->emitter, OP_SCOPE_ENTER, 0, 0, 0);
// 改为: emit_abc(c->emitter, OP_SCOPE_ENTER, node->scope_mode, 0, 0);
```

### Phase 2：VM 运行时（关联行为）

#### Step 2.1：linked go — 双向错误传播

**文件**：`src/vm/xvm_cold_paths.c` — `vm_go`

在 coro 创建后，读取 OP_NOP A=3 的 link_mode：

```c
int link_mode = 0;
if (GET_OPCODE(next_inst) == OP_NOP && GETARG_A(next_inst) == 3) {
    link_mode = GETARG_Bx(next_inst);
    pc++;
    next_inst = *pc;
}

if (link_mode == XR_LINK_LINKED) {
    xr_coro_link(parent, coro);  // 建立双向关联
}
```

**文件**：`src/coro/xcoro_link.c`（新文件）

```c
void xr_coro_link(XrCoroutine *a, XrCoroutine *b) {
    // a->ext->links 添加 b
    // b->ext->links 添加 a
    // b->ext->link_parent = a (快捷指针)
}

void xr_coro_unlink(XrCoroutine *a, XrCoroutine *b) {
    // 从双方链表中移除对端
}

// 协程退出时调用（在 xr_coro_on_exit 之后）
void xr_coro_propagate_link_error(XrayIsolate *X, XrCoroutine *coro) {
    XrCoroExt *ext = coro->ext;
    if (!ext || !ext->links) return;

    bool is_error = (coro->error != xr_null());
    bool is_cancelled = xr_coro_is_cancelled(coro);

    XrLinkEntry *entry = ext->links;
    while (entry) {
        XrCoroutine *peer = entry->peer;
        if (is_error || is_cancelled) {
            // 向对端传播：设置 cancel 标志
            xr_coro_cancel(peer);
            // 如果对端有 trap（未来扩展），投递 LinkError
        }
        // 从对端的 links 列表中移除自己
        xr_coro_remove_link_entry(peer, coro);
        XrLinkEntry *next = entry->next;
        xr_free(entry);
        entry = next;
    }
    ext->links = NULL;
}
```

#### Step 2.2：monitored go — 单向通知

**文件**：`src/vm/xvm_cold_paths.c` — `vm_go`

```c
if (link_mode == XR_LINK_MONITORED) {
    // 为父协程创建一个 monitor Channel
    // Channel 引用存在父的 pending_monitor_ch 临时字段
    XrChannel *mon_ch = xr_channel_new(isolate, 1);  // buffered(1)
    XrCoroExt *child_ext = xr_coro_ensure_ext(coro);
    XrCoroMonitor *mon = xr_calloc(1, sizeof(XrCoroMonitor));
    mon->channel = mon_ch;
    mon->next = child_ext->watched_by;
    child_ext->watched_by = mon;

    // mon_ch 作为 monitored go 的第二个返回值
    // 通过 OP_GO 的 result slot+1 返回
    base[a + 1] = xr_value_from_channel(mon_ch);
}
```

**关键**：`monitored go` 的返回值从 1 个变为 2 个：`Task` + `Channel`。

语法层面：

```javascript
let task, mon = monitored go compute(42)

select {
    event from mon => { ... }
}
```

或简化写法（只要 Task，不关心 Channel）：

```javascript
let task = monitored go compute(42)   // 忽略第二个返回值
```

#### Step 2.3：linked scope — 结构化错误传播

**文件**：`src/vm/xvm.c` — `OP_SCOPE_ENTER`

```c
vmcase(OP_SCOPE_ENTER) {
    int scope_mode = GETARG_A(instr);  // 0=WAIT, 1=LINKED, 2=SUPERVISOR
    XrCoroutine *current = (XrCoroutine *)VM_CURRENT_CORO;
    // ... 现有逻辑 ...
    if (scope) {
        scope->mode = (uint8_t)scope_mode;
        atomic_store(&scope->cancel_requested, false);
        scope->first_error = xr_null();
        scope->errors = NULL;
    }
    vmbreak;
}
```

**文件**：`src/vm/xvm.c` — `OP_SCOPE_EXIT`

```c
vmcase(OP_SCOPE_EXIT) {
    XrCoroutine *current = (XrCoroutine *)VM_CURRENT_CORO;
    XrScopeContext *scope = current->current_scope;
    if (!scope) vmbreak;

    if (atomic_load(&current->wait_count) > 0) {
        // 子协程仍在运行 — 阻塞等待
        // ...现有逻辑...
        return XR_VM_BLOCKED;
    }

    // 所有子完成
    if (scope->mode == XR_SCOPE_LINKED) {
        // linked scope: 如果有子失败，scope 抛错
        if (scope->first_error != xr_null()) {
            current->current_scope = scope->parent;
            XrValue err = scope->first_error;
            xr_free(scope);
            // 设置错误并抛出
            VM_THROW_VALUE(frame, pc, err);
        }
    } else if (scope->mode == XR_SCOPE_SUPERVISOR) {
        // supervisor scope: 收集错误到 result
        if (scope->errors) {
            // 将错误数组放到指定寄存器
            base[GETARG_A(instr)] = xr_value_from_array(scope->errors);
        }
    }

    current->current_scope = scope->parent;
    xr_free(scope);
    vmbreak;
}
```

**文件**：`src/coro/xcoro.c` — `xr_coro_wake_waiter`

子协程完成时的回调，需要检查 scope->mode：

```c
void xr_coro_wake_waiter(XrCoroutine *coro) {
    XrScopeContext *scope = coro->parent_scope;
    if (scope) {
        // 检查子协程是否失败
        bool child_failed = (coro->error != xr_null());

        if (child_failed && scope->mode == XR_SCOPE_LINKED) {
            // linked scope: 记录首个错误 + 取消所有兄弟
            if (scope->first_error == xr_null()) {
                scope->first_error = coro->error;
            }
            if (!atomic_exchange(&scope->cancel_requested, true)) {
                // 首次失败：取消 scope 内所有其他子协程
                xr_scope_cancel_all(scope, coro);  // 新函数
            }
        } else if (child_failed && scope->mode == XR_SCOPE_SUPERVISOR) {
            // supervisor scope: 收集错误
            xr_scope_collect_error(scope, coro);  // 新函数
        }

        // 现有逻辑: decrement count, wake waiter...
    }
}
```

#### Step 2.4：Task.monitor() 运行时 API

**文件**：`src/vm/xvm_cold_paths.c` — Task 的 INVOKE/GETPROP handler

Task 已有 `.done`/`.cancelled`/`.result`/`.error`/`.cancel()` 方法。新增：

```c
// task.monitor() — 创建 monitor Channel
case SYM_MONITOR: {
    XrCoroutine *target = task->coro;
    XrChannel *ch = xr_channel_new(isolate, 1);
    if (target && !xr_coro_is_done(target)) {
        // 协程还活着：注册 monitor
        XrCoroExt *ext = xr_coro_ensure_ext(target);
        XrCoroMonitor *mon = xr_calloc(1, sizeof(XrCoroMonitor));
        mon->channel = ch;
        mon->next = ext->watched_by;
        ext->watched_by = mon;
    } else {
        // 协程已完成：立即投递事件
        XrValue event = xr_make_monitor_event(task);
        xr_channel_try_send(ch, event);
    }
    return xr_value_from_channel(ch);
}

// task.link(other) — 双向关联
case SYM_LINK: {
    // 参数检查 + xr_coro_link(self_coro, other_coro)
}
```

#### Step 2.5：monitor 事件格式

monitor Channel 投递的事件值结构：

```javascript
// 成功完成:
{ ok: true, value: <result>, task: <task_ref> }

// 失败:
{ ok: false, error: <error>, task: <task_ref> }

// 取消:
{ ok: false, error: "cancelled", task: <task_ref> }
```

C 层面用 XrMap 或结构化的 JSON 值表示。

**文件**：`src/coro/xcoro_registry.c` — `xr_coro_notify_monitors`

当前实现只发送 reason 字符串。需要扩展为发送结构化事件：

```c
void xr_coro_notify_monitors(XrayIsolate *X, XrCoroRegistry *reg,
                              XrCoroutine *coro, const char *reason) {
    // ... 现有 detach 逻辑 ...

    // 构建结构化事件（替代简单的 reason 字符串）
    XrValue event = xr_make_monitor_event_from_coro(X, coro, reason);

    while (mon) {
        XrCoroMonitor *next = mon->next;
        xr_channel_notify_send(mon->channel, event);
        xr_free(mon);
        mon = next;
    }
}
```

### Phase 3：与 cluster 统一

#### Step 3.1：统一 monitor 事件格式

确保本地和远程 monitor 的事件格式一致：

```
本地:  task.monitor()                    → Channel<MonitorEvent>
远程:  cluster.monitor("n", "coro")      → Channel<MonitorEvent>
节点:  cluster.monitorNode("n")          → Channel<string>  (保持现有)
```

`MonitorEvent` 的 C 表示：

```c
// 用 XrMap 表示（与 JSON 互操作）
XrValue xr_make_monitor_event(XrayIsolate *X, bool ok,
                               XrValue value_or_error,
                               const char *coro_name) {
    XrMap *m = xr_map_new(X, 3);
    xr_map_set(m, xr_string_intern(X, "ok", 2, 0),
               ok ? xr_true() : xr_false());
    if (ok)
        xr_map_set(m, xr_string_intern(X, "value", 5, 0), value_or_error);
    else
        xr_map_set(m, xr_string_intern(X, "error", 5, 0), value_or_error);
    if (coro_name)
        xr_map_set(m, xr_string_intern(X, "name", 4, 0),
                   xr_string_value(xr_string_intern(X, coro_name, strlen(coro_name), 0)));
    return xr_value_from_map(m);
}
```

#### Step 3.2：cluster_monitor.c 适配

当前 `xr_cluster_monitor_coro` 发送的是简单字符串。改为发送相同格式的 MonitorEvent。

## 5. 文件改动清单

### Phase 1（前端语法 + 字节码）

| 文件 | 改动 |
|------|------|
| `src/frontend/parser/xast_nodes.h` | GoExprNode +link_mode, ScopeBlockNode +scope_mode |
| `src/frontend/parser/xast.c` | xr_ast_go_expr / xr_ast_scope_block 签名扩展 |
| `src/frontend/parser/xast_api.h` | 同步声明 |
| `src/frontend/parser/xparse_coroutine.c` | go/scope 前缀修饰符识别 |
| `src/frontend/parser/xparse.c` | 语句分派中增加 linked/monitored/supervisor 前瞻 |
| `src/frontend/codegen/xstmt_coroutine.c` | compile_go_expr 编码 link_mode, compile_scope_block 编码 scope_mode |
| `src/frontend/analyzer/xanalyzer_mono.c` | AST clone 同步新字段 |
| `src/frontend/analyzer/xanalyzer_visitor_expr.c` | monitored go 返回 (Task, Channel) 元组类型 |

### Phase 2（VM 运行时）

| 文件 | 改动 |
|------|------|
| `src/coro/xcoroutine.h` | XrLinkMode/XrScopeMode 枚举, XrCoroExt 新增 links 字段, XrScopeContext 扩展 |
| `src/coro/xcoro_link.h` | 新文件：link/unlink API 声明 |
| `src/coro/xcoro_link.c` | 新文件：link/unlink/propagate 实现 |
| `src/vm/xvm_cold_paths.c` | vm_go 读取 link_mode, 建立 link/monitor |
| `src/vm/xvm.c` | OP_SCOPE_ENTER 读取 scope_mode, OP_SCOPE_EXIT 错误传播/收集 |
| `src/coro/xcoro.c` | xr_coro_wake_waiter 中检查 scope->mode |
| `src/coro/xcoro_registry.c` | xr_coro_notify_monitors 发送结构化事件 |
| `src/coro/xworker.c` | 协程退出路径调用 xr_coro_propagate_link_error |
| `src/vm/xvm_cold_paths.c` | Task INVOKE 新增 monitor()/link() 方法 |

### Phase 3（cluster 统一）

| 文件 | 改动 |
|------|------|
| `stdlib/cluster/cluster_monitor.c` | 事件格式统一为 MonitorEvent |

## 6. 关键设计决策

### 6.1 linked 的错误传播时机

子协程 panic 后，错误何时传播到父？

**方案**：在子协程退出路径中立即设置父的 `cancel_requested` 标志。父在下次 yield 点（reduction check / await / channel op）检查该标志并抛出 `LinkError`。

**理由**：不中断父的当前执行（避免异步异常的复杂性），但在最早的安全点传播。这和 Erlang 的 EXIT 信号语义一致——信号在进程调度时处理。

### 6.2 monitored go 的返回值设计

**方案 A**：多值返回 `let task, ch = monitored go fn()`
**方案 B**：Task 自带 monitor 方法 `let ch = task.monitor()`

**选择：两者都支持**。
- `monitored go` 自动创建 monitor Channel 并作为第二返回值（高性能，无后续分配）
- `task.monitor()` 作为通用 API（灵活，任意时刻调用）

### 6.3 scope 内 go 的默认关联行为

- `scope { go fn() }` — go 保持独立（现有行为）
- `linked scope { go fn() }` — go 自动变为 linked（编译期推断）
- `supervisor scope { go fn() }` — go 自动变为 monitored（编译期推断）

**实现**：编译器在 compile_go_expr 时检查是否在 linked/supervisor scope 内。如果是，且 go 没有显式修饰符，则自动注入对应的 link_mode。

### 6.4 多次 monitor 的行为

每次 `task.monitor()` 创建一个**独立的** Channel。协程完成时向所有注册的 Channel 发送事件。Channel 是 buffered(1)，不会阻塞完成路径。

这和 `xcoro_registry.c` 现有实现完全一致（watched_by 链表，每个 monitor 一个 Channel）。

### 6.5 link 的对称性

`xr_coro_link(A, B)` 在双方都建立 entry。A 退出时传播到 B，B 退出时传播到 A。

如果 A 正常退出（无错误），不触发 B 的 LinkError。只有 panic / cancel 才传播。

这和 Erlang 的 link 行为一致：normal exit 不触发 EXIT 信号。

## 7. 测试计划

### 回归测试文件

```
tests/regression/11_coroutine/1120_linked_go.xr
tests/regression/11_coroutine/1121_monitored_go.xr
tests/regression/11_coroutine/1122_linked_scope.xr
tests/regression/11_coroutine/1123_supervisor_scope.xr
tests/regression/11_coroutine/1124_task_monitor_api.xr
tests/regression/11_coroutine/1125_task_link_api.xr
tests/regression/11_coroutine/1126_monitor_select.xr
tests/regression/11_coroutine/1127_linked_scope_cancel.xr
tests/regression/11_coroutine/1128_supervisor_error_collect.xr
```

### 关键测试场景

**1120 linked go 基础**：
```javascript
@test fn test_linked_go_child_fail() {
    let ok = false
    try {
        linked go { throw new Exception("child error") }
        await Task.sleep(100)   // 等待 link 传播
    } catch (e) {
        ok = true
        assert(e.message.contains("LinkError"))
    }
    assert(ok)
}
```

**1121 monitored go 基础**：
```javascript
@test fn test_monitored_go() {
    let task, mon = monitored go { return 42 }
    select {
        event from mon => {
            assert(event.ok == true)
            assert(event.value == 42)
        }
        after 5000 => { throw new Exception("timeout") }
    }
}
```

**1122 linked scope**：
```javascript
@test fn test_linked_scope() {
    let ok = false
    try {
        linked scope {
            go { return 1 }                    // 正常
            go { throw new Exception("fail") }   // 失败 → 取消兄弟 → scope 抛错
            go { await Task.sleep(10000) }     // 应被取消
        }
    } catch (e) {
        ok = true
    }
    assert(ok)
}
```

**1123 supervisor scope**：
```javascript
@test fn test_supervisor_scope() {
    supervisor scope {
        go { return 1 }
        go { throw new Exception("fail") }   // 失败但不影响兄弟
        go { return 3 }
    }
    // scope 正常退出，errors 可查询
    print("all done")
}
```

**1126 monitor + select 混合**：
```javascript
@test fn test_monitor_select() {
    shared const data_ch = Channel(1)
    let t = go { await Task.sleep(50); return 42 }
    let mon = t.monitor()

    data_ch.trySend(100)

    select {
        event from mon => { print("task done: " + event.value) }
        msg from data_ch => { print("data: " + msg) }
    }
    // 由于 data_ch 立即可用，应先收到 data: 100
    // 第二次 select 收到 task done: 42
}
```

### 压力测试

```javascript
// 大量 linked go + cancel 级联
@test fn test_link_cascade_stress() {
    for (let i = 0; i < 10000; i++) {
        linked scope {
            go { return i }
            go { return i + 1 }
        }
    }
}

// 大量 monitor Channel 并发读写
@test fn test_monitor_stress() {
    let channels: Channel[] = []
    for (let i = 0; i < 1000; i++) {
        let t = go { return i }
        channels.push(t.monitor())
    }
    for (let ch of channels) {
        let event = <-ch
        assert(event.ok)
    }
}
```

## 8. 实施顺序与里程碑

```
Phase 1 (前端语法, ~3天)
  ├── Step 1.1: Parser 前缀修饰符识别
  ├── Step 1.2: AST 节点扩展
  ├── Step 1.3: 字节码编码
  ├── 编译测试: --dump-bytecode 验证
  └── Git checkpoint

Phase 2A (link 运行时, ~4天)
  ├── Step 2.1: XrCoroutine link 数据结构
  ├── Step 2.2: linked go — vm_go 读取 + 建立 link
  ├── Step 2.3: 退出路径 propagate_link_error
  ├── Step 2.4: linked scope — SCOPE_ENTER/EXIT 扩展
  ├── 回归测试: 1120/1122/1127
  └── Git checkpoint

Phase 2B (monitor 运行时, ~3天)
  ├── Step 2.5: monitored go — 返回 Channel
  ├── Step 2.6: Task.monitor() API
  ├── Step 2.7: notify_monitors 结构化事件
  ├── Step 2.8: supervisor scope
  ├── 回归测试: 1121/1123/1124/1126/1128
  ├── 压力测试: 50K 并发
  └── Git checkpoint

Phase 3 (cluster 统一, ~1天)
  ├── Step 3.1: 统一事件格式
  └── 集成测试
```

## 9. 已有基础设施复用清单

| 需求 | 已有设施 | 复用方式 |
|------|---------|---------|
| Monitor Channel 创建 | `xr_channel_new()` | 直接使用 |
| Monitor 注册/通知 | `xcoro_registry.c` watched_by 链表 | 直接复用 |
| 协程退出钩子 | `xr_coro_on_exit()` | 扩展调用 propagate_link_error |
| select 多路复用 | 现有 select 语法 + OP_SELECT | 零改动 |
| 按名查找协程 | `xr_coro_registry_whereis()` | 直接使用 |
| 远程协程监控 | `cluster_monitor.c` | 事件格式统一 |
| 按需分配扩展 | `xr_coro_ensure_ext()` | 直接使用 |
| 协程取消 | `xr_coro_cancel()` | linked 传播时调用 |
