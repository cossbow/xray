# Tasks 索引

日常执行文档：分析、计划、重构方案。每个文件一个编号，**编号永不复用**。
完成或被取代后挪到 `../archive/`，编号保留以便追溯。

## 状态约定

- 进行中（active）
- 待启动（planned）
- 暂缓（paused）
- 已完成 → 应迁入 `../archive/`

## 当前任务

| # | 文件 | 主题 | 状态 |
|---|------|------|------|
| 001 | [`001-vm-refactor.md`](001-vm-refactor.md) | VM 重构（入口契约、IC 归位、文件拆分） | active |
| 002 | [`002-runtime-refactor.md`](002-runtime-refactor.md) | Runtime/value/gc/object 重构 | active |
| 003 | [`003-class-refactor.md`](003-class-refactor.md) | Class/symbol 重构 | active |
| 004 | [`004-aot-refactor.md`](004-aot-refactor.md) | AOT 重构（删半成品、显式契约、runtime lifecycle） | active |
| 005 | [`005-aot-implementation.md`](005-aot-implementation.md) | AOT 实施文档 v2.0（Phase A/B/C/D） | active |
| 006 | [`006-jit-stabilization.md`](006-jit-stabilization.md) | JIT 当前稳定化（先修 crash、再收准入） | active |
| 007 | [`007-jit-refactoring.md`](007-jit-refactoring.md) | JIT 重构计划 | active |
| 008 | [`008-jit-multi-backend.md`](008-jit-multi-backend.md) | JIT 多后端实施 | active |
| 009 | [`009-jit-x64-parity.md`](009-jit-x64-parity.md) | JIT x64/ARM64 对齐 3 Phase | active |
| 010 | [`010-x64-jit-hardening.md`](010-x64-jit-hardening.md) | x64 backend 硬化批次（含进度日志） | active |
| 011 | [`011-jit-next-phase.md`](011-jit-next-phase.md) | JIT Next Phase 计划 | planned |
| 012 | [`012-frontend-refactor.md`](012-frontend-refactor.md) | Frontend 最终重构方案（Final 决策版） | active |
| 013 | [`013-dap-refactor.md`](013-dap-refactor.md) | DAP 重构（生命周期、停止链、能力契约） | active |
| 014 | [`014-lsp-refactor.md`](014-lsp-refactor.md) | LSP 重构（correctness、workspace、索引） | active |
| 015 | [`015-mcp-improvement.md`](015-mcp-improvement.md) | MCP 模块改进 | active |
| 016 | [`016-pkg-registry.md`](016-pkg-registry.md) | 包管理 + Registry 实施 | active |
| 017 | [`017-coro-audit.md`](017-coro-audit.md) | Coro 审计实施（剩余 9 个 issue） | active |
| 018 | [`018-gc-refactor.md`](018-gc-refactor.md) | GC 模块重构 | active |
| 019 | [`019-api-module-refactor.md`](019-api-module-refactor.md) | src/api 模块重构 | active |
| 020 | [`020-base-module-refactor.md`](020-base-module-refactor.md) | src/base 模块重构 | active |
| 021 | [`021-module-optimization.md`](021-module-optimization.md) | src/module 模块优化 | active |
| 022 | [`022-upgrade-roadmap.md`](022-upgrade-roadmap.md) | Xray 升级优化总路线图 | planned |
| 024 | [`024-runtime-phase0-control-plane.md`](024-runtime-phase0-control-plane.md) | Runtime 控制面与状态归属分析 | active |
| 025 | [`025-runtime-phase1-value.md`](025-runtime-phase1-value.md) | Runtime value/ 层分析 | active |
| 026 | [`026-runtime-phase2-gc-closure.md`](026-runtime-phase2-gc-closure.md) | Runtime gc/ + closure/ 分析 | active |
| 027 | [`027-runtime-phase3-object.md`](027-runtime-phase3-object.md) | Runtime object/ 层分析 | active |
| 028 | [`028-runtime-phase4-class-symbol.md`](028-runtime-phase4-class-symbol.md) | Runtime class/ + symbol/ 层分析 | active |
| 029 | [`029-runtime-phase5-coro.md`](029-runtime-phase5-coro.md) | Runtime src/coro/ 层分析 | active |
| 030 | [`030-runtime-cross-cutting-recap.md`](030-runtime-cross-cutting-recap.md) | Runtime 横切复盘 | active |
| 031 | [`031-aot-architecture.md`](031-aot-architecture.md) | AOT 目标架构（复用 runtime+coro，CPS 协程，边界 DAG） | active |
| 032 | [`032-aot-binary-size.md`](032-aot-binary-size.md) | AOT 二进制体积策略（feature gate + 自动推断 + size report） | active |
| 033 | [`033-aot-implementation.md`](033-aot-implementation.md) | AOT 缺陷清单与实施方案（R1..R18 + Phase S1..S9） | active |
| 034 | [`034-xsem-unified-value.md`](034-xsem-unified-value.md) | XIR: Typed SSA IR + 统一值表示/GC 契约（单一管线） | active |
| 035 | [`035-codegen-to-xir-impact-analysis.md`](035-codegen-to-xir-impact-analysis.md) | xcodegen → XIR 迁移影响分析（逐项评估） | active |
| 036 | [`036-stdlib-refactor-analysis.md`](036-stdlib-refactor-analysis.md) | 标准库逐库分析与重构计划 | active |
| 037 | [`037-stdlib-foundation-analysis.md`](037-stdlib-foundation-analysis.md) | stdlib 横切基础层分析与优化建议 | active |
| 038 | [`038-stdlib-time-analysis.md`](038-stdlib-time-analysis.md) | stdlib/time 分析与优化建议 | active |
| 039 | [`039-stdlib-math-analysis.md`](039-stdlib-math-analysis.md) | stdlib/math 分析与优化建议 | active |
| 040 | [`040-stdlib-gc-analysis.md`](040-stdlib-gc-analysis.md) | stdlib/gc 分析与优化建议 | active |
| 041 | [`041-stdlib-path-analysis.md`](041-stdlib-path-analysis.md) | stdlib/path 分析与优化建议 | active |
| 042 | [`042-stdlib-base64-analysis.md`](042-stdlib-base64-analysis.md) | stdlib/base64 分析与优化建议 | active |
| 043 | [`043-stdlib-encoding-analysis.md`](043-stdlib-encoding-analysis.md) | stdlib/encoding 分析与优化建议 | active |
| 044 | [`044-stdlib-url-analysis.md`](044-stdlib-url-analysis.md) | stdlib/url 分析与优化建议 | active |
| 045 | [`045-stdlib-datetime-analysis.md`](045-stdlib-datetime-analysis.md) | stdlib/datetime 分析与优化建议 | active |
| 046 | [`046-stdlib-log-analysis.md`](046-stdlib-log-analysis.md) | stdlib/log 分析与优化建议 | active |
| 047 | [`047-stdlib-json-analysis.md`](047-stdlib-json-analysis.md) | stdlib/json 分析与优化建议 | active |
| 048 | [`048-stdlib-csv-analysis.md`](048-stdlib-csv-analysis.md) | stdlib/csv 分析与优化建议 | active |
| 049 | [`049-stdlib-yaml-analysis.md`](049-stdlib-yaml-analysis.md) | stdlib/yaml 分析与优化建议 | active |
| 050 | [`050-stdlib-toml-analysis.md`](050-stdlib-toml-analysis.md) | stdlib/toml 分析与优化建议 | active |
| 051 | [`051-stdlib-xml-analysis.md`](051-stdlib-xml-analysis.md) | stdlib/xml 分析与优化建议 | active |
| 052 | [`052-stdlib-compress-analysis.md`](052-stdlib-compress-analysis.md) | stdlib/compress 分析与优化建议 | active |
| 053 | [`053-stdlib-crypto-analysis.md`](053-stdlib-crypto-analysis.md) | stdlib/crypto 分析与优化建议 | active |
| 054 | [`054-stdlib-http-analysis.md`](054-stdlib-http-analysis.md) | stdlib/http 分析与优化建议 | active |
| 055 | [`055-stdlib-ws-analysis.md`](055-stdlib-ws-analysis.md) | stdlib/ws 分析与优化建议 | active |
| 056 | [`056-stdlib-net-analysis.md`](056-stdlib-net-analysis.md) | stdlib/net 分析与优化建议 | active |
| 057 | [`057-stdlib-cluster-analysis.md`](057-stdlib-cluster-analysis.md) | stdlib/cluster 分析与优化建议 | active |
| 058 | [`058-stdlib-io-analysis.md`](058-stdlib-io-analysis.md) | stdlib/io 分析与优化建议 | active |
| 059 | [`059-stdlib-os-analysis.md`](059-stdlib-os-analysis.md) | stdlib/os 分析与优化建议 | active |
| 060 | [`060-stdlib-regex-analysis.md`](060-stdlib-regex-analysis.md) | stdlib/regex 分析与优化建议 | active |
| 061 | [`061-stdlib-test-yield-analysis.md`](061-stdlib-test-yield-analysis.md) | stdlib/test_yield 分析与优化建议 | active |
| 062 | [`062-stdlib-cross-cutting-recap.md`](062-stdlib-cross-cutting-recap.md) | stdlib 横切复盘与实施优先级 | active |
| 063 | [`063-io-runtime-refactor.md`](063-io-runtime-refactor.md) | IO runtime 重构实施方案（统一 yieldable IO、typed handle、async file IO） | active |
| 064 | [`064-json-type-system-refactor.md`](064-json-type-system-refactor.md) | Json 类型系统重构（删 JsonValue、拆 Object、合并 json stdlib） | planned |
| 065 | [`065-prelude-refactor.md`](065-prelude-refactor.md) | Prelude 重构方案 | planned |
| 066 | [`066-xi-aot-migration-plan.md`](066-xi-aot-migration-plan.md) | Xi IR AOT 迁移方案 | active |
| 067 | [`067-architecture-unification-plan.md`](067-architecture-unification-plan.md) | Xray 架构统一重构方案（XIR、Semantic Model、Runtime Contract） | planned |
| 068 | [`068-compiler-pipeline-optimization.md`](068-compiler-pipeline-optimization.md) | 编译器管线优化（frontend/ir/jit 正确性、拆分、加固、对等） | planned |
| 069 | [`069-compiler-architecture-refactor.md`](069-compiler-architecture-refactor.md) | 编译器架构重构方案（frontend binding、Xi stage、backend contract） | active |
| 070 | [`070-regression-bug-triage.md`](070-regression-bug-triage.md) | 回归测试失败分类与修复优先级 | active |
| 071 | [`071-comptime-implementation-plan.md`](071-comptime-implementation-plan.md) | comptime 特性实施方案（编译期求值、特化、分支消除） | planned |
| 072 | [`072-jit-method-call-tag-fix.md`](072-jit-method-call-tag-fix.md) | JIT method call / tag 契约修复（1120 regex 回归） | active |
| 076 | [`076-codegen-master-spec.md`](076-codegen-master-spec.md) | JIT codegen 最终 master 规约（三层真相源 + 全 hardening 纪律，取代 073/074/075） | planned |
| 077 | [`077-xi-ir-optimization-roadmap.md`](077-xi-ir-optimization-roadmap.md) | Xi IR 优化 master roadmap（Xi-0 债务清理 → Xi-4 向量化/LTO/partial eval） | planned |
| 078 | [`078-repl-globals-unification.md`](078-repl-globals-unification.md) | REPL × top-level globals 统一化（删除 shared offset，dict 替代整型槽） | planned |
| 079 | [`079-tuple-first-class.md`](079-tuple-first-class.md) | Tuple first-class 设计方案（圆括号路线，多返回值统一） | planned |
| 080 | [`080-try-optional-expression.md`](080-try-optional-expression.md) | `try?` 表达式（异常折叠为 null，与 ?. ?? ! 形成完整容错链） | planned |
| 081 | [`081-error-handling-redesign.md`](081-error-handling-redesign.md) | 错误处理重设计 | planned |
| 082 | [`082-arrow-syntax-unification.md`](082-arrow-syntax-unification.md) | 箭头语法统一 | planned |
| 083 | [`083-mcp-final-redesign.md`](083-mcp-final-redesign.md) | MCP Server 最终重构方案（不保留旧接口） | planned |

 **下一个编号**：084

