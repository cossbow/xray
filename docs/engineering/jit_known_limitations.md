# JIT 已知限制

当前 JIT 编译器不支持或部分支持的功能，以及对应的 workaround。

---

## 不支持的字节码指令

以下指令在 JIT builder 中未实现，遇到时会 fallback 到解释器执行：

| 指令 | 说明 | 影响 |
| ---- | ---- | ---- |
| `OP_GETFIELD` | 实例字段读取 | 方法体中访问 `this.field` 会 fallback |
| `OP_SETFIELD` | 实例字段写入 | 同上 |

> 注：这些指令的方法会通过 `xr_jit_invoke_direct` → `xr_vm_call_closure` → 解释器执行，
> 而不是完全不支持。只是该方法本身不被 JIT 编译。

## 已知的 workaround

- **JIT timeout 调试**：设置 `XRAY_NO_JIT_CRASH_HANDLER=1` 可禁用 JIT crash handler，
  暴露底层 SIGSEGV 而非表现为 timeout

## JIT → 解释器 fallback 路径

当 JIT 代码需要调用未 JIT 编译的方法时（如 `INVOKE_DIRECT` 调用含 GETFIELD 的方法），
通过 `xr_vm_call_closure` 递归进入解释器执行。注意事项：

- **XrValue 构造必须完整**：传给 `xr_vm_call_closure` 的 args 必须含正确 heap_type（Pattern 1）
- **信号处理冲突**：解释器中的 SIGSEGV 会被 JIT crash handler 捕获，可能导致无限循环（Pattern 6）
- **调试建议**：用 `XRAY_NO_JIT_CRASH_HANDLER=1` 禁用 crash handler 暴露真实崩溃

## JIT yield/resume 限制

JIT 代码中调用 yieldable C 函数时，无法在 JIT 帧内 yield：

- 当前行为：遇到 yieldable → deopt 到解释器，解释器处理 yield/resume
- `invoke_deopt_id` 机制保证 deopt 到精确的 OP_INVOKE 位置（而非从头执行）
- Try-mode IO 快速路径：IO 就绪时零开销内联完成，未就绪时无副作用返回 WOULD_BLOCK → deopt

## OP_CHECKTYPE JIT 支持

- 编译时折叠：已知 VTAG_I64/F64/BOOL → 静态检查 mask bit，通过则完全消除指令
- 运行时：TAGGED/PTR 值通过 `CALL_C xr_jit_checktype` + GUARD_NONNULL 处理
- 失败时 deopt 到解释器，由解释器抛出 TypeError

## 嵌套 JIT deopt 未处理（P0 正确性问题）

`--jit-force` 下所有函数被 JIT，回调函数（map/filter lambda）deopt 返回 DEOPT_MARKER，
外层 JIT 代码未检查 marker → SIGSEGV。影响 ~17 个 JIT diff CRASH。

- **修复方向**：CALL 指令后生成 deopt marker 检查（见 Pattern 16）

## 返回值类型丢失（P0 正确性问题）

无返回类型注解且无法精确推断的函数 → `return_type = UNKNOWN` → `jit_reconstruct_value()` 用启发式推断。
float 3.14 的 IEEE754 位模式被误判为 PTR。影响 ~9 个 JIT diff EXIT_DIFF。

- **修复方向**：RET 指令保存返回值 tag，或从 TFA 推导（见 Pattern 17）

## 已知的 JIT diff 失败模式

某些测试在 JIT diff 中失败但手动运行通过，常见原因：

- **timeout**：JIT 编译耗时导致测试超时（非功能性失败）
- **GC 时序**：JIT 模式下 GC 触发时机不同，可能暴露 safepoint bitmap 问题
- **浮点精度**：JIT 和解释器可能使用不同的浮点运算路径

---

（持续补充）
