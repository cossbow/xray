# 830 CPS 协程挂起/恢复机制

## 设计目标

将含 await 的函数在 XIR 层面做 CPS (Continuation Passing Style) 变换，使 JIT 和 AOT 统一受益：
- **JIT**：await 冷路径不再 deopt 回解释器，继续在 native 代码中执行
- **AOT**：生成 C 状态机代码（C 语言无法挂起函数，CPS 是唯一路线）
- **解释器**：不变（frame->pc + VM_BLOCKED 已是隐式 CPS）

## 现状分析

### 当前 JIT await 路径

```
OP_AWAIT → CALL_C(xr_jit_await)
         → fast path: task done → 返回 result ✅
         → slow path: DEOPT_MARKER → 退化到解释器 ❌ (丢失 JIT 性能)
```

### 已有但未完成的 Phase 2 机制

| 组件 | 状态 | 说明 |
|------|------|------|
| `XIR_SUSPEND` codegen (xir_codegen_mem.c) | ✅ 已完成 | 保存 x1-x15,x20-x27 到 jit_suspend_regs[] |
| `xr_jit_await_block` (xir_jit.c) | ❌ 空壳 | 始终返回 0，无 CAS 协调 |
| `xir_jit_resume` (声明在 xir_jit_runtime.h) | ❌ 未实现 | 声明但无实现 |
| OP_AWAIT builder (xir_builder_misc.c) | ⚠️ 走 deopt | 未使用 XIR_SUSPEND |
| `coro->jit_resume_entry/proto/suspend_id` | ✅ 字段存在 | xcoroutine.h 中已定义 |
| resume 跳转表 codegen | ✅ 已完成 | suspend_cont_offsets[] 在 codegen 中记录 |

## 架构设计

### CPS 在 XIR 层面的实现层次

```
源码 → AST → 字节码 → XIR(SSA) → [CPS 变换] → JIT(ARM64) / AOT(C)
                                       ↑
                                   在这里做变换
```

### JIT 路径（Phase 1）

XIR_SUSPEND 已有的 codegen 机制可以直接使用。只需：
1. 修改 OP_AWAIT builder：slow path 发射 XIR_SUSPEND 而非 XIR_DEOPT
2. 实现 `xr_jit_await_block`：CAS task->await_state 协调
3. 实现 `xir_jit_resume`：加载保存的寄存器，跳转到续体点

变换后的 JIT await 路径：
```
OP_AWAIT → CALL_C(xr_jit_await)
         → fast path: task done → 返回 result ✅
         → slow path: XIR_SUSPEND → 保存寄存器 → CAS 设置 waiter
           → 阻塞: 返回 SUSPEND_MARKER → worker yield
           → 恢复: xir_jit_resume → 加载寄存器 → 跳转表续体 → 继续 JIT ✅
```

### AOT 路径（Phase 2）

AOT codegen (`xcgen.c`) 遇到含 XIR_SUSPEND 的函数时，生成 C 状态机：

```c
// 生成的 C 代码
typedef struct foo_cont {
    int resume_label;
    XrtValue saved_v3;  // 跨 await 的活跃变量
} foo_cont;

XrtValue foo(XrtCoro *coro, foo_cont *cont, XrtValue resume_val) {
    switch (cont->resume_label) {
    case 0: goto L_entry;
    case 1: goto L_resume_1;
    }
L_entry:
    XrtValue v1 = bar(coro);
    if (xrt_task_done(v1)) { v3 = xrt_task_result(v1); goto L_after_1; }
    cont->saved_v3 = v3;
    cont->resume_label = 1;
    return XRT_SUSPENDED;
L_resume_1:
    v3 = cont->saved_v3;
    v3 = resume_val;
L_after_1:
    return v3;
}
```

AOT 需要额外的 **活跃变量分析**（liveness at suspend point），这在 XIR SSA 上很自然。

### 活跃变量分析

对每个 XIR_SUSPEND 点，计算哪些 vreg 在 suspend 之后仍被使用（live-out）。
这些 vreg 就是需要保存到 continuation 的变量。

JIT：保存所有物理寄存器（当前行为，简单可靠）
AOT：只保存 live vreg（精确，生成更紧凑的 continuation struct）

## Task await CAS 协调协议

### 状态机（task->await_state）

