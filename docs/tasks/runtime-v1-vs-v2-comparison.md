# Runtime 文档 V1 vs V2 取舍报告

> 7 份 V2（`024-030`）已全部完成。本文是用户原计划"最后取长补短"的对比汇总。
>
> 目的：给出"V1 哪些保留、V2 哪些采纳、最终合并版怎么写"的可执行取舍建议。

## 1. 总体评估

| 维度 | V1 | V2 |
|---|---|---|
| **覆盖广度** | 6 大类目录概览 | 文件级别（57 + 50 + 35 + 36 + 17 + 6 + ~10）全核对 |
| **论断准确率** | 主要正确，少数夸大或不够精确 | 全部 grep/read 验证 |
| **完整性** | 漏掉若干子模块 | 补全缺漏 |
| **风险点定位** | 正确识别风险类型 | 加文件:行号 + 修复路径 |
| **改进建议** | 每章 4-6 条方向性 | 每章 10 条可执行 |
| **跨章汇总** | 030 V1 高质量总论 | 030 V2 30 条 P0/P1/P2 清单 + owner matrix |
| **写作风格** | 偏抽象/描述 | 偏具体/可执行 |

**结论**：V1 的"宏观判断"准确率高，作为 runtime 设计哲学的入门文档质量很好。V2 的"微观证据 + 修复路径"是 V1 的工程化补充。两者并不冲突，**可合并为单一最终版**。

## 2. 逐章取舍建议

### 2.1 `024` 控制面

**V1 保留**：

- `XrayIsolate` 是状态总装配点（§1）
- `xr_vm_current_ctx` 是权威入口（§2.4）
- `xerror.c` 已空、`init_common` 死声明（§2.7、§2.8）
- TLS isolate 在 value/ 层穿透（§5.5）
- module_registry 兜底恢复（§5.2）

**V2 采纳**：

- `XrGlobalsTable` 实际是 codegen-time global value lookup（V2 §3.5，纠正 V1"几乎闲置"）
- `XrVMState`/`XrVMContext` 是设计意图的 storage host vs access path（V2 §3.4，纠正 V1"还没收口"）
- `XrVMState` 14 字段全清单 + `XrVMContext` 含 tmp_strbuf/struct_ret_arena/preempt（V2 §2.2、§2.3）
- trace_execution 漂移精确链路（V2 §5.1）
- 10 条最佳设计建议（V2 §6）

**合并版结构**：保留 V1 §1-§3 整体框架；§2 各模块条目按 V2 字段全列；§5 风险点用 V2 精确定位；§6 改进建议用 V2 可执行版。

### 2.2 `025` value 层

**V1 保留**：

- value/ 是共享契约层（§1.1）
- `xopcode_info` / `xtype_feedback` / `xtype_opt_hint` 放置合理（§7.1-§7.3）
- `XrProto` 膨胀（§4.4）
- `xr_valuearray_add` 走 deep_eq 隐含 TLS（§6.6）

**V2 采纳**：

- 类型系统 3 层（V2 §3.4，纠正 V1"4 层"）
- file-scope mutable global 仅 2 个（V2 §3.5，纠正 V1"两类"）
- `g_type_*` 是 frozen singleton 不算可变（V2 §3.5）
- `proto_id` 风险温和（V2 §3.6，纠正 V1"夸大")
- `XrProto` 55 字段 12 组全列（V2 §3.7）
- `XrValue` 16-byte 设计基线（V2 §3.1）
- `xr_value_typeid` 双表 + X-macro 正面示范（V2 §3.2、§3.3）
- 10 条最佳设计建议（V2 §7）

**合并版结构**：保留 V1 §1-§4 模块识别框架；类型系统层次用 V2 §3.4 精确版；风险点列用 V2 §5；改进建议用 V2 §7。

### 2.3 `026` GC + closure

**V1 保留**：

- 三堆域并存（§1）
- closure flat upvals + cell（§4）
- bound method 走 fixedgc 靠 defensive traversal（§4.3）
- bytecode stackmap 是 live-prefix 不是 precise（§6.2）
- `g_gc_pool_*` 进程级 mutable global（§9.1）
- xgc.h 注释过时（§9.2）

**V2 采纳**：

