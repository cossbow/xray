# Xray AOT 实施计划（合并终版）

> Date: 2026-04-23
> 基于 `src/aot/` 18 文件 / 约 6500 行代码审计 + VM/AOT 输出实测对拍
> 合并自 `docs/archive/aot-implementation-plan-gpt.md`（语义视角）+ 早期 opus 草稿（架构视角，已覆盖）
> 关联 `docs/design/aot-design.md`（目标设计）
> 已归档的历史分析：`docs/archive/aot-implementation-plan-gpt.md` · `docs/archive/aot-refactor-plan.md`

---

## 0. 执行摘要

### 0.1 定位

`src/aot/` 是 **"XIR → C 转写器 + header-only 运行时"** 原型：

```
.xr → frontend → bytecode → XIR → xcgen(C 转写) → cc → 独立二进制
```

**VM-AOT 输出一致**：`hello` · `arithmetic` · `control_flow` · `class_methods`
**能编但语义错**：`shared_vars`（VM `3` vs AOT `0`）· `try_catch`（VM `null` vs AOT `-1`）
**架构阻塞**：硬编码进程内指针 · class 裸偏移 · 零 ARC 调用 · 双 `ObjHeader` · JIT bridge
**完全没跑**：跨进程复现 · 多模块链接 · 长进程 · `go/Channel`

### 0.2 核心原则（铁律）

**① 不考虑向后兼容** · 无外部用户，每一步选最佳设计；不做兼容层、不保留旧接口。

**② Unsupported IR 必须 hard fail** · 禁止 `default: /* TODO */` + 构建成功；禁止 `XIR_GUARD_*/RETAIN` 静默 no-op。未支持即 `abort()` 或编译错误。

**③ VM-AOT 对拍是验收金标准** · 支持范围内每个 `.xr` 必须 `diff <(xray run X) <(./aot_X)` 为空；`-fsyntax-only` 通过**不算**通过。

**④ 禁止 fallback-to-bytecode** · 不允许"AOT + runtime hybrid + 未覆盖路径回退字节码"。宁可明确失败，也不走半 VM 半 AOT。

**⑤ 架构错误不是"技术债"** · 硬编码指针、双 header、JIT bridge 必须当下修复，不允许打包 P2 延后。

### 0.3 阶段划分（8 阶段 · 11-14 周）

```
阶段 0   (P0) · 正确性基线 + 对拍 + hard fail         [1 周]    ← 前置门
阶段 I   (P0) · 解除硬编码运行时指针                  [1-2 周]
阶段 II  (P0) · Class → typed C struct                [2-3 周]
阶段 III (P0) · ARC codegen (retain/release)          [2-3 周]
阶段 IV  (P1) · 统一 XrtObjHeader                      [1 周]
阶段 V   (P1) · 消除 JIT bridge                        [1 周]
阶段 VI  (P1) · 多模块输出 + 具名 shared              [2 周]
阶段 VII (P2) · 代码清理 + runtime 封装               [1-2 周]
```

**阶段 0 是所有后续的入场门**。P0（0/I/II/III）完成（6-9 周）即可交付可分发、正确的 AOT。

---

## 1. 现状基线

### 1.1 源码规模（6446 行）

| 模块 | 文件 | 行数 |
|------|------|------|
| 代码生成 | `xcgen.c` / `xcgen_call.c` / `xcgen_expr.c` / `xcgen_stmt.c` / `xcgen_struct.c` | 1386 / 1443 / 1097 / 221 / 491 |
| runtime | `xrt_arc.h` / `xrt_method.h` / `xrt_coll.h` / `xrt_class.h` 等 | 173 / 253 / 259 / 204 / … |

### 1.2 VM vs AOT 实测对拍

```
case            VM              AOT             状态
hello           42              42              ✅
arithmetic      7/10/55         7/10/55         ✅
class_methods   25/11/6/8/4/6   25/11/6/8/4/6   ✅
class_basic     7/3             7/3             ✅
shared_vars     3               0               ❌ shared closure 调用被吞
try_catch       10/null/0       10/-1/0         ❌ catch 返回 fallback 而非异常值
```

现有 `tests/aot/run_aot_tests.sh` 只做 `cc -fsyntax-only`，从不运行产物，**语法正确 ≠ 语义正确**，当前测试基线不可信，阶段 0 必须先修复。

### 1.3 生成产物样本（`class_basic.xr`）

```c
static void xr___module_init(XrtContext xrt_ctx) {
    int64_t v0=0, …, v19=0;     // 死变量未剪
    XrValue v1={0}, …, v20={0};

    v1  = xrt_closure_new((void*)xr_constructor, 0);   // (A) 非逃逸仍堆分配
    v11 = xrt_arc_alloc(32);
    xr_constructor(xrt_ctx, v11, xrt_box_int(3), xrt_box_int(4));
    v12 = (int64_t)0xaa3000960;                        // (B) 进程内指针硬编码
    v14 = xr_sum(xrt_ctx, v11);
    v17 = *(int64_t*)((char*)v11.ptr + 0);             // (C) 裸偏移 class 字段
}
```

**(A)** 冗余闭包 → 阶段 VII · **(B)** 阻塞级 → 阶段 I · **(C)** class 未 struct 化 → 阶段 II

---

## 2. 问题清单

### 2.1 P0 · 阻塞级（8 个）

#### 架构阻塞

