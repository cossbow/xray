# XIR JIT 多后端实施方案

> **作者**: Cascade (pair-programming with Xinglei Xu)
> **日期**: 2026-04-23
> **状态**: Draft
> **参考**: MIR (vnmakarov/mir), QBE (c9x/qbe), LuaJIT (LuaJIT/LuaJIT)

---

## 1. 现状分析

### 1.1 当前架构

```
                    ┌─────────────────────────────────────────────────┐
                    │              XIR Pipeline (架构无关)             │
                    │  Builder → SSA → GVN → LICM → DCE → RegAlloc   │
                    └──────────────────┬──────────────────────────────┘
                                       │
                           ┌───────────┴───────────┐
                           │  #ifdef __aarch64__    │  编译期选择
                           │  #elif __x86_64__      │
                           └───────────┬───────────┘
                    ┌──────────────────┼──────────────────┐
                    ▼                                     ▼
            xir_codegen.c (ARM64)               xir_codegen_x64.c (x64)
            xir_codegen_call.c                  xir_codegen_x64_call.c
            xir_codegen_mem.c                   (无 mem 分拆)
            xir_arm64.c/h (encoder)             xir_x64.c/h (encoder)
            xir_target_arm64.c                  xir_target_x64.c
            xir_codegen_internal.h              xir_codegen_x64_internal.h
```

**文件清单** (`src/jit/` 下):

