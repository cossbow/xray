# XVM 核心执行循环架构分析（Opus）

> 日期：2026-04-25
> 范围：`src/vm/xvm.c` / `src/vm/xvm_cold_paths.c` / `src/vm/xvm_builtins.c` 等
> 来源：源码顺序阅读 + 指令分类整理
> 性质：纯架构分析与代码导览（不含改造计划）

---

## 1. 文件总览

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/vm/xvm.c` | 7784 | 主 `run()` 函数 — 完整指令分发循环 |
| `src/vm/xvm_cold_paths.c` | 3110 | `__attribute__((noinline))` 冷路径函数（减少 I-cache 压力） |
| `src/vm/xvm_builtins.c` | 1575 | 内建类型方法分发（Map/Json/String/Array/Set/Int/Float/Bool） |
| `src/vm/xvm_api.c` | 449 | 公共 C API：调用闭包、执行 proto/module、异常传播 |
| `src/vm/xvm_helpers.c` | 327 | 运行时错误、栈追踪、VM 初始化/清理 |
| `src/vm/xvm_ops.c` | 482 | 数值运算辅助、深度比较（带循环检测）、字符串拼接 |
| `src/vm/xvm_profiler.c` | 419 | 性能剖析器：opcode 统计与报告 |
| `src/vm/xvm_internal.h` | 423 | VM 内部宏、上下文/帧结构 |
| `src/vm/xvm_jumptab.h` | 318 | computed-goto 跳转表 |
| `src/vm/xvm_cold_paths.h` | 131 | 冷路径函数声明、`VM_COLD_*` 返回码 |
| `src/vm/xvm_checks.h` | 64 | `VM_RUNTIME_ERROR` / `VM_BARRIER_VAL` 等热路径宏 |
| `src/vm/xvm_profiler.h` | 158 | profiler 接口 |
| `src/vm/xvm.h` | 130 | VM 公共头 |
| `src/vm/xdebug.c` / `.h` | 393 / 34 | 调试钩子（断点/单步/异常断点） |
| `src/vm/xic_field.c` / `.h` | 131 / 142 | 字段访问 Inline Cache（mono/poly） |
| `src/vm/xic_field_table.c` / `.h` | 58 / 36 | per-proto 字段 IC 表 |
| `src/vm/xic_method.c` / `.h` | 201 / 265 | 方法分发 Inline Cache + Json Shape IC |
| **合计** | **16630** | |

---

## 2. `run()` 主循环结构（`xvm.c`）

### 2.1 入口与上下文

- 主函数签名（节选）：每次进入 `run()` 时从 `vm_ctx` / `ci` / `frame` / `pc` / `base` 中取出执行状态。
- 关键宏：
  - `R(x)` / `K(x)` / `RA/RB/RC/KB/KC` — 寄存器/常量访问
  - `savepc()` — 把 `pc` 同步回 `frame->pc`
  - `vmcase(OP)` / `vmbreak` — 由 `XR_USE_COMPUTED_GOTO` 切换 dispatch 模式
  - `VM_DISPATCH_COLD(rc)` — 统一处理冷路径返回码（BREAK/STARTFUNC/BLOCKED/YIELD/ERROR/FATAL）
- 顶部标签：
  - `startfunc:` — 进入新 frame 后的初始化（pc/base/cl 重新加载）
  - `handle_closure_pending:` — C 函数通过 `xr_yield_call_closure` 推入闭包帧后，从 `OP_RETURN*` 跳回的恢复点

### 2.2 dispatch 模式

- 默认采用 **computed-goto**：通过 `xvm_jumptab.h` 编织 `&&op_label[]` 表，避免 `switch` 的预测开销
- 回退到 `switch` 模式时由 `vmcase`/`default` 处理，不能识别的 opcode → `VM_RUNTIME_ERROR`

---

## 3. 指令分类详解

### 3.1 基础加载/存储（约 L200–L700）

- 寄存器/字面量：`OP_MOVE` / `OP_LOADI` / `OP_LOADF` / `OP_LOADK` / `OP_LOADNULL` / `OP_LOADTRUE` / `OP_LOADFALSE`
- Upvalue：`OP_GETUPVALUE` / `OP_SETUPVALUE` / `OP_CLOSE_UPVALS`
- 全局/共享：`OP_GETGLOBAL` / `OP_SETGLOBAL` / `OP_GETSHARED` / `OP_SETSHARED`

### 3.2 算术 & 位运算（约 L700–L1300）

- 快路径：int/float 直接运算
- BigInt 提升：溢出自动升级到 `XrBigInt`
- 运算符重载：`VM_TRY_BINARY_OP_OVERLOAD` 宏在快路径失败时查找类方法
- `OP_ADD/SUB/MUL/DIV/MOD` + 立即数变体（`ADDI/SUBI/MULI`）+ 常量变体（`ADDK/SUBK/MULK/DIVK/MODK`）
- 位运算：`OP_BAND/BOR/BXOR/BNOT/SHL/SHR`
- box/unbox：`OP_BOX_I64/F64` / `OP_UNBOX_I64/F64`

### 3.3 比较 & 控制流（约 L1300–L2800）

- `OP_EQ/NE/LT/LE/GT/GE` + 立即数 / 常量变体
- `OP_JMP` — 向后跳转时执行 GC safepoint + reductions 检查（协程让步点）
- `OP_FORPREP/FORLOOP/FORIN` — 数值/迭代器 for 循环
- `OP_TEST/TESTSET` — 条件跳转（截短求值）

### 3.4 集合操作（约 L2800–L3360）

- Array：`OP_ARRAY_GET/GETC/SET/SETC/PUSH/LEN/INIT` + `OP_ARRAY_GET_NOCHECK`
- Map：`OP_MAP_GET/GETK/SET/SETK/INCREMENT`
- 泛型索引：`OP_INDEX_GET/SET` 支持 Array / Map / String / Range / Json / FixedArray + 运算符重载
- 切片 / 字符串：`OP_SLICE` / `OP_SUBSTRING` / `OP_STR_REPEAT`
- StringBuilder：`OP_STRBUF_NEW/APPEND/FINISH`

### 3.5 函数调用（约 L3360–L4560，VM 中最复杂的部分）

`OP_CALL` 的处理流程：

1. 快速分发：`XR_IS_FUNCTION` → 闭包路径；`XR_IS_CFUNCTION` → C 函数路径
2. 枚举转换：`Status(200)` → `Status.Success`
3. 类构造器：查找 `call` 或 `constructor` 方法，支持 shared 分配
4. 绑定方法 `XrBoundMethod`
5. C 函数：区分 yieldable 与 slow C 函数
6. **JIT 快速路径**：热函数检测 → `xir_jit_try_compile` → `xir_jit_call`
7. **AOT 快速路径**：`proto->jit_entry` 直接调用编译好的原生代码
8. **Deopt 恢复**：JIT 失败时尝试 mid-function recovery；超阈值时失效 `jit_entry`
9. **Type feedback**：收集参数类型供 PGO 使用
10. Vararg：将多余实参收集到 rest array

其他调用类指令：

- `OP_CALL_KEEP` — 保留函数寄存器（map/filter/reduce 内联用）
- `OP_CALL_STATIC` — 编译期已知为闭包，直接进入 `OP_CALL` 闭包路径
- `OP_CALLSELF` — 递归自调用（带 JIT 快速路径）
- `OP_TAILCALL` — 真尾调用：复用栈帧，无深度增长
- `OP_LOOP_BACK` — 尾递归→循环转换：单指令替代 `CLOSE+MOVE+JMP`

### 3.6 返回（约 L4185–L4555）

- `OP_RETURN` — 完整返回：多返回值、defer LIFO 执行、constructor 栈管理、toString print、模块边界检查、operator overload 条件跳转
- `OP_RETURN0` — 快速零返回值
- `OP_RETURN1` — 快速单返回值（含 struct_ref 生存期救援 `struct_ret_arena`）

> 三条 RETURN 指令共享同一套 frame pop / defer / handler 收尾逻辑，但按返回值数量裁剪以提升常见路径性能。

### 3.7 OOP（约 L4560–L5510）

- 类创建：`OP_CLASS_CREATE_FROM_DESCRIPTOR`（支持运行时父类解析）
- 静态构造器：`OP_CLINIT_CALL`
- 抽象保护：`OP_ABSTRACT_ERROR`
- 字段访问：
  - `OP_GETFIELD` / `OP_SETFIELD` — 按索引直接访问
  - `OP_GETFIELD_IC` — 带 IC（mono / poly）
- **`OP_INVOKE` —— 统一方法调用分发（O(1) jump table）**
  - 类型分发顺序：Task → Coro → Channel(热 send/recv) → isEmpty 内联 → struct_ref → 类型 switch（Instance / String / Array / Map / Set / Json / Module / Enum / Iterator / BigInt / StringBuilder / Slice / Range / NativeType）
  - Channel 热路径：`send` / `recv` 内联在主循环
  - Channel 冷方法（trySend / tryRecv / sendTimeout / recvTimeout / close / isClosed）→ `vm_invoke_channel`
  - Instance 方法：使用 `XrICMethod` 多态 IC 加速
- `OP_INVOKE_TAIL` — 方法尾调用
- `OP_INVOKE_DIRECT` — 编译期确定的方法索引直接调用
- `OP_INVOKE_BUILTIN` — 编译期确定的内建类型方法直接分发（含 `Map/Array/String/Set/Json/Int/Float/Bool/BigInt/StringBuilder/Slice` 全分支）
- `OP_SUPERINVOKE` — 超类方法调用（构造器 `:super()` 与 `super.method()`）

### 3.8 属性访问（约 L5700–L6040）

- `OP_GETPROP`：
  - FixedArray `.length` 快速路径
  - Struct ref 字段快速路径（按 `XrStructFieldLayout` 解码 native 类型）
  - Instance 快速路径 + mono / poly IC
  - Json 快速路径 + Shape IC（in-object 字段命中）
  - Getter 方法查找
- `OP_SETPROP`：同上对称（含 Json Shape IC + Setter 查找 + 字段 IC + 写屏障 `XR_GC_BARRIER_BACK_SAFE`）

### 3.9 异常处理（约 L6043–L6200）

- `OP_TRY` — 懒分配 handler 栈（`vm_ctx->handlers`），记录 `catch_offset` / `finally_offset` / `stack_size` / `frame_count`
- `OP_CATCH` — 提取异常值（自动解包 `XrException.userData`）
- `OP_FINALLY` — 标记 `in_finally`，使 re-throw 向外传播
- `OP_END_TRY` — 检查 pending exception 并 re-throw
- `OP_THROW` — 自动包装非异常值 / 添加栈追踪 / 调用 debug `on_exception` hook / 栈展开

### 3.10 协程（约 L6446–L6675）

- 创建：`OP_GO` / `OP_GO_INVOKE` / `OP_SPAWN_CONT`
- 等待：`OP_AWAIT`（内联快路径：已完成 + 立即值，不走深拷贝） / `OP_AWAIT_TIMEOUT` / `OP_AWAIT_ALL` / `OP_AWAIT_ANY`
- `OP_YIELD` — 协作让步：A=0 立即 yield；A>0 hint yield（递减 reductions，避免上下文切换风暴）
- `OP_CANCELLED`
- 线程绑定：`OP_LOCK_THREAD` / `OP_UNLOCK_THREAD`（嵌套计数）
- 协程局部：`OP_SET_LOCAL` / `OP_GET_LOCAL`（main coro 走 `isolate->vm.main_locals`）
- 优先级：`OP_SET_PRIORITY` — 重新入队以应用新优先级
- `OP_CORO_CTRL` — 协程监控/诊断（冷路径）

### 3.11 Channel（约 L6680–L6985）

- 创建：`OP_CHAN_NEW` / `OP_CHAN_NEW_NAMED`（cluster 注册）
- 阻塞 send / recv：`OP_CHAN_SEND` / `OP_CHAN_RECV`
  - 进入前 pre-save：`frame->pc = pc - 1; call_status |= XR_CALL_YIELDED`
  - `XR_CHAN_BLOCK` → `return XR_VM_BLOCKED`
  - `XR_RESUME_CHANNEL` 标志：恢复时检测到则跳过实际 channel 调用
- 非阻塞：`OP_CHAN_TRY_SEND` / `OP_CHAN_TRY_RECV`（含 unbuffered rendezvous wake）
- 超时：`OP_CHAN_SEND_TIMEOUT` / `OP_CHAN_RECV_TIMEOUT`（冷路径）
- `OP_CHAN_CLOSE` / `OP_CHAN_IS_CLOSED`
- Select：`OP_SELECT_BLOCK`（冷路径）；`OP_SELECT_START/CASE/END` 当前抛“not yet implemented”

### 3.12 结构化并发（约 L7133–L7240）

- `OP_SCOPE_ENTER` — 创建 `XrScopeContext`，挂到 `current->current_scope`，模式：`linked` / `supervisor`
- `OP_SCOPE_EXIT`：
  - 子任务未结束：`return XR_VM_BLOCKED`，重新执行
  - linked：`first_error` 不空则抛异常
  - supervisor：把 `errors[]` 写入 `result_reg`
- main thread fallback：自旋 `count > 0`，必要时 `sched_yield()`

### 3.13 Defer / 时间 / 杂项

- `OP_DEFER` — 把 closure + nargs + args 推入 `isolate->vm.defer_stack`（懒分配 + 倍增扩容）；由 `OP_RETURN*` 在 frame pop 时 LIFO 执行
- `OP_TIME_AFTER` — 创建 timer channel，注册到当前 worker 的 timer wheel
- `OP_SLEEP` — 协程模式：注册到 timer wheel + `XR_VM_BLOCKED`；非协程模式：`usleep`
- `OP_BYTES_NEW` — 创建 `Array<uint8>`，支持 shared 分配
- `OP_REGEX_COMPILE` — 编译失败返回 `null`（不抛异常）
- 测试断言：`OP_ASSERT` / `OP_ASSERT_EQ` / `OP_ASSERT_NE`（走 `xr_value_deep_eq`）
- Spill：`OP_SPILL` / `OP_RELOAD`（基于 `MAXREGS + slot` 偏移）
- 模块系统：`OP_IMPORT`（注意 import 可能引发 stack realloc，必须刷新 `base`）/ `OP_EXPORT` / `OP_EXPORT_ALL`
- 类型化数组：`OP_TARRAY_GET/GETC/SET/PUSH`（按 `arr->elem_type` 直接读写 native 类型，零 box/unbox）
- Json 直访：`OP_TFIELD_GET` / `OP_TFIELD_SET`
- 实例化类型参数：`OP_INST_TYPE_ARGS`
- Struct 原生存储：`OP_NEW_STRUCT` / `OP_STRUCT_GET` / `OP_STRUCT_SET` / `OP_STRUCT_COPY`
  - 在 `vm_ctx->struct_areas[VM_FRAME_COUNT - 1]` 中按 16 字节槽分配
  - 头 8 字节存 `XrClass*`，后续按 `XrStructLayout` 排布字段
  - 支持 `XR_NATIVE_I8/I16/I32/I64/U8/U16/U32/U64/F32/F64/BOOL/STRING/STRUCT/ARRAY`
- `OP_NOP`

---

## 4. 关键设计模式

### 4.1 Hot / Cold 分离

- 高频路径直接内联在 `run()` 中（如 `CALL` 闭包路径、`AWAIT` 立即值快路径、Channel send/recv 热路径）
- 低频路径通过 `__attribute__((noinline))` 提取到 `xvm_cold_paths.c`：
  - `vm_invoke_channel` / `vm_invoke_task_handle` / `vm_invoke_coro_handle` / `vm_invoke_enum`
  - `vm_invoke_class` / `vm_invoke_module`
  - `vm_superinvoke`
  - `vm_setprop_type_dispatch` / `vm_getprop_type_dispatch` / `vm_getprop_instance_getter` / `vm_setprop_instance_setter`
  - `vm_go` / `vm_go_invoke` / `vm_spawn_cont` / `vm_await*` / `vm_chan_send_timeout` / `vm_chan_recv_timeout` / `vm_select_block` / `vm_coro_ctrl`
- 冷路径返回码统一通过 `VM_DISPATCH_COLD(rc)` 处理：`BREAK / STARTFUNC / BLOCKED / YIELD / ERROR / FATAL / CONTINUE`

### 4.2 Inline Cache

- **字段 IC**（`xic_field.[ch]`）：mono → poly → mega 退化；按 `pc - PROTO_CODE_BASE` 索引到 per-proto IC 表（`proto->ic_fields`）
- **方法 IC**（`xic_method.[ch]`）：Instance 方法分发；`XrICMethod` 按 `(class, symbol)` 命中
- **Json Shape IC**：按 `(shape_id, symbol)` 命中 → 直接 `json->fields[idx]`，仅 in-object 字段；overflow 字段走 slow path

### 4.3 GC Safepoint 策略

GC safepoint（`checkGC` / 协程切换点）出现在：

- `OP_CALL` 入口（函数调用边界）
- `OP_RETURN*`（函数退出）
- `OP_JMP` 向后跳转（循环回边）
- `OP_LOOP_BACK`（尾递归循环）
- `OP_BYTES_NEW` 等显式分配路径（带 storage 模式判断）

### 4.4 协程调度集成

- `reductions` 计数器：每次循环回边递减；耗尽时强制 `return XR_VM_YIELD`，由 worker 调度器切换协程
- `OP_YIELD` hint：A>0 时只递减 reductions，避免高频上下文切换
- 阻塞操作（channel / await / sleep / scope / select）统一返回 `XR_VM_BLOCKED`，scheduler 负责唤醒后再次进入 `run()`，从 `frame->pc` 恢复

### 4.5 Channel 安全协议

所有阻塞 send / recv 在调用 `xr_channel_*` 之前：

1. 把待发送/接收所需的状态写入协程字段：`current->send_value` / `current->recv_slot`
2. `savepc()` 并把 `frame->pc` 重置为当前指令本身（不是下一条），保证 BLOCKED 恢复时重新执行该指令
3. `frame->call_status |= XR_CALL_YIELDED`，告知 scheduler 这是显式让步而非异常
4. 恢复时通过 `xr_coro_resume_load(current)` 区分 `XR_RESUME_CHANNEL` / `XR_RESUME_CHANNEL_CLOSED` / `XR_RESUME_OK`

### 4.6 Closure Pending 机制

- C 函数通过 `xr_yield_call_closure` 推入闭包帧并设置 `XR_CALL_CLOSURE_PENDING`
- 子帧 `OP_RETURN*` 检测到调用方有 pending → 跳到 `handle_closure_pending` 标签
- 调用 continuation 函数 `_cont(isolate, XR_RESUME_CLOSURE_DONE, ...)`
- 根据返回的 `XrCFuncResult`（DONE / BLOCKED / YIELD / CALL_CLOSURE / ERROR）决定后续控制流

### 4.7 Defer LIFO 执行

- `OP_DEFER` 推入 `[closure, nargs, args...]` 三元组到全局 defer stack
- frame entry 时记录 `defer_frame_marks[frame_idx] = defer_count`
- `OP_RETURN*` 在 frame pop 时把当前 frame 之上的 defer 项逆序执行

### 4.8 Struct Native Storage（值类型）

- 栈分配，零堆压力
- 每帧在 `vm_ctx->struct_areas[frame_idx]` 中以 16 字节为粒度分槽
- `xr_struct_ref(ptr, sub_layout_id)` 把栈指针包装成 `XR_TAG_STRUCT_REF` 的 `XrValue`，可在寄存器间传递
- `OP_RETURN1` 对返回的 struct_ref 执行 "struct return arena" 救援（防止返回后栈帧被复用导致 use-after-free）

---

## 5. `xvm_cold_paths.c` 结构（已读 0–700 行 + 抽样）

按职责分组的 noinline 函数：

- **OP_INVOKE 各类型**：`vm_invoke_channel` / `vm_invoke_task_handle` / `vm_invoke_coro_handle` / `vm_invoke_enum` / `vm_invoke_class` / `vm_invoke_module`
- **OP_SUPERINVOKE**：`vm_superinvoke`
- **OP_SETPROP / OP_GETPROP 分支**：`vm_setprop_type_dispatch` / `vm_getprop_type_dispatch` / `vm_getprop_instance_getter` / `vm_setprop_instance_setter`
- **协程指令**：`vm_go` / `vm_go_invoke` / `vm_spawn_cont` / `vm_await` / `vm_await_timeout` / `vm_await_all` / `vm_await_any` / `vm_coro_ctrl`
- **Channel 超时**：`vm_chan_send_timeout` / `vm_chan_recv_timeout`
- **Select**：`vm_select_block`
- **辅助**：`vm_chan_copy_send` / `vm_chan_copy_recv`（深拷贝可变值，保证 channel 缓冲安全）

返回值约定（`xvm_cold_paths.h`）：

- `VM_COLD_BREAK` — 完成，主循环 `vmbreak`
- `VM_COLD_STARTFUNC` — 已设置新 frame，主循环 `goto startfunc`
- `VM_COLD_CONTINUE` — 未匹配，主循环继续 fall-through
- `VM_COLD_BLOCKED` / `VM_COLD_YIELD` — 立即 `return XR_VM_BLOCKED/YIELD`
- `VM_COLD_ERROR` — 异常已抛，主循环根据 `VM_HANDLER_COUNT` 决定 startfunc 还是 return
- `VM_COLD_FATAL` — 立即 `return XR_VM_RUNTIME_ERROR`

---

## 6. `xvm_builtins.c` 结构（已读 0–200 行 + 抽样）

- 用 jump table（按 symbol → handler index）优化方法分发
- 每种类型一组 handler + 一个 `*_method_call_by_symbol(...)` 入口
- 已观测到的类型：
  - **Map**：`has / get / set / delete / clear / keys / values / entries / isEmpty / hasValue` + WeakMap / iterator 特殊处理
  - **Json**：`json_method_call_by_symbol` 类似 Map
  - **String**：`string_charat` 等
  - **Array / Set / Int / Float / Bool / BigInt**：在 `OP_INVOKE` / `OP_INVOKE_BUILTIN` 中以 `*_method_call_by_symbol` 形式调用
- 未命中返回 `XR_NOTFOUND`，由调用方（`OP_INVOKE` / `OP_INVOKE_BUILTIN`）转换为 `XR_ERR_TYPE_NO_METHOD` 异常

---

## 7. 主循环底部尾巴（L7700–L7785）

- `OP_NOP`
- `#if !XR_USE_COMPUTED_GOTO` 分支下的 `default:` 处理未知 opcode
- 主循环外的 `handle_closure_pending:` 标签 — 唯一一个跨 dispatch loop 的恢复点
- `#undef R/RA/RB/RC/K/KB/KC/savepc/vmbreak/DISPATCH_OPCODE` 收尾
- 函数尾部注释：`/* ========== API functions moved to xvm_api.c ========== */`

