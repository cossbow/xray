# VM 重构实施计划（`src/vm`）

> 日期：2026-04-25
> 状态：Draft
> 范围：`src/vm/*`
> 来源：合并自 `vm_audit_plan-opus.md`（架构导览）和 `vm_audit_plan-gpt.md`（问题清单）
> **开发原则：不考虑向后兼容性，直接采用最佳设计 — 不保留旧接口，不做兼容层**

---

## 1. 文档定位

这是一份把当前 `src/vm` 的架构事实和审计结论收敛成**单一可执行重构计划**的文档。

**核心立场**：

- Xray 没有外部用户、代码量小，每个阶段都直接选最佳设计
- 不保留任何 "临时方案" / "兼容包装"
- 凡涉及共享可变状态、入口契约、错误语义、文件职责 — 一次到位重写
- 性能优化必须先有契约和测试保护，但**契约本身要按目标形态写**

---

## 2. 当前架构实况（基线事实）

### 2.1 文件规模

| 文件 | 行数 | 职责 |
|------|------|------|
| `xvm.c` | 7784 | 主 `run()` 函数 — 全部 opcode dispatch |
| `xvm_cold_paths.c` | 3110 | `noinline` 冷路径函数 |
| `xvm_builtins.c` | 1575 | 内建类型方法分发 |
| `xvm_api.c` | 449 | 公共 C API |
| `xvm_internal.h` | 423 | 内部宏 + 上下文结构 |
| `xvm_profiler.c` | 419 | opcode 统计 |
| `xdebug.c` | 393 | 调试钩子 |
| `xvm_jumptab.h` | 318 | computed-goto 表 |
| `xvm_helpers.c` | 327 | 错误/栈追踪/init |
| `xvm_ops.c` | 482 | 数值运算 + 深比较 |
| `xic_method.c/h` | 466 | 方法 IC + Json Shape IC |
| `xic_field.c/h` + `xic_field_table.c/h` | 367 | 字段 IC |
| **合计** | **16630** | |

### 2.2 已经成熟的资产（保留 + 增强，不重写）

- **执行主循环路线**：computed goto + 局部宏 + 寄存器布局 — 性能导向设计已成型
- **Hot / Cold 分离**：`__attribute__((noinline))` 提取冷路径 + `VM_DISPATCH_COLD` 统一返回码
- **协程深集成**：BLOCKED / YIELD 的恢复语义、reductions 让步点、Channel 安全协议（pre-save frame + RESUME 标志）
- **IC 基础设施**：mono / poly 字段 IC、方法 IC、Json Shape IC（按 `pc - PROTO_CODE_BASE` 索引）
- **统一调用契约的雏形**：`OP_INVOKE` jump table、`vm_ctx` / `frame` / `pc` / `base` 入参四件套
- **Closure Pending 机制**：C 函数推闭包帧 + `handle_closure_pending` 恢复标签

### 2.3 指令分类总览（13 大类）

1. 基础加载 / 存储（move / loadi / loadk / upvalue / global / shared）
2. 算术 + 位运算（含运算符重载 + BigInt 自动提升）
3. 比较 + 控制流（含向后跳转 GC safepoint + reductions 让步）
4. 集合操作（Array / Map / String / Slice / StringBuilder）
5. **函数调用（最复杂）**：`OP_CALL` / `CALL_KEEP` / `CALL_STATIC` / `CALLSELF` / `TAILCALL` / `LOOP_BACK`
6. 返回：`RETURN` / `RETURN0` / `RETURN1`（含 defer LIFO + struct_ref 救援）
7. **OOP**：类创建 / `INVOKE` 统一分发 / `INVOKE_TAIL` / `INVOKE_DIRECT` / `INVOKE_BUILTIN` / `SUPERINVOKE`
8. 属性访问：`GETPROP` / `SETPROP`（含 IC + Json Shape IC + Getter/Setter）
9. 异常：`TRY` / `CATCH` / `FINALLY` / `END_TRY` / `THROW`
10. 协程：`GO` / `AWAIT*` / `YIELD` / `LOCK_THREAD` / `SET_LOCAL` / `SET_PRIORITY`
11. Channel：`CHAN_NEW*` / `CHAN_SEND/RECV` / `CHAN_TRY_*` / `CHAN_*_TIMEOUT` / `CHAN_CLOSE`
12. 结构化并发：`SCOPE_ENTER` / `SCOPE_EXIT`（linked / supervisor）
13. Defer / 时间 / Bytes / Regex / 断言 / Spill / 模块系统 / 类型化数组 / Struct 栈分配

