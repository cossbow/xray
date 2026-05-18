# Native-Width Consistency: Explicit Narrowing in Xi IR

## Problem Statement

VM/JIT/AOT 对原生窄类型（uint8, int16, float32 等）存在行为不一致：

| 操作 | VM | JIT | AOT |
|------|-----|-----|-----|
| `Array<uint8>`: `arr[0] = 256` | `(uint8_t)256` → **0** ✅ | 存 **256** ❌ | 存 **256** ❌ |
| `Array<int8>`: 读 `-1` | sign-extend → **-1** ✅ | 仅 I64+ANY ⚠️ | 无类型特化 ❌ |
| `Array<float32>`: 存 `3.14159...` | `(float)` roundtrip ✅ | 无 roundtrip ❌ | 无 roundtrip ❌ |
| `Struct { x: uint8 }`: `s.x = 300` | `(uint8_t)300` → **44** ✅ | 需确认 ⚠️ | **当 map 处理，存 300** ❌ |

### Root Causes

1. **AOT runtime 独立重写** — `src/aot/xrt_*.h` (~2000 行) 是 VM runtime 的独立实现，
   无类型特化存储
2. **JIT typed array 不完整** — `xr_jit_tarray_get/set` 只处理 I64+ANY
3. **Struct 在 Xi IR 层缺失** — `OP_STRUCT_GET/SET` 仅存在于 bytecode 层，
   `xi_cgen.c` 把 struct 字段当 map entry 处理
4. **截断语义隐式依赖运行时** — 截断/扩展逻辑分散在各后端，
   遗漏不可能在编译期发现

### Design Goal

**截断/扩展语义在 Xi IR 层显式表达，编译期可验证，三后端各自用最优方式实现。**

---

## Design Principles

1. **No backward compatibility** — 直接采用最佳设计
2. **Explicit narrowing in IR** — 截断/扩展是 IR 指令，不是运行时 switch
3. **Compile-time verifiable** — xi_verify.c 静态检查："写入窄类型前必须有 NARROW"
4. **Backend-trivial** — 每个后端只需实现一个 cast，不可能写错
5. **Shared storage ops** — Array/Struct 的存储操作共享 static inline 实现

---

## Architecture Overview

```
                        Source Code
                            │
                        Analyzer (type inference)
                            │
                        Xi Lowering ← 自动插入 XI_NARROW / XI_WIDEN
                            │
                        Xi IR (typed SSA, explicit narrowing)
                            │
               ┌────────────┼────────────┐
               │            │            │
          xi_emit.c    xi_to_xm.c   xi_cgen.c
          (bytecode)    (JIT XIR)    (C source)
               │            │            │
               │         ┌──┘            │
               ▼         ▼               ▼
           VM interp  JIT codegen    C compiler
               │         │              │
               └────┬────┘              │
                    │                   │
          Shared storage ops     C type system
        (xr_typed_get/set)     (natural truncation)
```

**关键区别**：截断发生在 Xi IR 层（编译期），不是在存储操作内部（运行时）。

---

## Part A: Xi IR Explicit Narrowing (Core)

### A.1 New Xi IR Instructions

Add to `src/ir/xi.h` enum:

```c
/* Explicit narrowing: truncate int64/double to sub-width, re-extend back.
 * Result is still I64/F64 rep, but value range is restricted.
 * Inserted automatically by xi_lower at typed-storage write points. */
XI_NARROW_I8,       // int64 → (int8_t)  → int64  (sign-extend)
XI_NARROW_U8,       // int64 → (uint8_t) → int64  (zero-extend)
XI_NARROW_I16,      // int64 → (int16_t) → int64
XI_NARROW_U16,      // int64 → (uint16_t)→ int64
XI_NARROW_I32,      // int64 → (int32_t) → int64
XI_NARROW_U32,      // int64 → (uint32_t)→ int64
XI_NARROW_F32,      // double → (float)  → double (precision roundtrip)

/* Explicit widening: sign/zero extend sub-width to int64.
 * Inserted automatically by xi_lower at typed-storage read points.
 * Makes sign-extension vs zero-extension unambiguous in IR. */
XI_WIDEN_I8,        // int8  value in int64 → sign-extend to full int64
XI_WIDEN_U8,        // uint8 value in int64 → zero-extend (mask 0xFF)
XI_WIDEN_I16,       // int16  → sign-extend
XI_WIDEN_U16,       // uint16 → zero-extend
XI_WIDEN_I32,       // int32  → sign-extend
XI_WIDEN_U32,       // uint32 → zero-extend
XI_WIDEN_F32,       // float32 in double → (double)(float) roundtrip
```

