# 调试手册

高频问题的排查流程，从历史调试经验中提炼。

---

## JIT 测试 Timeout / Hang

**症状**：`--jit-force` 运行某个测试时挂起，最终超时。

**排查步骤**：

1. 禁用 JIT crash handler 确认是否为信号循环：
   ```bash
   timeout 3 env XRAY_NO_JIT_CRASH_HANDLER=1 ./build-reldbg/xray test --jit-force <file> 2>&1
   ```
   如果输出 `SIGSEGV` 或 `SIGBUS`，说明是 crash handler 信号循环导致的 timeout。

2. 确认崩溃位置：
   ```bash
   lldb -- ./build-reldbg/xray test --jit-force <file>
   # 在 lldb 中 run，崩溃后 bt 查看调用栈
   ```

3. 常见根因：
   - **XrValue heap_type 缺失**：检查 JIT runtime helper 中的 XrValue 构造
   - **Stack layout 不一致**：检查 `xr_vm_call_closure` 的参数传递
   - **Guard page handler 误判**：检查 `try_handle_guard_page_fault` 逻辑

---

## JIT 输出结果与解释器不一致 (Diff Fail)

**症状**：`run_jit_diff_tests.sh` 报告 diff fail。

**排查步骤**：

1. 查看字节码确认指令序列：
   ```bash
   ./build-reldbg/xray --dump-bytecode <file>
   ```

2. 用 `--jit-force` 和 `--no-jit` 分别运行，对比输出：
   ```bash
   ./build-reldbg/xray test --jit-force <file> 2>&1
   ./build-reldbg/xray test --no-jit <file> 2>&1
   ```

3. 常见根因：
   - **类型推导错误**：JIT builder 拿到的 slot type 与运行时不符
   - **OSR entry 遗漏 through-value**：检查 PHI input 是否被错误跳过
   - **deopt 值重建错误**：检查 `deopt_reconstruct` 的类型处理

---

## SIGSEGV in OP_GETFIELD / OP_SETFIELD

**症状**：VM 解释器在字段访问时崩溃。

**排查步骤**：

1. 在 GETFIELD handler 中加临时检查：
   ```c
   XrInstance *inst_obj = xr_value_to_instance(inst_val);
   if (!inst_obj) {
       fprintf(stderr, "[GETFIELD] inst_obj=NULL tag=%d heap_type=%d ptr=%p\n",
               inst_val.tag, inst_val.heap_type, inst_val.ptr);
       return XR_VM_RUNTIME_ERROR;
   }
   ```

2. 检查 receiver 来源：
   - 从 JIT `xr_jit_invoke_direct` 传入？→ 检查 XrValue 构造是否设置了 heap_type
   - 从 VM 栈上读取？→ 检查栈上的值是否被覆盖

---

## GC 相关崩溃

**症状**：随机位置崩溃，开 ASan 后报 use-after-free 或 heap-buffer-overflow。

**排查步骤**：

1. 用 ASan 构建运行：
   ```bash
   cmake -B build-asan -DXRAY_SANITIZE=ON -DCMAKE_BUILD_TYPE=Debug
   cmake --build build-asan -j4
   ./build-asan/xray test <file>
   ```

2. 检查 safepoint bitmap 是否包含死亡指针（Pattern 5）

3. 检查 callee-saved 寄存器偏移是否与 `alloc_regs[]` 一致（Pattern 4）

---

## 编译器 type_info 缺失

**症状**：JIT 将 float 值加载到整数寄存器，或值类型错误。

**排查步骤**：

1. dump 字节码确认 upval/local 类型信息：
   ```bash
   ./build-reldbg/xray --dump-bytecode <file>
   ```

2. 检查编译器 Phase 1 是否为所有 upval 设置了 `compile_type`

3. 检查 JIT builder 中 `CELL_GET` / `CELL_SET` 的类型推导路径