详细分类见附录 A。

---

## 3. 核心问题诊断（按优先级）

### P0 — 正确性 / 所有权 / 入口契约

| ID | 主题 | 文件 |
|----|------|------|
| **VM-01** | 入口 API grow 不完整，release 越界风险 | `xvm_api.c` |
| **VM-02** | 字段/方法 IC 挂在共享 `proto` 上，多 worker 数据竞争 | `xvm.c` `xic_*` `jit/*` |
| **VM-03** | `xr_calloc` 未判空、`xr_realloc` 直接覆盖原指针 | `xvm.c` `xic_*` |
| **VM-04** | `vm_ctx` 权威分散，残留 `isolate->vm` 旧式 helper | `xvm_api.c` `xvm_helpers.c` `xvm_internal.h` |

### P1 — 语义一致性 / 架构边界

| ID | 主题 | 文件 |
|----|------|------|
| **VM-05** | 异常 stacktrace 实为 "throw 点记录"，非完整调用链 | `xvm_api.c` `xvm.c` |
| **VM-06** | builtin 错误语义混乱：`null` / `false` / `XR_NOTFOUND` / `fprintf` 并存 | `xvm_builtins.c` |
| **VM-07** | VM 直接吞掉 stdlib 类型方法语义 + `vm -> api` 反向 include | `xvm_builtins.c` `xvm_helpers.c` |
| **VM-08** | profiler 全局可变对象，多 worker 语义不清 | `xvm_profiler.c` |
| **VM-09** | opcode 元数据多处分散维护（名称/分类/profile 名/disasm） | `xvm_jumptab.h` `xvm_profiler.c` `xopcode_info.c` |
| **VM-10** | `xvm.c` 7784 行 + `xvm_cold_paths.c` 3110 行严重超限 | `xvm.c` `xvm_cold_paths.c` |

### P2 — 长期语义 / 性能

| ID | 主题 | 文件 |
|----|------|------|
| **VM-11** | 深度比较固定深度 + visiting 上限 → 假阴性 | `xvm_ops.c` |
| **VM-12** | `startfunc` 仍有动态分配 + line hook 逐指令检查 | `xvm.c` |

---

## 4. 目标架构（直接采用最佳设计）

### 4.1 `XrProto` = 100% 不可变

`XrProto` 只承载：

- 字节码
- 常量表
- 调试信息（行号、符号、源文件）
- 已发布的 JIT/AOT 入口指针 + 元数据（写一次，原子发布）

**禁止任何热路径可变状态挂在 `proto` 上**。当前的 `proto->ic_fields` / `proto->ic_methods` 必须迁移走。

### 4.2 IC 重新归位 = `vm_ctx`-local

- **四类 IC** 全部挂在 `XrVMContext`（per-coroutine，天然无竞争）：
  - 字段 IC（`OP_GETFIELD_IC` / `OP_SETFIELD_IC`）
  - 方法 IC（`OP_INVOKE`）
  - Json Shape IC
  - **invoke-IC**（`OP_INVOKE_BUILTIN` 上的 type+symbol → fn_ptr 缓存，Phase 3 引入）
- IC 表索引方式不变（`pc - PROTO_CODE_BASE`），但表本身不再共享
- JIT 通过显式 snapshot/publish API 读 feedback：`xr_vm_ic_snapshot(ctx, proto)` 返回只读副本，**禁止 JIT 直接读 ctx 内部**

### 4.3 `vm_ctx` 唯一权威

- 所有运行期状态字段统一进 `XrVMContext`：stack / frames / handlers / current_coro / stack_top / module_base_frame / defer_stack / ic_tables
- **公开 API 只有一个 getter**：`xr_vm_current_ctx()`
- **所有 VM 内部 helper 强制接收 `XrVMContext *ctx`**，禁止从 `isolate->vm` 偷取
- 删除 `xvm_internal.h` 中所有 `isolate->vm.*` 直连 helper

### 4.4 VM 入口统一 grow

- 所有 VM 入口（`xr_vm_call_closure` / `xr_vm_interpret_proto` / `xr_vm_execute_module`）经过同一函数：
  ```
  static int xr_vm_prepare_entry(XrVMContext *ctx, XrProto *proto, int extra_stack);
  ```
