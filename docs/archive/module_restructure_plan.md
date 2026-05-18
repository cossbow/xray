# 模块重构迁移计划

**目标**：消除 `scripts/check_architecture.sh` 报告的 14 处跨层反向依赖，在不推倒重来的前提下把模块划分调整到"先进合理"。

**原则**：
- 改动最小化，每一步独立可回滚
- 每步完成后必须通过 `scripts/run_regression_tests.sh`
- 每步合并前 `scripts/check_architecture.sh` 的错误数只能减少、不能增加

**现状基线（来自 `scripts/check_architecture.sh`）**：
- 7 errors、45 warnings
- runtime → vm：10 处
- runtime → jit：2 处
- runtime → frontend：2 处

---

## 升级清单总览

| # | 名称 | 影响规模 | 风险 | 预估工时 | 消除反向边 |
|---|------|----------|------|----------|-----------|
| 1 | `XrClosure` 下沉到 `runtime/closure/` | 48 文件 | 中 | 0.5 d | 5 处 (runtime/class, runtime/object→vm) |
| 2 | `Isolate` opaque 化（**不整体上移**） | 50+ 头文件，但多为删 include | 中 | 1 d | 5 处 (runtime/*→vm/xvm_state) |
| 3 | GC ↔ VM 契约反转（StackMap + 根扫描回调） | 3 模块 | 高 | 1.5 d | 2 处 (gc→vm, gc→jit) |
| 4 | Type 系统 Symbol 解耦 | 3 文件 | 低 | 0.5 d | 2 处 (value→frontend) |
| 5 | `vm_value_to_string` / `xchunk debug` 下沉 | 2 文件 | 低 | 0.5 d | 2 处 (value→vm) |

**总计预估**：4 天有效开发时间（不含 review + 回归测试）。

---

## 升级 1：`XrClosure` 下沉到 `runtime/closure/`（L3）

### 动机
`XrClosure` 是"函数对象"的运行时表示，属于对象系统的一等公民，却被定义在 `src/vm/xvm_state_frame.h:26-31`。这导致：
- `runtime/class/xclass_from_descriptor.c` 为用 `xr_vm_closure_new` 反向 include `vm/xvm.h`
- `runtime/class/xmethod.c` / `xclass_builder.c` 同样
- `runtime/object/xarray.c` / `xmap.c` / `xset.c` 也依赖 `XrClosure`（用于容器内函数值）

全代码库 48 个文件引用 `XrClosure`，其中 runtime/ 层就有 10+ 个，这是**结构性错位**。

### 目标布局
```
src/runtime/closure/
├── xclosure.h         // XrClosure 结构、xr_closure_new 等 API
├── xclosure.c         // 实现
└── xclosure_internal.h // 内部字段（如果需要）
```

### 迁移步骤

1. **新建目录与骨架**
   ```bash
   mkdir -p src/runtime/closure
   ```
2. **迁移定义**
   - `XrClosure` struct 从 `@/Users/xuxinglei/workspace/xray-lang/xray/src/vm/xvm_state_frame.h:26-31` 搬到 `src/runtime/closure/xclosure.h`
   - `xr_vm_closure_new` 从 `src/vm/` 对应 .c 文件搬到 `src/runtime/closure/xclosure.c`
   - **重命名**为 `xr_closure_new`（`xr_vm_*` 前缀语义错）；在 vm/ 保留 thin wrapper `xr_vm_closure_new = xr_closure_new` 过渡
3. **更新 include**
   - 全部 48 个引用者：删除 `#include "../../vm/xvm.h"` → 改为 `#include "../../closure/xclosure.h"`（runtime 内）或 `"runtime/closure/xclosure.h"`（vm/app 内）
   - `vm/xvm_state_frame.h` 改为 `#include "runtime/closure/xclosure.h"`
4. **CMakeLists.txt 更新**
   - 把新 `src/runtime/closure/*.c` 加入 `libxray_core` source 列表
5. **验证**
   - `cmake --build build -j8`
   - `ctest --output-on-failure --test-dir build`
   - `scripts/check_architecture.sh` 应看到 Q-12 reduce

### 风险与应对
- **Closure 有循环引用 vm?**：`XrClosure` 内通常只持 `XrFunctionProto*`、upvalues 数组，proto 属于 runtime/object，不循环。先确认 struct 字段。
- **过渡期别名**：保留一周的 `xr_vm_closure_new` alias，避免一次性改 48 个文件产生冲突
- **回滚**：单文件可逆——删除新建目录，git revert

---

## 升级 2：`Isolate` opaque 化（而非上移）

### 重要认识纠正
原建议是"把 Isolate 上移到 L6"。但 `Isolate` 在 runtime/ 下被 50+ 个头文件引用（像 Lua 的 `lua_State`、V8 的 `Isolate`）。**它本质是贯穿所有层的"执行上下文句柄"，不是应用层容器。**

### 真正的问题
不是 Isolate 所在的层错，是 **Isolate 的"内部结构"被过度泄漏**：
- `src/runtime/xisolate_internal.h:36` 暴露 `XrVMState` 完整结构，导致任何 include 它的文件都拖进 vm/
- 很多使用者只需要 `Isolate *` 指针，根本不需要看内部字段

### 目标设计
分离 handle 和实现：
```
src/runtime/xisolate.h           // opaque type: typedef struct XrIsolate XrIsolate;
                                 // 只有访问器 API，无 struct 定义
src/runtime/xisolate_internal.h  // 完整 struct 定义（含 VMState、gc、module 等）
                                 // 只有 isolate 模块自己 include
```

### 迁移步骤

1. **审计**
   ```bash
   grep -rln 'xisolate_internal\.h' src/ | wc -l  # 期望：从当前 ~40 降到 <10
   ```
2. **在 `xisolate.h`（新建或现有 `xisolate_api.h` 重命名）中只留 opaque type**
   ```c
   typedef struct XrIsolate XrIsolate;  // forward declaration only
   // 所有外部访问必须通过 xr_isolate_get_xxx() 访问器
   ```
3. **为 runtime/ 各头文件改 include**
   - 只需要 `XrIsolate*` 类型的：include `xisolate.h`（opaque）
   - 真需要读内部字段的（xgc.c 等少数）：仍 include `xisolate_internal.h`
4. **访问器补齐**
   - 每个内部字段访问（`isolate->vm_state->X`）改为 `xr_isolate_get_vm_state(isolate)->X`
   - 已有部分访问器（见 `src/runtime/xisolate_api.h`），补完整
5. **把 `xray_isolate.h`（public）的内部 include 清理**
   - `runtime/xstrbuf.c:18`、`module/xbytecode_io.c:18`、`api/xisolate_params.c:17` 应改为 include 内部头
   - 规则："内部代码只 include 内部头；`include/xray_*.h` 只给嵌入者用"

### 风险与应对
- **访问器 overhead**：Release 下 inline（访问器用 `static inline` 在 opaque 头里已有位置时例外），或关键路径保留直接访问（带注释说明）
- **过渡期**：允许 `xisolate_internal.h` 仍被多处 include，只要每次只清一个文件
- **回滚**：每个文件独立改，每次 commit 只动 ≤ 5 个文件

---

## 升级 3：GC ↔ VM 契约反转

### 动机
`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/gc/xcoro_gc.c:25` include `vm/xvm_state.h`（扫栈需要 frame 布局）
`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/gc/xcoro_gc.c:32` include `jit/xir_codegen.h`（精确栈扫描需要 StackMap）

这是**客观存在的双向信息需求**，不能靠搬家解决，必须用回调反转。

### 目标设计
在 `runtime/gc/` 定义**纯契约**，VM/JIT 实现并注册：

```c
// src/runtime/gc/xgc_roots.h（新建）

/* VM/JIT 栈帧布局的数据契约 */
typedef struct XrStackMapEntry {
    uint32_t pc_offset;
    uint32_t num_slots;
    const uint8_t *live_bitmap;
} XrStackMapEntry;

typedef struct XrStackMapTable { /* ... */ } XrStackMapTable;

/* 根扫描回调：VM/JIT 注册，GC 调用 */
typedef void (*XrRootScanFn)(void *ctx, void (*visit)(void *, XrValue *));

typedef struct XrRootScanner {
    XrRootScanFn scan;
    void *ctx;
} XrRootScanner;

XR_FUNC void xr_gc_register_root_scanner(XrCoroGC *gc, XrRootScanner scanner);
```

### 迁移步骤

1. **新建 `src/runtime/gc/xgc_roots.h`**，把 `XrStackMapTable` 从 `jit/xir_codegen.h` 搬过来
2. **`jit/xir_codegen.h` include `runtime/gc/xgc_roots.h`**（只用不定义）
3. **xcoro_gc.c 删除 `#include "../../vm/xvm_state.h"`**
   - 改为通过 `XrRootScanner` 回调扫栈
   - VM 启动时注册自己的 scanner：`xr_gc_register_root_scanner(gc, (XrRootScanner){.scan = vm_scan_stack, .ctx = vm_state})`
4. **验证**：GC 压力测试（`tests/regression` 里有 GC 压力测试用例），ASan 跑一遍

### 风险与应对
- **性能回归**：回调比直接访问慢。关键路径（minor GC）用 hot-path inline。ASan 之外再跑一次 perf 回归 `scripts/perf_regression.sh`
- **并发安全**：scanner 注册需在 GC 启动前完成，运行期不可变
- **回滚**：改动集中在 xcoro_gc.c + vm 初始化代码，diff 小可 revert

---

## 升级 4：Type 系统 Symbol 解耦

### 动机
`@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xtype.c:12` 和 `xtype_generic.c:18` include `frontend/analyzer/xanalyzer_symbol.h`，把编译期符号结构泄漏到运行期 XrType。

### 目标设计
运行期 XrType 只持 opaque handle，不认识 analyzer 内部结构：

```c
// src/runtime/value/xtype.h
typedef struct XrTypeSymbolHandle XrTypeSymbolHandle;  // opaque

typedef struct XrType {
    /* ... 其他字段 ... */
    XrTypeSymbolHandle *symbol;  // 不再是 XrAnalyzerSymbol*
} XrType;

// 如需读 symbol 名字、位置，通过访问器：
XR_FUNC const char *xr_type_symbol_name(const XrTypeSymbolHandle *h);
```

analyzer 在自己模块里实现：
```c
// src/frontend/analyzer/xanalyzer_symbol.c
struct XrTypeSymbolHandle {
    XrAnalyzerSymbol *impl;  // 真正的内部结构
};
```

### 迁移步骤

1. 在 `runtime/value/xtype.h` 声明 `XrTypeSymbolHandle`（opaque）
2. 把 `XrType` 中的 symbol 字段改为 `XrTypeSymbolHandle *`
3. 删除 `xtype.c` / `xtype_generic.c` 里对 `xanalyzer_symbol.h` 的 include
4. analyzer 模块里实现 handle wrapper
5. 验证：类型相关测试 + 泛型测试

### 风险与应对
- **AOT/序列化**：如类型信息要持久化，handle 序列化策略需确认
- **回滚**：改动仅 3 文件 + analyzer 接口，小

---

## 升级 5：函数摆位修正（零散）

### 5a. `vm_value_to_string` 下沉
- `@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xvalue_print.c:34` include `vm/xvm_internal.h`
- `vm_value_to_string()` 从 vm/ 搬到 `runtime/value/xvalue_format.c`
- 重命名为 `xr_value_to_string`（去 `vm_` 前缀）

### 5b. `xchunk debug` helper 下沉
- `@/Users/xuxinglei/workspace/xray-lang/xray/src/runtime/value/xchunk.c:22` include `vm/xdebug.h`
- 确认 `xchunk.c` 调用了哪些 debug helper：如果只是行号映射这类数据访问，把它们搬到 `runtime/value/xchunk_debug.c`

---

## 执行策略建议

### 推荐顺序

1. **第 1 步：升级 4**（Type 系统解耦）——3 文件、低风险、先试水
2. **第 2 步：升级 5**（函数搬家）——2 文件、低风险
3. **第 3 步：升级 1**（XrClosure 下沉）——48 文件但机械操作，引入新模块
4. **第 4 步：升级 3**（GC 契约反转）——设计难点，但升级 1、2 完成后剩下的反向边就只剩 GC 这两条最典型的
5. **第 5 步：升级 2**（Isolate opaque 化）——影响面最广，最后做以免和其他升级产生冲突

### 每一步的验收标准
```bash
# 必须全部通过
cmake --build build -j8 && ctest --output-on-failure --test-dir build
scripts/run_regression_tests.sh
cmake -B build-asan -DENABLE_ASAN=ON -DBUILD_TESTS=ON && cmake --build build-asan -j8 && ctest --test-dir build-asan

# 架构检查：错误数只能减少
bash scripts/check_architecture.sh | tee /tmp/arch_after.txt
# 对比 baseline /tmp/arch_before.txt，errors 必须减少
```

### 基线记录
开工前先保存基线：
```bash
bash scripts/check_architecture.sh > docs/engineering/_arch_baseline_$(date +%Y%m%d).txt
```

### 并行度
- 升级 1、4、5 可并行（独立文件集）
- 升级 2、3 需要串行（都改 isolate/gc 接口）

---

## 非目标（明确不做的事）

1. **不拆 `xvm.c`（7777 行）**：这是另一件事（代码重构 ≠ 架构调整），不放在本计划
2. **不改 `.c` 文件规模超限的其他 4 个文件**：同上
3. **不做 `../../` 相对路径改造**：等模块重构完成后再整体做 CMake include path
4. **不提升 static 比例**：等模块边界清晰后才有意义做

---

## 回滚策略

每个升级都是一个独立 commit（或 PR）。回滚手段：
- Git revert 对应 commit
- 升级 1、3、4 使用 "add alias 过渡"，老名字保留 1-2 周
- 回归测试失败时立即 revert，不强行修

---

## 跟踪

建议在 `module_restructure_progress.md` 记录进度（每完成一步写一行）。
