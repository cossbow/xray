# 066b — Xi AOT: Migrate Legacy AOT to Xi Pipeline

**Companion**: `066-xi-aot-audit.md` — the audit report that motivates this plan.

## Goal

Replace the legacy AOT path (bytecode → xir_builder → xcgen) with
the Xi IR path (AST → xi_lower → xi_opt → xi_cgen) for **all** AOT
compilation, including multi-module.  Adopt proven logic from the
legacy path rather than reimplementing from scratch, then delete the
legacy code.

**Principle**: extract → adapt → verify → delete.  Never blind-delete.

---

## Current State

| Component | Legacy (`xaot_build`) | Xi (`xaot_build_xi`) |
|-----------|----------------------|---------------------|
| Single-file AOT | ✅ Full | ✅ Full |
| Multi-module (bundle + topo) | ✅ Full | ❌ Missing |
| Cross-module imports | ✅ export map + GETSHARED | ❌ Missing |
| Class / vtable / inheritance | ✅ preregister_classes | ❌ Missing |
| Struct promotion (shapes) | ✅ xcgen_struct.c | ❌ Missing |
| Closure escape analysis | ✅ xcg_prescan_closure_escape | ❌ Missing |
| Non-escaping closure inlining | ✅ xcgen_call.c | ❌ Missing |
| Feature inference (coro/chan/...) | ✅ infer_features_proto | ❌ Missing |
| Try/catch/finally codegen | ✅ setjmp/longjmp frame | ⚠️ Stub |
| Defer | ✅ LIFO _defer_N pattern | ⚠️ Stub |
| Indirect call (dynamic dispatch) | ✅ xrt_invoke_method | ⚠️ Stub |
| Method dispatch (>2 args) | ✅ xrt_call_variadic | ⚠️ Missing |
| Dead vreg elimination | ✅ seed-then-propagate | ❌ SSA uses count exists but not in cgen |
| Bump alloc / ARC runtime | ✅ xrt_bump + xrt_arc | ⚠️ No init in main() |
| Tests (basic) | 20 pass | covered by xi compare |
| Tests (modules) | 7 pass | ❌ None |

**Legacy code to retire after migration** (~14,400 lines):

| File | Lines | Content |
|------|-------|---------|
| `src/jit/xir_builder.c` | 2,946 | bytecode → XIR SSA builder |
| `src/jit/xir_builder_call.c` | 2,262 | call/method lowering |
| `src/jit/xir_builder_misc.c` | 1,717 | misc opcodes |
| `src/jit/xir_builder_object.c` | 1,325 | object/import |
| `src/jit/xir_builder.h` | 233 | public API |
| `src/jit/xir_builder_internal.h` | 129 | internals |
| `src/aot/xcgen.c` | 923 | per-function C codegen |
| `src/aot/xcgen_call.c` | 752 | call emission |
| `src/aot/xcgen_expr.c` | 1,235 | expression emission |
| `src/aot/xcgen_intrinsic.c` | 989 | intrinsic lowering |
| `src/aot/xcgen_prescan.c` | 763 | reachability + DCE |
| `src/aot/xcgen_stmt.c` | 251 | statement emission |
| `src/aot/xcgen_struct.c` | 480 | struct promotion |
| `src/aot/xcgen.h` | 308 | header |
| `src/aot/xcgen_struct.h` | 106 | header |

---

## Migration Strategy

**Not rewrite-from-scratch. Extract from legacy, adapt to Xi IR, test, then delete.**

For each capability gap in xi_cgen, the process is:
1. Read the corresponding xcgen handler — understand the invariants
2. Translate to Xi IR terms (XiValue/XiBlock vs XirRef/XirBlock)
3. Where the logic is pure C emission, port nearly verbatim
4. Where the logic depends on XirFunc/XirRef, replace with Xi equivalents
5. Run existing AOT test suite against new path
6. When all tests pass on Xi path, wire CLI to call `xaot_build_xi` and remove old path

---

## Phases

