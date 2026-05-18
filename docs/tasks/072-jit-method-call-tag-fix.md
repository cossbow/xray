# 072 — JIT Method Call / Tag Contract Fix

> 合并自两份独立分析（Opus + GPT），交叉核对源码后形成的可执行修复方案。

## 问题概述

`tests/regression/10_stdlib/1120_regex_re2_compat.xr` 在 `--no-jit` 下全通过，
JIT 开启后 14 个子测试返回 `null`/`false`。根因不是 regex 引擎，而是
**Xi → Xm → JIT CFunction bridge 的动态 XrValue tag 契约多处断裂**。

两份分析对 6 个 bug 的定位完全一致，差异仅在建议侧重点：
- **Opus** 更强调契约文档化（per-op aux 表、双向引用注释、lint 规则）
- **GPT** 更强调常量表示的长期设计（kpool index 替代 strcmp）和 bc_slot 传播完整性

下面合并为统一执行方案。

---

## 已确认的 6 个独立 Bug

| # | 简述 | 触发链 | 文件 | dirty fix |
|---|------|--------|------|-----------|
| 1 | `XI_CALL_METHOD` 落入 `xr_jit_call_func`（receiver 当 closure） | `match_ok` JIT → method call → 通用 closure 分支 | `xi_to_xm.c` | ✅ |
| 2 | `call_args[15]` 旧位图 vs `call_arg_tags[]` 字节 tag 读取错位 | codegen 写 byte tags → runtime 读 bitmap 槽 → tag=0 → null | `xm_jit_runtime.c` | ✅ |
| 3 | `XI_CALL_METHOD.aux_int` 是 super 标志，被当 method SymbolId | `method_symbol=0` → 方法表查找 100% 失败 | `xi_to_xm.c` | ✅ |
| 4 | Xi 字符串常量是裸 `char*`，缺 GC 头 | `XR_IS_STRING()` 读裸指针 → type != XR_TSTRING → pattern NULL | `xi_to_xm.c` | ✅ (strcmp) |
| 5 | `jit_value_from_tag` UNKNOWN tag → INT（应为 PTR） | receiver payload 当 INT → `jit_classify_receiver` 走 INT 分支 | `xm_jit_runtime.c` | ✅ |
| 6 | 动态 CALL_C 返回 tag 未写回 `slot_runtime_tags[]` | `regex.compile()` 返回 PTR 但 tag 丢失 → 后续参数重建错误 | `xi_to_xm.c` | ✅ |

**额外 fix**：`XI_GET_SHARED` 原有 `XM_LOAD` 路径在 ARM64 对 const-ref base 触发
XZR→SP aliasing（加载栈垃圾），改为 `XM_CALL_C(xr_jit_get_shared)` + TAGGED + bc_slot。

### 共性根因

6 个 bug 的本质都是 **同一字段在不同消费者之间含义漂移**：

| 数据 | 写入侧约定 | 读取侧约定 | 漂移结果 |
|------|-----------|-----------|---------|
| `call_args[15]` | codegen 不写（旧位图已废弃） | 旧 invoke 当位图读 | Bug 2 |
| `call_arg_tags[]` | codegen 写 packed bytes | 旧 invoke 没读 | Bug 2 |
| `XiValue.aux` (string) | `xi_const_str` 存裸 char* | `lower_const` 需要 XrString* | Bug 4 |
| `XiValue.aux_int` (CALL_METHOD) | lower 存 super 标志 | xi_to_xm 当 SymbolId 用 | Bug 3 |
| receiver tag (UNKNOWN) | codegen 写 0xFF | 重建器默认 INT | Bug 5 |
| return tag (动态 CALL_C) | helper 在 x1 返回 | 后续读静态 tag | Bug 6 |

---

## 当前工作区风险

dirty 状态包含 **大量调试残留** 和 **一个行为性改动**，不能直接提交。

### 调试日志（全部删除）
| 文件 | 残留类型 |
|------|---------|
| `stdlib/regex/xregex_binding.c` | `g_regex_dbg` 全局 FILE*, `/tmp/regex_trace.log` |
| `stdlib/regex/xregex.c` | `fprintf(stderr, "[MAT#...]")` |
| `stdlib/regex/xregex_nfa.c` | `fprintf(stderr, "[NFA#...]")` |
| `src/jit/xm_jit_runtime.c` | `fopen("/tmp/shared_debug.log")` × 2 |
| `src/jit/xm_jit_runtime_coro.c` | `fopen("/tmp/shared_debug.log")` |
| `src/vm/xvm_dispatch_closure.inc.c` | `fopen("/tmp/shared_debug.log")` |
| `src/ir/xi_emit.c` | `fprintf(stderr, "[EMIT] func=...")` |

### 行为性调试改动（必须回退）
- `src/vm/xvm_cold_object.c` — `vm_invoke_module` 中 debug block
  **直接调用 CFunction 并提前 return**，改变了 VM 非 yieldable CFunction 执行语义

### 独立改动（单独评估）
- `src/module/xmodule.c` — cluster 条件编译从 `|| !defined(XR_STDLIB_MODULAR)` 改为
  仅 `defined(XR_HAS_CLUSTER)`，可能是 ASAN build workaround，与本 bug 无关

---

## 执行计划

分 3 个 commit，每个必须独立通过 ctest + regex 测试。

### Commit 1: 清除调试残留

回退所有纯调试文件到 HEAD：