**Rep rule**: All NARROW/WIDEN ops keep rep=I64 (or F64 for float variants).
The value width changes, the container width does not. Same as JVM `i2b`.

### A.2 Xi Lowering: Auto-Insert NARROW/WIDEN

**File**: `src/ir/xi_lower.c` (or `xi_lower_expr.c`)

```c
/* When lowering XI_INDEX_SET to a typed array: */
static XiValue *lower_typed_store(XiLowerCtx *ctx, XiValue *arr, XiValue *idx, XiValue *val) {
    XrArrayElemType etype = get_array_elem_type(arr->type);
    if (etype != XR_ELEM_ANY) {
        val = xi_emit_narrow(ctx, etype, val);  // insert XI_NARROW_U8 etc.
    }
    return xi_func_emit3(ctx->func, XI_INDEX_SET, arr->type, arr, idx, val);
}

/* When lowering XI_INDEX_GET from a typed array: */
static XiValue *lower_typed_load(XiLowerCtx *ctx, XiValue *arr, XiValue *idx) {
    XiValue *raw = xi_func_emit2(ctx->func, XI_INDEX_GET, ctx->int_type, arr, idx);
    XrArrayElemType etype = get_array_elem_type(arr->type);
    if (etype != XR_ELEM_ANY) {
        raw = xi_emit_widen(ctx, etype, raw);  // insert XI_WIDEN_I8 etc.
    }
    return raw;
}

/* Helper: emit the correct NARROW for an element type */
static XiValue *xi_emit_narrow(XiLowerCtx *ctx, XrArrayElemType etype, XiValue *val) {
    XiOp op;
    switch (etype) {
        case XR_ELEM_I8:  op = XI_NARROW_I8;  break;
        case XR_ELEM_U8:  op = XI_NARROW_U8;  break;
        case XR_ELEM_I16: op = XI_NARROW_I16; break;
        case XR_ELEM_U16: op = XI_NARROW_U16; break;
        case XR_ELEM_I32: op = XI_NARROW_I32; break;
        case XR_ELEM_U32: op = XI_NARROW_U32; break;
        case XR_ELEM_F32: op = XI_NARROW_F32; break;
        default: return val;  // I64, U64, F64, BOOL: no narrowing needed
    }
    return xi_func_emit1(ctx->func, op, val->type, val);
}
```

**Same logic for Struct field writes** — `XI_STORE_FIELD` with a typed struct
field gets a NARROW inserted before the store.

### A.3 Verifier: Static Consistency Check

**File**: `src/ir/xi_verify.c`

```c
/* Rule: typed-container stores must be preceded by appropriate NARROW.
 * This catches backend bugs at compile time. */
if (v->op == XI_INDEX_SET && is_narrowable_array(v->args[0])) {
    XrArrayElemType etype = get_array_elem_type(v->args[0]->type);
    XiOp expected = narrow_op_for_elem(etype);
    if (expected != 0 && v->args[2]->op != expected) {
        verr(ctx, "func '%s': v%u INDEX_SET to typed array in b%u "
             "missing NARROW (arg v%u op=%s, expected %s)",
             f->name, v->id, blk->id,
             v->args[2]->id, xi_op_name(v->args[2]->op),
             xi_op_name(expected));
    }
}
```

**This is the key advantage**: if any future code path forgets to insert NARROW,
the verifier catches it at compile time — not at runtime via wrong results.

### A.4 Three-Backend Implementation

