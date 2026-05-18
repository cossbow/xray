# Prelude

The **prelude** is xray's set of built-in type names that every program
sees without writing `import prelude`. It plays the same role as Rust's
`core::prelude::v1` or Haskell's `Prelude`: a small, stable, always-on
table of names that the compiler — not the user — owns.

```xray
// All of these compile without any import:
let xs: Array<int> = [1, 2, 3]
let m:  Map<string, int> = {"a": 1}
let n:  BigInt = 10n ** 100n
let dt: DateTime = datetime.now()
let r:  Regex = regex.compile("\\d+")
```

The prelude is loaded automatically into every full-runtime isolate.
Minimal-runtime isolates (created without `xray_isolate_setup_full`)
do not have it; they cannot use any of these names.

## Catalogue

| Name | Kind | Notes |
|---|---|---|
| `Array<T>` | generic-1 | dynamic array, primary indexed collection |
| `Map<K, V>` | generic-2 | hash map; `K` must be hashable |
| `Set<T>` | generic-1 | hash set |
| `Channel<T>` | generic-1 | coroutine channel; `Channel(8)` constructs one |
| `Json` | singleton | dynamic value (numbers, strings, arrays, objects, null) |
| `Bytes` | generic-1 | synonym for `Array<uint8>`; same underlying GC type. Mutable. |
| `BigInt` | simple | arbitrary-precision integer; `10n` literal |
| `DateTime` | simple | absolute moment in time, ISO 8601 friendly |
| `Range` | simple | half-open integer range, e.g. `for i in 0..10` |
| `Regex` | simple | compiled regular expression |
| `StringBuilder` | simple | mutable string accumulator |
| `Logger` | simple | structured logger, returned by `log.child(...)` |
| `NetConn` | simple | TCP/TLS/UDP connection handle |
| `NetListener` | simple | TCP listener handle |

**Kinds** map to the `XrPreludeKind` enum that the parser dispatches
on:

- **simple** — written bare, e.g. `let dt: DateTime`. Resolves to
  `XR_KIND_INSTANCE` with `class_name == "<Name>"`.
- **generic-1 / generic-2** — requires angle brackets:
  `Array<int>`, `Map<string, int>`. Inference may fill in
  `Array<unknown>` in a few places where bare containers are allowed.
  `Bytes` is a `GENERIC_1` alias whose GC type is `XR_TARRAY` —
  it shares the same runtime representation as `Array<uint8>`.
- **singleton** — `Json` resolves to a process-wide singleton
  `XrType` (the analyzer keeps it interned).

## What is *not* in the prelude

xray draws a sharp line between **language primitives** (always
recognised, never overridable) and **prelude entries** (always
recognised, *user-overridable*).

Primitives — these are lexer keywords:

```
int / int8 / int16 / int32 / int64
uint8 / uint16 / uint32 / uint64
float / float32 / float64
bool / string / void / null / unknown / never
```

Prelude entries are *not* lexer keywords. The lexer hands them through
as plain `TK_NAME` tokens; the parser consults the prelude registry to
discover what they mean. The consequence is that user-defined classes
shadow prelude entries:

```xray
class Array {
    fn surprise(): string { return "shadowed" }
}

let a = Array()  // refers to your class, not the prelude entry
```

This matches Rust and Haskell. Note that the shadow only applies
within the user's module — a `class Array {}` does not affect other
imported modules' usage of `Array<T>`.

## Adding a new native type

Suppose you want a `Uuid` type backed by a C struct in `stdlib/uuid/`.
The end-to-end flow is **two files, two lines, one function**:

### 1. Add a row to `prelude_types.def`

```c
// stdlib/prelude/prelude_types.def
//                  name        native_type    kind
XR_PRELUDE_TYPE("Uuid",      XR_TUUID,      SIMPLE)
```

The `name` is the surface identifier the parser will accept. The
`native_type` is the runtime GC type id (defined in your `stdlib/uuid/`
module). The `kind` selects parser dispatch — `SIMPLE` for a leaf type,
`GENERIC_1` for `T<X>`, `GENERIC_2` for `T<X, Y>`.

### 2. Implement `xr_uuid_register_native_type`

In your stdlib module, expose one function that builds and registers
the runtime `XrClass`:

```c
// stdlib/uuid/uuid.c

static const XrNativeMethod uuid_methods[] = {
    {"toString", uuid_to_string, 1},
    {"version",  uuid_version,   1},
    {NULL, NULL, 0},
};

void xr_uuid_register_native_type(XrayIsolate *isolate) {
    static const XrNativeTypeInfo info = {
        .name = "Uuid",
        .gc_type = XR_TUUID,
        .methods = uuid_methods,
        .getters = NULL,
        .static_methods = NULL,
    };
    xr_register_native_type(isolate, &info);
}
```

`xr_register_native_type` is idempotent — calling it twice for the
same `gc_type` returns the existing class — so the function is safe
to invoke from anywhere.

### 3. Wire the registration into the prelude

