# 900 - 架构治理：世界级 C 项目的约束规则参考

## 一、你工作区已有项目的架构精华

### 1. QBE — 极简编译器典范（9,015 行）

**核心约束**：

| 规则 | QBE 做法 | xray 现状 |
|------|----------|-----------|
| 单一头文件 | `all.h` 包含所有类型定义 + 所有函数声明（632行） | 31个头文件交叉 include |
| 每个 .c 只 include all.h | 19个文件中17个只有1个 #include | xvm.c 有30个 #include |
| 函数声明按模块分组 | `/* cfg.c */` `/* mem.c */` 注释分隔 | 分散在各自头文件中 |
| 最大单文件 | parse.c 1433行（最大） | xvm.c 10276行 |
| 无嵌套目录 | 全部在根目录 | 22个子目录 + 深层嵌套 |
| 零重复数据结构 | 所有 struct 定义在 all.h 中集中 | 类型分散在 ~20 个头文件 |
| Pass 接口统一 | 每个 pass 是 `void xxx(Fn *)` | pass 签名五花八门 |

**可借鉴的规则**：
- **R1**: 所有 pass 函数的签名统一为 `void pass_name(Fn *)` — 强迫每个 pass 只做一件事
- **R2**: `ops.h` 用 X-Macro 模式定义所有操作码，编译期生成 enum — 消除手动编号错误
- **R3**: `Target` 结构体用函数指针抽象后端差异（isel, emit, abi）— 后端可插拔

### 2. LuaJIT — 高性能 VM 的架构标杆（~80,000 行）

**核心约束**：

| 规则 | LuaJIT 做法 | xray 现状 |
|------|-------------|-----------|
| 命名前缀 | 全部 `lj_` 前缀，55个模块名清晰区分 | `xr_`/`xray_`/`xa_`/`xir_` 四套前缀 |
| 头文件极简 | lj_str.h 仅7个导出/31行，lj_func.h 仅7个导出/24行 | xarray.h 312行含10个 inline |
| 可见性控制 | `LJ_FUNC`/`LJ_FUNCA` 宏控制符号导出 | 无统一可见性宏 |
| 核心类型集中 | `lj_obj.h` 定义所有 GC 对象类型（1062行） | 类型分散在 object/ value/ gc/ |
| #include .h 拼接 | `lj_asm.c` include `lj_asm_arm64.h` 等后端 | 独立编译单元 |
| 最大文件 | lj_record.c 2916行（trace recorder） | xvm.c 10276行（3.5x 大） |
| 模块间接口 | 每个 lj_xxx.h 只暴露必要的函数声明 | 头文件暴露过多内部细节 |

**可借鉴的规则**：
- **R4**: 每个头文件导出函数数 < 20，行数 < 100（冷路径头文件）
- **R5**: 平台差异用 `#include "lj_asm_arm64.h"` 条件编译拼入，保持单编译单元
- **R6**: `LJ_FUNC` 宏统一控制函数可见性（static build 下所有内部函数变 static）
- **R7**: 核心数据结构定义在单一文件（lj_obj.h），避免循环依赖

### 3. SQLite — 工业级可靠性的分层架构（~150,000 行）

**核心约束**（来自官方架构文档）：

| 层 | 职责 | 接口 |
|----|------|------|
| Interface | C API 入口 | sqlite3.h |
| Tokenizer + Parser | SQL → AST | tokenize.c + parse.y → Lemon |
| Code Generator | AST → 字节码 | 分散：select.c, insert.c, where*.c, expr.c |
| Bytecode Engine (VDBE) | 执行字节码 | vdbe.c（核心）+ vdbeaux.c + vdbeapi.c + vdbemem.c |
| B-Tree | 存储引擎 | btree.h |
| Page Cache | 磁盘 I/O | pager.h |
| OS Interface | 系统抽象层 | os.h (VFS) |

**可借鉴的规则**：
- **R8**: **严格分层，上层只通过头文件定义的接口访问下层**（btree.h 是 B-Tree 的唯一入口）
- **R9**: 代码生成器按 SQL 语句类型拆分文件（select.c, insert.c, delete.c）— 对应 xray 可以按语句类型拆 codegen
- **R10**: VDBE（字节码引擎）拆分为：核心执行(vdbe.c) + 辅助(vdbeaux.c) + API(vdbeapi.c) + 值操作(vdbemem.c) — 对应 xvm.c 拆分方案
- **R11**: 命名避免冲突：所有外部符号以 `sqlite3` 开头，API 以 `sqlite3_` 开头
- **R12**: **合并编译 (amalgamation)**：开发时多文件，发布时合并为 sqlite3.c 单文件 — 开发清晰 + 发布性能最优