### S0: Multi-Module Framework (Xi AOT driver) — P0

**Goal**: `xaot_build_xi` supports multi-module bundle compilation.

**Extract from legacy**:
- `xaot_driver.c`: bundle discovery (L824-L862) — reuse verbatim
- `derive_module_name()` (L617-L631) — reuse verbatim
- `derive_import_string()` (L635-L666) — reuse verbatim
- `XaotModuleInfo` struct (L285-L296) — adapt (remove xir-specific fields)
- `free_modules()` / `free_import_map()` — reuse verbatim
- `assemble_c_source()` (L727-L785) — adapt to Xi output format

**New logic**:
- Per-module: parse → analyze → `xi_pipeline_compile_program()` with `run_select_rep=true`
- `xi_cgen_program()` per module → collect C fragments
- Generate multi-module `main()` calling inits in topo order
- Add `xrt_bump_enabled = 1; xrt_arc_init();` and `xrt_bump_destroy();` to main()

**Not needed** (Xi path avoids bytecode entirely):
- `build_shared_proto_map()` — Xi uses `XI_GET_SHARED/XI_SET_SHARED` natively
- `collect_aot_protos()` — Xi has all funcs in `XiFunc.children`
- `preregister_classes()` — Xi has `XI_CLASS_CREATE` with full AST info

**Verification**:
- All 20 basic AOT tests pass through Xi path
- Basic multi-module test (import_fn) passes through Xi path

### S1: Cross-Module Import Resolution — P0

**Goal**: `import "X"` + `X.foo()` works in Xi AOT.

**Current Xi lowering**: `xi_lower.c` lowers `import "X"` to `XI_CALL_BUILTIN("import")`.
This is a runtime call that won't work in standalone AOT.

**Design**: At the AOT driver level (not in xi_lower):
1. Build a cross-module export map: `{module_path, export_name} → shared_index`
   - Extract `build_export_map()` logic from legacy (L668-L723)
   - Adapt: instead of scanning bytecode for OP_EXPORT, scan Xi IR for XI_SET_SHARED
     whose variable is exported (marked at lowering time via `XiFunc.exports[]`)
2. Add `XiExport` metadata to `XiFunc`: name + shared_index for each export
   - Lowerer populates this from `AST_EXPORT` nodes
3. Post-pipeline pass: rewrite import references
   - Walk IR of importing module, replace `XI_CALL_BUILTIN("import")` + `XI_LOAD_FIELD`
     with `XI_GET_SHARED` pointing to exporter's shared index
   - Or: add import metadata to `xi_cgen` so it emits `xrt_shared[N]` directly

**Extract from legacy**:
- `build_export_map()` (L668-L723) — adapt to Xi IR terms
- `collect_exports()` (L474-L589) — greatly simplified, no bytecode scanning
- Import map data structures (`XirAotImportEntry` → `XiAotImportEntry`)

**Verification**:
- All 7 module AOT tests pass through Xi path
- VM-AOT diff is zero for all module tests

### S2: Feature Inference on Xi IR — P1

**Goal**: Infer runtime feature requirements from Xi IR (instead of bytecode).

**Extract from legacy**:
- `XaotFeatureSet` struct + `apply_feature_closure()` — reuse verbatim
- `infer_features_proto()` — rewrite to walk Xi IR opcodes:

| Legacy (bytecode) | Xi IR equivalent |
|-------------------|------------------|
| `OP_GO` | `XI_GO` |
| `OP_AWAIT` | `XI_AWAIT` |
| `OP_CHAN_NEW/SEND/RECV` | `XI_CHAN_NEW/XI_CHAN_SEND/XI_CHAN_RECV` |
| `OP_SCOPE_ENTER` | `XI_SCOPE_ENTER` |
| `OP_TRY/CATCH/THROW` | `XI_TRY/XI_CATCH/XI_THROW` |
| `OP_IS` | `XI_IS` |
| `OP_TYPEOF` | `XI_TYPEOF` |
| `OP_IMPORT` + constant | `XI_CALL_BUILTIN("import")` + aux string |

