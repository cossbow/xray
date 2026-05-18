# JIT Architecture Cleanup Plan

## Current Compilation Pipeline

```
Source → AST → Xi IR (typed SSA)
                 ├─ [VM]  xi_emit → bytecode → interpreter
                 ├─ [JIT] xi_to_xir → XIR → passes → codegen → machine code
                 └─ [AOT] xi_cgen → C source → system CC → binary
```

## Current Directory Stats

| Directory | Lines | Role |
|-----------|-------|------|
| `src/ir/` | 11,384 | Xi IR: shared SSA (lower, verify, opt, emit, cgen) |
| `src/jit/` | 47,022 | XIR infrastructure + JIT runtime + legacy builder |
| `src/aot/` | 2,526 | AOT driver + AOT runtime headers |

## Problems

1. **Legacy builder is dead code (7,980 lines)**
   - `xir_builder.c` (2830), `xir_builder_call.c` (2134),
     `xir_builder_misc.c` (1498), `xir_builder_object.c` (1199),
     `xir_builder.h` (191), `xir_builder_internal.h` (128)
   - xi_to_xir now covers all 77 Xi ops including EH
   - All protos have xi_func attached; builder fallback never executes

2. **Fallback code in 3 files** referencing dead builder:
   - `xir_jit.c:569-572` — `xir_build_from_proto_jit` fallback
   - `xjit_compile_queue.c:107-115` — `xir_build_from_proto_jit_ex` fallback
   - `xir_pass_advanced.c:1494-1495` — `xir_build_from_proto` fallback

3. **`xi_cgen.c` misplaced** in `src/ir/` — it's an AOT backend, not IR infra

4. **Dead JIT infra tied to bytecode builder**:
   - `xir_opcode_support.h` (301 lines) — bytecode opcode whitelist, skipped when xi_func present
   - `xir_blueprint.c` (375) — loop liveness from bytecode; still used by codegen for OSR
   - `xir_fold.c` (477) — peephole fold, only called by builder, but reusable

5. **`aot_mode` dead branches** — ~30 `if (b->aot_mode)` in builder (always false)

## Target Architecture

```
src/ir/                    # Xi IR — shared typed SSA
  xi.h / xi.c             # Core types and operations
  xi_lower.c              # AST → Xi IR
  xi_lower_stmt.c         # Statement lowering
  xi_lower_class.inc.c    # Class lowering
  xi_lower.h / xi_lower_internal.h
  xi_verify.c / xi_verify.h
  xi_opt.c / xi_opt.h     # Xi-level optimizations
  xi_emit.c / xi_emit.h   # Xi IR → bytecode
  xi_pipeline.c / xi_pipeline.h
  xi_dump.c               # Debug printing
  xi_analysis.c / xi_analysis.h
  xi_rep.h                # Representation mapping

src/jit/                   # JIT: Xi IR → XIR → machine code
  # --- XIR core ---
  xir.h / xir.c           # XIR types, instructions, functions
  xir_ops.h               # XIR opcode definitions
  xir_bset.h              # Bit-set utility

  # --- Xi IR → XIR lowering (sole path) ---
  xi_to_xir.c / xi_to_xir.h
  xir_fold.c / xir_fold.h   # Peephole fold (used by xi_to_xir)

  # --- XIR optimization passes ---
  xir_pass.c / xir_pass.h / xir_pass_limits.h
  xir_pass_advanced.c       # Inlining, etc.
  xir_pass_cfg.c            # CFG simplification
  xir_pass_type.c           # Type propagation
  xir_pass_sccp.c / xir_pass_sccp.h
  xir_pass_internal.h
  xir_tfa.c / xir_tfa.h     # Type flow analysis
  xir_peephole.c / xir_peephole.h

  # --- XIR analysis ---
  xir_domtree.c / xir_domtree.h
  xir_looptree.c / xir_looptree.h
  xir_defuse.c / xir_defuse.h
  xir_alias.c / xir_alias.h
  xir_liveness2.h

  # --- Register allocation ---
  xir_regalloc.c
  xir_coalesce.c / xir_coalesce.h

  # --- Machine code generation ---
  xir_codegen.c / xir_codegen.h / xir_codegen_internal.h
  xir_codegen_call.c
  xir_codegen_mem.c
  xir_codegen_x64.c / xir_codegen_x64_call.c / xir_codegen_x64_mem.c
  xir_codegen_x64_internal.h
  xir_arm64.c / xir_arm64.h / xir_arm64_disasm.h
  xir_x64.c / xir_x64.h
  xir_code_alloc.c / xir_code_alloc.h
  xir_target.h / xir_target_arm64.c / xir_target_x64.c
  xir_offsets.h / xir_offsets_verify.c
  xir_sentinels.h
  xir_helper_table.c / xir_helper_table.h

  # --- JIT orchestration ---
  xir_jit.c / xir_jit.h / xir_jit_internal.h
  xir_jit_runtime.c / xir_jit_runtime.h
  xir_jit_debug.c / xir_jit_debug.h
  xjit_compile_queue.c / xjit_compile_queue.h
  xir_eligibility.c / xir_eligibility.h
  xir_blueprint.c / xir_blueprint.h  # OSR loop liveness
  xir_intrinsic.c / xir_intrinsic.h
  xir_printer.c / xir_printer.h

src/aot/                   # AOT: Xi IR → C source → binary
  xaot_driver.c / xaot_driver.h
  xi_cgen.c / xi_cgen.h     # MOVED from src/ir/
  xrt_*.h                   # AOT runtime headers
  xrt_*_check.c             # AOT runtime checks
```

## Execution Plan

### Step 1: Integrate xir_fold into xi_to_xir (trivial)
- xi_to_xir's `xir_emit` calls → `xir_fold_emit` for arithmetic/comparison/unary
- xir_fold.c stays as-is, just gets a new caller
- **Result**: early peephole optimization on xi_to_xir output

### Step 2: Add CALL_KNOWN in xi_to_xir
- When XI_CALL's callee has a known proto (via XiValue->aux or slot_map),
  emit XIR_CALL_KNOWN instead of generic CALL_DIRECT
- Propagate callee return_type_info for precise result typing
- **Result**: direct call optimization without IC

### Step 3: Remove legacy builder + fallback paths
- Delete: xir_builder.c, xir_builder_call.c, xir_builder_misc.c,
  xir_builder_object.c, xir_builder.h, xir_builder_internal.h
- Remove fallback in xir_jit.c, xjit_compile_queue.c, xir_pass_advanced.c
- Remove dead `xir_opcode_support.h` (opcode whitelist, bypassed when xi_func present)
- Clean xir_eligibility.c (remove bytecode scan path)
- Clean xir_pass_internal.h (remove builder include)
- **Result**: -8,300 lines, single compilation path

### Step 4: Move xi_cgen to src/aot/
- `src/ir/xi_cgen.c` → `src/aot/xi_cgen.c`
- `src/ir/xi_cgen.h` → `src/aot/xi_cgen.h`
- Fix includes in xaot_driver.c
- **Result**: clean separation — src/ir/ is pure IR, src/aot/ has all AOT code

### Step 5: (Future) JIT profile-guided pass
- New `xir_pass_feedback.c`: inject GUARD_KLASS + CALL_KNOWN from IC snapshots
- New `xir_pass_feedback.c`: inject GUARD_SHAPE + LOAD_FIELD from field IC
- Deopt infrastructure in xi_to_xir or pass
- CHA devirtualization using isolate class hierarchy
- **Result**: runtime-profiled speculative optimization (currently in builder)
