# Runtime `gc/ + closure/` 分析（`026`）

> 本轮聚焦 `src/runtime/gc/` 与 `src/runtime/closure/`。
>
> 这里不能只看 GC 算法本身。真正重要的是：
>
> - 对象到底分配在哪个堆域
> - closure/cell/bound method 分别归谁拥有
> - VM 栈、struct_area、JIT frame、shared object、task、module export 是如何接入 root scan 的
> - `slot_type` / `storage_mode` / `bc_stackmap` 哪些已经落到运行时，哪些还只是元数据

## 1. 本轮结论

- **当前 runtime 实际是“三种内存域并存”，不是单一 GC。**
  - `XrGC`：isolate 级 fixedgc，只管固定对象/早期 bootstrap fallback。
  - `XrCoroGC`：每协程一个 Immix mark-region 堆，负责绝大多数运行期对象。
  - `XrSystemHeap`：系统级对象池 / arena / shared refcount，对象不受 per-coro tri-color GC 管理。

- **closure 子系统本质是“flat upvalue snapshot + optional cell indirection”。**
  - `const` / 已确定按值捕获的局部：closure 直接快照 `XrValue`
  - `let` mutable captured local：先 cellify 成 `XrCell*`，closure 实际捕获的是 cell 引用
  - runtime `OP_CLOSURE` 只管复制，不做复杂 capture 解析

- **GC root scan 已经明显不是 L2 纯算法，而是运行时横切接线层。**
  - 它直接知道 VM frame、bytecode stackmap、JIT stack map、task、module export、shared array、channel send value、struct_area、cfunc result。

- **`UpvalInfo` 的多个字段不是等价地被运行时消费。**
  - `source`：VM `OP_CLOSURE` 真正使用
  - `storage_mode`：主要给 compiler 协程安全检查 / 序列化使用
  - `slot_type`：当前主要给 JIT/AOT/backend metadata 使用，不是 closure GC 的直接输入

- **`xbc_stackmap` 的“precise GC”表述偏乐观。**
  - 当前 builder 只是把 `[0, freereg)` 全部标成 live
  - 这比整帧 conservative scan 更好，但还不是精确到真实活跃指针槽位

## 2. 模块地图

| 子域 | 关键文件 | 真实职责 |
|---|---|---|
| fixed/global GC | `xgc.h` `xgc.c` `xgc_internal.h` | isolate 级 fixedgc、destroy function 表、bootstrap/fallback 分配 |
| per-coro GC 引擎 | `xcoro_gc.h` `xcoro_gc.c` `ximmix.*` | Immix 分配、增量/分代状态机、gray list、write barrier、shared ref tracking |
| per-type traverse | `xcoro_gc_traverse.c` | Array/Map/Set/Json/Closure/Instance/Iterator/Task/Module/Exception 等遍历 |
| 系统堆 | `xsystem_heap.*` | coroutine struct pool、class/module arena、shared refcount 分配 |
| 统一分配桥 | `xalloc_unified.*` | `xr_alloc()` / `xr_coro_alloc()` / barrier bridge，减少头文件反向依赖 |
| bytecode root map | `xbc_stackmap.*` | bytecode safepoint live-slot bitmap |
| closure 层对象 | `xclosure.*` `xcell.*` `xbound_method.*` | first-class closure、mutable capture cell、bound method 值 |

## 3. 三种堆域与所有权

### 3.1 `XrGC`：isolate 级 fixedgc，不是主运行时堆

`xgc_internal.h` / `xgc.c` 表明 `XrGC` 只维护：

- `fixedgc` 链表
- `g_destroy_funcs[]`
- isolate 生命周期内统一 cleanup

`xr_gc_alloc()` 的注释也明确了：

- 它主要用于 fixed objects / 初始化阶段对象
- 常规运行期对象应该走 `xr_alloc()` 或 `xr_coro_gc_newobj()`

所以这里的 `global GC` 更像：

- **固定对象回收器**
- **bootstrap fallback allocator**

而不是“全局 tracing heap”。

### 3.2 `XrCoroGC`：每协程主堆

`xcoro_gc.h` 里的 `XrCoroGC` 真正承载了运行期主堆状态：

- `immix`
- `GCdebt`
- `gc_requested`
- `gray` / `grayagain` / `weak`
- `large_objects`
- `shared_refs` / `prev_shared_refs`
- `owner`
- external root callbacks

对象创建主路径是：

- `xr_alloc(coro, size, type)`
- `xr_coro_ensure_gc(coro)`
- `xr_coro_gc_newobj(gc, type, size)`

也就是说：**绝大多数 runtime heap object 的 owner 是 coroutine，不是 isolate。**

### 3.3 `XrSystemHeap`：不参与 per-coro tri-color 的系统域

`xsystem_heap.h` 清楚定义了系统域职责：

