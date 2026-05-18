# src/base Module Refactor Design

> **Principle**: No backward compatibility. Always choose best design.

## Overview

The `src/base` module is the L0 foundation layer. This refactor addresses correctness bugs,
performance bottlenecks, and code quality issues identified through code review.

## Changes by Priority

### P0 — Correctness Bugs

#### 1. xhashmap arena resize: silent failure in Release

**Problem**: `resize()` uses `XR_DCHECK(false, ...)` which becomes no-op in Release.
Arena-allocated hashmaps that exceed load factor will corrupt data silently.

**Fix**: Change to `XR_CHECK` (always-on). Arena maps must never hit resize — it's a
programming error that must be caught in all build modes.

#### 2. xconfig.c hardcoded version string

**Problem**: `xray_config_default()` has `.version = "0.32.4"` — stale and violates
the single-source-of-truth rule (CMakeLists.txt → xray_version.h).

**Fix**: `#include "xray_version.h"` and use `XRAY_VERSION_STRING`.

---

### P1 — Performance & Portability

#### 3. xhashmap: add cached hash + short hash filtering

**Problem**: Every probe calls `strcmp()`. No hash caching means rehash recomputes all hashes.

**Design**:
```c
typedef struct XrHashMapEntry {
    char *key;
    void *value;
    uint32_t hash;      // cached full hash (NEW)
} XrHashMapEntry;
```

- Store hash in entry on insert
- Probe: compare hash first, then `strcmp()` only on hash match
- Rehash: reuse cached hash, skip `strlen + FNV` recomputation
- Empty slot: `key == NULL && hash == 0`
- Tombstone: `key == NULL && hash != 0` (use hash field instead of value sentinel)

This eliminates the `XR_HASHMAP_TOMBSTONE` value hack and gives ~99% probe filtering.

#### 4. xlog.h portability + spinlock backoff

**Problem A**: `#include <unistd.h>` at header level breaks Windows.
**Fix**: Move to `.c` file behind `#ifdef XR_OS_POSIX`.

**Problem B**: Log spinlock is pure CAS spin with no backoff.
**Fix**: Add `XR_CPU_PAUSE()` in spin loop.

---

### P2 — Performance & Quality

#### 5. xr_hash_int: use fast integer mixing

**Problem**: `xr_hash_int(int64_t)` calls byte-by-byte FNV-1a (8 iterations).

**Fix**: Use splitmix64 finalizer — 3 multiply-xorshift ops, much faster:
```c
static inline uint32_t xr_hash_int(int64_t val) {
    uint64_t x = (uint64_t)val;
    x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9ULL;
    x = (x ^ (x >> 27)) * 0x94d049bb133111ebULL;
    x = x ^ (x >> 31);
    uint32_t h = (uint32_t)(x ^ (x >> 32));
    return h == 0 ? 1 : h;
}
```

Also move to inline in header (hot path function, same as xr_hash_bytes).

#### 6. xintmap: eliminate double initialization

**Problem**: `xr_calloc` zeros memory, then loop sets `key = 0xFFFFFFFF`.

**Fix**: Use `xr_malloc` + `memset(0xFF)` for key bytes, or just `xr_malloc` + init loop.
Since entries have pointer fields, safest is `xr_malloc` + loop (current loop is correct,
just drop the `xr_calloc`).

#### 7. Arena save/restore (savepoint mechanism)

**Design**: Add lightweight mark/rewind for temporary allocations:
```c
typedef struct XrArenaState {
    XrArenaSegment *head;
    char *position;
    size_t total_allocated;
} XrArenaState;

XR_FUNC XrArenaState xr_arena_save(XrArena *arena);
XR_FUNC void xr_arena_restore(XrArena *arena, XrArenaState state);
```

Constraint: restore only valid if no new segments were allocated after save
(i.e., all allocations since save fit in the current segment). This covers the
common case of tentative parsing. For simplicity, `restore` asserts same head segment.

#### 8. XR_AVEC_RESERVE macro

```c
#define XR_AVEC_RESERVE(arena, v, min_cap) do { \
    if ((v).cap < (min_cap)) { \
        int _nc = (min_cap); \
        void *_nb = xr_arena_alloc((arena), (size_t)_nc * sizeof(*(v).data)); \
        XR_CHECK(_nb != NULL, "XR_AVEC_RESERVE: arena alloc failed"); \
        if ((v).data && (v).count > 0) \
            memcpy(_nb, (v).data, (size_t)(v).count * sizeof(*(v).data)); \
        (v).data = _nb; \
        (v).cap = _nc; \
    } \
} while (0)
```

#### 9. Add assertions to weak files

Files needing more assertions: `xsource_cache.c`, `xswar.c`, `xlog.c`.
Add `XR_DCHECK` at public function entries per coding standard.

#### 10. TLS encapsulation in xarena.c

Wrap TLS variables in a struct with explicit exemption comment:
```c
// TLS cache: exempt from "no mutable file-scope globals" rule
// because __thread gives per-thread isolation (no shared state).
typedef struct {
    XrArenaSegment *segments;
    int count;
} XrArenaSegmentCache;

static __thread XrArenaSegmentCache tls_cache = {NULL, 0};
```

#### 11. xdefs.h formatting cleanup

Add blank lines between `#endif` and next `#if` blocks for readability.

---

## Non-Goals (deferred)

- Full Unicode case conversion tables (large data, low priority)
- xpoll.h dynamic fd array (current 64 limit sufficient for now)
- MSVC support for `XR_REALLOC` statement expressions (no Windows target yet)
- xmem_profiler thread safety (debug-only tool, acceptable trade-off)

## Testing

After all changes: `cd build && cmake .. && make -j8 && ctest --output-on-failure`
