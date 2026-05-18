# 023 — 源码注释阶段说辞清理

> 日期：2026-04-25
> 状态：🔵 待启动（可分批进行）
> 范围：`src/`、`stdlib/`、`include/`、`tests/unit/`
> 依据：`.windsurf/rules/main.md` "注释与 commit 铁律"
> 同时建议：CI 加 lint 防回归

---

## 1. 背景

`.windsurf/rules/main.md` 与 `docs/rules/c-coding-standards.md` 已确立两条铁律：

- **A 禁止**：源码注释和 git commit message 中引用任何 `.md` 文档路径
- **B 禁止**：源码注释和 git commit message 中出现"阶段说辞"
  （`Phase A/B`、`Step N`、`P0/P1`、`Round N`、"本次重构"、"这一步"等）

A 类违规：源码已扫净（迁移 docs/ 时已清理 ~10 处）。
B 类违规：源码里仍有约 **103 个文件 / 317 处命中**，是这次任务的目标。

## 2. 违规分类（重要：不要一刀切）

扫描时必须区分**算法 phase**（合法）与**重构实施 phase**（违规）。

### 2.1 合法保留：算法 / 数据结构相位

属于"程序流水线本身的相位"，去掉反而让算法不可读。例：

```c
// xir_tfa.c
// Phase 2: Fixed-point iteration over abstract values
// Phase 3: Fixed-point solve

// xir_pass_type.c (bound elimination)
// Phase 1: Collect all integer definitions and GUARD_BOUNDS
// Phase 2: Detect simple induction variables
// Phase 3: Forward propagate ranges
// Phase 4: Eliminate redundant GUARD_BOUNDS

// xcgen.c (codegen buffers)
// --- Phase 1: locals buffer ---
// --- Phase 2: stmts buffer ---
```

判定标准：

- 是否描述当前函数 / 文件本身的算法步骤？→ 合法
- 是否能用 `Step 1 / 2` 之类替换且仍是算法描述？→ 合法
- 是否描述跨函数 / 跨模块 / 跨提交的工程进度？→ 违规

### 2.2 必须删除：重构 / 工程实施 phase

```c
// 重构历史溯源 — 违规
* Originally part of xir.h; extracted during Phase 7.
* Originally part of xir_pass_advanced.c; extracted during Phase 7.
* Originally part of xir_jit.c; extracted during Phase 7 to keep ...

// 重构计划占位 — 违规
* Phase 3 will fold them into this struct.
* Phase 1: holds only an `initialized` flag; Phase 3+ will gain ...
* Phase C: switch to _Thread_local.
* (Phase 4: CORO-08)
* Reductions Check for C Extensions (Phase 4)
* Disassembly API (Phase 4)
* reserved for Phase 4 pipeline
* Phase 2.2 replaces the fixed ... array

// 重构里程碑 / commit 引用 — 违规
* STATUS: Phase F.4.2 — integer arithmetic + control flow.
// Phase 1 (CORO-02 fix): if the coroutine has migrated ...
// Rebind pd to current worker when coro has migrated (Phase 1: CORO-02 fix).
* Phase 4.2: free_list changed from mutex-protected list to a lock-free Treiber stack.
* Initial bucket count. Seen hash dynamically grows (Phase 7.3) when ...

// "本次重构 / 这一步" 等中文同义违规
```

### 2.3 修改原则

不是机械删词，而是**把"阶段坐标"换成"事实陈述"**：

| 违规模式 | 改写策略 |
|----------|----------|
| `extracted during Phase X` | 直接删除整段历史溯源；当前文件位置就是事实 |
| `Phase X will do Y` | 删除 — 未来计划不属于当前代码注释；放 `tasks/` 文档 |
| `Phase X introduced caches` | 改成"Caches are invalidated whenever ..."（说事实和原因） |
| `Phase N fix: <symptom>` | 改成"<symptom>: <root cause + invariant>" |
| `STATUS: Phase F.4.2` | 改成"COVERAGE: integer arithmetic + control flow"（描述能力，不描述阶段） |