```bash
git checkout HEAD -- \
  stdlib/regex/xregex_binding.c \
  stdlib/regex/xregex.c \
  stdlib/regex/xregex_nfa.c \
  src/jit/xm_jit_runtime_coro.c \
  src/vm/xvm_dispatch_closure.inc.c \
  src/vm/xvm_cold_object.c \
  src/ir/xi_emit.c \
  scripts/lldb_debug.sh \
  src/module/xmodule.c
```

手动清除 `src/jit/xm_jit_runtime.c` 中两处 `fopen("/tmp/shared_debug.log")` debug
block（保留结构性修复代码）。

**验证**：
```bash
cd build && ctest --output-on-failure
grep -rn '/tmp/.*\.log\|g_regex_dbg\|shared_debug' src/ stdlib/  # 期望零匹配
```

### Commit 2: JIT method call / tag 契约修复

保留并清理 dirty fix 中的**结构性修复**：

| 子项 | 文件 | 改动 |
|------|------|------|
| 2a | `xi_to_xm.c` | XI_CALL_METHOD/BUILTIN 走 `xr_jit_invoke_method`；从 `proto->code[bc_pc]` 解析 SymbolId；bc_slot 传播 |
| 2b | `xi_to_xm.c` | `lower_const` STRING 类型从 kpool 查找 XrString* |
| 2c | `xi_to_xm.c` | XI_GET_SHARED 改用 `XM_CALL_C(xr_jit_get_shared)` + TAGGED + bc_slot |
| 2d | `xm_jit_runtime.c` | `xr_jit_invoke_method` 只从 `call_arg_tags[]` 读 tag；UNKNOWN→PTR 升级 |
| 2e | `xm_jit_internal.h` | 删除 `jit_bitmap_tag()` 函数和旧位图注释 |
| 2f | `xm_offsets.h` | `XM_JIT_LOAD_TAG_SCRATCH` → `XM_JIT_TAG_SCRATCH_OFFSET`，更新注释 |
| 2g | `test_xi_to_xm.c` | `lower_shared_var` 断言 XM_LOAD → XM_CALL_C |

**验证**：
```bash
cd build && ctest --output-on-failure
build/xray test --no-jit tests/regression/10_stdlib/1120_regex_re2_compat.xr
build/xray test tests/regression/10_stdlib/1120_regex_re2_compat.xr
```

### Commit 3: 防御性断言 + 契约文档

| 子项 | 文件 | 改动 |
|------|------|------|
| 3a | `xm_jit_runtime.c` | `xr_jit_invoke_method` 入口：`XR_DCHECK(method_symbol > 0, ...)` |
| 3b | `xi_to_xm.c` | method symbol 解析失败：`XR_DCHECK(method_sym > 0, ...)` |
| 3c | `xi.h` | XiOp 枚举上方添加 per-op aux/aux_int 语义表注释 |
| 3d | `xm_offsets.h` | `_Static_assert` 锁定 call_arg_tags 布局 |

**验证**：
```bash
cd build && cmake .. && ctest --output-on-failure
```

---

## 后续重构项（独立排期，按优先级排序）

### R1. Xi 字符串常量持 constant pool index [高]

**现状**：`xi_const_str` 存裸 `char*`，JIT `lower_const` O(n) strcmp 查 kpool。

**目标**：lower 阶段将字符串注册到 proto 常量池，`XiValue.aux_int` 存 index，
JIT 直接 `kpool[v->aux_int]`（O(1)）。

**文件**：`xi.c`, `xi_lower.c`, `xi_emit.c`, `xi_to_xm.c`, `xi_cgen.c`

### R2. XI_CALL_METHOD lower 阶段解析 SymbolId [高]

**现状**：xi_to_xm 从 `proto->code[bc_pc]` 反查 OP_INVOKE B 字段。依赖 bytecode 存在。

**目标**：`xi_lower`（主线程，可访问 isolate symbol table）将方法名→SymbolId 解析后
存入 `aux_int`（`symbol_id << 1 | is_super`）。后台 JIT 零依赖 isolate。

**文件**：`xi_lower.c` / `xi_lower_expr.c`, `xi_to_xm.c`, `xi_emit.c`

### R3. JIT/VM 等价回归测试 [高]

每个 .xr 跑两遍（`--no-jit` / 默认），diff stdout。CI 增加专门 job。

**文件**：`scripts/run_regression_tests.sh`, `.github/workflows/ci.yml`

### R4. JIT module invoke 专项 unit test [中]

`tests/unit/jit/test_jit_module_invoke.c`：注册 fake module + CFunction，
热循环调用确保触发 JIT，验证返回值/tag/receiver identity。

### R5. `jit_value_from_tag` UNKNOWN→PTR 全局审计 [中]

审计所有 `jit_value_from_tag` 调用点，评估 UNKNOWN 默认值是否应改为上下文感知。

### R6. xi_to_xm 消除 isolate 依赖 [低]

所有需 isolate 的解析做成预处理快照，`LowerCtx.isolate` 字段删除。

### R7. JIT debug ringbuffer [低]

`XR_JIT_TRACE` 编译开关 + 环形缓冲日志（receiver tag, method_symbol, recv_type,
result tag/payload）。DEBUG/ASAN 构建默认开启。

---

## 验收标准

全部 3 个 commit 完成后：

```bash
# 单元测试
cd build && ctest --output-on-failure

# JIT regex 回归（关键）
build/xray test tests/regression/10_stdlib/1120_regex_re2_compat.xr
build/xray test --no-jit tests/regression/10_stdlib/1120_regex_re2_compat.xr
# 两者输出应完全一致

# 完整回归
scripts/run_regression_tests.sh

# 确认无调试残留
grep -rn '/tmp/.*\.log\|g_regex_dbg\|shared_debug' src/ stdlib/
# 期望：零匹配
```
