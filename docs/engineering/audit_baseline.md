# Xray 源码审计基线报告（Step A）

**生成时间**：2026-04-17
**工具**：cloc 2.08 · lizard 1.21 · cppcheck 2.20 · clang-tidy 22.1.3 · `scripts/check_architecture.sh`
**原始日志**：`docs/archive/audit_raw_2026_04_17/`

## 🟢 已修复（2026-04-17）

| # | 严重 | 修复内容 | 验证 |
|---|------|----------|------|
| 1 | 🔴 | `src/aot/xcgen.c` 栈/堆双形态缓冲：引入显式 `is_heap` 标志，消除 `free` 栈变量的隐式契约风险（clang-tidy false positive + 防御性重构） | ASan 通过 |
| 2 | 🔴 | `scripts/check_architecture.sh` Q-3 正则重写：覆盖 `malloc/free/calloc/realloc` 全集，扫描 `.c` + `.h`，精确白名单 `xmalloc.h`。重跑后暴露 **121 处真实违规**（之前是假 PASS） | 手工验证 |
| 3 | 🔴 | `src/jit/xir_regalloc.h` 6 个 `static inline` 查询函数加入 `r != NULL` 前置 check | clang-analyzer NullDeref 消除 |
| 4 | 🔴 | `xparse_coroutine.c` / `xparse_import.c` / `xparse_decl.c` / `xparse_oop.c` 错误路径 leak：两批 commit（`501eb28` + `42dfba3`）引入 `goto fail` 模式 + 4 个文件级 cleanup helper（`free_import_members`、`free_reexport_members`、`free_generic_params`、`oop_free_*`），覆盖全部真实 leak；剩余 warning 为 AST `(char*)` cast-transfer 假阳性 | ASan 61/61 + 回归 274/274 |
| 5 | 🔴 | 新增 `XR_REALLOC` / `XR_REALLOC_OR_ABORT` 宏 + 4 个单元测试，并将 **33 处** `p = xr_realloc(p, ...)` 泄漏模式迁移到新宏 | test_xmalloc 18/18 + 回归 274/274 |
| 6 | 🔴 | `src/runtime/value/xchunk.c` `xr_proto_add_symbol` / `xr_proto_add_raw_constant` 加入 count/capacity 入口 invariant DCHECK，消除 ArrayBound 警告 | ASan 通过 |
| 7 | 🔴 | `stdlib/http/http2.c` 两处 `headers_buf` 零初始化 + `memcpy` 零长度守卫 | ASan 通过 |
| 8 | 🔴 | `stdlib/regex/xregex_dfa.c` 4 个函数加 OOM NULL 检查 + 部分清理，另将 **8 处** unsafe realloc（xregex/parse/nfa/compile）迁到 `XR_REALLOC_OR_ABORT` | test_regex 通过 + ASan |

**回归验证（2026-04-17 最终）**：Debug 61/61 · ASan 61/61 · Regression 274/274 · 无内存错误

🔴 高优 8 项全部闭环（见 §7）。剩余待办：#9 120 处 raw allocator、#10/#11 复杂度、§5.1 其余 NullDeref/未初始化/越界等。

---

## 0. 执行摘要

工程整体规模 **17.3 万行 C 代码**，模块划分清晰、有完整的架构约束脚本，但本轮扫描暴露出 4 类需要关注的问题：

| 类别 | 严重度 | 数量 | 典型样本 |
|------|-------|------|----------|
| **真实内存安全 bug**（NullDereference / 未初始化 / realloc 泄漏 / 越界） | 🔴 高 | **80+** | `src/jit/xir_codegen.c`、`src/aot/../base/xmalloc.h:117`、`xparse_oop.c` |
| 架构约束违规（直接 `malloc/free/realloc`，绕过 `xr_*` 包装） | 🟠 中高 | **~120** | `src/aot/xrt.h`、`src/app/cli/xcmd_test.c` |
| 极端复杂度函数（CCN > 100） | 🟠 中 | **30+** | `vm/xvm.c run` CCN=**1373** |
| 工程整洁度（include 冗余、struct padding、隐式转换） | 🟡 低中 | **12000+** | 9797 misc-include-cleaner |

**自检脚本 bug**：`scripts/check_architecture.sh` Q-3 只检查了 `malloc(`，遗漏 `free/calloc/realloc`，导致 **PASS 是假阳性**。需修复（详见 §6.1）。

---

## 1. 代码规模基线（cloc）

```
Language        files    blank   comment    code
C                 351    28631     29551  150115
C/C++ Header      323     7782     14216   22545
SUM:              676    36416     43767  172681
```

