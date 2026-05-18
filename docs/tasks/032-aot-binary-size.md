# AOT 二进制体积策略与自动 Tradeoff

> 日期：2026-04-27
> 状态：Draft
> 范围：AOT 输出的 standalone 二进制体积；不涉及 JIT、不涉及主仓内部测试用 binary
> 配套文档：`031-aot-architecture.md`（架构边界）、`033-aot-implementation.md`（缺陷修复实施）

---

## 0. 文档定位

本文是**体积卷**，回答两个问题：

1. **为什么 Xray AOT 二进制比 Nim / OCaml 同等程序大？**
2. **怎么把它切得更细，让用户按需选择"省体积 vs 完整特性"？**

不讲架构（在 031），不讲缺陷（在 033）。

---

## 1. 体积现状（事实）

### 1.1 Xray AOT 默认带的运行时表面

按 031 的架构（per-coroutine Immix GC + 复用 `src/coro` 全套），最小可执行二进制至少包含：

| 子系统 | 来源 | 估算（对象代码） | 是否运行时必需 |
|---|---|---:|---|
| Tagged value 表征 + box/unbox + 算术 | `runtime/value`、`src/aot/runtime/xrt_value.h`、`xrt_arith.h` | ~30 KB | 是 |
| 字符串 + strbuf + concat | `runtime/object/xstring*` | ~25 KB | 是 |
| Array / Map（含 hash、resize） | `runtime/object/xarray*`、`xmap*` | ~60 KB | 看程序 |
| Class 元数据 + 反射钩 | `runtime/class/*` | ~30 KB | 看程序 |
| Closure / 逃逸闭包 | `runtime/closure/*` | ~12 KB | 看程序 |
| 异常 setjmp/longjmp + stacktrace | `xrt_exception.h`、`runtime/value/xerror*` | ~15 KB | 看程序 |
| **per-coro Immix GC** | `runtime/gc/xcoro_gc*` | **~80 KB** | 见 §3 |
| **system heap + cross-coro deep_copy** | `runtime/gc/xsystem_heap*`、`coro/xdeep_copy*` | **~40 KB** | 见 §3 |
| **Coro 调度器（worker / sched / task）** | `coro/xworker*`、`xcoro*`、`xtask*` | **~120 KB** | 见 §3 |
| **Channel + select** | `coro/xchannel*` | **~50 KB** | 见 §3 |
| **timer wheel + netpoll（kqueue/epoll）** | `coro/xtimer_wheel*`、`xnetpoll*` | **~70 KB** | 见 §3 |
| stdlib bridge（json / regex / fs / time / ...） | `stdlib/*` | 视使用 | 看程序 |

最小 "hello world" 默认会带 GC + 调度器 + channel + timer + netpoll，因为它们之间有静态引用链，链接器无法剥离。**这是体积偏大的根本原因，不是代码膨胀。**

### 1.2 与 Nim / OCaml / Go 的对比口径

| 语言 | 默认运行时形态 | 一个 hello | 关键差异 |
|---|---|---:|---|
| Nim (`-d:release --gc:arc`) | ARC + 极简 stdlib | ~80 KB | 无调度器、无并发原语；ARC 无 cycle |
| OCaml (`ocamlfind ocamlopt`) | minor/major GC + 静态链接 stdlib | ~300 KB | 单线程语义，无 M:N 调度 |
| Go (`go build`) | full goroutine sched + GC + netpoll + reflect | ~1.5 MB | 与 Xray 设计目标对齐 |
| Xray AOT 当前 | 见 1.1 | ~1.2 MB（估算） | 与 Go 同档，比 Nim/OCaml 多一整套并发运行时 |

**结论**：Xray 与 Go 是同一档（要并发就要带调度器）；与 Nim/OCaml 不在同一档。**不能在保留协程语义的前提下追平 Nim 的体积**——除非显式放弃并发。

这就要求 AOT 给出**显式的 feature gate**，让用户自己选要不要付协程的体积成本。

---

## 2. 设计原则

1. **默认完整**：默认编译出"完整 Xray 程序"，包括协程、channel、timer、stdlib 常用模块。
2. **可向下退化**：用户能明确说"我这个 binary 不用协程 / 不用 channel / 不用 timer"，体积按比例下降。
3. **静默失败禁止**：剥掉一个特性后，源代码若仍使用该特性，**编译期报错**，不在运行时崩。
4. **自动推断优先**：能从源代码静态分析得出"未使用某特性"时，自动剥离，不要求用户手动指定。
5. **零运行时开销**：feature gate 只影响链接面，不影响运行时性能。
6. **单一二进制约定**：不引入"两套 runtime 并存的二进制"——每个 binary 只装一套。

