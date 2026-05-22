# MCP Knowledge 生成输出

本目录是 MCP 内置知识负载的**生成输出**，不是语言语义本身的独立真相源。它服务于
`xray mcp-server`，生成 `xray_syntax_lookup`、`xray_stdlib_search`、
`xray_definition` 和 `xray://...` resource URI 返回给 AI 客户端的内容。

请不要直接编辑 `docs/knowledge`。所有人工维护内容位于 `docs/spec/source`：

```
docs/spec/source/
  sections/*.md           # 中英文语言参考手册的结构化源
  knowledge/topics/*.md   # MCP topic 投影源
  knowledge/stdlib/*.md   # stdlib 模块说明源；API 表由 analyzer metadata 注入
  knowledge/resources/*.md# MCP resource 投影源
```

`scripts/gen_language_docs.py` 从上述源生成 `LANGUAGE_SPEC_CN.md`、
`LANGUAGE_SPEC.md` 和本目录内容；`scripts/gen_mcp_knowledge.py` 再把本目录和
`xray builtin-dump` 输出合成为 `xmcp_knowledge_generated.c`。

## 真相层级

| 层级 | 责任 | 维护规则 |
|--|--|--|
| 实现代码 | 语言语义的最终事实 | parser / analyzer / runtime 与文档冲突时，必须修正代码或文档 |
| `docs/spec/source/sections/*.md` | 人类可读语言参考手册的唯一手写源 | 同时维护中文、英文正文；必须保持当前文档质量不降级 |
| `LANGUAGE_SPEC_CN.md` / `LANGUAGE_SPEC.md` | 生成出来的人类可读语言参考手册 | 不手写；由结构化 spec source 生成 |
| analyzer builtin metadata | 标准库 API 签名事实 | 由 analyzer builtin 表和 `xray builtin-dump` 导出，不在 Markdown 手写复制 |
| `docs/spec/source/knowledge` | MCP 面向 AI 的索引化摘要源 | 只写 aliases、短说明、示例、gotchas；必须链接到生成后的语言参考手册 anchor |
| `docs/knowledge` | MCP knowledge 生成输出 | 不手写；由结构化 spec source 生成 |
| `xmcp_knowledge_generated.c` | 生成产物 | 不手写；由 generator 覆盖 |

因此，`docs/spec/source/knowledge/topics/*.md` 不能定义和语言参考手册冲突的新语义。它只负责把
参考手册中的知识压缩成适合 MCP 检索和 LLM 使用的知识卡片。

## 目录结构

```
docs/spec/source/knowledge/
  topics/<id>.md       # 每个语言主题一份 MCP 知识卡片
  stdlib/<module>.md   # 每个标准库模块一份人工说明
  resources/<id>.md    # 长文本资源，如 cheatsheet / concurrency / stdlib_list
```

文件名是 canonical id。文件正文会生成到 `docs/knowledge`，并作为 Markdown 原样进入
MCP tool/resource 输出。

## Frontmatter

每个文件都以 `---` 包裹的 YAML-like frontmatter 开头。

### `topics/*.md`

```yaml
---
id: channel
title: Channel
spec: #105-channel
aliases: [chan, send, recv, buffered]
---
```

* `id` — 必须等于文件名 stem，是主 lookup key。
* `title` — 短标题。
* `spec` — 必须指向 `LANGUAGE_SPEC_CN.md` 中真实存在的 heading anchor。
* `aliases` — 额外 lookup key，大小写不敏感，支持 substring 匹配。

### `stdlib/*.md`

```yaml
---
module: json
summary: JSON parsing, serialisation, and Json value type.
---
```

* `module` — 必须等于文件名 stem。
* `summary` — 单行摘要，用于 ranked `xray_stdlib_search`。

正文只写人工说明、使用场景、示例和注意事项。每个 symbol 的 signature / summary 来自
analyzer builtin metadata，由 `xray builtin-dump` 导出并在生成时自动注入 `## API`
表格，避免 Markdown 与实际 API 漂移。

### `resources/*.md`

```yaml
---
id: cheatsheet
---
```

* `id` — `cheatsheet`、`concurrency` 或 `stdlib_list`。

## 修改流程

### 语言语法或语义变更

1. 修改实现代码与回归测试。
2. 更新 `docs/spec/source/sections/*.md` 的中文和英文正文。
3. 更新对应 `docs/spec/source/knowledge/topics/*.md` 的摘要、示例和 `spec` anchor。
4. 重新生成语言参考手册和 MCP knowledge。

### 标准库 API 变更

1. 修改标准库实现和 analyzer builtin metadata。
2. 通过 `xray builtin-dump` 导出 API metadata。
3. 只在 `docs/spec/source/knowledge/stdlib/*.md` 更新人工说明、示例和 gotchas。
4. 重新生成 MCP knowledge。

## 生成与校验

推荐使用 CMake target 一次性生成全部文档投影和 MCP C 表：

```sh
cmake --build build --target regen-mcp-knowledge
```

该 target 会先生成语言文档和 `docs/knowledge`：

```sh
python3 scripts/gen_language_docs.py --root .
```

然后运行 `xray builtin-dump`，再调用 MCP generator：

```sh
python3 scripts/gen_mcp_knowledge.py \
    --docs docs/knowledge \
    --spec LANGUAGE_SPEC_CN.md \
    --builtins build/xmcp_builtin_dump.json \
    --out src/app/mcp/xmcp_knowledge_generated.c
```

`tests/mcp/test_knowledge_generation.py` 会检查：

* `LANGUAGE_SPEC_CN.md` / `LANGUAGE_SPEC.md` 和 `docs/knowledge` 必须由
  `docs/spec/source` 生成且保持最新。
* 生成后的 SPEC 必须满足质量门禁，防止章节、示例、表格和 EBNF 内容意外减少。
* `topics/*.md` 的 `spec` anchor 必须存在于语言参考手册。
* 所有 `xray` code fence 必须能被 parser 接受。
* generated C 必须和当前 docs / builtin dump 保持一致。
* generated stdlib symbols 必须来自 analyzer builtin dump。
* prompt smoke examples 必须保持可解析。

## 禁止事项

* 禁止手写修改 `LANGUAGE_SPEC_CN.md` / `LANGUAGE_SPEC.md` / `docs/knowledge` /
  `xmcp_knowledge_generated.c`。
* 禁止在 `docs/spec/source/knowledge/stdlib/*.md` 手写 API signature 副本。
* 禁止让 MCP knowledge 成为和语言参考手册并列竞争的语义说明。
* 禁止修改语言语法后只更新 MCP knowledge 而不更新结构化语言参考源。
