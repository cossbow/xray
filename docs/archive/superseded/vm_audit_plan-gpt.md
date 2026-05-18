# VM 模块审计与收敛计划（`src/vm`）[GPT]

> 日期：2026-04-25
> 状态：Draft
> 范围：`src/vm`
> 来源：基于一次静态源码审计整理而成
> 原则：**不考虑向后兼容性，直接采用最佳设计**
> 目标：先收敛 correctness / ownership / architecture boundary，再推进性能和文件拆分

---

## 1. 文档定位

这不是一份“给 VM 再加几个快路径”的优化清单，而是一份把当前 `src/vm` 审计结论收敛成可执行工程计划的文档。

这次审计的总体判断是：

- `src/vm` 已经不只是字节码执行器
- 它同时承担了内建方法语义、协程执行胶水、JIT 反馈采集、异常传播和调试接线
- 当前主要矛盾不是“缺少某个微优化”，而是 **职责膨胀 + 共享可变状态 + 入口 API 健壮性不一致 + 文件规模失控**

因此本计划优先级如下：

1. **先止血**：修入口路径、分配安全、共享反馈状态、异常语义等 correctness 问题
2. **再收边界**：缩小 VM 直接承载的语言/stdlib 语义，恢复层次关系
3. **最后瘦身**：拆大文件、清理重复元数据、把性能优化放到测试保护下进行

---

## 2. 审计范围

这次静态审计主要覆盖以下文件：

- `src/vm/xvm.c`
- `src/vm/xvm_api.c`
- `src/vm/xvm_helpers.c`
- `src/vm/xvm_ops.c`
- `src/vm/xvm_internal.h`
- `src/vm/xvm_cold_paths.c`
- `src/vm/xvm_builtins.c`
- `src/vm/xvm_profiler.c`
- `src/vm/xdebug.c`
- `src/vm/xic_field.c` / `xic_field_table.c`
- `src/vm/xic_method.c`
- `src/vm/xvm_jumptab.h`

重点检查的维度包括：

- VM 入口点与执行主循环
- `vm_ctx` / 调用帧 / 协程上下文模型
- opcode dispatch 与冷路径拆分
- method/field inline cache 所有权
- builtins / property dispatch 的职责边界
- 异常传播与 stacktrace
- GC / safepoint / 调试 hook 接线
- profiler / disassembler / opcode 元数据
- 单测与回归覆盖缺口

---

## 3. 当前基线：优点与保留资产

当前 `src/vm` 并不是“推倒重来”的状态。相反，它已经有一批值得保留的成熟资产：

- **执行主循环路线是对的**
  - `run()` 采用 computed goto
  - 有明显的 hot/cold path 意识
  - 局部宏、寄存器布局、dispatch 方式都已经是性能导向设计

- **协程集成比较深**
  - VM 不是脱离 runtime 的独立解释器
  - `vm_ctx`、call frame、await/yield、worker/coro 模型已经进入统一运行时路径

- **性能与调试设施已经具备基础形态**
  - 有 inline cache
  - 有 profiler
  - 有 disassembler
  - 有 debugger hook 接线

- **JIT 反馈和解释器执行有一定协同**
  - `proto->ic_fields` / `proto->ic_methods` 说明解释器与后续编译路径之间已经建立了反馈通道

这些部分都应该保留，但必须把 ownership、边界和契约理顺，否则越往后改越难。

---

## 4. 核心结论

### 4.1 这次审计的核心判断

`src/vm` 当前最大的问题不是“慢”，而是：

- **VM 过胖**：执行器直接承载大量 builtins/stdlib 语义
- **上下文权威分散**：`vm_ctx` 获取路径不统一，旧模型残片仍在
- **共享可变反馈结构风险高**：IC 挂在共享 `proto` 上，却缺少明确同步/owner 模型
- **入口 API 健壮性不一致**：某些路径做 grow，某些路径只靠 `XR_DCHECK`
- **文件规模与重复元数据已影响继续演进**

### 4.2 如果继续按当前状态迭代，风险会集中在五个方面

1. **正确性风险**
   - C 调脚本闭包入口在 release 下可能直接越界
   - 分配失败与 `realloc` 失败处理不完整

2. **并发/所有权风险**
   - 多 worker 并发运行同一 `proto` 时，IC 更新可能出现 data race
   - profiler 是全局可变对象，当前并发语义不清晰

