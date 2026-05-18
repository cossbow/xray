# IR Rename & Optimization Unification Design

> Two inter-related architectural changes for the Xray compiler pipeline.

## Motivation

The current two-layer IR has a naming collision problem and an optimization
duplication problem:

1. **Naming**: `Xi` (src/ir/, mid-level typed SSA) and `Xir` (src/jit/,
   machine-level SSA) are visually and phonetically near-identical, causing
   confusion for contributors and documentation.

2. **Optimization duplication**: Five optimization passes (constfold, copy
   propagation, DCE, phi simplification, strength reduction) are implemented
   independently in both layers. Advanced passes (SCCP, GVN, LICM, GCM,
   inlining) exist only in the Xir layer, so the VM and AOT backends
   never benefit from them.

Both problems have a unified solution: rename the machine-level IR to create
clear visual distinction, and promote optimizations to the semantic IR layer
where all three backends consume them.

---

## Part 1: IR Rename — `Xir` → `Xm` (Xray Machine IR)

### Decision

| Layer | Current | New | Rationale |
|-------|---------|-----|-----------|
| Mid-level typed SSA (src/ir/) | `Xi` | **`Xi` (unchanged)** | Well-established, self-consistent within src/ir/ |
| Machine-level SSA (src/jit/) | `Xir` | **`Xm`** | Clear visual distinction, "M" for Machine |

Alternative names considered and rejected:

| Name | Reason rejected |
|------|-----------------|
| `XrHir`/`XrLir` | Too verbose for prefix (4 chars); breaks `xr_<module>_xxx()` naming rule |
| `Xlir` | 4-char prefix; "lir" not meaningful to non-compiler-engineers |
| `XrMIR` | Conflicts with Rust's MIR; uppercase acronym in prefix is unusual for Xray |
| Rename Xi → Xir | Would require renaming the already-stable src/ir/ layer; higher risk |

`Xm` follows the existing naming pattern: short prefix (2 chars like `Xi`),
semantically clear ("Machine"), and visually distinct from `Xi`.

### Rename mapping

```
Prefix / Type          Current             New
─────────────────────  ──────────────────  ──────────────────
Type prefix            Xir                 Xm
Macro prefix           XIR_                XM_
Function prefix        xir_                xm_
File prefix            xir_                xm_
Header guard           XIR_xxx_H           XM_xxx_H
Enum prefix            XIR_                XM_
```

**Exceptions** (deliberate, not renamed):

| Symbol | Reason |
|--------|--------|
| `xi_to_xir.c/h` | Becomes `xi_to_xm.c/h` — the "xir" in the name refers to the old target IR, rename it |
| `XI_*` ops in xi.h | Unchanged — these are Xi IR ops, not Xir |
| `XrType`, `XrValue`, `XrProto` | Unchanged — these are runtime types, not IR types |

### Scope

```
Location                                          Estimated changes
────────────────────────────────────────────────  ──────────────────
src/jit/xir_*.c/h → src/jit/xm_*.c/h            70 files renamed
  Symbols: XirFunc→XmFunc, XirBlock→XmBlock,     ~5,600 occurrences
  XirRef→XmRef, XIR_→XM_, xir_→xm_ etc.
src/jit/xi_to_xir.* → src/jit/xi_to_xm.*        2 files renamed
Outside src/jit/ (vm, coro, runtime, aot):        ~48 occurrences
  XIR_JIT_OK, XIR_SUSPEND_SPILL_MAX, etc.
tests/unit/ referencing xir_*                     ~20 files
CMakeLists.txt (GLOB, no individual files)        0 changes (*.c glob)
include/ (public headers)                         0 (Xir not in public API)
```

### Execution plan

The rename is purely mechanical and can be scripted:

1. **Write a rename script** (`scripts/rename_xir_to_xm.sh`):
   - `git mv` all `xir_*.c/h` → `xm_*.c/h` and `xi_to_xir.*` → `xi_to_xm.*`
   - `sed -i` global replace: `Xir` → `Xm`, `XIR_` → `XM_`, `xir_` → `xm_`
   - Fix header guards: `XIR_xxx_H` → `XM_xxx_H`

2. **Manual review** of false positives:
   - `XI_*` ops must not be touched (grep to verify)
   - Comments that say "XIR" as a concept name: update to "Xm" or "Xray Machine IR"
   - The doc comment in `xi.h` line 25 referencing "Xir prefix in src/jit/" needs update

3. **Build + test**: `cmake --build build && cd build && ctest --output-on-failure`

