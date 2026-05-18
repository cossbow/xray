# VM 重构计划（`src/vm/`）

**开发原则**：
- ⚡ **不考虑向后兼容**：Xray 无外部用户，直接采用最佳设计
- ✅ 避免临时 workaround 与兼容层，每一步落到"长期最优"
- ✅ 每个阶段结束必须 **`scripts/run_regression_tests.sh` 全绿** 才能合并
- ✅ 每阶段 commit 粒度控制在 1~3 次；功能子步骤可拆 commit，但不留半成品状态
- ✅ 冷/热路径边界必须明确：热路径宏必须零开销，冷路径函数必须 `__attribute__((noinline))`

**当前基线**（2026-04-19）：
- 总规模：21 文件，~16 660 行
- **重大超限**：
  - `xvm.c` **7 773 行**（上限 3000）
  - `run()` 函数 **~7 200 行单函数**（上限 150）
  - `xvm_cold_paths.c` 3 101 行（轻微超限）
  - `vmcase(OP_CALL)` ~520 行、`vmcase(OP_INVOKE)` ~630 行、`vmcase(OP_RETURN*)` 合计 ~370 行
- 明确缺陷：
  - push-frame 模板重复 10+ 处、invoke error 模板重复 10 处、IC 表懒分配重复 5 处
  - `xr_vm_call_closure` 与 `_ex` 代码漂移风险
  - `xr_vm_call_closure` 不支持 vararg/default params（和 `OP_CALL` 语义不一致）
  - try/catch/finally 内 catch-block-throw 跳过 finally 的语义 bug
  - IC 表每 proto 按指令数全分配（~80KB per proto 典型浪费）
  - JIT/AOT 调用约 200 行直接写在 `OP_CALL` 内，VM↔JIT 层级耦合
  - `VM_CURRENT_CORO` 每次展开都走 TLS；`VM_DEBUG_CHECK` 每指令多级间接

---

## 阶段总览

| # | 阶段 | 文件范围 | 风险 | 预估工时 | 关键收益 |
|---|------|---------|------|---------|---------|
| P1 | API 正确性与对齐 | `xvm_api.c`、`xvm.c`（OP_CATCH/END_TRY） | 中 | 1 d | 修复 try/finally 语义、合并 call_closure/_ex、defer 语义对齐 |
| P2 | 宏抽取与模板去重 | `xvm.c`、`xvm_internal.h` | 低 | 1 d | 消除 push-frame / invoke-error / unary-op 重复，减 ~350 行 |
| P3 | RETURN 三处合并 | `xvm.c` | 低 | 0.5 d | 减 ~150 行，统一 return slot/defer 路径 |
| P4 | OP_INVOKE 与 OP_CHAN_* 热路径合流 | `xvm.c` | 低 | 0.5 d | 减 ~120 行；未来 channel 协议改只改一处 |
| P5 | JIT/AOT 从 OP_CALL 解耦 | `xvm.c`、`jit/xir_jit.h`、`aot/xcgen_bridge.h` | 中 | 1.5 d | VM 主 loop 零 JIT 知识；`OP_CALL` 缩 200 行 |
| P6 | `run()` 按 opcode 分组拆分 | `xvm.c` → `xvm_ops_*.inc.c` | 中 | 2 d | 解决 7 773 行红线；最大可读性收益 |
| P7 | IC 表稀疏化（codegen + VM 协同） | `frontend/codegen/*`、`xic_field.[ch]`、`xic_method.[ch]`、`xvm.c` | 高 | 2.5 d | 典型 proto IC 内存 -80%；cache 命中显著提升 |
| P8 | 热路径微调 | `xvm.c`、`xic_method.h` | 低 | 1 d | `VM_CURRENT_CORO` 缓存、`VM_DEBUG_CHECK` 慢路径剥离、IC hit_count 写放大修复 |
| P9 | cold_paths 再拆分 | `xvm_cold_paths.c` → `xvm_cold_{invoke,prop,coro}.c` | 低 | 0.5 d | 每文件 < 1500 行；职责清晰 |
| P10 | 头文件 / 命名 / 修饰符统一 | `xvm.h`、`xvm_internal.h`、`xvm_helpers.c` | 低 | 0.5 d | 消除 TODO、`XR_FUNC` 修饰缺失、`xvm_ops.c` 职责下沉 |

**总计**：10.5 天有效开发时间。

**依赖图**（必须串行）：`P1 → P2 → P3 → P4 → P5 → P6 → P7`；`P8/P9/P10` 可在 P6 完成后任意顺序并行。

---

## P1：API 正确性与对齐

### 动机

三处相互耦合的正确性问题，必须最先处理，否则后续重构会把 bug 一起放大：

1. **`try/catch/finally` catch-block-throw 跳过 finally**
   - `@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm.c:6102-6131` `OP_CATCH` 进入时直接 `handler->caught = true`；
   - `@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm_api.c:432-436` `xr_vm_throw_exception` 见到 `caught` 就 pop handler；
   - 语义：`try { ... } catch (e) { throw f } finally { F }` 应先执行 `F` 再上抛；现状跳过 `F`。

2. **`xr_vm_call_closure` 与 `OP_CALL` 参数语义不一致**
   - `@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm_api.c:48-52`：`nargs != numparams` 就返回 null；
   - `@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm.c:3655`（`OP_CALL`）允许 `min_params <= nargs <= numparams` 并自动填 null；
   - 后果：`defer f(x)`、higher-order callback 走 C 入口时**完全不支持默认参数和 vararg**。