- 注释占比 ≈ **15%**，低于内部规范 18% 的目标。
- `src/` 下 285 .c + 261 .h；`include/` 共 8 .h；`stdlib/` 121 文件（无 `.xr` 扩展，可能用其他后缀）。
- 总编译单元：411（compile_commands.json），其中 284 个匹配本次 clang-tidy 模式。

## 2. 架构基线（`scripts/check_architecture.sh`）

详见 `docs/archive/audit_raw_2026_04_17/architecture.txt`。结果：**7 errors, 43 warnings**。

### 超大文件（行数硬限）

| 文件 | 行数 | 限制 |
|------|------|------|
| `src/vm/xvm.c` | **7764** | 3000 |
| `src/jit/xir_jit.c` | 3820 | 3000 |
| `src/jit/xir_pass_advanced.c` | 3266 | 3000 |
| `src/coro/xworker.c` | 3166 | 3000 |
| `src/vm/xvm_cold_paths.c` | 3100 | 3000 |
| `src/aot/xrt.h` | 883 | 800 |
| `src/jit/xir.h` | 849 | 800 |

### 头文件公共 API 数超限（27 个 .h 超出 25 个 export 上限）

最高：`xast_api.h` 105、`xir_jit_runtime.h` 82、`xparse.h` 77、`xir_arm64.h` 72。
处理建议：拆成 `_internal.h` + `_public.h` 双层。

### static 函数比例

**38%（1575/4091）**，目标 ≥ 80%。说明大量"伪公共"函数泄漏到 `XR_FUNC`/裸声明。这是后续可批量收紧的高 ROI 项。

### 注释/Assert 密度低位

- `vm/`：1 assert / 204 行（目标 ≤ 1/100）
- `module/`：1 / 213
- 整体注释比 15%

## 3. 复杂度基线（lizard，CCN>15 或长度>200 报警）

共 **475 个函数告警**。Top 10：

| 函数 | CCN | NLOC | 行数 |
|------|-----|------|------|
| `vm/xvm.c::run` | **1373** | 5440 | 7601 |
| `frontend/codegen/xexpr_call.c::compile_call_internal` | 300 | 878 | 1344 |
| `jit/xir_builder_call.c::xir_translate_call_ops` | 286 | 1485 | 1908 |
| `jit/xir_builder.c::translate_instruction` | 235 | 983 | 1240 |
| `jit/xir_pass_advanced.c::xir_pass_type_prop` | 231 | 427 | 508 |
| `aot/xcgen_expr.c::xcg_emit_instruction` | 226 | 672 | 804 |
| `app/lsp/xlsp_completion.c::xlsp_analyze_completion` | 223 | 364 | 419 |
| `jit/xir_jit.c::is_jit_eligible` | 220 | 173 | 226 |
| `jit/xir_builder_object.c::xir_translate_object_ops` | 213 | 903 | 1110 |
| `frontend/analyzer/xanalyzer_visitor.c::xa_visit_infer_stmt` | 184 | 421 | 495 |

`vm.run` 这类 dispatch 大循环 CCN 高是 VM 实现的固有特征，可接受；但 `xlsp_analyze_completion`（应用层）CCN 223 属于必须拆分的对象。

## 4. clang-tidy 总览（13478 警告）

按 check 名（前 15）：

| 数量 | Check |
|------|-------|
| 9797 | `misc-include-cleaner` |
| 905  | `bugprone-multi-level-implicit-pointer-conversion` |
| 780  | `readability-redundant-declaration` |
| 439  | `bugprone-branch-clone` |
| 409  | `bugprone-tagged-union-member-count` |
| 377  | `bugprone-macro-parentheses` |
| 290  | `bugprone-narrowing-conversions` |
| 245  | `clang-analyzer-optin.performance.Padding` |
| 155  | `performance-no-int-to-ptr` |
| 110  | `readability-redundant-casting` |
| 95   | `bugprone-implicit-widening-of-multiplication-result` |
| 92   | `clang-analyzer-optin.core.EnumCastOutOfRange` |
| **61** | **`clang-analyzer-unix.Malloc`** |
| **34** | **`clang-analyzer-security.ArrayBound`** |
| **23** | **`clang-analyzer-core.NullDereference`** |
| 16   | `bugprone-suspicious-realloc-usage` |
| **3** | **`clang-analyzer-core.uninitialized.Assign`** |
| 3    | `clang-analyzer-unix.cstring.NullArg` |
| 4    | `clang-analyzer-core.CallAndMessage` |
| 2    | `clang-analyzer-core.UndefinedBinaryOperatorResult` |