**New function**: `xi_aot_infer_features(XiFunc *f, XaotFeatureSet *fs)` — recursive walk.

**Verification**: Feature output matches legacy path on identical input.

### S3: Class / Inheritance in xi_cgen — P1

**Goal**: `XI_CLASS_CREATE` emits proper C code for class construction.

**Current state**: `xi_cgen.c` has no handler for `XI_CLASS_CREATE`.

**Legacy approach** (in old xcgen):
- `preregister_classes()` scans bytecode for `OP_CLASS_CREATE_FROM_DESCRIPTOR`
- Creates runtime `XrClass` objects at compile time for type info
- Emits `xrt_class_register(name, parent, nfields)` calls in generated C

**Xi approach**:
- `XI_CLASS_CREATE` carries `XiClassData*` with full AST info
- `xi_cgen` emits: `xrt_class_new("Name", parent_ref, nfields)` call
- Method closures already lowered as `XI_CLOSURE_NEW` children
- Constructor registered via method table setup

**Extract from legacy**:
- `XcgenClassInfo` struct concept — adapt
- `xcgen_register_class()` pattern — adapt to Xi data model
- C emission pattern for class registration — port verbatim

**Verification**: `class_basic`, `class_inherit`, `class_methods` AOT tests pass.

### S4: Closure / Upvalue in xi_cgen — P1

**Goal**: Closures that capture variables produce correct C code.

**Current state**: `xi_cgen.c` emits `XR_NULL_VAL /* closure */` placeholder.

**Legacy approach** (in old xcgen):
- **Escaping closures**: emit `xrt_closure_new(fn_ptr, nupvals)` + upval init
- **Non-escaping closures**: detected by `xcg_prescan_closure_escape()`,
  pass upvals as extra function parameters (no heap allocation)

**Xi approach**:
- `XI_CLOSURE_NEW` already has `XiCapture[]` with full upvalue info
- Emit struct allocation + upval stores for escaping
- For non-escaping (local-only, never stored/returned): inline as extra params

**Extract from legacy**:
- Escape analysis logic from `xcgen_prescan.c` `xcg_prescan_closure_escape()` — adapt
- Non-escaping call pattern from `xcgen_call.c` — adapt
- Closure creation emission — port to Xi value model

**Verification**: `closures`, `closure_counter`, `nested_closure` AOT tests pass.

### S5: Try/Catch/Finally + Defer — P1

**Goal**: Exception handling and defer emit correct C code.

**Current state**: `xi_cgen.c` has stubs (XI_TRY → "0", XI_DEFER → "TODO").

**Legacy approach** (in old xcgen):
- `setjmp`/`longjmp` exception frames
- `xrt_exc_frame` struct with `prev` chain
- `xrt_exc_top` thread-local for current frame
- Defer: `_defer_N` locals + LIFO cleanup before returns

**Extract from legacy**:
- Exception frame emission pattern from `xcgen_expr.c` TRY/CATCH/FINALLY handlers
- Defer pattern — port to Xi block structure
- `exc_catch_frame[]` tracking — adapt to XiBlock IDs

**Verification**: `try_catch`, `finally`, `defer` AOT tests pass.

### S6: Indirect Calls + Dynamic Dispatch — P2

**Goal**: Calling closures stored in variables, method dispatch on user classes.

**Current xi_cgen**: Only resolves static calls (XI_CLOSURE_NEW → direct call).
Falls back to `XR_NULL_VAL /* TODO: indirect call */`.

**Legacy approach**:
- `xrt_invoke_closure(closure_val, args, nargs)` for unknown callees
- `xrt_invoke_method_sentinel(recv, method_id, args, nargs)` for user methods

**Extract from legacy**:
- Indirect call emission from `xcgen_call.c` — adapt to Xi value model
- Method dispatch with >2 args: `xrt_call_variadic()` pattern

