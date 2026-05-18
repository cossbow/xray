# Runtime `value/` 层分析（`025`）

> 本轮聚焦 `src/runtime/value/`。
>
> 这里名义上是 L1 value 层，但源码实际已经混合了：值表示、静态类型系统、bytecode/proto 契约、格式化展示、builtin method dispatch、struct ABI、type feedback。
>
> 本轮延续上一轮的判断：runtime 的有效范围不止 `src/runtime`，还要把 `src/coro` 视为运行时的一部分；但本文只讨论 `value/` 与这些外部层的交点，不展开 `gc/object/class/symbol/coro` 本体。

## 1. 本轮结论

- **`runtime/value/` 不是单一“值层”，而是一个共享契约层。**
  - 它既定义 `XrValue`，也定义 `XrType`、`XrProto`、`OpCode` 元数据、builtin method table、profile feedback。
  - VM、JIT、AOT、analyzer、DAP、runtime error 打印都直接依赖这里。

- **真正纯 L1 的部分并不多。**
  - 相对纯净的只有：`xvalue.h`、`xvalue_hash.*`、`xslot_type.h`、`xopcode_info.*`、`xstruct_layout.*`、`xtype_feedback.*`、`xtype_opt_hint.*`。
  - `xvalue_format.c`、`xvalue_print.c`、`xmethod_table.*`、`xint/xfloat/xbool_methods.h` 已经明显跨到 object/class/symbol/module/coro/stdlib。

- **`XrProto` 是 `value/` 里最重的共享契约面。**
  - `xchunk.h` 里同时塞了 bytecode、symbol 映射、shared offset、struct layout cache、return/inst/param types、JIT entry、OSR/deopt、stack map、bytecode stackmap、type feedback、blueprint。
  - 这已经不是“值层字节码结构”，而是 runtime/compiler/JIT/AOT 的总协议。

- **类型系统的真实所有权是“进程级 singleton + analyzer pool + isolate 当前池 + TLS fallback”的混合模型。**
  - 这比上一轮 isolate 控制面里看到的状态更分裂。
  - `xtype.c` 并没有完全隔离成“显式传 `X` 的纯 API”。

- **TLS 仍然穿透到了 value/type 热点路径。**
  - `xvalue.c` 的 JSON deep-eq
  - `xvalue_print.c`
  - `xtype.c`
  - `xtype_format.c`

- **本层也存在明显的全局可变状态。**
  - `xtype.c` 的 `g_types_initialized` 与 `g_type_*`
  - `xchunk.c` 的 `s_proto_id_counter`

## 2. 子模块地图

| 子域 | 关键文件 | 主要职责 | 主要消费者 |
|---|---|---|---|
| 值表示 | `xvalue.h` `xvalue.c` `xvalue_hash.*` | `XrValue` 布局、构造、类型检查、深浅比较、hash | VM、object 容器、JIT/AOT helper |
| 静态类型系统 | `xtype.h` `xtype.c` `xtype_format.c` `xtype_generic.c` `xtype_pool.*` `xtype_names.h` | `XrType`、rep/slot/tag 派生、泛型替换、格式化、pool 分配 | analyzer、compiler、JIT/AOT、部分 runtime |
| bytecode/proto | `xchunk.h` `xchunk.c` | `OpCode`、`XrProto`、常量池、lineinfo、proto metadata | compiler、VM、JIT、AOT、DAP |
| opcode 元数据 | `xopcode_info.*` | opcode 名称 / 格式 / 描述 SSOT | disassembler、profiler、JIT builder、DAP |
| struct ABI | `xstruct_layout.*` | value-type struct 的 native layout | compiler、VM、JIT、AOT |
| 类型反馈 | `xtype_feedback.*` | per-proto runtime profile | VM 写入、JIT/AOT 读取 |
| 展示与打印 | `xvalue_format.*` `xvalue_print.*` | 值到字符串、dump、用户输出 | VM、JIT runtime、print/assert/toString |
| builtin method dispatch | `xmethod_table.*` `xbool/xint/xfloat_methods.h` | primitive/builtin method 统一分发表 | VM invoke、JIT runtime、AOT codegen |

## 3. 关键职责与真实消费者

### 3.1 `xvalue_format` 已经是 runtime 横切展示服务

`xr_value_to_string()` / `xr_value_to_strbuf()` 的消费者不是 value 层自己，而是：

- VM 字符串拼接慢路径
- VM `assert_*` 诊断
- VM `toString()` fallback
- JIT runtime string concat / print helper
- `xvalue_print.c` 的统一输出入口

同时 `xvalue_format.c` 实际会格式化：

- Array / Map / Set / Json
- BigInt / DateTime / Enum
- Instance / Class
- Coroutine / Channel
- Module / Exception / Regex / Range / BoundMethod / StringBuilder

