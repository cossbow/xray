# AOT 目标架构（仅基于源码分析）

> 日期：2026-04-27
> 状态：Draft
> 范围：`src/aot/`、`src/app/cli/xcmd_build.c::cmd_build_native`、`src/jit/xir_intrinsic.h`、`src/jit/xir_builder*.c`（`aot_mode` 分支）、`src/coro/*`、`src/runtime/gc/xcoro_gc.c`、`src/runtime/gc/xsystem_heap.c`、`src/runtime/xisolate_internal.h`
> 配套文档：`032-aot-binary-size.md`（体积策略）、`033-aot-implementation.md`（缺陷修复实施）

---

## 0. 文档定位

本文是**架构卷**，只回答一个问题：**重构后的 AOT 应该长什么样**。
不讲当前缺陷与修复路径（在 033），不讲二进制体积切割（在 032）。

只读源码、不参考任何已有 task / design 文档得出。设计原则与 `004-aot-refactor.md` 一致：**不保兼容、直选最佳设计、删半成品**。

---

## 1. 关键约束（来自源码事实）

源码层面已经确定的、不可动的事实约束：

1. **AOT 是 XIR → C → 原生二进制**：`src/aot/xcgen*.c` 输出 C 源；`src/app/cli/xcmd_build.c::invoke_cc_standalone` 调 cc 链接（参考 Nim `cgen.nim`）。
2. **协程必须以 CPS 形态落地 C**：原生 C 没有 Xray 协程概念，AOT 不得携带 VM 解释器，那就只剩 CPS 翻译一条路（参考 Go runtime / Kotlin coroutines）。
3. **运行时已有完整 per-coroutine Immix GC**：`src/runtime/gc/xcoro_gc.c` 已实现，`XrIsolate::coro_gc` 在 `xisolate_internal.h` 暴露。
4. **跨协程对象共享走两条事实路径**：深拷贝（`src/coro/xdeep_copy.c`）或 system heap 引用计数（`src/runtime/gc/xsystem_heap.c`）。
5. **JIT 已有 `XirIntrinsicId` 枚举**：`src/jit/xir_intrinsic.h` 注释明确写"3. Update `xcgen_call.c::emit_call_intrinsic` to lower it to C"——半成品。
6. **VM 解释器与运行时是两层东西**：`src/vm/`（解释器）和 `src/runtime/`（值/GC/对象/类）可以解耦；当前耦合点只剩 `XrCoroutine` 内嵌的 VM 状态字段。

**结论**：AOT 与 VM 共享的不是"解释器"而是"运行时"，AOT 应当**复用整个 `src/runtime/` + `src/coro/`，但不嵌入 `src/vm/`**。

---

## 2. 核心架构决策

### D1 — AOT 复用 runtime + coro，不复用 vm

```
[AOT 二进制]
  ├─ AOT 生成代码（CPS 状态机 + 直接函数调用）
  ├─ src/runtime/   （value / gc / object / class / symbol / closure）
  ├─ src/coro/      （worker / sched / channel / task / timer / netpoll）
  └─ src/base/      （malloc / arena / vector / map / hash / log）
```

**不打包**：`src/vm/`（字节码解释器）、`src/jit/`（动态 codegen）、`src/aot/codegen/`（编译期工具）、`src/frontend/`（lexer/parser/analyzer）、`src/app/`（CLI/LSP/DAP）。

### D2 — 协程：CPS 翻译 + 复用 `src/coro/` 调度器

每条 Xray 协程函数被翻译成 C 状态机 struct：

```c
typedef struct co_user_main_env {
    XrCoroState state;          // 与 src/coro/xcoroutine.h 兼容
    XrCoroGC    *gc;            // per-coro Immix GC（D3）
    int          pc;            // 状态机当前位置
    /* 捕获的寄存器与局部 */
    XrValue      r0, r1, r2;
    XrtArrayT   *acc;
} co_user_main_env;

static XrCoroResult co_user_main_step(co_user_main_env *env);
```

`src/coro/xworker_exec.c` 增加 `CPS` 分发分支（`#ifdef XRAY_AOT_CPS`），调用 `step()` 回调而不是 `xr_vm_resume()`；`xworker_handle_vm_result` / `xchannel.c` / `xtask.c` / `xworker_sched.c` / `xtimer_wheel.c` / `xnetpoll.c` 全部原样复用。

