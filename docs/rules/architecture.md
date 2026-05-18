# 架构约束

## 源码目录结构

```
src/
├── base/                  L0  基础工具（与平台无关）
├── os/                    L0+ 跨平台 OS 抽象层
│   ├── os_*.h             头文件 API（thread / time / fd / dir / dylib / mem / net / random）
│   ├── unix/*_unix.c      POSIX 实现（macOS / Linux / BSD）
│   └── win/*_win.c        Windows 实现
├── runtime/
│   ├── value/             L1  值系统 → base
│   ├── gc/                L2  GC → value, base
│   ├── object/            L2  对象系统 → value, gc, base
│   ├── class/             L3  类系统 → object, value
│   └── symbol/            L3  符号表 → value
├── coro/                  L3  协程 → gc, object, value
├── frontend/
│   ├── lexer/             L4  词法分析 → base
│   ├── parser/            L4  语法分析 → lexer, value
│   ├── codegen/           L4  字节码生成 → parser, object
│   └── analyzer/          L4  类型分析 → parser, value
├── module/                L4  模块系统 → object, class
├── vm/                    L5  虚拟机 → gc, object, coro, class, module
├── jit/                   L5  JIT（含 XIR）→ vm
├── aot/                   L6  AOT → jit
├── api/                   L7  公共 API → vm, object, gc
└── app/
    ├── cli/               L8  命令行 → api
    ├── lsp/               L8  LSP → api, analyzer
    └── dap/               L8  DAP → api
```

## 依赖规则

- Layer N **不能** include Layer N+1 的头文件
- 每个 .h 只 include 比自己更底层的头
- 双向通信用回调/函数指针，不用反向 include
- 禁止深层相对路径 `../../` 以上
- 内部头文件禁止包含 `include/xray.h`

## 新增/修改文件检查清单

1. 文件是否超过 3,000 行？→ 拆分
2. 头文件是否只 include 更底层的头文件？→ 检查依赖方向
3. 新函数需要被外部调用？不需要 → 必须 static
4. .c 和 .h 是否一一对应？
5. 是否添加了足够断言？（每 50-80 行 ≥ 1 个）
6. 内存分配是否通过 xr_malloc？
7. 预处理条件用 `XR_OS_*` 而不是 `_WIN32`/`__APPLE__`？（R3）
8. 跨平台 OS 调用是否走 `src/os/os_*.h` shim？（R2，理想态）

## 哈希函数

统一使用 `src/base/xhash.h`：
- `xr_hash_bytes()` / `xr_hash_bytes64()`
- `xr_hash_int()` / `xr_hash_float()` / `xr_hash_bool()`
- **禁止在其他模块重复实现哈希算法**

## 平台抽象（R1 / R2 / R3）

xray 用三条规则隔离 OS 差异。`scripts/check_platform_layering.sh` 自动校验 R1 + R3（hard fail），R2 计数追踪。

### R1 — 公共头不带系统头（hard）

`include/*.h` 不得 include 任何系统 OS 头（`<windows.h>` / `<unistd.h>` / `<pthread.h>` 等）。嵌入方应该只看到 freestanding 头 + xray 内部头。

### R2 — 调用方走 `src/os/` shim（advisory）

`src/`（除 `src/os/` 与 `src/base/xplatform.h`）和 `stdlib/` 的代码理想上不直接 include 系统 OS 头，所有 OS 调用都走 `src/os/os_*.h` 或其衍生 API。

当前状态：`src/coro/xnetpoll_*.c`、`src/jit/xir_code_alloc.c`、`src/vm/xvm.c`、`src/app/*` 等仍直接调用 POSIX/Win32 — 这部分迁移分步进行，lint 用 advisory 模式追踪计数变化。

### R3 — 用 `XR_OS_*`，禁止原始编译器宏（hard）

预处理条件中**只能**使用 `src/base/xplatform.h` 暴露的宏：

| ✅ 允许 | ❌ 禁止 |
|---|---|
| `XR_OS_WINDOWS` | `_WIN32` |
| `XR_OS_MACOS` | `__APPLE__` |
| `XR_OS_LINUX` | `__linux__` |
| `XR_OS_BSD` | `__FreeBSD__` / `__OpenBSD__` / `__NetBSD__` |
| `XR_OS_POSIX` | （手写 `__APPLE__ \|\| __linux__`） |

唯一例外：`src/base/xplatform.h` 自己使用原始宏来做 detection。

`XR_OS_*` 通过 `src/base/xdefs.h` 自动传递到几乎所有 TU；新增 .c/.h 文件不需要显式 `#include "xplatform.h"`，只要正常 include 任何项目内部头即可。

### `src/os/` 内部规则

| 文件 | 内容 |
|------|------|
| `os_*.h` | 跨平台 API。可 `#include "../base/xplatform.h"` 选系统头，也可暴露 static-inline 实现 |
| `unix/*_unix.c` | POSIX 实现。CMake 仅在非 Windows 编译；不需要外层 `#ifndef _WIN32` 包裹 |
| `win/*_win.c` | Windows 实现。CMake 仅在 Windows 编译；不需要外层 `#ifdef _WIN32` 包裹 |

CMake 通过 `if(WIN32) file(GLOB OS_SRC "src/os/win/*.c") else() file(GLOB OS_SRC "src/os/unix/*.c") endif()` 实现平台互斥编译。

## 新增 native type

把 C 端 GC-managed 对象（如 `Uuid`、`SHA256`）暴露成 xray 的内置类型名，统一走 prelude。完整流程见 `docs/language/prelude.md`，这里列硬性约束：

1. `stdlib/<mod>/<mod>.c` 实现一个 `xr_<type>_register_native_type(XrayIsolate *)` 函数，内部用 `static const XrNativeTypeInfo` + `xr_register_native_type(...)`。**不**在模块 loader (`xr_load_module_<mod>`) 里直接调 `xr_register_native_type` —— 由 prelude 集中调用。
2. `stdlib/prelude/prelude_types.def` 加一行：`XR_PRELUDE_TYPE("<Name>", XR_T<Name>, <KIND>)`。`KIND` ∈ `SIMPLE` / `GENERIC_1` / `GENERIC_2` / `BYTES` / `SINGLETON`。
3. `stdlib/prelude/prelude.c::xr_prelude_register_all_native_types` forward-decl 新函数并调用一次。
4. cfunc 签名（如果有）用 `XR_DEFINE_BUILTIN(..., "(): <Name>?", ...)` 直接写新类型名 —— analyzer 的 cfunc-signature parser 通过 prelude 表自动认识。

**禁止**：

- 在 `xkeywords.def` 加新关键字（破坏用户 shadow 能力，且需要同步改 lexer/parser/test）
- 在 `xparse_type.c` 里加 `if (strcmp(name, "<Name>") == 0)` 分支（绕过 prelude 表，破坏单一真相源）
- 在多个 stdlib 模块的 loader 里重复 `xr_register_native_type`（违反集中注册原则）

**单一真相源**：`stdlib/prelude/prelude_types.def` 的一行决定该名字在 lexer / parser / analyzer / runtime 里的所有行为。删除该行 ⇔ 删除该类型。