它还直接用到：

- `isolate->symbol_table`
- `xmodule.h`
- `xchannel.h`
- `xcoroutine.h`
- `xinstance.h` / `xclass.h`
- `xexception.h`
- `xshape.h`
- `xstdlib_bridge.h`

结论：**`xvalue_format` 不再是“值层工具”，而是 runtime 展示服务。**

### 3.2 `xmethod_table` 是 builtin method 的共享调度协议

`xmethod_table.h` 定义：

- `XrMethodFn`
- `XrMethodSlot`
- `xr_builtin_method_tables[]`
- `xr_method_table_lookup()`

实际消费者有：

- `vm/xvm_dispatch_invoke.inc.c`
- `vm/xvm_cold_coro.c`
- `jit/xir_jit_runtime.c`

这套协议把 builtin method dispatch 从 VM/JIT/AOT 抽成了单一真相源，方向是对的；但它也带来了两个现实：

- value 层直接依赖 `symbol_id`
- primitive method 头文件要 include object/coro/stdlib 能力

### 3.3 `xopcode_info` 的放置是合理的

`xopcode_info.*` 当前是比较干净的一块：

- `xdebug.c` 用它做反汇编
- `xvm_profiler.c` 用它做 opcode 命名
- `xir_builder.c` 用它做 NYI 诊断
- `app/dap/xdap_variables.c` 用它展示指令

这说明把 opcode metadata 放在 `runtime/value` 而不是 `vm/` 是对的：**它是 bytecode 契约，不是 VM 私货。**

### 3.4 `xtype_feedback` 与 `xtype_opt_hint` 也是正向放置

这两块都属于“多消费者共享元信息”：

- `xtype_feedback`：VM 写，JIT/AOT 读
- `xtype_opt_hint`：analyzer 和 codegen 都用

它们放在 `runtime/value` 可以避免 analyzer/codegen/JIT 之间产生反向 include。

## 4. 状态归属

### 4.1 `XrValue`

`XrValue` 是 16-byte POD tagged union：

- 没有集中 owner
- 调用者按值传递
- `PTR` payload 的真实 owner 在 GC heap/object/coro

这部分语义相对清楚。

### 4.2 `XrType`

`XrType` 的所有权是混合式：

- **进程级 singleton**
  - `xtype.c` 中的 `g_type_int`、`g_type_float`、`g_type_json` 等
- **pool arena 分配类型**
  - `XrTypePool` 持有复杂类型、泛型实例、union、tuple、object field 名数组等
- **当前 pool 入口**
  - isolate 上有 `current_type_pool`
  - 同时还保留 `xray_isolate_current()` + TLS fallback

这导致 `XrType` API 表面上允许显式传 `X`，但很多路径仍然依赖“当前线程已经装好 type pool”。

### 4.3 `XrTypePool`

`xtype_pool.*` 本身设计是干净的：

- arena 分配
- pool reset/destroy 统一回收
- `next_type_id` per-pool

但 `xtype.c` 又通过：

- `resolve_isolate(X)`
- `xray_isolate_current()`
- `X->current_type_pool`

把它重新绑回 isolate/TLS 上。

结论：**pool 实现是独立的，pool 使用协议不是独立的。**

### 4.4 `XrProto`

`XrProto` 当前由 compiler 构建、由 owner 负责 free，但内部已经混合了：

- 编译期数据
  - code/constants/protos/upvalues/lineinfo/locvars
  - `symbols`
  - `shared_offset`
  - `param_types` / `inst_types` / `return_type_info`

- 运行期 profile
  - `type_feedback`
  - `call_count` / `exec_count`

- JIT/AOT metadata
  - `jit_entry` / `jit_resume_entry`
  - `bb_leaders` / `loop_headers`
  - `osr_entries` / `deopt_table`
  - `stack_map` / `bc_stackmap`
  - `blueprint`

它名义上叫 function prototype，实际上已经是 **bytecode 单元 + runtime side metadata 的总仓库**。

### 4.5 全局静态表

这类状态相对清楚：

- `xopcode_info.c` 的 `opcode_table[]`：全局只读
- `xmethod_table.c` 的 `xr_builtin_method_tables[]`：全局只读
- `xtype_names.h` 的名字常量：编译期常量

## 5. 边界判断

### 5.1 相对纯净的低层部分

这几块比较接近真正的 L1：

- `xvalue.h`
- `xvalue_hash.*`
- `xslot_type.h`
- `xopcode_info.*`
- `xstruct_layout.*`
- `xtype_feedback.*`
- `xtype_opt_hint.*`

它们要么是纯数据布局，要么是跨多层共享但不强依赖高层实现。

### 5.2 已经不是纯值层的部分

#### `xvalue.c`

虽然名义是 value implementation，但它直接 include：