| # | 位置 | 问题 | 阶段 |
|---|------|------|------|
| P0-1 | `xcgen_expr.c:122,878` | `CONST_PTR` 硬编码进程内指针 | I |
| P0-2 | `xcgen_expr.c:490`, `xcgen_call.c:266` | class 字段裸偏移 `(char*)ptr + offset - 24` | II |
| P0-3 | `xcgen_expr.c:1071`（`XIR_RETAIN/RELEASE` 全 no-op） | 零 retain/release，长进程内存只增 | III |
| P0-4 | 生成 `main()` 无 `xrt_arc_init()` | bump 默认关闭 | III |
| P0-5 | `xrt_arc.h XrtArcHdr` vs `xrt_class.h XrtObjHeader` | 两套 header 布局不兼容 | IV |
| P0-6 | `xcgen_bridge.h` + libxray_core 链接 | AOT 产物仍依赖 JIT runtime | V |

#### 语义阻塞（实测不一致）

| # | 位置 | 问题 | 阶段 |
|---|------|------|------|
| P0-7 | `xcgen_call.c:1163-1196` GETSHARED 对 native dst "elided" 注释 | 顶层 shared closure 调用被吞 | 0 |
| P0-8 | `xcgen_call.c:770-777` + `xcgen_stmt.c` 块终止 | `xrt_throw_exc` 后仍 emit 死 `return`，catch 读到 phi 垃圾 | 0 |

### 2.2 P1 · 一致性（4 个）

| # | 位置 | 问题 | 阶段 |
|---|------|------|------|
| P1-1 | `xcgen_call.c:1163-1223` | `xrt_shared[N]` 扁平，多模块不能 `extern` | VI |
| P1-2 | `xcmd_build.c` `single_file=true` | 多模块仅编入口 | VI |
| P1-3 | `xcgen_call.c:770` | `_Noreturn` 后死 return（P0-8 根因） | 0 |
| P1-4 | `xrt_arc.h:67` `xrt_bump_enabled=0` | 默认关闭 bump | III |

### 2.3 P2 · 可维护性（10 个）

| # | 位置 | 问题 |
|---|------|------|
| P2-1 | `xcgen.c:391` `CALL_ARGS_BASE_OFFSET=688` | 硬编码 coroutine 偏移 |
| P2-2 | `xcgen_call.c:emit_call_c` ~790 行 | 超 150 行规则 |
| P2-3 | `xcgen.c` / `xcgen_call.c` > 1000 行 | 混合 5 个职责 |
| P2-4 | `xcgen.h` | `tmp_count`/`needs_gc`/`shadow_stack_count`/`c_var`/`single_file` 死字段 |
| P2-5 | `xcgen_struct.c:262,279` | `strdup()` 违反硬禁令 |
| P2-6 | `xrt_arc.h` / `xrt_coll.h` | 直接 `malloc`，缺封装层 |
| P2-7 | `tests/aot/run_aot_tests.sh` | 只 `-fsyntax-only`（阶段 0 修复） |
| P2-8 | `tests/unit/jit/test_aot_e2e.c` | 命名误导（实为 JIT ARM64） |
| P2-9 | `xcgen.c:820-852` | 声明全部 vreg 非 used 子集 |
| P2-10 | `xcgen.c:1297` | 无条件 `#include "xrt.h"`，未自包含 |

---

## 3. 关键设计决策

实施前锁定的 5 个决策（不再反复）：

### 3.1 `--native` 定位

**纯 AOT standalone**。生成的 C 可脱离 libxray_core 编译运行。**不做** AOT + bytecode fallback hybrid。

### 3.2 跨模块调用策略

| 场景 | 路径 |
|------|------|
| typed + 静态已知函数 | `extern + direct C call`，命名 `mod_<name>__<fn>` |
| 动态 export 访问 | `xrt_module_lookup()` 查表 |
| closure value | 函数指针 + closure runtime |

不混用。

### 3.3 异常 lowering 统一模型

- `TRY_BEGIN`：`setjmp` + push frame
- `TRY_END`：pop frame
- `XIR_THROW` / `AOT_CALL_THROW` → 统一进入 `xrt_throw_exc`（`_Noreturn` + longjmp）
- `XIR_CATCH`：读 `xrt_exception` 绑定目标 vreg
- **正常路径禁止 `abort()`**，仅允许作 unreachable guard
- `_Noreturn` 调用后**不 emit 任何 terminator**

### 3.4 对象生命周期模型（一句话）

> "AOT 对象默认走 ARC；bump arena 作为 scope-local 临时内存，`main()` 入口 `xrt_arc_init()` / 出口 `xrt_bump_destroy()`；`XIR_RETAIN/RELEASE` 由 codegen 实际发射，**不允许**在 AOT 前被消除。"

| 归属 | 对象 |
|------|------|
| ARC | string / class instance / collection / promoted struct 的 tagged 字段 |
| bump arena | 临时 tagged value / 短命 closure env |
| stack | primitive typed locals |

### 3.5 内存分配策略

- **runtime 内部**（`xrt_arc.h` / `xrt_coll.h`）：统一封装 `xrt_alloc` / `xrt_free`，底层直接 libc（AOT 运行时豁免 `xr_malloc` 规则，头部注释说明）
- **生成代码侧**：只调 `xrt_obj_alloc()` / `xrt_array_new()` 等封装，永不直接 `malloc`
- **OOM**：统一 `abort()` + stderr（不返回 NULL）

---

## 4. 分阶段实施

### 阶段 0 · 正确性基线 + 对拍（前置门）

#### 目标