```
NONE ──CAS──→ WAITING ──executor completes──→ RESOLVED
  ↑                                              │
  └──────────── waiter consumes ←────────────────┘
```

### xr_jit_await_block 实现逻辑

```c
int64_t xr_jit_await_block(XrCoroutine *coro, int64_t extra_arg) {
    XrTask *task = (XrTask *)coro->jit_ctx->call_args[0]; // from fast path
    int discard = (int)(extra_arg & 0xFF);
    
    // Set waiter fields
    task->waiter = coro;
    task->waiter_index = -1;
    
    // CAS: NONE → WAITING
    int expected = XR_AWAIT_NONE;
    if (CAS(&task->await_state, &expected, XR_AWAIT_WAITING)) {
        // Successfully registered as waiter — will be woken by executor
        coro->await_task = task;
        set_wait_flags(coro, XR_CORO_WAIT_AWAIT);
        return 0;  // blocked → JIT saves regs, returns SUSPEND_MARKER
    }
    
    if (expected == XR_AWAIT_RESOLVED) {
        // Race: executor completed between fast-path check and CAS
        // Result already available — inline resume
        XrValue res = discard ? xr_null() 
            : xr_deep_copy_to_coro(coro->isolate, task->result, coro);
        coro->jit_suspend_regs[23] = res.i;  // slot 23 = result
        task->await_state = XR_AWAIT_NONE;
        return 1;  // not blocked → JIT reloads regs, continues
    }
    
    // XR_AWAIT_WAITING: another waiter (shouldn't happen for single await)
    return 0;
}
```

### xir_jit_resume 实现逻辑

```c
bool xir_jit_resume(XrCoroutine *coro, XrValue *result) {
    if (!coro->jit_resume_entry) return false;  // not JIT-suspended
    
    // Store await result into suspend_regs[23]
    XrTask *task = atomic_load(&coro->await_task);
    if (task && result) {
        XrValue res = xr_deep_copy_to_coro(coro->isolate, task->result, coro);
        coro->jit_suspend_regs[23] = res.i;
        // Detach + recycle executor
        XrCoroutine *exec = task->coro;
        if (exec) { task->coro = NULL; exec->task = NULL; }
        atomic_store(&task->await_state, XR_AWAIT_NONE);
    }
    
    // Re-enter JIT code via resume entry
    void *entry = coro->jit_resume_entry;
    coro->jit_resume_entry = NULL;  // one-shot
    
    // Call resume entry: it reloads saved regs and jumps to continuation
    XirJitFn fn = (XirJitFn)entry;
    XrJitResult r = fn((intptr_t)coro, NULL);
    // ... handle result
    return true;
}
```

## 分阶段实施

### Phase 1: JIT CPS（本次实施）

| 步骤 | 文件 | 内容 |
|------|------|------|
| 1.1 | `xir_jit.c` | 实现 `xr_jit_await_block` CAS 协调 |
| 1.2 | `xir_jit.c` | 实现 `xir_jit_resume` 恢复逻辑 |
| 1.3 | `xir_builder_misc.c` | 修改 OP_AWAIT: slow path → XIR_SUSPEND |
| 1.4 | `xworker.c` | worker 恢复 JIT-suspended coro 走 resume |
| 1.5 | 测试 | 回归 + JIT diff 验证 |

### Phase 2: AOT CPS（后续实施）

| 步骤 | 文件 | 内容 |
|------|------|------|
| 2.1 | `xir_pass_cps.c` | 活跃变量分析 + suspend metadata |
| 2.2 | `xcgen_expr.c` | AOT codegen: XIR_SUSPEND → C 状态机 |
| 2.3 | `xcgen.c` | 生成 continuation struct |
| 2.4 | 测试 | AOT 协程测试 |

### Phase 3: 扩展（更多阻塞操作）

Channel send/recv, Select, Sleep 等阻塞操作的 CPS 化。

## 实现状态

| 功能 | 状态 | Commit |
|------|------|--------|
| Phase 1.1: xr_jit_await_block CAS | 🔧 进行中 | — |
| Phase 1.2: xir_jit_resume | 待做 | — |
| Phase 1.3: OP_AWAIT builder 改造 | 待做 | — |
| Phase 1.4: worker resume 集成 | 待做 | — |
| Phase 2: AOT CPS | 待做 | — |