- `object/xstring.h`
- `object/xarray.h`
- `object/xmap.h`
- `object/xjson.h`
- `gc/xgc.h`
- `class/xclass.h`

更重要的是，`xr_json_equals_deep()` 会走：

- `xray_isolate_current()`
- `xr_json_shape(X, json)`

也就是说，**deep equality 已经隐式依赖 TLS isolate。**

#### `xtype.*`

`xtype.h/c` 不只是“类型标签”，而是：

- 编译期类型系统
- 运行期表示推导
- class/interface/constraint 关系
- generic substitution
- JSON field compatibility
- iterator/iterable 结构性判断

它已经带有明显 analyzer/codegen 语义，只是被放在 runtime/value 作为共享契约。

#### `xvalue_print.c`

`xr_value_fprint()` 先走 `xray_isolate_current()`：

- 有 isolate → 走 canonical formatter
- 无 isolate → 走手写 fallback

同时 dump 路径还 include class/object/symbol/stdlib bridge。它不是纯 I/O wrapper，而是另一套展示逻辑入口。

#### `xmethod_table` 与 primitive method headers

- `xmethod_table.c` 直接 include object methods 和 `stdlib/datetime` methods
- `xint_methods.h` include `xcoroutine.h`，因为 `int.toBigInt()` 走 `xr_current_coro(iso)` 分配

这说明 builtin primitive method 已经把 value 层拉到了 object/coro/stdlib 边界。

## 6. 已确认高风险点

### 6.1 进程级可变全局状态仍在 value/type 核心路径中存在

当前至少有两类：

- `xtype.c`
  - `g_types_initialized`
  - `g_type_*` / `g_type_*_nullable`
- `xchunk.c`
  - `s_proto_id_counter`

这和“控制面尽量回到 isolate / owner 上”是冲突的，也违反当前项目对文件作用域可变全局的长期约束。

### 6.2 `proto_id` 是进程级单调计数，和 per-coro IC table 存在天然张力

`xchunk.c` 明确写了：

- `proto_id` 全进程单调递增、永不复用
- `XrVMContext.ic_*_tables` 以 `proto_id` 直接索引

这意味着：

- IC table 容量受**全进程最大 proto_id**影响
- 不是受“当前 isolate 内活跃 proto 数量”影响

如果一个进程里顺序创建很多 isolate / proto，后续 isolate 的 IC table 可能会被迫向很大的稀疏索引增长。

### 6.3 `xchunk.h` 已经成为高压共享头

`XrProto` 当前同时暴露给：

- compiler
- VM
- JIT
- AOT
- DAP/disassembler
- GC stack map
- feedback/profile

这会带来两个问题：

- 任何字段变更都会波及多个模块
- 头文件层面的“看起来属于 value/”会掩盖真实的跨层冲击面

### 6.4 类型层仍然严重依赖 TLS fallback

典型路径：

- `xtype.c:resolve_isolate()`
- `xr_type_set_current_pool()`
- `xr_type_get_current_pool()`
- `xtype_format.c:xr_type_to_string()`
- analyzer 多处在诊断前先显式 `xr_type_set_current_pool(...)`

这说明：**`xr_type_to_string()` 等 API 的隐式前提不是“给我 type 就行”，而是“当前线程已经装好了 pool/TLS 上下文”。**

### 6.5 `xr_type_to_string()` 的返回值所有权是隐藏契约

它可能返回：

- 静态字符串
- 也可能返回 pool-arena 分配字符串

而调用者签名只看到 `const char *`。

当前 analyzer 大量把它用于诊断字符串，这是可工作的，但契约是隐藏的：**调用点必须保证当前 pool 活着。**

### 6.6 `xr_valuearray_add()` 的常量去重隐式依赖 deep-eq 语义

`xchunk.c` 的 constant dedup 走的是：

- `xr_value_deep_eq(existing, value)`

而 `xr_value_deep_eq()` 对 JSON 比较又会依赖当前 isolate 取 shape。

因此 constant pool 去重并不是纯数据操作，而是带有：

- object 语义
- JSON shape 语义
- TLS isolate 依赖

### 6.7 `xstruct_layout.h` 用裸数字映射 `XrTypeKind`

`xr_type_kind_to_native()` 当前写的是：

- `case 3` // bool
- `case 0` // int
- `case 1` // float
- `case 2` // string

这非常脆弱：只要 `XrTypeKind` 枚举顺序调整，struct native mapping 就会静默漂移。

### 6.8 equality 语义已经分裂成两套

- `xr_value_eq()`：浅比较
  - primitive 按值
  - string 按内容
  - object 按指针
- `xr_value_deep_eq()`：深比较
  - Array/Map/Json 递归

这本身不一定错，但目前：

- Map/Set/hash 更接近 shallow 语义
- constant pool dedup 走 deep 语义