1. VM-AOT 对拍脚本可用
2. P0-7 shared closure 语义修复
3. P0-8 异常返回值修复
4. Unsupported IR 全部 hard fail

#### 0.1 · 对拍脚本

重写 `tests/aot/run_aot_tests.sh`：

```bash
TMP="${TMPDIR%/}"; TMP="${TMP:-/tmp}"
run_case() {
    local xr="$1" name=$(basename "$1" .xr)
    local c="$TMP/aot_${name}_$$.c"  bin="$TMP/aot_${name}_$$"
    "$XRAY" build --native -c "$xr" -o "$c" || { echo "FAIL transpile"; return 1; }
    cc -O2 -Wall -Wno-initializer-overrides -I src/aot -I include \
       "$c" -o "$bin" 2>&1 || { echo "FAIL compile"; return 1; }
    local vm=$("$XRAY" run "$xr" 2>&1)  aot=$("$bin" 2>&1)
    [ "$vm" = "$aot" ] || { echo "FAIL mismatch"; echo "VM:"; echo "$vm"; echo "AOT:"; echo "$aot"; return 1; }
    echo "PASS"
}
```

失败时保留 `.c`，便于定位。

#### 0.2 · 修复 shared closure（P0-7）

根因：`xcgen_call.c:1183-1194` GETSHARED 对 native-typed dst emit `/* elided */` 注释 → 后续调用读到垃圾。

策略：

1. `xcgen.c` pass0 扫所有顶层 `SETSHARED`，建 `shared_idx → proto` 映射
2. `GETSHARED` 命中映射 → 记录到 `cf->vreg_direct_fn[dst]`
3. `XIR_CALL_DIRECT` 若 closure 是直接函数引用 → 改发 `xr_<fn>(xrt_ctx, args…)`，跳过 `xrt_shared[]` 间接层

#### 0.3 · 修复异常返回值（P0-8, P1-3）

根因：`xrt_throw_exc` 是 `_Noreturn`，但 `xcgen_stmt.c` 块终止器不知道，继续 emit fallback `return`，catch 读到未初始化 phi。

修复：

```c
// xcgen_stmt.c 块 terminator 前
static bool is_noreturn_call(XirFunc *f, XirIns *i) {
    if (i->op != XIR_CALL_C) return false;
    int64_t kind;
    if (!xcg_resolve_const_i64(f, i->args[0], &kind)) return false;
    return kind == AOT_CALL_THROW;   // 阶段 V 后；现阶段 fn_ptr 比较
}

XirIns *last = &blk->ins[blk->nins - 1];
if (is_noreturn_call(func, last)) {
    xcgen_buf_puts(b, "    /* unreachable after throw */\n");
    return;
}
```

#### 0.4 · Unsupported IR hard fail

`XcgenCompilation` 新增 `bool has_error`。`xcgen_expr.c` 所有 `default:` 改：

```c
default: {
    fprintf(stderr, "AOT: unsupported XIR op %u (%s) in %s:%s\n",
            ins->op, xir_op_name(ins->op), mod->name, func->name);
    mod->comp->has_error = true;
    return;
}
```

`xcmd_build.c` 检测 `has_error` → 不写输出文件，返回非零。

**No-op 白名单**（允许保留的空操作）：
- `XIR_NOP` / `XIR_PHI` / `XIR_SAFEPOINT`
- `XIR_BARRIER_FWD/BACK`（阶段 III 由 ARC 接管）
- `XIR_TRY_BEGIN/END`（`xcgen_stmt` setjmp 逻辑接管）

其他 `XIR_GUARD_*` / `XIR_DEOPT` / `XIR_RETAIN` / `XIR_RELEASE` 当前 no-op → 改 hard fail（阶段 III 实际 emit 后才解除）。

#### 验收

```bash
tests/aot/run_aot_tests.sh                 # 含 VM-AOT diff，shared_vars/try_catch 全 PASS
./build/xray build --native tests/aot/negative/unsupported.xr && fail   # 预期失败
! grep -r '/\* TODO: op' src/aot/*.c        # 无静默 TODO
```

**关联 GPT WP**：WP-01 / WP-02 / WP-03 / WP-04
**工作量**：1 周

---

### 阶段 I · 解除硬编码运行时指针（P0-1）

#### 目标

生成的 `.c` 源码**跨进程可复现**。消除所有 `(int64_t)0xXXXX` 形式地址常量。

#### 根因

| 字节码 op | 指针类型 |
|-----------|----------|
| `OP_CLASS_CREATE_FROM_DESCRIPTOR` | `XrClassDescriptor*` |
| `OP_NEWJSON` | `XrShape*` |
| `OP_CLOSURE` | `XrProto*` |
| `OP_LOADK` (string) | `XrString*`（已特判） |

`xcgen_expr.c:878` 裸 emit `(void*)0x%" PRIx64 "`。

#### 修复

重写 `xcg_emit_const_ptr`：