Each backend implements NARROW/WIDEN trivially:

#### VM Bytecode Emitter (`xi_emit.c`)

```c
/* Option A: emit OP_NARROW_U8 bytecode (dedicated opcode) */
case XI_NARROW_U8:
    emit_inst(ctx, CREATE_AB(OP_NARROW_U8, dst, src));
    break;

/* Option B: fold into TARRAY_SET — no extra opcode needed.
 * Since the value is already narrowed in Xi IR, OP_TARRAY_SET can
 * use simpler storage (no runtime switch on elem_type for truncation). */
```

Recommend **Option A** — dedicated narrow opcodes. They're cheap (1 instruction)
and make the VM's behavior explicitly match the IR. The VM interprets:
```c
vmcase(OP_NARROW_U8) {
    R(GETARG_A(i)).i = (int64_t)(uint8_t)R(GETARG_B(i)).i;
    vmbreak;
}
```

#### JIT Backend (`xi_to_xm.c`)

```c
case XI_NARROW_U8:
    xm_emit(blk, XM_AND_IMM, dst, src, 0xFF);  // 1 machine instruction
    break;
case XI_NARROW_I8:
    xm_emit(blk, XM_SEXTB, dst, src);           // movsx r, r8
    break;
case XI_NARROW_F32:
    xm_emit(blk, XM_CVTSD2SS, tmp, src);        // double → float
    xm_emit(blk, XM_CVTSS2SD, dst, tmp);        // float → double (roundtrip)
    break;
```

#### AOT C Codegen (`xi_cgen.c`)

```c
case XI_NARROW_U8:
    fprintf(out, "(int64_t)(uint8_t)(");
    emit_vref(out, v->args[0]);
    fprintf(out, ")");
    break;
case XI_NARROW_F32:
    fprintf(out, "(double)(float)(");
    emit_vref(out, v->args[0]);
    fprintf(out, ")");
    break;
```

**AOT output is plain C casts — the C compiler naturally handles truncation.**

---

## Part B: Struct in Xi IR

### B.1 Problem

Struct 字段在 bytecode 层有原生布局（`OP_STRUCT_GET/SET`），但 Xi IR 没有对应指令。
AOT codegen 看到 `XI_STORE_FIELD` 时不知道 target 是 struct，当 map 处理。

### B.2 New Xi IR Instructions

```c
/* Struct operations: native-layout field access.
 * aux_int encodes field_index; type carries struct layout info. */
XI_STRUCT_NEW,      // allocate struct (stack or heap depending on escape analysis)
XI_STRUCT_GET,      // read field: args[0]=struct, aux_int=field_idx
XI_STRUCT_SET,      // write field: args[0]=struct, args[1]=value, aux_int=field_idx
```

### B.3 Xi Lowering Changes

**File**: `src/ir/xi_lower.c`

When the analyzer resolves a field access on a struct type, lower to
`XI_STRUCT_GET/SET` instead of `XI_LOAD_FIELD/XI_STORE_FIELD`:

```c
/* Lower struct.field = value */
if (target_type->kind == XR_KIND_STRUCT) {
    XrStructField *field = lookup_struct_field(target_type, field_name);
    /* Insert NARROW if field is sub-width */
    XiValue *narrowed = xi_emit_narrow(ctx, field->elem_type, val);
    xi_func_emit2(ctx->func, XI_STRUCT_SET, void_type,
                  struct_val, narrowed);
    v->aux_int = field->index;
    return;
}
/* Fallthrough: non-struct → XI_STORE_FIELD (map-based) */
```

### B.4 AOT Codegen: Emit Real C Structs

**File**: `src/aot/xi_cgen.c`