加粗为内存安全焦点，下面专项展开。

---

## 5. 内存安全专项（用户重点关注：初始化 / 所有权 / 边界）

### 5.1 🔴 CRITICAL：路径敏感分析检出的真实 bug

#### 5.1.1 NullDereference（23 处）

参见 `audit_raw/critical_memory.txt`。最值得立刻修的 8 条：

| 文件:行 | 现象 | 模块 |
|---------|------|------|
| `src/jit/xir_codegen.c:142` | 通过空 `xra` 字段 → `nvreg` 解引用 | JIT |
| `src/jit/xir_regalloc.h:128/153/162` | `r` 为空时访问 `nvreg`（同模式 3 处） | JIT |
| `src/jit/xir_builder.c:2626` | 直接对空指针解引用 | JIT |
| `src/jit/xir_regalloc.c:1021` | 同上 | JIT |
| `src/runtime/gc/xcoro_gc.c:491` | `stack[i]` 数组首地址为空 | GC |
| `src/runtime/class/xclass_builder.c:470` | `vtable` 字段为空时数组访问 | runtime |
| `src/vm/xvm_api.c:91` | `frame->closure` 在 frame 为空时解引用 | VM |
| `src/app/lsp/xlsp_server.c:580/2538/2548/2577` | 4 处 server / new_data 为空数组访问 | LSP |
| `src/frontend/analyzer/xanalyzer_visitor_*.c` | `subject_type`、`var_type` 等推导出空后字段访问（3 处） | 类型分析 |

#### 5.1.2 未初始化变量被使用（10 处）

`clang-analyzer-core.uninitialized.Assign` 3 处 + `core.CallAndMessage` 4 处 + `core.UndefinedBinaryOperatorResult` 2 处 + cppcheck 2 处：

| 文件:行 | 现象 |
|---------|------|
| `src/frontend/codegen/xoop_class_descriptor_builder.c:442` | 赋值的源是未初始化值 |
| `src/vm/xvm_cold_paths.c:1236` | 同上 |
| `src/frontend/codegen/xstmt_coroutine.c:885` | 同上 |
| `src/runtime/class/xenum.c:137` | 函数第 2 实参未初始化 |
| `src/frontend/codegen/xstmt_destructure.c:306, 395` | 同上（2 处） |
| `src/frontend/codegen/xexpr_match.c:272` | 同上 |
| `src/jit/xir_codegen.c:2206` | `-` 右操作数是 garbage |
| `src/runtime/class/xclass_builder.c:654` | `>` 左操作数是 garbage |
| `stdlib/http/http2.c:838, 1080` | `memcpy` 读未初始化 `headers_buf`（cppcheck） |

#### 5.1.3 内存泄漏（unix.Malloc 61 处 + suspicious-realloc 16 处 + memleakOnRealloc 1 处）

集中区：

- **`src/frontend/parser/xparse_oop.c`**：18 处 leak（错误返回路径未释放局部 `xr_strdup` 结果）
- **`src/frontend/parser/xparse_decl.c`**：8 处 leak
- **`src/frontend/parser/xparse_import.c, xparse_coroutine.c`**：5 处 leak
- **realloc 误用模式**（`p = xr_realloc(p, ...)`）：11 处，`xr_realloc` 返回 NULL 时丢失原指针。重点位置：
  - `src/jit/xir_regalloc.c:1414`、`src/jit/xir_coalesce.c:323`
  - `src/runtime/object/xarray.c:754, 800`
  - `src/runtime/class/xclass_builder.c:58`
  - `src/module/xbundle.c:175`
  - `src/frontend/parser/xparse_type.c:256, 257`
- **app/cli/xcmd_test.c**：3 处 raw `realloc` 同样问题（且违反"禁直接 malloc"规则）

#### 5.1.4 ⚠️ 极严重：栈变量被 free

```
src/aot/../base/xmalloc.h:117  (called from src/aot/...)
  Argument to 'free()' is the address of the local variable 'reachable_buf',
  which is not memory allocated by 'malloc()'
```

这是 **未定义行为且会立即崩溃**。需要在调用现场和 `xmalloc.h` 中确认是否为 macro 误用。

#### 5.1.5 数组越界（34 处 security.ArrayBound）

按风险归类：

