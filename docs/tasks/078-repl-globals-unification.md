# 078 — REPL × Top-level Globals 统一化

> 把 top-level 绑定从「编译期整型槽 + 运行期偏移」改成「运行期 name-keyed globals dict」。
> REPL 由此变成「普通脚本编译 + 共享 globals」，不再需要任何 REPL-专用的 seed / collect / 偏移修复代码。
> 与 077（Xi IR 优化）正交：本任务动的是 top-level 存储模型，不动 Xi IR 内部 SSA / pass 链。

## 开发原则

**不考虑向后兼容。不妥协。只选最优设计。**

xray 没有外部用户、没有发布的 ABI。本方案直接瞄准最终模型：

- ✅ 删除 `XrSharedArray` / `shared_offset` / `XrReplSymbolTable` 等所有"伪 dict"基础设施
- ✅ 每一个改动直接进入最终形态，不做并存路径、不做开关
- ✅ 测试必须先红再绿——验收测试在动手前写好

## 一、问题陈述（Why）

### 1.1 现状架构（用一句话画出来）

```
top-level let / fn / const        →  isolate->vm.shared[]    (整型绝对槽)
proto bytecode (GETSHARED Bx)     →  Bx + proto->shared_offset
REPL 跨输入名字解析               →  XrReplSymbolTable (扁平 name→slot 表)
REPL 类型保留                     →  isolate->repl_analyzer (持久 XaSymbol)
```

四份状态必须时刻一致。任何一份 drift = 一类 bug。

### 1.2 已踩过的坑

| Bug | 根因 |
|---|---|
| `xi_lower_program` 不种 prior symbol → 跨输入引用 unresolved | 一致性靠手工 seed |
| 持久 analyzer 释放 → fatal abort | 持久状态生命周期靠 ad-hoc |
| `let _ = ...` 解析失败 | `_` 是 wildcard token，REPL `it` 是 workaround |
| **嵌套 proto `shared_offset` 没清零 → 跨输入函数调用返回闭包本身** | 整型槽模型本质不可救药 |
| persistent analyzer `node_table` 持悬空 `AstNode*` 键 | "约定不查"，无 enforcement |
| 跨输入类型丢失（seed 时 `type_any`） | symbol table 不带类型 |

每一个 bug 都不是「实现不细」，是「模型本身要求几份状态人工同步」。

### 1.3 主流 REPL 怎么没这问题

| Runtime | top-level 存储 | 跨输入解析 |
|---|---|---|
| CPython | `__main__.__dict__` | `LOAD_GLOBAL`（dict 查表） |
| Node.js | REPL context dict | V8 scope chain |
| Lua | `_G` 全局表 | `OP_GETGLOBAL` |
| GHCi | persistent typecheck env | module import |
| SBCL | symbol-value cells | symbol slot deref |

**共同观察**：top-level = **按名字索引的 dict**。REPL 在编译期不需要协商槽位，运行期按名字查即可。我们目前的整型槽路线属于自找麻烦。

## 二、目标架构（What）

### 2.1 数据结构

```c
/* runtime/xglobals.h —— 替换现有 xglobals_table.h 的整型槽模型 */
typedef struct XrGlobals {
    XrMap *table;            // XrString* → XrValue（GC 根）
    /* IC 缓存：JIT / 解释器 inline-cache 命中后直接拿 cell 指针 */
    XrGlobalCell **ic_cache; // 可选，第二阶段加
} XrGlobals;
```

`XrGlobals` **存活于 isolate**，每个 module 独享一份；REPL 与脚本 `<main>` 模块共享一份 main globals。

### 2.2 字节码

新增两条 opcode（替换 `OP_GETSHARED` / `OP_SETSHARED`）：

```
OP_GETGLOBAL  A Bx     R[A] = globals[constants[Bx]]      // Bx = interned XrString 在常量池下标
OP_SETGLOBAL  A Bx     globals[constants[Bx]] = R[A]
```

`OP_GETBUILTIN` / `OP_SETGLOBAL`（已存在的旧 builtin 用）按需保留或统一为新形态。

### 2.3 编译期改动