### 4. MIR — 轻量级 JIT 的单人维护模式（~40,000 行）

| 规则 | MIR 做法 |
|------|----------|
| 超大文件 | mir-gen.c 10,018行、c2mir.c 14,258行 — 说明单人项目不拆也能维护 |
| 后端分离 | mir-gen-aarch64.c / mir-gen-x86_64.c 各自独立 |
| 头文件即接口 | mir.h (739行) 是唯一公开接口 |
| 数据结构内联 | mir.h 内定义所有核心类型 + inline 函数 |

**教训**：MIR 的大文件模式在单人维护下可行，但 **AI 协作开发不适合**——AI 每次只能看有限上下文，大文件导致 AI 频繁遗漏上下文。

---

## 二、推荐下载的世界级 C 项目

### 🥇 Tier 1: 必须参考（与 xray 高度相关）

#### 1. **Lua 5.4** — 最佳分层 VM 架构参考
```bash
# ~30,000 行，完美的 VM 架构分层
git clone https://github.com/lua/lua.git
```

**为什么必须看**：
- xray 的 GC、VM、值系统都参考了 Lua，但 Lua 5.4 在同等功能下**只有 30K 行**（xray 已 184K）
- 关键文件大小对比：

| 模块 | Lua 5.4 | xray | 比例 |
|------|---------|------|------|
| VM 执行 | lvm.c ~1800行 | xvm.c 10276行 | **5.7x** |
| GC | lgc.c ~1600行 | xcoro_gc.c 1711行 + ximmix.c等 | ~2x |
| 解析器 | lparser.c ~1800行 | xparse.c 3731行 | 2.1x |
| 字符串 | lstring.c ~300行 | xstring.c 1473行 | 4.9x |
| 表 | ltable.c ~850行 | xmap.c+xarray.c+xjson.c等 | >5x |

- Lua 的 `lvm.c` 只有 1800 行却实现了完整 VM — 它怎么做到的？答案是**极致的分层**：VM 不直接操作 GC，通过 `lgc.h` 的宏；不直接操作表，通过 `ltable.h` 的函数
- 关键架构约束：**每个 .h 文件是模块的唯一接口，没有 "_internal.h"**

#### 2. **wren** — 小而美的嵌入式语言 VM
```bash
# ~16,000 行，极其清晰的类 + GC + VM 架构
git clone https://github.com/wren-lang/wren.git
```

**为什么值得看**：
- 只有 16K 行但实现了：类系统 + 闭包 + GC + 模块 + Fiber(协程)
- 代码质量极高，Robert Nystrom（Crafting Interpreters 作者）写的
- **单一 wren.h 公共头文件**（145 行），内部头文件用 `wren_xxx.h`
- 最大文件 `wren_compiler.c` 约 3800 行、`wren_vm.c` 约 2200 行
- 命名规范一致：`wren` 前缀 + 模块名 + 函数名

#### 3. **CPython** — 大型 VM 项目的模块化治理
```bash
# ~500,000 行 C 代码，但模块化做得很好
git clone --depth 1 https://github.com/python/cpython.git
```

**为什么值得看**（不需要全看，只看以下目录）：
- `Python/ceval.c` — VM 主循环（~7000行），但用宏和 #include 拆分得很好
- `Objects/` 目录 — 每个类型一个文件（listobject.c, dictobject.c, ...）
- `Include/cpython/` vs `Include/` — 公共 API vs 内部 API 的分离
- **PEP 7**（C 编码规范）是 C 项目编码规范的标杆文档

### 🥈 Tier 2: 值得参考（某个方面特别优秀）

#### 4. **Redis** — 事件驱动 + 模块化的经典
```bash
# ~100,000 行，事件循环 + 数据结构 + 模块系统
git clone --depth 1 https://github.com/redis/redis.git
```

**值得看的点**：
- `src/server.h` 是唯一的大头文件（所有结构体定义），类似 QBE 的 all.h
- `src/ae.c/ae.h` — 极简事件循环抽象（200行），平台差异用 `#include "ae_epoll.c"` / `ae_kqueue.c`
- 模块 API：`redismodule.h` 是外部模块的唯一接口（稳定 ABI）
- 命名规范：`redis` 前缀 + 功能名，统一且不冲突

