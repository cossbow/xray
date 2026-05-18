# GC 与内存管理

## 协程 GC (XrCoroGC) — Immix Mark-Region

### 核心特性

- **Immix block-line 分配器**：bump-pointer 速度 + line 级回收
- **对象不移动**：C 扩展天然安全
- **每协程独立 GC**：无全局 STW，百万并发友好
- **批量释放**：协程结束时释放所有 Immix 块
- **增量 GC**：Lua GCdebt 机制，避免长暂停

### 内存布局

```
1. 执行栈（独立分配，realloc 增长）
2. 对象堆（Immix blocks：full_blocks / recycle_blocks / current_block / old_blocks）
3. 大对象（>4KB，单独 malloc）
```

### 三色标记

```c
XGC_WHITE0 / XGC_WHITE1   // 白色（双白色位交替）
XGC_BLACK                  // 黑色（已标记，引用已扫描）
// 灰色 = 非白非黑（在 gray/grayagain/weak 链表中）
// 不变量：黑色对象不能指向白色对象
```

### GC 状态机（4 状态）

```
PAUSE → PROPAGATE → ATOMIC → SWEEP → PAUSE
         (增量)      (原子)   (含内联 finalize)
```

- **PAUSE**: 空闲，GCdebt 触发转入 PROPAGATE
- **PROPAGATE**: 增量标记，可中断返回 mutator
- **ATOMIC**: 不可中断，重标记根、清弱表、翻转白色位
- **SWEEP**: 非增量扫描所有块和大对象，内联调用终结器

### 分代模式（Sticky Immix，可选）

增量模式和分代模式可切换。分代模式下 minor GC 只扫描新分配对象。

### 写屏障

```c
xr_coro_gc_barrier(gc, parent, child);     // Forward Barrier
xr_coro_gc_barrierback(gc, obj);           // Back Barrier（容器批量修改）
```

### 默认配置

| 配置 | Main 协程 | Spawn 协程 |
|------|-----------|------------|
| Arena 初始 | 4MB | 16KB |
| GC 阈值 | 8MB | 32KB |
| GC 类型 | 分代 | 增量 |

## 系统堆 (XrSystemHeap) — 跨协程对象

| 对象类型 | 分配策略 | 释放方式 |
|----------|----------|----------|
| 协程结构 | 对象池复用 | 返回池 |
| Class/Module | Arena 分配 | Isolate 销毁时批量释放 |
| shared 对象 | malloc + 引用计数 | refcount=0 释放 |
| 大对象 (≥64KB) | mmap | munmap |

## 关键代码位置

- `src/runtime/gc/xcoro_gc.c/h` — Immix Mark-Region GC 实现
- `src/runtime/gc/ximmix.c/h` — Immix block/line 分配器
- `src/runtime/gc/xgc.c/h` — GC 公共接口
- `src/runtime/gc/xalloc_unified.c/h` — 统一分配接口
- `src/runtime/gc/xsystem_heap.c/h` — 系统堆
- `src/base/xarena.c/h` — Arena 分配器（base 层）