```c
void xcg_emit_const_ptr(XcgenBuf *b, XirFunc *f, XirIns *ins,
                         XcgenModule *mod, XcgenFunc *cf) {
    XirConst *c = &f->consts[XIR_REF_INDEX(ins->args[0])];
    void *p = (void *)(uintptr_t)c->val.raw;
    uint32_t dst = XIR_REF_INDEX(ins->dst);

    if (p == NULL) { emit_null(b, dst); return; }
    if (is_string_const(c, p)) { emit_string_literal(b, f, ins); return; }

    const char *fn = xcg_lookup_proto_name(mod, p);
    if (fn) {
        xcgen_buf_printf(b, "    v%u = xrt_mkptr((void*)&%s, XRT_TAG_PTR);\n", dst, fn);
        return;
    }

    int sidx = mod->struct_reg ? xcgen_find_struct(mod->struct_reg, p) : -1;
    if (sidx >= 0) {
        xcgen_buf_printf(b, "    v%u = xrt_mkptr((void*)(uintptr_t)%d, XRT_TAG_PTR);\n", dst, sidx);
        return;
    }

    fprintf(stderr, "AOT: CONST_PTR 0x%" PRIx64 " not resolvable\n", (uint64_t)c->val.raw);
    mod->comp->has_error = true;
}
```

`xcg_emit_ref`（`xcgen_expr.c:122`）：PTR 常量不再 `0x%x`。

XIR builder 侧：`OP_CLASS_CREATE_FROM_DESCRIPTOR` 在 AOT 模式下 (`xb->mode == XIR_MODE_AOT`) 不发 descriptor const_ptr，依靠 DCE 消除。

#### 验收

```bash
build/xray build --native -c tests/aot/basic/class_basic.xr -o /tmp/a.c
build/xray build --native -c tests/aot/basic/class_basic.xr -o /tmp/b.c
diff /tmp/a.c /tmp/b.c                                    # 空
! grep -E '\(int64_t\)0x[0-9a-f]{6,}' /tmp/a.c          # 零
tests/aot/run_aot_tests.sh                               # 全 PASS
```

**工作量**：1-2 周

---

### 阶段 II · Class → Typed C struct（P0-2）

#### 目标

```c
typedef struct { XrtObjHeader hdr; int64_t x; int64_t y; } XrtObj_Point;

static XrValue xr_constructor(XrtContext ctx, XrtObj_Point *self, int64_t x, int64_t y) {
    self->x = x; self->y = y;
    return xrt_mkptr(self, XRT_TAG_PTR);
}
```

#### 实施步骤

**II.1 · 提取 class schema**

新增 `xcgen_collect_classes`（`xcgen_struct.c`）：扫 constructor proto 的 `OP_TFIELD_SET`，记 `(field_index, type_from_param)`。`XcgenStruct` 加 `uint8_t kind` 区分 shape/class。

**II.2 · 触发 struct promotion**

`xcgen_call.c:emit_call_known` 的 `is_ctor_call`：

```c
int cls_sidx = find_class_struct_id(mod->struct_reg, cp);
if (cls_sidx >= 0) {
    XcgenStruct *st = &mod->struct_reg->structs[cls_sidx];
    xcgen_buf_printf(b,
        "    { %s *_inst = (%s *)xrt_obj_alloc(%d, sizeof(%s));\n"
        "      %s(xrt_ctx, _inst", st->c_name, st->c_name, cls_sidx, st->c_name, callee_name);
    /* args */
    xcgen_buf_printf(b, ");\n      v%u = xrt_mkptr(_inst, XRT_TAG_PTR); }\n", dst_idx);
    cf->vreg_struct_id[dst_idx] = (int16_t)cls_sidx;
}
```

**II.3 · 字段访问走 struct**

`xcgen_expr.c:420-500` 的 `EMIT_STRUCT_BASE` 扩展到 class。未 promote 的 class 字段访问 → hard fail。

**II.4 · Constructor 签名改造**

self param 为 class 时，签名改 `XrtObj_Point *self`。`xcg_c_type` 对 `vreg_struct_id[0] >= 0` 返回 `struct_name *`。

**II.5 · vtable 生成**

```c
static XrtMethodFn XrtVT_Point[] = { xr_constructor, xr_sum };
static XrtTypeInfo XrtType_Point = { "Point", sizeof(XrtObj_Point), XrtVT_Point, 2, xrt_deinit_Point };
uint16_t XRT_TYPE_Point;
void xr___module_init(XrtContext ctx) {
    XRT_TYPE_Point = xrt_type_register(&XrtType_Point);
}
```

CHA-devirtualized 直接调用用 C 函数名；多态走 `xrt_method_N` fallback。

#### 验收

```bash
grep -q 'typedef struct.*XrtObj_Point' /tmp/cb.c          # 必须
! grep -q '\*(int64_t\*).*ptr + ' /tmp/cb.c               # 零裸偏移
tests/aot/run_aot_tests.sh                                 # 含 class/* 全 PASS
```

新增：`tests/aot/class/{inheritance,field_types,nested_class,method_chain,dynamic_prop_fail}.xr`。

**工作量**：2-3 周

---

### 阶段 III · ARC Codegen（P0-3, P0-4, P1-4）

#### 目标

- 长进程 RSS 稳定
- AddressSanitizer clean
- `retain_count == release_count`
- `XIR_RETAIN/RELEASE` 实际 emit

#### 实施步骤

**III.1 · Main 加 ARC 生命周期**

```c
int main(int argc, char **argv) {
    xrt_arc_init();
    xrt_modules_init(xrt_modules, xrt_modules_count, NULL);  // 阶段 VI
    xr___module_init(NULL);
    xrt_bump_destroy();
    return 0;
}
```

`xrt_arc.h:67` `xrt_bump_enabled=1` 默认开。

**III.2 · 函数级 retain/release**

param 为 `XrValue/PTR/class struct*`：

```c
static XrValue xr_scale(XrtContext ctx, XrtObj_Vec2 *self, double factor) {
    xrt_obj_retain(self);
    /* body */
    xrt_obj_release(self);
    return result;
}
```