3. **语义漂移风险**
   - builtin 错误有的抛异常，有的返回 `null`，有的直接 `fprintf(stderr)`
   - exception stacktrace 更像“当前位置记录”，未明确保证完整调用链

4. **架构沉积风险**
   - `vm` 继续吸收 `runtime/class/object/stdlib/api` 细节后，边界只会越来越脏
   - 已经出现 `vm -> api` 反向 include 的迹象

5. **回归盲区风险**
   - 当前更多依赖 `.xr` 黑盒回归
   - VM API / IC / exception / disassembler 缺少对应的低层单元测试

---

## 5. 当前问题总览

| ID | 优先级 | 主题 | 主要文件 | 结论 |
|----|--------|------|----------|------|
| VM-01 | P0 | 入口 API grow 不完整 | `xvm_api.c` | `xr_vm_call_closure()` / `xr_vm_interpret_proto()` 仍有 release 越界风险 |
| VM-02 | P0 | IC 共享可变状态 | `xvm.c` `xic_*` `jit/*` | method/field IC 挂在共享 `proto` 上，无锁更新有 data race 风险 |
| VM-03 | P0 | 分配安全与 `realloc` 纪律 | `xvm.c` `xic_*_table.c` | `xr_calloc` 判空、`xr_realloc` 临时指针等规则未完全遵守 |
| VM-04 | P0 | `vm_ctx` 权威分散 + 旧模型残片 | `xvm_api.c` `xvm_helpers.c` `xvm_internal.h` | 仍残留 `isolate->vm` 风格 helper，后续极易误用 |
| VM-05 | P1 | 异常 stacktrace 语义偏浅 | `xvm_api.c` `xvm.c` | 目前更像“throw 位置记录”，不清楚是否保证完整调用链 |
| VM-06 | P1 | builtin 错误语义不一致 | `xvm_builtins.c` | `null` / `false` / `XR_NOTFOUND` / `fprintf` 混用，脚本层难收敛 |
| VM-07 | P1 | VM 职责过胖，直接承载 stdlib 语义 | `xvm_builtins.c` | builtins 太多，且直接 include stdlib/runtime 细节 |
| VM-08 | P1 | profiler 全局可变且并发模型不清 | `xvm_profiler.c` | 全局对象 + 每次 `run()` 初始化，不适合多 worker |
| VM-09 | P1 | opcode 元数据重复维护 | `xvm_jumptab.h` `xvm_profiler.c` `xopcode_info.c` | 名称/分类/说明分散，多处重复 |
| VM-10 | P1 | 文件规模超限 | `xvm.c` `xvm_cold_paths.c` | 已经显著超出项目建议规模，review 和回归风险高 |
| VM-11 | P2 | deep compare 固定上限导致假阴性 | `xvm_ops.c` | 深度和 visiting pair 上限更像实现限制，不像稳定语义 |
| VM-12 | P2 | 热路径仍有动态分配与细粒度 debug hook | `xvm.c` | `startfunc` 中仍有分配；debug line hook 逐指令检查 |

---

## 6. 问题明细与处理方向

### 6.1 P0：correctness / ownership / 基础边界

### VM-01 `xr_vm_call_closure()` / `xr_vm_interpret_proto()` 的 grow 逻辑不完整

当前问题：

- `xr_vm_execute_module()` 已经有比较完整的 stack/frame grow 处理
- 但 `xr_vm_call_closure()` 与 `xr_vm_interpret_proto()` 仍主要依赖 `XR_DCHECK`
- release 构建下，这类入口如果遇到大 `maxstacksize` 或深调用，可能直接越界写

结论：

- **所有进入 VM 的公共入口都必须经过统一的 prepare/grow 路径**
- 不允许“某个入口能 grow，另一个入口只断言”的状态继续存在

设计决策：

- 新增统一内部 helper，例如：
  - `xr_vm_prepare_entry_frame()`
  - `xr_vm_ensure_stack_and_frames()`
- `xr_vm_call_closure()` / `xr_vm_interpret_proto()` / `xr_vm_execute_module()` 全部走同一套容量准备逻辑

验收标准：

- 不再存在只靠 `XR_DCHECK` 的 VM 入口容量保护
- 为大 `maxstacksize`、深 frame、vararg/rest 路径补单测

### VM-02 method/field IC 挂在共享 `proto` 上，但当前无明确并发契约

当前问题：

- 解释器路径会读写 `proto->ic_methods` / `proto->ic_fields`
- JIT builder 也会读取这些 feedback
- `xic_method.c` / `xic_field.c` 的更新逻辑不是原子，也没有锁

