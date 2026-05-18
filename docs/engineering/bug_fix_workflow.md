# Bug 修复工作流

本文档描述 xray 项目中发现、诊断和修复 bug 的标准流程。

## 1. Bug 分类与优先级

| 级别 | 描述 | 响应时间 |
|------|------|----------|
| P0 | Crash / 数据损坏 / 安全漏洞 | 立即修复 |
| P1 | 功能错误（用户可观测的错误结果） | 当日修复 |
| P2 | 性能退化 / 非关键路径错误 | 本周修复 |
| P3 | 文档 / 代码质量 / 边缘 case | 排期修复 |

## 2. 诊断流程

### 2.1 复现

```bash
# 最小复现用例放在 tests/tmp/ 目录
./build/xray tests/tmp/repro_bug.xr

# JIT 相关 bug：对比 JIT 与解释器输出
./build/xray tests/tmp/repro_bug.xr              # 默认（可能触发 JIT）
./build/xray --no-jit tests/tmp/repro_bug.xr     # 纯解释器
./build/xray --jit-force tests/tmp/repro_bug.xr  # 强制 JIT
```

### 2.2 诊断工具

| 工具 | 命令 | 用途 |
|------|------|------|
| 字节码 dump | `./build/xray --dump-bytecode file.xr` | 查看编译后的字节码 |
| IC dump | `./build/xray --dump-ic file.xr` | 查看 inline cache 类型反馈 |
| ASan | `cmake -DCMAKE_BUILD_TYPE=Debug -DUSE_ASAN=ON ..` | 内存错误检测 |
| UBSan | `cmake -DCMAKE_BUILD_TYPE=Debug -DUSE_UBSAN=ON ..` | 未定义行为检测 |
| JIT profiler | 启用 `src/backends/bytecode/vm/xvm_profiler.h` | JIT 性能分析 |
| MEM stats | `xr_mem_dump_stats()` | Debug 模式内存分配统计 |

### 2.3 JIT 特定诊断

```bash
# JIT differential test — 对比所有回归用例
bash scripts/run_jit_diff_tests.sh

# 单个测试的 JIT diff
bash scripts/run_jit_diff_tests.sh -f tests/regression/05_functions/0540_higher_order.xr
```

## 3. 修复流程

### 3.1 Git 备份

重大修改前先备份：

```bash
git stash  # 或 git commit -m "WIP: before fix attempt"
```

### 3.2 修复原则

1. **根因优先**：修复根本原因，不做下游 workaround
2. **最小改动**：能一行修复就不写十行
3. **添加断言**：在 bug 位置添加 `XR_DCHECK` 防止回归
4. **添加测试**：为每个 bug 创建回归测试用例

### 3.3 回归测试

```bash
# 单元测试
cd build && ctest --output-on-failure -j$(sysctl -n hw.ncpu)

# 回归测试（274+ 用例）
bash scripts/run_regression_tests.sh

# JIT differential test
bash scripts/run_jit_diff_tests.sh
```

### 3.4 测试用例命名

回归测试放在 `tests/regression/` 对应目录：

```
tests/regression/
├── 01_basics/          # 基础语法
├── 05_functions/       # 函数
├── 06_objects/         # 对象和 JSON
├── 07_classes/         # 类
├── 08_concurrency/     # 并发
├── 11_jit/             # JIT 特定
│   ├── 1100_jit_basic.xr
│   └── 1101_nested_jit_deopt.xr
├── 13_types/           # 类型系统
└── ...
```

单元测试放在 `tests/unit/`，命名 `test_<module>.c`。

## 4. 提交规范

```
<简短描述> (<任务ID>)

<详细说明>
- 根因分析
- 修复方案
- 影响范围
```

示例：

```
Clear stale deopt_id after callee JIT deopt (P0.1)

Root cause: when callee JIT function deoptimizes, its deopt_id
leaks to the caller's post-CALL_C CBNZ check, causing the caller
to mistakenly deopt with the callee's deopt_id.

Fix: clear jit_ctx->deopt_id = 0 immediately after detecting
callee deopt in xr_jit_call_func().

Added regression test: 1101_nested_jit_deopt.xr
```

## 5. 已知失败管理

JIT diff 测试的已知失败记录在 `tests/jit/known_failures.txt`：

```
# 格式：文件名  原因说明
0800_basic_channel.xr    channel not supported in JIT
0810_select.xr           select not supported in JIT
```

新增已知失败必须附带注释说明原因。

## 6. 常见 Bug 模式速查

| 模式 | 症状 | 典型修复 |
|------|------|----------|
| JIT deopt_id 泄漏 | 嵌套调用后 caller 意外 deopt | 清除 deopt_id |
| OSR PHI skip 遗漏 | 循环后变量为垃圾值 | 检查 coalesced 条件 |
| scope 不匹配 | Pass 2 报 "undeclared variable" | 检查 enter/exit scope 配对 |
| GC root 遗漏 | ASan 报 use-after-free | 检查 PTR spill writeback |
| 类型窄化丢失 | if 分支内类型推导错误 | 检查 flow graph antecedent |
| int64→int 截断 | 大数运算结果错误 | 添加范围检查 |