- `XrCoroutine`：结构体池复用
- `XrClass` / `XrModule`：arena 分配
- shared object：refcount + malloc/mmap

这意味着：

- coroutine 结构本身不是 `XrCoroGC` 管的
- class/module 不是 `XrCoroGC` 管的
- channel/shared 不是 `XrCoroGC` tri-color 标记的

`xcoro_gc_traverse.c` 里也能看到对应假设：

- `XR_TCOROUTINE`：不在这里递归扫描内部状态
- `XR_TCHANNEL`：shared object，内部缓冲不在这里走 tri-color 遍历

## 4. closure/cell/bound method 的真实模型

### 4.1 `XrClosure`：flat upvals[]

`xclosure.h` 定义：

- `proto`
- `upval_count`
- trailing `upvals[]`

`xclosure.c` 分配时按 `proto->upvalues` 长度一次性拉平分配，upvals 初始全 `null`。

运行期 `OP_CLOSURE` 的逻辑很直接：

- `UPVAL_SRC_REG` → 从当前寄存器快照进新 closure
- `UPVAL_SRC_UPVAL` → 从外层 closure 的 `upvals[]` 复制

所以这里没有 Lua-style open upvalue 链，也没有 VM 级 upvalue 对象表。**capture 模型是 flat snapshot。**

### 4.2 `XrCell`：mutable capture 的唯一间接层

`xcell.h` / `xcell.c` 很简单：

- 32B：`[XrGCHeader][XrValue]`
- 用于“单个 mutable captured variable”

VM 指令语义：

- `OP_CELL_NEW`：把当前寄存器值包成 cell，再把寄存器改成 cell 指针
- `OP_CELL_GET`：解引用 cell
- `OP_CELL_SET`：写回 cell

这说明 closure 并不区分“值捕获 / 引用捕获”两套容器：

- closure 一律存 `XrValue`
- mutable capture 只是把这个 `XrValue` 变成 `XrCell*`

### 4.3 `XrBoundMethod`：闭包层对象，但不走 per-coro heap

`xbound_method.c` 的分配路径和 `XrClosure` / `XrCell` 不一样：

- `XrClosure` / `XrCell`：优先走 `coro->coro_gc`，必要时 fallback 到 `isolate->gc`
- `XrBoundMethod`：直接 `xr_gc_alloc(&isolate->gc, ...)`

也就是说：**bound method 是 closure 层对象，但其 owner 更接近 isolate fixedgc。**

这带来一个很关键的正确性需求：

- bound method 自己在 fixedgc
- 但它的 `receiver` 可能指向某个协程堆对象

`xr_coro_gc_markobject()` 的“defensive traversal for external objects not managed by this coro GC” 正是在兜这件事：

- 如果对象不是 shared
- 且已经不是当前 GC 白对象
- 但它有 refs
- 仍然递归 `xr_gc_traverse_object()`

这保证了：**fixedgc `XrBoundMethod` 仍能把 per-coro receiver 保活。**

## 5. Capture 元数据：哪些字段真正生效

`UpvalInfo` 记录：

- `index`
- `storage_mode`
- `is_const`
- `slot_type`
- `source`
- `type_info`

### 5.1 `source` 是 VM 运行期真实输入

`xvm_dispatch_closure.inc.c` 里：

- `UPVAL_SRC_REG`
- `UPVAL_SRC_UPVAL`

直接决定 `OP_CLOSURE` 从哪里复制 upvalue。

### 5.2 `storage_mode` 当前主要服务于 compiler 约束

`storage_mode` 的主要消费点在 frontend/codegen：

- `go closure` 是否允许捕获 mutable shared let
- 非 shared capture 的错误信息与约束
- bytecode 序列化保留元信息

但 VM `OP_CLOSURE` 本身并不看 `storage_mode`。

### 5.3 `slot_type` 当前主要服务于 backend，不是 closure GC

当前可见消费点主要在：

- `jit/xir_builder.c`
- AOT/backend 推导
- bytecode IO

但 closure runtime 路径：

- `OP_CLOSURE`
- `xr_gc_traverse_closure()`

都不使用 `slot_type`。因为 closure `upvals[]` 本身存的是 tagged `XrValue` / cell ref，GC 直接 `markvalue()` 即可。

结论：**`slot_type` 是 upvalue 的 backend metadata，不是当前 closure root map 的核心数据。**

## 6. GC root scan：真实根集合

### 6.1 VM 栈与 frame 是第一根集

`mark_coro_roots()` 明确写了：

- `vm_ctx` 是 stack/frames 的 single source of truth
- 逐帧扫描，避免仅看 `stack_top`
- C frame / 无 proto → conservative
- 有 `bc_stackmap` → 使用 safepoint bitmap
- 无 `bc_stackmap` → 整帧 conservative

这是当前解释器路径最核心的 root scan。