这在多 worker 并发执行同一 `proto` 时属于典型共享可变状态风险。

结论：

- **解释器 IC 不应继续作为共享 `proto` 上的无锁可变对象存在**

最终方向：

- `proto` 只保留 **不可变字节码 + 只读编译产物/发布元数据**
- 解释器 IC 改为 **worker-local** 或 **vm_ctx-local** feedback
- JIT 如需使用 feedback，应通过显式 snapshot / publish 机制读取，而不是直接读写共享热结构

临时止血方案：

- 如果 Phase 1 无法在一轮内完成迁移，至少先收紧为：
  - 只允许单向 publish 的 monomorphic feedback
  - poly/mega 状态与统计字段先移出共享热路径
  - 关键写路径加 `XR_DCHECK` 说明当前 owner/线程约束

验收标准：

- 共享 `proto` 上不再存在无契约的热路径可变 IC 状态
- JIT 与解释器之间的 feedback 读取方式有清晰发布协议

### VM-03 分配失败处理与 `realloc` 纪律不完整

当前问题：

- `xvm.c` 热路径里仍有 `xr_calloc()` 后不查 NULL、`xr_realloc()` 直接覆盖原指针的模式
- `xic_field_table.c` / `xic_method.c` 也存在类似薄弱点

这不只是代码风格问题，而是 release 下真实的崩溃/内存破坏风险。

结论：

- 必须严格执行项目内存纪律：
  - `xr_malloc/xr_calloc` 后立即判空
  - `xr_realloc` 必须临时指针中转
  - 分配失败要么返回错误码，要么走显式 fail-fast，不允许半更新状态

验收标准：

- `src/vm` 内存分配路径符合现有项目规则
- 关键结构扩容失败时不会遗留半初始化对象

### VM-04 `vm_ctx` 权威分散，且仍残留旧式 `isolate->vm` helper

当前问题：

- `xvm_api.c` 有 `xr_vm_get_current_ctx()`
- `xvm_helpers.c` 又有一套当前上下文选择逻辑
- `xvm_internal.h` 里还残留直接面向 `isolate->vm` 的 frame helper

这会导致：

- 同一概念在多个文件里各自实现
- 新代码很容易误走旧模型
- 后续 `vm_ctx` 再演进时修改点不收敛

结论：

- **`vm_ctx` 的 authoritative getter 必须只有一个**
- `isolate->vm` 风格的旧 helper 要么删除，要么改写为显式接收 `XrVMContext *ctx`

目标形态：

- 所有 VM 内部 helper 都以 `ctx` 为核心参数
- 不再从 helper 内部偷偷回退到 `isolate->vm`

验收标准：

- `src/vm` 内不存在新的 `isolate->vm.*` 直连路径
- 旧 helper 清理完成后，调用方只依赖统一的 `ctx` 模型

---

### 6.2 P1：语义一致性 / 架构边界 / 可维护性

### VM-05 exception stacktrace 语义需要从“当前点记录”升级为“完整调用链”

当前问题：

- `xr_vm_add_stacktrace()` 当前更像记录当前帧
- `xr_vm_throw_exception()` 的展开过程没有形成清晰的“完整栈追踪 contract”

结果是：

- 异常对象上的 stacktrace 可能不完整
- JIT/解释器/冷路径 throw 的观感可能不一致

结论：

- **异常对象上的 stacktrace 应代表完整调用链**
- 不能继续停留在“遇到 throw 就记一帧”的半隐式语义

设计决策：

- 明确选择“在 throw + unwind 过程中完整追加帧信息”的模型
- 把这一语义写进 VM 内部 contract，而不是靠调用点猜测

验收标准：

- 同一异常在普通解释路径、冷路径、JIT 回退路径下都能给出一致 stacktrace
- 新增 stacktrace 完整性测试

### VM-06 builtin 错误语义必须统一

当前问题：

`xvm_builtins.c` 中当前混用了：

- `xr_null()`
- `xr_bool(false)`
- `XR_NOTFOUND`
- `fprintf(stderr, ...)`

这会导致脚本层难以区分：

- 数据语义上的“不存在”
- 调用契约上的“类型错误/参数错误”
- 真正的运行时异常

结论：

- builtin 行为必须收敛成统一规则：
  - **数据语义结果**：如 `Map.get` miss、`match` not found，可返回约定值
  - **调用契约错误**：参数个数/参数类型错误，一律抛可捕获异常
  - **方法不存在**：统一走 VM 层 method-not-found 路径
