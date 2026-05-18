# 工程知识库

本目录只放**跨任务沉淀**的工程实践与不变量，用于防止 bug 复发、指导日常开发，**不放任何"做完就归档"的执行计划**。

> 进行中的 refactor / audit / implementation plan 在 [`../tasks/`](../tasks/)。
> 已完成的在 [`../archive/`](../archive/)。

## 内容分类

### 工程纪律与回填型文档

| 文档 | 用途 | 维护时机 |
|------|------|----------|
| [`bug_patterns.md`](bug_patterns.md) | Bug 模式分类 + 防御手段 + 检测方法 | **每修一个 bug** |
| [`invariants.md`](invariants.md) | 系统不变量 + 对应断言位置 | **每修一个 bug** |
| [`debug_cookbook.md`](debug_cookbook.md) | 高频问题排查流程 | 遇到新的难调试问题 |
| [`bug_fix_workflow.md`](bug_fix_workflow.md) | 标准化修 bug 流程 | 流程改动时 |
| [`architecture_decisions.md`](architecture_decisions.md) | 重要架构决策记录 (ADR) | **重大设计决策时** |
| [`audit_baseline.md`](audit_baseline.md) | 各模块审计基线（行数 / 复杂度 / 耦合） | 周期性回填 |

### 长期边界与策略文档

| 文档 | 用途 |
|------|------|
| [`jit_vm_boundary.md`](jit_vm_boundary.md) | JIT ↔ VM 边界契约 |
| [`jit_known_limitations.md`](jit_known_limitations.md) | JIT 已知限制与 workaround |
| [`jit_verifier_framework.md`](jit_verifier_framework.md) | JIT verifier 框架定义 |
| [`cross_platform_jit.md`](cross_platform_jit.md) | 跨平台 JIT 后端工程策略 |
| [`xisa_design.md`](xisa_design.md) | xisa codegen DSL 设计 spec（取代手写 emit；对接 RISC-V） |
| [`stdlib_security_audit.md`](stdlib_security_audit.md) | stdlib 安全审计基线 |

## 维护规则

1. **每修一个 bug**：在 `bug_patterns.md` 中检查是否属于已有 pattern，没有则新增。
2. **每修一个 bug**：在 `invariants.md` 中添加对应不变量与断言位置。
3. **重要架构决策**：记录到 `architecture_decisions.md`。
4. **信息过时**：标记 `[OUTDATED]` 而非直接删除，便于追溯。
5. **新增的 plan / audit / implementation 文档不放这里**：去 `../tasks/`。