### 6.2 bytecode stackmap 不是“真实精确活跃指针图”

`xbc_stackmap.h` 说的是：

- 记录 live GC-managed slots
- precise root scan

但 `xcompiler.c:xr_codegen_record_gc_safepoint()` 的实现其实是：

- `freereg = xreg_get_freereg(...)`
- bitmap 把 `[0, freereg)` 全部置 1

也就是说当前记录的是：

- **寄存器前缀保守活跃区间**
- 不是“真实 SSA/slot liveness”
- 更不是“仅 pointer slot 的精确图”

因此现状更准确的描述应该是：

- 比整帧 conservative scan 更窄
- 但还不是严格 precise

### 6.3 struct_area 是独立扫描口

raw struct data 不是 `XrValue[]`，普通栈扫描看不到里面的 GC 指针。

`mark_struct_string_fields()` 当前只做一件事：

- 遍历 struct_area header
- 找到 `struct_layout`
- 只标记 `XR_NATIVE_STRING` 字段

这和当前 `xr_type_kind_to_native()` 主要支持 bool/int/float/string 相匹配；但 `xstruct_layout.h` 已经定义了：

- `XR_NATIVE_STRUCT`
- `XR_NATIVE_ARRAY`

所以这里存在一个未来漂移点：**一旦 native struct 真开始允许更复杂的 GC 指针字段，扫描逻辑必须同步扩张。**

### 6.4 JIT 根扫描是另一套体系

`mark_coro_roots()` 还会额外扫描：

- `jit_ctx->call_closure`
- 当前 safepoint 对应的 `XrStackMapTable`
- innermost frame 寄存器 / spill
- FP chain 上连续 JIT frame
- `jit_frame_stack` 里的 caller frame

说明当前 runtime 已经有 **bytecode GC map + JIT GC map 双 root 系统**。

### 6.5 其他 runtime roots 已被显式补齐

`mark_coro_roots()` 还额外标记：

- entry closure
- 主协程的 `vm->shared` 数组
- `coro->task`
- `coro->result` / `error` / `pending_closure_result`
- blocked send 的 `send_value`
- C frame 的 `cfunc_result`
- 每个 frame 的 `closure`
- external root callbacks

这个覆盖面已经不是“GC 模块自己能推出来”的，需要 runtime 运行时知识。

## 7. 遍历层的真实边界

`xcoro_gc_traverse.c` 表面在 `gc/`，实际已经是跨层遍历矩阵：

- object：Array / Map / Set / Json / Iterator / Exception
- closure：Closure / Cell / BoundMethod
- class：Instance
- module：Module export values
- task：structured concurrency tree + completion listeners
- runtime：VM frame / isolate extension callbacks

这意味着：

- `gc/` 不只是算法层
- `traverse` 文件已经承担 object/class/module/task/runtime 的聚合逻辑

这是本轮最重要的边界判断之一。

## 8. 正向发现

### 8.1 三堆域划分比以前清楚

当前至少在代码层已经区分出：

- fixedgc
- per-coro heap
- system/shared heap

这是清晰且可分析的，不再是“一个 GC 管所有东西”。

### 8.2 `xalloc_unified.c` 是一次成功的依赖减压

`xalloc_unified.h` 只暴露：

- `xr_coro_ensure_gc`
- `xr_coro_alloc`
- `xr_current_coro_gc`
- barrier bridge

而把真正需要 `xcoroutine.h` / `xworker.h` 的实现挪到 `.c`。这是比较健康的边界做法。

### 8.3 `xr_coro_gc_reset()` 很适合 coroutine pool 复用

`gc_reset` 会：

- finalize immix objects
- reset immix
- free large objects
- 清 root callbacks
- 保留 gray list buffer / tuning / shared_refs buffer

这和 coroutine struct pool 搭配是合理的。

### 8.4 `xr_coro_gc_markobject()` 的 defensive traversal 很关键

它让：

- fixedgc fallback object
- isolate fixedgc bound method
- 其他非本协程主堆但可达的对象

依然能继续把子对象标活。

这对当前混合所有权模型是必要的安全阀。

## 9. 已确认高风险点

### 9.1 `xcoro_gc.c` 里仍有进程级可变全局池状态

当前存在：

- `g_gc_pool_mu`
- `g_gc_pool_head`
- `g_gc_pool_count`

这意味着 `XrCoroGC` 结构体池是全进程共享的，而不是 isolate-owned。

它的收益是复用，但代价是：

- ownership 不再局部
- 和“尽量消灭文件作用域可变全局”的长期方向冲突

### 9.2 `xgc.h` 的公共叙述已经落后于真实实现

`xgc.h` 还在写：

- per-coro GC
- incremental Mark-Sweep with Arena allocation

而真实实现已经是：

- per-coro Immix mark-region
- fixedgc + system heap + shared refcount 并存
- JIT stack maps + bytecode stack maps 双轨

