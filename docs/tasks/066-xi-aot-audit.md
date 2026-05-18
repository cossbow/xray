# 066a — 034/035/036 Implementation Audit Report

**Companion**: `066-xi-aot-migration-plan.md` — actionable migration plan based on this audit.

## Scope

Systematic audit of three design documents against current codebase:
- **034** — Typed SSA XIR + unified value/GC contract
- **035** — Codegen-to-XIR migration impact analysis
- **036** — Xi IR lowering/emission coverage audit

**Date**: 2026-04-30
**Baseline**: 96/96 ctest, 127/127 Xi compare tests passing

---

## Conclusion

**Core architecture goals achieved.** Xi Typed SSA IR is the sole
compilation path for VM bytecode. Old xcodegen deleted. But **4
legacy items** and **2 design concerns** remain.

---

## Completed Items

### 034 — Typed SSA XIR + Unified Value/GC Contract

| Step | Status | Evidence |
|------|--------|----------|
| S1: XrRep unified | ✅ | `xi_rep.h` shared `xi_value_def_rep()` used by `xi_opt.c` and `xi_cgen.c`. No `b->aot_mode` branches in `src/aot/` |
| S3: Xi Typed SSA framework | ✅ | `src/ir/` — 21 files, ~9,900 lines. 66 XiOp kinds, Braun SSA, arena alloc, full CFG |
| S4: Optimization passes | ✅ | 7 passes: const_fold, strength_reduce, copy_prop, phi_simplify, dce, select_rep, box_elim |
| S5: Single pipeline | ✅ | `xcompiler.c` 146 lines, calls `xi_pipeline_compile_program()` directly, no legacy fallback |
| xcodegen deletion | ✅ | `src/frontend/codegen/` reduced to 4 files (context + thin compiler shell), ~20KB |
| Xi→C AOT | ✅ | `xi_cgen.c` 966 lines, `xaot_build_xi()` uses full Xi pipeline |
| Pipeline config | ✅ | `XiPipelineConfig` with `run_verify/run_optimize/run_select_rep/dump_ir_*` |

### 035 — xcodegen Migration Impact Analysis

| Subsystem | Document expectation | Actual |
|-----------|---------------------|--------|
| Constant folding | SSA pass replaces peephole | ✅ `xi_opt_const_fold` |
| Instruction fusion | Lowering stage handles | ✅ `xi_emit.c` fuses ADDI/MULK/LTI etc at instruction selection |
| Peephole optimization | Mostly eliminated | ✅ SSA naturally removes jump chains / redundant MOVEs |
| XrExprDesc | SSA regalloc replaces | ✅ `xi_emit.c` contains full register allocator |
| inst_types | Xi value carries type | ✅ `XiValue.type` is authoritative |
| Three-stage compilation | Lowering layer equivalent | ✅ `xi_lower.c` prescan_shared_vars etc |
| for-in 10 variants | Migrated individually | ✅ All covered (index-based + iterator-based) |
| Coroutine statements | Intrinsic call + CFG | ✅ XI_GO/XI_AWAIT/XI_CHAN_*/XI_SCOPE_* |

### 036 — Xi IR Coverage Audit

| Metric | Final status |
|--------|-------------|
| AST expression coverage | ✅ All 38 node types with check_exec |
| AST statement coverage | ✅ All 16 node types |
| Test count | 127 (122 check_exec=true, 5 similarity) |
| ctest | 96/96 ✅ |
| Bugs fixed | 8 (Bug 1-8 all closed) |

---

## Outstanding Legacy Items

### Legacy-1: Old AOT Path Fully Retained — Compromise Design

`xaot_driver.c` `xaot_build()` (lines 1-1072) still uses the
**xir_builder → xcgen** path:
- Depends on `src/jit/xir_builder*.c` (~8,600 lines)
- Depends on `src/aot/xcgen*.c` (~5,800 lines)
- Uses bytecode pattern scanning (build_shared_proto_map, collect_exports, preregister_classes)

The Xi path `xaot_build_xi()` (lines 1073-1197) only supports
**single-file compilation**, not multi-module.

**Impact**: Document 034 S5 explicitly says "delete xir_builder*".
Two parallel AOT pipelines is the "dual-path" anti-pattern 034 rejects.

**Resolution**: 066-xi-aot-migration.md S0-S1-S8.

### Legacy-2: XrtValue Not Deleted — S2 Incomplete

`src/aot/xrt_value.h` is a standalone redefinition of XrValue, not
`#include "xvalue.h"`:
- Layout already unified (tag@0, payload@8), macro names unified as `XR_FROM_*/XR_TAG_*`
- AOT defines extra tags >= 8 (STR=14, ARRAY=15, MAP=16, CLOSURE=18, STR_ARC=19)
- VM's `xvalue.h` does not have these tags
- `xr_truthy()` independently implemented in both files