- 该函数负责：stack grow、frame grow、handler 容量、struct_areas 容量
- **删除所有 `XR_DCHECK` 容量保护**（DCheck 仅用于不变量，不用于运行时容量）

### 4.5 VM 只做 dispatch，不实现类型语义（per-type 方法表 + invoke-IC + AOT 共享）

**核心立场**：每个 builtin 类型在自己的 owning 模块里持有一张 `static const XrMethodSlot[]` 编译期方法表。VM 不再拥有任何方法实现；AOT 编译期直接读这些表生成 inline 代码。

#### 4.5.1 `XrMethodSlot` 协议

```c
// runtime/value/xmethod_table.h（新建）
typedef XrValue (*XrMethodFn)(XrayIsolate *iso, XrValue self, XrValue *args, int argc);

typedef struct XrMethodSlot {
    XrMethodFn fn;
    int8_t     min_args;
    int8_t     max_args;        // -1 = vararg
    uint8_t    flags;           // pure / may_yield / may_throw / no_gc
} XrMethodSlot;

// 全局类型 ID → 方法表指针，编译期常量数组
extern const XrMethodSlot *const xr_builtin_method_tables[XR_TYPEID_COUNT];
```

#### 4.5.2 每个类型自带方法表（owner module）

| Owner 模块 | 文件 | 类型 |
|---|---|---|
| `runtime/value/xint_methods.{c,h}` | int 的 abs/min/max/toString/... | 原始 |
| `runtime/value/xfloat_methods.{c,h}` | float 同上 | 原始 |
| `runtime/value/xbool_methods.{c,h}` | bool toString | 原始 |
| `runtime/object/xstring_methods.{c,h}` | indexOf/replace/split/... | runtime |
| `runtime/object/xarray_methods.{c,h}` | push/pop/sort/... | runtime |
| `runtime/object/xmap_methods.{c,h}` | get/set/has/... | runtime |
| `runtime/object/xset_methods.{c,h}` | add/has/union/... | runtime |
| `runtime/object/xjson_methods.{c,h}` | parse/stringify/... | runtime |
| `runtime/object/xbigint_methods.{c,h}` | 大整数运算 | runtime |
| **`stdlib/datetime/datetime_methods.{c,h}`** | 由 stdlib 自己持表 → 消除 vm→stdlib 反向 include | stdlib |
| **`stdlib/regex/regex_methods.{c,h}`** | 同上 | stdlib |

每个 `*_methods.h` 把**热小方法**（访问器、快路径）声明为 `static inline`，AOT 生成的代码 `#include` 后**直接展开**，零 call 开销。复杂方法（sort/replace/json parse 等）保持 `XR_FUNC` extern，方法表里取函数地址迫使编译器为 inline 方法生成 out-of-line 副本供 VM 间接调用。

#### 4.5.3 `OP_INVOKE_BUILTIN` 用 invoke-IC

```c
case OP_INVOKE_BUILTIN: {
    XrTypeId type = xr_value_typeid(receiver);
    int symbol    = ...;

    XrInvokeICSlot *ic = &ic_slot[pc - PROTO_CODE_BASE];
    if (XR_LIKELY(ic->type == type && ic->symbol == symbol)) {
        return ic->fn(iso, receiver, args, argc);   // 1 cmp + 1 indirect call
    }
    /* miss → 查表 + 填充 IC + 参数校验 */
}
```

invoke-IC 是新的第 4 类 IC（与 field / method / json shape IC 并列），同样**挂在 `XrVMContext` 上**（per-coroutine，无竞争），同样支持 `xr_vm_ic_snapshot()` 给 JIT 读 feedback。

#### 4.5.4 AOT 自动 tradeoff

`xcgen_call.c` 编译 `OP_INVOKE_BUILTIN` 时：

```c
const XrMethodSlot *table = xr_builtin_method_tables[known_type];
const XrMethodSlot *slot  = &table[known_symbol];
emit_direct_call_to(slot->fn, ...);
```

由于 `slot->fn` 是编译期常量地址：
- 小方法（在 header 里 `static inline`）→ AOT 生成的 C 文件 `#include` 后**完全展开**，零调用
- 大方法 → 直接 C `call`，和手写 C 无差别
- AOT-only 构建时 dead code elimination 把 `OP_INVOKE_BUILTIN` 整段拿掉，运行时不需要 IC、不需要表