**III.3 · Last-use 优化（Move）**

新增 `xcg_analyze_liveness`（`xcgen_analysis.c`）：

```
for vreg:
    if uses == 1 and is_arg_to_call:    mark_move(vreg)
    elif is_ptr_type(vreg):             emit_retain_at_def; emit_release_at_last_use
```

**III.4 · 赋值**

```c
// a = b   →
xrt_obj_retain(b);    // 先 retain new
xrt_obj_release(a);   // 再 release old（防 a == b）
a = b;
```

**III.5 · 循环**

```c
XrValue x = xrt_mkptr(NULL, XRT_TAG_NULL);
for (...) {
    XrValue _new = next_value();
    xrt_obj_retain(_new); xrt_obj_release(x);
    x = _new;
}
xrt_obj_release(x);
```

**III.6 · 调试工具**

```c
#ifdef XRT_DEBUG_ARC
extern uint64_t xrt_arc_retain_count, xrt_arc_release_count;
void xrt_arc_dump_stats(void);  // atexit 打印
#endif
```

#### 验收

```bash
cc -fsanitize=address -O1 -g -I src/aot /tmp/cb.c -o /tmp/bin && /tmp/bin   # no leak
cc -DXRT_DEBUG_ARC -O2 -I src/aot /tmp/cb.c -o /tmp/bin && /tmp/bin         # retain == release
/usr/bin/time -l /tmp/loop_1m                                                # RSS 稳定
scripts/run_aot_asan.sh                                                       # 全 PASS
```

**工作量**：2-3 周

---

### 阶段 IV · 统一 XrtObjHeader（P0-5）

#### 目标

**一个 header、一套 ARC 接口**：

```c
typedef struct { uint32_t rc; uint16_t type_id; uint16_t flags; } XrtObjHeader;
typedef struct { XrtObjHeader hdr; /* user fields */ } XrtObj_XXX;
```

#### 修改

| 文件 | 动作 |
|------|------|
| `xrt_arc.h` | 删 `XrtArcHdr`，`xrt_arc_alloc` → `xrt_obj_alloc_raw` |
| `XRT_ARC_HDR(p)` 宏 | 废弃，直接 `p->hdr` |
| `xrt_class.h` | `XrtObjHeader` 唯一定义 |
| `xrt_coll.h` | `xrt_array_t` / `xrt_map_t` / `xrt_closure_t` 首字段 `XrtObjHeader hdr` |
| `xcgen_call.c:580` | 删 `XrtArcHdr *_h = XRT_ARC_HDR(...)` |
| `xcgen_struct.c` | typedef 首字段强制 `XrtObjHeader hdr` |
| `xrt_compat.h` | **删除** |

#### 验收

```bash
! grep -r XrtArcHdr src/aot                 # 零
! ls src/aot/xrt_compat.h                   # 不存在
tests/aot/run_aot_tests.sh + ASAN           # 全 PASS
```

**工作量**：1 周

---

### 阶段 V · 消除 JIT Bridge（P0-6）

#### 目标

AOT 二进制**不链接 libxray_core**。

#### 方案：Sentinel 常量

```c
// src/aot/aot_sentinels.h
typedef enum {
    AOT_CALL_THROW      = 0x1001,
    AOT_CALL_GETPROP    = 0x1002,
    AOT_CALL_INDEX_GET  = 0x1003,
    AOT_CALL_INDEX_SET  = 0x1004,
    AOT_CALL_GET_SHARED = 0x1005,
    AOT_CALL_SET_SHARED = 0x1006,
    AOT_CALL_PRINT      = 0x1007,
    AOT_CALL_RT_ADD     = 0x1010,
    /* ... */
} AotCallKind;
```

XIR builder AOT 模式填枚举，codegen `switch(kind)` 分发。

#### 修改

| 文件 | 动作 |
|------|------|
| `src/aot/aot_sentinels.h` | 新建 |
| `src/xir_builder_*.c` | `xb->mode == XIR_MODE_AOT` 改填 sentinel |
| `src/aot/xcgen_call.c:emit_call_c` | 地址比较改 `kind` 分发 |
| `src/aot/xcgen_bridge.h` | **删除** |
| `src/app/cli/xcmd_build.c` | AOT 路径不暴露 JIT runtime |

#### 验收

```bash
otool -L /tmp/gen_bin | grep libxray    # 空
! ls src/aot/xcgen_bridge.h              # 不存在
! grep -r 'xr_jit_' src/aot             # 零
```

**工作量**：1 周

---

### 阶段 VI · 多模块 + 具名 shared（P1-1, P1-2）

#### 目标

多 `.xr` 模块分别输出 `.c`/`.o`，链接成单一二进制。

#### 命名约定

```c
// mod_math.c
XrValue mod_math__PI = xrt_box_float(3.14159);
int64_t mod_math__compute_area(XrtContext, double);

// mod_main.c
extern XrValue mod_math__PI;
extern int64_t mod_math__compute_area(XrtContext, double);
```

#### shared 改造

`GETSHARED/SETSHARED`：`xrt_shared[0]` → `mod_<module>__<name>`。数据来源 `XrProto.shared_names`。

#### 编译驱动

`xcmd_build.c:cmd_build_native`：

```c
for each module in topo-sorted bundle:
    read source + compile to XrProto
    create XcgenModule
    collect exports / shared
    output mod_<name>.c + mod_<name>.h

cc -O2 -I src/aot mod_A.c mod_B.c ... -o binary
```

