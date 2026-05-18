# 架构决策记录 (ADR)

重要的架构决策及其上下文和理由。格式：决策 → 上下文 → 理由 → 后果。

---

## ADR-001: XrValue 使用 16 字节 tagged union + heap_type 缓存

**决策**：XrValue 采用 16 字节结构体（8 字节 descriptor + 8 字节 payload），在 descriptor 中缓存 `heap_type`。

**上下文**：需要在不解引用 GC header 的情况下判断堆对象类型，避免 cache miss。

**理由**：
- `xr_value_is_instance()` 等类型检查只需读 descriptor，无需解引用指针
- 比 NaN-boxing 更灵活，支持完整 64 位整数

**后果**：
- 所有 XrValue 构造必须正确设置 `heap_type`（见 Pattern 1）
- 手动构造 XrValue 时极易遗漏 `heap_type`

---

## ADR-002: JIT 通过 C runtime helper 实现复杂操作

**决策**：JIT 不内联复杂操作（如方法调用、断言），而是通过 `CALL_C` 调用 C runtime helper。

**上下文**：ARM64 JIT codegen 复杂度管理，避免在汇编层实现完整的 VM 语义。

**理由**：
- 降低 codegen 复杂度和 bug 风险
- 复杂操作（INVOKE_DIRECT、ASSERT_EQ）的语义由 C 代码保证正确性
- 便于调试（C 代码可设断点）

**后果**：
- JIT ↔ C 边界存在值转换开销（raw int64 ↔ XrValue）
- 边界处的 XrValue 重建必须正确（见 `jit_value_from_tag`）
- C helper 中调用 `xr_vm_call_closure` 会递归进入 VM 解释器

---

## ADR-003: GC safepoint 使用 guard page 机制（JIT）+ 轮询（解释器）

**决策**：JIT 代码使用 guard page safepoint（mprotect），解释器使用 flag 轮询。

**上下文**：JIT 代码无法在任意位置插入轮询检查，需要低开销的 safepoint 机制。

**理由**：
- Guard page 方式对 JIT 热路径零开销（仅在 GC 触发时 mprotect）
- 解释器已有自然的 safepoint（循环回边、函数入口）

**后果**：
- JIT crash handler 必须正确区分 guard page fault 和普通 SIGSEGV（见 Pattern 6）
- 从 JIT 调用 VM 解释器时，信号处理逻辑需要特殊考虑

---

---

## ADR-004: Upvalue 采用 Cell 方案而非 V8 式 Context chain

**决策**：闭包捕获使用 const 快照 + Cell（mutable）方案，而非 V8 的 Context chain。

**上下文**：需要为闭包变量捕获选择实现策略，同时考虑 JIT/AOT 友好性。

**理由**：
- xray 是静态类型语言，有 const/let 区分，编译期完全确定作用域
- const 捕获零分配（直接拷贝 XrValue 到 upvals 数组）
- Cell 布局固定简单（GC header + value），JIT 直接 load/store
- JIT 可将 const upvalue 内联为立即数
- 没有 JavaScript 的 eval/with/动态作用域需求

**后果**：
- let 变量多一次间接（2 次 vs Context chain 的 1 次），但只影响 ~30% 的捕获场景
- 编译器 Phase 1 必须为所有 upval 正确设置 type_info（见 Pattern 3）

---

## ADR-005: VTAG_NULL 精确标记 null 值

**决策**：在 JIT 内部引入 `VTAG_NULL=6`，null 值通过精确 VTAG 携带，不依赖 raw=0 启发式。

**上下文**：`jit_value_from_tag` 的 UNKNOWN 路径用 `raw==0 → NULL` 启发式，但 int(0) 也是 raw=0。

**理由**：
- bitmap 编码 nibble=0 对应 XR_TAG_NULL，精确无歧义
- 消除 int(0)/null 混淆，不需要启发式判断
- 所有 VTAG 枚举可直接映射到 XrValueTag

**后果**：
- `OP_LOADNULL` 必须使用 VTAG_NULL 而非 VTAG_TAGGED
- UNKNOWN+raw==0 安全地解释为 I64(0)

---

## ADR-006: 自适应并行工作窃取（yield_streak 压力检测）

**决策**：通过 `yield_streak` 计数器检测计算密集型工作负载，自动启动 worker 线程。

**上下文**：需要区分计算密集型（Parallel Sum）和 IO 密集型（Fanout）协程调度模式。

**理由**：
- 计算密集协程反复 yield 回 run queue → streak 增长 → 触发 workers
- IO 密集协程快速 block（channel recv）→ streak 重置 → workers 不启动
- SPAWN_CONT handler 中重置 streak，避免 spawn 阶段的 yield 计入

**后果**：
- Parallel Sum 获得 5x 加速，其他基准零退化
- streak 阈值 = worker_count，简单且自适应

---

## ADR-007: 类型系统 Single Source of Truth（inst_types 替代 slot_types）

**决策**：废弃 `slot_types[]`（per-register 静态类型），改用 `inst_types[]`（per-PC 流敏感类型）+ `param_types[]`（per-parameter 不变量）。