---

## 3. Feature 切割方案

### 3.1 Feature 维度

```
XR_FEAT_GC_FULL        per-coro Immix（默认）
XR_FEAT_GC_BUMP        bump-only（短命脚本）
XR_FEAT_GC_NONE        全栈分配（实验，受限）

XR_FEAT_CORO           协程 + worker + sched
XR_FEAT_CHANNEL        channel + select   （隐含 CORO）
XR_FEAT_SCOPE          scope / structured concurrency （隐含 CORO）
XR_FEAT_TIMER          timer wheel        （隐含 CORO）
XR_FEAT_NETPOLL        kqueue/epoll       （隐含 CORO）
XR_FEAT_DEEP_COPY      cross-coro deep copy （默认随 CORO 开）

XR_FEAT_EXCEPTION      try / catch / finally
XR_FEAT_REFLECTION     class metadata 字段名 / 类型名
XR_FEAT_STACKTRACE     异常带 PC → 源行
XR_FEAT_INSTANCEOF     运行时类型测试

XR_FEAT_STDLIB_JSON    json / parseStrict
XR_FEAT_STDLIB_REGEX   pcre 引擎
XR_FEAT_STDLIB_FS      文件系统
XR_FEAT_STDLIB_TIME    日期/时区
XR_FEAT_STDLIB_HTTP    http server / client
... 每个 stdlib 子模块一个 flag
```

### 3.2 Feature 之间的依赖图

```
GC_FULL  ──┐
GC_BUMP  ──┼──> 必选其一
GC_NONE  ──┘

CORO  ──> CHANNEL / SCOPE / TIMER / NETPOLL / DEEP_COPY  （CORO 关，全部关）
GC_BUMP / GC_NONE  + CORO  →  禁止（CPS env 必须可回收）

EXCEPTION  独立
REFLECTION → INSTANCEOF
STACKTRACE → EXCEPTION

STDLIB_HTTP → STDLIB_FS + NETPOLL + CORO
STDLIB_REGEX 独立
...
```

driver 在编译期检查依赖一致性，违反则 hard fail。

### 3.3 切割粒度落到代码

每条 feature 对应一组 `.c/.h`，CMake 拆成 OBJECT library，AOT 链接时按需选：

```cmake
add_library(xrtcore_core OBJECT runtime/value/* base/*)
add_library(xrtcore_string OBJECT runtime/object/xstring*)
add_library(xrtcore_array OBJECT runtime/object/xarray*)
add_library(xrtcore_map OBJECT runtime/object/xmap*)
add_library(xrtcore_class OBJECT runtime/class/*)
add_library(xrtcore_exception OBJECT xrt_exception.c runtime/value/xerror*)

add_library(xrtcore_gc_full OBJECT runtime/gc/xcoro_gc*)
add_library(xrtcore_gc_bump OBJECT src/aot/runtime/xrt_alloc_bump.c)

add_library(xrtcore_coro    OBJECT coro/xcoroutine* coro/xworker*)
add_library(xrtcore_channel OBJECT coro/xchannel*)
add_library(xrtcore_scope   OBJECT coro/xscope*)
add_library(xrtcore_timer   OBJECT coro/xtimer_wheel*)
add_library(xrtcore_netpoll OBJECT coro/xnetpoll*)
add_library(xrtcore_deepcopy OBJECT coro/xdeep_copy*)

add_library(xrtstdlib_json  OBJECT stdlib/json/*)
add_library(xrtstdlib_regex OBJECT stdlib/regex/*)
...
```

AOT driver 在调 cc 时按 enabled features 拼链接列表：

```bash
cc -o app build/aot/*.o \
   $(xrtcore_core)        $(xrtcore_string)  $(xrtcore_array) ...   \
   $(xrtcore_gc_$gc)                                                 \
   $(if $coro,    $(xrtcore_coro))                                   \
   $(if $channel, $(xrtcore_channel))                                \
   $(if $netpoll, $(xrtcore_netpoll))                                \
   ...
```

为了让 dead-strip 真正生效，所有这些 OBJECT lib 用 `-ffunction-sections -fdata-sections`，链接 `-Wl,--gc-sections`（macOS：`-dead_strip`）。

### 3.4 替换桩（不让源码崩）

被剥掉的 feature 必须有"编译期错误"的桩，避免运行期 SIGSEGV：