- 不允许在 VM builtin 里继续直接 `fprintf(stderr)` 作为错误模型

验收标准：

- 所有 builtin handler 的错误语义清晰一致
- 脚本层能通过 `try/catch` 稳定接收 contract error

### VM-07 VM 职责过胖，builtins 不应继续在 VM 层膨胀

当前问题：

- `xvm_builtins.c` 已经直接承载大量 `String/Array/Map/Set/Json/Regex/Datetime` 等语义
- VM 还直接 include stdlib/runtime 对象细节
- `xvm_helpers.c` 出现了 `../api/xglobal_object.h` 的反向 include 痕迹

这说明 VM 层已经开始吞掉上层语言对象系统的职责。

结论：

- **VM 只负责 dispatch，不应继续直接实现越来越多的类型方法语义**

最终方向：

- builtin method registry 下沉到 `runtime/class` 或更低层的对象系统
- `xvm_builtins.c` 只保留极薄的桥接层，或者最终删除
- 移除 `vm -> api` 反向 include；若确需能力，应改成向低层 hook / registry 取服务

验收标准：

- `src/vm` 不再直接知道大批 stdlib 类型实现细节
- `vm -> api` 反向依赖被彻底消除

### VM-08 profiler 需要从“全局实验工具”收敛为“并发安全的可观测设施”

当前问题：

- `g_vm_profiler` 是全局对象
- `run()` 进入时会初始化 profiler
- 多 worker 并发时，统计和 reset 的语义都不清晰

结论：

- profiler 必须明确到底是：
  - per-run
  - per-worker
  - per-isolate
- 当前最合理方向是 **per-isolate / per-worker opt-in collector**，禁止全局隐式共享

验收标准：

- profiler 生命周期与并发模型文档化
- 不再依赖全局可变对象承载多 worker 统计

### VM-09 opcode 元数据需要单一真相源

当前问题：

- opcode 名称、说明、跳转表、profile 名称等分散在多个文件维护
- 其中有些已经开始复用，有些还在各自维护副本

结论：

- opcode 元数据必须改成单一真相源
- 建议采用统一表/X-macro，派生：
  - `xopcode_info`
  - profiler 名称
  - disassembler 显示名
  - 如有必要的 dispatch 声明辅助信息

验收标准：

- 新增 opcode 时只改一处主表
- profiler/disasm/info 不再手工同步字符串

### VM-10 文件规模已明显超限，必须拆分

当前问题：

- `xvm.c` 体量已经远超项目建议规模
- `xvm_cold_paths.c` 也已经变成第二个大文件

这会直接带来：

- review 成本上升
- unrelated bug 互相污染
- 断言、不变量和局部约束难以维护

结论：

- 必须把 VM 拆成按职责聚类的文件，而不是继续在 `xvm.c` 上加 case

建议拆分方向：

- `xvm_exec.c`：主循环和 dispatch 骨架
- `xvm_call.c`：`CALL` / `RETURN` / frame management
- `xvm_object.c`：`GETPROP` / `SETPROP` / invoke / builtin bridge
- `xvm_exception.c`：`TRY` / `THROW` / unwind / stacktrace
- `xvm_coro.c`：`GO` / `AWAIT` / `YIELD` / scope / channel 指令桥接
- `xvm_debug.c`：debug hook / line stepping / trace

验收标准：

- 每个文件只承担单一职责簇
- `xvm_internal.h` include 扇出缩小

---

### 6.3 P2：长期语义完善 / 性能优化

### VM-11 deep compare 的固定深度/visited 上限更像实现限制，不像语言契约

当前问题：

- `xvm_ops.c` 的深比较依赖固定深度与固定 visiting pair 上限
- 环状结构与超大对象比较会出现假阴性风险

结论：

- 需要在两条路径中二选一：
  - 明确把当前限制写入语言/运行时语义
  - 或改成真正的 visited set/hash 实现

推荐方向：

- 采用真实 visited set，避免“结构相等但因为上限而返回 false”的隐式行为

### VM-12 热路径分配与 debug line hook 仍有优化空间

当前问题：

- `startfunc` 中仍有动态分配/扩容
- debug line hook 逐指令检查，debug 模式下开销偏高
- builtin dispatch 仍有大量手写分支链

结论：

- 这些问题目前不是第一优先级，但只能在测试覆盖补齐后推进
- 顺序必须是：**先有 contract 和测试，再开快路径**