**取代 `src/aot/xrt_method.h` / `xrt_value.h` 现有的重复 builtin 实现**，VM 与 AOT 永远共享同一份单一真相源。

#### 4.5.5 删除清单

- `src/vm/xvm_builtins.c`（1586 行）→ 全部删除
- `src/vm/xvm.c` 中对 `stdlib/regex/*.h`、`stdlib/datetime/*.h` 的反向 include → 全部删除
- `src/aot/xrt_method.h` 中重复的 builtin 实现 → 改为引用同一张表

### 4.6 异常 = 完整调用链

- `XrException.stacktrace` 在 throw 时**不**记录帧，而在 unwind 过程中**逐帧追加**
- 解释器 / 冷路径 / JIT 回退三条路径走同一个 `xr_vm_unwind_with_trace(ctx, exc)`
- 完整 contract 写入 `xvm.h` 注释

### 4.7 Builtin 错误语义二分法

只有两类返回：

- **数据语义 miss**：`Map.get(key)` / `String.indexOf(x)` 等可正常 miss 的 → 返回约定值（`null` / `-1` / `false`）
- **契约错误**：参数个数错、类型错、方法不存在 → **统一抛 catchable exception**

**禁止 `fprintf(stderr)` 作为错误模型**。

### 4.8 Profiler = per-isolate opt-in collector

- 删除 `g_vm_profiler` 全局
- `XrayIsolate` 持有 `XrVMProfiler *profiler`（默认 NULL）
- 通过 `xr_vm_profiler_enable(isolate, mode)` 显式启用
- 多 worker 下走 per-coro 累加 + 退出时合并到 isolate

### 4.9 Opcode 元数据 = X-macro 单一真相源

```
// xopcode_def.h - 唯一真相
#define XR_OPCODE_TABLE(_) \
    _(OP_MOVE,    "MOVE",    "load/store",    fmt_AB) \
    _(OP_LOADK,   "LOADK",   "load/store",    fmt_ABx) \
    ...
```

派生：`xopcode_info.c` / profiler 名称 / disasm / jumptab — **新增 opcode 只改一处主表**。

### 4.10 `xvm.c` 按职责拆分

| 拆分文件 | 职责 | 目标行数 |
|---------|------|---------|
| `xvm_exec.c` | 主循环骨架 + 基础指令（load / move / arith / cmp / control） | ≤ 1500 |
| `xvm_call.c` | `CALL` / `RETURN*` / frame management / closure pending | ≤ 1200 |
| `xvm_object.c` | `GETPROP` / `SETPROP` / `INVOKE*` / IC 接线 | ≤ 1500 |
| `xvm_exception.c` | `TRY` / `CATCH` / `FINALLY` / `THROW` / unwind / stacktrace | ≤ 600 |
| `xvm_coro.c` | `GO` / `AWAIT*` / `YIELD` / `SCOPE_*` / `SLEEP` 桥接 | ≤ 800 |
| `xvm_chan.c` | `CHAN_*` 指令（包括内联热路径） | ≤ 700 |
| `xvm_collection.c` | Array / Map / Json / String / Slice / StringBuilder 指令 | ≤ 1000 |
| `xvm_struct.c` | Struct 栈分配 + Bytes + 类型化数组指令 | ≤ 600 |
| `xvm_misc.c` | Defer / Time / Regex / Assert / Spill / Module / Type args | ≤ 500 |

`run()` 函数本身在 `xvm_exec.c`，通过宏 `#include "xvm_dispatch_inc.c"` 把每个分类文件 include 进 dispatch 块（保持 computed-goto 性能 + 物理拆分）。

`xvm_cold_paths.c` 同步按上述边界拆为 `xvm_call_cold.c` / `xvm_object_cold.c` / `xvm_coro_cold.c` / `xvm_chan_cold.c`。

### 4.11 Deep compare = 真实 visited set

- 删除固定深度 / 固定 visiting pair 上限
- 改为 hash set（`khash` 或 `xr_set`）记录 `(left_ptr, right_ptr)` 对
- 时间复杂度：O(n) 节点数；空间 O(n)，可接受

---

## 5. 实施计划（4 Phases，约 12 天）

### Phase 0：基础正确性（2 天）

**目标**：止住 release 下的真实风险。