```c
/* xrt_channel_stubs.c — 当 XR_FEAT_CHANNEL=0 时链接 */
void xchan_send(...) {
    xrt_runtime_abort("channel.send called but XR_FEAT_CHANNEL is disabled "
                      "for this AOT binary; recompile with --feat=channel");
}
```

driver 在元数据阶段（031 §D8）已经知道用户代码用了哪些特性。如果 user code 里出现 `chan.send(...)` 而 user 又显式 `--no-feat=channel`，**编译期 hard fail**，不要等到 link/run。

---

## 4. 自动 Feature 推断

### 4.1 推断器

在 `src/aot/driver/xaot_metadata.c` 增加 feature 推断 pass：

```c
typedef struct AotFeatureSet {
    bool need_coro;
    bool need_channel;
    bool need_scope;
    bool need_timer;
    bool need_netpoll;
    bool need_deep_copy;
    bool need_exception;
    bool need_reflection;
    bool need_stacktrace;
    bool need_instanceof;
    AotStdlibSet stdlib;       // bitset for each stdlib module
} AotFeatureSet;

AotFeatureSet xaot_infer_features(const AotBundle *bundle);
```

推断规则（来自 `XrProto::aot_meta` + frontend 已知用法）：

| 推断对象 | 触发条件 |
|---|---|
| `need_coro` | bundle 中任意 proto 出现 `OP_GO` / `OP_GO_SCOPE` / `OP_AWAIT` |
| `need_channel` | 出现 `OP_CHAN_NEW` / `OP_CHAN_SEND` / `OP_CHAN_RECV` / `OP_SELECT` |
| `need_scope` | `OP_SCOPE_BEGIN` |
| `need_timer` | 调用 `time.sleep` / `time.after` / `setTimeout` |
| `need_netpoll` | 调用 `net.*` / `tcp.*` / `http.*` / `fs.openAsync` |
| `need_deep_copy` | `need_coro && (need_channel || OP_GO_PARAM)` |
| `need_exception` | 出现 `OP_TRY_BEGIN` / `OP_THROW` |
| `need_reflection` | 调用 `class.fields()` / `class.name()` / `typeof` 取字段名 |
| `need_stacktrace` | `need_exception && (catch 中调 .stack 或 .toString 含栈)` |
| `need_instanceof` | 出现 `OP_IS` / `instanceof` |
| `stdlib.json` | proto 里 import `json` |
| `stdlib.regex` | import `regex` |
| ... | 每个 stdlib 模块一对一 |

依赖闭包：`need_channel → need_coro`，etc.

GC 模式不自动推断（语义影响太大）：默认 `XR_FEAT_GC_FULL`，`--gc=bump` 显式 opt-in。

### 4.2 用户控制

```
xray build --native main.xr           # 自动推断（默认）
xray build --native main.xr -v        # 自动推断 + 打印 feature 列表
xray build --native main.xr --features=+regex     # 强制开
xray build --native main.xr --features=-stacktrace  # 强制关
xray build --native main.xr --gc=bump --no-coro    # 极小二进制
xray build --native main.xr --features=full        # 全开（debug 用）
xray build --native main.xr --features=min         # 极小（仅 GC + value + 算术）
```

`--features=+x / -x` 是相对调整；`full / min / auto` 是预设。

### 4.3 体积报告

driver 链完后输出：

```
$ xray build --native main.xr --size-report

[features auto]
  ✓ gc       = full     (per-coro Immix)
  ✓ coro     = on       (used by main.xr:42 (go fn))
  ✓ channel  = on       (used by main.xr:55 (chan<int>))
  ✗ timer    = off      (no time.sleep/after)
  ✗ netpoll  = off      (no net/http/tcp)
  ✓ exception= on
  ✗ reflection = off
  ✓ stdlib.json = on    (used by main.xr:12)
  ✗ stdlib.regex = off

[size breakdown]
  user code         48 KB
  xrtcore_core      32 KB
  xrtcore_string    24 KB
  xrtcore_array     38 KB
  xrtcore_map       58 KB
  xrtcore_class     28 KB
  xrtcore_exception 14 KB
  xrtcore_gc_full   78 KB
  xrtcore_coro     124 KB
  xrtcore_channel   48 KB
  xrtcore_deepcopy  36 KB
  xrtstdlib_json    52 KB
  ─────────────────────
  total            580 KB
```

让用户**看见**自己付了什么钱。

---

## 5. 与体积更小的 Tradeoff 路径

某些场景用户愿意放弃语言特性换更小体积，driver 给出"梯度路径"：