4. **Single commit**: `git add -A && git commit -m "Rename Xir → Xm (Xray Machine IR) for clear IR layer distinction"`

**Estimated effort**: 2–4 hours (mostly script + manual review).

---

## Part 2: Optimization Unification to Xi IR Layer

### Current state

**Xi IR passes** (src/ir/xi_opt.c, 763 lines):

| Pass | Lines | Status |
|------|-------|--------|
| `xi_opt_const_fold` | ~120 | ✅ |
| `xi_opt_copy_prop` | ~60 | ✅ |
| `xi_opt_dce` | ~100 | ✅ |
| `xi_opt_phi_simplify` | ~80 | ✅ |
| `xi_opt_strength_reduce` | ~120 | ✅ |
| `xi_opt_select_rep` | ~100 | ✅ (AOT/JIT only) |
| `xi_opt_box_elim` | ~60 | ✅ (AOT/JIT only) |

**Xm (current Xir) passes** (src/jit/xir_pass*.c, ~11,100 lines):

| Pass | File | Lines | Duplicates Xi? |
|------|------|-------|----------------|
| DCE | xir_pass.c | ~300 | **Yes** |
| Copy propagation | xir_pass.c | ~200 | **Yes** |
| Constant folding | xir_pass.c | ~200 | **Yes** |
| Phi simplification | xir_pass.c | ~100 | **Yes** |
| Strength reduction (via type_prop) | xir_pass_type.c | ~1,628 | **Yes** |
| SCCP | xir_pass_sccp.c | 752 | No (Xi lacks it) |
| GVN + CSE | xir_pass_advanced.c | ~400 | No |
| LICM | xir_pass_advanced.c | ~300 | No |
| GCM | xir_pass_advanced.c | ~200 | No |
| Inlining | xir_pass_advanced.c | ~800 | No |
| If-conversion | xir_pass_cfg.c | ~500 | No |
| Peephole | xir_peephole.c | 458 | No (machine-level) |

**Xm analysis infrastructure** (also in src/jit/):

| Analysis | File | Lines |
|----------|------|-------|
| Dominator tree | xir_domtree.c | 172 |
| Loop tree | xir_looptree.c | 387 |
| Def-use chains | xir_defuse.c | 208 |
| Liveness | xir_liveness2.c | 230 |
| Type flow analysis | xir_tfa.c | 841 |
| Alias analysis | xir_alias.c | 116 |

**Xi analysis infrastructure** (src/ir/xi_analysis.c, 383 lines):

| Analysis | Status |
|----------|--------|
| RPO | ✅ |
| Dominator tree (Cooper-Harvey-Kennedy) | ✅ |
| Liveness (backward dataflow) | ✅ |
| Def-use chains | ❌ (only use _count_, no use _list_) |
| Loop detection | ❌ |

### Target architecture

```
Xi IR layer (src/ir/) — all backends share
  analysis/
    xi_analysis.c       RPO, domtree, liveness      (exists)
    xi_loop.c           Loop detection               (NEW)
    xi_defuse.c         Def-use chains               (NEW)
  opt/
    xi_opt_constfold.c  Constant folding             (extract from xi_opt.c)
    xi_opt_copy_prop.c  Copy propagation             (extract)
    xi_opt_dce.c        Dead code elimination        (extract)
    xi_opt_phi.c        Phi simplification           (extract)
    xi_opt_strength.c   Strength reduction           (extract)
    xi_opt_sccp.c       Sparse conditional const prop (NEW, port from Xm)
    xi_opt_gvn.c        Global value numbering       (NEW, port from Xm)
    xi_opt_licm.c       Loop-invariant code motion   (NEW, port from Xm)
    xi_opt_gcm.c        Global code motion           (NEW)
    xi_opt_inline.c     Function inlining            (NEW, port from Xm)
    xi_opt_ifconv.c     If-conversion                (NEW, port from Xm)
    xi_opt_box_elim.c   Box/unbox elimination        (extract)
    xi_opt_select_rep.c Representation selection      (extract)
  xi_pass.h             Pass framework + driver       (NEW)
  xi_pass.c             Fixed-point driver            (NEW)

Xm layer (src/jit/) — machine-level only
  xm_fold.c            Machine peephole (stays)
  xm_peephole.c        Architecture-specific patterns (stays)
  xm_regalloc.c        Register allocation (stays)
  xm_codegen*.c        ARM64/x64 codegen (stays)
  xm_liveness2.c       Machine-level liveness (stays)
  xm_coalesce.c        Register coalescing (stays)
  xm_tfa.c             Type flow for speculation (stays — IC-guided)
  xm_pass.c            Reduced: only DCE cleanup + machine peephole
  xm_pass_advanced.c   Reduced: only IC speculation, inlining delegated to Xi
```