| 文件 | 改动 |
|---|---|
| `xi.h` | 删除 `XiFunc.nshared / shared_map / slot_owned_names / slot_owned_consts / export_names` 中的"slot"语义；保留 `slot_owned_names` 作为"top-level 声明的名字列表"用于 module 导出 |
| `xi_lower_expr.c::lower_variable` | 顶级变量 → `XI_GET_GLOBAL`（aux = interned name） |
| `xi_lower_stmt.c` 顶级 `let / const / fn / class` | → `XI_SET_GLOBAL` |
| `xi_emit_object.c` | 新增 `xi_emit_get_global` / `xi_emit_set_global`；删 `xi_emit_get_shared / set_shared` |
| `xi_lower.c::prescan_shared_vars` | 不再分配 slot，仅扫描顶级声明列表用于 forward refs |

### 2.4 运行期改动

| 文件 | 改动 |
|---|---|
| `xvm_dispatch_closure.inc.c` | 实现 `OP_GETGLOBAL / OP_SETGLOBAL` |
| `xchunk.h` | 删除 `XrProto.shared_offset` |
| `xexec_state.h` | 删除 `XrVMState.shared`；新增 `XrGlobals *main_globals` |
| `xcoro_gc.c` | GC 根遍历改为 `xr_globals_iter` |
| JIT IC | 加 globals-cell 形态 IC（cache: interned name → cell*） |

### 2.5 REPL 改动（核心收益）

REPL 删除全部以下代码（约 600 行）：

- `XrReplSymbolTable` 及其增删查改、seed_context、collect_from_xi
- `xr_repl_zero_shared_offsets` 递归 patch
- xi_lower 中的 prior-symbol seed 循环
- `isolate->vm.shared.count` 推进 / `xr_shared_array_ensure` 调用
- xrepl.c 中的 `repl_symbols_collect_from_xi`、`xr_repl_peek_int` 内部 shared 访问

`xr_repl_compile` 缩成：

```c
XrProto *xr_repl_compile(XrayIsolate *iso, const char *src) {
    AstNode *ast = xr_parse(iso, src);
    if (!ast) return NULL;
    repl_maybe_echo_last_expr(iso, ast);     // 唯一 REPL-specific：auto-echo + it
    if (!iso->repl_analyzer)
        iso->repl_analyzer = xa_analyzer_new(iso);
    xa_analyzer_clear_diagnostics(iso->repl_analyzer);
    XrCompilerContext *ctx = xr_compiler_context_new_with_analyzer(iso, iso->repl_analyzer);
    ctx->repl_mode = true;
    XrProto *proto = xr_compile(ctx, ast);
    xr_compiler_context_free(ctx);
    xr_program_destroy(ast);
    return proto;
}
```

跨输入符号解析变成「analyzer 持久 global_scope + 运行期 globals dict 自然查名字」—— **再没有需要同步的两份状态**。

### 2.6 模块系统

每个 module 一份 `XrGlobals`。`import M` = 把 M 的 globals 装订到当前编译单元的 namespace。删除 `shared_offset` 切片机制。

## 三、阶段计划（How）

每个阶段一个 commit，独立可回退；前阶段绿灯方启动下阶段。

### Phase 0 — 验收测试先行（半天）

**目的**：写下"完成后必须通过"的测试，作为后续阶段的红线。

工作项：
- [ ] `tests/regression/13_repl/` 新增 .xr 套件（驱动方式参考 `tests/integration/repl_harness.c`，若不存在则新建）
  - `0001_cross_input_call.xr` — `let x = 10; fn f():int { return x }; f()` 在三段输入跨步骤断言
  - `0002_cross_input_mutate.xr` — bump counter
  - `0003_redefine_let.xr` — `let x = 1; let x = "two"` 类型改变
  - `0004_closure_captures_outer.xr` — `let arr = []; fn push_it(){arr.push(1)}; push_it(); arr`
  - `0005_class_across_inputs.xr` — 一段定义 `class Point`，另一段 `Point(1, 2).x`
- [ ] `tests/unit/api/test_repl.c` 增加：
  - `repl_globals_is_only_source_of_truth` — 反射式断言：编译后 `isolate->vm.shared` **不存在**或为空（构造期不变量）
  - `repl_proto_no_shared_offset_field` — 静态断言 `offsetof(XrProto, shared_offset)` 不再存在（编译期反向不变量）