**上下文**：`slot_types[reg]` 是寄存器级别的类型，同一寄存器在不同 PC 处类型可能不同，slot_types 只能记录最后一次赋值的类型，不够精确。

**理由**：
- `inst_types[pc]` 在每条指令处记录精确类型，天然 flow-sensitive
- `param_types[i]` 覆盖函数参数（不变），无需 per-PC 查找
- 两个来源足以覆盖所有 JIT/AOT 类型查询需求
- 消除 slot_types 的不确定性和维护负担（24 处引用全部删除）

**后果**：
- JIT builder 使用 `builder_inst_xrtype(b, pc)` 查询
- CHA devirt fallback 改用 `builder_find_reg_type()`（反向扫描 inst_types）

---

## ADR-008: 省略返回类型默认 void

**决策**：函数/方法声明省略返回类型时默认为 void，而非报错。

**上下文**：之前省略返回类型会报 "missing return type annotation" 错误。

**理由**：
- 符合直觉：没有 return 值就是 void
- 主流惯例：Kotlin、Swift、Rust、Go 都是省略 = 无返回值
- 安全网：如果函数体有 `return expr` 但没有返回类型注解 → 报编译错误

**后果**：
- `xa_body_has_return_expr()` 辅助函数：遍历 AST 但不递归进嵌套函数/lambda
- getter/setter 方法保持自动推导

---

## ADR-009: VTAG_CALLEE_SETS 消除

**决策**：从 `XrVRegTag` 中移除 `VTAG_CALLEE_SETS`，CALL_DIRECT 结果退化为 `VTAG_TAGGED`。

**上下文**：VTAG_CALLEE_SETS 表示"被调函数通过 RET epilogue 写入了 tag，从 slot_runtime_tags 读取"，但 VTAG_TAGGED 已能传达"编译期类型未知"语义。

**理由**：
- JIT helper 的 UNKNOWN 路径已通过 `val_bc_slot != 0xFF` 检查 slot_runtime_tags
- CALL_DIRECT 结果标记为 VTAG_TAGGED 后，自然走 UNKNOWN 动态 tag 解析路径
- XrVRegTag 从 7 值精简为 6 值，减少代码复杂度

**后果**：
- RET epilogue 统一输出静态 0xFF tag
- 9 个文件改动，净减 39 行

---

## ADR-010: netpoll 抽象层 + io_uring 后端

**决策**：将 netpoll 改为函数指针表（`XrNetpollOps`）+ 后端插件架构，新增 io_uring 后端。

**上下文**：原来 kqueue/epoll 代码直接嵌入 xnetpoll.c，添加新后端需要大量 `#ifdef`。

**理由**：
- 函数指针表分发：公共 API 通过 `ops->xxx()` 调用，共享逻辑集中管理
- 运行期自动检测：Linux 上 io_uring init 失败自动回退 epoll
- 新后端只需实现 `XrNetpollOps` 表，不影响其他后端代码

**后果**：
- macOS: kqueue（唯一选项）
- Linux + liburing: io_uring 优先，失败回退 epoll
- Linux 无 liburing: 编译时选 epoll

---

## ADR-011: AOT lowering 复用 JIT XIR builder，`--native` 与 `XRAY_HAS_JIT` 共编译开关

**决策**：AOT (`xray build --native`) 的 frontend → IR lowering 不另起炉灶，直接复用 `src/jit/xir_builder*.c`。因此 `cmd_build_native()` 当前在 `XRAY_HAS_JIT=0` 时整体禁用，编译开关二者**故意**绑在一起。

**上下文**：审视 AOT 实现状态时，自然问题是 “AOT 是否独立于 JIT”。代码现状：`xcgen_*` 只消费 `XirFunc/XirIns`，`XirFunc` 由 `xir_build_from_proto_aot_ex()` 构造，该函数定义在 JIT 模块。如果硬拆，意味着复制一份 “只发 AOT-friendly opcode 子集” 的 builder，长期维护两套 lowering。

**理由**：
- IR builder 复杂度高，发出的 XIR 既要喂 JIT 也要喂 AOT；同源保证两后端语义一致。
- AOT 与 JIT 的差异由 `aot_mode` 标志精细控制（少量分支），不构成独立模块的强动机。
- 真正能解耦的边界在更下层（XIR → C / asm 后端），那一层已经分得清：`src/aot/xcgen_*` ↔ `src/jit/xir_codegen_*`。
- “生成产物不依赖 libxray_core” 与 “编译期是否依赖 JIT 子系统” 是两个独立问题；ADR-011 只承认后者。

**后果**：
- `XRAY_HAS_JIT=0` 同时关闭 JIT 与 AOT，是设计选择，不视为 bug；CLI 层在这种模式下会明确报错（`Error: --native requires JIT support`）。
- 若未来真要拆，目标应是 “XIR lowering 独立成 subsystem（如 `src/lower/`）”，而不是把它复制进 AOT 目录。
- AOT refactor plan 的 Phase 1.5 正式记录此决策，避免后续误把它列为 “未完成项”。

---

（持续补充）
