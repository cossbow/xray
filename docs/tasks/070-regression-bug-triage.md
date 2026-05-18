# Regression Test Bug Triage

> Date: 2026-05-05
> Baseline: 294 files, 137 pass, 154 fail, 3 timeout
> ctest: 98/98 pass, Xi Compare: 127/127 pass

---

## Summary by Category

| Cat | Count | Error Pattern | Root Cause | Priority |
|-----|-------|---------------|------------|----------|
| A | 64 | `method 'X' called on unsupported type` | stdlib static methods not dispatched in VM | low — feature gap |
| B | 16 | `method 'constructor' called on unsupported type` | class `new` expression / constructor dispatch broken | high — compiler bug |
| C | 14 | `go: closure captures non-thread-safe variables` | coroutine capture safety overly strict or test uses `let` | medium — design review |
| D | 11 | `attempt to call a non-function value` | function hoisting / cell wrapping incomplete | high — compiler bug |
| E | 19 | `assertion failed at line N: values not equal` | miscompile — wrong value computed | high — compiler bug |
| F | 12 | `type 'null' does not support property access` | `typeof` returns null; type conversion returns null | high — compiler bug |
| G | 5 | parse error / compile error | unsupported syntax (match guard, etc.) | low — feature gap |
| H | 10 | misc (timeout, network, gc stress) | runtime / environment | low — skip |
| **Total** | **~154** | | | |

Notes: some files have multiple error categories; counts are approximate first-error based.

---

## Category A: stdlib static method dispatch (64 files)

**Pattern**: `method 'parse/compile/create/now/...' called on unsupported type`

These are calls to stdlib module static methods like `Json.parse()`, `Regex.compile()`,
`DateTime.now()`, `Math.floor()`, `Os.getenv()`, `Crypto.sha512()`, etc.

**Root cause**: The VM does not dispatch static method calls on module-level class objects.
The Xi pipeline emits `OP_CALL_METHOD` but the VM's method dispatch cannot resolve static
methods on type objects (as opposed to instance methods).

**Priority**: Low — this is a feature gap, not a regression. These stdlib modules were
designed for the old compiler and the Xi pipeline hasn't implemented static method dispatch.

**Files** (representative):
- `1000_json.xr` through `1010_json_*.xr` — Json.parse/stringify
- `1020_math.xr` through `1023_math_*.xr` — Math.floor/abs/sin/...
- `1100_regex.xr` through `1130_regex_*.xr` — Regex.compile
- `1010_time.xr`, `1011_time_edge.xr` — Time.now/monotonic
- `1160_os_basic.xr`, `1161_os_extended.xr` — Os.getenv/...
- `1190_io_*.xr` — Io.readFile/tempFile
- `1200_log_*.xr` — Log.info/...
- `1300_encoding.xr` through `1309_xml_*.xr` — encoding modules
- `1400_crypto_*.xr` — Crypto.sha512/hmac/...
- `1410_url_*.xr` — Url.parse/encode
- `1420_datetime_*.xr` — DateTime.now/create/...
- `1430_net_*.xr` — Net.lookup/listen/...
- `1500_http_*.xr` — Http modules

---

## Category B: class constructor dispatch (16 files)

**Pattern**: `method 'constructor' called on unsupported type`

**Root cause**: `new ClassName(args)` fails because the class object's constructor
cannot be resolved at runtime. This affects both user-defined classes and stdlib classes.

**Priority**: High — affects a large portion of language functionality.

**Files**:
- `0560_method_chaining.xr`, `0570_hoisting.xr`
- `0720_stringbuilder.xr`
- `0800_class_basic.xr` through `0870_advanced.xr`
- `0890_final.xr`, `0910_typeof.xr`
- `1150_string_byte_ops.xr`

---

## Category C: coroutine capture safety (14 files)

**Pattern**: `go: closure captures non-thread-safe variables, cannot run in coroutine`

**Root cause**: The runtime rejects coroutine closures that capture `let` variables.
Some tests may need updating to use `shared const` or parameter passing; others may
reveal overly strict capture checking.

**Priority**: Medium — needs design review to distinguish real safety violations from
false positives.

**Files**:
- `0740_context_auto_select.xr`, `0741_context_buffer_isolation.xr`
- `0751_threshold_branch.xr`, `0760_concurrent_concat_property.xr`
- `0770_string_pool_concurrent_property.xr`
- `1102_go_await.xr`, `1104_coroutine_combined.xr`
- `1107_go_deferred.xr` through `1112_go_block.xr`
- `1142_shared_move.xr`, `1146_multicore_string_concat.xr`

