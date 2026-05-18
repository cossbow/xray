# Xray AOT 实施计划（GPT Draft）

> Version: 0.1 | Date: 2026-04-23
> Status: Draft
> Relationship: 本文是 `docs/design/aot-design.md` 的实施补充文档，重点描述当前 `src/aot` 的落地路径、优先级、工作包、验收标准与测试策略。

---

## 一、文档目标

本文不重复完整功能设计，而是回答以下问题：

1. 当前 `src/aot` 到底处于什么阶段。
2. 哪些能力已经可用，哪些问题会阻止它成为真正可用的 `--native` 后端。
3. 接下来应该按什么顺序推进，分别改哪些文件、补哪些测试、以什么标准验收。
4. 如何把工作收敛到一个可以宣称为 **AOT Alpha** 甚至后续 **AOT Beta** 的状态。

本文默认遵循以下原则：

- 先保证 **语义正确性**，再谈优化。
- 未支持能力必须 **显式失败**，不能静默生成错误 C 代码。
- 多模块初始化必须遵循 **拓扑顺序**。
- AOT runtime 必须逐步收敛到项目统一工程规则，尤其是内存分配与测试纪律。
- 每个阶段都必须有明确的 **输入、输出、验证命令、退出条件**。

---

## 二、当前状态摘要

### 2.1 当前可用能力

截至本文撰写时，`src/aot` 已经具备以下基础能力：

- 基础算术、控制流、部分 typed 路径可以稳定转成 C。
- 简单 class / method 场景已经能运行。
- `xcgen.c` / `xcgen_call.c` / `xcgen_expr.c` / `xcgen_stmt.c` / `xcgen_struct.c` 已形成清晰职责分层。
- `xrt.h` 及子头文件已经提供 value / ARC / class / collection / module / exception 等 AOT runtime 骨架。
- `xcgen_emit_source()` 已能生成：
  - 模块级共享变量数组 `xrt_shared[]`
  - module export tables
  - `xrt_modules[]` 描述表

### 2.2 当前关键缺口

当前实现仍然存在以下阻断项：

| 维度 | 当前状态 | 结论 |
|------|----------|------|
| `shared` 语义 | `shared_vars.xr` AOT 输出与 VM 不一致 | **P0 阻断** |
| 异常语义 | `try_catch.xr` AOT 与 VM 输出不一致 | **P0 阻断** |
| 未支持 IR 处理 | 仍存在 TODO 注释式降级与 no-op 吞指令 | **P0 阻断** |
| 多模块编译 | `cmd_build_native()` 实际只 AOT 编译入口模块 | **P1 阻断** |
| import 测试覆盖 | `tests/aot` 当前没有真实 `import` 用例 | **P1 阻断** |
| runtime 内存管理 | `xrt_arc.h` / `xrt_class.h` / `xrt_coll.h` 仍直接使用 libc 分配函数 | **P2 工程债** |
| ARC 生命周期 | runtime 有 ARC 设施，但生成端大量 retain/release 是 no-op | **P2 模型未闭环** |

### 2.3 当前阶段判断

当前 `src/aot` 的准确定位应为：

- **不是** 完整可用的 native backend
- **是** 一个已经能跑通有限子集、但仍需收口语义与模块链路的 **AOT Alpha 前阶段实现**

建议对外描述为：

- **“单入口、有限 typed 子集可试用；多模块 / shared / 异常一致性仍在收敛中。”**

---

## 三、实施范围与非目标

### 3.1 本轮实施范围

本轮实施的目标是把 `src/aot` 收敛到至少满足 **AOT Alpha**：

1. 单入口子集的语义与 VM 一致。
2. `shared` / `try-catch` / unsupported IR 的错误路径收口。
3. `--native` 真正开始利用 bundle 中的模块信息，而不是只处理入口文件。
4. module init / export / import 形成闭环。
5. 测试矩阵覆盖真实 AOT 风险点。

### 3.2 本轮非目标

下列事项不作为本轮必须完成项：

- coroutine / channel / 并发 AOT 映射
- 全量标准库 AOT 化
- 多文件 C 输出与增量编译
- 高级性能优化（内联、去虚拟化、跨模块 DCE 等）
- 面向生产环境的 ABI 稳定承诺

这些内容可以在 `AOT Beta` 之后继续推进。

---

## 四、实施原则

### 4.1 先语义，后优化