| 目标体积 | 牺牲 | 配置 |
|---|---|---|
| ~1.2 MB | — | `--features=full`（默认 + stdlib 全开） |
| ~600 KB | 关掉未用的 stdlib | `auto`（默认推断） |
| ~350 KB | 关 coro / channel / timer / netpoll | 程序确实是单线程脚本 |
| ~200 KB | + 关 exception / reflection / stacktrace | 数值计算 / 离线工具 |
| ~120 KB | + GC 换 bump | 短命脚本（process exit = free all） |
| ~80 KB  | + 关 array/map（仅 value + 算术 + string） | 极简 CLI |

每档都是**可达**的，driver 给出验证：把所有用到的 opcode 列出，与该档的可用 opcode 集合做差，差集为空才能编出来。

每档都是**显式**的：`auto` 推断到的若超出用户指定档位，就 hard fail 并打印"原因 + 哪行代码导致需要该特性"。

---

## 6. 与 GC 选型的关系

031 已确定默认 GC = per-coro Immix。本文补足：bump-only / arena / no-GC 都是**体积 opt-in**，不是默认。

| 场景 | 推荐 GC | 是否允许 coro |
|---|---|---|
| HTTP server / 长期协程 | full Immix | 是 |
| CLI 工具（< 1s） | bump 也可 | 否（CPS env 无法回收） |
| 嵌入式 firmware | malloc + 显式 free | 否 |
| 数值计算 | full 或 bump | 是（bump 时无 cycle） |

**禁止组合**：`gc=bump|none + coro=on`——CPS env 跨 step 存活，bump 无法回收，进程级泄漏。driver 编译期 hard fail。

---

## 7. 实现阶段

体积切割不是单独 Phase，而是**贯穿** `033-aot-implementation.md` 的所有 Phase。具体落点：

| 033 中的 Phase | 本文相关动作 |
|---|---|
| S1 现状清理 | 加 `-ffunction-sections / -fdata-sections / -dead_strip` 链接 flag，立得 5–10% 收益 |
| S2 Intrinsic 化 | feature flag 注入 intrinsic 表（关 `STRBUF_*` 时表里整段不进 binary） |
| S3 元数据前移 | 顺手做 §4.1 的 feature 推断器（`AotFeatureSet`） |
| S4 内存模型决策 | 落 `XR_FEAT_GC_FULL/BUMP/NONE` 三档 |
| S5 runtime lifecycle | 把"没用到的子系统不调 init"做实 |
| S6 driver 抽出 | 落 `--features` CLI 选项 + size report |
| S7 / S8 / S9 | 把 stdlib 切成一模块一 OBJECT lib，落 `xrtstdlib_*` 拆分 |

每个 Phase 后跑一次 size 报告，观察落地效果。

---

## 8. 验收

体积层面的硬性指标：

1. **`hello world` 默认（auto）≤ 600 KB**（macOS arm64 release，开 LTO + dead-strip）。
2. **`hello world` 极小档（min + bump）≤ 150 KB**。
3. **HTTP server 示例（auto）≤ 1.5 MB**，与 Go 等价示例同档。
4. **size report 与 `bloaty`/`size` 实测误差 < 5%**。
5. **强制关闭一个被用到的 feature**：driver 编译期 hard fail，**绝不允许**进入链接。
6. **CI 加 size 回归**：每条预设档位上限作为门禁，超过自动失败。

---

## 9. 与 Nim / OCaml / Go 的最终对照

| 维度 | Nim ARC | OCaml | Go | Xray auto | Xray min |
|---|---|---|---|---|---|
| GC | refcount | major/minor | tracing | per-coro Immix | bump |
| 协程 | — | — | M:N | M:N | — |
| 异常 | option/panic | exn | panic | try/catch | — |
| 反射 | 编译期 | — | runtime | runtime（可关） | — |
| 体积 | ~80 KB | ~300 KB | ~1.5 MB | ~600 KB | ~150 KB |

**Xray auto** 是"完整语言 + 自动剥未用特性" → 主流场景比 Go 更小、比 OCaml 略大；**Xray min** 是"主动放弃协程换体积" → 与 Nim 同档。

这就是 032 卷的全部承诺：**默认不大，按需可小，每一寸体积都是用户显式付的**。

---

## 10. 不在本卷范围

- **架构边界**：见 `031-aot-architecture.md`。
- **缺陷修复 / Phase 顺序**：见 `033-aot-implementation.md`。
- **JIT 后端的体积**（不分发给用户的 binary）。
- **stdlib 内部实现优化**（如 regex 引擎换轻量替代品）——独立任务。
- **加密/压缩 binary**（UPX 之类）——超出语言范畴。