| 层次 | ARM64 文件 | x64 文件 | 共享文件 |
|------|-----------|----------|---------|
| **Target 描述** | `xir_target_arm64.c` | `xir_target_x64.c` | `xir_target.h` (XirTarget 结构体) |
| **指令编码** | `xir_arm64.c/h` | `xir_x64.c/h` | — |
| **Codegen 主体** | `xir_codegen.c` (2545行) | `xir_codegen_x64.c` (2521行) | `xir_codegen.h` (XirCodegenResult) |
| **Codegen 子模块** | `xir_codegen_call.c`, `xir_codegen_mem.c` | `xir_codegen_x64_call.c` | — |
| **Codegen 内部头** | `xir_codegen_internal.h` | `xir_codegen_x64_internal.h` | — |
| **平台代码分配** | — | — | `xir_code_alloc.c/h` |
| **JIT 入口** | — | — | `xir_jit.c` (#ifdef 分派) |

### 1.2 现有问题

1. **平台覆盖不全**: `xir_code_alloc.c` 只支持 macOS (MAP_JIT) 和 Linux (mmap/mprotect)，**无 Windows 支持** (VirtualAlloc/VirtualProtect)
2. **codegen 硬编码 ARM64**: `xir_codegen_internal.h` 直接 `#include "xir_arm64.h"`，常量全是 ARM64 寄存器编号（`A64_X16` 等）
3. **x64 offset 乘 4 bug**: `xir_jit.c:510-512` 对 `fast_entry_offset` 和 `resume_entry_offset` 统一乘以 4（ARM64 指令粒度），x64 的 offset 已经是字节单位，乘 4 会导致错误偏移
4. **x64 icache flush**: `xir_code_flush_icache()` 的 x64 分支是 no-op（正确），但 Windows ARM64 需要 `FlushInstructionCache()` 未处理
5. **XirTarget 只有数据没有函数指针**: 不像 QBE 的 `Target` 结构体包含 `isel`/`emitfn`/`abi1` 等回调
6. **调用约定绑死**: x64 后端隐式使用 System V ABI（`rdi, rsi, rdx, rcx, r8, r9`），Windows x64 用 `rcx, rdx, r8, r9` + 32 字节 shadow space

### 1.3 好的基础

- **`XirTarget` 抽象已存在**，寄存器表和栈帧布局已参数化
- **codegen 结果统一**: `XirCodegenResult` 对两个后端是完全相同的返回类型
- **优化 pass 全部架构无关**: Builder, SSA, GVN, LICM, DCE, RegAlloc 不依赖具体后端
- **文件拆分模式成熟**: ARM64 codegen 按 call/mem 拆分，x64 按 call 拆分，均有独立 internal header

---

## 2. 目标平台矩阵

| 平台 | x86-64 | ARM64 |
|------|--------|-------|
| **macOS** | ❌ 不支持 (Intel Mac 已停产) | ✅ 现有主力 |
| **Linux** | 🟡 代码存在，需验证 | 🟡 代码基本可用，需验证 |
| **Windows** | 🔴 需新增 | 🔴 需新增 |

短期目标（Phase 1-2）: **macOS ARM64 + Linux x86-64 + Linux ARM64**
中期目标（Phase 3-4）: **+ Windows x86-64 + Windows ARM64**

---

## 3. 参考代码分析

### 3.1 MIR — 后端 dispatch 与内存分配

**文件**: `参考代码/mir/mir-gen.c:313-332`

```c
#if defined(__x86_64__) || defined(_M_AMD64)
#include "mir-gen-x86_64.c"
#elif defined(__aarch64__)
#include "mir-gen-aarch64.c"
// ... ppc64, s390x, riscv64
#else
#error "undefined or unsupported generation target"
#endif
```

**关键设计点**:
- 编译期 `#include` 拉入后端代码（和 xray 的 `#ifdef` 分派本质相同）
- 后端通过同名 `static` 函数实现多态（如 `target_call_used_hard_reg_p`）
- **每个后端自带 ABI 处理**: `mir-gen-x86_64.c` 内部用 `#ifdef _WIN32` 区分 SysV/Win64

**文件**: `参考代码/mir/mir-code-alloc-default.c` (91 行，三路内存抽象)

```c
#ifndef _WIN32
  // POSIX: mmap/munmap/mprotect
  #if defined(__APPLE__) && defined(__aarch64__)
    // macOS: MAP_JIT + pthread_jit_write_protect_np
  #else
    // Linux/BSD: mmap + mprotect
  #endif
#else
  // Windows: VirtualAlloc/VirtualFree/VirtualProtect
  #define WIN32_LEAN_AND_MEAN
  #include <windows.h>
#endif
```

**文件**: `参考代码/mir/mir-code-alloc.h` (函数指针抽象)

```c
typedef struct MIR_code_alloc {
  void *(*mem_map)(size_t, void *);
  int (*mem_unmap)(void *, size_t, void *);
  int (*mem_protect)(void *, size_t, MIR_mem_protect_t, void *);
  void *user_data;
} *MIR_code_alloc_t;
```

**xray 可借鉴**:
- 内存分配的函数指针模式太重了（xray 用编译期 `#ifdef` 更简单高效）
- 但 MIR 的三路 `#ifdef` 模板可以直接复制到 `xir_code_alloc.c`

### 3.2 QBE — Target 结构体与多 ABI 分离

**文件**: `参考代码/qbe/all.h:44-67`

```c
struct Target {
    char name[16];
    char apple;
    char windows;
    int gpr0, ngpr, fpr0, nfpr;
    bits rglob;
    int nrglob;
    int *rsave;
    int nrsave[2];
    bits (*retregs)(Ref, int[2]);   // 返回值寄存器
    bits (*argregs)(Ref, int[2]);   // 参数寄存器
    int  (*memargs)(int);           // 内存操作数支持
    void (*abi0)(Fn *);             // ABI lowering pass 1
    void (*abi1)(Fn *);             // ABI lowering pass 2
    void (*isel)(Fn *);             // 指令选择
    void (*emitfn)(Fn *, FILE *);   // 代码发射
    void (*emitfin)(FILE *);        // 文件收尾
};
```

**文件**: `参考代码/qbe/amd64/targ.c` — **同架构 3 个 Target 实例**

```c
Target T_amd64_sysv = {  .name = "amd64_sysv",  .abi1 = amd64_sysv_abi,  ... };
Target T_amd64_apple = { .name = "amd64_apple", .apple = 1,              ... };
Target T_amd64_win = {   .name = "amd64_win",   .windows = 1,
                          .abi1 = amd64_winabi_abi, .emitfn = amd64_winabi_emitfn, ... };
```

**文件**: `参考代码/qbe/main.c:54-121` — **通过 Target 函数指针驱动编译管线**

```c
T.abi0(fn);   // ABI lowering
T.abi1(fn);   // ABI lowering phase 2
T.isel(fn);   // instruction selection
T.emitfn(fn, outf);  // code emission
```

**xray 可借鉴**:
- **`XirTarget` 扩展为包含函数指针**，至少要有 `codegen()` 和 `flush_icache()`
- **同架构多 ABI**: `xir_target_x64_sysv` + `xir_target_x64_win` 分实例
- 全局 `Target T` 变量在初始化时赋值，整个管线通过回调驱动

### 3.3 LuaJIT — 平台代码管理

**文件**: `参考代码/LuaJIT/src/lj_mcode.c:42-154` — **最紧凑的跨平台 JIT 内存管理**

三层结构:
1. `lj_mcode_sync()` — icache flush (5 平台分支，18 行)
2. `mcode_alloc_at()` / `mcode_free()` / `mcode_setprot()` — 平台原语 (Windows / POSIX+MAP_JIT / POSIX)
3. `mcode_protect()` — W^X 安全策略切换

**文件**: `参考代码/LuaJIT/src/lj_target_x86.h:81-106` — **同文件 SysV/Win ABI 分支**

```c
#if LJ_ABI_WIN
  #define RSET_SCRATCH  (...)  // Windows caller-save set
  #define REGARG_GPRS   (RID_ECX|RID_EDX|RID_R8D|RID_R9D)
  #define REGARG_NUMGPR 4
#else
  #define RSET_SCRATCH  (...)  // SysV caller-save set
  #define REGARG_GPRS   (RID_EDI|RID_ESI|RID_EDX|RID_ECX|RID_R8D|RID_R9D)
  #define REGARG_NUMGPR 6
#endif
```

**xray 可借鉴**:
- `lj_mcode_sync()` 的 `FlushInstructionCache` Windows 路径
- `lj_mcode.c` 的 jump range 管理（x64 分支跳转需要在 ±2GB 范围内）
- 宏级 ABI 分支（编译期零开销）

---

## 4. 架构设计

### 4.1 扩展 XirTarget → XirBackend

在现有 `XirTarget`（纯数据）基础上增加 `XirBackend`（数据 + 行为），仿 QBE 的 `Target`。

```c
/* xir_target.h */

typedef struct XirBackend {
    const XirTarget   *target;     // register/frame description (existing)

    // --- Codegen callback ---
    XirCodegenResult (*codegen)(XirFunc *func, XirCodeAlloc *alloc);

    // --- Platform callbacks ---
    void (*flush_icache)(void *ptr, size_t size);

    // --- ABI info ---
    const int  *arg_gpr;           // integer argument registers (ordered)
    int         n_arg_gpr;         // count (6 for SysV, 4 for Win64)
    const int  *arg_fpr;           // FP argument registers
    int         n_arg_fpr;
    int         shadow_space;      // 0 for SysV/AAPCS, 32 for Win64
    bool        args_interleave;   // true for Win64 (int/fp share position slots)

    // --- Entry offset scaling ---
    int         entry_offset_scale; // 4 for ARM64 (instr), 1 for x64 (byte)
} XirBackend;

extern const XirBackend *xir_current_backend;  // set once at JIT init
```

### 4.2 后端实例

```c
/* xir_backend_arm64.c */
const XirBackend xir_backend_arm64 = {
    .target             = &xir_target_arm64,
    .codegen            = xir_codegen_arm64,
    .flush_icache       = xir_code_flush_icache_arm64,
    .arg_gpr            = (const int[]){A64_X0,A64_X1,...,A64_X7},
    .n_arg_gpr          = 8,
    .arg_fpr            = (const int[]){0,1,...,7},
    .n_arg_fpr          = 8,
    .shadow_space       = 0,
    .args_interleave    = false,
    .entry_offset_scale = 4,
};

/* xir_backend_x64_sysv.c */
const XirBackend xir_backend_x64_sysv = {
    .target             = &xir_target_x64,
    .codegen            = xir_codegen_x64,
    .flush_icache       = xir_code_flush_icache_x64,
    .arg_gpr            = (const int[]){X64_RDI,X64_RSI,X64_RDX,X64_RCX,X64_R8,X64_R9},
    .n_arg_gpr          = 6,
    .arg_fpr            = (const int[]){0,1,...,7},
    .n_arg_fpr          = 8,
    .shadow_space       = 0,
    .args_interleave    = false,
    .entry_offset_scale = 1,
};

/* xir_backend_x64_win.c (Phase 3) */
const XirBackend xir_backend_x64_win = {
    .target             = &xir_target_x64_win,   // 不同的 caller-save 集合
    .codegen            = xir_codegen_x64,        // 同一个 codegen，通过 backend 读 ABI
    .flush_icache       = xir_code_flush_icache_x64,
    .arg_gpr            = (const int[]){X64_RCX,X64_RDX,X64_R8,X64_R9},
    .n_arg_gpr          = 4,
    .arg_fpr            = (const int[]){0,1,2,3},
    .n_arg_fpr          = 4,
    .shadow_space       = 32,
    .args_interleave    = true,
    .entry_offset_scale = 1,
};
```

### 4.3 JIT 初始化时选择后端

```c
/* xir_jit.c - 替换现有的 #ifdef 分派 */

static const XirBackend *detect_backend(void) {
#if defined(__aarch64__)
    return &xir_backend_arm64;
#elif defined(__x86_64__) || defined(_M_AMD64)
  #if defined(_WIN32)
    return &xir_backend_x64_win;
  #else
    return &xir_backend_x64_sysv;
  #endif
#else
    return NULL;
#endif
}

XirJitState *xir_jit_init(XrayIsolate *isolate, int threshold) {
    const XirBackend *backend = detect_backend();
    if (!backend) return NULL;  // unsupported platform
    xir_current_backend = backend;
    xir_current_target = backend->target;
    // ... rest of init
}
```

### 4.4 统一 codegen 分派（修复 offset bug）

```c
/* xir_jit.c - 替换现有的 #if defined(__aarch64__) 块 */

    // Generate platform-specific machine code
    XirCodegenResult res = xir_current_backend->codegen(func, &jit->code_alloc);
    if (!res.success) { ... }

    // Store compiled entry point (use backend's offset scale)
    int scale = xir_current_backend->entry_offset_scale;
    proto->jit_entry = res.code;
    proto->jit_fast_entry = (char *)res.code + res.fast_entry_offset * scale;
    proto->jit_resume_entry = res.resume_entry_offset
        ? (char *)res.code + res.resume_entry_offset * scale : NULL;
```

### 4.5 平台内存抽象 (xir_code_alloc.c)

仿 MIR `mir-code-alloc-default.c` + LuaJIT `lj_mcode.c` 的三路分支:

```c
/* ========== Platform: Memory Map ========== */

#if defined(_WIN32)

#define WIN32_LEAN_AND_MEAN
#include <windows.h>

static XirCodePage *alloc_code_page(size_t min_size) {
    size_t size = min_size < XIR_CODE_PAGE_SIZE ? XIR_CODE_PAGE_SIZE : page_align(min_size);
    void *mem = VirtualAlloc(NULL, size, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    if (!mem) return NULL;
    // ... create XirCodePage
}

static void free_code_page(XirCodePage *page) {
    if (page) {
        VirtualFree(page->base, 0, MEM_RELEASE);
        xr_free(page);
    }
}

void xir_code_make_executable(void *ptr, size_t size) {
    DWORD old_prot;
    VirtualProtect(ptr, size, PAGE_EXECUTE_READ, &old_prot);
}

void xir_code_make_writable(void *ptr, size_t size) {
    DWORD old_prot;
    VirtualProtect(ptr, size, PAGE_READWRITE, &old_prot);
}

#elif defined(__APPLE__)

// 现有 MAP_JIT + pthread_jit_write_protect_np 代码不变

#else  // Linux / BSD / Other POSIX

// 现有 mmap + mprotect 代码不变

#endif
```

### 4.6 icache flush 统一

仿 LuaJIT `lj_mcode_sync()`:

```c
void xir_code_flush_icache(void *ptr, size_t size) {
#if defined(_WIN32)
    FlushInstructionCache(GetCurrentProcess(), ptr, size);
#elif defined(__APPLE__)
    sys_icache_invalidate(ptr, size);
#elif defined(__aarch64__)
    // Linux ARM64: manual dc cvau / ic ivau (现有代码)
    ...
#elif defined(__GNUC__) || defined(__clang__)
    __builtin___clear_cache((char *)ptr, (char *)ptr + size);
#else
    (void)ptr; (void)size;  // x86: coherent i-cache
#endif
}
```

---

## 5. 实施计划

### Phase 1: Linux 双架构启用（1-2 天）

**目标**: Linux x86-64 和 Linux ARM64 可以编译和运行 JIT。

**改动**:

1. **修复 x64 offset bug** (`xir_jit.c:510-512`)
   - `fast_entry_offset * 4` → `fast_entry_offset * scale`
   - 短期可先用 `#ifdef` 区分，Phase 2 改为 backend 字段

2. **验证 Linux ARM64**
   - `xir_code_alloc.c` 的 `#else` 分支（mmap + mprotect）在 Linux ARM64 已可用
   - icache flush 的 `#elif defined(__aarch64__)` 路径已有手动 dc cvau/ic ivau
   - 需要 CI 验证

3. **验证 Linux x86-64**
   - x64 codegen 使用 System V ABI，Linux 天然兼容
   - 需要 CI 验证

4. **CI 矩阵**
   - GitHub Actions 添加 `ubuntu-latest` (x86-64) 构建+测试

**不改动**: 架构不重构，只修 bug 和加 CI。

### Phase 2: 后端抽象层（3-5 天）

**目标**: 引入 `XirBackend` 抽象，消除 codegen 路径中的 `#ifdef`。

**改动**:

1. **新建 `xir_backend.h`** — 定义 `XirBackend` 结构体
2. **新建后端实例文件**:
   - `xir_backend_arm64.c` (合并现有 `xir_target_arm64.c`)
   - `xir_backend_x64_sysv.c` (合并现有 `xir_target_x64.c`)
3. **重构 `xir_jit.c`**:
   - `xir_jit_init()` 调用 `detect_backend()` 设置全局 backend
   - codegen 分派改为 `xir_current_backend->codegen()`
   - offset 乘法改用 `entry_offset_scale`
4. **`xir_code_alloc.c`**: icache flush 改用 `xir_current_backend->flush_icache()` 或保持独立（不强制绑定到 backend）
5. **CMakeLists.txt**: 按 target arch 条件编译后端文件（已有基础）

**不改动**: codegen 内部实现不改，只是外层分派重构。

### Phase 3: Windows 平台支持（5-7 天）

**目标**: Windows x86-64 可编译运行（不含 JIT），Windows x86-64 JIT 基本工作。

**改动**:

1. **`xir_code_alloc.c` 添加 Windows 路径**
   - `VirtualAlloc` / `VirtualFree` / `VirtualProtect`
   - 参照 MIR `mir-code-alloc-default.c` 和 LuaJIT `lj_mcode.c`

2. **`xir_code_flush_icache()` 添加 Windows 路径**
   - `FlushInstructionCache(GetCurrentProcess(), ptr, size)`

3. **新建 `xir_target_x64_win.c`**
   - 参照 QBE `amd64/winabi.c` 和 MIR `mir-gen-x86_64.c` 的 `#ifdef _WIN32` 分支
   - 寄存器分配表调整: RSI/RDI 变为 callee-save，XMM6-15 变为 callee-save
   - 栈帧增加 32 字节 shadow space

4. **新建 `xir_backend_x64_win.c`**
   - 参数寄存器: `RCX, RDX, R8, R9`
   - shadow_space = 32

5. **x64 codegen 调用约定参数化**
   - helper call 的参数传递从硬编码改为读 `xir_current_backend->arg_gpr[]`
   - 或者更实际: 用 `#ifdef _WIN32` 在 codegen 内部区分（仿 MIR 做法）

6. **`xjit_compile_queue.c` 线程相关**
   - Windows 线程 API（`CreateThread` / `WaitForSingleObject`）或统一用 `<threads.h>` / pthreads-win32

7. **CMakeLists.txt**
   - 添加 Windows 构建支持（已有 `if(WIN32)` 框架但未完善）

### Phase 4: Windows ARM64（3-5 天，Phase 3 之后）

**目标**: Windows ARM64 JIT 基本工作。

**改动**:

1. **新建 `xir_target_arm64_win.c`**
   - 和 Linux/macOS ARM64 大部分相同
   - x18 (TEB 指针) 不可用（xray 已不分配 x18，影响小）
   - 可能需要微调 vararg 处理

2. **新建 `xir_backend_arm64_win.c`**
   - 继承 ARM64 codegen
   - icache 使用 `FlushInstructionCache`

3. **SEH unwind info (可延后)**
   - Windows 要求动态代码注册 `RtlAddFunctionTable`
   - 第一版可不做（影响调试器栈回溯和异常传播，但基本功能不受影响）

### Phase 5: 稳定化与 CI（持续）

1. **CI 矩阵**:
   - `macos-14` (ARM64) — 已有
   - `ubuntu-latest` (x86-64)
   - `ubuntu-arm64` (self-hosted 或 QEMU)
   - `windows-latest` (x86-64)
   - `windows-arm64` (self-hosted，可延后)

2. **测试**:
   - 所有平台运行 `ctest --output-on-failure`
   - `--jit-force` 模式回归测试
   - 新增: `tests/unit/test_code_alloc_platform.c` — 测试 alloc/protect/flush 循环

3. **性能基准**:
   - 各平台 JIT 编译速度对比
   - 生成代码质量对比（code size, execution speed）

---

## 6. 文件变更总览

### 新增文件

| 文件 | Phase | 说明 |
|------|-------|------|
| `src/jit/xir_backend.h` | 2 | XirBackend 结构体定义 |
| `src/jit/xir_backend_arm64.c` | 2 | ARM64 后端实例 (macOS/Linux) |
| `src/jit/xir_backend_x64_sysv.c` | 2 | x64 SysV 后端实例 (Linux/macOS) |
| `src/jit/xir_backend_x64_win.c` | 3 | x64 Win64 后端实例 |
| `src/jit/xir_target_x64_win.c` | 3 | x64 Win64 寄存器/栈帧描述 |
| `src/jit/xir_backend_arm64_win.c` | 4 | ARM64 Windows 后端实例 |
| `src/jit/xir_target_arm64_win.c` | 4 | ARM64 Windows 目标描述 |

### 修改文件

| 文件 | Phase | 改动 |
|------|-------|------|
| `src/jit/xir_target.h` | 2 | 增加 `XirBackend` 或新头文件 |
| `src/jit/xir_jit.c` | 1→2 | Phase 1: 修 offset bug; Phase 2: 改用 backend dispatch |
| `src/jit/xir_code_alloc.c` | 3 | 添加 `_WIN32` 路径 |
| `src/jit/xir_code_alloc.h` | 3 | 添加 Windows 头文件条件包含 |
| `CMakeLists.txt` | 2→3 | 条件编译后端文件，Windows 构建支持 |
| `.github/workflows/ci.yml` | 1→5 | 扩展 CI 矩阵 |

### 删除/合并文件

| 文件 | Phase | 说明 |
|------|-------|------|
| `src/jit/xir_target_arm64.c` | 2 | 合并入 `xir_backend_arm64.c` |
| `src/jit/xir_target_x64.c` | 2 | 合并入 `xir_backend_x64_sysv.c` |

---

## 7. 设计决策记录

### D1: 编译期 #ifdef vs 运行时函数指针

**决定**: 混合方案 — 外层用函数指针（`XirBackend.codegen`），codegen 内部保留 `#ifdef`。

**理由**:
- MIR 完全用编译期 `#include` — 简单但不支持交叉编译
- QBE 完全用运行时 `Target` 函数指针 — 灵活但 QBE 是 AOT 编译器，代价不同
- **xray 的 codegen 内部** 有大量架构特定常量（寄存器编号、指令编码宏），用运行时分派不现实
- **xray 的 dispatch 层** 只在 `xir_jit.c` 调用一次 codegen，用函数指针零开销

### D2: 同架构多 ABI 的处理方式

**决定**: 新建 target 实例 + codegen 内部 `#ifdef`（仿 MIR 而非 QBE）。

**理由**:
- QBE 的方式（`winabi.c` 独立文件，700+ 行完整 ABI 实现）更干净，但 QBE 的 codegen 很薄（只是文本发射）
- xray 的 codegen 有 2500 行复杂的机器码生成，ABI 差异只影响 call 序列（~200 行）
- 用 `#ifdef _WIN32` 在 `xir_codegen_x64_call.c` 内部处理更实际（仿 MIR `mir-gen-x86_64.c` 做法）

### D3: 是否支持交叉编译 JIT

**决定**: 不支持，只编译当前平台的 JIT 后端。

**理由**:
- JIT 是运行时编译，只需要在运行平台工作
- AOT 编译器（`src/aot/`）如果要支持交叉目标，是独立的问题
- 所有参考项目（MIR, LuaJIT, V8）的 JIT 都只编译 host target

### D4: entry_offset_scale 还是让后端返回字节偏移

**决定**: 让两个后端都返回**字节偏移**（统一语义），ARM64 codegen 内部 `* 4`。

**理由**:
- 当前 ARM64 返回指令数（需 `* 4`），x64 返回字节数（不需 `* 4`），语义不一致是 bug 源
- 统一为字节偏移后，`xir_jit.c` 不需要 scale 字段，更简单
- 这是最小改动原则

---

## 8. 已知风险

| 风险 | 影响 | 缓解 |
|------|------|------|
| x64 codegen 不够成熟 ("Phase F.4.2") | Linux x64 JIT 可能功能不全 | 回退到 `--no-jit` 解释执行 |
| Windows 线程模型差异 | 后台编译队列 (`xjit_compile_queue.c`) 用 pthread | 用 C11 `<threads.h>` 或条件编译 |
| SEH unwind 缺失 | Windows 调试器无法回溯 JIT 栈帧 | 第一版可接受，后续用 `RtlAddFunctionTable` 补充 |
| 间歇性 SIGBUS (现有 ARM64 bug) | 跨平台前应先修复 | 独立排查（可能是 arena allocator 或并发 bug） |
| x64 ABI shadow space | 所有 helper call 需预留 32 字节栈空间 | 在 prologue 预分配或每次 call 前 sub rsp |

---

## 9. 参考代码速查

| 需求 | 最佳参考 | 文件 |
|------|---------|------|
| Windows VirtualAlloc/VirtualProtect | MIR | `mir-code-alloc-default.c:67-82` |
| Windows FlushInstructionCache | LuaJIT | `lj_mcode.c:50` |
| Target 结构体 + 函数指针 | QBE | `all.h:44-67` |
| 同架构多 ABI (SysV vs Win64) | QBE | `amd64/targ.c` (3 个 Target 实例) |
| Win64 调用约定实现 | QBE | `amd64/winabi.c` (764 行) |
| Win64 ABI 差异 (寄存器) | MIR | `mir-gen-x86_64.c:15-126` |
| ARM64 ABI (AAPCS64) | MIR | `mir-gen-aarch64.c:22-116` |
| 跨平台 icache flush (5路) | LuaJIT | `lj_mcode.c:42-60` |
| W^X 安全策略 | LuaJIT | `lj_mcode.c:156-213` |
| Jump range 管理 (x64 ±2GB) | LuaJIT | `lj_mcode.c:228-263` |
| 移植新后端检查清单 | MIR | `HOW-TO-PORT-MIR.md` |
| 后端 dispatch (编译期) | MIR | `mir-gen.c:313-332` |
| 后端 dispatch (运行时) | QBE | `main.c:54-121` |

---

## 10. 多平台测试策略

### 10.1 核心原则

> **一个 bug 在 A 平台发现，必须在 B/C/D 平台同步排查。**

JIT 后端的 bug 分三类，每类有不同的跨平台影响：

| Bug 类别 | 示例 | 跨平台影响 | 处理方式 |
|----------|------|-----------|---------|
| **共享逻辑 bug** | XIR 优化 pass（GVN/DCE/RegAlloc）错误 | **全平台都有** | 修一处，全平台自动受益 |
| **模式 bug** | codegen 的 CALL_C 序列少了一步 spill writeback | **其他后端可能有同类问题** | 修 A 后端时，同步检查 B 后端对应代码 |
| **平台特定 bug** | Windows VirtualProtect 参数错误 | **只影响该平台** | 仅在该平台修复，但需写平台特定测试 |

### 10.2 测试分层

```
┌────────────────────────────────────────────────────────────────────┐
│ L4: 平台专属测试 (Platform-Specific)                               │
│     code_alloc W^X 循环、icache flush、线程创建、SEH               │
│     只在对应平台运行                                                │
├────────────────────────────────────────────────────────────────────┤
│ L3: JIT 差分测试 (Differential)                                    │
│     --no-jit vs --jit-force 对比输出                               │
│     tests/jit/*.xr (87 个用例)                                     │
│     所有 JIT 平台必须跑                                             │
├────────────────────────────────────────────────────────────────────┤
│ L2: 回归测试 (Regression)                                          │
│     tests/regression/**/*.xr (311+ 个用例)                         │
│     默认模式(自动JIT) + --jit-force 模式各跑一遍                    │
│     所有平台必须跑                                                  │
├────────────────────────────────────────────────────────────────────┤
│ L1: 单元测试 (Unit)                                                │
│     ctest (63+ 个 C 单元测试)                                      │
│     所有平台必须跑                                                  │
└────────────────────────────────────────────────────────────────────┘
```

### 10.3 CI 矩阵设计

```yaml
# .github/workflows/ci.yml 扩展
strategy:
  fail-fast: false
  matrix:
    include:
      # ---- 现有 ----
      - os: macos-14            # Apple Silicon
        arch: arm64
        jit: true
        name: "macOS ARM64"

      # ---- Phase 1 新增 ----
      - os: ubuntu-latest       # GitHub 默认 x86-64
        arch: x86_64
        jit: true
        name: "Linux x64"

      # ---- Phase 1 新增 (ARM64 Linux) ----
      - os: ubuntu-24.04-arm    # GitHub ARM64 runner (公开预览)
        arch: arm64
        jit: true
        name: "Linux ARM64"

      # ---- Phase 3 新增 ----
      - os: windows-latest      # x86-64
        arch: x86_64
        jit: true
        name: "Windows x64"

      # ---- Phase 4 新增 (如果有 runner) ----
      # - os: windows-arm64
      #   arch: arm64
      #   jit: true
      #   name: "Windows ARM64"
```

每个矩阵条目运行:
1. `cmake --build` 构建
2. `ctest --output-on-failure` (L1)
3. `scripts/run_regression_tests.sh` (L2)
4. `scripts/run_jit_diff_tests.sh` — 如果 `jit: true` (L3)

### 10.4 JIT 差分测试脚本

**核心思想**: 同一个 `.xr` 文件，分别用 `--no-jit` 和 `--jit-force` 执行，比较输出。

```bash
# scripts/run_jit_diff_tests.sh (伪代码)
for test in tests/jit/*.xr; do
    # 基准: 解释执行
    expected=$($XRAY --no-jit "$test" 2>&1)
    rc_expected=$?

    # 被测: JIT 强制编译
    actual=$($XRAY --jit-force "$test" 2>&1)
    rc_actual=$?

    # 比较退出码和输出
    if [ "$rc_expected" != "$rc_actual" ] || [ "$expected" != "$actual" ]; then
        if grep -q "$test" tests/jit/known_failures.txt; then
            echo "XFAIL: $test"  # 已知失败，不计入失败
        else
            echo "FAIL: $test"
            FAIL_COUNT=$((FAIL_COUNT + 1))
        fi
    else
        echo "PASS: $test"
    fi
done
```

**`tests/jit/known_failures.txt`** 的作用:
- 记录已知的 JIT 特定失败（如 GC 交互 bug）
- CI 不会因已知失败而红
- **每个平台可以有自己的 known_failures 文件**:
  - `tests/jit/known_failures.txt` — 通用
  - `tests/jit/known_failures_linux_x64.txt` — Linux x64 特定
  - `tests/jit/known_failures_win_x64.txt` — Windows x64 特定

### 10.5 平台专属测试 (L4)

新建 `tests/unit/test_platform_jit.c`:

```c
/* test_platform_jit.c - Platform-specific JIT infrastructure tests */

/* T1: alloc → write → make_executable → execute → free 循环 */
static void test_code_alloc_roundtrip(void) {
    XirCodeAlloc alloc;
    xir_code_alloc_init(&alloc, 4096);

    // Allocate a page
    void *code = xir_code_alloc_reserve(&alloc, 64);
    assert(code != NULL);

    // Write a trivial function: return 42
    // (use platform-specific machine code bytes)
    #if defined(__aarch64__)
    uint32_t *p = code;
    p[0] = 0xD2800540;  // mov x0, #42
    p[1] = 0xD65F03C0;  // ret
    #elif defined(__x86_64__) || defined(_M_AMD64)
    uint8_t *p = code;
    p[0] = 0xB8; p[1]=42; p[2]=0; p[3]=0; p[4]=0; // mov eax, 42
    p[5] = 0xC3;                                     // ret
    #endif

    xir_code_make_executable(code, 64);
    xir_code_flush_icache(code, 64);

    // Execute it
    typedef int (*fn_t)(void);
    int result = ((fn_t)code)();
    assert(result == 42);

    xir_code_alloc_destroy(&alloc);
}

/* T2: W^X toggle 正确性 */
static void test_wx_toggle(void) {
    // make_writable → write → make_executable → execute
    // make_writable → overwrite → make_executable → execute new code
    // 验证两次结果不同
}

/* T3: icache flush 必要性 (ARM64/Windows ARM64) */
static void test_icache_flush_required(void) {
    // 写入代码 → 不 flush → 执行 → 可能得到旧值
    // 写入代码 → flush → 执行 → 一定得到新值
    // (x86 上两者都通过，ARM64 上不 flush 可能失败)
}
```

### 10.6 跨平台 Bug 同步流程

当某平台发现 JIT bug 时，执行以下检查清单:

```
┌─────────────────────────────────────────────────────────┐
│           Bug 发现 (例: Linux x64 SIGSEGV)              │
└───────────────────────┬─────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────┐
│ Step 1: 定位 bug 所在层                                  │
│                                                         │
│  共享层?  (xir_pass_*, xir_regalloc*, xir_builder_*)   │
│     → 所有平台都受影响，修一处即可                        │
│     → 但所有平台都需要跑测试验证                          │
│                                                         │
│  Codegen 层?  (xir_codegen_*.c)                         │
│     → 检查对应后端是否有同类模式                          │
│     → 见 Step 2                                          │
│                                                         │
│  Platform 层?  (xir_code_alloc.c, 线程相关)              │
│     → 只影响该平台，写平台特定测试                        │
└───────────────────────┬─────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────┐
│ Step 2: 模式 Bug 交叉检查                                │
│                                                         │
│  如果 bug 在 ARM64 codegen 的某个 XIR opcode 处理中:     │
│                                                         │
│  grep -n "case XIR_<OP>:" xir_codegen.c                 │
│  grep -n "case XIR_<OP>:" xir_codegen_x64.c             │
│                                                         │
│  逐行对比两个后端的实现:                                  │
│  - 是否都正确处理了 spill/reload?                         │
│  - 是否都正确保存/恢复了 caller-save 寄存器?              │
│  - 是否都正确处理了 GC safepoint writeback?               │
│  - 是否都正确处理了 deopt?                                │
│                                                         │
│  发现同类问题 → 一起修复                                  │
│  未发现 → 写入 commit message: "checked x64: not affected"│
└───────────────────────┬─────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────┐
│ Step 3: 写回归测试                                       │
│                                                         │
│  在 tests/jit/ 新增 .xr 测试文件                         │
│  该测试必须能在 --jit-force 下触发原 bug                  │
│  该测试会被所有平台的 CI 自动运行                          │
│                                                         │
│  如果是平台特定 bug:                                      │
│  在 tests/unit/test_platform_jit.c 新增 C 单元测试       │
└─────────────────────────────────────────────────────────┘
```

### 10.7 Codegen 对称性审计

每当一个后端新增或修改了某个 XIR opcode 的处理，应同步审计另一个后端。
建议维护一张**实现状态矩阵**:

```
# tests/jit/backend_status.md (或放入本文档)

| XIR Opcode         | ARM64 | x64 SysV | x64 Win | 备注 |
|--------------------|-------|----------|---------|------|
| XIR_ADD            | ✅     | ✅        | —       |      |
| XIR_SUB            | ✅     | ✅        | —       |      |
| XIR_CALL_C         | ✅     | ✅        | 🔴 需ABI | shadow space |
| XIR_CALL_KNOWN     | ✅     | ❌ disabled| —       | builder_call.c: if(false&&...) |
| XIR_GETFIELD       | ✅     | ✅        | —       |      |
| XIR_SETFIELD       | ✅     | ✅        | —       |      |
| XIR_GUARD_TAG      | ✅     | ✅        | —       |      |
| XIR_DEOPT          | ✅     | ✅        | —       |      |
| XIR_SUSPEND        | ✅     | ✅        | —       |      |
| ...                |       |          |         |      |
```

这张表的价值:
- 一目了然哪些 opcode 在哪些后端缺失
- 新增后端时作为检查清单
- Code review 时快速判断改动是否需要跨后端同步

### 10.8 Sanitizer 矩阵

现有 CI 只在 `ubuntu-latest` (x86-64) 跑 ASan/UBSan/TSan。多后端后建议扩展:

| Sanitizer | Linux x64 | Linux ARM64 | macOS ARM64 | Windows x64 |
|-----------|-----------|-------------|-------------|-------------|
| **ASan** | ✅ CI | ✅ CI | ✅ CI | ❌ (MSVC ASan 有限) |
| **UBSan** | ✅ CI | ✅ CI | ✅ CI | ❌ |
| **TSan** | ✅ CI | 🟡 可选 | 🟡 可选 | ❌ |

**重点**: TSan 对 JIT 后台编译线程特别重要 — 现有的间歇性 SIGBUS 可能就是并发 bug。

### 10.9 本地跨平台测试 (开发者工作流)

开发者日常在 macOS ARM64 开发，如何快速验证其他平台？

**方案 1: Docker + QEMU (推荐)**

```bash
# Linux x86-64 测试
docker run --rm -v $(pwd):/xray -w /xray ubuntu:24.04 bash -c \
  "apt-get update && apt-get install -y cmake gcc && \
   cmake -B build && cmake --build build -j8 && \
   cd build && ctest --output-on-failure"

# Linux ARM64 测试 (QEMU user-mode, 慢但可用)
docker run --rm --platform linux/arm64 -v $(pwd):/xray -w /xray ubuntu:24.04 bash -c \
  "apt-get update && apt-get install -y cmake gcc && \
   cmake -B build && cmake --build build -j8 && \
   cd build && ctest --output-on-failure"
```

**方案 2: 交叉编译 + SSH 远程测试**

```bash
# 假设有一台 Linux x86-64 远程机器
rsync -az --exclude build . remote:~/xray/
ssh remote "cd ~/xray && cmake -B build && cmake --build build -j8 && cd build && ctest --output-on-failure"
```

**方案 3: GitHub Actions 手动触发**

```yaml
# .github/workflows/ci.yml 增加 workflow_dispatch
on:
  workflow_dispatch:
    inputs:
      platform:
        description: "Platform to test"
        required: true
        default: "all"
        type: choice
        options: [all, linux-x64, linux-arm64, windows-x64, macos-arm64]
```

### 10.10 测试覆盖率目标

| 组件 | 目标 | 当前 | 说明 |
|------|------|------|------|
| XIR 优化 pass | 80%+ | ~70% (估) | 所有平台共享，一份测试覆盖全部 |
| ARM64 codegen | 70%+ | ~60% (估) | 87 个 JIT 测试 + 回归测试覆盖 |
| x64 codegen | 50%+ | ~30% (估) | x64 仍在 "Phase F.4.2"，功能覆盖有限 |
| code_alloc (macOS) | 90%+ | 有 | 有单元测试 |
| code_alloc (Linux) | 90%+ | 部分 | 需补充 |
| code_alloc (Windows) | 90%+ | 0% | Phase 3 新增 |

### 10.11 回归测试的跨平台注意事项

某些测试在不同平台可能有合法的输出差异:

1. **浮点精度**: `printf("%.15g", ...)` 在不同平台可能最后几位不同
   - 解决: 测试用 `%.6g` 或整数比较
2. **指针打印**: `print(some_object)` 输出地址不同
   - 解决: 测试不比较含指针的输出行
3. **错误消息格式**: 可能含平台特定路径分隔符
   - 解决: expected output 用正则匹配或 `__normalize_output()`
4. **未定义行为**: 某些 UB 在不同架构表现不同
   - 解决: 消除 UB（这本身就是 bug）

建议在 `run_regression_tests.sh` 中增加输出归一化步骤:

```bash
normalize_output() {
    # Strip pointer addresses: <0x7fff...> → <PTR>
    # Normalize path separators: \ → /
    # Trim trailing whitespace
    sed -E \
        -e 's/0x[0-9a-fA-F]+/<PTR>/g' \
        -e 's/\\/\//g' \
        -e 's/[[:space:]]+$//'
}
```