任何“更快”的 lowering，在没有 VM/AOT 一致性之前都不是有效优化。优先保证：

- 输出一致
- 控制流一致
- 异常行为一致
- shared/global/import 一致

### 4.2 不支持即失败

以下行为必须被禁止：

- 生成 `/* TODO: op ... */` 但整体构建仍返回成功
- 把异常/guard/retain/release 等高语义指令直接吞掉却没有前提证明
- 依赖“当前测试没覆盖到”来容忍错误 lowering

### 4.3 模块初始化必须有唯一入口

模块初始化、导出注册、shared 变量布局、导入解析，都必须挂到同一套 `module init` 机制下。不能继续同时存在：

- `xrt_modules[]` 已生成但未使用
- standalone main 只调用入口 module init
- import/export 仅在表结构层面存在但不参与执行

### 4.4 生命周期模型必须单一

AOT 中对象生命周期必须明确：

- 哪些对象依赖 ARC
- 哪些对象允许进程退出整体回收
- `XIR_RETAIN/RELEASE` 是否应该在 AOT 路径中出现
- bump allocator 在程序生命周期中何时初始化、何时销毁

### 4.5 工程规则必须逐步对齐

AOT runtime 不能长期作为“工程例外区”。尤其需要收敛：

- 内存分配 API
- 错误处理策略
- 测试命令与回归流程
- 限制值超限时的显式诊断

---

## 五、里程碑与退出标准

### 5.1 里程碑定义

| 里程碑 | 目标 | 退出标准 |
|--------|------|----------|
| M1: Alpha Correctness | 收口当前语义错误 | `shared_vars` / `try_catch` 对齐 VM；unsupported IR hard fail；AOT 对拍脚本可用 |
| M2: Module Pipeline | 打通多模块编译与初始化链路 | 至少 2 个真实 import 用例通过；native 编译日志显示多个模块进入 AOT；main 使用 module init 体系 |
| M3: Runtime Cleanup | 收口 runtime 工程债 | runtime 分配统一封装；生命周期模型文档化并实现；关键 AOT 用例通过快速测试与完整回归 |

### 5.2 AOT Alpha 完成标准

满足以下全部条件后，才可以对外宣称 `src/aot` 达到 **AOT Alpha**：

- `tests/aot/basic/*` 中已纳入支持范围的用例全部与 VM 输出一致。
- `shared_vars.xr` 和 `try_catch.xr` 已修复。
- 未支持 IR 会明确报错，不会静默生成错误 C。
- 至少 1 个跨模块函数调用用例和 1 个跨模块常量导入用例通过。
- `--native` 生成的程序实际使用模块初始化表，而不是只调用入口文件的 init。

### 5.3 AOT Beta 候选标准

满足以下条件后，可进入 **AOT Beta** 评估：

- closure / upvalue / shared / import / class / exception 组合路径都有稳定测试。
- runtime 内存管理与项目规范对齐。
- ASAN 或同级别内存检查能跑通关键 AOT 用例。
- 多模块和异常边界路径有清晰错误模型与文档说明。

---

## 六、核心文件与改动焦点

| 文件 | 当前职责 | 本轮改动焦点 |
|------|----------|--------------|
| `src/app/cli/xcmd_build.c` | `--native` 入口，bundle/AOT 编译串联 | 从“只编入口”升级到“遍历 bundle 模块”；接通 module init 与导出收集 |
| `src/aot/xcgen.c` | AOT 编译主上下文、源码拼装 | module/source 组织、错误状态传播、`xrt_modules[]` 真正接线 |
| `src/aot/xcgen_call.c` | call/shared/closure/method lowering | 修复 `GETSHARED/SETSHARED + closure + top-level call`；跨模块调用策略 |
| `src/aot/xcgen_expr.c` | 大多数 XIR 指令 lowering | 异常、unsupported IR、retain/release/no-op 审计 |
| `src/aot/xcgen_stmt.c` | terminator / phi lowering | 与异常 frame / return 恢复逻辑协同 |
| `src/aot/xcgen_struct.c` | struct promotion | 超限诊断、类型推断收敛 |
| `src/aot/xrt_module.h` | 模块初始化与导出查找 runtime | `xrt_modules_init()` / `xrt_module_lookup()` 真正参与执行 |
| `src/aot/xrt_exception.h` | AOT 异常 runtime | try/catch 与 throw 模型统一 |
| `src/aot/xrt_arc.h` | ARC / bump allocator | 生命周期模型、初始化/销毁、分配封装 |
| `src/aot/xrt_class.h` | class/object runtime | 类型表与对象分配策略收敛 |
| `src/aot/xrt_coll.h` | Array/Map/StringBuilder/Closure runtime | 分配封装、析构与 tagged 语义收敛 |
| `tests/aot/**` | AOT 回归用例 | 补真实 import/shared/exception/closure 测试矩阵 |

