# Xray Docs

```
docs/
├── rules/         # 编码规则与语言规范（长期、稳定）
├── design/        # 长期系统/语言设计（不随单次重构变动）
├── engineering/   # 工程知识沉淀（best practice、bug pattern、不变量、ADR）
├── tasks/         # 日常执行文档（编号 001+，进行中）
└── archive/       # 已完成、被取代或过时的文档
```

## 目录用途与放置规则

| 目录 | 放什么 | 不放什么 | 何时改动 |
|------|--------|----------|----------|
| `rules/` | 项目纪律：架构、C 编码、GC、语言规范、开发流程 | 一次性的临时规则、单个 bug 修复笔记 | 项目长期规则有调整时 |
| `design/` | 系统级/语言级长期设计文档（如 `aot-design.md`、`coroutine_kotlin_alignment.md`、`mcp-server.md`、800/900 系列） | 实施步骤、阶段计划、批次硬化（这些属于 `tasks/`） | 系统设计本身演进时 |
| `engineering/` | 跨任务沉淀的知识：bug 模式、不变量、调试手册、架构决策记录、长期边界文档（如 `jit_vm_boundary.md`） | 任何"做完就归档"的执行计划与重构 plan | 修 bug 后回填 / 每次重要架构决策 |
| `tasks/` | 当前进行中的实施 plan、refactor plan、audit plan，统一编号 `NNN-name.md` | 长期最佳实践（属于 `engineering/`）；已完成（属于 `archive/`） | 每次新分析/计划时新增；执行完毕迁 `archive/` |
| `archive/` | 已完成、被取代、或不再有用的文档，**保留编号便于追溯** | 还在执行的文档 | 任务做完时把对应文件挪进来 |

## 日常工作流

```
新分析  ─┐
        ├─►  写文档落进 tasks/NNN-xxx.md
新计划  ─┘                │
                          ▼
                       开始执行
                          │
                  ┌───────┴───────┐
                  ▼               ▼
              执行完成         被新版本取代
                  │               │
                  ▼               ▼
        archive/NNN-xxx.md   archive/superseded/...
```

**规则**：

1. 任何"分析 / 计划 / 审计"完成后，先落进 `tasks/`，分配下一个编号。
2. 执行结束（或永久搁置）后，把文件挪进 `archive/`，**保留原编号**，便于通过编号追溯任务历史。
3. 多版本并存的同主题文档（user / cascade / final），**保留 final 进 `tasks/`**，前置版本进 `archive/superseded/`。
4. 工程沉淀（每修一个 bug、做完一个重要设计决策）回填到 `engineering/` 对应文档（`bug_patterns.md` / `invariants.md` / `architecture_decisions.md`），不走 `tasks/` 流程。
5. 编号一旦分配**永不复用**。`tasks/` 中的最大编号 + 1 就是下一个新任务的编号。

## 当前最新编号

参见 `tasks/README.md`。新建任务前先看那里取下一个号。

## 索引快查

- 任务索引（含状态）：[`tasks/README.md`](tasks/README.md)
- 工程知识库：[`engineering/README.md`](engineering/README.md)
- 编码规则：[`rules/`](rules/)
- 长期设计：[`design/`](design/)