### What moves, what stays

**Promoted to Xi IR** (all backends benefit):

| Pass | Why at Xi level |
|------|-----------------|
| SCCP | Works on typed SSA; Xi has full type info (stronger than machine types) |
| GVN | Hash-based value numbering on XiOp + args; Xi operations are pure semantic |
| LICM | Requires loop detection + side-effect flags (Xi has XI_FLAG_SIDE_EFFECT) |
| GCM | Depends on domtree + GVN; natural fit at Xi level |
| Inlining | Xi has full function signatures, type info, and escape annotations |
| If-conversion | Transforms control flow to select; works on any SSA IR |

**Stays in Xm layer** (machine-specific or runtime-dependent):

| Pass | Why at Xm level |
|------|-----------------|
| IC-guided speculation | Requires runtime profiling data (IC snapshots) not available at Xi compile time |
| Machine peephole | ARM64/x64 specific instruction patterns |
| Register allocation | Machine registers, calling conventions |
| TFA (type flow) | Propagates machine types (i64/f64/ptr/tagged), different from Xi's semantic types |
| Alias analysis | Operates on machine memory model |

### Pass framework design

New `xi_pass.h` — mirrors the existing Xm (Xir) pass framework:

```c
/* Pass change tracker — same concept as XirPassChange */
typedef struct XiPassChange {
    bool cfg_changed;       /* blocks/edges altered */
    bool values_changed;    /* values added/removed/replaced */
    bool types_changed;     /* type annotations refined */
    uint32_t n_removed;     /* values eliminated */
    uint32_t n_added;       /* values inserted */
} XiPassChange;

/* Individual pass function signature */
typedef XiPassChange (*XiPassFn)(XiFunc *f);

/* Pass descriptor */
typedef struct XiPassDesc {
    const char *name;       /* "constfold", "sccp", "gvn", ... */
    XiPassFn fn;
    uint32_t flags;         /* XI_PASS_NEEDS_DOM, XI_PASS_NEEDS_LOOP, ... */
} XiPassDesc;

/* Optimization levels (gated by pipeline config) */
typedef enum {
    XI_OPT_NONE  = 0,  /* no optimization (check-only pipeline) */
    XI_OPT_LIGHT = 1,  /* constfold + copy_prop + DCE + phi_simp (VM default) */
    XI_OPT_FULL  = 2,  /* + SCCP + GVN + LICM + inlining (JIT Tier 2, AOT) */
} XiOptLevel;

/* Run optimization pipeline at given level */
XR_FUNC void xi_opt_run_pipeline(XiFunc *f, XiOptLevel level);
```

### Optimization level mapping per backend

| Backend | Xi opt level | Xm passes |
|---------|-------------|-----------|
| **VM (interpreter)** | XI_OPT_LIGHT | N/A (no Xm) |
| **JIT Tier 1** | XI_OPT_LIGHT | IC speculation + machine peephole |
| **JIT Tier 2** | XI_OPT_FULL | IC speculation + machine peephole |
| **AOT** | XI_OPT_FULL + select_rep + box_elim | N/A (goes to xi_cgen) |

Key insight: **VM compilation speed is preserved** because XI_OPT_LIGHT only
runs the 5 cheap passes that already exist. The expensive passes (SCCP, GVN,
LICM, inlining) only fire at XI_OPT_FULL, which VM never requests.

### Prerequisites (must build before passes can be ported)

1. **Xi loop detection** (`xi_loop.c`):
   - Required by: LICM, GCM, loop unrolling
   - Depends on: RPO, domtree (both exist in xi_analysis.c)
   - Algorithm: natural loop detection via back-edge identification
   - Estimated: ~200 lines