---

## 8. 阅读建议（顺序）

如果重读 VM，建议按以下顺序：

1. `xvm_internal.h` — 上下文/帧/宏定义（理解 `R/K/savepc/VM_FRAMES`）
2. `xvm_jumptab.h` — 跳转表
3. `xvm.c` 顶部 + `startfunc:` 标签 — 帧初始化
4. 基础指令（加载/算术/比较/控制流）
5. `OP_CALL` / `OP_RETURN*` —— **VM 中最复杂的部分**
6. OOP / `OP_INVOKE` / `OP_GETPROP` / `OP_SETPROP`
7. `xvm_cold_paths.c` 配合 `OP_INVOKE` 等理解冷路径出口
8. 协程 / Channel / Scope —— 理解 BLOCKED / YIELD 的恢复语义
9. `xvm_builtins.c` —— 理解 builtin dispatch 的结构
10. `xvm_api.c` —— 理解从 C 端进入 VM 的入口路径
11. `xvm_helpers.c` / `xvm_ops.c` — 辅助函数

---

## 9. 备注

- 本文为**纯架构导览**，不含改造计划。
- 若需要把这些事实转成可执行工程计划，参见独立文档（如 `vm_audit_plan-gpt.md`）。
- 行号引用基于读取时的快照（`xvm.c` = 7784 行）。后续若文件演进，相对位置仍可作为定位参考。