| # | 任务 | 文件 |
|---|------|------|
| 0.1 | 实现 `xr_vm_prepare_entry()` 统一 grow | `xvm_api.c` `xvm_internal.h` |
| 0.2 | 三个入口（`call_closure` / `interpret_proto` / `execute_module`）走 prepare_entry | `xvm_api.c` |
| 0.3 | 清理所有 `xr_calloc/xr_realloc` 违规点 | `xvm.c` `xic_*` |
| 0.4 | 单一 `xr_vm_current_ctx()` getter；删除 `xvm_helpers.c` 中重复实现 | `xvm_api.c` `xvm_helpers.c` |
| 0.5 | 删除 `xvm_internal.h` 中所有 `isolate->vm.*` 直连 helper | `xvm_internal.h` |
| 0.6 | 删除 `xvm_helpers.c → api/xglobal_object.h` 反向 include | `xvm_helpers.c` |
| 0.7 | 新增 `tests/unit/vm/test_vm_api.c`：大栈帧 / 深调用 / vararg | `tests/unit/vm/` |

**验收**：

- 任意大 `maxstacksize` / 深 frame 入口不崩
- `grep -r "isolate->vm\." src/vm/` 返回 0 条业务路径
- ctest 全绿

### Phase 1：IC 与共享状态归位（3 天）

**目标**：把 `proto` 锁定为只读，IC 迁到 `vm_ctx`。

| # | 任务 | 文件 |
|---|------|------|
| 1.1 | `XrVMContext` 增加 `ic_field_table` / `ic_method_table` / `ic_json_shape_table` | `xvm_internal.h` |
| 1.2 | 字段 IC 读写从 `proto->ic_fields` 迁到 `ctx->ic_field_table[proto_id]` | `xvm.c` `xic_field.c` |
| 1.3 | 方法 IC 同样迁移 | `xvm.c` `xic_method.c` |
| 1.4 | Json Shape IC 同样迁移 | `xvm.c` |
| 1.5 | 删除 `XrProto.ic_fields` / `ic_methods` 字段 | `runtime/object/xproto.h` |
| 1.6 | 实现 `xr_vm_ic_snapshot(ctx, proto)` 供 JIT 读 | `xvm_api.c` `jit/*` |
| 1.7 | 修改 JIT builder 从 snapshot 读 feedback | `jit/xir_builder*.c` |
| 1.8 | 新增 `tests/unit/vm/test_ic_*`：状态迁移 / mono→poly→mega | `tests/unit/vm/` |

**验收**：

- `XrProto` 上无可变字段（`grep -i "ic_" src/runtime/object/xproto.h` 返回 0）
- 多 worker 同时执行同一 proto 的并发测试通过
- JIT/AOT 测试 100% 绿

### Phase 2：语义统一（3 天）

**目标**：异常 / builtin error / profiler 全部走目标 contract。

| # | 任务 | 文件 |
|---|------|------|
| 2.1 | 实现 `xr_vm_unwind_with_trace()`，throw 时不记帧、unwind 时逐帧追加 | `xvm_exception.c`（新建） |
| 2.2 | 三条 throw 路径（解释器 / 冷路径 / JIT 回退）统一走 unwind | `xvm.c` `xvm_cold_paths.c` `jit/*` |
| 2.3 | 删除 `xvm_builtins.c` 中所有 `fprintf(stderr)`；改抛 contract exception | `xvm_builtins.c` |
| 2.4 | 统一 builtin 错误语义：data miss vs contract error 二分 | `xvm_builtins.c` |
| 2.5 | 删除 `g_vm_profiler` 全局；改为 `isolate->profiler` opt-in | `xvm_profiler.c/h` |
| 2.6 | opcode X-macro 单一真相源 + 派生 info/profiler/disasm | `src/runtime/object/xopcode_def.h`（新建） |
| 2.7 | 新增 `tests/unit/vm/test_vm_exception.c`：stacktrace 完整性 | `tests/unit/vm/` |

**验收**：

- 解释器 / 冷路径 / JIT 抛出的同一异常 stacktrace 一致
- builtin 参数错误 100% 可 `try/catch`
- 新增 opcode 只改 X-macro 一处

### Phase 3：架构边界（per-type 方法表 + invoke-IC + AOT 共享，约 4 天）

**目标**：VM 不再持有任何 builtin 方法实现，全部下沉到 owning 模块的 `static const XrMethodSlot[]` 编译期方法表；AOT 与 VM 共享同一张表；`xvm_builtins.c` 与 vm→stdlib 反向 include 一并清零。