## 3. 当前规模（基线）

```
A 类（docs/*.md 引用）：             0 处（已清理）
B 类（Phase X / Step N / Round N）： 103 文件 / 317 处
  其中明显违规模式：~96 处
  其余需逐处判定：~221 处
```

## 4. 工作分解

按域分批，每批一次 commit + 一次构建 + 测试。

### 4.1 批次 1：JIT 模块（最高密度）

文件：

- `src/jit/xir_pass_advanced.c`（14）
- `src/jit/xir_jit_runtime.c`（10）
- `src/jit/xir_pass_type.c`（9 — 注意大部分是合法算法 phase，仔细甄别）
- `src/jit/xir_pass.c`（7）
- `src/jit/xir_coalesce.c`（7）
- `src/jit/xir_tfa.c`（6 — 大部分合法）
- `src/jit/xir_jit.c`（6）
- `src/jit/xir_builder.c`（6）
- `src/jit/xir_builder_misc.c`（4）
- `src/jit/xir_codegen.c`（4）
- `src/jit/xir_codegen_x64.c`（含 `STATUS: Phase F.4.2`）
- `src/jit/xir_pass_limits.h`（含 `Phase 2.2` / `Phase 4 pipeline`）
- `src/jit/xir_ops.h`（`extracted during Phase 7`）

预计实际改动：~50 处。

### 4.2 批次 2：Coro 模块

文件：

- `src/coro/xworker_sched.c`（10）
- `src/coro/xtimer_wheel.c`（9）
- `src/coro/xchannel.c`（8）
- `src/coro/xworker_sysmon.c`（6）
- `src/coro/xcoro.c`（6）
- `src/coro/xworker_exec.c`（4）
- `src/coro/xworker.c`（4）
- `src/coro/xproc.c`（4）
- `src/coro/xcoroutine.h`（4）
- `src/coro/xnetpoll.c`（3 — `Phase 1 (CORO-02 fix)` 模式）
- `src/coro/xyieldable.c`（3）
- `src/coro/xchannel.h` / `xproc.h` / `xtask.c` / `xworker_blocked.c` / `xworker_pool.c` / `xasync.c`
- `src/coro/xjit_hooks.c`（`Phase 4: CORO-08`）
- `src/coro/xcoro_pool.h`（`Phase 4.2: free_list ...`）
- `src/coro/xdeep_copy.c`（`Phase 7.3`）

预计实际改动：~40 处。

### 4.3 批次 3：Frontend / Analyzer

文件：

- `src/frontend/codegen/xcompiler.c`（19 — 最大；可能多为算法 phase 描述，仔细甄别）
- `src/frontend/analyzer/xanalyzer_mono.c`（8）
- `src/frontend/analyzer/xanalyzer_visitor.c`（7）
- `src/frontend/analyzer/xanalyzer.c`（6）
- `src/frontend/analyzer/xanalyzer_visitor_decl.c`（5）
- `src/frontend/codegen/xstmt_typed.c`（4）
- `src/frontend/codegen/xoop_enum.c`（4）
- `src/frontend/codegen/xstmt_destructure.c`（3）
- `src/frontend/codegen/xstmt_assignment.c`（3）
- `src/frontend/codegen/xexpr_call.c`（3）
- `src/frontend/analyzer/xanalyzer_visitor_stmt.c`（3）
- `src/frontend/analyzer/xanalyzer_visitor_expr.c`（3）
- `src/frontend/analyzer/xanalyzer_visitor_call.c`（3）

预计实际改动：~30 处（多数算法 phase 保留）。

### 4.4 批次 4：VM / AOT / 其它

文件：