这两条静态断言在阶段未完成时**应当编译失败**，作为「未到位」的硬信号。

**完成标准**：测试套件存在并标红（fail）。

### Phase 1 — `XrGlobals` + 两条 opcode + VM dispatch（1 天）

工作项：
- [ ] `src/runtime/xglobals.h / .c` 新文件，覆盖：
  ```c
  XrGlobals *xr_globals_new(XrayIsolate *iso);
  void xr_globals_free(XrGlobals *g);
  XrValue xr_globals_get(XrGlobals *g, XrString *name);            // 找不到 → null
  void    xr_globals_set(XrGlobals *g, XrString *name, XrValue v); // upsert
  bool    xr_globals_has(XrGlobals *g, XrString *name);
  void    xr_globals_iter(XrGlobals *g, void (*fn)(XrString*, XrValue*, void*), void *ud);
  ```
  内部：`XrMap *table`（复用 runtime/object/xmap.h）。
- [ ] `src/runtime/xexec_state.h`：
  - 在 `XrVMState` 加 `XrGlobals *globals;`（暂未删 `shared`，但本阶段也不写它）
- [ ] `src/api/xisolate_full.c`：init 时 `xr_globals_new`，cleanup 时 free
- [ ] `xopcode_def.h`：新增 `OP_GETGLOBAL` / `OP_SETGLOBAL`（格式 `FMT_ABx`）
- [ ] `xvm_dispatch_closure.inc.c`：实现两条 dispatch：
  ```c
  vmcase(OP_GETGLOBAL) {
      int a = GETARG_A(i);
      int k = GETARG_Bx(i);
      XrString *name = (XrString*) PROTO_CONST_FAST(cl->proto, k).ptr;
      R(a) = xr_globals_get(isolate->vm.globals, name);
      vmbreak;
  }
  vmcase(OP_SETGLOBAL) { /* mirror */ }
  ```
- [ ] `xcoro_gc.c`：GC 根遍历加 `xr_globals_iter` mark callback
- [ ] `xdebug.c`：两条 opcode 的 disassembler 输出

**完成标准**：xray_core 编译通过；新 opcode 可以手工 emit + 跑通（写一个一次性 unit test 验证 set + get）。脚本走老路径仍工作。

### Phase 2 — REPL 切换到 globals（1 天）

**先迁 REPL，因为它最小、收益最大、bug 已知**。

工作项：
- [ ] `xi_lower_expr.c::lower_variable`：
  - 在 `is_program && shared_map[var_id] >= 0` 分支 → 改发 `XI_GET_GLOBAL`（aux = interned name）
  - 删除「找到 var_id 但走 shared」的旧逻辑（在本阶段仅 REPL 模式启用 globals，由 `ctx->repl_mode` 或 `l->use_globals` 标志门控）
- [ ] `xi_lower_stmt.c`：top-level `let/const/fn/class` 在 REPL 模式发 `XI_SET_GLOBAL`
- [ ] `xi_emit_object.c`：`xi_emit_get_global` / `xi_emit_set_global`
- [ ] `xi.h` / `xi_lower.c`：在 REPL 模式跳过 `shared_map` 分配
- [ ] `xrepl.c`：
  - 删除 `XrReplSymbolTable` 整套接口（声明 + 实现 + accessor）
  - 删除 `xr_repl_zero_shared_offsets`
  - `xr_repl_compile` 精简到上面 §2.5 的形态
  - `xr_repl_peek_int` 改走 globals
- [ ] `xrepl.h`：删除 `XrReplSymbol` / `XrReplSymbolTable` / `xr_repl_symbols_*` 整套接口
- [ ] `xcli_isolate.c` / `xcmd_repl.c`：清理对 repl_symbols 的所有引用

**门控**：本阶段脚本路径不动，REPL 单独走 globals。

**完成标准**：
- Phase 0 的所有 REPL 回归测试通过
- 现有 `tests/unit/api/test_repl.c` 22 个 case 全绿
- 手工 `xray repl < /tmp/*.txt` 试遍 it / null / .vars / .type 全正常
- 113/113 ctest 全绿
- 295/295 regression 全绿

### Phase 3 — 脚本 top-level 切换到 globals（1.5 天）

**目的**：删除 REPL 路径与脚本路径分叉，让两者走同一逻辑。