3. **`xr_vm_call_closure` 与 `xr_vm_call_closure_ex` 代码漂移**
   - 两者差异仅在 `*out_result` 返回与 `XR_VM_BLOCKED` 透传；
   - 分别 90 行 / 85 行，90% 重复，字段初始化已经出现分叉（`frame->flags = 0` / `result_offset = 0` 在一个里有另一个没有）。

### 目标设计

**try/finally 语义修正**：
- 在 `xr_vm_throw_exception` 里 pop 前，若 handler 有 `finally_offset`，先跳到 finally 并保留 `exception` 未消费状态；
- `OP_END_TRY` 读 `handler->exception` 决定是否继续上抛。

**`xr_vm_call_closure` 对齐 `OP_CALL`**：
- 支持 `proto->is_vararg`（多余 args 打包成 rest 数组）；
- 支持 `min_params..numparams` 区间 + 自动 null 填充；
- `nargs < min_params` 时 **抛 `XR_ERR_WRONG_ARG_COUNT` 异常**，不返回 null 掩盖。

**合并 `_ex`**：
- 只保留 `xr_vm_call_closure`，签名统一为：

```c
XR_FUNC XrVMResult xr_vm_call_closure(
    XrayIsolate *isolate,
    XrClosure *closure,
    XrValue *args, int nargs,
    XrValue *out_result   // 可为 NULL，等价原 _ex 的"不关心 blocked"
);
```

- `out_result != NULL` 时等价原 `_ex`；`out_result == NULL` 且执行 block 时打印 warning 并返回 `XR_VM_BLOCKED`（或 panic，由用户政策决定，**默认选 panic**：defer 路径本不应 block）。

### 实施步骤

1. **先写测试**
   - `tests/try_catch_finally_rethrow.xr`：`try { throw 1 } catch (e) { throw e + 1 } finally { log("F") }`，断言输出包含 `F`。
   - `tests/defer_default_args.xr`：`defer f(1)` 其中 `f(a, b = 2)`，断言执行。
2. **修 try/finally**（`xvm.c` OP_CATCH / OP_END_TRY + `xvm_api.c:xr_vm_throw_exception`）。
3. **合并 `xr_vm_call_closure` 和 `_ex`**（删 `_ex`，调用方一次性更新）。
4. **让合并后的 call_closure 复用 `OP_CALL` 同一段 frame 构造**——抽出 `vm_build_frame_for_call(...)` inline helper，被 `OP_CALL` 和 `xr_vm_call_closure` 共享。
5. `scripts/run_regression_tests.sh` 全绿 + 新测试通过。

### 风险
- try/finally 修正可能触发现有测试行为变化。必须**先读测试**确认没有反向断言当前错误语义。
- `_ex` 删除后所有 `#include` 及调用点（JIT、AOT、defer）要全量 grep 更新。

### 验证
- 新增两个测试用例。
- 对比重构前后 `tests/coroutine_safety/` 全部 pass。

---

## P2：宏抽取与模板去重

### 动机

以下三类模板在 `xvm.c` 里大量复制，每次修改 frame 布局 / 错误格式都要改 10+ 处，drift 风险高：

| 模板 | 出现次数 | 示例位置 |
|---|---|---|
| push frame + `goto startfunc` | 10+ | `@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm.c:3505-3512`、`:3881-3885`、`:3946-3951`、`:4625-4629`、`:5094-5097`、`:5512-5515`、`:1093-1097`、`:1138-1141`、cold_paths 多处 |
| builtin invoke + "method not found" 抛错 | 10 | `@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm.c:5170-5277` 全部 `invoke_xxx` label |
| 一元 operator overload 落地 | 2 | UNM `@:1073-1100`、NOT `@:1117-1144` |
| IC field 表懒分配 | 5 | `@:4734-4741`、`:5777-5783`、`:5847-5856`、`:5933-5940`、`:6004-6011` |

### 目标设计

**push-frame 宏**（必须是宏，因为要 `goto startfunc`）：

```c
// xvm.c (run() 内部)
// 在 startfunc 之前定义，所有热路径 push 帧用它
#define VM_PUSH_FRAME_AND_JUMP(closure_, new_base_, result_off_, cstatus_) do { \
    savepc(); \
    if (unlikely(VM_FRAME_COUNT >= XR_FRAMES_MAX)) { \
        VM_RUNTIME_ERROR(XR_ERR_STACK_OVERFLOW, \
            "stack overflow: recursion exceeds %d levels", XR_FRAMES_MAX); \
    } \
    int _fidx = VM_FRAME_COUNT; VM_INC_FRAME_COUNT; \
    XrBcCallFrame *_nf = &VM_FRAMES[_fidx]; \
    _nf->closure = (closure_); \
    _nf->pc = PROTO_CODE_BASE((closure_)->proto); \
    _nf->base_offset = (int)((new_base_) - VM_STACK); \
    _nf->result_offset = (result_off_); \
    _nf->call_status = (cstatus_); \
    _nf->flags = 0; \
    _nf->u.l.pending_operator_check = false; \
    goto startfunc; \
} while(0)
```

**builtin invoke 宏**：

```c
#define VM_INVOKE_BUILTIN_OR_THROW(type_name_str, expr) do { \
    XrValue _r = (expr); \
    if (unlikely(XR_IS_NOTFOUND(_r))) { \
        XrSymbolTable *_st = (XrSymbolTable*)isolate->symbol_table; \
        const char *_mn = xr_symbol_get_name_in_table(_st, method_symbol); \
        VM_RUNTIME_ERROR(XR_ERR_TYPE_NO_METHOD, \
            type_name_str " has no method '%s'", _mn ? _mn : "?"); \
    } \
    R(a) = _r; \
    vmbreak; \
} while(0)
```

**UNM/NOT 统一用 `VM_TRY_UNARY_OP_OVERLOAD`**（已存在于 `@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm.c:428-457`）。