每个 `.c` 首行 `#include "mod_<name>.h"`；`.h` 自身 `#include "xrt.h"`（修复 P2-10）。

#### Module init 真正执行

```c
int main(int argc, char **argv) {
    xrt_arc_init();
    xrt_modules_init(xrt_modules, xrt_modules_count, NULL);  // 拓扑序初始化所有模块
    xrt_bump_destroy();
    return 0;
}
```

`xrt_modules[]` 从"生成了没人用"升级为真实执行入口。

#### Import lowering

| 被导入项 | 生成 |
|----------|------|
| typed fn（已知签名） | `extern T mod_X__fn(…);` 直调 |
| const / let | `extern XrValue mod_X__name;` 直访 |
| dynamic symbol | `xrt_module_lookup("mod_X", "name")` |

#### 验收

```bash
build/xray build --native tests/aot/modules/mod_{a,b,main}.xr -o /tmp/bin
/tmp/bin                                  # 输出匹配
! grep 'xrt_shared\[' /tmp/*.c            # 零
for f in /tmp/mod_*.c; do cc -fsyntax-only -I src/aot $f; done   # 每个独立编译
```

新增测试：`tests/aot/modules/{main_import_function,main_import_const,transitive_import}.xr`。

**关联 GPT WP**：WP-05 / WP-06
**工作量**：2 周

---

### 阶段 VII · 代码清理 + runtime 封装（P2-*）

#### 7.1 · 代码拆分

`xcgen.c`（1386）→

```
xcgen.c             public API (<200)
xcgen_compile.c     compilation driver
xcgen_analysis.c    逃逸 / liveness / DCE
xcgen_emit.c        代码装配
xcgen_forward.c     前向声明
```

`xcgen_call.c`（1443）→

```
xcgen_call.c              dispatcher
xcgen_call_known.c        CALL_KNOWN / SELF_DIRECT
xcgen_call_c.c            CALL_C 分发
xcgen_call_c_builtin.c    string/array/map/math 内联
xcgen_call_c_shim.c       sentinel 分发
```

`emit_call_c`（790 行）按 sentinel 拆：`emit_call_c_throw` / `_getprop` / `_index_get` / `_method_invoke`，每个 ≤ 150 行。

#### 7.2 · 死字段清理

| 字段 | 位置 | 动作 |
|------|------|------|
| `tmp_count`/`needs_gc`/`shadow_stack_count` | `xcgen.h:80,82,103` | 删 |
| `XcgenExport.c_var` | `xcgen.h:117` | 删 |
| `single_file` | `xcgen.h:167` | 阶段 VI 后删 |

#### 7.3 · `strdup` → `xr_strdup`

`xcgen_struct.c:262,279`。补 `xcgen_struct_registry_destroy` 的 `xr_free` 路径。

#### 7.4 · Runtime 分配封装

新建 `src/aot/xrt_alloc.h`：

```c
static inline void *xrt_alloc(size_t n) {
    void *p = malloc(n);
    if (!p) { fprintf(stderr, "xrt: OOM %zu\n", n); abort(); }
    return p;
}
static inline void *xrt_calloc(size_t n, size_t s) { /* ... */ }
static inline void *xrt_realloc(void *p, size_t n) { /* ... */ }
static inline void xrt_free(void *p) { free(p); }
```

`xrt_arc.h` / `xrt_coll.h` 改走封装。头部注释：

```c
/* NOTE: AOT runtime 私有 malloc/free，被生成代码 include。
 * 豁免 xr_malloc 规则。生成代码只调 xrt_obj_alloc 封装。
 */
```

#### 7.5 · vreg 声明精简

`xcgen.c:820-852`：

```c
for (uint32_t vi = 0; vi < func->nvreg; vi++) {
    if (cf->used_vregs && !cf->used_vregs[vi]) continue;
    /* declare v%u */
}
```

#### 7.6 · 硬编码 offset

`xcgen.c:391` `CALL_ARGS_BASE_OFFSET=688` → 用 `xir_offsets.h` 的 `XIR_CORO_CALL_ARGS_OFFSET`（若无则补）。

#### 7.7 · 重命名 `test_aot_e2e.c`

`tests/unit/jit/test_aot_e2e.c` 实为 JIT ARM64 测试 → 改名 `tests/unit/jit/test_jit_arm64_e2e.c`；`tests/unit/aot/` 下建立真 AOT 单元测试。

#### 7.8 · ASAN 回归脚本

```bash
# scripts/run_aot_asan.sh
for xr in tests/aot/**/*.xr; do
    build/xray build --native -c "$xr" -o /tmp/t.c || continue
    cc -fsanitize=address -O1 -g -I src/aot /tmp/t.c -o /tmp/t || fail
    /tmp/t || fail "$xr ASAN"
done
```

集成到 `scripts/run_regression_tests.sh`。

#### 验收

- `wc -l src/aot/*.c` 每个 ≤ 500
- `! grep -E 'strdup|XrtArcHdr|single_file|tmp_count|needs_gc' src/aot`
- `cc -Wall -Wextra` 无 `-Wunused-variable`
- `scripts/run_aot_asan.sh` 全 PASS

**工作量**：1-2 周

---

## 5. 验收矩阵总览