这会误导后续阅读者。

### 9.3 `xbc_stackmap` 的“precise”命名过强

实现层面当前没有：

- 真实寄存器/slot liveness
- pointer-only bitmap
- bitmap dedup

所以它现在更像：

- safepoint live-prefix map

而不是完全 precise stack map。

### 9.4 closure 相关对象的 owner 不统一

当前：

- `XrClosure`：通常 per-coro heap
- `XrCell`：通常 per-coro heap
- `XrBoundMethod`：直接 isolate fixedgc

这使得“closure 层对象”不是同一套生命周期模型。

### 9.5 `UpvalInfo.slot_type` 的语义注释与实际消费存在张力

现状是：

- 记录了 `slot_type`
- comments 常把它说成和 GC/root 相关
- 但 closure GC 真正扫描的是 `XrValue upvals[]`

所以如果以后有人以为“改了 slot_type 就会影响 closure scan”，很容易判断错。

### 9.6 `root_callbacks` 机制当前几乎还只是能力接口

源码里能看到：

- 注册/注销 API
- mark 时调用

但暂时没有 in-tree runtime 使用方。

这本身不一定错，但表示：

- 机制已经有了
- 但 root ownership policy 还没在核心模块里真正形成统一使用习惯

### 9.7 `traverse.c` 已经承担太多“对象细节 + 偏移协议”

例如：

- `XR_TCELL` 直接按 layout 偏移取 `value`
- `XR_TBOUND_METHOD` 直接按 layout 偏移取 `receiver`

这避免了额外 include，但也让 GC traverse 对对象物理布局有隐式依赖。

## 10. 改进建议

### 10.1 把“堆域选择”从隐式 fallback 变成更显式的协议

尤其是：

- `xr_alloc()` fallback 到 isolate->gc
- `xr_closure_new()` / `xr_cell_new()` 的 fallback
- `xr_bound_method_new()` 永远走 isolate->gc

这些都应该在接口/命名/注释层明确告诉调用者：**对象会落在哪个 owner 下。**

### 10.2 重新描述并收敛 `xbc_stackmap`

两条路径至少要做一条：

- **短期**：把文档和命名改成更真实的“conservative live-prefix map”
- **长期**：真正改成 pointer-only / live-slot precise bitmap

### 10.3 明确 `UpvalInfo` 字段的消费者分工

建议直接把语义写清楚：

- `source` → VM closure materialization
- `storage_mode` → compile-time coroutine safety / serialization
- `slot_type` → JIT/AOT/backend metadata
- `type_info` → backend / diagnostics / optimization

### 10.4 继续把 GC traverse 从“巨大 switch”往 owner 模块回推

不一定要完全拆散，但至少可以考虑：

- per-type traverse helper 更靠近对象 owner
- `gc/` 只保留注册/调度

现在 `xcoro_gc_traverse.c` 已经明显过于横切。

### 10.5 把 GC struct pool 从进程级全局降到更局部的 owner

一个自然方向是：

- 归 worker / system heap / isolate

避免 `g_gc_pool_*` 继续作为 process-wide mutable state 存在。

### 10.6 重新审视 bound method 的堆域

当前 fixedgc 方案是可工作的，但它意味着：

- 绑定方法值比普通 closure 更“粘住” isolate 生命周期
- 可达 receiver 需要依赖 defensive traversal

后续要么保持并明确文档，要么统一到更一致的 owner 模型。

## 11. 给下一轮 `object/` 的输入

下一轮建议重点沿这几条线进入：

- Array/Map/Set/Json/Iterator 在三种堆域中的分配策略
- shared object 与 per-coro object 的写屏障和深拷贝边界
- `XR_GC_IS_SHARED` / `XR_GC_FLAG_MMAP` / `data_on_gc_heap` / `nodes_on_gc` / `entries_on_gc` 的对象内存模型
- Json shape / object field / class instance field / typed field 在 traverse 与 formatter 中的差异
- `xmethod_table` 与 object methods 怎样继续放大跨层依赖

---

## 12. 本轮状态

- **已完成**
  - fixedgc / per-coro GC / system heap 三域划分
  - closure / cell / bound method 生命周期与 owner 判断
  - VM root scan、bytecode stackmap、JIT stack map、struct_area 特殊扫描梳理
  - `UpvalInfo` 字段的运行时/编译时消费者拆分
  - `root_callbacks`、defensive traversal、shared ref tracking 的定位

- **未完成**
  - 还没进入 Array/Map/Set/Json/Instance 的对象分配与 shared/normal 双路径
  - 还没回溯 write barrier 在各 object mutator 中的覆盖率
  - 还没系统盘点 `ximmix.*` 内部 block/line 策略对对象布局的影响

- **下一步**
  - 开始 `027-runtime-phase3-object.md`