- **解析模块带 tainted index**（用户输入未充分校验）：`xbundle.c:60`、`xproject.c:79`、`xpkg_client.c:60`、`xlockfile.c:184`、`xmodule.c:592/731`、`xbytecode_io.c:158/760`、`xlsp_*.c` 多处
- **越界写入堆前后**：
  - `src/runtime/value/xchunk.c:271, 320`（chunk 操作 — runtime 核心，**优先级最高**）
  - `src/runtime/object/xbigint.c:800`
  - `src/runtime/xglobals_table.c:89`
  - `src/runtime/class/xclass_builder.c:635`
  - `src/jit/xir_pass.c:42`
  - `src/frontend/codegen/xcompiler.c:1448, 1487`
  - `src/api/xrepl.c:93`
  - `src/app/lsp/xlsp_server.c:2243`（写到 `indent_str` 之后）

### 5.2 🟠 架构违规：直接调用 raw `malloc/free/calloc/realloc`

**总计 ~120 处违反"禁止直接 malloc"规则**。分布：

| 模块 | 数量 | 备注 |
|------|------|------|
| `src/aot/xrt.h` | 28 处 (malloc/calloc/realloc) | AOT 运行时——**疑似有意独立**，需文档化白名单 |
| `src/app/cli/xcmd_test.c` | 30+ free/realloc | 测试运行器 |
| `src/app/lsp/xlsp_completion.c` | 3 free | LSP |
| `src/app/dap/xdap_protocol.c` | 6 free | DAP |
| `src/api/xruntime.c:37` | 1 realloc | API 层（写明 `return realloc(ptr, new_size)`，可能是给外部用） |
| `src/jit/xir_builder.c, xir_codegen.c, xir_builder_misc.c` | 4 realloc | JIT 性能路径 |
| `src/runtime/gc/xcoro_gc.h` | 2 realloc | 内联 GC 头文件 |
| `src/runtime/xexec_state.h` | 2 malloc/realloc | exec state |
| `src/coro/xtask.c, xnetpoll.h, xcoro_debug.c` | 3 calloc/realloc | 协程 |
| `src/vm/xic_method.h, xvm_cold_paths.c` | 2 calloc | VM IC |
| `src/jit/xir_bset.h` | 1 calloc | bitset |

**结论**：要么修复全部走 `xr_*`，要么将 `aot/` 与几个性能关键路径加入显式白名单并在 `check_architecture.sh` 中支持白名单语法。

### 5.3 🟡 cppcheck 80 处 nullPointerOutOfMemory

`xr_malloc` 后未检查 NULL 直接解引用。集中在 `stdlib/regex/xregex_dfa.c`、`stdlib/http/http2.c`。
建议：要么统一 OOM 策略（abort vs propagate），要么增加 `XR_CHECK_OOM(p)` 宏并强制使用。

### 5.4 🟡 113 处栈结构体未初始化

样本：

```
src/app/lsp/xlsp_server.c:1226    XrPoll poll;
src/app/cli/xcmd_test.c:479       XrTestConfig config;
src/app/dap/xdap_debug.c          XrDebugFrameCtx fctx;       (8 处)
src/frontend/codegen/xpeephole.c  XrDynArray new_code_arr;
```

多数后续会调用 `*_init(&x)` 函数，**但人工无法快速判定每一处都安全**。建议：

- 给所有"必须先初始化"的类型加上 `[[clang::require_explicit_init]]` 或在 init 函数里加 `XR_DCHECK(x->magic == 0)` 校验。
- 或统一推广 `= {0}` / 指定初始化器，让"零字段也能安全析构"。

---

## 6. 工具链/脚本本身的问题

### 6.1 🔴 `scripts/check_architecture.sh` Q-3 漏检

```@/Users/xuxinglei/workspace/xray-lang/xray/scripts/check_architecture.sh:65-77
echo "--- Q-3: No direct malloc/free/calloc/realloc ---"
q3_hits=$(grep -rn '\bmalloc\s*(' --include='*.c' "$SRC_DIR" 2>/dev/null \
    | grep -v 'xr_malloc\|xr_calloc\|xr_realloc\|luaM_\|#.*include\|//.*malloc\|/\*.*malloc' \
    | grep -v '_alloc\|allocator\|alloc_' \
    || true)
if [ -n "$q3_hits" ]; then
    count=$(echo "$q3_hits" | wc -l | tr -d ' ')
    fail "Found $count direct malloc() calls (use xr_malloc instead)"
```

只 grep 了 `malloc(`，注释/标题写的是 `malloc/free/calloc/realloc` 全集。
而且 `_alloc|allocator|alloc_` 过滤过宽，会同时屏蔽掉 `xr_malloc_aligned` 等合法名（虽然 `xr_*` 已先过滤但模式不严谨）。

