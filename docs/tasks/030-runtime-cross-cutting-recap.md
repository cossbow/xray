# Runtime 横切复盘（`030`）

> 本文汇总 `024` 到 `029` 六轮 runtime 分析，目标不是重复逐层结论，而是给出跨模块、跨 owner、跨 heap domain 的统一解释。
>
> 如果只按目录读 `src/runtime/`，很容易误以为项目已经有一套单一的“GC object 模型”。真正的源码结论并不是这样：runtime 更像是多个内存协议并存，而 `XrGCHeader` 只是其中一条常见外观，不是单一真相源。

## 1. 最终总论

- **Xray runtime 不是单一 GC 模型，而是四种 owner 机制并存。**

- **“是否带 `XrGCHeader`”不能推出“由哪套 owner 负责回收”。**

- **真正稳定的判断标准不是类型名，也不是注释，而是：**
  - 分配入口
  - 销毁入口
  - traversal / barrier / deep-copy 是否闭合
  - 是否允许跨 coroutine 直接共享

- **当前 runtime 的核心张力来自“统一外观”与“分裂 owner”同时存在。**
  - 外观看起来像一套统一 object system
  - 实际上 owner 分别落在 per-coro GC、isolate fixedgc、system-heap shared/refcount、malloc graph

## 2. 四类 owner / heap domain

### 2.1 per-coro GC（`XrCoroGC`）

代表入口：

- `xr_alloc(coro, size, type)`
- `xr_coro_gc_newobj(gc, type, size)`

代表特征：

- coroutine 私有 Immix heap
- tri-color + barrier + incremental/generational
- coroutine 结束时整堆销毁或 reset
- shared object 不参与 tri-color，只做 ref tracking

代表对象：

- array / map / set / closure / json / task 的常规路径
- deep-copy 到目标 coroutine 时生成的新对象
- 一部分 instance / datetime / iterator / module value graph

### 2.2 isolate fixedgc（`XrGC`）

代表入口：

- `xr_gc_alloc(&isolate->gc, size, type)`
- `xr_alloc()` 在 coro_gc 不可用时 fallback 到 fixedgc

代表特征：

- 本质是 malloc + fixedgc linked list
- 不做 mark-sweep
- isolate cleanup 时统一释放
- 主要承担 bootstrap/early-init/fallback 语义

代表对象：

- early-init/fallback 分配对象
- normal instance 当前实际常走这里
- 某些 deep-copy fallback 目标

### 2.3 system-heap shared/refcount

代表入口：

- `xr_sysheap_alloc_shared(...)`
- `xr_shared_incref/decref/destroy`

代表特征：

- `XR_GC_STORAGE_SHARED`
- `gc_next` 被复用为 refcount 槽位
- 不进 GC 链表
- per-coro GC 只记录 shared refs，不做 tri-color 标记

代表对象：

- channel
- shared array/map/set/instance/json/stringbuilder/closure/datetime copy
- shared strings / global pool strings（其中 global pool 又是更特化的一支）

### 2.4 plain malloc / arena / pool graph

代表入口：

- `xr_malloc/xr_calloc/xr_realloc`
- `xr_sysheap_alloc_class`
- coroutine pool / worker state / runtime state / symbol table / reflect wrapper

代表特征：

- 完全不属于 coro GC 或 fixedgc 的常规生命周期
- 依赖显式 destroy/free 或 isolate shutdown bulk free
- 可能带 `XrGCHeader`，但 header 只是 type/storage/ABI 外观

代表对象：

- class metadata 及其 side tables
- symbol table
- reflection wrappers/cache
- shape registry
- exception
- runtime / worker / machine / timer / blocked queue
- coroutine executor shell

## 3. `XrGCHeader` 不是 owner 真相源

这六轮最大的统一结论是：

- **header 是必要信息，但不是充分信息。**

从源码能确认至少有以下几类“带 header，但 owner 不同”的对象：

### 3.1 真正的 per-coro GC object

- 由 `xr_alloc` / `xr_coro_gc_newobj` 分配
- 参与 tri-color
- 需要 barrier/traverse/finalize 闭合

### 3.2 fixedgc object

- 由 `xr_gc_alloc` 分配
- 挂在 `fixedgc` 链表上
- cleanup 时统一 destroy + free
- 不参与 per-coro GC 周期

### 3.3 shared object

- 由 `xr_sysheap_alloc_shared` 分配
- header 上带 shared storage
- `gc_next` 变成 refcount 槽
- per-coro GC 只记录引用，不会 tri-color mark

### 3.4 pseudo-GC / protocol object

- 有 header，便于复用统一 type tagging / ABI / value representation
- 但真实 owner 是 malloc/arena/service

典型例子：

- reflection wrappers
- exception
- shape 相关对象里的部分 header 语义
- class/enum 混合对象
- coroutine executor shell

因此跨层文档里若出现“某对象是 GC object”，必须继续追问：

- 哪个 GC？
- 真参与 mark/sweep 吗？
- 还是只是 header-compatible？