**IC field 表懒分配抽 inline 函数**（放 `xvm_internal.h`）：

```c
static inline XrICField *vm_ic_field_get_or_create(XrProto *proto, int pc_idx) {
    if (unlikely(proto->ic_fields == NULL)) {
        int cache_count = PROTO_CODE_COUNT(proto);
        proto->ic_fields = xr_ic_field_table_new(cache_count);
        for (int i = 0; i < cache_count; i++) xr_ic_field_table_alloc(proto->ic_fields);
    }
    XR_VM_IC_ASSERT_INDEX(pc_idx, proto);
    return xr_ic_field_table_get(proto->ic_fields, pc_idx);
}
```

> **注**：P7 会彻底改造 IC 分配，此处 inline 化是为了让 P3~P6 改动时不继续扩散重复代码。

### 实施步骤

1. 在 `xvm.c` 顶部（startfunc 之前）定义 `VM_PUSH_FRAME_AND_JUMP` / `VM_INVOKE_BUILTIN_OR_THROW`。
2. 替换所有 10+ 处 push-frame：优先从**参数最规整的 OP_INVOKE 静态方法、CALL 普通闭包、SUPERINVOKE、CALLSELF** 开始；struct_ref 构造器和 operator overload 路径涉及更多字段（pending_operator_check 等）最后做。
3. 把 9 个 `invoke_xxx` label 内部逐行替换为 `VM_INVOKE_BUILTIN_OR_THROW`。
4. UNM/NOT 改用已有的 `VM_TRY_UNARY_OP_OVERLOAD`，消除 symbol 字符串 lookup（目前写的是 `"-"` / `"!"` 字符串 register），改用 `SYMBOL_OP_MINUS` / `SYMBOL_OP_NOT` 常量。
5. 所有 IC 表懒分配改用 `vm_ic_field_get_or_create`。
6. `scripts/run_regression_tests.sh`。

### 风险
- `VM_PUSH_FRAME_AND_JUMP` 吞掉了 `u.l.pending_operator_check = false` 的隐式初始化 —— **正好消除** 当前 UNM/NOT / binary op overload 路径遗漏初始化的潜在 bug。
- UNM 的 operator 名称原为字符串 `"-"` 现为 `SYMBOL_OP_MINUS`，需确认 symbol table 有预注册。

### 验证
- 行数下降 ≥ 300 行。
- 二进制 `strings build/xray | grep -c "has no method"` 保持不变（宏展开不影响 string literal 共享性）。

---

## P3：RETURN 三处合并

### 动机

`OP_RETURN` (`@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm.c:4202-4376`) / `OP_RETURN0` (`@:4378-4443`) / `OP_RETURN1` (`@:4445-4571`) 共 370 行。共同逻辑：

- clean up exception handlers belonging to current frame
- defer 执行（含 LIFO 序、bounds check）
- toString print flags
- 写 return slot（根据 `XR_CALL_KEEP_FUNC`）
- 弹 frame
- ctor_call_stack 管理
- module boundary 检查
- 恢复 caller 的 `ci` / `base` / `stack_top`
- pending operator check（conditional jump）
- closure pending handler 跳转

### 目标设计

抽出 **static inline** helper（**inline 关键** —— 必须保持 goto startfunc 行为）：

```c
// 在 run() 内部定义（保留 goto），或抽成带 goto-by-return 约定的 enum:
typedef enum {
    VM_RET_STARTFUNC,
    VM_RET_OK,
    VM_RET_CLOSURE_PENDING,
} VmReturnAction;

static inline VmReturnAction vm_pop_frame_common(
    XrayIsolate *isolate, XrVMContext *vm_ctx,
    XrBcCallFrame *ci, XrValue *base,
    XrValue *return_slot, XrValue ret_val, int nret,
    uint8_t tostring_flags);
```

三个 vmcase 各自：
1. 读 A / nret 局部化
2. 处理 defer / handler cleanup 的**特殊部分**（OP_RETURN0 无 tostring；OP_RETURN1 走 type feedback + struct_ref rescue）
3. 共享路径：`vm_pop_frame_common` → 根据返回值决定 `goto startfunc` / `return XR_VM_OK` / `goto handle_closure_pending`

### 实施步骤

1. 把 `return_with_defer` label 废弃 —— OP_RETURN0/1 直接自带 defer check 且 fall-through 回 OP_RETURN 的模式换成"都调 common helper"。
2. type feedback 的 `xfb_record_return` 下沉到 common helper（OP_RETURN 也应记录，目前只有 OP_RETURN1 记录，不完整）。
3. struct_ref rescue（`@:4493-4521`）下沉到 common helper，OP_RETURN 也适用。
4. 三个 vmcase 变成 ≤ 40 行。

### 风险
- defer stack 边界 check 逻辑复杂，合并后需**重跑 `tests/coroutine_safety/`**。
- type feedback 修正会改变 JIT 编译触发条件 → 需跑 `run_regression_tests.sh` 确认 JIT 路径无回归。

### 验证
- `cd build && ctest -R 'return|defer|closure' --output-on-failure`。
- 对比 disassembly：RETURN/RETURN0/RETURN1 仍产生不同 opcode，只是 handler 共享代码。

---

## P4：OP_INVOKE 与 OP_CHAN_* 热路径合流

### 动机

`@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm.c:4961-5027` 在 `OP_INVOKE` 内联了 `ch.send(v)` / `ch.recv()` 热路径；`@:6743-6856` 的 `OP_CHAN_SEND` / `OP_CHAN_RECV` 逐字重复相同逻辑。

编译器实际对 `ch.send(v)` 生成 OP_INVOKE，对 `ch <- v` 生成 OP_CHAN_SEND —— 两条路径应当共用一套 channel protocol 代码。