- `src/vm/xvm.c`（10）
- `src/vm/xvm_cold_paths.c`（4）
- `src/aot/xcgen.c`（5 — 多算法 phase）
- `src/aot/xrt_runtime.h`（3 — `Phase 1` / `Phase 3`）
- `src/aot/xrt_exception.h`（含 `Phase C`）
- `src/runtime/value/xtype.h`（3）
- `src/app/cli/xcmd_build.c`（6）
- `src/app/lsp/xlsp_rename.c`（4）
- `src/app/dap/xdap_debug.h`（`Phase 4`）
- `stdlib/cluster/cluster_node.c`（6）
- `src/base/xchecks.h`（`Phase 4` of reductions）

预计实际改动：~30 处。

## 5. 执行流程

每批严格走以下流程：

```
1. 列出本批文件
2. for each file:
     - 通读 grep -n "Phase\|Step\|Round\|P0\|P1\|本次重构\|这一步" file
     - 区分算法 phase（保留）vs 实施 phase（删/改写）
     - 用 edit / multi_edit 修改
3. cmake --build build -j8
4. cd build && ctest --output-on-failure
5. git commit（commit message 也禁止 Phase / Step 字眼）
```

提交规范示例：

```
BAD:  refactor(jit): Phase 0 cleanup — remove refactor-phase comments
GOOD: refactor(jit): remove refactor-phase coordinates from doc comments

      Doc comments like "extracted during Phase 7" / "Phase 3 will fold ..."
      describe transient implementation milestones that became stale once the
      work landed. Replaced with self-contained statements of fact where the
      original intent was preserved.
```

## 6. 防回归（CI lint）

在 `scripts/check_architecture.sh` 或新建 `scripts/check_comment_rules.sh` 加：

```bash
# 检查 .c/.h 注释里的违禁模式
violations=$(grep -rn -E '(docs/[a-zA-Z_/-]+\.md|本次重构|本阶段|这一步)' \
                  src/ stdlib/ include/ \
                  --include='*.c' --include='*.h' 2>/dev/null | wc -l)
if [ "$violations" -gt 0 ]; then
    echo "ERROR: doc reference / refactor-phase wording in comments"
    grep -rn -E '(docs/[a-zA-Z_/-]+\.md|本次重构|本阶段|这一步)' \
         src/ stdlib/ include/ --include='*.c' --include='*.h'
    exit 1
fi
```

`Phase X` 因有合法用法不便机械检测，靠 code review 守住。

也可以加 commit-msg hook（`.git/hooks/commit-msg` 或 `.githooks/`）：

```bash
#!/bin/bash
msg=$(cat "$1")
if echo "$msg" | grep -qiE '(\bPhase\s*[0-9A-F]|\bStep\s+[0-9]|\bRound\s+[0-9]|\bP[0-3]\b|本次重构|这一步|docs/[a-zA-Z_/-]+\.md)'; then
    echo "ERROR: commit message contains forbidden refactor-phase wording or doc reference."
    echo "See .windsurf/rules/main.md '注释与 commit 铁律'."
    exit 1
fi
```

## 7. 验收标准

完成定义：

1. `grep -rn -E 'extracted during Phase|introduced.*Phase|reserved for Phase|Phase [0-9].*fix|originally part of'` 在 `src/` `stdlib/` `include/` 全部为 0
2. `grep -rn -E 'docs/[a-zA-Z_/-]+\.md'` 在 `src/` `stdlib/` `include/` `scripts/` 全部为 0（已达成）
3. 余下 `Phase X` 出现处必须可被合理解释为算法 / 数据结构本身的相位
4. CI 加入 `scripts/check_comment_rules.sh`（或合并进现有 lint），失败即拦截
5. 全量 `ctest` 与 `scripts/run_regression_tests.sh` 通过

## 8. 不在范围

- 不做超出"阶段说辞清理"的代码风格调整
- 不修改 `docs/`（文档本身使用阶段词无问题）
- 不修改 `tests/`（除非测试代码注释含有违规；目前已知一处已清理）
- 不动 git history（只管将来；不 force-push 重写过去 commit）

## 9. 完成后归档

任务完成后：

```bash
mv docs/tasks/023-comment-cleanup.md docs/archive/
```

并在 `docs/tasks/README.md` 中删除该行（或在末尾标记 ✅ 后批量清理）。