---

## 七、实施前必须确认的 5 个决策

在大规模修改前，建议先确认以下 5 个设计决策。未确认时，实施容易反复返工。

### 7.1 `--native` 的定位

必须明确：

- 是继续保持 **纯 AOT standalone** 方向；
- 还是短期接受 **AOT + runtime hybrid**，但未覆盖路径保留字节码 fallback。

建议：**本轮先以“纯 AOT 子集可用”作为主目标**，不要同时引入 fallback 复杂性。

### 7.2 跨模块调用策略

建议采用如下规则：

- typed + 已知函数：`extern + direct C call`
- dynamic export access：`xrt_module_lookup()`
- closure value：按函数指针/closure runtime 路径调用

不要混用多套跨模块调用机制。

### 7.3 异常 lowering 策略

必须统一为一套模型。建议：

- `XIR_THROW` 与 `xr_jit_throw` 最终都通过 runtime 异常机制传播；
- `TRY_BEGIN/TRY_END` 负责 frame push/pop；
- `XIR_CATCH` 只读取当前异常对象；
- 不再允许正常路径使用 `abort()`。

### 7.4 生命周期策略

必须回答：

- `XIR_RETAIN/RELEASE` 应该被 AOT lowering，还是应该在进入 AOT 前被消除？
- bump allocator 是否是默认路径？
- 程序结束时是否统一做 arena destroy？

### 7.5 内存分配策略

项目规则要求避免直接散落 `malloc/calloc/realloc/free`。建议：

- 先统一收口到单一封装层，如 `xrt_alloc.h`；
- 再决定该封装层是否直接接 `xr_malloc/xr_free`，或保留一层可切换实现。

---

## 八、工作包（Work Packages）

以下工作包按推荐顺序排列。每个工作包都包含：目标、现状、涉及文件、实施步骤、测试与完成标准。

### WP-01：建立 AOT 对拍与失败诊断基线

#### 目标

建立最小但可重复的 VM/AOT 对拍流程，避免后续修改只能靠人工 eyeballing。

#### 现状 / 根因

当前虽然已有 `tests/aot`，但缺少统一对拍机制：

- 没有一条命令同时跑 VM 与 AOT
- 失败时不保留生成的 C 文件
- 没有标准化的差异输出

#### 涉及文件

- `tests/aot/**`
- 可选新增脚本：`scripts/run_aot_tests.sh`
- 可选接入测试定义文件 / CTest 配置

#### 实施步骤

1. 约定每个 AOT 用例至少支持两种执行方式：
   - VM：`./build/xray <case>.xr`
   - AOT：`./build/xray build --native -o /tmp/<case> <case>.xr && /tmp/<case>`
2. 新增对拍脚本或测试包装器：
   - 逐个执行用例
   - 采集 stdout / stderr / exit code
   - 对失败用例保留生成的 `.c` 文件
3. 输出统一的失败诊断信息：
   - case 路径
   - VM 输出
   - AOT 输出
   - 生成 C 文件路径
4. 后续可逐步接入 `ctest`，但第一步以脚本稳定为主。

#### 新增 / 更新测试

- 先覆盖当前已知高风险用例：
  - `tests/aot/basic/shared_vars.xr`
  - `tests/aot/basic/try_catch.xr`
  - `tests/aot/basic/class_methods.xr`

#### 完成标准

- 可以一条命令跑完整个 AOT case 列表。
- 失败时能直接看到差异与生成物位置。
- 新增问题不再需要手工重复构建才能定位。

---

### WP-02：修复 `shared` 变量与顶层闭包调用链路

#### 目标

修复 `GETSHARED/SETSHARED + closure + top-level call` 组合路径，使 `shared_vars.xr` 与 VM 输出一致。

#### 现状 / 根因

当前已知问题包括：

- `xcgen_call.c` 中 `GETSHARED` 对 native-typed 目标寄存器会生成“elided”注释而不是实际值；
- 顶层 shared closure 被存入 `xrt_shared[]` 后，后续调用没有被正确还原成 C 调用；
- 结果是 top-level 初始化函数里本应执行的 shared function 调用被吞掉或错误 lowering。