### 目标设计

抽 `static inline int vm_channel_send_fast(isolate, ch, send_v, current, frame, pc, a, base)` 和对应 `recv_fast`，两个 vmcase 都调用它，仅包装返回值。

保留 inline 是因为：
- `VM_RUNTIME_ERROR` 需要 run() 上下文（`VM_HANDLER_COUNT` / `goto startfunc`）——helper 返回 enum，调用方处理控制流。

### 实施步骤

1. 定义：
```c
typedef enum {
    VM_CHAN_HOT_DONE,       // 已完成，R[a] 写好，vmbreak
    VM_CHAN_HOT_BLOCKED,    // return XR_VM_BLOCKED
    VM_CHAN_HOT_FALLBACK,   // 走冷路径 vm_invoke_channel / 或抛错
    VM_CHAN_HOT_ERROR,      // VM_RUNTIME_ERROR
} VmChanHotResult;

static inline VmChanHotResult vm_channel_send_hot(
    XrayIsolate*, XrChannel*, XrValue send_v,
    XrCoroutine* current, XrBcCallFrame* frame,
    XrInstruction* pc, XrValue* base, int a);
```

2. `OP_INVOKE` 的 `if (xr_value_is_channel(receiver) && method_symbol == SYMBOL_SEND)` 分支调用 hot helper；失败 fallback 到 `vm_invoke_channel`。
3. `OP_CHAN_SEND` 整个 body 改为调 hot helper（已经只处理 channel，不用 fallback）。
4. recv 对称处理。
5. 冷路径 `vm_invoke_channel`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm_cold_paths.c:40-263`）与 hot helper 保持对等参数顺序，便于维护。

### 风险
- Channel block 语义非常微妙（pre-save frame、`wait_channel`、`recv_slot` 写入时序）。修改时必须**逐字比对两处当前实现**，防止把细微差异丢掉。
- 必须跑 `tests/coroutine_safety/01_channel_deep_copy.xr` ~ `09_*` 全部。

### 验证
- `ctest -R 'channel|coro' --output-on-failure`
- 性能对比：hot helper 必须保持 `__attribute__((always_inline))`（GCC/Clang），实测 `benchmarks/channel_pingpong` 不回退。

---

## P5：JIT/AOT 从 OP_CALL 解耦

### 动机

`@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm.c:3666-3858` `OP_CALL` 中直接嵌入了 ~200 行：
- AOT thunk 调用（raw int64 args 打包）
- JIT 编译触发（hot threshold）
- JIT 调用（`xir_jit_call`）
- Deopt 恢复（mid-function PC 恢复 + frame 重建）
- Deopt 阈值失效 JIT 入口

这让 `src/vm/xvm.c` 直接 `#include "../jit/xir_jit.h"` —— VM 在 L5，JIT 在 L5，但**VM 不应直接感知 JIT 具体 ABI**。此外 AOT 和 JIT 有相近代码但互斥（`#ifdef XRAY_HAS_JIT`），阅读难度高。

### 目标设计

在 JIT/AOT 模块分别提供统一 entry：

```c
// jit/xir_jit.h
typedef enum {
    XJIT_OK,        // jit_result 可用
    XJIT_MISS,      // 未编译、未安装，让 VM 走解释
    XJIT_SUSPEND,   // 协程 suspend；VM 应 return XR_VM_BLOCKED
    XJIT_DEOPT_RESUME,  // jit_result 中带有 recover_pc，VM 应以 recover_pc 建 frame
    XJIT_EXCEPTION, // VM_EXCEPTION 已设置，VM 应按异常流程处理
} XrJitCallResult;

XR_FUNC XrJitCallResult xr_jit_try_call(
    XrayIsolate *isolate,
    XrClosure *closure,
    XrValue *args, int nargs,
    XrValue *out_result,
    int32_t *out_recover_pc    // XJIT_DEOPT_RESUME 时填充
);
```

`OP_CALL` 只需：
```c
XrValue jit_result;
int32_t recover_pc = -1;
XrJitCallResult jr = xr_jit_try_call(isolate, closure, &R(a+1), nargs, &jit_result, &recover_pc);
switch (jr) {
    case XJIT_OK:       R(a) = jit_result; vmbreak;
    case XJIT_SUSPEND:  savepc(); return XR_VM_BLOCKED;
    case XJIT_EXCEPTION: if (VM_HANDLER_COUNT == 0) return XR_VM_RUNTIME_ERROR; goto startfunc;
    case XJIT_DEOPT_RESUME: VM_PUSH_FRAME_AT_PC(closure, new_base, recover_pc, ...);
    case XJIT_MISS:     /* 落回解释器 */ break;
}
```

AOT 同样暴露 `xr_aot_try_call`（即原 `proto->jit_entry` 路径）。VM 依次问：
1. `xr_aot_try_call`（若 `XRAY_HAS_JIT` 未定义）
2. `xr_jit_try_call`
3. 解释器路径

两个 `try_call` 内部自己负责：类型特化 param_types 安装、deopt_count 计数、`proto->jit_entry = NULL` 触发失效。

### 实施步骤

1. 在 `src/jit/xir_jit.c` 里实现 `xr_jit_try_call` 聚合当前 OP_CALL 中的 JIT 全部逻辑。
2. 在 `src/aot/xcgen_bridge.c` 里实现 `xr_aot_try_call`。
3. `OP_CALL` 删去 `#ifdef XRAY_HAS_JIT` 块，整体缩减到 ~50 行。
4. `OP_CALLSELF`（`@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm.c:4003-4140`）复用同一 helper。
5. 验证 `jit_pending` / `type_feedback` 推广逻辑全部转移且不丢失。