| 阶段 | 检查命令 | 期望 |
|------|----------|------|
| 0 | `tests/aot/run_aot_tests.sh` 含 VM-AOT diff | 全 PASS；shared_vars/try_catch 修复 |
| 0 | unsupported IR 负测 | 构建失败，退出码非零 |
| I | `grep -E '\(int64_t\)0x[0-9a-f]{6,}' gen.c` | 零 |
| I | 跨进程生成 diff | 空 |
| II | `grep 'typedef struct.*XrtObj_' gen.c` | ≥ 1 |
| II | `grep '\*(int64_t\*).*ptr + [0-9]' gen.c` | 0（Json 除外） |
| III | `grep -c 'xrt_obj_retain\|xrt_obj_release' gen.c` | > 0 |
| III | `cc -fsanitize=address ... && ./bin` | no leak |
| IV | `grep -c XrtArcHdr src/aot` | 0 |
| V | `otool -L gen_bin \| grep libxray` | 空 |
| V | `ls src/aot/xcgen_bridge.h` | 不存在 |
| VI | 多模块 `.xr` 链接运行 | 正确输出 |
| VI | `grep 'xrt_shared\[' gen.c` | 0 |
| VII | `wc -l src/aot/*.c` | ≤ 500 |
| VII | `scripts/run_aot_asan.sh` | 全 PASS |

---

## 6. 代码改造地图

### 6.1 新增

```
src/aot/aot_sentinels.h            阶段 V
src/aot/xrt_alloc.h                阶段 VII
src/aot/xcgen_module.c             阶段 VI
src/aot/xcgen_compile.c            阶段 VII
src/aot/xcgen_analysis.c           阶段 III / VII
src/aot/xcgen_emit.c               阶段 VII
src/aot/xcgen_forward.c            阶段 VII
src/aot/xcgen_call_known.c         阶段 VII
src/aot/xcgen_call_c.c             阶段 VII
src/aot/xcgen_call_c_builtin.c     阶段 VII
src/aot/xcgen_call_c_shim.c        阶段 VII

scripts/run_aot_asan.sh            阶段 III

tests/aot/negative/*.xr            阶段 0
tests/aot/class/*.xr               阶段 II
tests/aot/arc/*.xr                 阶段 III
tests/aot/exceptions/*.xr          阶段 0
tests/aot/modules/*.xr             阶段 VI
tests/unit/aot/*.c                 阶段 VII
```

### 6.2 删除

```
src/aot/xcgen_bridge.h             阶段 V
src/aot/xrt_compat.h               阶段 IV
```

### 6.3 重大修改

```
xcgen.c       1386 → 5 文件，每个 ≤ 400
xcgen_call.c  1443 → 4 文件，每个 ≤ 400
xcgen_expr.c  1097 → 保留，补 const_ptr / struct 访问
xcgen_struct.c 491 → 扩展 class schema
xrt_arc.h      173 → 统一 XrtObjHeader
xrt_class.h    204 → ARC 唯一入口
tests/aot/run_aot_tests.sh         → 三阶段验证
tests/unit/jit/test_aot_e2e.c      → 重命名为 test_jit_arm64_e2e.c
```

---

## 7. 风险与降级

### 7.1 整体降级策略

阶段内遇阻塞时：

1. **Runtime fallback** — 生成 `xrt_call_dynamic(...)` 动态分发
2. **标记 SKIP_AOT** — 测试脚本豁免
3. **绝不走 VM 回退** — 宁可 `abort()`，不走"半 VM 半 AOT"
4. **绝不容忍 silent 错误** — 不支持的必须 hard fail，不能"编译成功但输出错"

### 7.2 阶段性交付

每阶段产物可独立 cherry-pick：

| 完成阶段 | 可交付 |
|----------|--------|
| 阶段 0 | 正确性基线，shared/exception 修复，测试基线可信 |
| 阶段 I | `hello`/`arithmetic`/`control_flow` 是真正独立二进制（跨进程可复现） |
| 阶段 II | `class_basic`/`class_methods`/继承可独立运行 |
| 阶段 III | ASAN 通过，长进程内存稳定 |
| 阶段 IV | 单一 header，ARC 语义闭环 |
| 阶段 V | 产物不链接 libxray_core |
| 阶段 VI | 多模块项目可 AOT |
| 阶段 VII | 代码结构符合项目规范 |

### 7.3 具体风险

| 风险 | 阶段 | 缓解 |
|------|------|------|
| XIR builder 改动影响 JIT | I / V | `xb->mode` 分叉，JIT 路径单测先行 |
| 继承字段对齐 | II | 线性布局"父先于子"；递归收集 descriptor |
| retain/release 配对错误 → UAF | III | 按 XIR 规则逐条单测；ASAN 强制 |
| 过度 retain 性能退化 | III | Last-use 分析 + benchmark |
| 跨模块循环 import | VI | 拓扑排序时检测 cycle，hard fail |

---

## 8. 附录 A · 关键代码 before/after

### A.1 硬编码指针（阶段 I）

```c
/* Before */  v12 = (int64_t)0xaa3000960;
/* After  */  /* CONST_PTR elided (dead) */  v12 = xrt_mkptr(NULL, XRT_TAG_NULL);

/* Before */  v1 = xrt_mkptr((void*)0xaa2f8a1b0, XRT_TAG_PTR);
/* After  */  v1 = xrt_mkptr((void*)(uintptr_t)0, XRT_TAG_PTR);  /* shape id 0 */
```

### A.2 Class 字段（阶段 II）

```c
/* Before */
{ int64_t _sv = xrt_unbox_int(v1); memcpy((char*)v0.ptr + 0, &_sv, 8); }
v17 = *(int64_t*)((char*)v11.ptr + 0);

/* After */
((XrtObj_Point*)v0.ptr)->x = xrt_unbox_int(v1);
v17 = ((XrtObj_Point*)v11.ptr)->x;
```