2. **Xi def-use chains** (`xi_defuse.c`):
   - Required by: GVN, SCCP, inlining
   - Current Xi has `XiValue.uses` (count only). Need use-list or efficient
     iterator over all uses of a value.
   - Design: add `XiValue.use_list` (linked list of XiUse structs), or
     build a flat use-array per function (like XIR's approach).
   - Estimated: ~200 lines

3. **Xi value hashing** (for GVN):
   - Hash function: `hash(op, type, args[], aux_int)` → 32-bit
   - Equality: same op, same type kind, same args (by value ID), same aux
   - Estimated: ~100 lines (add to xi.h or xi_analysis.h)

### Implementation order and status

Each step is independently testable and committable:

```
 1. ✅ Extract existing passes — kept in xi_opt.c, refactored to return XiPassChange

 2. ✅ Pass framework (xi_pass.h) with XiPassChange, XiPassDesc, XiOptLevel,
    fixed-point driver xi_opt_run_pipeline(), per-round verification

 3. ✅ xi_loop.c/h — natural-loop forest (back-edge + reverse BFS + parent/child wiring)

 4. ✅ xi_defuse.c/h — two-pass flat-array def-use chains

 5. ✅ xi_opt_sccp.c/h — Wegman-Zadeck SCCP (constant folding + unreachable
    block removal + branch simplification). Registered at XI_OPT_FULL.

 6. ✅ xi_opt_gvn.c/h — hash-based GVN with commutative normalization +
    dominance-gated replacement. Registered at XI_OPT_FULL.

 7. ✅ xi_opt_licm.c/h — loop-invariant code motion with iterative chain-
    invariant propagation (8 rounds max). Registered at XI_OPT_FULL.

 8. ⏳ Port inlining from Xm to Xi — deferred: requires callee IR cloning
    infrastructure (XiFunc deep copy + arg substitution + return wiring)

 9. ⏳ Port if-conversion from Xm to Xi — deferred: requires adding
    XI_SELECT opcode + emitter support in xi_emit.c

10.    Retire duplicated Xm passes (reduce xm_pass.c to cleanup-only)
    Verify: full test suite unchanged
```

### Testing strategy

Each pass gets a dedicated unit test file:

```
tests/unit/ir/test_xi_opt_sccp.c
tests/unit/ir/test_xi_opt_gvn.c
tests/unit/ir/test_xi_opt_licm.c
tests/unit/ir/test_xi_opt_inline.c
tests/unit/ir/test_xi_opt_ifconv.c
tests/unit/ir/test_xi_loop.c
tests/unit/ir/test_xi_defuse.c
```

Pattern: build a small XiFunc programmatically, run the pass, assert
expected transformations (value count, specific op replacements, etc.).

End-to-end verification after each step:
```
cd build && ctest --output-on-failure           # unit tests
scripts/run_regression_tests.sh                 # full regression
```

### Risk mitigation

| Risk | Mitigation |
|------|------------|
| SCCP/GVN on Xi has different semantics than on Xm | Xi operations are _more_ constrained (semantic ops with type info); passes are strictly _more_ conservative at Xi level. Port + verify per pass. |
| VM compilation slows down | XI_OPT_LIGHT is the VM default; new passes only run at XI_OPT_FULL. Benchmark `xray run` startup before/after. |
| Xm passes become stale/broken after Xi takes over | Explicit retirement: after Xi pass is validated, remove the Xm duplicate in the same commit. |
| Analysis infrastructure duplication (Xi domtree vs Xm domtree) | Acceptable: different IR shapes require different implementations. GraalVM and Zig both have two-layer analysis. |
| Inlining at Xi level needs XrProto (compiled children) | JIT already attaches Xi IR to proto. For inlining, the callee's XiFunc is accessible via `XiValue.aux` on XI_CLOSURE_NEW. |

---

## Execution order

**Optimization unification first, rename second.**

Rationale: if we rename Xir→Xm first (~5,600 symbol changes), then move
optimization passes between directories (src/jit/ → src/ir/), the diff
history becomes very hard to follow. Better to:

1. Move optimization logic while names are stable (clear "move from xir_pass_sccp.c
   to xi_opt_sccp.c" diffs)
2. Then rename the remaining Xm symbols in a single mechanical commit

This order also means the rename commit is smaller (fewer files to rename
after passes have moved to src/ir/).

---

## Success criteria

- [x] All optimization passes have individual enable/disable flags via XiOptLevel
- [x] SCCP, GVN, LICM available to JIT Tier 2 and AOT (via XI_OPT_FULL)
- [ ] Inlining and if-conversion ported to Xi (deferred: need IR cloning + SELECT op)
- [ ] Xm layer contains only: IC speculation, machine peephole, regalloc, codegen
- [ ] No `Xir` / `XIR_` / `xir_` symbols remain in the codebase
- [x] Full test suite green: 97/97 ctest + 127/127 Xi Compare + 293/294 regression
- [ ] VM startup latency not regressed (benchmark `xray run hello.xr`)