```c
/* XI_STRUCT_NEW: emit C struct declaration */
case XI_STRUCT_NEW: {
    XrStructLayout *layout = (XrStructLayout *)v->aux;
    /* Emit typedef at file scope (once per unique struct type) */
    emit_struct_typedef(ctx, out, layout);
    /* In function body: stack-allocate */
    fprintf(out, "%s v%u", layout->c_name, v->id);
    break;
}

/* XI_STRUCT_GET: native field read */
case XI_STRUCT_GET: {
    int field_idx = (int)v->aux_int;
    emit_vref(out, v->args[0]);
    fprintf(out, ".%s", get_field_name(v->args[0]->type, field_idx));
    break;
}

/* XI_STRUCT_SET: native field write — NARROW already applied by lowerer */
case XI_STRUCT_SET: {
    int field_idx = (int)v->aux_int;
    emit_vref(out, v->args[0]);
    fprintf(out, ".%s = ", get_field_name(v->args[0]->type, field_idx));
    emit_vref(out, v->args[1]);  // already narrowed
    break;
}
```

**Generated C code**:
```c
typedef struct { uint8_t sample; int16_t channel; float gain; } xr_AudioFrame;

xr_AudioFrame v3;
v3.sample = (uint8_t)(v5);   // XI_NARROW_U8 already applied → C cast
v3.gain = (float)(v7);       // XI_NARROW_F32 already applied → C cast
```

**C compiler handles native layout + truncation naturally.**

### B.5 VM Large Struct Fallback: Heap-Backed Inline Arrays

**Problem**: VM struct_area is limited by bytecode addressing (~4KB per frame,
expandable to ~64KB with Bx encoding). Large `[N]T` fields (e.g. `[8192]int8`)
may exceed this limit. AOT has no such limit — C structs can be arbitrarily large.

**Solution**: When a struct's total_size exceeds the VM struct_area threshold,
the compiler transparently downgrades large `[N]T` fields to heap-backed typed
arrays. This preserves value semantics while lifting the size constraint.

#### Compiler logic

```
Threshold: XR_VM_STRUCT_INLINE_LIMIT = 4096 bytes (one struct_area page)

During struct layout computation (analyzer):
  1. Compute total_size normally (all fields inline)
  2. If total_size > XR_VM_STRUCT_INLINE_LIMIT:
     a. Sort [N]T fields by size descending
     b. Convert largest [N]T fields to heap_backed until total_size fits
     c. heap_backed fields occupy 8 bytes in layout (pointer to typed array)
     d. Emit compiler warning (see below)
```

#### Compiler warning

```
warning: struct 'AudioBuffer' total size (32776 bytes) exceeds VM inline limit (4096 bytes).
         Field 'data: [32768]int8' will use heap-backed storage in VM/JIT mode.
         This has no effect on AOT builds, which use native C struct layout.
         To suppress: use Array<int8> instead, or compile with --native.
```

Diagnostic level: `XR_DIAG_SEV_WARNING` — not an error, code still compiles and
runs correctly. The warning helps users understand that VM debugging may show
slightly different performance characteristics than the final AOT build.

#### VM runtime changes

```c
// OP_NEW_STRUCT: allocate heap-backed arrays for marked fields
for (int i = 0; i < layout->field_count; i++) {
    XrStructFieldLayout *f = &layout->fields[i];
    if (f->native_type == XR_NATIVE_ARRAY && f->heap_backed) {
        XrArray *arr = xr_array_with_capacity_typed(coro, f->elem_count,
                           xr_native_to_elem_type(f->elem_native_type));
        arr->length = f->elem_count;
        *(XrArray**)(struct_ptr + 8 + f->offset) = arr;
    }
}

// OP_STRUCT_SET (struct-to-struct copy): deep copy heap-backed fields
if (field->native_type == XR_NATIVE_ARRAY && field->heap_backed) {
    XrArray *src_arr = *(XrArray**)(src_fp);
    XrArray *dst_arr = xr_array_clone_typed(coro, src_arr);
    *(XrArray**)(dst_fp) = dst_arr;
} else {
    memcpy(dst_fp, src_fp, field->size);
}
```

#### AOT: completely unaffected

AOT codegen **ignores** the `heap_backed` flag entirely and generates inline C
struct fields as normal:

```c
// struct AudioBuffer { data: [32768]int8 }
// AOT output — always inline, no heap, no warning:
typedef struct { int8_t data[32768]; } xr_AudioBuffer;
```