---

## Category D: function call failures (11 files)

**Pattern**: `attempt to call a non-function value`

**Root cause**: Function hoisting / cell wrapping not complete. Some cases may be
related to the hoisting fix we just applied; others may be different root causes
(arrow functions, tail recursion, nested iterators).

**Priority**: High — core compiler bug.

**Files**:
- `0302_int64_native.xr` — int64 uses functions
- `0511_tail_recursion.xr` — tail call optimization
- `0530_arrow.xr` — arrow function expressions
- `0880_custom_iterator.xr`, `0881_nested_iterator_combo.xr` — iterator protocol
- `0911_dump_builtin.xr`, `0912_typename_builtin.xr`, `0913_copy_builtin.xr`
- `1203_nullable_type.xr`, `1204_class_type.xr`, `1205_typeof_operator.xr`

---

## Category E: assertion failures / wrong values (19 files)

**Pattern**: `assertion failed at line N: values not equal`

**Root cause**: Various miscompile bugs — the code runs but produces wrong results.
Each needs individual investigation.

**Priority**: High — real compiler bugs.

**Files**:
- `0210_scope.xr` — scope variable capture (cell read after hoisted fn)
- `0230_type_annotation.xr` — type annotation handling
- `0320_logical.xr` — logical operator evaluation
- `0390_strict_equality.xr` — equality comparison
- `0520_closure.xr`, `0521_cell_upval.xr` — closure/upvalue
- `0650_nested_higher_order.xr`, `0651_nested_higher_order_combo.xr`
- `0930_practical.xr`, `0940_edge_cases.xr`
- `1201_function_inference.xr`, `1206_generic_functions.xr`
- `1210_multi_return.xr`, `1230_function_type_annotation.xr`, `1231_type_alias.xr`
- `1301_is_as_union.xr`
- `1302_json_basic.xr`, `1303_json_type_alias.xr`, `1307_json_comprehensive.xr`

---

## Category F: typeof / type conversion returns null (12 files)

**Pattern**: `type 'null' does not support property access '.int'`

**Root cause**: `typeof` operator or type conversion functions return null instead
of the expected value. Likely the `XI_TYPEOF` emission or `as` operator implementation.

**Priority**: High — compiler bug.

**Files**:
- `0301_int48_overflow.xr` — type conversion
- `0600_array.xr`, `0620_map.xr`, `0630_set.xr` — container typeof
- `0920_type_conversion.xr` — explicit conversion
- `1200_basic_inference.xr`, `1202_generic_inference.xr` — type system tests

---

## Category G: parse/compile errors (5 files)

**Pattern**: parse errors on unsupported syntax

**Root cause**: Feature not yet implemented in parser/analyzer.

**Priority**: Low — feature gap.

**Files**:
- `0440_match.xr` — match guard syntax (`n if n < 0 =>`)
- `0661_jit_cross_boundary.xr` — JIT specific
- `0680_deep_eq_cyclic.xr` — deep equality
- `0750_short_string_concat.xr` — string optimization

---

## Category H: runtime / environment (10 files)

**Pattern**: timeout, network dependency, GC stress

**Priority**: Low — skip or environment-specific.

**Files**:
- `1106_select.xr`, `1113_select_send.xr`, `1114_select_after.xr` (timeout)
- `1207_gc_stress.xr`, `1204_gc.xr`, `1205_gc_incremental_pressure.xr`, `1206_gc_enhanced.xr`
- `1501_http_*.xr`, `1510_http2_client.xr`, `1520_websocket.xr` (network)

---

## Recommended Fix Order

1. **Cat E** (assertion failures) — 19 files, likely a few root causes
   - Start with `0520_closure.xr`, `0521_cell_upval.xr` (closure/cell — related to current work)
   - Then `0210_scope.xr`, `0320_logical.xr` (may share root cause)
2. **Cat D** (call failures) — 11 files
   - `0530_arrow.xr` (arrow functions)
   - `0511_tail_recursion.xr` (tail calls)
   - Builtin functions (`0911-0913`)
3. **Cat F** (typeof null) — 12 files, likely 1-2 root causes
4. **Cat B** (constructor) — 16 files, likely 1 root cause
5. **Cat C** (coroutine capture) — 14 files, needs design review
6. **Cat A** (stdlib) — 64 files, feature work, defer to after refactor
7. **Cat G/H** — low priority