CPS 翻译只是改变"如何 step"，不改"何时 step"。后者已经被现有调度器解决得很好（17,600 行成熟代码）。

### D3 — 内存模型：每协程独立 Immix GC（与 VM 同款）

直接复用 `src/runtime/gc/xcoro_gc.c`。AOT 生成的 step 函数运行在 `XrCoroutine` 上下文，分配走 `xcoro_gc_alloc`，回收走 Immix mark/sweep。

**为什么不是 bump-only？**

源码层面 CPS 翻译有两个特性把 bump-only 排除掉：

| CPS 后果 | bump 失败原因 |
|---|---|
| 闭包 env 在 heap 上，跨 step 调用边界存活 | env 即使逃出当前 step，bump 也无法精确回收 |
| 长期协程（HTTP server / GUI）持续运行 | bump 单调增长，进程级泄漏 |
| Channel / Future / Promise 的内部缓冲跨 step | 同上 |

bump-only 仅适合**确定短命**的脚本/CLI 工具。这是体积策略（032）问题，不是默认架构。

**为什么不是 ARC？**

ARC 在 `xrt_arc.h` 已经写了一半，但跑不下去：

- 容器（array/map/strbuf/closure）当前裸 malloc，没 ARC 头；
- ARC 在协程间传值时需要原子计数 + 防 cycle；
- 加完原子 + cycle collector 复杂度逼近 tracing GC，但性能更差。

ARC 在 VM/JIT 已经被 tracing GC 取代（参见 `MEMORY[634f5777]`：runtime 是多个所有权域并存），AOT 没必要再走一次老路。

### D4 — 跨协程对象：deep_copy + system_heap，与 VM 同语义

`src/coro/xdeep_copy.c` 与 `src/runtime/gc/xsystem_heap.c` 已经定义了 cross-coro 边界：

- 默认走 `xr_deep_copy_to_coro(src_val, dst_coro)`；
- `shared`/`const` 修饰的值走 `xsystem_heap_retain` 引用计数；
- AOT 生成 `chan.send(v)` 直接调上述两个 helper，**不再实现一份**。

### D5 — runtime 入口：`XrtRuntime` 持有所有可变全局

当前 AOT runtime 散落 file-scope 全局：`xrt_type_table[]`、`xrt_exc_top`、`xrt_shared[]`、`xrt_bump_enabled`。重构后全部进 `XrtRuntime`：

```c
typedef struct XrtRuntime {
    XrIsolate     *isolate;     // 复用 runtime/xisolate
    XrCoroSched   *sched;       // 复用 src/coro/xworker_sched
    XrtTypeTable   types;
    XrtExcFrame   *exc_top;     // 异常链表头（per-thread 实际放 TLS）
    XrValue       *shared;
    int            shared_count;
} XrtRuntime;
```

生成的 `main()`：

```c
int main(int argc, char **argv) {
    XrtRuntime rt;
    xrt_runtime_init(&rt, argc, argv);     // 启动 isolate + 起 coro 0
    xrt_runtime_run(&rt, &co_user_main);   // 把入口协程提交给调度器
    xrt_runtime_shutdown(&rt);
    return rt.exit_code;
}
```

这条链路与 VM 启动 isolate 的代码 **同源、同形**，只是 AOT 不创建 `XrVM`。

### D6 — 与 JIT 的解耦：`XirIntrinsicId` 单一权威

AOT codegen **不**通过函数指针地址识别 helper（当前 `emit_call_c` 的 ~900 行 `if (fn_ptr == &xr_jit_x)` ladder）。改为：

```
XIR builder (aot_mode)  →  emit XIR_CALL_INTRINSIC(id, args...)
                                      │
                                      ↓
AOT codegen   xcgen_intrinsic.c::xcgen_intrin_table[id](...)
JIT codegen   xir_codegen_*.c    ::jit_intrin_table[id](...)
```

`XirIntrinsicId` 在 `src/jit/xir_intrinsic.h` 已定义（GETPROP / INDEX_GET/SET / MAP_GET/SET / STRBUF_* / SUBSTRING / STR_REPEAT / CHR / INVOKE_METHOD / GET_SHARED / SET_SHARED / PRINT / RT_* / THROW / JSON_NEW_SHAPE）。

枚举即"唯一权威"，JIT/AOT 两侧各有一张 table 注册自己的 lowering，**没有任何一侧通过指针识别另一侧**。

