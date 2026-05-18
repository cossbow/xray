# 系统不变量

每修一个 bug，在这里添加对应的不变量。这些是代码中的**隐式假设**，违反时会导致 bug。

---

## XrValue 不变量

1. **`tag==XR_TAG_PTR` 时，`heap_type` 必须从 GC header 读取**
   - `xr_make_ptr_val()` 和 `xr_value_from_xxx()` 自动处理
   - `descriptor=0` 会清除 heap_type，禁止对 PTR 类型先清零再赋值
   - 违反后果：`xr_value_to_instance()` 返回 NULL → SIGSEGV

2. **XrValue 构造必须通过工厂函数**
   - PTR 类型：`xr_value_from_instance()` / `xr_value_from_closure()` / `xr_make_ptr_val()`
   - 数值类型：`XR_FROM_INT()` / `XR_FROM_FLOAT()` / `xr_null()` / `xr_bool()`
   - 例外：性能关键的内联代码可裸构造，但必须加 `// raw-ok` 注释

## JIT ↔ VM 边界不变量

3. **`xr_vm_call_closure` 的 args[0]（receiver）必须是完整 XrValue**
   - 必须包含正确的 `heap_type`，VM 解释器依赖它做类型检查
   - JIT runtime helper 中构造 receiver 时必须用 `xr_value_from_instance()`

4. **JIT 快速路径必须是慢速路径的严格子集**
   - 快速路径可以跳过检查，但不能跳过状态设置
   - 典型：`XIR_CALL_KNOWN` 必须设置 closure，与 `XIR_CALL` 一致

5. **safepoint bitmap 中只包含该点存活的 PTR vreg**
   - `record_safepoint` 必须检查 `vreg->last_use >= current_pos`
   - 违反后果：GC 扫描时访问已死亡指针 → use-after-free 或 crash

## 编译器 → JIT 边界不变量

6. **所有 upval 必须在编译器 Phase 1 设置 `type_info`**
   - 包括 mutable capture，`compile_type` 必须在 hoisting 阶段设置
   - JIT builder 依赖 `type_info` 决定寄存器类型（GP vs FP）
   - 违反后果：float upval 加载到 GP 寄存器 → 值损坏

## JIT codegen 不变量

7. **callee-saved 寄存器偏移必须与 `alloc_regs[]` 数组精确对应**
   - `callee_saved_offsets[i]` 对应 `alloc_regs[FIRST_CALLEE_SAVED + i]`
   - 修改寄存器分配顺序时必须同步更新 GC 偏移表

8. **OSR stub 中 PHI input 只在与 PHI dst coalesced 时才跳过加载**
   - 不同物理寄存器的 PHI input 必须正常加载
   - 违反后果：OSR 入口后 through-value 丢失

## JIT crash handler 不变量

9. **guard page fault 必须与普通 SIGSEGV 区分**
   - 仅当 fault_addr 精确匹配 `jit_ctx->safepoint_page` 时才处理
   - C 代码中的普通 NULL 解引用不应被当作 guard page fault
   - 违反后果：无限信号循环 → 测试 timeout

## 编译器分析器不变量

10. **多 Pass 分析器的 scope 树结构必须一致**
    - Pass 1（声明收集）和 Pass 3（类型推断）创建的 scope 层次必须相同
    - AST_BLOCK 等节点的 scope 创建策略必须在所有 Pass 中统一
    - 违反后果：跨 scope 变量查找失败（sym=NULL）

11. **枚举哨兵值不得与有意义的枚举值重叠**
    - 不得用 `== 0` 判断"未匹配"，因为枚举首元素可能为 0
    - 使用 `-1` 或 `UINT32_MAX` 作为明确的哨兵值
    - 违反后果：枚举值为 0 的分支被跳过

## JIT IR 不变量

12. **所有引用 vreg 的数据结构必须对 liveness 可见**
    - 不仅是指令的 `args[]`，还包括 call_arg_pool、side table 等
    - 新增辅助数据结构时必须同步更新 liveness 和 regalloc
    - 违反后果：vreg 寄存器被提前释放，调用参数损坏