工作项：
- [ ] 删除 xi_lower 的 `repl_mode` 门控，所有顶级声明 / 引用统一发 GLOBAL
- [ ] `xi_lower.c`：删除 `prescan_shared_vars` 中的 slot 分配段（仅保留名字收集用于 `slot_owned_names` → 改名 `top_level_names`）
- [ ] `xchunk.h`：删除 `XrProto.shared_offset`（**会触发大面积编译错误**）
- [ ] `xi.h`：删除 `XiFunc.nshared / shared_map`
- [ ] 修复所有连带：
  - `xi_emit.c`：删除 emit 期 `proto->shared_offset = isolate->vm.shared.count` 那段
  - `xbytecode_io.c`：序列化层删 shared_offset / set_shared_offset_recursive
  - `xm_jit_runtime.c` 等：所有跟 shared offset 相关的 IC 路径暂用 fallback（待 Phase 5 上 globals IC）
- [ ] `xexec_state.h`：删除 `XrSharedArray shared`
- [ ] `xshared.h` 整文件删除
- [ ] `xdeep_copy.c` / `xcoro_gc.c` / `xchannel.c` / `xsystem_heap.c`：把 shared array 引用换 globals iter

**完成标准**：
- xray_core 重新编译通过
- Phase 0 + 22 test_repl + 113 ctest + 295 regression 全绿
- 删除 ≥500 行 shared-* 旧代码

### Phase 4 — 模块系统切换（1.5 天）

工作项：
- [ ] `xmodule.h / .c`：`XrModule` 持有 `XrGlobals *exports`
- [ ] `import` 语句：lower 到「拿目标 module 的 globals dict 引用」，按名字查
- [ ] `export`：写当前 module 的 globals
- [ ] `xbytecode_io.c`：模块 .xrc 序列化层重写（每个 module dump 自己的 globals 名字 + 值）
- [ ] 删除"模块切片" shared_offset 残留

**完成标准**：
- 现有 import / export 回归（tests/regression/02_modules）全绿
- AOT 路径（如果当前依赖 shared_offset）跟着同步

### Phase 5 — JIT globals IC（2 天）

工作项：
- [ ] 新增 `XmGlobalIC`：interned name + cached cell pointer + 版本号
- [ ] `xi_to_xm.c`：emit globals access 走 IC 路径
- [ ] `xm_jit_runtime.c`：实现 IC miss fallback + invalidation
- [ ] dirty-flag：当 `xr_globals_set` 插入新键导致 dict rehash 时 bump 全局 version，让所有 IC 重 validate
- [ ] 微基准：`bench/vm_microbench.xr` 中加 top-level 频繁访问 case，确保 IC 命中 ≤ 旧 shared slot 路径 1.2×

**完成标准**：
- 所有 JIT 测试全绿
- 基准回归 ≤ 5%

### Phase 6 — 清理 + 文档（0.5 天）

- [ ] 删除 `xglobals_table.h/c` 旧 builtin 版本（合并入新 globals 或单独 builtin dict）
- [ ] 更新 `docs/rules/architecture.md` 反映新分层
- [ ] 删除本任务相关的 TODO / 临时注释
- [ ] 把本文档归档到 `docs/archive/`，编号保留

## 四、测试策略

### 4.1 不变量（写成 static_assert 或 runtime DCHECK）

- `XrProto` 没有 `shared_offset` 字段（Phase 3 后）
- `XrVMState` 没有 `shared` 字段（Phase 3 后）
- 任何 REPL 输入编译后 `isolate->vm.globals->count` 严格不减
- `OP_GETGLOBAL/SETGLOBAL` 的 Bx 必须是 `XR_IS_STRING(constants[Bx])`

### 4.2 测试矩阵

