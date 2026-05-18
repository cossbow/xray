# JIT↔VM 边界接口文档

本文档描述 xray JIT 编译器与 VM 解释器之间的边界协议，包括调用约定、入口/出口点、反优化（deopt）、OSR、GC 安全点等。

## 1. ARM64 调用约定

### 1.1 JIT 函数签名

```c
typedef struct { int64_t payload; int64_t tag; } XrJitResult;
typedef XrJitResult (*XirJitFn)(intptr_t coro, int64_t *args);
```

| 寄存器 | 用途 |
|--------|------|
| `x0` | 入参：`XrCoroutine*`；返回：`payload`（原始 int64/f64 bits/ptr） |
| `x1` | 入参：`int64_t* args`（原始参数数组指针）；返回：`tag`（XrValue tag 或 0xFF=UNKNOWN） |
| `x16` | CALL_C stub：C 函数指针 |
| `x17` | CALL_C stub：额外参数 |
| `x18` | 保留（平台 TLS） |
| `x19` | JIT_CTX_REG：`coro->jit_ctx`（XrJitScratch*） |
| `x28` | FP（JIT 帧基指针） |
| `x20-x27` | 分配寄存器（callee-saved） |
| `x1-x15` | 分配寄存器（caller-saved，CALL_C 前保存到栈） |
| `d0-d7` | 浮点分配寄存器（caller-saved） |
| `d8-d15` | 浮点分配寄存器（callee-saved） |

### 1.2 返回值协议

JIT 函数通过 `XrJitResult` 返回：
- `payload`（x0）：原始 8 字节值（int64/double bits/pointer）
- `tag`（x1）：XrValue tag（0-15），或 `XR_RTAG_UNKNOWN`（0xFF）

特殊 payload 哨兵值：
- `XIR_DEOPT_MARKER` = `0xDEAD0001DEAD0001`：触发反优化
- `XIR_SUSPEND_MARKER` = `0xDEAD0002DEAD0002`：await 挂起

## 2. 入口点（Interpreter → JIT）

### 2.1 正常调用 `xir_jit_call()`

```
VM OP_CALL → xir_jit_call(jit_entry, coro, args, nargs, return_type_info, &result)
```

流程：
1. 重置 `jit_frame_depth = 0`
2. 设置 `active_stack_map` 和 `active_safepoint_id`
3. **类型守卫**：验证参数 tag 与编译时特化类型匹配，不匹配则 bail out
4. 存储 `param_tags[]`（用于 JIT 中的 null 检查）
5. **拆箱**：从 `XrValue` 提取 8B payload 到 `raw_args[]`
6. 调用 JIT 函数：`((XirJitFn)jit_entry)(coro, raw_args)`
7. **重建**：根据返回 tag 将 payload 恢复为 `XrValue`

### 2.2 OSR 入口 `xir_jit_osr_enter()`

```
VM 循环回边 → xir_jit_osr_trigger() → xir_jit_osr_enter(osr_entry, coro, values, return_type, &result)
```

- `values[]`：字节码寄存器的原始 int64 值（按 bc_slot 索引）
- OSR stub 根据 `XirOsrEntry.slots[]` 将值加载到物理寄存器
- 每个 OSR entry 关联一个循环头基本块

### 2.3 恢复入口 `xir_jit_resume()`

```
Worker wakeup → xir_jit_resume(coro, &result)
```

- await 挂起后恢复执行
- 从 `coro->jit_resume_entry` 进入恢复 stub
- 恢复 stub 从 `coro->jit_suspend_regs[]` 重新加载寄存器

## 3. 出口点（JIT → Interpreter）

### 3.1 正常返回

JIT epilogue 设置 `x0=payload`, `x1=tag`，执行 `ret`。调用方根据 tag 重建 `XrValue`：