推荐方向：

- 把与 frame 生命周期绑定的辅助存储预分配或池化
- line hook 改成 line-change / statement boundary 级触发
- builtin dispatch 改用表驱动/registry 驱动，而不是持续扩长 `if (symbol == ...)`

---

## 7. 目标架构形态

本计划结束后的理想形态应当是：

### 7.1 `proto` 以不可变为主

`XrProto` 主要承载：

- 字节码
- 常量表
- 调试信息
- 只读发布后的 JIT 入口/元数据

不再承载无契约的共享热路径可变反馈结构。

### 7.2 `vm_ctx` 是唯一运行时权威

所有运行期状态通过 `XrVMContext` 进入：

- stack
- frames
- handler stack
- current coro
- stack top
- module base frame

VM helper 不再隐式访问旧式 `isolate->vm` 字段。

### 7.3 VM 只做执行与调度，不直接承载大块对象语义

VM 负责：

- opcode dispatch
- call frame 管理
- 异常与控制流
- 与对象系统/类系统的最小桥接

对象方法语义、builtin registry、类型细节下沉到更合适的 runtime/class/object 层。

### 7.4 异常与调试具备明确 contract

- exception stacktrace = 完整调用链
- builtin contract error = catchable exception
- debugger line hook = 明确的触发粒度
- profiler = 明确生命周期和并发归属

---

## 8. 实施计划

## Phase 0：止血与入口收敛（2-3 天）

目标：先把 release 越界、分配安全、上下文权威这些“必须马上收住”的问题处理掉。

| # | 任务 | 主要文件 |
|---|------|----------|
| 0.1 | 统一 VM 入口 grow helper | `xvm_api.c` `xvm_internal.h` |
| 0.2 | 修 `xr_vm_call_closure()` / `xr_vm_interpret_proto()` 容量准备 | `xvm_api.c` |
| 0.3 | 清理 `xr_calloc` / `xr_realloc` 违规点 | `xvm.c` `xic_*` |
| 0.4 | 统一当前 `vm_ctx` getter | `xvm_api.c` `xvm_helpers.c` `xvm_internal.h` |
| 0.5 | 删除或改写残留 `isolate->vm` helper | `xvm_internal.h` |
| 0.6 | 清掉 `vm -> api` 反向 include | `xvm_helpers.c` |
| 0.7 | 新增 VM 入口 API 单测 | `tests/unit/vm/*` |

验收标准：

- VM 所有公共入口都具备一致的 grow 策略
- `src/vm` 内存分配路径符合现有项目规则
- `vm_ctx` 权威路径唯一化
- 低层 helper 不再暗含 `isolate->vm` 旧模型

## Phase 1：共享状态与反馈模型收敛（3-4 天）

目标：解决最危险的共享可变 feedback 问题。

| # | 任务 | 主要文件 |
|---|------|----------|
| 1.1 | 明确 IC ownership 设计（worker-local / ctx-local） | `xic_*` `xvm.c` `jit/*` |
| 1.2 | 迁移或收紧 `proto->ic_*` 读写模型 | `xvm.c` `xic_*` `jit/*` |
| 1.3 | 为 feedback publish 定义最小 contract | `jit/*` `vm/*` |
| 1.4 | 新增并发/一致性回归测试 | `tests/unit/vm/*` `tests/*` |
| 1.5 | profiler 生命周期定型（per-worker/per-isolate） | `xvm_profiler.*` |

验收标准：

- 不再存在共享 `proto` 上的无契约热路径可变 IC
- JIT/解释器 feedback 交互方式显式化
- profiler 并发归属清晰，不再是全局隐式共享

## Phase 2：语义与边界收敛（3-4 天）

目标：统一 builtin 错误语义、异常 trace 语义，并把 VM 从过胖对象语义中剥离出来。

| # | 任务 | 主要文件 |
|---|------|----------|
| 2.1 | 明确 exception stacktrace contract 并实现 | `xvm_api.c` `xvm.c` |
| 2.2 | 统一 builtin contract error 语义 | `xvm_builtins.c` |
| 2.3 | 移动 builtin registry 到更低层对象系统 | `xvm_builtins.c` `runtime/class/*` |
| 2.4 | 收敛 method-not-found / data-miss 语义 | `xvm.c` `xvm_builtins.c` |
| 2.5 | opcode 元数据改成单一真相源 | `xopcode_info.c` `xvm_profiler.c` `xdebug.c` |