Document 034 §3.7: "AOT abandons XrtValue independent layout, unifies to XrValue" / "eventually delete xrt_value.h". Current state: **layout aligned but file not merged**.

**Impact**: Two truth sources. AOT-specific tag extensions (≥8) are
legitimate (standalone AOT has no GC headers), but should be achieved
via conditional compilation in `xvalue.h`, not a separate file.

**Resolution**: 066-xi-aot-migration.md S8 cleanup step.

### Legacy-3: Old JIT XIR Type System Fully Retained — 034 S1 Partial

`src/jit/xir.h` still defines:
- `XirTypeKind` (12 kinds)
- `VTAG_*` enum
- `vtag_to_type_kind()` / `type_kind_to_vtag()` / `xir_type_from_vtag()` (~120 lines)

Document 034 §3.6: "delete XirTypeKind / VTAG mapping". Still used by
old JIT builder (because Legacy-1 not yet cleaned).

**Resolution**: Automatically deletable after Legacy-1 is resolved.

### Legacy-4: Old xcgen Uses Fragile fn_ptr Matching

`xcgen_bridge.h` deleted (✅), but old `xcgen_call.c` still identifies
CALL_C targets via `fn_ptr` comparison. This is the "most fragile AOT
design" pattern flagged in documents 034/035. The Xi path (`xi_cgen.c`)
already uses table-driven `XRT_SYM_*` dispatch.

**Resolution**: Eliminated together with Legacy-1.

---

## Design Review Findings

### Finding A: `xi_lower.c` at 2,810 lines — Near Scale Limit

C coding standard limits `.c` to ≤ 3,000 lines. `xi_lower.c` is at
2,810, with only 190 lines of growth headroom. Satellite files
(`xi_lower_class.inc.c` 152 lines, `xi_lower_stmt.c` 581 lines)
are reasonable splits.

**Recommendation**: Extract for-in / destructure / coroutine lowering
into `xi_lower_forin.inc.c` / `xi_lower_coro.inc.c`.

### Finding B: `xi_emit.c` at 2,019 lines — Acceptable

Contains complete register allocator + instruction selection + bytecode
emission. Scale is reasonable.

### Finding C: Xi→C codegen (`xi_cgen.c`) at 966 lines — Limited Coverage

Compared to old xcgen (5,807 lines), the Xi cgen covers a smaller
feature subset. Multi-module, class vtable, full OOP, closure escape
analysis, try/catch/finally, defer, indirect calls all present in old
path but missing or stubbed in new path. This is why Legacy-1 cannot
be deleted yet.

**Resolution**: 066-xi-aot-migration.md S0-S6 systematically ports
needed capabilities.

### Finding D: SCCP, Type Specialization, GCM Not Implemented

Document 034 S4 lists 5 pass targets:

| Pass | Status |
|------|--------|
| Constant folding | ✅ `xi_opt_const_fold` |
| DCE | ✅ `xi_opt_dce` |
| GCM (Global Code Motion) | ❌ Not implemented |
| Type specialization | ❌ Not implemented |
| Inlining | ❌ Not implemented |

Document 036 §P2 already marks these as "not implemented". These are
**performance optimizations** that do not block correctness.

---

## Recommended Actions (Priority Order)

| Priority | Action | Debt eliminated | Effort | Tracking |
|----------|--------|----------------|--------|----------|
| **P0** | Xi AOT multi-module + delete old path | ~14,400 lines | 3-5d | 066-xi-aot-migration.md S0/S1/S8 |
| **P0** | Merge `xrt_value.h` into `xvalue.h` | Duplicate defs | 1d | 066-xi-aot-migration.md S8 |
| **P1** | Delete `XirTypeKind`/`VTAG_*` mappings (after P0) | ~200 lines | 0.5d | 066-xi-aot-migration.md S8 |
| **P1** | Split `xi_lower.c` into .inc.c files | Scale risk | 0.5d | Standalone |
| **P2** | Implement SCCP, GCM, type specialization | AOT perf | 2-3d each | Standalone |

---

## Summary

034/035/036 core goals — single Xi SSA pipeline, full VM bytecode
coverage, 127 execution comparison tests — are all achieved. The
largest remaining debt is **the old AOT path being fully retained**,
directly violating the "single pipeline, no dual paths" principle.

Per Xray's development principle (no backward compat, bold rewrites),
the P0 action is to extend Xi AOT to multi-module and delete the old
path. This is tracked in `066-xi-aot-migration.md`.