- `xbc_stackmap.h:108` promise dedup vs `.c:103` no dedup 的注释漂移（V2 §6.2，V1 漏报）
- `g_destroy_funcs` 是 .rodata 编译期常量正面示范（V2 §3.6，V1 漏）
- `XrCell` `_Static_assert(32B)` 守卫（V2 §3.8）
- `bound_method_stub` silent null 是真实 TODO（V2 §3.7）
- `mark_struct_string_fields` 加 fail-fast assertion 建议（V2 §6.7）
- 10 条最佳设计建议（V2 §7）

**合并版结构**：V1 §1-§7 整体框架质量高，直接保留；加入 V2 §3.4 字段表 + §6.x 修复方案 + §7 P0/P1/P2 清单。

### 2.4 `027` object 层

**V1 保留**：

- 5 种生命周期模型并存（§1）
- 容器 header/backing owner 不一致（§3）
- `xstring` / `xshape` / `xexception` 跨层职责（§4）
- container-level back barrier（§5.1）
- external accounting 仅 Array 完整（§5.2）

**V2 采纳**：

- `XrException` 真实泄漏（V2 §3.5，V1 仅说 "leak risk"）
- `XR_ALLOCATE` 模式扩散到 reflection 4+ 类型（V2 §3.6，V1 仅识别 exception）
- `xr_gc_destroy_shape` 是 dead code（V2 §3.7，grep 验证零调用）
- `XR_SET_FLAG_ENTRIES_ON_GC` 永无 GC blob 阶段（V2 §3.3，纠正 V1"历史残留"）
- `g_destroy_funcs` + `has_refs` 双表合并方案（V2 §5.6）
- 10 条可执行建议（V2 §7）

**合并版结构**：V1 §1-§5 框架保留；§6 高风险点用 V2 §5 精确化版本（特别是泄漏定性 + 修复一行代码示意）。

### 2.5 `028` class + symbol

**V1 保留**：

- `class/` 不是普通 L3 元数据层（§1）
- `XrClass` 走 sysheap + fallback（§3.1）
- `XrInstance` 走 fixedgc，叙述偏差（§3.2）
- builtin symbol enum 是 ABI 协议（§4.1）
- enum 跨 fixedgc/class/malloc 三域（§3.7）

**V2 采纳**：

- `xreflect_cache.c:91` 注释 promise GC-managed vs 实际 `XR_ALLOCATE` 永不释放（V2 §3.3，V1 仅 "tension"）
- `xreflect_api.c` 4 类 wrapper + cache 2 类 wrapper + method.c 1 类 metadata 全列（V2 §3.4）
- `XR_TBLOB` 不在 destroy + has_refs 系统（V2 §3.4）
- enum side table 泄漏精确化（V2 §3.7）
- `xclass_descriptor` / `xclass_reflect.c` 漏列（V2 §5.7-§5.8）
- 10 条 P0/P1 建议（V2 §7）

**合并版结构**：V1 §1-§4 框架保留；§3 各模块条目按 V2 全列；§7 风险点用 V2 §5 精确版。

### 2.6 `029` coro 层

**V1 保留**：

- coro/ 是 cross-owner 控制层（§1）
- coroutine sysheap/pool 三层分配（§3.1）
- task per-coro Immix（§3.2）
- channel shared sysheap（§3.3）
- deep-copy 三目标域（§5）
- task tree + scope child list 双链表（§7）
- `xr_coro_wake_waiter` 责任过重（§10.3）
- cross-worker wake routing（§8）

**V2 采纳**：

- coro/ 实际 57 文件（V2 §2，V1 仅列 6 大类）
- `XrInstance` 三路径漂移精确化（V2 §3.5，与 028 V2 协同）
- `xtask.c:55-75` lifetime invariant 注释级文档化是好设计（V2 §3.2）
- `xworker.h` API 边界清晰，是好设计（V2 §3.8）
- deep-copy 不覆盖类型清单 + 跨 coro 协议建议（V2 §5.6）
- 10 条 P0/P1 建议（V2 §7）

**合并版结构**：V1 §1-§11 整体框架直接保留（V1 这章本身写得最详细）；补 V2 §2 完整文件清单；§5 cross-cutting 修复清单。

### 2.7 `030` 横切复盘

**V1 保留**：