如果不把消费者边界写清楚，后续很容易再出现“某处以为是值语义，实际走的是引用语义”的问题。

## 7. 正向发现

### 7.1 `xopcode_info` 是成功的单一真相源

这一点已经服务了：

- VM debug
- profiler
- JIT builder
- DAP

这是可以继续复用的好模式。

### 7.2 `xtype_feedback` 的放置优于放在 `jit/`

它把“runtime profile 是共享资产”表达得比较清楚，避免 runtime 反向 include jit。

### 7.3 `xtype_opt_hint` 也放在了正确的共享层

它不是 codegen 私货，也不是 analyzer 私货，而是 type-derived hint。放在 `runtime/value` 是合理的。

### 7.4 `xstruct_layout` 把 struct 的 native ABI 明确定义出来了

对于后续分析：

- value type
- GC stack scan
- VM `OP_STRUCT_*`
- JIT/AOT struct lowering

它都是关键底座。

## 8. 改进建议

### 8.1 把 `runtime/value` 重新按职责拆成更清楚的子域

建议至少在心智模型上先分成：

- **value-core**
  - `xvalue.*`
  - `xvalue_hash.*`
  - `xslot_type.h`

- **type-system**
  - `xtype.*`
  - `xtype_pool.*`
  - `xtype_format.c`
  - `xtype_generic.c`
  - `xtype_feedback.*`
  - `xtype_opt_hint.*`

- **bytecode-contract**
  - `xchunk.*`
  - `xopcode_info.*`
  - `xstruct_layout.*`

- **presentation / builtin-dispatch**
  - `xvalue_format.*`
  - `xvalue_print.*`
  - `xmethod_table.*`
  - `xbool/xint/xfloat_methods.*`

即使暂时不挪目录，也应该先按这个边界审视依赖。

### 8.2 把 TLS fallback 从类型层 API 中逐步清掉

优先关注：

- `xr_type_to_string()`
- `xr_type_set_current_pool()`
- `resolve_isolate(X)`
- `xr_json_equals_deep()`

目标不是“一刀切删 TLS”，而是让这些 API 的上下文来源显式可见。

### 8.3 重新定义 `XrProto`：核心字节码 vs side metadata

一个可行方向是拆成：

- immutable proto core
  - code / constants / subprotos / upvalues / lineinfo / symbols
- side metadata
  - JIT runtime state
  - feedback
  - stackmaps
  - blueprint
  - AOT/JIT helper data

至少要让“谁改、谁拥有、谁释放”更清楚。

### 8.4 重新审视 `proto_id` 的作用域

要么：

- 改成 per-isolate / per-module 分配

要么：

- 保留全局 ID，但不要让 IC tables 直接用它做稀疏索引

否则 value 层会继续把全进程历史编译规模泄漏到每个新 isolate 的 ctx 内存布局里。

### 8.5 把 `xvalue_format` 从“值层工具”当成横切服务来治理

它后续最好明确成：

- 接受显式上下文
- 不再直接碰 `isolate->symbol_table`
- 尽量通过 accessor/helper 读取 module/class/coro/debug 相关信息

### 8.6 清理裸数字与隐藏契约

优先级不一定最高，但收益很稳：

- `xstruct_layout.h` 里的裸 `kind` 数字改成枚举常量
- `xr_type_to_string()` 返回值所有权写清楚
- `xr_value_eq()` / `xr_value_deep_eq()` 的消费者边界写清楚

## 9. 给下一轮 `gc/ + closure/` 的输入

下一轮重点建议沿这几条线继续：

- `XrValue` 的 `PTR` / `STRUCT_REF` / `TAGGED` 边界如何被 GC 扫描使用
- `XrProto.bc_stackmap` / `stack_map` / `slot_type` 和 GC root 扫描的接线
- `UpvalInfo.slot_type` / `storage_mode` / `source` 与 closure 捕获策略的关系
- `xvalue_format` / `xmethod_table` 里哪些分配实际落到 `current_coro` 或 coro heap
- `XrTypeRep` / `XrSlotType` 与 GC traversal、typed storage 的交汇点

---

## 10. 本轮状态

- **已完成**
  - `value/` 目录地图梳理
  - `XrValue` / `XrType` / `XrProto` 三条主契约的职责与 ownership 判断
  - formatter / method table / opcode metadata / feedback 的消费者映射
  - TLS fallback、全局可变状态、`proto_id`、`XrProto` 膨胀四类热点识别

- **未完成**
  - 还没进入 `gc/closure` 对 `slot_type` / `stackmap` / `upvalue` 的深入验证
  - 还没统计 `xtype.h` 导出 API 的消费者分布
  - 还没做 `xvalue_eq` 与容器语义的一致性回溯

- **下一步**
  - 开始 `026-runtime-phase2-gc-closure.md`
