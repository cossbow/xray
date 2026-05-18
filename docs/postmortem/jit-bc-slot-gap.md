# Postmortem: JIT bc_slot Metadata Gap (commit 90ef326)

## Bug 本质

**Silent Metadata Gap** — 关键元数据字段 (`bc_slot`) 被留在默认哨兵值 (-1)，
所有下游消费者遇到哨兵值时静默跳过 (`if (bc_slot < 0) continue`)，
没有任何断言或日志。

### 直接 root cause

`xi_to_xm_lower` 分配 param vregs 时未设置 `bc_slot`，导致：
1. Codegen prologue 跳过 `slot_runtime_tags[]` 的参数初始化
2. `emit_call_args_from_pool` 的动态 tag patch 也被跳过
3. `call_arg_tags[]` 全为 `0xFF (UNKNOWN)`
4. 指针参数被误标记为 `XR_TAG_I64` → native 函数类型检查失败

### 现象

`regex.compile(pattern)` 在 JIT 编译的 `match_ok` 中返回 null。
表现为 `1120_regex_re2_compat.xr` 的 flaky test failures。

---

## 同类隐患排查结果

### 1. X64 后端缺少 param_tags → slot_runtime_tags 初始化 ⚠️ 确认

**文件**: `xm_codegen_x64.c` `x64_emit_prologue()` (line 311-434)

ARM64 prologue (`xm_codegen.c:1009-1027`) 有如下初始化：
```c
for (uint32_t i = 0; i < nparams && i < 8; i++) {
    if (ctx->func->vregs[i].rep != XR_REP_TAGGED)
        continue;
    int16_t bc_slot = ctx->func->vregs[i].bc_slot;
    if (bc_slot < 0 || bc_slot >= 256) continue;
    // load param_tags[i] → slot_runtime_tags[bc_slot]
}
```

**X64 prologue 完全没有这段代码。** 在 x86-64 上运行时，TAGGED 参数的
`slot_runtime_tags[]` 将保持未初始化状态，导致与 ARM64 修复前相同的 bug。

**修复优先级**: P1（目前 x64 backend 未在生产环境使用，但一旦启用必然命中）

### 2. Inlining pass 新建 vregs 无 bc_slot

**文件**: `xm_pass_advanced.c:59`

```c
vreg_map[i] = xm_new_vreg(caller, callee->vregs[i].rep);
// propagates heap_type, xrtype, callee_proto, shape_hint, layout, struct_idx
// does NOT propagate bc_slot
```

如果 inlined callee 内部有 CALL_C 指令，其参数 vreg 来自 inlined 新建的 vreg，
且这些 vreg 的 ctype 是 UNKNOWN → tag 为 0xFF → 动态 patch 需要 bc_slot → 失败。

**风险评估**: 中等。当前 inlining 倾向于小函数，call args 通常来自 caller 已有 vregs。
但随着 inlining scope 增大，必然命中。

### 3. REDEFINE / scalar replacement 新建 vregs 无 bc_slot

**文件**: `xm_pass_type.c:1560`, `xm_pass_advanced.c:1032`

这些 vregs 通常有具体类型（narrowed type / I64），不触发动态 tag patch。**风险低。**

### 4. `bc_slot < 0` 静默跳过点 — 完整清单 (15+ 处)

| 文件 | 行号 | 场景 |
|------|------|------|
| `xm_codegen.c` | 1021 | prologue param_tags → slot_runtime_tags init |
| `xm_codegen_call.c` | 99 | CALL_C arg dynamic tag patch |
| `xm_codegen_call.c` | 185 | CALL_C result tag writeback |
| `xm_codegen_call.c` | 408 | CALL_SELF result tag |
| `xm_codegen_call.c` | 455 | CALL_SELF tag reload after live-reg restore |
| `xm_codegen_call.c` | 615 | CALL_C_LEAF result tag |
| `xm_codegen_call.c` | 769 | CALL_KNOWN result tag |
| `xm_codegen_x64_call.c` | 114 | x64 CALL_C result tag |
| `xm_codegen_x64_call.c` | 320 | x64 CALL_SELF result tag |
| `xm_codegen_x64_call.c` | 506 | x64 CALL_KNOWN result tag |
| `xm_codegen_x64_call.c` | 839 | x64 CALL_KNOWN_DIRECT result tag |
| `xm_codegen_x64_call.c` | 930 | x64 dynamic arg tag patch |
| `xm_codegen_x64_ins.c` | 1207 | x64 SETPROP arg tag |
| `xm_codegen_x64_ins.c` | 1374 | x64 SUSPEND result tag |
| `xm_codegen_x64_mem.c` | 521 | x64 STORE arg tag |
| `xm_pass_cfg.c` | 142 | copy propagation bc_slot inheritance |

每一个跳过点在 bc_slot == -1 时都**完全静默**，不报错、不降级、不日志。

---

## 为什么这个 bug 难以发现

1. **Flaky 表现**: 只在 bg JIT 编译时出现（call_count 达到阈值后），测试通常在 JIT
   介入前跑完 → 绝大多数运行通过
2. **无断言保护**: `bc_slot == -1` 是合法的哨兵值，所有消费者认为这是"不需要处理"
3. **无日志**: 跳过时没有任何 warning/trace
4. **跨组件**: bug 在 lowering (xi_to_xm.c) 中产生，在 codegen 中表现，
   在 runtime helper 中导致错误行为，最终在 test 中体现为 wrong result

---

## 预防措施

### A. Debug 断言（立即实施）

在 `emit_call_args_from_pool` 的动态 patch 循环中，当 vreg 的 ctype 为 UNKNOWN
且 bc_slot == -1 时，加 `XR_DCHECK` 断言：

```c
// This vreg has no compile-time type and no runtime tag source.
// It WILL be reconstructed as XR_TAG_I64 which may be wrong.
XR_DCHECK(bc_slot >= 0,
    "emit_call_args_from_pool: arg %d has UNKNOWN ctype "
    "and bc_slot=-1, tag will be wrong", i);
```

这不会影响正确代码，但会在开发阶段立即捕获同类 bug。

### B. X64 prologue 补齐 param_tags init（P1 修复）

从 `xm_codegen.c:1009-1027` 复制 ARM64 的 `slot_runtime_tags` init 逻辑到
`x64_emit_prologue`。

### C. `xm_new_vreg` 后的 bc_slot invariant 文档化

在 `xm.h` 的 `XmVReg` 结构体注释中明确：
- `bc_slot == -1` 意味着此 vreg **没有 slot_runtime_tags[] 位**
- 如果此 vreg 将来作为 CALL_C arg 且 ctype 为 UNKNOWN → tag 丢失
- 所有创建 vreg 的代码必须评估是否需要设置 bc_slot

### D. 长期：消除对 bc_slot 的依赖

当前 `slot_runtime_tags[bc_slot]` 机制本质是用 bytecode slot 索引来间接传递
runtime type info。更健壮的方案：

- **方案 1**: 在每个 CALL_C arg store 指令旁直接 store tag byte（不依赖 bc_slot）
- **方案 2**: 强制所有 CALL_C arg 的 ctype 在 type pass 中 resolve 为具体类型
  （消除 UNKNOWN → 消除对动态 patch 的需求）

方案 2 更优：如果 type pass 能为每个 CALL_C 的 arg 推断出具体 ctype，
编译时就能 bake 正确的 tag byte，完全不需要 runtime tag 查找。