验收标准：

- builtin handler 不再使用 `fprintf(stderr)` 作为错误模型
- exception stacktrace 在解释/JIT/cold path 下语义一致
- `src/vm` 对 stdlib/runtime 细节依赖显著缩小
- opcode 元数据增改不再需要多处手工同步

## Phase 3：结构瘦身与性能收敛（4-6 天）

目标：在 Phase 0-2 的 contract 和测试保护下，拆分大文件并处理确定性的性能热点。

| # | 任务 | 主要文件 |
|---|------|----------|
| 3.1 | 拆分 `xvm.c` 为按职责聚类的文件 | `src/vm/*` |
| 3.2 | 精简 `xvm_cold_paths.c`，把真正冷路径与共享逻辑分离 | `xvm_cold_paths.c` |
| 3.3 | 优化 frame enter 相关动态分配 | `xvm.c` / 拆分后对应文件 |
| 3.4 | 优化 debugger line hook 粒度 | `xvm.c` / `xdebug*` |
| 3.5 | builtin dispatch 改为 registry/table 驱动 | `runtime/class/*` `vm/*` |
| 3.6 | 评估并修复 deep compare 假阴性问题 | `xvm_ops.c` |

验收标准：

- `xvm.c` 不再承担全部 VM 职责
- 热路径 allocations 和 debug overhead 有明确下降
- 深比较语义从“实现限制”升级为“明确 contract”

---

## 9. 测试计划

当前 `src/vm` 的短板不是没有回归，而是 **缺低层、定点、可复现的单元测试**。

建议新增 `tests/unit/vm/`，至少覆盖：

- `test_vm_api.c`
  - `xr_vm_call_closure()` 大栈/深帧/vararg/rest
  - `xr_vm_interpret_proto()` 大 `maxstacksize`

- `test_vm_exception.c`
  - `throw/catch/finally`
  - stacktrace 完整性
  - builtin contract error 可 catch

- `test_ic_field.c`
  - field IC 状态迁移
  - publish/lookup 契约

- `test_ic_method.c`
  - method IC 状态迁移
  - 多态/mega 退化策略

- `test_vm_builtins.c`
  - 参数错误 vs 数据 miss 语义
  - `XR_NOTFOUND` 与 catchable error 分界

- `test_vm_debug.c`
  - disassembler 基本输出
  - line hook / breakpoint 粒度契约

另外，语言级 `.xr` 回归应补以下场景：

- 深调用链异常与 stacktrace
- 大栈帧函数调用
- builtin 参数类型错误
- 多 worker 下共享函数/方法热调用
- `Map/String/Json` 的 method-not-found 与合法 miss 场景

---

## 10. 工程纪律

### 10.1 先收 contract，再做快路径

没有测试和语义边界保护的路径，不继续加优化。

### 10.2 每个 VM bug 双落地

每修一个 VM 相关 bug，至少补两类回归：

- 一个语言级 `.xr` case
- 一个 VM/unit 级最小 case

### 10.3 `proto` 不再承载无契约共享可变状态

新代码禁止继续往 `proto` 上挂隐式可变热数据，除非其 publish/read 语义已经被明确写入 contract。

### 10.4 新 helper 以 `ctx` 为中心

VM 内部 helper 默认接收 `XrVMContext *ctx`，禁止重新回到“从任何地方都能摸到全局 VM 状态”的旧模式。

### 10.5 Builtin 语义不能再偷偷扩散进 VM

新增对象/类型方法时，优先扩展对象系统/registry，不允许继续把 `xvm_builtins.c` 变成第二个 runtime。

---

## 11. 非目标

本轮不做：

- 重写解释器 dispatch 模型
- 把 VM/JIT/AOT 统一成一个大而全执行框架
- 以“先抽象后落地”的方式重构 backend/VM 边界
- 在没有测试保护前追逐微基准性能

---

## 12. 结语

`src/vm` 当前的问题不是“已经不可维护”，而是**已经走到必须先收住核心风险，再继续往上堆功能**的节点。

这份计划的关键不是把 VM 变成更花哨的架构，而是收敛几个最重要的事实：

- VM 入口必须在 release 下也安全
- 共享 feedback 必须有 owner 或 publish contract
- builtin/error/stacktrace 语义必须统一
- `vm_ctx` 必须成为唯一运行时权威
- VM 只做执行器该做的事

只要这几条收住，后面的文件拆分、性能优化和 JIT/调试协同都会顺很多。