## 新建任务

```bash
# 取下一个编号，写新文档
NEXT=084
$EDITOR docs/tasks/${NEXT}-<short-name>.md
# 编辑完后在本文件追加一行
```

## 完成归档

```bash
# 任务完成后
mv docs/tasks/NNN-name.md docs/archive/
# 并在本文件中删除该行（或在该行末尾标记 ✅ 后批量清理）
```

## 已归档（参考）

完整归档列表见 [`../archive/`](../archive/)。最近归档：

- `075-codegen-final-spec.md` — codegen 路线对比第二方案（2026-05-12，被 076 取代）
- `074-codegen-final-plan.md` — codegen 路线对比第一方案（2026-05-12，被 076 取代）
- `073-codegen-hardening.md` — codegen 工程化加固原方案（2026-05-12，被 076 取代）
- `023-comment-cleanup.md` — 源码注释阶段说辞清理（铁律落地，2026-04-25）
- `cli_refactor_plan.md` — CLI 重构 3 个 Phase 全部完成（2026-04-24）
- `coro_refactor_plan.md` — Coro refactor 7 个 Phase 核心工作完成（2026-04-24）
- `module_restructure_plan.md` — 模块结构重构（2026-04-17）
- `superseded/` — 被取代的多版本前置文档
- `analysis/` — stdlib 分析与 TODO（已落地）
- `audit_raw_2026_04_17/` — 一次性审计原始扫描结果