## 4. 统一的边界判断法

跨 phase 之后，可以把 owner 判断收敛成四个问题：

### 4.1 分配入口是谁？

- `xr_coro_gc_newobj` / `xr_alloc` → per-coro / fixedgc fallback
- `xr_gc_alloc` → fixedgc
- `xr_sysheap_alloc_shared` → shared
- `xr_sysheap_alloc_class` / `xr_malloc` / `xr_calloc` → malloc/arena/pool

### 4.2 销毁入口是谁？

- `xr_coro_gc_destroy/reset` 路径
- `xr_gc_cleanup`
- `xr_shared_destroy`
- 显式 `xr_free` / arena destroy / runtime destroy

### 4.3 traversal 是否真的闭合？

如果 traversal 不闭合，即使对象带 header，也不能简单归为“统一 GC 模型”。

### 4.4 跨 coroutine 时是共享还是复制？

- shared → incref passthrough
- deep-copy → 新 owner 域
- non-deep / raw pointer → 协议层自己保证

## 5. 深拷贝才是真正的跨域边界

前几轮里很多“ownership confusion”最终都能落到同一个解释：

- **对象一旦跨 coroutine，就不再由声明位置决定 owner，而是由 deep-copy/shared 规则决定 owner。**

`xdeep_copy.c` 明确编码了这条边界：

- shared object：直接 `xr_shared_incref`
- deep-copy kind：复制到目标 coro_gc 或 fixedgc
- 专门的 to-shared 路径：复制到 shared sysheap

这带来两个重要后果。

### 5.1 object layer 的 owner 不是绝对属性，而是上下文属性

例如 instance：

- normal new path：fixedgc
- deep-copy to coro：dst coro_gc
- deep-copy to shared：sysheap shared

也就是说“instance 属于哪个堆”不是单句就能说完，必须附上下文。

### 5.2 coro 层不是单纯 scheduler，而是 owner 变换层

`src/coro/` 真正重要的不是 runq，而是：

- shared passthrough
- deep-copy target selection
- await/completion/wake 协议
- cross-worker routing

## 6. 注释模型与实现模型并不总一致

### 6.1 `instance` 是最明显的例子

实现上：

- `xr_instance_new()` 正常分配走 isolate fixedgc

但很多注释或直觉上容易把 instance 归到“普通 GC object / coroutine heap object”。

### 6.2 `XrCoroutine` 看起来像 GC object，实际上是 sysheap/pool executor shell

它本体通常来自：

- runtime pool
- sysheap coro pool
- malloc fallback

但内部又持有自己的 `XrCoroGC`。

因此：

- coroutine 外壳 owner ≠ coroutine 内部对象堆 owner

### 6.3 reflection wrappers 看起来像 instance/blob，实际上是 malloc-backed wrapper

它们借用了 header/type/value 表示，但没有进入统一 owner 模型。

### 6.4 exception traversal 存在，但分配并不走常规 per-coro path

这类对象最容易制造“它明明能 traverse，为何 owner 却不统一”的认知错位。

## 7. traversal/barrier 闭合度并不均匀

### 7.1 per-coro GC 的理论模型相当完整

`xcoro_gc.h/c` 能看出很成熟的局部体系：

- tri-color
- gray/grayagain/weak
- remembered set
- shared ref tracking
- large object list
- external root callback
- external memory accounting API

### 7.2 但跨 runtime 全局并没有完全闭合成“所有 header object 都纳入这套模型”

`xcoro_gc_traverse.c` 已经显式写出了很多特例：

- `XrClass` 通常在 system heap，instance traversal 不标它
- `XR_TCOROUTINE` 不遍历内部状态，它有自己的 `XrCoroGC`
- `XR_TCHANNEL` 是 shared object，不在这里 traverse 内部 buffer
- iterator/context/symbol table 等是 raw runtime/service pointer

这实际上是在告诉读者：

- traversal dispatcher 是“兼容多 owner 世界”的适配层
- 不是统一 object universe 的证明

### 7.3 barrier 也只对一部分对象族真正成立

barrier 前提是：

- parent/child 都处于 per-coro GC 语义内
- shared object 直接 early-return
- fixedgc / malloc / service object 很多时候只做 defensive traverse 或根本不适用

## 8. external memory accounting 是局部有效，不是全局闭合

`xcoro_gc.h` 给出了：

- `xr_gc_add_external`
- `xr_gc_sub_external`

但实际搜下来，明显闭合的代表主要是：

- `xarray.c` 对外部 data buffer 的 accounting

而 map/set/json/stringbuilder/exception/reflect/class side table 等大量 malloc-backed 子结构，并没有统一进入一套完整 external accounting 体系。

因此当前 external memory accounting 的实际语义更接近：

- **局部补丁式可见性增强**
- 而不是“所有外部内存都被 GC 感知”

这也解释了为什么某些对象图虽然看似不大，实际 malloc pressure 却可能远高于 `totalbytes` 直觉。