| tag | 重建方式 |
|-----|----------|
| `XR_TAG_F64` (12) | `memcpy(&result.f, &payload, 8)` |
| `XR_TAG_PTR` (6) | 设置 tag=PTR, heap_type 从 GC header 读取；payload=0 则 tag=NULL |
| `XR_TAG_BOOL` (3) | payload 为 0/1 |
| `XR_TAG_I64` (6) | 直接赋值 |
| `XR_RTAG_UNKNOWN` (0xFF) | 根据 `return_type_info` 推导 tag |

### 3.2 Deopt 返回

Guard 失败时：
1. JIT deopt stub 保存所有活跃寄存器到 `jit_ctx->deopt_regs[]` / `deopt_fp_regs[]`
2. 设置 `jit_ctx->deopt_id` 为 deopt 点 ID
3. 返回 `XIR_DEOPT_MARKER`

调用方检测到 deopt 后调用 `xir_jit_deopt_recover(coro, frame, maxstack)`：
- 查找 deopt table entry（按 `deopt_id` 索引）
- 遍历 `entry->slots[]`，从保存的寄存器/spill/常量中恢复值
- 通过 `deopt_reconstruct()` 将原始值 + xr_tag + XIR 类型重建为 `XrValue`
- 返回恢复的字节码 PC

### 3.3 Suspend 返回

`XIR_SUSPEND` 指令：
1. 保存活跃寄存器到 `coro->jit_suspend_regs[]`
2. 设置 `coro->jit_resume_entry` 和 `jit_resume_proto`
3. 返回 `XIR_SUSPEND_MARKER`

## 4. JIT→JIT 调用

### 4.1 CALL_DIRECT（通过 `xr_jit_call_func`）

JIT 代码通过 CALL_C stub 调用 `xr_jit_call_func(coro, nargs_encoded)`：
1. 参数预存在 `jit_ctx->call_args[0..n]`（[0]=closure, [1..n]=args）
2. 如果 callee 有 `jit_entry`：直接 JIT→JIT 调用
3. 否则：尝试按需编译，失败则回退到 VM 解释器
4. **deopt_id 清零**：callee deopt 后清除 `deopt_id = 0`，防止泄漏到 caller

### 4.2 CALL_SELF（递归自调用）

`xr_jit_call_self()` 直接调用同一 proto 的 JIT 代码，跳过参数检查。

## 5. GC 安全点协议

### 5.1 Stack Map

编译时为每个 CALL_C 位置生成 `XrStackMapEntry`：
- `reg_bitmap`：哪些分配寄存器持有 PTR 类型 vreg
- `spill_bitmap`：哪些 spill slot 持有 PTR 类型 vreg

### 5.2 Safepoint 流程

CALL_C 前：
1. 将 PTR 寄存器写回 spill slot（`emit_ptr_spill_writeback`）
2. 记录 safepoint ID 到 `jit_ctx->active_safepoint_id`
3. 保存帧指针到 `jit_ctx->jit_frame_sp`

GC 扫描时：
- 读取 `active_safepoint_id` 定位 stack map entry
- 按 bitmap 扫描 spill slot 中的 GC root

### 5.3 Guard Page Safepoint

JIT 轮询一个 guard page 地址（`jit_ctx->safepoint_page`），GC 通过 `mprotect` 触发 SIGSEGV 实现安全点。

## 6. Deopt Table 结构

```c
typedef struct {
    int16_t  bc_slot;      // 字节码寄存器索引
    uint8_t  type;         // XrRep (I64/F64/PTR/TAGGED)
    uint8_t  loc_kind;     // REG / FP_REG / SPILL / CONST_*
    uint8_t  xr_tag;       // XrValue tag (0-15) 或 0xFF=unknown
    union { uint8_t phys_reg; int16_t spill_offset; int64_t const_i64; ... } loc;
} XirRtDeoptSlot;

typedef struct {
    uint32_t       bc_pc;       // 恢复的字节码 PC
    uint16_t       nslots;      // 活跃 slot 数量
    uint16_t       deopt_id;    // deopt stub ID
    XirRtDeoptSlot slots[32];   // 每个活跃 bc_slot 的位置信息
} XirRtDeoptEntry;
```