#### Semantic guarantee

| Property | Inline (small struct) | Heap-backed (large struct) | Identical? |
|----------|----------------------|---------------------------|------------|
| `s.data[i]` read | direct offset | typed array get | ✅ same value |
| `s.data[i] = v` write | direct store + truncate | typed array set + truncate | ✅ same truncation |
| `let b = a` copy | memcpy all fields | memcpy scalars + deep copy arrays | ✅ independent copies |
| Modify after copy | independent | independent (deep copy) | ✅ value semantics |
| Lifetime | stack frame | GC-managed (or frame cleanup) | ✅ invisible to user |
| `s.data.length` | compile-time N | `arr->length` = N | ✅ same |

**Zero behavioral difference. Only performance impact in VM mode (extra malloc per
struct creation, ~1ns extra indirection per element access).**

---

## Part C: Shared Storage Ops (Runtime Support)

Even with explicit narrowing in IR, the runtime still needs shared typed-storage
helpers for **dynamic paths** (method dispatch, generic container ops).

### C.1 New directory: `src/shared/`

```
src/shared/
  xr_elem_type.h       — XrArrayElemType enum + sizes + tid mapping (~90 lines)
  xr_array_ops.h       — xr_typed_get/set: static inline (~80 lines)
  xr_value_ops.h       — xr_truthy, xr_coerce_int/f64 (~50 lines)
  xr_compare_ops.h     — xr_values_equal, xr_values_less (~60 lines)
```

Extract from `src/runtime/object/xarray.h` (currently sole owner):
- `XrArrayElemType` enum → `xr_elem_type.h`
- `xr_array_get_element / xr_array_set_element` → `xr_array_ops.h`

**Dependency rule**: shared headers depend only on `<stdint.h>` + XrValue struct.
No `xmalloc.h`, no `xgc_header.h`. Callers include their own XrValue header first.

### C.2 AOT `xrt_array_t` Rewrite

```c
/* Before: */
typedef struct { int64_t len, cap; XrValue *data; } xrt_array_t;

/* After: */
typedef struct {
    int64_t  len, cap;
    void    *data;        // uint8_t[] / int64_t[] / XrValue[] — depends on elem_type
    uint8_t  elem_type;   // XR_ELEM_ANY / XR_ELEM_U8 / ...
    uint8_t  elem_size;   // cached: bytes per element
    uint8_t  _pad[6];
} xrt_array_t;
```

- `xrt_array_new_typed(cap, elem_type)` — allocate with correct element size
- `xrt_index_get/set` — delegate to `xr_typed_get/set` from shared header
- `xrt_array_push` — uses `xr_typed_set` for correct truncation
- **Delete**: `xrt_tarray_get/set`, `xrt_array_get_i/set_i/get_f/set_f`, `xrt_array_push_i/push_f`

### C.3 JIT Runtime Helpers

- `xr_jit_rt_array_new` — accept `elem_tid` in `extra_arg`, create typed array
- `xr_jit_tarray_get/set` — delegate to `xr_typed_get/set` from shared header
- `xi_to_xm.c` — `XI_ARRAY_NEW` encode `elem_tid` into extra_arg

### C.4 VM Simplification

`OP_TARRAY_GET/SET` in `xvm_dispatch_struct.inc.c`:
- Replace ~160 lines of hand-written 12-case switches
- With `xr_typed_get/set` calls (~10 lines per opcode)

### C.5 AOT Method Dispatch

`xrt_method.h` methods that touch array elements (fill, reverse, indexOf, has, join):
- Replace direct `a->data[i]` access with `xr_typed_get/set`
- ~30 lines of changes for full method-level consistency

---

## Part D: TypedArray Cleanup

### D.1 Delete dead test
- `tests/unit/object/test_typed_array.c` — references non-existent headers

### D.2 Remove from type table
- `src/ir/xi_lower_expr.c`: delete `{"TypedArray", 14}` entry

