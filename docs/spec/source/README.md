# 结构化语言规范源

本目录是语言参考手册和 MCP knowledge 投影文件的唯一手写源。

生成输出：

```text
LANGUAGE_SPEC_CN.md
docs/knowledge/**
LANGUAGE_SPEC.md
```

不要直接编辑这些生成输出。需要重新生成时运行：

```sh
python3 scripts/gen_language_docs.py --root .
```

## 目录结构

```text
docs/spec/source/
  sections/*.md            # 中英文语言参考手册章节源
  knowledge/README.md      # docs/knowledge/README.md 的源
  knowledge/topics/*.md    # MCP topic 投影源
  knowledge/stdlib/*.md    # stdlib 说明源；API 表后续自动注入
  knowledge/resources/*.md # MCP resource 投影源
```

## 章节格式

`sections/` 下每个文件都包含一个小 frontmatter，后面跟两个显式语言块：

```md
---
id: spec.type_system
order: 003
---

<!-- xr-spec:cn -->
Chinese reference text.
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
English reference text.
<!-- /xr-spec:en -->
```

规则：

- `id` 必须全局唯一。
- `order` 决定输出顺序，也必须全局唯一。
- `cn` 和 `en` 块都必须存在。
- Xray 示例必须继续使用 `xray` 代码块；语法高亮属于渲染层，不属于源格式。
- 禁止用摘要替代详细说明；生成后的 SPEC 必须保持或提升当前文档质量。

## 质量门禁

`tests/mcp/test_knowledge_generation.py` 会检查生成后的 SPEC 是否保持最新，并确保不会低于当前质量基线：

- 非空内容行数
- 标题数量
- 顶层章节数量
- Xray 示例代码块数量
- EBNF 代码块数量
- Markdown 表格数量
