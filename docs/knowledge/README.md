# MCP Knowledge — Single Source of Truth

This directory is the **single source of truth** for the language and
standard-library knowledge that the MCP server (`xray mcp-server`) hands
to AI clients via `xray_syntax_lookup`, `xray_stdlib_search`,
`xray_definition`, and the `xray://...` resource URIs.

It feeds a generator (`scripts/gen_mcp_knowledge.py`) that emits the
auto-generated C file `src/app/mcp/xmcp_knowledge_generated.c`, which is
committed alongside the docs so the build does not depend on Python at
compile time. CI re-runs the generator and asserts that the working tree
stays clean — any drift between docs and the generated C blob fails the
build, by design.

## Layout

```
docs/knowledge/
  topics/<id>.md       # one syntax topic per file (channel, coroutine, ...)
  stdlib/<module>.md   # one stdlib module per file (math, json, http, ...)
  resources/<id>.md    # long-form resources (cheatsheet, concurrency, ...)
```

Filenames are the canonical id; the file body is rendered Markdown that
ends up in the tool reply verbatim.

## Frontmatter

Every file starts with a YAML-like frontmatter block delimited by `---`.
Recognised keys:

### `topics/*.md`

```yaml
---
id: channel
title: Channel
aliases: [chan, send, recv, buffered]
---
```

* `id` — must equal the filename stem; lookup key.
* `title` — short human title.
* `aliases` — extra lookup keys (case-insensitive substring match).

### `stdlib/*.md`

```yaml
---
module: json
summary: JSON parsing, serialisation, and Json value type.
---
```

* `module` — must equal the filename stem.
* `summary` — single line, used by ranked `xray_stdlib_search`.

The body is the long-form module description. Per-symbol signatures
(`json.parse`, `json.stringify`, ...) are not authored here — they come
from the analyzer's builtin metadata via `xray builtin-dump` and are
merged in by the generator.

### `resources/*.md`

```yaml
---
id: cheatsheet
---
```

* `id` — `cheatsheet`, `concurrency`, or `stdlib_list`. The generator
  emits one C string constant per id.

## Editing rules

* **Never edit `xmcp_knowledge_generated.c` by hand.** It carries an
  `@generated` banner; any edit will be silently overwritten on the next
  generator run.
* All `xray` code fences in topic/stdlib bodies are exercised by a
  parser smoke test. Outdated syntax fails the build.
* Frontmatter aliases are matched case-insensitively after a substring
  check; no need to enumerate every case variant.
* Output is deterministic: entries are sorted by id/module before being
  emitted so reviewer-visible diffs reflect only real content changes.

## Regeneration

Manual:

```sh
python3 scripts/gen_mcp_knowledge.py \
    --docs docs/knowledge \
    --out src/app/mcp/xmcp_knowledge_generated.c
```

Or via CMake (after configuring the build):

```sh
cmake --build build --target regen-mcp-knowledge
```

The build target is **not** chained into the default build to avoid a
chicken-and-egg dependency on a working Python in every environment;
contributors are expected to run it after touching `docs/knowledge/`.