#### 3a. invoke-IC 基础设施（独立、可单独 benchmark）

**目标**：在不动方法分发实现的前提下，给 `OP_INVOKE_BUILTIN` 加一层 IC 缓存。这是后续所有迁移的承重点。

| # | 任务 | 文件 |
|---|------|------|
| 3a.1 | `XrInvokeICEntry { type_id, symbol, fn_ptr }` 定义 + 表协议 | `src/vm/xic_invoke.{c,h}`（新建） |
| 3a.2 | `XrVMContext` 增加 `XrInvokeICTable **ic_invoke_tables`（与 field/method IC 并列） | `src/vm/xvm_internal.h` `src/vm/xvm_ic.c` |
| 3a.3 | `OP_INVOKE_BUILTIN` 改造：先查 IC，命中直接 fn 调用；miss 走原 `*_method_call_by_symbol` 后回填 IC | `src/vm/xvm.c` |
| 3a.4 | snapshot API 扩展：`xr_vm_ic_snapshot()` 输出包含 invoke 表 | `src/vm/xvm_ic.c` `src/jit/xir_builder*.c` |
| 3a.5 | benchmark：`bench/vm_invoke_microbench.xr`（access/method 调用密集），证明 IC 命中 ≥ 旧分发性能 | `bench/` |
| 3a.6 | 单测：`tests/unit/vm/test_invoke_ic.c`（mono / poly / mega 迁移、跨 ctx 隔离） | `tests/unit/vm/` |

**验收**：
- `OP_INVOKE_BUILTIN` 的 IC 命中率在 builtin-heavy 测试上 ≥ 95%
- microbench 显示 IC 命中路径 ≥ 当前两层 switch 性能
- 所有现有 ctest + 回归测试通过

#### 3b. `XrMethodSlot` 协议 + per-type 方法表骨架

**目标**：定义统一的方法表协议，但暂不迁实现；让 `OP_INVOKE_BUILTIN` 优先走方法表，未注册的类型回落到 `xvm_builtins.c`。

| # | 任务 | 文件 |
|---|------|------|
| 3b.1 | 定义 `XrMethodSlot` / `XrMethodFn` / `xr_builtin_method_tables[]` 协议 | `src/runtime/value/xmethod_table.{c,h}`（新建） |
| 3b.2 | 在 owner 模块下建立空骨架（仅头文件 + 空表）：`xint_methods` / `xfloat_methods` / `xbool_methods` 等 11 张 | 见 4.5.2 表 |
| 3b.3 | `OP_INVOKE_BUILTIN` miss 路径：先查 `xr_builtin_method_tables[type]`，找不到再回落旧 dispatch | `src/vm/xvm.c` |
| 3b.4 | 单测：`tests/unit/vm/test_method_table.c`（注册 / 查找 / 参数校验 / fallback） | `tests/unit/vm/` |

**验收**：所有方法表当前都是空，`OP_INVOKE_BUILTIN` 走 fallback；ctest + 回归全绿（无功能变化）。

#### 3c. 逐 owner 模块迁移（每类型一个 commit）

**目标**：把 `xvm_builtins.c` 中各类型的 `*_method_call_by_symbol` 切片迁到对应 `*_methods.c`，每完成一个类型就删 builtins.c 里那一段。

迁移顺序（简单 → 复杂）：

| # | 类型 | 预计行数 | inline 候选（小热路径） |
|---|------|---------|-----------------------|
| 3c.1 | bool / int / float / bigint | ~250 行 | toString / abs / sign / isFinite |
| 3c.2 | string | ~430 行 | length / charAt / startsWith / endsWith |
| 3c.3 | array | ~280 行 | length / first / last / isEmpty / push（容量足时） |
| 3c.4 | map / set | ~150 行 | size / isEmpty / has |
| 3c.5 | json | ~100 行 | （多为 cold） |
| 3c.6 | datetime（搬到 stdlib/datetime） | ~100 行 | （cold，但消除反向 include） |
| 3c.7 | regex（搬到 stdlib/regex） | ~150 行 | （cold，消除反向 include） |

**每步必做**：
- 把热小方法在 `*_methods.h` 里声明 `static inline`
- 复杂方法在 `*_methods.c` 里 `XR_FUNC` 实现
- 方法表 `xr_<type>_method_table[]` 取 `&fn` 强制 inline 生成 out-of-line 副本
- 单元测试：每个类型至少 2 条 case 覆盖正常路径 + 契约错误路径
- 回归测试：每步必须 283/283 全绿