#### 涉及文件

- `src/aot/xcgen_call.c`
- `src/app/cli/xcmd_build.c`
- `src/aot/xcgen.c`
- 必要时复查 `src/jit/xir_builder_object.c`

#### 实施步骤

1. 明确顶层 shared function 的 canonical lowering 规则：
   - 如果 shared 槽映射到已知 proto，优先恢复为直接 C 符号调用；
   - 如果确实是动态 closure，再走 `xrt_shared[idx]` + closure runtime 调用路径。
2. 检查 `build_shared_proto_map()` 与实际 call site 是否一致：
   - 不能只记录“shared 槽来自哪个 proto”，还要在调用端真正使用这个映射。
3. 调整 `emit_call_c()` 中 `GETSHARED` 与 `SETSHARED` 逻辑：
   - shared 槽布局、导出索引、closure value 的生成方式必须统一；
   - native-typed 目的寄存器不能再用注释代替真实语义。
4. 审计顶层 `__module_init` 中的 call lowering：
   - 确认生成的 C 会真的调用 shared function，而不是只构造 closure 然后直接打印错误值。
5. 增加回归 case：
   - shared function 调用 1 次
   - shared function 调用多次
   - shared let 在函数中更新并在另一个函数中读取

#### 新增 / 更新测试

- `tests/aot/basic/shared_vars.xr`
- 可新增：
  - `tests/aot/basic/shared_closure_once.xr`
  - `tests/aot/basic/shared_closure_multi.xr`

#### 完成标准

- `shared_vars.xr` 的 AOT 输出与 VM 完全一致。
- 生成 C 中不再出现 shared function 调用被吞掉的现象。
- `xrt_shared[]` 与 export table 使用同一套 shared index 约定。

---

### WP-03：统一异常 lowering 与运行时语义

#### 目标

让 `try/catch` 在 AOT 下的行为与 VM 一致，并建立单一异常传播模型。

#### 现状 / 根因

当前异常路径存在明显不一致：

- `xcgen_expr.c` 中：
  - `XIR_TRY_BEGIN` / `XIR_TRY_END` 基本无实质 lowering
  - `XIR_CATCH` 只是读取 `xrt_exception`
  - `XIR_THROW` 仍可能直接 `abort()`
- `xcgen_call.c` 中对 `xr_jit_throw` 做了 runtime throw 映射，但不是整套统一模型。

这导致 AOT 只是“部分路径能跑”，而不是语义完整。

#### 涉及文件

- `src/aot/xcgen_expr.c`
- `src/aot/xcgen_stmt.c`
- `src/aot/xcgen_call.c`
- `src/aot/xrt_exception.h`

#### 实施步骤

1. 统一异常模型：
   - `TRY_BEGIN`：push 异常 frame
   - `TRY_END`：pop 异常 frame
   - `XIR_THROW` / `xr_jit_throw`：统一进入 runtime throw 路径
   - `XIR_CATCH`：读取当前异常对象并绑定到目标寄存器