**修复建议**：重写为统一正则 `\b(malloc|free|calloc|realloc)\s*\(`，加白名单文件清单。

### 6.2 头文件中含可执行代码

`src/aot/xrt.h`、`src/runtime/xexec_state.h`、`src/runtime/gc/xcoro_gc.h`、`src/jit/xir_bset.h`、`src/vm/xic_method.h` 等 `.h` 文件含 `static inline` 函数，里面直接调用 raw allocator。这是 raw allocator 调用集中在 `.h` 的原因。

需要决策：是把这些函数挪到 `.c`，还是给"单 TU 内联"模式开白名单。

---

## 7. 优先级 Issue 清单

按 **(严重度 × 可立即修复) 排序**。前 8 项建议立刻处理。

| # | 严重 | 成本 | 模块 | 问题 | 操作 |
|---|------|------|------|------|------|
| 1 | ✅ | 极低 | base | `xmalloc.h:117` 对栈变量调用 `xr_free_tracked → free`，UB | 已修 — `3d0e8c6`（AOT xcgen 栈/堆缓冲） |
| 2 | ✅ | 低 | scripts | `check_architecture.sh` Q-3 漏检 | 已修 — 本地 scripts/ 修复暴露出 121 处真实违规 |
| 3 | ✅ | 低 | jit/regalloc | `xir_regalloc.h:128/153/162` 同模式 3 处 NullDeref | 已修 — `a8dfbdf` |
| 4 | ✅ | 中 | parser | `xparse_oop.c` / `xparse_decl.c` 共 30+ 处错误路径 leak | 已修 — `501eb28`（import/coroutine/decl） + `42dfba3`（oop），真实 leak 全部补上，剩余 warning 为 AST cast-transfer 假阳性 |
| 5 | ✅ | 低 | jit/runtime | 11 处 `p = xr_realloc(p, ...)` | 已修 — `3ea454f`（`XR_REALLOC` 宏） + `302f2af`（33 处迁移） |
| 6 | ✅ | 中 | runtime/value | `xchunk.c:271, 320` 写堆前后 | 已修 — `0c6f3a3` xchunk 入口 invariant |
| 7 | ✅ | 中 | http2 stdlib | `headers_buf` 未初始化 memcpy | 已修 — `9af208c` |
| 8 | ✅ | 中 | regex stdlib | `xregex.c:414` realloc leak + 多处 OOM 解引用 | 已修 — `1fa23f0`（DFA OOM + 8 处 realloc 迁移） |
| 9 | 🟠 | 低 | 全工程 | 直接 malloc/free 120 处 | 分批替换或加白名单 |
| 10 | 🟠 | 高 | vm | `xvm.c::run` CCN 1373 | 长期任务，按 opcode 分组拆 |
| 11 | 🟠 | 中 | lsp | `xlsp_analyze_completion` CCN 223 | 拆函数 |
| 12 | 🟡 | 低 | 全工程 | 9797 misc-include-cleaner | 跑 IWYU 一次性精简 |
| 13 | 🟡 | 低 | 全工程 | 245 padding 警告 | 用 `pahole` / 重排字段 |
| 14 | 🟡 | 中 | 113 处 | 栈结构体未初始化 | 一次性补 `= {0}` |

---

## 8. 推荐下一步行动

按 ROI（投入产出比）排序：

1. **立刻**（< 1 天）：修 #1（栈变量 free 是必崩 bug）+ #2（脚本 bug）+ #3 / #5 / #7（小范围、模板化修复）。
2. **本周**：批量处理 #4（parser 错误路径 leak），引入 `cleanup` goto 或 `XR_DEFER` 宏。
3. **本月**：#6 / #8 内存与边界专项；推动 #9 全工程改 `xr_*`，同时给 AOT 加官方白名单文档。
4. **背景任务**：#10 / #11 复杂度治理（重构 + 测试覆盖），#12 / #13 / #14 整洁度改造（适合用脚本批量做）。

每个 issue 处理后建议：

- 添加最小回归测试。
- 跑 `/build-asan` 验证。
- 跑 `scripts/run_regression_tests.sh` 防回归。

---

## 附：原始日志索引

```
docs/archive/audit_raw_2026_04_17/
├── architecture.txt        # check_architecture.sh 完整输出
├── cloc.txt                # 代码规模
├── lizard_warnings.txt     # 475 处复杂度告警
├── cppcheck.txt            # 1773 行 cppcheck 警告
├── cppcheck_high.txt       # 92 行高严重度子集
├── clang_tidy.txt          # 13478 警告（9 MB）
└── critical_memory.txt     # 内存相关 check 抽取
```