#### 3d. AOT 接入 + 删除重复

**目标**：让 AOT 共享同一张方法表，删除 `xrt_method.h` 中重复的 builtin 实现。

| # | 任务 | 文件 |
|---|------|------|
| 3d.1 | `xcgen_call.c` 编译 `OP_INVOKE_BUILTIN`：编译期已知 type+symbol → 直接 emit `slot->fn` 调用 | `src/aot/xcgen_call.c` |
| 3d.2 | AOT 生成的 C 文件 `#include` 各 `*_methods.h`，热小方法自动展开 | `src/aot/xcgen.c` |
| 3d.3 | 删除 `src/aot/xrt_method.h` 中与方法表重复的 bridge 函数 | `src/aot/` |
| 3d.4 | 验证：现有 AOT 回归测试（`tests/aot/`）VM-AOT 输出零 diff | `tests/aot/` |

**验收**：AOT 编译生成的 C 文件大小不增反减；70/70 ctest + 280/280 regression + 24/24 AOT 全绿。

#### 3e. 收口

| # | 任务 | 文件 |
|---|------|------|
| 3e.1 | 删除 `src/vm/xvm_builtins.c` / `xvm_builtins.h` | — |
| 3e.2 | 删除 `xvm.c` / `xvm_builtins.c` 对 `stdlib/regex/*.h` / `stdlib/datetime/*.h` 的所有 include | `src/vm/` |
| 3e.3 | grep 验证：`grep -rE 'include.*\\.\\./(api\|stdlib)' src/vm/` 返回 0 | — |
| 3e.4 | grep 验证：`xvm_builtins` 在仓库中无引用 | — |

**Phase 3 总验收**：

- `src/vm/xvm_builtins.{c,h}` 不存在
- `grep -E 'include.*\\.\\./(api\|stdlib)' src/vm/` 返回 0 条
- 所有 ctest / 回归 / AOT / JIT 测试 100% 通过
- builtin-heavy microbench 性能 ≥ Phase 3 之前
- AOT 生成代码体积不增加（共享方法表收益）

### Phase 4：文件拆分（2 天）

**目标**：按 4.10 的拆分方案落地，每文件 ≤ 1500 行。

| # | 任务 | 文件 |
|---|------|------|
| 4.1 | 拆 `xvm.c` 为 9 个文件（见 4.10） | `src/vm/*` |
| 4.2 | 拆 `xvm_cold_paths.c` 为 4 个 cold 文件 | `src/vm/*_cold.c` |
| 4.3 | 主循环用 `#include "xvm_dispatch_*.inc.c"` 在 `xvm_exec.c` 中合成 | `xvm_exec.c` |
| 4.4 | 验证 computed-goto 性能无回退 | benchmark |

**验收**：

- `wc -l src/vm/*.c` 全部 ≤ 1500
- 性能 benchmark 与拆分前差 < 1%
- 所有测试通过

---

## 6. 测试计划

### 6.1 新增 `tests/unit/vm/`

| 测试文件 | 覆盖点 |
|---------|--------|
| `test_vm_api.c` | 大栈帧 / 深 frame / vararg / rest / 入口 grow |
| `test_vm_exception.c` | throw / catch / finally / stacktrace 完整性 / contract error catch |
| `test_ic_field.c` | mono → poly → mega 状态迁移 / 多 ctx 隔离 |
| `test_ic_method.c` | 同上 |
| `test_vm_builtins.c` | data miss vs contract error 边界 |
| `test_vm_concurrent.c` | 多 worker 同 proto 执行 / IC 无竞争 |
| `test_vm_debug.c` | line hook 粒度 / disassembler 输出 |

### 6.2 `.xr` 回归补充场景

- 深调用链异常 + stacktrace 完整性
- 大栈帧函数（>4096 寄存器）
- 多 worker 共享类的方法调用
- `Map/String/Json` 的 method-not-found 与合法 miss
- 环状结构 deep_eq 不再假阴性

### 6.3 性能 benchmark

- Phase 4 后必须跑 `bench/vm_microbench.xr`，与拆分前差异 < 1%
- 关键 opcode（CALL / INVOKE / GETPROP / GETFIELD_IC）单独基准

---

## 7. 工程纪律

### 7.1 一次到位，不留兼容层