### 风险
- **最高风险阶段之一**。JIT deopt 路径涉及 `xir_jit_deopt_recover` 精确恢复 VM slots，需精细验证。
- 建议先在 JIT off 下跑全量回归，再开 JIT 跑 `tests/jit_hot_*`（如有）。

### 验证
- `cmake -DXRAY_HAS_JIT=OFF -B build-nojit && cd build-nojit && ctest`
- `cmake -DXRAY_HAS_JIT=ON -B build-jit && cd build-jit && ctest`
- `scripts/run_regression_tests.sh` 全绿
- benchmarks 无回退

---

## P6：`run()` 按 opcode 分组拆分

### 动机

`xvm.c` 7 773 行 / `run()` ~7 200 行，超项目红线 2.6× / 48×。computed goto 导致无法用普通函数拆。业界标准做法：**`#include` .inc.c 片段**。

### 目标设计

**最终目录结构**：

```
src/vm/
├── xvm.c                    # ~800 行：run() 骨架、startfunc、宏、computed goto 表、入口
├── xvm_ops_data.inc.c       # MOVE/LOADI/LOADF/LOADK/LOADNULL/LOADTRUE/LOADFALSE/BOX*/UNBOX*
├── xvm_ops_arith.inc.c      # ADD/SUB/MUL/DIV/MOD/UNM/NOT/BAND/BOR/BXOR/BNOT/SHL/SHR
├── xvm_ops_cmp.inc.c        # EQ/LT/CMP_*/IS/CHECKTYPE/ISNULL*/TEST/TESTSET/JMP
├── xvm_ops_call.inc.c       # CALL/CALL_KEEP/CALL_STATIC/LOOP_BACK/CALLSELF/TAILCALL/RETURN/RETURN0/RETURN1
├── xvm_ops_invoke.inc.c     # INVOKE/INVOKE_TAIL/SUPERINVOKE/INVOKE_DIRECT/INVOKE_BUILTIN
├── xvm_ops_prop.inc.c       # GETPROP/SETPROP/GETFIELD*/SETFIELD*/GETSUPER/JSON_*/NEWJSON
├── xvm_ops_coll.inc.c       # NEWARRAY/NEWMAP/NEWSET/NEWRANGE/ARRAY_*/MAP_*/RANGE_*/STRBUF_*/SLICE
├── xvm_ops_except.inc.c     # TRY/CATCH/FINALLY/END_TRY/THROW/ASSERT*
├── xvm_ops_coro.inc.c       # GO/GO_INVOKE/SPAWN_CONT/AWAIT*/YIELD/CANCELLED/CORO_CTRL/LOCK*/GET_LOCAL/SET_LOCAL/SET_PRIORITY
├── xvm_ops_chan.inc.c       # CHAN_NEW*/CHAN_SEND/CHAN_RECV/CHAN_TRY_*/CHAN_SEND_TIMEOUT/CHAN_RECV_TIMEOUT/CHAN_CLOSE/CHAN_IS_CLOSED/SELECT_*
├── xvm_ops_class.inc.c      # CLASS_CREATE_FROM_DESCRIPTOR/CLINIT_CALL/INHERIT/ABSTRACT_ERROR/SET_STORAGE_CTX/TO_SHARED/MAP_SETKS
├── xvm_ops_module.inc.c     # IMPORT/EXPORT/EXPORT_ALL/REGEX_COMPILE
├── xvm_ops_misc.inc.c       # SPILL/RELOAD/NEW_STRUCT/STRUCT_*/TARRAY_*/TFIELD_*/INST_TYPE_ARGS/DEFER/BYTES_NEW/SCOPE_*/TIME_AFTER/SLEEP/CHR/PRINT/TYPEOF/TYPENAME/DUMP/TOINT/TOFLOAT/TOSTRING/TOBOOL/COPY/NOP/GETBUILTIN/GETSHARED/SETSHARED/CLOSURE/UPVAL_GET/CELL_*/ENUM_*/SUBSTRING/STR_REPEAT/INDEX_*
```

**`xvm.c` 最终骨架**：

```c
XrVMResult run(XrayIsolate *isolate, XrVMContext *vm_ctx) {
    // ... 所有宏定义、局部变量、VM_STACK_CHECK ...

startfunc:
    // ... frame 初始化、struct_area、defer marks ...

#if XR_USE_COMPUTED_GOTO
    #include "xvm_jumptab.h"
    i = *pc++; VM_DEBUG_CHECK();
    OpCode _first_op = GET_OPCODE(i); VM_PROFILE_COUNT(_first_op); vmdispatch(_first_op);
#else
    for (;;) {
        i = *pc++; VM_DEBUG_CHECK(); OpCode op = GET_OPCODE(i); VM_PROFILE_COUNT(op);
        switch (op) {
#endif

#include "xvm_ops_data.inc.c"
#include "xvm_ops_arith.inc.c"
#include "xvm_ops_cmp.inc.c"
#include "xvm_ops_call.inc.c"
#include "xvm_ops_invoke.inc.c"
#include "xvm_ops_prop.inc.c"
#include "xvm_ops_coll.inc.c"
#include "xvm_ops_except.inc.c"
#include "xvm_ops_class.inc.c"
#include "xvm_ops_module.inc.c"
#include "xvm_ops_coro.inc.c"
#include "xvm_ops_chan.inc.c"
#include "xvm_ops_misc.inc.c"

#if !XR_USE_COMPUTED_GOTO
        default: VM_RUNTIME_ERROR(XR_ERR_TYPE_MISMATCH, "Unknown opcode %d", op);
        }
    }
#endif

handle_closure_pending:
    // ...

    // undefs ...
}
```