| 维度 | Phase 0 写好的测试 | 每个阶段必跑 |
|---|---|---|
| 单输入正确性 | regression/01-12 全套 | ctest |
| REPL 跨输入引用 | regression/13_repl/*.xr | `xray repl < file` |
| REPL 单元 API | tests/unit/api/test_repl.c | ctest test_repl |
| 模块 import/export | regression/02_modules | regression script |
| JIT 等价性 | tests/unit/jit + bench microbench | ctest + bench |
| 内存安全 | scripts/asan_test.sh + tests/regression | ASAN run |

### 4.3 手工验证脚本

每个 Phase 2 之后跑一次：
```
./build/xray repl <<EOF
let x = 10
fn getx(): int { return x }
let r = getx()
r                              # 期望 10
let arr = []
fn push_it() { arr.push(1) }
push_it()
push_it()
arr.length                     # 期望 2
class Point { let x: int; constructor(x: int) { this.x = x } }
Point(42).x                    # 期望 42
.vars
.exit
EOF
```

## 五、风险与回退

| 风险 | 监控 | 回退策略 |
|---|---|---|
| JIT globals IC 性能差 | bench microbench | 暂用 cold-path globals dict 查表，IC 推后 |
| 模块系统 .xrc 兼容 | regression/02_modules | 本任务不支持向后兼容 .xrc——重新生成全部 .xrc |
| AOT 依赖 shared_offset | 在 Phase 3 阶段触发编译错误 | AOT 同步迁移（如必要拉为 Phase 3.5） |
| 大爆炸式改动 | 每个 Phase 独立 commit | git revert 单 commit 即回退 |
| GC 根遗漏导致 use-after-free | ASAN + GC stress | 在 Phase 1 加 globals_iter 单测 |

每个 Phase 完成后建一个 git tag（如 `repl-globals-phase2`），方便后续 bisect。

## 六、验收清单（任务完成定义）

- [ ] `XrReplSymbolTable` / `xr_repl_zero_shared_offsets` / `proto->shared_offset` / `XrVMState.shared` 全部源码不再存在
- [ ] `xr_repl_compile` ≤ 30 行
- [ ] 22 个 test_repl + 113 ctest + 295 regression 全绿
- [ ] Phase 0 新增的 5 个跨输入 .xr 回归全绿
- [ ] AddressSanitizer 跑完整 regression 套件 0 报错
- [ ] `bench/vm_microbench.xr` 顶级访问 case 回归 ≤ 5%
- [ ] 文档归档，README 索引更新到 078 → done

## 七、不在本任务范围

- 不重写 analyzer 内部数据结构（持久 analyzer 继续用现有 global_scope）
- 不引入 prompt-toolkit 风格的 REPL UI（input highlighting、hover 等）
- 不动协程 / channel / GC 算法
- 不引入新的语言特性

## 八、时间盒

| 阶段 | 估时 | 累计 |
|---|---|---|
| Phase 0 | 0.5 天 | 0.5 |
| Phase 1 | 1 天 | 1.5 |
| Phase 2 | 1 天 | 2.5 |
| Phase 3 | 1.5 天 | 4 |
| Phase 4 | 1.5 天 | 5.5 |
| Phase 5 | 2 天 | 7.5 |
| Phase 6 | 0.5 天 | 8 |

**总计 ~8 工日（约 1.5 周）**，按当前 codebase 规模可行。

## 九、决策记录

| 决策点 | 选择 | 备选 | 理由 |
|---|---|---|---|
| top-level 存储模型 | name-keyed dict | 整型槽 + 持久 symbol 表 | 与 Python/Lua/Node 一致；REPL 自然正确 |
| dict 实现 | 复用 `XrMap` | 写专用开放寻址 hash | 减少代码量；XrMap 已经过 GC 测试 |
| 是否保留 shared offset 兼容层 | 否 | 加 flag 切换 | 项目原则：不留兼容层 |
| 模块 globals 是否共享 isolate-wide | 否，每 module 一份 | 单全局 dict | 隔离命名冲突；import 走 dict-ref 仍是 O(1) |
| JIT IC 是否本任务必做 | 是 | 单独 task | 不做 IC 等于 globals 比 shared 慢 3-5×，会被性能回归卡住 |
| `it` 是否改名 `_` | 否 | 改 lexer 给 `_` 让路 | `_` match-wildcard 是已发布语义；`it` 借鉴 GHCi 更稳 |

## 十、相关历史

- REPL 持久 analyzer：commit `e22641f`
- REPL 跨输入函数调用 bug 修复：commit `3b011ce`（本任务旨在让这类 bug **结构上不可能再出现**）
- 当前 shared-offset 模型源自 002-runtime-refactor、011-jit-next-phase 等早期决策