## 9. pseudo-GC object 是 runtime 里最稳定的风险源

跨六轮重复出现的一类对象是：

- header 在
- traversal/ABI/value conversion 有一部分在
- 但 owner 不在统一 GC 模型里

典型风险：

- 阅读者误判生命周期
- deep-copy / barrier / mark / finalizer 只覆盖一半
- 文档说成“GC object”，实现却靠 service owner 或 manual free

这一类对象包括但不限于：

- exception
- reflection wrappers
- class metadata 及 side tables
- shape / registry 服务对象
- 某些 runtime control objects

## 10. structured concurrency 说明 runtime 已经进入“对象 owner + 调度 owner 分离”阶段

从 task/coro/scope 三层能看出另一个横切趋势：

- executor (`XrCoroutine`) 不再等于 user-visible handle
- task (`XrTask`) 才是 await / completion / structured concurrency 的协议中心
- scope 又维护额外的 child list / policy state

这代表 runtime 设计已经从“一个 object = 一个 owner”演化到：

- 执行 owner
- 语义 owner
- 调度 owner
- heap owner

并不总是同一个。

这虽然提高了表达能力，但也明显提高了系统复杂度。

## 11. 统一风险清单

### 11.1 “带 header 就归 GC” 的误解风险

这是最根本的风险，会误导：

- 新增类型的分配决策
- traverse/destroy 实现位置
- 文档对 owner 的描述

### 11.2 fixedgc / per-coro / shared / malloc graph 交叉时容易出现边界漏项

典型漏项形式：

- traversal 漏掉 raw child
- destroy 漏掉 malloc side buffer
- external accounting 不对称
- deep-copy 漏类型
- shared refcount 与 global pool 特例冲突

### 11.3 注释长期漂移风险

代码注释和头文件说明很多时候写的是“期望模型”或“主路径模型”，但实现已经分化出：

- fallback path
- bootstrap path
- shared path
- pooled shell + owned subheap path

如果不持续回收这些叙述差异，文档和实现会越来越远。

### 11.4 目录分层不等于 owner 分层

例如：

- `class/` 不全是 class-heap
- `object/` 不全是 per-coro object
- `coro/` 不只是调度器
- `gc/` 也不是唯一 owner 解释处

## 12. 推荐的统一描述框架

以后若继续补 runtime 文档，建议每个类型都用同一模板描述：

### 12.1 Allocation domain

- 分配入口
- 主路径
- fallback 路径

### 12.2 Ownership authority

- 谁决定生命周期结束
- 谁负责 destroy/free
- 是否可跨 coroutine 直接共享

### 12.3 GC participation level

- fully managed
- fixed-list managed
- shared-refcounted
- header-only compatibility
- manual / service-owned

### 12.4 Cross-boundary protocol

- deep-copy
- incref passthrough
- raw pointer / no transfer

### 12.5 External memory / side tables

- 是否有 malloc side buffers
- 是否做 external accounting
- destroy 是否闭合

这会比单纯写“是否是 GC object”准确得多。

## 13. 建议的后续整治方向

### 13.1 给所有核心 runtime type 补一张 owner matrix

矩阵列建议：

- type
- alloc API
- storage domain
- destroy API
- traversed by
- deep-copy behavior
- shared?
- external memory?

### 13.2 明确把 pseudo-GC object 单列成一类

不要再混在“普通 runtime object”里讲。

### 13.3 把 external accounting 从局部策略提升为系统策略

至少明确：

- 哪些 side buffer 必须记账
- 哪些是故意不记
- 哪些未来应该补齐

### 13.4 把“注释模型 vs 实现模型”差异逐项消掉

尤其是：

- instance owner
- coroutine owner
- task / scope / executor 关系
- reflection / exception / class side tables 的真实 owner

## 14. 最终结论

如果只用一句话总结当前 runtime：

> **Xray runtime 采用的是“多 owner 协议共存”的对象系统，而不是单一 GC 宇宙。**

更具体一点：

- per-coro GC 负责高频、局部、可批量回收的对象图
- fixedgc 负责 bootstrap/fallback/固定生命周期对象
- shared/refcount 负责跨 coroutine 直接共享对象
- malloc/arena/pool graph 负责元数据、服务对象和控制平面
- deep-copy / shared passthrough 是它们之间的边界协议

因此后续任何 runtime 改造，如果目标是“统一模型”，真正要统一的不是 header 形状，而是：

- owner authority
- destroy path
- traversal closure
- cross-boundary protocol

---

## 15. 本轮状态

- **已完成**
  - 汇总 `024` 到 `029` 的统一 owner 模型
  - 确认四类 heap/owner domain
  - 确认 `XrGCHeader` 不是 owner 真相源
  - 收束 deep-copy/shared 为 runtime 统一边界协议
  - 收束 external accounting / pseudo-GC / 注释漂移 三类横切风险

- **建议下一步**
  - 若继续做工程化整理，优先产出 owner matrix
  - 若继续做代码整改，优先清 pseudo-GC object 的生命周期契约
