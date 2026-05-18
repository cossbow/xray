# 跨平台 JIT 后端工程策略

新增 ISA 后端（x86-64, RISC-V 等）时的工程纪律和防御手段。

---

## 架构复用原则

**共享层**（不因新后端而改动）：
- XIR IR 定义（`xir.h`）、所有优化 pass（`xir_pass*.c`）
- 寄存器分配器（`xir_regalloc.c`）— 通过 `XirTarget` 描述差异
- Deopt 表结构、stack map、OSR 框架
- Code cache 管理（`xir_code_alloc.*`）
- JIT 编译驱动（`xir_jit.c` 的 try_compile / bg queue）

**平台相关层**（每个后端独立实现）：
- `xir_target_<arch>.c` — 寄存器集合、calling convention、spill 策略
- `xir_<arch>.h/c` — 指令编码（ARM64: 固定 4B; x86-64: 变长 1-15B）
- `xir_codegen_<arch>.c` — XIR → machine code 翻译
- `xir_codegen_<arch>_call.c` — 调用约定适配
- `xir_peephole_<arch>.c` — 后端特定 peephole

**关键设计决策**：不抽象 codegen。ARM64 和 x86-64 差异太大（3-operand
vs 2-operand, flags 语义, 变长编码），强行抽象两边都写不好。

---

## 三层防御体系

### Layer 1: 指令编码验证

**最早拦截**编码错误，在指令发射层就捕获。

```bash
# 开发流程: 写一条指令 → nasm 生成参考 → assert 逐字节一致
echo "BITS 64" > /tmp/ref.asm
echo "add rax, rcx" >> /tmp/ref.asm
nasm -f bin -o /tmp/ref.bin /tmp/ref.asm
xxd -i /tmp/ref.bin  # → 对比 x64_add_rr() 的输出
```

**单元测试 (`test_x64_emit.c`)**：
- 每条新增指令必须配一个编码 assert
- 覆盖 REX prefix 的 4 种组合（W/R/X/B）
- 覆盖 ModR/M 的 3 种 mod（reg-reg, [reg+disp8], [reg+disp32]）
- 覆盖 SIB 字节（[base + index*scale + disp]）

**参考**：现有 `test_arm64_emit.c` 已验证 ARM64 指令编码。

### Layer 2: 差分测试

**最强正确性保障**：对每个测试用例，解释器 vs JIT 行为必须一致。

```bash
# 已有基础设施，直接继承
bash scripts/run_jit_diff_tests.sh
```

x86-64 后端的差分测试流程（Rosetta 2 模式）：
1. `arch -x86_64 cmake -B build-x64 -DCMAKE_OSX_ARCHITECTURES=x86_64 ..`
2. `arch -x86_64 cmake --build build-x64 -j8`
3. 跑 `--no-jit` vs `--jit-force` 差分，对比输出

### Layer 3: 分层递进

参考 `docs/design/jit_next_phase.md` F.4 子阶段设计：

| 子阶段 | 指令集 | 测试范围 | 风险 |
|--------|--------|----------|------|
| F.4.1 | 空框架（编译通过但 codegen fail） | 编译测试 | 低 |
| F.4.2 | 整数算术 + 控制流 | 001-010 | 中 |
| F.4.3 | 函数调用 + 参数传递 | 012-013 | 高（calling convention） |
| F.4.4 | 浮点运算 | 002, 006, 011 | 中 |
| F.4.5 | 对象/数组/字符串（CALL_C） | 015-023 | 中 |
| F.4.6 | Deopt + OSR | 033-036 | 高 |
| F.4.7 | 完整功能 | 全部 | — |

**每步纪律**：
- 只加当步需要的指令编码（不贪多）
- 当步新增指令必须有 `test_x64_emit.c` 编码 assert
- 当步 JIT 回归子集全过才推进下一步
- 不改动 ARM64 codegen 代码（隔离风险）

---

## 高频 Bug 模式（跨平台特有）

从 `bug_patterns.md` 和 ARM64 开发历史中提炼：

### BP-F1: REX prefix 遗漏

x86-64 访问 r8-r15 或使用 64-bit 操作数时需要 REX prefix。遗漏导致：
- 操作 r8 实际操作 rax（r8 的低 3 位 = 000 = rax）
- 32-bit 操作替代 64-bit 操作

**防御**：编码函数强制判断 `reg > 7 → emit REX.B/R`。

### BP-F2: ModR/M 特殊情况

- `[rbp]` 无 disp 不可编码（mod=00 + rm=101 表示 [rip+disp32]）→ 必须用 `[rbp+0]`
- `[rsp]` 需要 SIB 字节（rm=100 表示 SIB follows）
- 这两个边界在 ARM64 中不存在，是 x86-64 的高频坑

**防御**：`x64_modrm_mem()` 内部自动处理 rbp/rsp 特殊情况。

### BP-F3: 调用约定差异

| | ARM64 | x86-64 System V |
|---|---|---|
| 整数参数 | x0-x7 | rdi, rsi, rdx, rcx, r8, r9 |
| 浮点参数 | d0-d7 | xmm0-xmm7 |
| 返回值 | x0 (+ x1) | rax (+ rdx) |
| Callee-saved | x19-x28 | rbx, rbp, r12-r15 |
| 红区 | 无 | 128 bytes below rsp |

**防御**：`xir_target_x64.c` 描述所有差异；codegen 只读 target 描述，不硬编码。

### BP-F4: flags 寄存器副作用

x86-64 大部分 ALU 指令修改 flags（ARM64 只有 ADDS/SUBS/CMP 等 S-variant）。
连续的 ADD + CMP 在 x86-64 上 ADD 会覆盖 flags，ARM64 上不会。

**防御**：codegen 不假设 flags 跨指令存活。如需 flags，紧贴 CMP 发射 Jcc。

---

## 开发环境（macOS ARM64 + Rosetta 2）

```bash
# 一次性：创建 x86-64 构建目录
arch -x86_64 cmake -B build-x64 \
  -DCMAKE_OSX_ARCHITECTURES=x86_64 \
  -DCMAKE_BUILD_TYPE=Debug ..

# 日常：构建 + 测试
arch -x86_64 cmake --build build-x64 -j8
arch -x86_64 ctest --test-dir build-x64 --output-on-failure

# 差分测试
arch -x86_64 bash scripts/run_jit_diff_tests.sh
```

**限制**：
- Rosetta 2 翻译的性能数据**不可信**，只做正确性验证
- 性能测试必须在原生 x86-64 机器或 CI 上跑
- W^X 在 Rosetta 下由系统自动处理，无需 pthread_jit_write_protect_np

---

## 验收清单模板

每个 F.4.x 子阶段完成时检查：

- [ ] 新增指令全部有 `test_x64_emit.c` 编码 assert
- [ ] ARM64 构建不受影响（`cmake --build build -j8 && ctest`）
- [ ] x86-64 构建通过（`arch -x86_64 cmake --build build-x64 -j8`）
- [ ] 当步测试范围全过
- [ ] 无新增 cppcheck / clang-tidy 告警