13. **JIT CALL 指令后必须检查 DEOPT_MARKER 返回值**
    - 被调函数可能 deopt 回解释器并返回 DEOPT_MARKER
    - 外层 JIT 代码将 marker 当作普通值使用会导致 SIGSEGV
    - 当前状态：尚未修复，影响 ~17 个 --jit-force CRASH

## 寄存器分配不变量

14. **线性扫描 worklist 只能前向移动**
    - 驱逐分割（`alloc_blocked_reg`）产生的 tail 起始位置必须 >= 当前扫描位置
    - `split_between` 的下界必须为 `max(ostart, rstart)`，防止 tail 插入 worklist 的位置早于当前处理位置
    - 违反后果：`forward_state_to` 向后移动，已过期范围从 Active 消失，冲突检测失效 → 两个 vreg 分配同一寄存器
    - 验证器：`xra_run()` 末尾的 overlap verifier（debug-only）
    - Commit: `edd5e065`

15. **非同 bundle 的 vreg 在任何位置不得分配相同物理寄存器**
    - 同 bundle 的 PHI dst/arg 通过 `phi_ranges_conflict` 验证后可合法共享
    - `alloc_free_reg` 中的 bundle skip 只允许跳过**同 bundle**的冲突
    - 违反后果：寄存器值覆写 → 运行时数据损坏
    - 验证器：`xra_run()` 末尾的 overlap verifier（debug-only，跳过同 bundle 对）

16. **`call_arg_tags[]` 必须精确反映每个 CALL_C 参数的运行时类型**
    - 所有 RT_* codegen 必须在调用前写入 `call_arg_tags[i]`
    - tag 为 UNKNOWN 时必须使用动态补丁（从 `slot_runtime_tags[]` 加载）
    - 违反后果：JIT helper 读到错误的 tag → 类型误判 → 值损坏或崩溃

## 安全不变量

17. **密码学比较必须是 constant-time**
    - padding 验证和 timing-safe 比较不得有依赖数据的 early-return
    - 固定迭代次数 + 位运算积累结果

18. **解析器输入边界检查必须 >= 最小完整结构大小**
    - JSON surrogate pair 需要 6 字符、gzip trailer 需要 18 字节等
    - 不得假设输入以 NULL 结尾

## 资源管理不变量

19. **`_free` 函数必须释放对象的所有堆分配字段**
    - 新增字段时必须同步更新对应的 `_free` 函数
    - GC 类型标记必须与实际布局匹配（如 TBLOB vs TINSTANCE）
    - 违反后果：内存泄漏或 GC 越界遍历崩溃

## JIT CALL_C 边界不变量

20. **返回 `int64_t` 的 CALL_C helper 必须显式设置 `call_result_tag`**
    - `call_c_stub` 总是将 x1 存储到 `call_result_tag`，但 `int64_t` 返回类型的函数 x1 是未定义值
    - 如果 builder 标记结果为 `VTAG_TAGGED/UNKNOWN`，codegen 将垃圾 tag 写入 `runtime_tags`
    - 返回 `XrJitResult` 的 helper 不受影响（x1 天然携带 tag）
    - Builder 设置具体 VTAG（PTR/I64/F64/BOOL）的 helper 不受影响（使用静态 tag）
    - 违反后果：后续多态 helper 使用错误 tag → 类型混淆 → 值损坏或崩溃

21. **CPS suspend result vreg 的 `bc_slot` 必须被显式设置**
    - `braun_write_var` 不设置 `bc_slot`（只写 Braun SSA 定义表）
    - `builder_set_slot` 同时设置 `bc_slot` + `slot_map` + Braun 定义表
    - suspend result vreg 在 `builder_emit_cps_suspend` 中必须显式设 `bc_slot = result_slot`
    - 违反后果：`runtime_tags` 不被写入 → resume 后 stale tag → 类型混淆