2. 移除“正常路径上的 `abort()` throw fallback`”：
   - 仅允许它作为真正不可达/内部错误保护，不允许用于支持中的异常语义。
3. 明确同函数与跨函数 throw 的一致行为：
   - 不能一部分靠 goto，一部分靠 longjmp，一部分直接终止进程。
4. 与 `xcgen_stmt.c` 协同：
   - 确认 return、goto、phi 拷贝与异常 frame push/pop 顺序正确。
5. 补齐异常回归矩阵。

#### 新增 / 更新测试

- `tests/aot/basic/try_catch.xr`
- 建议新增：
  - `tests/aot/exceptions/throw_cross_func.xr`
  - `tests/aot/exceptions/nested_try.xr`
  - `tests/aot/exceptions/catch_string.xr`

#### 完成标准

- `try_catch.xr` 输出与 VM 完全一致。
- 不再存在“异常语义依赖 `abort()`”的正常执行路径。
- 同函数 / 跨函数异常都有稳定回归用例。

---

### WP-04：unsupported IR 改为 hard fail，并审计 no-op 白名单

#### 目标

禁止静默生成错误 C，任何未支持或语义不明的 IR 都必须显式失败。

#### 现状 / 根因

当前 `xcgen_expr.c` 里仍存在：

- `default:` 生成 `/* TODO: op ... */`
- `XIR_GUARD_*` / `XIR_DEOPT` / `XIR_RETAIN` / `XIR_RELEASE` / `XIR_BARRIER_*` 等直接 no-op

这会让 AOT 构建“表面成功、语义错误”。

#### 涉及文件

- `src/aot/xcgen_expr.c`
- `src/aot/xcgen.c`
- `src/app/cli/xcmd_build.c`

#### 实施步骤

1. 在 AOT 编译上下文中引入统一错误状态：
   - compilation 级或 module 级都可以，但必须能向 CLI 传播错误。
2. `default:` 改成诊断并终止当前 native 构建：
   - 错误信息至少包含：op、函数、模块路径。
3. 对当前 no-op 类别建立白名单：
   - 只有能证明“进入 AOT 时一定已被消除”或“对当前子集是安全空操作”的指令才允许保留。
4. 所有“不确定是否安全”的 no-op 先改为 hard fail。
5. 在 CLI 层给出明确报错，不再继续输出可执行文件。

#### 新增 / 更新测试

- 至少新增 1 个“当前不支持特性触发 hard fail”的 case。
- AOT 对拍脚本需要识别“预期失败”用例。

#### 完成标准

- 遇到未支持 IR 时，`--native` 明确失败。
- 错误信息足够定位到具体文件与函数。
- 不再通过 TODO 注释掩盖缺失 lowering。

---

### WP-05：把 `--native` 从“单入口 AOT”升级到“遍历 bundle 模块”

#### 目标

让 native 构建真正处理 bundle 中的所有模块，而不是只编译入口文件。

#### 现状 / 根因

当前 `cmd_build_native()` 的实际行为是：

- Phase 1：bundle 能收集模块列表；
- Phase 2：AOT 编译阶段实际上只读取并编译入口 `input`；
- 结果是 `bc_source` 只是生成过，但没有成为 AOT 多模块编译的输入。

#### 涉及文件

- `src/app/cli/xcmd_build.c`
- 必要时复查 `src/module/xbundle.*`
- `src/aot/xcgen.c`

#### 实施步骤

1. 将 bundle entries 作为 native 构建的权威模块列表。
2. 对每个模块执行统一流程：
   - 读取源文件
   - 解析/编译出 `XrProto`
   - 创建对应 `XcgenModule`
   - 收集导出与共享信息
3. 确保编译顺序与初始化顺序都基于 topo-sorted bundle 顺序。
4. 在编译任何函数之前，先全量注册 proto map：
   - 避免跨模块函数引用时只看得到当前模块。
5. 区分“入口模块”和“普通模块”：
   - entry 负责最终 `main()` 或顶层入口调用；
   - 普通模块负责编译和 init，不直接生成程序入口。

#### 新增 / 更新测试

- 新增真实 import 样例：
  - `main -> math_lib`
  - `main -> constants`
  - `main -> util -> math_lib`

#### 完成标准

- native 构建日志中能看到多个模块进入 AOT 流程。
- 至少一个双模块 import 用例通过。
- bundle 收集结果不再只是用于打印或临时 bytecode 源生成。

---

### WP-06：接通 module init / export / import 的执行闭环

#### 目标

让 `xcgen_emit_source()` 生成的 module 表真正参与运行。

#### 现状 / 根因

当前虽已生成：

- `xrt_shared[]`
- module export tables
- `xrt_modules[]`

但 standalone `main()` 仍然只调用入口模块初始化函数，未使用 `xrt_modules_init()`，也没有让 import 侧真正消费导出表。

#### 涉及文件

- `src/aot/xcgen.c`
- `src/aot/xrt_module.h`
- `src/app/cli/xcmd_build.c`

#### 实施步骤

1. 为每个模块发射明确的 init 函数指针并填入 `XrtModule`。
2. 调整 standalone `main()`：
   - 初始化 AOT runtime（如 ARC/bump/exception 上下文）
   - 调用 `xrt_modules_init(xrt_modules, nmodules, ctx)`
   - 再进入入口模块的执行逻辑
3. 设计 import lowering：
   - typed imported function：优先 `extern + direct C call`
   - imported const / let / dynamic symbol：`xrt_module_lookup()`
4. 保证 module export naming、shared index、import resolver 三者一致。
5. 清理“表生成了但没人用”的中间状态。

#### 新增 / 更新测试

- `tests/aot/modules/main_import_function.xr`
- `tests/aot/modules/main_import_const.xr`
- `tests/aot/modules/transitive_import.xr`

#### 完成标准

- 生成程序实际走 `xrt_modules_init()`。
- import/export 在运行期形成闭环。
- 多模块 case 不依赖手工串接或入口文件特例。

---

### WP-07：收口 closure / upvalue 的 escaping 与 non-escaping 两条路径

#### 目标

让 closure 在 AOT 下有清晰、可测、可解释的两条实现路径：

- non-escaping closure：不分配或轻量表示
- escaping closure：有显式 env / runtime closure 分配与访问

#### 现状 / 根因

当前已有部分 closure 相关逻辑，但 shared/global/跨函数组合时仍不稳定：

- non-escaping 路径依赖 prescan 结果
- `LOAD_UPVAL/STORE_UPVAL` 与 closure creation 的约定需要继续对齐
- shared closure 与 direct call 的交界目前是出错热点之一

#### 涉及文件

- `src/aot/xcgen_call.c`
- `src/aot/xcgen_expr.c`
- `src/aot/xcgen.c`

#### 实施步骤

1. 文档化 closure 两条路径的准入条件。
2. 审计 non-escaping closure 优化：
   - 只有在能够证明不逃逸时才跳过分配。
3. 审计 escaping closure 的 env 布局与访问规则：
   - `LOAD_UPVAL/STORE_UPVAL` 必须和 runtime closure 表示一致。
4. 检查 shared / import / return / nested function 这些典型逃逸点是否都会触发 escaping 路径。
5. 增加组合用例。

#### 新增 / 更新测试

- `tests/aot/basic/closures_basic.xr`
- `tests/aot/basic/closures_escape.xr`
- `tests/aot/basic/closures_shared.xr`

#### 完成标准

- closure 行为在 non-escaping 与 escaping 两类场景下都可预测。
- shared/global/return/nested 场景不会误走错误路径。

---

### WP-08：整改 AOT runtime 的内存分配封装

#### 目标

把 AOT runtime 从“直接调用 libc 分配函数”收口到单一封装层，减少规则冲突和后续切换成本。

#### 现状 / 根因

当前 `src/aot` runtime 头文件中仍广泛出现：

- `malloc`
- `calloc`
- `realloc`
- `free`

这与项目内存规则不一致，也让 OOM 处理和调试策略分散在多个文件里。

#### 涉及文件

- `src/aot/xrt_arc.h`
- `src/aot/xrt_class.h`
- `src/aot/xrt_coll.h`
- 可考虑新增：`src/aot/xrt_alloc.h`

#### 实施步骤

1. 增加统一分配封装层：
   - 第一阶段目标是“所有 runtime 分配都走同一组 API”；
   - 第二阶段再决定底层绑定到 `xr_malloc/xr_free` 还是其他实现。
2. 替换 runtime 头文件中的裸分配调用。
3. 统一 OOM 处理策略：
   - 失败时报错并终止；
   - 或统一返回空并在调用方处理；
   - 必须全 runtime 一致。
4. 为后续 ASAN/allocator instrumentation 预留单点入口。

#### 新增 / 更新测试

- 先依赖现有 AOT 回归 + ASAN 构建验证。
- 关键是新增 grep 级静态检查，确认 runtime 不再散落裸分配调用。

#### 完成标准

- `src/aot` runtime 头文件中不再散落裸 `malloc/calloc/realloc/free`。
- OOM 行为一致。
- 后续如果要切 allocator，实现只需改封装层。

---

### WP-09：明确并落实 AOT 生命周期模型

#### 目标

让 ARC / bump allocator / retain-release 之间的关系可解释、可实现、可验证。

#### 现状 / 根因

当前 runtime 已有：

- `xrt_arc_alloc`
- `xrt_arc_retain`
- `xrt_arc_release`
- `xrt_arc_release_val`

但生成端又把：

- `XIR_RETAIN`
- `XIR_RELEASE`
- `XIR_BARRIER_*`

当成 no-op。模型还没有闭环。

#### 涉及文件

- `src/aot/xcgen_expr.c`
- `src/aot/xrt_arc.h`
- `src/aot/xrt_class.h`
- `src/aot/xrt_coll.h`
- `src/aot/xcgen_struct.c`

#### 实施步骤

1. 明确哪些对象必须受 ARC 管理：
   - ARC string
   - class instance
   - collection object
   - promoted struct 中的 tagged/ptr 字段
2. 明确 `XIR_RETAIN/RELEASE` 的策略：
   - 如果 AOT 前端保证这些指令在进入 cgen 前已被消除，需要文档化并在断言里体现；
   - 否则就必须给出真实 lowering。
3. 在程序生命周期里明确调用：
   - `xrt_arc_init()`
   - bump arena destroy（若启用）
4. 确认 module init / main / exit 时的释放边界，避免“全靠进程退出回收”成为隐式默认。
5. 用 ASAN 或同类检查验证关键对象路径。

#### 新增 / 更新测试

- closure / collection / class / string 组合场景
- 优先选会创建较多堆对象的 AOT 用例做内存检查

#### 完成标准

- 能用一句话准确描述 AOT 对象生命周期模型。
- 实现与文档一致，不再同时存在“两套半语义”。

---

### WP-10：收口 class/object 与 struct promotion 的边界模型

#### 目标

减少 class/property/struct promotion 的偏移猜测和多模型并存问题。

#### 现状 / 根因

当前 AOT 已能处理部分 class/field 场景，但仍存在以下风险：

- class field access 仍依赖静态类型推断、偏移公式和部分 fallback
- promoted struct 与 class object 都有自己的布局假设
- struct promotion 仍存在上限值和部分 heuristic 推断路径

#### 涉及文件

- `src/aot/xcgen_expr.c`
- `src/aot/xcgen_struct.c`
- `src/aot/xcgen_struct.h`
- `src/aot/xrt_class.h`
- 必要时复查 `src/jit/xir_builder_object.c`

#### 实施步骤

1. 文档化 AOT 下 class field layout 约定，并确保 getter/setter 使用同一公式。
2. 明确哪些 property access 允许静态解析，哪些必须走 runtime，哪些直接判 unsupported。
3. 对 struct promotion 加强超限诊断：
   - 结构体数量超限
   - 字段数超限
   - 类型无法稳定推断
4. 优先使用更强的类型信息，而不是继续扩大基于 offset 的猜测。
5. 增加 class + struct 组合用例。

#### 新增 / 更新测试

- `tests/aot/basic/class_methods.xr`
- 新增：
  - `tests/aot/classes/instance_fields.xr`
  - `tests/aot/classes/dynamic_prop_fail.xr`
  - `tests/aot/structs/promoted_param.xr`

#### 完成标准

- class field / promoted struct 行为可预测。
- 超限和歧义不再 silent fallback。
- property access 的支持边界有文档和测试共同定义。

---

## 九、统一测试与验证策略

### 9.1 日常开发验证

每次代码修改后，至少执行快速验证：

```sh
cmake --build build -j8
(cd build && ctest --output-on-failure)
```

如果是集中调试 AOT 子系统，建议额外执行：

```sh
./build/xray build --native -c -o /tmp/aot_case.c tests/aot/basic/shared_vars.xr
./build/xray tests/aot/basic/shared_vars.xr
./build/xray build --native -o /tmp/aot_case tests/aot/basic/shared_vars.xr && /tmp/aot_case
```

### 9.2 里程碑关闭验证

每完成一个里程碑，至少执行：

```sh
scripts/run_regression_tests.sh
```

如需内存问题定位，可结合 AddressSanitizer 构建。

### 9.3 AOT 专项测试矩阵

建议把 AOT 测试逐步扩展为以下目录结构：

```text
tests/aot/
  basic/
    arithmetic.xr
    control_flow.xr
    shared_vars.xr
    closures_basic.xr
    closures_escape.xr
  modules/
    main_import_function.xr
    main_import_const.xr
    transitive_import.xr
  exceptions/
    try_catch.xr
    throw_cross_func.xr
    nested_try.xr
  classes/
    class_methods.xr
    instance_fields.xr
  structs/
    promoted_param.xr
