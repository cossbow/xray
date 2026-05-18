# JIT Verifier Framework

## 设计目标

将 `docs/engineering/invariants.md` 中的隐式不变量转化为**可执行的自动验证代码**，
在 Debug 构建中于每个 JIT 编译阶段结束后自动运行，
确保任何不变量违反在**产生位置**而非**表现位置**被捕获。

核心原则：**不变量不应只存在于文档中，应存在于代码中。**

---

## 验证器总览

| 验证器 | 状态 | 位置 | 触发时机 | 覆盖范围 |
|--------|------|------|----------|----------|
| `xir_verify_cfg` | ✅ 已有 | xir_pass_cfg.c | 每个 pass 后 | CFG pred/succ 一致性 |
| `xir_verify_types` | ✅ 已有 | xir_pass_cfg.c | 每个 pass 后 | 指令类型一致性 |
| `XIR_VERIFY_OFFSETS` | ✅ 已有 | xir_offsets_verify.c | 编译时 | 结构体偏移量 |
| V1: regalloc overlap | ✅ 已实现 | xir_regalloc.c | xra_run() 返回前 | 寄存器冲突检测 |
| V2: SE preservation | ✅ 已实现 | xir_pass.c 宏 | DCE/CSE/GVN/copy_prop 前后 | Side-effect 指令保护 |
| V3: SSA dominance | ⬜ 待实现 | — | 每个 pass 后 | def-use 支配关系 |
| V4: differential test | ✅ 已实现 | scripts/ | 手动运行 | JIT vs 解释器行为一致性 |

---

## V1: 寄存器分配验证器

**位置**: `src/jit/xir_regalloc.c`，`xra_run()` 返回前（`#ifndef NDEBUG`）

**验证内容**: 遍历所有 RA position，检查任意两个 vreg 不会在同一位置分配相同物理寄存器。
同 bundle 的 PHI 合并对被跳过（`phi_ranges_conflict` 已验证它们无 use 冲突）。

**能捕获的 bug**:
- 0741 (backward position movement → 两个 vreg 分配同一 reg)
- Pattern 10 (caller-saved clobber 范围不完整)

**复杂度**: O(blocks × vregs²)，debug 模式下通常 < 1ms。

## V2: Side-effect 保护验证器

**位置**: `src/jit/xir_pass.c`，`XIR_RUN_PASS_STRICT_SE` 宏

**验证内容**: 在 pass 执行前后对比 `XIR_FLAG_SIDE_EFFECT` 指令计数。
如果 pass 后 SE 计数减少，立即 abort。

**应用的 pass**: DCE、CSE、GVN、copy_prop
**不应用的 pass**: elim_guards、elim_write_barriers、DSE、store_to_load（这些可合法删除 SE 指令）

**能捕获的 bug**: 1130 (CSE 合并有副作用的 CALL_C)

## V3: SSA 支配验证器（待实现）

**验证内容**:
1. 每个 vreg 使用点被其定义点支配
2. PHI 节点参数与前驱块一一对应
3. call_arg_pool 中的 vreg 引用被 liveness 覆盖

**能捕获的 bug**: Pattern 14 (隐式数据引用对 liveness 不可见)

## V4: JIT vs 解释器差分测试

**位置**: `scripts/run_jit_diff_tests.sh`

**方法**: 对每个回归测试用例，分别用 `--no-jit` 和 `--jit-force` 运行，对比输出。支持并行执行和 allowlist。

**运行**: `bash scripts/run_jit_diff_tests.sh`

---

## 差分测试结果（2026-04-13）

```
Total:   274
Match:   258
Diff:    16 (JIT-specific, pre-existing)
```

### 已知 JIT-force 失败（16 个，全部是 pre-existing）

| 退出码 | 含义 | 测试文件 |
|--------|------|----------|
| 139 (SIGSEGV) | 空指针/野指针 | 0520_closure, 0521_cell_upval, 0560_method_chaining, 0650_nested_higher_order, 1008_json_yaml_fuzz, 1202_generic_inference |
| 138 (SIGBUS) | 地址对齐/访问错误 | 0610_array_functional, 0651_nested_higher_order_combo, 0881_nested_iterator_combo |
| 134 (SIGABRT) | 断言/regalloc verify | 0620_map, 1102_go_await, 1205_gc_incremental_pressure, 1206_gc_enhanced, 1207_gc_stress |
| 1 (测试失败) | 输出不匹配 | 0514_json_edge_cases, 1206_generic_functions |

大部分与 DEOPT_MARKER 未处理（invariants.md #13）和 closure/高阶函数 JIT 相关。

---

## 触发条件

- **Debug 构建**: V1、V2 自动运行（通过 `#ifndef NDEBUG` 控制）
- **Release 构建**: 验证器完全消除（零开销）
- **CI**: Debug 构建运行完整回归测试，验证器 abort 即 CI 失败
- **差分测试**: 手动运行 `bash scripts/run_jit_diff_tests.sh`，CI 可集成