### D.3 Update comments/messages
- Replace "TypedArray" with "typed array" in error messages and opcode descriptions

---

## Part E: AOT Runtime Tiered Linking

### Tier structure

```
Tier 0: Zero-cost (static inline, no runtime)     — ~0 KB binary overhead
  src/shared/xr_*_ops.h + src/aot/xrt_value.h
  Arithmetic, comparison, array typed get/set, struct field access, narrowing

Tier 1: Micro-allocator (header-only)              — ~5 KB binary overhead
  src/aot/xrt_arc.h
  Bump allocator, ARC retain/release, string alloc

Tier 2: Collection ops (static library)             — ~20-50 KB binary overhead
  libxr_shared.a
  Array grow/resize, Map rehash, sort, deep copy, toString, method dispatch

Tier 3: Concurrency runtime (optional)              — ~100-200 KB binary overhead
  libxr_coro_aot.a
  Coroutine scheduler, channel, network poller
```

---

## Implementation Order

| Step | Description | Lines Changed | Priority |
|------|-------------|---------------|----------|
| **A.1** | Xi IR: add NARROW/WIDEN opcodes to `xi.h` + `xi_op_name.h` + `xi_backend.h` | +60 | **P0** |
| **A.2** | xi_lower: auto-insert NARROW/WIDEN at typed store/load | +80 | **P0** |
| **A.3** | xi_verify: static check for missing NARROW | +30 | **P0** |
| **A.4a** | xi_emit: OP_NARROW/WIDEN bytecodes + VM interpreter | +80 | **P0** |
| **A.4b** | xi_to_xm: JIT NARROW/WIDEN → AND/SEXT/CVTSD2SS | +40 | **P0** |
| **A.4c** | xi_cgen: AOT NARROW/WIDEN → C casts | +30 | **P0** |
| **B.1-4** | Xi IR struct support: XI_STRUCT_NEW/GET/SET + xi_lower + xi_cgen | +200 | **P0** |
| **B.5** | VM large struct fallback: heap-backed arrays + compiler warning | +80 | **P1** |
| **C.1** | Create `src/shared/` headers (extract from xarray.h) | +200 (new), -200 (move) | **P0** |
| **C.2** | Rewrite `xrt_array_t` + AOT typed storage | -80 / +50 | **P0** |
| **C.3** | JIT array_new + tarray_get/set → shared ops | ~85 | **P0** |
| **C.4** | VM TARRAY opcodes → shared ops | -160 / +40 | **P0** |
| **C.5** | AOT method dispatch → shared typed ops | ~30 | **P1** |
| **D** | TypedArray cleanup | -400 (dead test), ~30 (names) | **P1** |
| **E** | Tiered linking (libxr_shared.a) | +300 | **P2** |

---

## File Impact Summary

### New files
```
src/shared/xr_elem_type.h         — enum + sizes + tid mapping (~90 lines)
src/shared/xr_array_ops.h         — xr_typed_get/set static inline (~80 lines)
src/shared/xr_value_ops.h         — truthiness, coerce (~50 lines)
src/shared/xr_compare_ops.h       — equality, ordering (~60 lines)
```

### Modified files (Xi IR + backends)
```
src/ir/xi.h                       — add XI_NARROW_* / XI_WIDEN_* / XI_STRUCT_* enums
src/ir/xi_op_name.h               — add op name strings
src/ir/xi_backend.h               — add to opcode classification
src/ir/xi_verify.c                — NARROW-before-store check
src/ir/xi_effect.h                — mark NARROW/WIDEN as pure (no side effects)
src/ir/xi_lower.c                 — auto-insert NARROW/WIDEN at typed boundaries
                                     lower struct field access → XI_STRUCT_GET/SET
src/ir/xi_emit.c                  — emit OP_NARROW_* / OP_STRUCT_GET/SET bytecodes
src/jit/xi_to_xm.c               — lower NARROW → AND/SEXT; STRUCT → native load/store
src/aot/xi_cgen.c                 — NARROW → C cast; STRUCT → C struct access
```