每个函数最多 `XIR_MAX_RT_DEOPT_ENTRIES` (64) 个 deopt 点。

## 7. OSR Entry 结构

```c
typedef struct {
    uint32_t block_id;       // 循环头基本块 ID
    uint32_t bc_offset;      // 循环头的字节码偏移
    uint32_t entry_offset;   // OSR stub 在代码中的字节偏移
    uint16_t nslots;         // 活跃寄存器 slot 数量
    struct {
        int16_t bc_slot;     // 字节码寄存器 slot (-1=unmapped)
        uint8_t phys_reg;    // 目标物理寄存器
        uint8_t type;        // XIR 类型 (I64/F64)
    } slots[32];
} XirOsrEntry;
```

每个函数最多 `XIR_MAX_OSR_ENTRIES` (8) 个 OSR 入口点。

## 8. JIT Scratch 区域（XrJitScratch）

`coro->jit_ctx` 指向的共享通信区域，关键字段：

| 字段 | 用途 |
|------|------|
| `call_args[16]` | CALL_DIRECT 参数传递 |
| `call_proto` | 当前调用的 proto |
| `call_closure` | 当前调用的 closure |
| `deopt_id` | deopt 点 ID |
| `deopt_regs[29]` | deopt 时保存的 GP 寄存器 |
| `deopt_fp_regs[16]` | deopt 时保存的 FP 寄存器 |
| `deopt_spill_base` | deopt 时的 spill 区域基址 |
| `param_tags[8]` | 入参 tag（用于 nullable null-check） |
| `slot_runtime_tags[256]` | CALL_C 返回值的运行时 tag |
| `call_result_tag` | 最近一次 CALL_C 的返回 tag |
| `active_safepoint_id` | 当前 safepoint ID（GC 用） |
| `active_stack_map` | 当前函数的 stack map 表 |
| `jit_frame_sp` | JIT 帧指针（GC spill 扫描用） |
| `jit_frame_depth` | JIT 帧栈深度（嵌套调用） |
| `exception` | JIT 中抛出的异常指针 |
| `safepoint_page` | Guard page 安全点地址 |

## 9. JIT 帧布局（ARM64）

```
高地址
┌──────────────┐ ← 调用方 SP
│   saved FP   │ +0
│   saved LR   │ +8
│   x19 (ctx)  │ +16
│   x20        │ +24
│   x21        │ +32
│   ...        │
│   x27        │ +80
│   x28        │ +88
│   d8         │ +96
│   ...        │
│   d15        │ +152
│  spill slots │ +160...
│  smap_id (w) │ frame_size - 4
└──────────────┘ ← SP (16-byte aligned)
低地址
```

## 10. 关键不变量

1. **jit_frame_depth = 0** 进入 JIT 时必须为 0（在 `xir_jit_call` 入口重置）
2. **active_safepoint_id = UINT32_MAX** JIT 返回后必须置为无效值
3. **deopt_id 清零**：callee deopt 后必须清除，防止泄漏到 caller 的 CBNZ 检查
4. **PTR spill writeback**：CALL_C 前必须将 PTR 寄存器写回 spill slot（GC 依赖）
5. **param_tags 填充**：缺失参数（默认参数）的 tag 必须为 NULL（0）
6. **stack_map magic**：`XR_STACK_MAP_MAGIC` = 0x534D4150（"SMAP"），用于 FP 链遍历验证

## 11. 关键源文件

| 文件 | 职责 |
|------|------|
| `xir_jit.h` | JIT API 声明、哨兵值定义 |
| `xir_jit.c` | 入口/出口桥接、deopt 恢复、OSR 触发 |
| `xir_codegen.h` | Deopt table / Stack map / OSR entry 结构定义 |
| `xir_codegen.c` | ARM64 代码生成、epilogue、OSR stub |
| `xir_codegen_call.c` | CALL_C / CALL_DIRECT / CALL_SELF 代码生成 |
| `xir_offsets.h` | XrJitScratch / XrCoroutine 字段偏移宏 |