### D7 — 与 CLI 的解耦：AOT driver 抽出

当前 600 行编排逻辑写在 `src/app/cli/xcmd_build.c` 里：bundle 拓扑 → 逐模块 compile → collect_exports → build_shared_proto_map → aot_preregister_classes → aot_build_export_map → 写 main。

重构后：

```
src/aot/driver/
  xaot_driver.c        // xaot_compile_bundle(bundle, output, options)
  xaot_metadata.c      // 在编译期收集 export/class/struct/import 元数据
  xaot_emit_main.c     // 输出 main() + runtime lifecycle
```

`cmd_build_native` 缩为 ≤ 50 行：参数解析 + 调 `xaot_compile_bundle()` + 调 cc。

### D8 — 元数据前移到前端

当前 `cmd_build_native` 反扫 bytecode 推断：构造函数字段数、shared_index → child proto、导出名、super 类、字段类型。这些信息**前端编译期已有**，只是没传下来。

`XrProto / XrShape / XrClassDescriptor` 加 `aot_meta` 字段，前端 codegen 时直接填。AOT driver 直接读，不再 pattern match。

bytecode 一旦改动，pattern match 全失效；元数据前移后，AOT 对 bytecode 演进无感。

---

## 3. 模块依赖 DAG

```
L0  base/                  (xr_malloc, vector, hash, log)
L1  runtime/value          (XrValue, tag union)
    ↓
L2  runtime/gc             (xcoro_gc, system_heap, fixedgc)
    runtime/closure
    ↓
L3  runtime/object         (array, map, string, strbuf)
    ↓
L4  runtime/class
    runtime/symbol
    ↓
L5  coro/                  (worker, sched, channel, task, timer, netpoll, deep_copy)
    ↓
L6  src/aot/runtime/       (AOT 专属薄壳：tagged 算术 / box-unbox / setjmp 异常框架)
    ↓
L7  AOT 生成的 .c          (用户代码翻译产物)
```

`src/aot/runtime/` 只是"把 runtime + coro 适配成生成代码方便调用的形状"，不再有自己的 GC / 类型系统 / 协程实现。

具体 AOT 私有头：

| 头 | 作用 |
|---|---|
| `xrt.h` | umbrella |
| `xrt_value.h` | tag union 别名（不再 forward decl `xrt_str_alloc`） |
| `xrt_alloc.h` | 三档策略宏：`XRT_ALLOC_GC`（默认，走 `xcoro_gc`）/ `BUMP` / `MALLOC` |
| `xrt_arith.h` | tagged 算术（与 `runtime/value/xvalue_arith.c` 共享实现） |
| `xrt_intrinsic.h` | `XirIntrinsicId` 共享头（`#include "../jit/xir_intrinsic.h"`） |
| `xrt_exception.h` | setjmp/longjmp 异常 frame（与 VM 异常链对齐） |
| `xrt_runtime.h` | `XrtRuntime` 结构 + init/run/shutdown |

**删除**的 AOT 私有头：

- `xrt_compat.h`（一个 typedef 内联进 `xrt_value.h`）
- `xrt_module.h`（`xrt_shared[]` 已是事实路径）
- `xcgen_bridge.h`（intrinsic 化后零必要）
- `xrt_arc.h` 中的 retain/release/deinit（D3 的 GC 模型不需要）
- `xrt_class.h` 中独立的 type table（合并进 `runtime/class`）
- `xrt_method.h` 中散户 `if/else`（合并进 `runtime/symbol` 的 X-macro）

---

## 4. 控制流：AOT 完整执行链路

### 4.1 编译期（host 工具链）

```
xray build --native
  │
  └─ src/app/cli/xcmd_build.c::cmd_build_native （≤ 50 行）
       │
       └─ src/aot/driver/xaot_driver.c::xaot_compile_bundle
            │
            ├─ for each module:
            │    frontend → XrProto + aot_meta（D8）
            ├─ xaot_metadata::collect_bundle_meta（直接读 aot_meta，不再反扫）
            ├─ for each proto:
            │    xir_build_from_proto_aot_ex →  XirFunc（带 INTRINSIC opcode）
            │    src/aot/codegen/xcgen_compile_func → C 字符串
            ├─ src/aot/driver/xaot_emit_main → 生成 main() + runtime hook
            ├─ 写入 build/aot/<bundle>.c（每模块独立 .c + main.c）
            └─ invoke_cc_standalone → 链接 src/runtime + src/coro + src/base + AOT .c
```