### Modified files (runtime)
```
src/runtime/object/xarray.h       — extract elem_type + typed ops to shared/
src/runtime/value/xstruct_layout.h — add heap_backed flag to XrStructFieldLayout
src/vm/xvm_dispatch_struct.inc.c  — TARRAY opcodes → xr_typed_get/set (~-160 lines)
                                     OP_NEW_STRUCT: allocate heap-backed arrays
                                     OP_STRUCT_SET (copy): deep copy heap-backed fields
src/jit/xm_jit_runtime.c          — tarray_get/set → shared ops; struct heap-backed support
src/jit/xm_jit_runtime_coro.c     — array_new accepts elem_tid
src/frontend/analyzer/xanalyzer_visitor_decl.c — large struct warning + heap_backed marking
src/aot/xrt_coll.h                — xrt_array_t + elem_type + void *data
src/aot/xrt_method.h              — array methods → xr_typed_get/set
```

### Deleted files
```
tests/unit/object/test_typed_array.c  — dead test referencing non-existent headers
```

---

## Validation Criteria

1. **Build**: `cmake --build build -j8` — zero warnings
2. **Unit tests**: `cd build && ctest --output-on-failure` — all pass
3. **Regression**: `scripts/run_regression_tests.sh` — all pass
4. **Verifier**: `xi_verify` catches missing NARROW at compile time
5. **Consistency test**: Same `.xr` source produces identical output in VM / JIT / AOT:
   ```javascript
   let u8: Array<uint8> = Array(4)
   u8[0] = 256       // expect: 0 (truncation)
   u8[1] = -1        // expect: 255 (unsigned wrap)
   assert(u8[0] == 0)
   assert(u8[1] == 255)

   let f32: Array<float32> = Array(2)
   f32[0] = 3.141592653589793
   assert(f32[0] != 3.141592653589793)  // precision loss is correct

   struct AudioFrame { sample: uint8, gain: float32 }
   let f = AudioFrame{}
   f.sample = 300    // expect: 44
   f.gain = 3.141592653589793
   assert(f.sample == 44)
   ```
6. **Memory**: `Array<uint8>` 1M elements uses ~1MB (not ~16MB)
7. **AOT binary size**: No regression for Tier 0-1 programs

---

## Consistency Guarantee Analysis

### Where native-width truncation occurs

| Storage boundary | Has native-width? | Consistency mechanism |
|-----------------|-------------------|----------------------|
| **Array element write** | ✅ | XI_NARROW before XI_INDEX_SET |
| **Array element read** | ✅ | XI_WIDEN after XI_INDEX_GET |
| **Struct field write** | ✅ | XI_NARROW before XI_STRUCT_SET |
| **Struct field read** | ✅ | XI_WIDEN after XI_STRUCT_GET |
| Map/Set/Json value | ❌ (XrValue) | N/A — always full-width |
| Class instance field | ❌ (XrValue) | N/A — always full-width |
| Local variable | ❌ (XrValue register) | N/A — always int64/double |
| Function param/return | ❌ (XrValue) | N/A — always int64/double |
| Channel payload | ❌ (XrValue + deep copy) | N/A — always full-width |

### Why this is complete

Truncation semantics only exist at the boundary between the computation layer
(int64/double) and the storage layer (native-width memory). In Xray:
- **Computation**: always int64/double (VM registers, JIT registers, C variables)
- **Storage**: Array data buffer, Struct field layout

There are exactly 2 storage types with native width: Array and Struct.
Both are now covered by explicit NARROW/WIDEN in the IR.

---

## What We Explicitly Do NOT Do

1. **No variable-width bytecodes** — AOT/JIT don't read bytecodes; won't help consistency
2. **No variable-width VM registers** — impractical; all modern VMs use uniform slots
3. **No Map/Set/Json type specialization** — hash table overhead dominates, ROI too low
4. **No Class instance native layout** — Struct already covers this use case
5. **No full VM runtime linking for AOT** — performance and size regression unacceptable
6. **No X-macro code generation** — over-engineering, debugging nightmare