#### 5. **chibicc** — 极简 C 编译器
```bash
# ~10,000 行，完整的 C11 编译器
git clone https://github.com/rui314/chibicc.git
```

**值得看的点**：
- 10K 行实现完整 C11 编译器 — 极致的代码精简
- 单人项目但架构清晰：tokenize.c → parse.c → codegen.c → main.c
- 所有结构体定义在 `chibicc.h` 单一头文件中
- 作者 Rui Ueyama（前 Google，LLVM lld 主要作者）

#### 6. **libuv** — 跨平台异步 I/O
```bash
# ~30,000 行，事件驱动 + 跨平台抽象
git clone --depth 1 https://github.com/libuv/libuv.git
```

**值得看的点**：
- 公共 API 只有 `include/uv.h`
- 平台抽象层：`src/unix/` vs `src/win/` 各自实现相同接口
- 头文件分层：`uv.h`（公共） → `uv-common.h`（内部共享） → `internal.h`（平台内部）
- 与 xray 的 netpoll/coro I/O 层高度相关

### 🥉 Tier 3: 扩展视野

#### 7. **Cosmopolitan libc** — 极致的 C 工程
```bash
git clone --depth 1 https://github.com/jart/cosmopolitan.git
```
- Justine Tunney 的作品，C 工程的艺术
- 编译期条件消除、零成本抽象的 C 语言实践

#### 8. **TinyCC (tcc)** — 最小的 C 编译器
```bash
git clone https://repo.or.cz/tinycc.git
```
- ~50K 行实现完整 C 编译器 + 链接器
- 对比 chibicc 更完整，对比 GCC/LLVM 更精简

---

## 三、从参考项目中提取的架构约束规则清单

### 第一类：结构约束（防止膨胀）

| 编号 | 规则 | 来源 | 应用场景 |
|------|------|------|----------|
| S1 | **单文件不超过 3000 行** | LuaJIT（最大 2916 行） | 所有 src/ 文件 |
| S2 | **头文件导出函数 < 20 个** | LuaJIT（lj_str.h 7 个） | 模块公共接口 |
| S3 | **头文件 < 150 行**（纯接口头文件） | LuaJIT/Wren | 非数据结构定义头文件 |
| S4 | **单个 #include 链深度 < 5** | QBE（深度 1） | 避免编译级联 |
| S5 | **热路径文件可用 .inc 拆分** | LuaJIT（lj_asm_arm64.h） | VM dispatch, codegen |

### 第二类：接口约束（防止耦合）

| 编号 | 规则 | 来源 | 应用场景 |
|------|------|------|----------|
| I1 | **每个模块只有一个公共头文件** | SQLite（btree.h, pager.h） | 模块间通信 |
| I2 | **禁止 `_internal.h` 被外部模块 include** | Lua（无 internal 头文件） | 模块封装 |
| I3 | **上层不能 include 下层的 _internal** | SQLite 分层 | 依赖方向 |
| I4 | **跨模块调用必须通过头文件声明的函数** | 所有项目 | 避免隐式依赖 |
| I5 | **可见性宏**：内部函数用 `XR_INTERNAL`，API 用 `XRAY_API` | LuaJIT（LJ_FUNC） | 符号导出控制 |

### 第三类：命名约束（防止冲突）

| 编号 | 规则 | 来源 | 应用场景 |
|------|------|------|----------|
| N1 | **统一前缀**：公共 `xray_`，内部 `xr_`，静态 `static` | SQLite（sqlite3/sqlite3_）| 全局 |
| N2 | **模块名在函数名中**：`xr_gc_mark()`，`xr_vm_dispatch()` | LuaJIT（lj_gc_、lj_str_）| 全局 |
| N3 | **宏前缀统一 `XR_`**，无例外 | 所有项目 | 全局 |
| N4 | **文件名反映模块**：`x<module>.c` 或 `x<module>_<sub>.c` | Lua（l前缀）| 文件命名 |

### 第四类：Pass/Pipeline 约束（防止 pass 间隐式耦合）

| 编号 | 规则 | 来源 | 应用场景 |
|------|------|------|----------|
| P1 | **Pass 函数签名统一**：`void pass_name(Fn *)` | QBE | XIR pass, 编译器 pass |
| P2 | **Pass 不持有跨 pass 状态** | QBE/Cranelift | 优化管线 |
| P3 | **Pass 顺序在 pipeline 定义处集中声明** | QBE main.c | 避免隐式 pass 依赖 |