**Verification**: Higher-order function tests pass, class method dispatch tests pass.

### S7: Struct Promotion + Advanced Optimizations — P2

**Goal**: Object-literal shapes promoted to C structs for field access.

**Legacy approach**: `xcgen_struct.c` + `xcgen_prescan.c`:
- `xcgen_collect_shapes()` scans bytecode for NEWOBJECT patterns
- Maps field names → struct member names
- LOAD_FIELD/STORE_FIELD → direct `->field` access (no map lookup)

**Decision**: Defer to later. Xi IR's typed SSA already carries type info,
and future type specialization pass can achieve the same without bytecode
scanning. The Xi approach would be to:
1. Recognize `XI_ALLOC + XI_STORE_FIELD*` patterns in xi_opt
2. Promote to typed struct with known layout
3. `xi_cgen` emits direct field access for promoted objects

For now, generic `xrt_map_set/get` is correct and sufficient.

### S8: Wire CLI + Delete Legacy — P0 (after S0-S1 pass)

**Goal**: `xray build --native` calls Xi path exclusively.

**Steps**:
1. In `xcmd_build.c`: change `cmd_build_native()` to call `xaot_build_xi()` instead of `xaot_build()`
2. Run full AOT test suite: 20 basic + 7 module tests
3. Run regression: `scripts/run_regression_tests.sh`
4. Delete `xaot_build()` from `xaot_driver.c`
5. Delete all `src/aot/xcgen*.c` and `src/aot/xcgen*.h` (5,807 lines)
6. Delete all `src/jit/xir_builder*.c` and `src/jit/xir_builder*.h` (8,612 lines)
7. Remove from `CMakeLists.txt`
8. Clean up `xaot_driver.h`: remove `xaot_build()` declaration
9. Verify 96/96 ctest + full regression still pass

**Also clean up**:
- `src/jit/xir.h`: remove `XirTypeKind`, `VTAG_*`, `vtag_to_*` / `*_to_vtag` conversions (~200 lines)
  - Only after confirming JIT (non-AOT) path doesn't use them
- `src/aot/xrt_value.h`: merge AOT-specific tags into `runtime/value/xvalue.h`
  via `#ifdef XR_AOT_STANDALONE` guard, delete `xrt_value.h`

---

## Execution Order

```
S0 (multi-module framework)    ─── must-have for parity
 └─ S1 (cross-module imports)  ─── must-have for parity
     └─ S8 (wire CLI + delete) ─── delete legacy

S2 (feature inference)         ─── parallel with S0/S1
S3 (class codegen)             ─── after S0
S4 (closure codegen)           ─── after S0
S5 (try/catch/defer)           ─── after S0
S6 (indirect calls)            ─── after S4
S7 (struct promotion)          ─── deferred
```

**Critical path**: S0 → S1 → S8 (delete).
S3/S4/S5 can proceed in parallel after S0.

---

## Verification Gates

| Gate | Criteria |
|------|----------|
| S0 done | 20/20 basic AOT via Xi path, VM-AOT diff = 0 |
| S1 done | 7/7 module AOT via Xi path, VM-AOT diff = 0 |
| S3 done | class_basic, class_inherit, class_methods, class_method_dispatch, class_string_fields AOT pass |
| S4 done | closures, collection_filter, collection_methods AOT pass |
| S5 done | try_catch, finally, defer AOT pass |
| S8 done | 96/96 ctest, 27/27 AOT, full regression pass. Legacy files deleted. |

---

## Estimated Effort

| Phase | Effort | Dependencies |
|-------|--------|-------------|
| S0 | 2 days | none |
| S1 | 1.5 days | S0 |
| S2 | 0.5 days | none |
| S3 | 1.5 days | S0 |
| S4 | 2 days | S0 |
| S5 | 1.5 days | S0 |
| S6 | 1 day | S4 |
| S7 | deferred | — |
| S8 | 0.5 days | S0+S1+S3+S4+S5 |
| **Total** | **~10 days** | |