**CMake 处理**：`xvm_ops_*.inc.c` 不单独编译（从 `add_library` 列表排除），仅被 `xvm.c` `#include`。在 `src/vm/CMakeLists.txt`（若无则在 `src/CMakeLists.txt` 源码列表）明确 skip：

```cmake
set_source_files_properties(
    src/vm/xvm_ops_data.inc.c
    src/vm/xvm_ops_arith.inc.c
    ... 
    PROPERTIES HEADER_FILE_ONLY TRUE)
```

**编辑器 / IDE 支持**：在每个 `.inc.c` 顶部加：

```c
/* This file is #include'd by xvm.c inside run(). Do not compile standalone.
 * All referenced identifiers (R, RA, KB, VM_STACK, base, pc, i, ci, cl, frame,
 * isolate, vm_ctx, VM_RUNTIME_ERROR, VM_GC_SAFEPOINT, ...) are in scope from
 * the enclosing run() function. */
#ifdef __clang__
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wunused-macros"
#endif
```

### 实施步骤

1. **先建空壳 + 一个最小分组（`data`）**：把 7 个 MOVE/LOADx 搬过去，`scripts/run_regression_tests.sh` 必须绿。
2. 若通过，按文件列表逐组搬迁（每组一个 commit）：
   - `arith` → `cmp` → `coll` → `except` → `module` → `misc` → `class` → `prop` → `chan` → `coro` → `call` → `invoke`
   - **顺序策略**：先搬指令独立性高、局部变量少的组；call/invoke 最后做（它们用了 ~50 个局部变量 label）。
3. 每次搬完单独 commit：`vm: move <group> opcodes to xvm_ops_<group>.inc.c`。
4. 最终验证 `xvm.c` ≤ 1000 行，每个 `.inc.c` ≤ 2000 行。
5. 验证 disassembly 无变化（编译器应完全内联）。

### 风险
- computed goto 的 label 跨文件可见性：GCC/Clang 支持跨 `#include` 的 label 引用，但需要确保 `disptab` 在 `.inc.c` 之前展开。此处通过 `#include "xvm_jumptab.h"` 放在 startfunc 之后、`.inc.c` 之前自然满足。
- `invoke_xxx` / `op_call_closure` / `op_call_cfunc` / `return_with_defer` 等跨 case 的 label 需留在它们被使用的文件内（或放 `invoke.inc.c` 中同组 case）。
- Debug 编译时间可能略增：`#include` 解析多次。可以接受。
- IDE（VS Code / Clangd）需要 `.inc.c` 作为 C 源；可能需要配置 `.clangd` 或 `compile_commands.json` hint。

### 验证
- `ctest -j8` 全绿
- 对比 `objdump -d build/libxray.a | grep -c "startfunc:"` 确认没重复展开
- 性能 benchmarks（`benchmarks/*`）不回退

---

## P7：IC 表稀疏化（codegen + VM 协同）

### 动机

当前每个 `XrProto` 有：
- `ic_methods->caches[PROTO_CODE_COUNT(proto)]`，每个 `XrICMethod` ≈ 96 B
- `ic_fields->entries[PROTO_CODE_COUNT(proto)]`，每个 `XrICField` ≈ 84 B
- 按每指令分配，但实际只有 `GETPROP/SETPROP/GETFIELD_IC/INVOKE/INVOKE_TAIL/SUPERINVOKE` 需要 IC；
- 典型 proto 1000 指令，IC 关键指令 ~100 条，实际浪费 90% 内存（**每 proto 约 140 KB**）。

### 目标设计

**核心改动**：在 codegen 阶段为每个需要 IC 的指令分配一个递增 `ic_index`，并把它编码到 instruction 的某个操作数里（或存一个 per-proto sidecar）。

**方案 A：sidecar 数组**（推荐，instruction 格式不变）

```c
// xchunk.h
struct XrProto {
    ...
    int     ic_method_count;    // 实际需要方法 IC 的指令数
    int     ic_field_count;     // 实际需要字段 IC 的指令数
    int32_t *ic_method_index;   // pc_idx -> ic_method slot（-1 表示无）
    int32_t *ic_field_index;    // pc_idx -> ic_field slot（-1 表示无）
    ...
};
```

- codegen 扫描 bytecode 给每条 OP_GETPROP 等写入正确 slot；
- VM 访问：

```c
int ic_slot = proto->ic_field_index[pc - PROTO_CODE_BASE(proto) - 1];
XrICField *cache = (ic_slot >= 0) ? &proto->ic_fields->caches[ic_slot] : NULL;
```

- sidecar 大小 = `instruction_count * 4 B`（而原来是 `instruction_count * 84 B`）。

**方案 B：操作数位**（更快但侵入 instruction 格式）

instruction 里 GETPROP/SETPROP 的 C 操作数（8 bit）其实是 const index，可以把 IC slot 另存一个 short 跟在指令后面（`XR_IC_HINT` pseudo-instruction）。不推荐，改动面过大。

**选 A**。

**codegen 改动**：`src/frontend/codegen/` 里产出 bytecode 的地方：
- 每当 emit `OP_GETPROP` / `OP_SETPROP` / `OP_GETFIELD_IC` 时调用 `codegen_alloc_ic_field(codegen)` 得到 slot；
- 每当 emit `OP_INVOKE` / `OP_INVOKE_TAIL` / `OP_SUPERINVOKE` 时调用 `codegen_alloc_ic_method(codegen)`。

**VM 改动**：
- `vm_ic_field_get_or_create` 改为 `vm_ic_field_get(proto, pc_idx)`，直接查 sidecar；未命中返回 `NULL` → VM 跳过 IC（但 P2 之后 IC miss 写回仍有效）。
- `xr_ic_field_table_new` 在 VM 端只分配 `proto->ic_field_count` 条，不再按指令数。