- 四 owner 并存现状（§2）
- `XrGCHeader` 不是 owner 真相源（§3）
- 四问题统一边界判断法（§4）
- deep-copy 是真实跨域边界（§5）
- 注释 vs 实现漂移（§6）
- pseudo-GC 是稳定风险源（§9）
- structured concurrency owner 分离（§10）
- 统一描述模板（§12）

**V2 采纳**：

- 四 owner → 三 owner 收敛建议（V2 §2.2，V1 仅描述现状）
- 5 表跨章证据汇总（V2 §3）
- 24 个核心 type 填好 owner matrix（V2 §4）
- 30 条 P0/P1/P2 修复清单 + 来源（V2 §5）
- "5 处修改优先级"快速 win list（V2 §7）
- "fixedgc 退役"建议（V2 §2.2、§4.4）

**合并版结构**：030 是 V1+V2 收益最大的合并对象。建议用 V1 §1-§12 的设计哲学叙述 + V2 §3-§7 的工程化清单作为附录章节。最终版是"前半 V1 哲学，后半 V2 工程"双段式。

## 3. 推荐的最终合并方案

### 3.1 单文件合并 vs 双文件并列

两条路：

- **方案 A（单文件合并）**：每章生成一份"V3 最终版"，删 V1 和 V2。
  - 优点：单一真相源，未来维护一份。
  - 缺点：丢失 V1 的写作风格特点（V1 哲学叙述很有阅读价值）；V2 工程化清单和 V1 描述风格混杂会冗长。

- **方案 B（双文件并列保留 + 加 README 索引）**：V1 和 V2 都保留，README 标注"V1 = 设计哲学版，V2 = 工程化补充版"。
  - 优点：保留两份各自的写作风格；新读者按需读。
  - 缺点：单一信息可能在两处出现，未来变更要双写。

**V2 推荐方案 B**：原因如下：

- 对账过程中发现 V1 ~85% 论断仍准确，只是不够精确/全面。
- V1 的"叙述哲学"风格对设计决策讨论有独特价值，强行合并会稀释。
- V2 的"工程清单"风格对修复执行有独特价值，强行合并会冗长。
- 两者并列 + 索引，新读者按需读：先读 V1 入门，需要修复时读 V2。

### 3.2 README 索引建议

`docs/tasks/README.md` 加一节：

```markdown
## Runtime 模块文档（024-030）

每个 phase 有两份文档：

- `NNN-runtime-phaseX-...md`：**设计哲学版**，每个子系统的边界、ownership 模型、跨层关系总论。新读者推荐先读此版。
- `NNN-runtime-phaseX-...-v2.md`：**工程化补充版**，文件:行级证据、注释/实现漂移定位、可执行修复清单（P0/P1/P2）。需要落地修复时读此版。

两份文档并行，不互替代。

`030-runtime-cross-cutting-recap-v2.md` §5 含 30 条 P0/P1/P2 修复清单，按优先级实施。
```

## 4. 修复实施路径

如果用户决定开始按 V2 修复，推荐的顺序：

1. **第一波（P0，5 处快速 win）**（030 V2 §7）：
   - `xr_instance_new` 改 per-coro
   - `xr_exception_alloc` 改 per-coro
   - `xr_reflect_cache_free` 加 wrapper 释放
   - `g_gc_pool_*` 迁 sysheap
   - `xreflect_cache.c:91` 注释 + 实现同步

2. **第二波（P0 剩余 + P1 核心）**：
   - reflection wrapper 4 类（api.c）
   - `xr_runtime_wake_channel*` cross-worker（与 audit plan CORO-01 协同）
   - `xgc.h` 头注释重写
   - `xbc_stackmap` dedup
   - `g_destroy_funcs[XR_TENUM_TYPE]` 加 destructor

3. **第三波（P1 剩余 + P2）**：按 030 V2 §5 顺序逐项。

每波修复后必须跑 `cd build && ctest --output-on-failure` 验证。

## 5. 总结

- 7 份 V2 全部完成，每份对应一份 V1，物理并列。
- V1 论断 ~85% 准确率，V2 主要补充：精确文件:行号、字段全清单、注释漂移定位、修复路径、P0/P1/P2 优先级。
- `030` 是 V2 收益最大的章节：从"现状描述"推到"四 owner → 三 owner 收敛建议 + 30 条修复清单"。
- 推荐双文件并列保留 + README 索引（方案 B）。
- 修复实施按 030 V2 §7 的"5 处快速 win" 起步。