### A.3 ARC（阶段 III）

```c
/* Before */
static XrValue xr_scale(XrtContext ctx, XrValue self, double factor) {
    XrValue result = /* ... */;
    return result;
}

/* After */
static XrValue xr_scale(XrtContext ctx, XrtObj_Vec2 *self, double factor) {
    xrt_obj_retain(self);
    XrValue result = /* ... */;
    xrt_obj_release(self);
    return result;
}
```

### A.4 Shared Closure（阶段 0）

```c
/* Before */
/* GETSHARED[0] → v2 (native, elided) */
v4 = xr_risky(xrt_ctx, v0);  /* ← 没调用到 shared function */

/* After */
v4 = xr_safe_double(xrt_ctx, v0);  /* shared closure 直接映射到 C 函数 */
```

### A.5 异常终止（阶段 0）

```c
/* Before */
xrt_throw_exc(v3);
v5 = INT64_C(-2401263018287759359);   /* 死代码 */
return INT64_C(-2401263018287759359); /* catch 读到 phi 垃圾 */

/* After */
xrt_throw_exc(v3);
/* unreachable after throw */
```

### A.6 JIT Sentinel（阶段 V）

```c
/* Before (XIR builder) */
xir_emit(CALL_C, const_ptr(xr_jit_throw), ...);
/* Before (codegen) */
if (fn_ptr == (void *)xr_jit_throw) { ... }

/* After (XIR builder, AOT mode) */
xir_emit(CALL_C, const_i64(AOT_CALL_THROW), ...);
/* After (codegen) */
int64_t kind; xcg_resolve_const_i64(func, ins->args[0], &kind);
switch ((AotCallKind)kind) { case AOT_CALL_THROW: ... }
```

### A.7 具名 Shared（阶段 VI）

```c
/* Before */
static XrValue xrt_shared[3];
xrt_shared[0] = xrt_box_float(3.14);
v5 = xrt_shared[0];

/* After */
XrValue mod_math__PI;                 /* 跨模块可见 */
mod_math__PI = xrt_box_float(3.14);
v5 = mod_math__PI;
/* 另一模块 */ extern XrValue mod_math__PI;
```

---

## 9. 附录 B · 里程碑时间线

```
Week 1       · 阶段 0   (对拍 + shared/exception + hard fail)   ← 正确性基线
Week 2-3     · 阶段 I   (硬编码指针)
Week 4-6     · 阶段 II  (Class struct)
Week 7-9     · 阶段 III (ARC codegen)                            ← P0 完成
Week 10      · 阶段 IV  (统一 Header)
Week 11      · 阶段 V   (JIT Bridge)
Week 12-13   · 阶段 VI  (多模块)
Week 14-15   · 阶段 VII (清理)
```

合计 **14-15 周**；P0（0/I/II/III）完成需 **7-9 周**即可交付可分发、正确的 AOT。

---

## 10. 附录 C · 不在本计划范围

以下留给 Phase C+：

- **并发**：`go` / `Channel` / `shared let` move 语义
- **增量编译**：`.o` 缓存 / 分布式构建
- **优化**：向量化 / SLP / PGO / 跨模块内联
- **工具链**：DWARF 内嵌 / panic stack trace
- **LSP**：AOT 模式跨模块跳转 / 调用图

这些都依赖本计划（阶段 0 - VII）先稳定。

---

## 11. 附录 D · 历史文档映射

本文合并吸收了两份历史文档（均已归档到 `docs/archive/`）：

### 11.1 历史文档位置

| 文件 | 状态 | 位置 |
|------|------|------|
| `aot-implementation-plan-gpt.md` | 归档 | `docs/archive/` |
| `aot-refactor-plan.md` | 归档 | `docs/archive/` |
| `aot-implementation-plan-opus.md`（本文） | **当前** | `docs/design/` |

> ⚠️ 归档文档仅作溯源参考，**实施以本文为唯一真相源**。

### 11.2 章节对应关系

| 本文章节 | 架构视角来源 | GPT 版（语义视角）来源 |
|----------|--------------|------------------------|
| §0.2 核心原则 | 用户指令 | WP-04 + 隐含原则 |
| §1.2 VM-AOT 对拍 | — | WP-01 |
| §2.1 P0-7/P0-8 语义 bug | — | WP-02 / WP-03 |
| §2.1 P0-1~6 架构 bug | 原 opus §2.1 | — |
| §3 设计决策 | 散见 | GPT §7 统一整理 |
| §4 阶段 0 | — | WP-01/02/03/04 |
| §4 阶段 I-VII | 原 opus §3 | — |
| §5 验收矩阵 | 原 opus §4 | GPT §9 |
| §6 代码地图 | 原 opus §5 | GPT §6 |
| §7 风险 | 原 opus §6 | GPT §11 |
| §8 代码对照 | 原 opus §7 | — |

### 11.3 已作废的原则（不再采纳）

- GPT §14 "不大幅重写架构" → 与用户 "不考虑向后兼容" 原则冲突
- GPT §7.1 "AOT + bytecode fallback" → 典型技术债累积模式
- GPT "AOT Alpha / Beta" 命名 → 过度流程化
- GPT §2.2 把 `malloc/free`、双 header、JIT bridge 归为 "P2 工程债" → 架构错误不是债务
- `aot-refactor-plan.md` Step 1-6 的节奏 → 被本文 8 阶段路线覆盖