```

### 9.4 测试结果要求

- 支持范围内的 case：**AOT 输出必须与 VM 完全一致**
- 未支持范围的 case：**AOT 必须明确失败，并给出诊断信息**
- 不允许存在“能编译、能运行、但输出 silently 错误”的状态

---

## 十、推荐实施顺序

建议按以下顺序推进：

1. `WP-01`：先建立对拍与失败诊断基线
2. `WP-02`：修 `shared_vars` 与 shared closure 调用
3. `WP-03`：统一异常 lowering
4. `WP-04`：unsupported IR 改为 hard fail
5. `WP-05`：native 遍历 bundle 模块
6. `WP-06`：接通 module init / import/export
7. `WP-07`：补 closure / upvalue 组合路径
8. `WP-08`：收口 runtime 分配封装
9. `WP-09`：落实生命周期模型
10. `WP-10`：收口 class/object/struct 边界

这个顺序的原则是：

- 先解决 **已知错误结果**
- 再接通 **结构性缺口**
- 最后收口 **工程债与模型一致性**

---

## 十一、风险清单与规避措施

### 11.1 风险：一边修 bug，一边继续扩大支持范围

**问题：** 容易造成“看起来支持更多，但回归不稳定”。

**规避：** 先做 `WP-01` 到 `WP-04`，把现有错误路径收口后再推进多模块。

### 11.2 风险：多模块实现和 shared 修复相互缠绕

**问题：** 很多跨模块调用本质上复用了 shared/export 机制。

**规避：** 先在单入口内把 shared function 语义修好，再做跨模块 import/export。

### 11.3 风险：异常模型半改半留

**问题：** 一部分用 goto，一部分用 longjmp，一部分 `abort()`，最终最难维护。

**规避：** 在 `WP-03` 开始前先明确异常策略，不允许中间态长期存在。

### 11.4 风险：runtime 分配整改导致 standalone 假设变化

**问题：** 如果直接把 runtime 改到依赖别的 allocator，可能破坏自包含目标。

**规避：** 先增加封装层，再决定封装层的底层实现，不要直接在业务代码里散改。

### 11.5 风险：retain/release 语义不明确导致“测试都过了但模型仍错”

**问题：** 当前很多对象路径可能只是被短命程序掩盖了泄漏或生命周期错误。

**规避：** `WP-09` 必须配合内存检查，不能只靠 stdout 相等判断“已经正确”。

---

## 十二、Definition of Done（DoD）清单

在本轮工作结束前，应逐项确认以下事项：

- `shared_vars.xr` 已修复并进入稳定回归。
- `try_catch.xr` 已修复并进入稳定回归。
- unsupported IR 会明确报错，不再静默 TODO。
- `cmd_build_native()` 不再只 AOT 编译入口文件。
- `xrt_modules[]` / `xrt_modules_init()` 已进入真实执行路径。
- 至少 2 个真实 import 用例通过。
- closure / upvalue 至少有 basic + escape 两类测试。
- runtime 分配走统一封装层。
- AOT 生命周期模型有文档且实现一致。
- 快速测试与完整回归都能通过。

---

## 十三、建议的首批任务拆分（可直接建 issue）

### Task 01

- **标题：** 建立 AOT VM/AOT 对拍脚本并接入 `tests/aot`
- **对应工作包：** `WP-01`
- **交付物：** 脚本、失败诊断输出、基础 case 列表

### Task 02

- **标题：** 修复 `shared_vars.xr` 的 shared closure lowering
- **对应工作包：** `WP-02`
- **交付物：** 修复后的 `xcgen_call.c` 路径与回归用例

### Task 03

- **标题：** 统一 `try/catch` 异常 runtime 与 lowering
- **对应工作包：** `WP-03`
- **交付物：** `XIR_THROW` / `xr_jit_throw` / `TRY_*` 的统一行为

### Task 04

- **标题：** unsupported IR hard fail 与 no-op 白名单审计
- **对应工作包：** `WP-04`
- **交付物：** 明确错误传播链与至少一个预期失败 case

### Task 05

- **标题：** `cmd_build_native()` 遍历 bundle 模块并建立真实多模块 AOT
- **对应工作包：** `WP-05` + `WP-06`
- **交付物：** 至少 1 个双模块 import case 跑通

### Task 06

- **标题：** 收口 runtime 分配封装与生命周期模型
- **对应工作包：** `WP-08` + `WP-09`
- **交付物：** 分配封装层、生命周期说明、ASAN 关键 case 验证

---

## 十四、结语

`src/aot` 当前最需要的不是继续“堆更多支持项”，而是把已有骨架收口成可验证、可维护、可解释的最小闭环：

- shared 要对
- 异常要对
- unsupported 要明确失败
- 多模块要真正接线
- runtime 要逐步摆脱例外状态

只要按本文的顺序推进，`src/aot` 可以在不大幅重写架构的前提下，从“局部可运行”收敛到“可宣称为 AOT Alpha”。