### 实施步骤

1. **先准备 bench**：记录改造前 `build-Release` 下 `tests/demos/cluster_blockchain_node.xr` 等大示例启动 RSS、方法调用总耗时。
2. **codegen 侧**：
   - 在 `XrCodegen` 结构体加 `int next_ic_field_slot; int next_ic_method_slot;`（通过工作 `src/frontend/codegen/xcodegen.h` 查找）。
   - emit 三个 OP 时递增并记录到临时数组。
   - 生成 `XrProto` 时把临时数组 copy 到 `proto->ic_field_index`。
3. **VM 侧**：
   - `xr_ic_field_table_new(int count)` 参数改为实际 count；
   - VM `OP_GETPROP/SETPROP/GETFIELD_IC` 改走 `vm_ic_field_get(proto, pc_idx)`。
4. **IC method 同等处理**。
5. **弃用 `xr_ic_method_table_alloc` 的按需 grow**：改造后表大小在 proto 加载时就完整分配，无需 `realloc`。
6. **验证 IC 命中率**：用 `XR_DEBUG_INLINE_CACHE` 构建，对比改造前后 hit rate 应一致。
7. **清理 `xr_ic_field_table_new` 里初始循环 alloc**：不再 `for (i=0; i<cache_count; ++i) xr_ic_field_table_alloc(...)`（因为 count 已是精确值）。

### 风险
- **最高风险阶段**。codegen 和 VM 必须同步改，不能分开 merge。
- bytecode serialize/deserialize（若有 `.xbc` 文件格式）需要带上新 sidecar。
- JIT 也读 IC type feedback（`@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xic_method.h:217-245`），需确认 JIT 侧读的是 `proto->ic_methods->caches[ic_slot]`，不是 `pc_idx`。

### 验证
- 内存占用对比：`/usr/bin/time -l build/xray tests/demos/cluster_blockchain_node.xr`，期望 peak RSS 下降 10~30%。
- IC 命中率：`cmake -DXR_DEBUG_INLINE_CACHE=ON`，前后对比日志一致。
- `ctest` 全绿。

---

## P8：热路径微调（可选）

### 动机

P1~P7 完成后的一些**低风险、低工时、收益 2~5% 级别**的微调：

| 子项 | 位置 | 效果 |
|---|---|---|
| `VM_CURRENT_CORO` 缓存到局部 `register XrCoroutine *cur_coro` | `xvm.c:200-206` | 每次避免 TLS 访问 |
| `VM_DEBUG_CHECK` 用 `isolate->debug_enabled` 单 bit 快速短路 | `xvm.c:332-380`、`xray_debug_hooks.h` | debug off 时热循环 -3% |
| IC `hit_count` 写放大 → 相对写 miss 时才累加，total_count 改采样（每 N hits +1） | `xic_method.h:134-143` | 减少共享 cacheline 污染 |
| `PROTO_SYMBOL` lookup 缓存 | `xvm.c:4933、5734、5925` | 少一次数组间接 |
| OP_ADD 混合路径 int+int → int（当前 TONUMBER 会强制 double） | `xvm.c:627-633` | 类型精度保留；配合后续 Int32 专有类型 |

### 实施步骤

每个子项独立一个 commit；每个 commit 先跑 `scripts/run_regression_tests.sh`；微调前后跑 `benchmarks/` 对比。

### 风险 / 验证
- 风险低，单项回滚容易。
- Benchmarks：`fib(35)` / `sudoku` / `binarytrees` 不回退，debug off 时 `aho-corasick` 有提升。

---

## P9：`xvm_cold_paths.c` 再拆分

### 动机

3101 行文件仍然略超 3000 红线，并且职责混杂：channel / task / invoke / prop / await / select / go / coro_ctrl。

### 目标设计

```
src/vm/
├── xvm_cold_invoke.c    # vm_invoke_{channel,task_handle,coro_handle,enum,class,module}, vm_superinvoke
├── xvm_cold_prop.c      # vm_{set,get}prop_type_dispatch, vm_{set,get}prop_instance_{getter,setter}
├── xvm_cold_coro.c      # vm_go, vm_go_invoke, vm_spawn_cont, vm_await*, vm_select_block, 
│                        # vm_chan_{send,recv}_timeout, vm_coro_ctrl, vm_collect_all_coros
└── xvm_cold_paths.h     # 仍是统一声明（拆 .c 不拆 .h，便于 `#include` 一次搞定）
```

### 实施步骤

1. 三个新 .c 文件创建，把 cold_paths.c 里函数按上表归位。
2. `xvm_cold_paths.h` 保持不变（三个 .c 都 `#include` 它）。
3. CMake 源码列表增加新文件。

### 风险 / 验证
- 风险低（纯机械搬迁，没有逻辑改动）。
- `ctest` 全绿。

---

## P10：头文件 / 命名 / 修饰符统一

### 动机

现状累积的小债务：
- `run()` 未加 `XR_FUNC` 修饰符（违反 user rule）
- `VM_DEBUG_PRINT` 在 `xvm.c` 和 `xvm_internal.h` 各定义一次
- `xvm_helpers.c` 含 `xr_runtime_error` 职责模糊
- `xvm_ops.c` 的 `deep_compare` 等与 `src/runtime/value/` 语义重叠
- `xvm_builtins.c` handler 签名统一但很多 handler 用 `(void)isolate` 等充数
- 3 处 TODO（最老的 `@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm_helpers.c:282`）

### 目标设计