```c
// stdlib/prelude/prelude.c
extern void xr_uuid_register_native_type(XrayIsolate *isolate);

void xr_prelude_register_all_native_types(XrayIsolate *isolate) {
    if (!isolate)
        return;
    /* ... existing entries ... */
    xr_uuid_register_native_type(isolate);
}
```

After this, every full-runtime isolate has `Uuid` available as a type
name. User code can write:

```xray
import uuid

let id: Uuid = uuid.gen()
print(id.toString())
```

with full LSP completion, hover documentation, and method-dispatch
support.

### 4. (Optional) typed cfunc signatures

If your stdlib module exposes factory cfuncs, advertise the precise
return type in the `XR_DEFINE_BUILTIN` declaration so the analyzer can
pick it up:

```c
XR_DEFINE_BUILTIN(uuid_gen, "gen", "(): Uuid", "Generate a new UUID")
```

Cross-module cfuncs that accept multiple handle types use union
syntax, exactly like xray source:

```c
XR_DEFINE_BUILTIN(net_close_handle, "close",
                  "(handle: NetConn | NetListener): void",
                  "Close a connection or listener")
```

The analyzer parses the union string at the same level as the
language parser, so users see uniform behaviour from either surface.

## Design rationale

**Why a registry instead of more lexer keywords?** Lexer keywords are
the most expensive form of "compiler knows about this name": they
consume identifier space, defeat user shadowing, and require parallel
edits to four files (lexer table, token enum, parser prefix table,
parse-type branch) to add a single name. Routing through `TK_NAME` +
prelude lookup compresses the cost of "add a type" to one line.

**Why eager registration?** `xr_prelude_register_all_native_types`
runs unconditionally during isolate init. The cost is that the linked
binary always contains the four stdlib modules whose XrClasses prelude
references (log, datetime, regex, net). The benefit is that
`let dt: DateTime = ...` just works — the user does not have to
remember to `import datetime` for the *class* even though they still
import the module to reach its factory functions. xray's "batteries
included" stance prefers this trade.

**Why is `Json` special?** `Json` is the dynamic-value escape hatch:
it carries no static structure, so the analyzer treats it as a
universal target for any read and any write (with a runtime check
inserted by the codegen for reads into typed sinks). To keep this
behaviour bit-perfect across compiler passes and back ends, `Json`
has its own `XrTypeKind` (`XR_KIND_JSON`) and a globally interned
singleton `XrType`. The prelude registers it as a `SINGLETON` so the
parser dispatches to the singleton constructor; everywhere else it is
just another instance of the dynamic-value contract.

## `Json` vs `unknown` — when to use which

xray has two different "I accept any value" types, and they answer
different questions. Pick the one that matches the *intent of the
receiver*, not the *shape of the caller*.

### `Json` — dynamic value the receiver may inspect

Use `Json` when the function or container actually does something
with the value's structure: reads a field, returns it back to the
caller, stores it for later retrieval, hashes it, prints it as JSON.

```xray
fn fieldCount(obj: Json): int          // inspects shape
fn store(key: string, value: Json)     // remembers and returns later
fn get(key: string): Json              // reader narrows on receipt
```

Properties:

- The receiver may use the dynamic API (`obj.keys()`, `obj["k"]`,
  `obj.has(...)`).
- A typed value (`int`, an instance, an array, etc.) flows in for
  free — it is a label-only coercion.
- `null` flows in for free — `Json` intrinsically includes null
  (`Json?` is rejected by the parser as redundant).
- Reading a `Json` into a typed sink compiles, but the codegen
  inserts an `OP_CHECKTYPE` so a wrong runtime shape becomes a
  clean exception.

### `unknown` — opaque value the receiver does not inspect

Use `unknown` when the function only **forwards, prints, or
otherwise ignores** the value's structure. The receiver makes no
guarantees about what is in there and the caller gets no help if it
relies on that.

```xray
fn debug(...args: unknown): void       // logger: stringifies, never reads fields
fn forward<T>(x: T, k: fn(unknown))    // pipeline stage, just passes along
```

Properties:

- The receiver cannot meaningfully call methods on the value
  without first narrowing (`as`, pattern match, runtime check).
- Designed for variadic / pipeline / logging APIs where requesting
  a precise type would be a lie.

### Quick test

If the function body would be improved by knowing it received a
`Json`, the parameter should be `Json`. Otherwise prefer `unknown`.

In practice, almost every stdlib API outside variadic logging picks
`Json` — anything that *stores* or *queries* a value benefits from
the dynamic-API contract.

## See also

- `stdlib/prelude/prelude.h` — prelude API
- `stdlib/prelude/prelude_types.def` — registry table
- `src/frontend/parser/xparse_type.c::try_resolve_prelude_type` — parser entry point
- `src/runtime/value/xtype.h::xr_is_json_coercion` — coercion matrix
- `src/runtime/value/xtype.h::xr_type_intrinsically_includes_null` — null-domain rule
- `docs/rules/architecture.md` — full module dependency layering