### 第五类：AI 协作约束（防止 AI 退化）

| 编号 | 规则 | 来源 | 应用场景 |
|------|------|------|----------|
| A1 | **每次 PR 必须包含"删了什么"** | 经验 | 防止代码只增不减 |
| A2 | **新增 > 200 行需要说明为什么不能更短** | 经验 | 防止 AI 冗余生成 |
| A3 | **新增跨模块依赖需要用户确认** | 经验 | 防止耦合蔓延 |
| A4 | **TODO/FIXME 必须带日期和原因** | 经验 | 防止遗忘 |
| A5 | **定期运行"瘦身审计"脚本** | 经验 | 防止熵增 |

---

## 四、xray 的具体适配建议

### 4.1 短期（可立即执行）

**对照 LuaJIT 的头文件极简原则**：

xray 当前的 `xcompiler.h` / `xemit.h` / `xcompiler_context.h` 被 31 个文件 include。
LuaJIT 的 `lj_parse.h` 只有 2 个导出函数（17行）。

建议：将编译器内部接口收敛为：
- `xcompiler.h` — 公共接口（< 20 个函数）
- `xcompiler_internal.h` — 内部接口（只被 frontend/codegen/*.c include）

**对照 QBE 的 pass 统一签名**：

xray 的 XIR pass 已经比较好了（`void xir_pass_xxx(XirFunc *)`）。但编译器 pass 不统一。
建议统一编译管线 pass 签名。

### 4.2 中期（拆分大文件）

**对照 SQLite 的 VDBE 拆分**：

SQLite 将虚拟机拆为 4 个文件：vdbe.c + vdbeaux.c + vdbeapi.c + vdbemem.c

xray 的 `xvm.c`（10276行）可以类似拆分：
- `xvm_dispatch.c` — 主循环 + 算术/逻辑
- `xvm_object.c` — 对象操作（对应 vdbeaux）
- `xvm_builtin_dispatch.c` — 内置方法分发（对应 func.c）
- `xvm_value.c` — 值操作（对应 vdbemem）

**对照 LuaJIT 的 .h 拼接模式**：

```c
// xvm_dispatch.c（主文件）
#include "xvm_internal.h"
// ... 主循环 ...

// 拆分到这些 .inc 文件中，但仍然是同一编译单元
#include "xvm_object.inc"
#include "xvm_coro.inc"
```

### 4.3 长期（架构重构）

**对照 Lua 5.4 的极致分层**：

Lua 5.4 的 lvm.c 为什么只有 1800 行？因为：
1. 值操作全部在 `lobject.c`（500行）
2. 字符串操作全部在 `lstring.c`（300行）
3. 表操作全部在 `ltable.c`（850行）
4. GC barrier 是宏，在 `lgc.h` 中定义
5. VM 只做 dispatch + 指令语义，不做类型转换和对象操作

xray 的 xvm.c 之所以 10K 行，是因为它直接做了太多事：
- 内联了属性查找逻辑
- 内联了类型转换逻辑
- 内联了 GC barrier
- 内联了内置方法分发

目标：让 xvm.c 像 Lua 的 lvm.c 一样，只做 dispatch + 指令语义。

---

## 五、推荐下载优先级

| 优先级 | 项目 | 行数 | 下载命令 | 主要学习点 |
|--------|------|------|----------|------------|
| **P0** | Lua 5.4 | 30K | `git clone https://github.com/lua/lua.git` | VM 分层、GC 接口、头文件极简 |
| **P0** | wren | 16K | `git clone https://github.com/wren-lang/wren.git` | 类系统、Fiber、整体架构 |
| P1 | chibicc | 10K | `git clone https://github.com/rui314/chibicc.git` | 极简编译器架构 |
| P1 | Redis | 100K | `git clone --depth 1 https://github.com/redis/redis.git` | 事件循环、模块化、命名 |
| P2 | CPython | 500K | `git clone --depth 1 https://github.com/python/cpython.git` | 大型 VM 治理、PEP 7 |
| P2 | libuv | 30K | `git clone --depth 1 https://github.com/libuv/libuv.git` | 跨平台异步 I/O 抽象 |
| P3 | chibicc | 10K | 已列 | 编译器极简 |
| P3 | tcc | 50K | `git clone https://repo.or.cz/tinycc.git` | 完整编译器+链接器 |

**你已有的项目中**，QBE 和 LuaJIT 是最值得深入研究的——它们分别代表了"极简编译器"和"高性能 VM"的架构标杆。