### 4.2 运行期（生成二进制）

```
main()
  └─ xrt_runtime_init               // 初始化 isolate + sched + types + shared
  └─ xrt_runtime_run(&co_user_main) // 提交入口 coro
       │
       └─ src/coro/xworker_exec.c::worker_loop
            │  CPS 分发分支（#ifdef XRAY_AOT_CPS）
            └─ co_user_main_step(env)  ← AOT 生成的状态机
                 │
                 ├─ xrt_box_int / xrt_array_get / ...   （AOT runtime 薄壳）
                 ├─ xrt_call_intrinsic_*                （表驱动）
                 └─ XR_CHECK / XR_THROW                 （setjmp/longjmp）
  └─ xrt_runtime_shutdown            // 等待 sched 排空、销毁 isolate
```

---

## 5. 与现有 Phase 路线图的关系

`005-aot-implementation.md` 把 AOT 切成 Phase A/B/C/D：

| Phase | 内容 | 本文（031）的位置 |
|---|---|---|
| A | 多模块编译管线 + ARC runtime | A 的"多模块管线"保留；"ARC runtime" **被 D3 取代** |
| B | Class / 异常 / 类型特化 | 完整保留 |
| C | CPS 并发 | 由 D2 全面具体化（CPS + 复用 `src/coro`） |
| D | stdlib bridge / 多文件 / 增量 | 完整保留 |

本文不重排 Phase，只把 Phase A 的"ARC + bump"决策替换成 D3 的"per-coro Immix GC"，并把 Phase C 的"CPS 并发"从抽象计划落到具体复用点。

---

## 6. 不变量（永远满足）

合入后必须永远成立：

1. **AOT 不依赖 `src/vm/`**：`src/aot/` 与生成代码不 include `src/vm/*.h`。
2. **AOT 不通过函数指针识别 helper**：`xcgen_*.c` 不出现 `xr_jit_*` 符号；统一 `XIR_CALL_INTRINSIC`。
3. **runtime 头不被 codegen 寄生**：所有 `xrt_*.h` 自包含、可独立单测。
4. **runtime 没有 file-scope 可变全局**：所有可变状态在 `XrtRuntime` 内（红线"禁止文件作用域可变全局"）。
5. **生成的 main() 总是成对调用 `xrt_runtime_init/shutdown`**。
6. **runtime DAG 单向**：见第 3 节，不允许 L_n → L_{>n} 反向 include。
7. **协程语义与 VM 完全等价**：调度器、channel、scope、deep_copy、system_heap 都是同一套。
8. **跨模块访问只走 `xrt_shared[]`**：不复活 `xrt_modules[]` / `xrt_module_lookup`。
9. **bytecode 元数据通过 `aot_meta` 显式传递**：禁止反扫 bytecode 推断。
10. **所有限制都是动态扩张**：除了语言层真正的常量（如 tag bit 位宽）。

`033-aot-implementation.md::§9` 给出每条不变量对应的 grep / 单测断言。

---

## 7. 不在本卷范围

- **缺陷修复**：见 `033-aot-implementation.md`（R1..R18 + Phase S1..S9）。
- **二进制体积**：见 `032-aot-binary-size.md`（与 Nim/OCaml 对比、自动 feature 推断、模块化 runtime）。
- **JIT 后端、协程内部实现、frontend 演进**：与本卷无关。

---

## 8. 总结

源码事实告诉我们：

1. AOT 与 VM 共享的边界**不是解释器**，而是 **runtime + coro**——这两层已经成熟，没必要为 AOT 重写。
2. CPS 翻译 + 复用 `src/coro` 调度器是唯一既"不嵌 VM"又"完整语言特性"的方案。
3. 内存模型选 **per-coroutine Immix GC**，与 VM 同款；bump-only 仅作为体积/短命场景的 opt-in（032 卷处理）。
4. 跨层耦合通过 `XirIntrinsicId` 单一权威 + `aot_meta` 元数据前移彻底切断。
5. CLI 退化成"参数解析 + 调 driver"，编译器逻辑回到 `src/aot/`。

按此架构落地后，AOT 是一个干净的、可独立演进的子系统，不会再和 JIT 用指针对齐、不会和 CLI 用反扫 bytecode 谈判、不会和 runtime 用 forward decl 通信。