1. **所有非 static VM 函数加 `XR_FUNC`**（`run()` / internal helpers）。
2. **`VM_DEBUG_PRINT` 只保留在 `xvm_internal.h`**，`xvm.c` 删除。
3. **下沉 `xvm_ops.c` 到 `src/runtime/value/`**：
   - `vm_values_equal` / `vm_values_equal_deep` / `vm_numeric_less*` / `vm_add_operation` 等下沉为 `xr_value_equal` / `xr_value_deep_equal` / `xr_value_lt` / ...；
   - VM 直接调用 runtime API，删除 `xvm_ops.c`。
4. **`xr_runtime_error` 移到 `src/runtime/xerror_report.c`**（新文件或现有 log 模块）。
5. **拆分 handler 签名**：
   - `typedef XrValue (*MethodHandlerSimple)(XrayIsolate*, XrValue receiver);`
   - `typedef XrValue (*MethodHandlerArgs)(XrayIsolate*, XrValue receiver, XrValue *args, int argc);`
   - handler 表里各自按需使用；dispatch code 根据 handler type 标志位调用。
6. **清理 TODOs**：`xvm_helpers.c:282` 的 `"TODO: Migrate all built-in methods to xclass_system.c"` —— 要么做，要么删除 comment（建议删除：builtin dispatch 用 symbol 已经最优）。

### 实施步骤

1. `grep -rn "^[A-Za-z].*vm_[a-z].*(" src/vm/ | grep -v static | grep -v XR_FUNC | grep -v XRAY_API` 找到所有无修饰符导出，逐个加 `XR_FUNC`。
2. `xvm_ops.c` 函数下沉 —— 跑 `scripts/run_regression_tests.sh`。
3. `xvm_helpers.c` 里 `xr_runtime_error` 下沉到 `src/runtime/xerror_report.c`。
4. handler 签名拆分（谨慎：`xvm_builtins.c` 1500 行，可作为 P10 的单独子 commit）。

### 风险 / 验证
- 风险低，大多数是机械移动。
- handler 签名拆分需要跑 benchmarks 确认 dispatch 开销没增加（若用函数指针 + type 标志位判断，可能多一个分支；建议保持统一签名 + `(void) arg` cast，不拆）。

---

## 验证基线总表

| 脚本 | 触发频率 | 目的 |
|---|---|---|
| `cd build && ctest --output-on-failure` | 每次 commit 前 | 单元与基础回归 |
| `scripts/run_regression_tests.sh` | 每个阶段结束 | 完整回归（含 .xr 端到端） |
| `cmake -DXR_ENABLE_VM_PROFILER=ON` + 运行典型脚本 | P6/P7/P8 前后 | 指令分布对照 |
| `cmake -DXRAY_HAS_JIT=ON/OFF` 双路径测试 | P5、P7 前后 | JIT/AOT ABI 正确性 |
| `/usr/bin/time -l` | P7 后 | RSS 下降对照 |
| `/usr/bin/time -p` + benchmarks | P4/P5/P6/P8 后 | 指令/调用热路径不回退 |

---

## 里程碑与节奏

- **Week 1**：P1 + P2 + P3（关键正确性 + 模板去重 + RETURN 合并）—— 合并前 `xvm.c` 预计降到 ~6 500 行
- **Week 2**：P4 + P5（热路径合流 + JIT 解耦）—— `xvm.c` 预计 ~6 100 行
- **Week 3**：P6（run() 拆分）—— `xvm.c` ≤ 1000 行，出现 13 个 `.inc.c`
- **Week 4**：P7（IC 稀疏化）—— 核心内存优化
- **Week 5**：P8 + P9 + P10（收尾微调）—— 达成全模块规模红线清零

全阶段结束的 KPI：

| 指标 | 当前 | 目标 |
|---|---|---|
| `src/vm/` 总行数 | 16 660 | ≤ 14 000 |
| `xvm.c` | 7 773 | ≤ 1 000 |
| 最大函数（`run()`） | ~7 200 | ≤ 150 |
| `vmcase(OP_CALL)` | ~520 | ≤ 80 |
| `vmcase(OP_INVOKE)` | ~630 | ≤ 120 |
| RETURN 三处合计 | ~370 | ≤ 150 |
| 典型 proto IC 内存 | ~140 KB | ≤ 15 KB |
| `VM_DEBUG_CHECK` 热开销（debug off） | ~10 cycle | ≤ 3 cycle |

---

## 附录 A：建议的 git 提交模板

```
vm: <short summary>

<body>

Phase: P<N>
Test: scripts/run_regression_tests.sh (pass)
Benchmarks: fib(35) -0.2%, sudoku -0.5% (acceptable)
```

## 附录 B：决策记录

- **为什么不用函数而用 `.inc.c`**：computed goto 必须在单函数体内；label 不能跨函数。
- **为什么 IC 选方案 A（sidecar）而非方案 B（instruction 内嵌）**：方案 B 需要改 instruction 编码、bytecode 格式、disassembler 等多处；方案 A 只加两个数组即可。
- **为什么 `xr_vm_call_closure` 合并 `_ex` 而非反过来**：主名字更短、API 更友好；`_ex` 的 `out_result` 可选语义更自然。
- **为什么 P1 先于 P6**：语义 bug 必须先修，否则 P6 拆分会把 bug 复制到新文件，后面 bisect 困难。
- **为什么 P5（JIT 解耦）先于 P6（拆分）**：`OP_CALL` 内 JIT 代码占 200 行，先解耦使 `OP_CALL` 足够小，`xvm_ops_call.inc.c` 才合理（否则单文件又超限）。

---

**最后更新**：2026-04-19
**适用版本**：`@/Users/xuxinglei/workspace/xray-lang/xray/CMakeLists.txt` 定义的 `Xray VERSION` 字段之后