每个 Phase 内的迁移是**完全替换**，禁止保留 "旧路径作为 fallback"。如果一次写不完，就缩小本 Phase 范围，**不允许引入半成品兼容代码**。

### 7.2 每个 VM bug 双落地

每修一个 VM bug，至少补：

- 一个 `.xr` 语言级 case
- 一个 `tests/unit/vm/` 最小复现

### 7.3 `XrProto` 不可变，违者 review 拒收

新代码不允许往 `XrProto` 上加任何 `mutable` / `atomic` / 可写字段。

### 7.4 VM 内部 helper 必须接收 `ctx`

签名禁止 `xr_vm_xxx(XrayIsolate *)` 形式（除了 `xr_vm_current_ctx()` 本身）。

### 7.5 Builtin 不再回流 VM

新增对象方法 → `runtime/class/` 的 method registry，永远不在 `src/vm/` 下加 builtin handler。

### 7.6 Opcode 元数据单点修改

新增 opcode → 改 `xopcode_def.h` X-macro 一处。手工同步 profiler/disasm 字符串的 PR 一律拒收。

---

## 8. 非目标

本轮**不做**：

- 重写 dispatch 模型（保留 computed goto）
- 新增 opcode（除非 Phase 内必需）
- 把 VM/JIT/AOT 统一成大执行框架
- 在测试覆盖前追逐微基准

---

## 9. 关键设计决策摘要

| 决策 | 选择 | 拒绝的替代方案 |
|------|------|---------------|
| IC 归属 | `vm_ctx`-local（含新增 invoke-IC） | per-thread / per-proto+lock |
| Builtin 实现位置 | 各 owner 模块 `*_methods.{c,h}` 持 `static const XrMethodSlot[]` | `runtime/class/` 中央 registry / `xvm_builtins.c` 内 / 各 opcode 内联 |
| Builtin 分发机制 | 编译期方法表 + invoke-IC | 运行期 `register()` / 双层 switch / 单点桥接函数 |
| Builtin 物理布局 | 热小方法 header `static inline` + 表内取地址生成 out-of-line 副本 | 全 extern / 全 inline / VM 与 AOT 各持一份 |
| AOT/VM 共享 | 同一张方法表，AOT 编译期常量折叠 + DCE 去除 IC | AOT 持独立 `xrt_method.h` 拷贝 |
| Opcode 元数据 | X-macro 单表 | 多文件手工同步 / 代码生成器 |
| 异常 stacktrace | unwind 时逐帧追加 | throw 时一次性收集 |
| Profiler | per-isolate opt-in | 全局自动启用 |
| 文件拆分 | 9 个 dispatch 文件 + `#include` 合成 | 函数指针表 / 真分文件牺牲性能 |
| Deep compare | 真实 visited set | 固定深度上限 |
| 入口 grow | 强制走 `xr_vm_prepare_entry` | 各入口各自 DCheck |

---

## 附录 A：完整指令分类（13 大类）

参见 `vm_audit_plan-opus.md` § 3。本文档仅在 § 4.10 引用拆分边界。

## 附录 B：Hot/Cold 返回码

```
VM_COLD_BREAK     -> vmbreak
VM_COLD_STARTFUNC -> goto startfunc
VM_COLD_CONTINUE  -> 继续 fall-through
VM_COLD_BLOCKED   -> return XR_VM_BLOCKED
VM_COLD_YIELD     -> return XR_VM_YIELD
VM_COLD_ERROR     -> handler != 0 ? startfunc : return RUNTIME_ERROR
VM_COLD_FATAL     -> return XR_VM_RUNTIME_ERROR
```

Phase 0-4 后，这套返回码协议保持不变。

---

## 10. 结语

`src/vm` 当前的问题不是"已经不可维护"，而是**已经走到必须先把核心契约和职责边界一次性收住**的节点。

这份计划的关键判断：

- **不允许临时方案**：要么彻底迁移，要么本期不做
- **`XrProto` 必须不可变**：一切共享可变状态归 `vm_ctx`
- **VM 只做 dispatch**：类型语义全部下沉到 `runtime/class/`
- **Opcode 元数据单一真相源**：X-macro 派生一切
- **入口 / 异常 / 错误 / profiler** 都要有明确 contract

按 4 Phase 推进，约 12 天可完成全部目标。完成后，`src/vm` 将成为**只负责字节码执行 + 协程调度桥接**的纯执行器，为后续 JIT/AOT 协同和性能优化打下干净基础。
