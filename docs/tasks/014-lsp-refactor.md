# LSP 模块重构实施文档（`src/app/lsp`）

> **⚡ 开发原则：不考虑向后兼容性！**
>
> Xray 没有外部用户，代码量小，可以大胆重写——每个阶段都选择最佳设计。
>
> - 不保留"已宣称但未真正实现"的 capability / config / 行为契约
> - 不保留旧接口、不做兼容层、不做"短期凑合 + 长期再改"的两步走
> - 发现设计错误就直接删除重写，不维持错误设计
> - 每个阶段独立可合并、独立可回滚
>
> **本文定位**：把 `src/app/lsp` 的审计结论收敛为可执行实施方案，方便在后续新对话中按阶段落地。

## 1. 背景

`src/app/lsp` 已经具备一个“能工作的 LSP server”骨架：

- 有完整的 JSON-RPC / LSP 事件循环
- 有文档表、请求分发、诊断 debounce、工作区扫描、索引池、补全/跳转/引用/重命名/格式化等能力
- 已经接入 `workspace_analyzer`，并不是纯 lexer 玩具实现

但当前实现存在明显的“**功能表面完整，内部成熟度不一致**”问题：

- 协议层、配置层、实现层之间有多处漂移
- multi-root workspace 只做了表层支持
- 后台索引做了并行化，但真实重成本仍堆在主线程
- 架构边界被破坏（`app/lsp -> app/cli`）
- 核心文件体量已逼近项目硬性上限，继续叠加功能风险很高

### 1.1 当前基线

- `xlsp_server.c` 约 `2835` 行，已接近 `.c <= 3000` 的项目硬线（截至 2025-04-23）
- `xlsp_analysis.c` 约 `2933` 行，同样接近上限
- `tests/unit/lsp/` 当前只有：
  - `test_lsp_json.c`
  - `test_lsp_document.c`
- `tests/unit/CMakeLists.txt` 中，`test_lsp_document` 仍直接编译 `src/app/cli/xfmt.c`，说明 formatter 仍然挂在错误层级

### 1.2 这次重构的核心判断

这不是一个“补几个 bug”的任务，而是一个 **correctness + contract + architecture + responsiveness** 的联合收敛任务。

如果继续小修小补，会长期维持以下糟糕状态：

- 客户端看到的 capability 与真实行为不一致
- 配置看似支持，实际不生效
- 大 workspace 下仍会卡在主线程 reparse/reanalyze
- workspace folder 动态增删后状态不一致
- 新功能继续堆进 `xlsp_server.c` / `xlsp_analysis.c`，最终变成不可维护的大文件

---

## 2. 当前问题总览

### 2.1 P0：correctness / contract 问题

### LSP-01 诊断 debounce 队列可能静默丢刷新

`xlsp_server.c:schedule_diagnostics()` 使用固定大小的 `pending_diag` 数组。

现状：

- 文档总会更新 `diagnostic_deadline`
- 但只有在 `pending_diag_count < MAX_PENDING_DIAG` 时才进入队列
- `flush_pending_diagnostics()` 只扫描队列，不扫描全量文档

后果：

- 一旦超过固定队列容量，后续文档可能**永远不再发布诊断**
- 这是 correctness bug，不是单纯性能问题

### LSP-02 `diagnostics_enabled` 配置未真正控制调度/发布

当前配置支持：

- `diagnostics.enabled`
- `diagnostics.debounceMs`

但实际只有 `debounceMs` 明确参与了调度；`diagnostics_enabled` 目前只被更新和打印日志，没有成为真实控制开关。

后果：

- 用户关闭 diagnostics 后，server 仍可能继续调度/发布
- 配置契约不成立

### LSP-03 multi-root 初始化和动态变更不完整

当前多根工作区存在以下缺口：

- `initialize` 虽然接收 `workspaceFolders`，但 `xray.toml` 只从 `root_path` 或第一个 workspace folder 加载
- `handle_initialized()` 只对 `root_path/root_uri` 启动后台索引
- `add_workspace_folder()` 在 `indexing_in_progress` 为真时，新增 folder 的索引请求可能被直接跳过
- `remove_workspace_folder()` 只删除 folder 元数据，没有清理 analyzer / 缓存 / stale symbols

后果：

- 新增 workspace folder 可能长期不被索引
- 删除 workspace folder 后，符号仍可能残留并参与 definition / references / workspace symbol

### LSP-04 新后台索引路径未真正使用 ignore 配置

`xlsp_config_load_from_toml()` 确实支持 `[lsp].ignore = [...]`，且 `find_xr_files_with_config()` 也支持传入配置。

但 `xlsp_workspace_start_background_index()` 直接调用 `find_xr_files()`（内部传空配置），绕过了 `background_index_task_scan()` 中已正确接线的带配置路径。

后果：

- `xray.toml` 中的 ignore 配置在主后台索引入口不生效
- 大仓库会扫入不该处理的目录/文件

### LSP-05 若干行为契约和真实实现不一致

已确认的不一致点包括：

- `rangeFormatting` 实际直接格式化整个文档
- `xlsp_analysis.h` / 内部注释容易让人误以为 `rename` 已是 cross-file rename，但实现实际只返回当前文档的 `WorkspaceEdit`
- `onTypeFormatting` 是自定义缩进逻辑，不是完整 formatter 集成

直接下线未实现的能力宣称，修正文档和注释。不做"先保留、后面再实现"的妥协。

### 2.2 P1：配置闭环与语义能力问题

### LSP-06 多个配置项“可解析但未生效”

当前已确认存在以下 drift：

- `completion_max_items`：可解析，但补全结果未裁剪
- `completion_auto_import`：可解析，但未见真实行为接线
- `analysis_type_checking`：可解析，但未控制 analyzer/type-checking 路径
- `format_tab_size` / `format_insert_spaces`：可解析，但 `xlsp_analyze_format()` 直接使用 `xfmt_default_config`
- `log_level`：字段存在，但日志未做级别过滤

### LSP-07 跨文件语义能力未闭环

当前状态：

- 当前文件内的 definition / references / highlight 已经部分使用 analyzer + scope
- 但跨文件 references 仍明显依赖 lexer/open-doc fallback
- `xlsp_call_hierarchy.c` 仍有“未搜索 unopened files” TODO

后果：

- 功能“有返回”不代表语义正确
- 打开文档和未打开文档之间行为不一致

### LSP-08 hover / navigation 仍混用 name-based fallback

例如 hover 仍有 `xa_analyzer_lookup(analyzer, word)` 这种按名字查找的路径。

风险：

- shadowing 场景容易返回错误信息
- 不同 feature 的语义精度不一致

### 2.3 P1：响应性和索引设计问题

### LSP-09 并行索引收益被主线程二次全量分析抵消

当前链路：

1. worker 线程读取文件、parse、提取浅层符号
2. 主线程 merge 结果后，再次打开文件、再次 parse、再次 `xa_analyzer_update_incremental()`

后果：

- 同一文件 I/O 与 parse 重复两次
- 真正昂贵的 analyzer 更新仍串行堆在主线程
- 大 workspace 索引完成期间，主循环仍可能被 merge 阶段拖慢

### LSP-10 后台索引和语义状态没有收敛为单一权威模型

当前至少存在三层“似是而非”的状态：

- `workspace_analyzer`
- worker 线程浅层符号结果
- `xlsp_workspace.[ch]` 中老的 `XrLspWorkspaceIndex` 抽象

其中老的 `XrLspWorkspaceIndex` 目前几乎不在主路径使用，处于半遗留状态。

后果：

- 维护者难以判断哪一层才是权威数据源
- 容易出现“改了 A 路径，B 路径没改”的漂移

### 2.4 P1/P2：架构与可维护性问题

### LSP-11 `app/lsp` 直接依赖 `app/cli` formatter

`xlsp_analysis.c` 通过 `../cli/xfmt.h` 复用 formatter。

这违反当前架构层次：

- `app/lsp` 与 `app/cli` 都属于高层应用模块
- `app/lsp` 不应横向 include `app/cli`
- formatter 应下沉到 `frontend/*` 或其他更低层公共模块

### LSP-12 核心文件已过大，继续堆功能风险极高

- `xlsp_server.c`：协议入口、生命周期、配置、文档事件、workspace 事件、主循环全部耦合在一起
- `xlsp_analysis.c`：诊断、hover、definition、references、rename、format、signature help 等大量能力耦合

再继续长下去，会直接碰到：

- 文件规模超线
- 函数过长
- 局部修改导致整体回归
- review 成本飙升

---

## 3. 重构目标

### 3.1 最终目标

1. **协议契约可信**
   - server 只宣称自己真正支持的能力
   - config 只暴露真正可生效的字段
   - comment / header / behavior 三者一致

2. **workspace 状态一致**
   - multi-root 初始化、增删 folder、文件删除/重建都能正确维护 analyzer 和缓存
   - 不保留 stale symbol / stale document / stale index

3. **响应性稳定**
   - 背景索引不能把主线程拖成“间歇性阻塞”
   - 大 workspace 下，open/change/completion 等前台请求优先级更高

4. **架构边界正确**
   - `app/lsp` 不再依赖 `app/cli`
   - formatter 下沉到公共层
   - LSP 内部也只保留单一权威语义状态模型

5. **代码结构可维护**
   - `xlsp_server.c` / `xlsp_analysis.c` 拆分
   - 遗留/废弃抽象直接删除

6. **测试覆盖补齐**
   - 不再只依赖 `test_lsp_json` / `test_lsp_document` 两个基础测试
   - 至少覆盖配置、workspace、索引、格式化、导航、协议契约等关键路径

### 3.2 非目标

本次实施文档**不要求**一步完成以下能力：

- 完整 workspace 级别的所有语义特性一次性到位
- 复杂的 AST 增量解析器重写
- 真正意义上的增量 range formatter（若 formatter 底层未准备好）
- LSP 性能极致优化（如更激进的并行 AST/IR 共享）

策略是：**先把 correctness / contract / responsiveness 收敛，再逐步完善高级能力。不做渐进兼容，直接选最佳设计。**

---

## 4. 目标架构

### 4.1 单一权威语义状态

未来应明确：

- `workspace_analyzer` 是 **唯一权威的语义状态源**
- worker 线程返回的数据如果保留，只能作为：
  - 后台扫描元数据
  - 暂时性的 lexical symbol cache
  - 进度和变更探测辅助信息
- 不允许再维持一个“看起来是 workspace index、但实际上主逻辑不依赖”的半废弃层

### 4.2 LSP 内部拆分结构

目标不是一次性大重写，而是逐阶段收敛到下面的布局：

```text
src/app/lsp/
├── xlsp_server.c                     # 薄入口 / 主循环 / server create/free
├── xlsp_server.h
├── xlsp_handlers_lifecycle.c         # initialize / initialized / shutdown / exit
├── xlsp_handlers_text_document.c     # didOpen / didChange / didClose / save / formatting / completion
├── xlsp_handlers_workspace.c         # workspaceFolders / watchedFiles / configuration
├── xlsp_diagnostics.c                # debounce / publish / clear / flush
├── xlsp_config.c                     # xray.toml + didChangeConfiguration + precedence
├── xlsp_workspace.c                  # workspace folders / scan / purge / file state
├── xlsp_index_pool.c                 # worker pool（若保留）
├── xlsp_navigation.c                 # hover / definition / references / highlight / call hierarchy
├── xlsp_rename.c                     # prepareRename / rename
├── xlsp_format.c                     # full/range/onType formatting bridge
├── xlsp_completion.c                 # completion
├── xlsp_semantic_tokens.c            # semantic tokens
└── xlsp_inlay_hints.c                # inlay hints
```

**关键边界：**

- `xlsp_server.c` 只管生命周期与主循环，不再承载具体 feature 实现
- diagnostics/config/workspace 独立成模块，避免交叉改动
- navigation / rename / format 从 `xlsp_analysis.c` 中拆出，形成可测单元
- formatter 不再位于 `src/app/cli`

### 4.3 formatter 的正确层级

建议最终落位：

```text
src/frontend/format/
├── xfmt.c
└── xfmt.h
```

依赖关系应调整为：

- `app/cli -> frontend/format`
- `app/lsp -> frontend/format`

而不是：

- `app/lsp -> app/cli`

---

## 5. 分阶段实施计划

### 阶段总览

| Phase | 目标 | 优先级 | 产出 |
|------|------|--------|------|
| 0 | 先修 correctness / contract 漏洞 | P0 | 不再静默丢诊断；multi-root 基本正确；ignore 生效；错误宣称下线 |
| 1 | 收敛配置模型与 capability 契约 | P0/P1 | 配置真正生效；不支持项不再暴露 |
| 2 | 统一 workspace 状态与索引调度 | P1 | 多根工作区、删除/新增文件、后台索引状态一致 |
| 3 | 降低主线程索引阻塞 | P1 | 背景索引 merge 预算化，前台请求优先 |
| 4 | 收敛语义能力正确性 | P1 | hover/references/callHierarchy/rename 语义更一致 |
| 5 | 清理架构边界与大文件 | P1/P2 | formatter 下沉，server/analysis 拆分，遗留层收敛 |
| 6 | 测试与回归体系补齐 | P1/P2 | unit + integration + architecture checks 覆盖核心链路 |

---

### 5.1 Phase 0 — Correctness / Contract 热修阶段

> **Phase 0 约束**：只做行为修复，不引入新的结构体（如 `XlspWorkspaceFolderState`），不做文件拆分。结构化改造留给 Phase 2/5。

### 目标

1. 不再静默漏刷 diagnostics
2. `diagnostics_enabled` 真正控制行为
3. multi-root 初始化和动态增删不再明显失效
4. `[lsp].ignore` 在真实后台索引路径上生效
5. 明确下线当前未真正实现的对外能力宣称

### 主要改动

#### 0.1 修复 pending diagnostics 队列

涉及文件：

- `src/app/lsp/xlsp_server.c`
- `src/app/lsp/xlsp_server.h`

方案：

1. 用**可增长容器**替换固定小数组（直接做最佳方案，不做兜底扫描的折中）
2. 文档对象增加显式 `diag_pending` 标记，避免重复入队
3. 若 diagnostics 被禁用，直接跳过调度
4. 若配置从 enabled → disabled，立即对所有已打开文档发布空 diagnostics 以清屏

#### 0.2 让 `diagnostics_enabled` 成为真实开关

涉及文件：

- `src/app/lsp/xlsp_server.c`

要求：

- `schedule_diagnostics()` 进入前检查配置
- `flush_pending_diagnostics()` 进入前检查配置
- `publish_diagnostics()` 前检查配置
- 关闭 diagnostics 时清空 pending 状态

#### 0.3 修复 multi-root 的初始索引和动态新增

涉及文件：

- `src/app/lsp/xlsp_server.c`
- `src/app/lsp/xlsp_workspace.c`

要求：

- `handle_initialized()` 要遍历所有 workspace folders 启动初始扫描，而不是只看 `root_path/root_uri`
- `add_workspace_folder()` 不能因为“已有索引任务在跑”就直接跳过
- 新增 folder 至少要进入一个待索引队列，等待当前批次结束后继续处理

#### 0.4 修复 workspace folder 删除后的 stale state

涉及文件：

- `src/app/lsp/xlsp_server.c`
- `src/app/lsp/xlsp_workspace.c`
- analyzer 相关适配点

要求：

- 删除 folder 后，清理该路径前缀下的语义状态/缓存/索引结果
- 若当前 analyzer 只支持按文件删除，则按文件前缀遍历清理
- `workspace/symbol` / definition / references 不应再看到已移除 folder 的残留符号

#### 0.5 让 ignore 配置真正接入主扫描路径

涉及文件：

- `src/app/lsp/xlsp_workspace.c`

要求：

- `xlsp_workspace_start_background_index()` 使用 `find_xr_files_with_config()`，而不是无配置路径
- 后台扫描、动态 folder 添加、手动重扫等路径统一使用同一套 ignore 规则

#### 0.6 收敛错误 capability 宣称

涉及文件：

- `src/app/lsp/xlsp_server.c`
- 必要时更新头文件注释和文档

要求：

- `rangeFormatting` 当前是全文替换，直接关闭 `documentRangeFormattingProvider`
- 注释/头文件中描述成 cross-file rename 的，改为“当前仅单文件 rename”
- 对外只保留真实能力，不保留乐观承诺

### 验收标准

- 打开/修改大量文档时，不会再出现“部分文档诊断永不刷新”
- 关闭 diagnostics 后，客户端旧诊断被清空，且后续不再发布新诊断
- 多根工作区初始化时，每个 folder 都进入索引链路
- 新增 folder 即使在索引进行中也会最终被扫描
- 删除 folder 后，相关 definition/workspace symbol 不再命中陈旧结果
- `xray.toml [lsp].ignore` 对后台索引真实生效

### 建议新增测试

- `tests/unit/lsp/test_lsp_diagnostics.c`
  - 诊断队列溢出/去重/禁用清屏
- `tests/unit/lsp/test_lsp_workspace.c`
  - multi-root 初始化、动态 add/remove
- `tests/unit/lsp/test_lsp_config.c`
  - ignore 配置加载与使用

---

### 5.2 Phase 1 — 配置模型与契约闭环

### 目标

1. 配置字段只要存在，就必须真正生效
2. 不准备支持的配置项，不再继续暴露/解析/记录日志误导用户
3. 明确配置来源和优先级

### 配置优先级建议

除日志路径类特殊项外，统一按以下优先级：

```text
defaults < xray.toml < workspace/didChangeConfiguration
```

日志路径保持：

```text
defaults < xray.toml < workspace/didChangeConfiguration < XRAY_LSP_LOG env
```

### 主要改动

#### 1.1 收敛配置 schema

涉及文件：

- `src/app/lsp/xlsp_server.h`
- `src/app/lsp/xlsp_server.c`
- `src/app/lsp/xlsp_config.c`

对每个字段做二选一：

- **要么真正接入逻辑**
- **要么从 schema 中删除/停止对外暴露**

#### 1.2 接通 `completion_max_items`

涉及文件：

- `src/app/lsp/xlsp_completion.c`
- `src/app/lsp/xlsp_server.c`

要求：

- completion 返回值在最终出口统一裁剪
- 裁剪逻辑应在去重/排序之后，而不是生成过程早停导致结果不稳定

#### 1.3 删除 `completion_auto_import`

auto-import 尚未实现，直接移除配置入口。未来真正实现时再添加。

#### 1.4 接通 `analysis_type_checking`

涉及文件：

- `src/app/lsp/xlsp_analysis.c`

要求：

- 若底层 analyzer 不支持安全地部分关闭 type-checking pass，直接删除该配置字段
- 不保留行为不明确的假开关

#### 1.5 接通 formatter 配置

涉及文件：

- `src/app/lsp/xlsp_analysis.c`（或未来 `xlsp_format.c`）

要求：

- `format_tab_size`
- `format_insert_spaces`

必须映射到真实 `XrFmtConfig`，而不是继续无条件使用 `xfmt_default_config`

#### 1.6 接通 `log_level`

涉及文件：

- `src/app/lsp/xlsp_server.c`

要求：

- `lsp_log()` 接入级别过滤
- 若 log 系统不支持分级，直接删除 `log_level` 字段

#### 1.7 扩展 `xray.toml` 支持范围

当前 `xlsp_config_load_from_toml()` 只覆盖 ignore 和日志配置。

直接补齐 diagnostics/completion/format/inlayHints/analysis 等与 runtime config 同步的子集。

`xray.toml` 和 `didChangeConfiguration` 必须覆盖同一个配置面，不允许“同一个 config struct，两套不同支持面”。

### 验收标准

- 配置字段不存在“可更新但不生效”的项
- completion 结果数量受 `completion_max_items` 控制
- formatter 配置真实影响输出
- diagnostics/typeChecking/logging 行为与配置一致
- `xray.toml` 与 `didChangeConfiguration` 的支持面完全一致

### 建议新增测试

- `tests/unit/lsp/test_lsp_completion.c`
  - `maxItems` 裁剪
- `tests/unit/lsp/test_lsp_format.c`
  - `tabSize/insertSpaces` 生效
- `tests/unit/lsp/test_lsp_config.c`
  - `xray.toml` + runtime config precedence

---

### 5.3 Phase 2 — Workspace 状态归一化

### 目标

1. 工作区 folder 状态、文件状态、索引状态有统一模型
2. 删除/新增文件与 folder 后，不再出现 stale state
3. 删除“隐式 root_path 模型”，initialize 阶段直接折叠为 workspace folder，不保留独立字段

### 设计建议

引入显式的 folder 状态结构，例如：

```c
typedef struct XlspWorkspaceFolderState {
    char *uri;
    char *path;
    char *name;
    bool config_loaded;
    bool index_requested;
    bool index_completed;
    uint64_t generation;
} XlspWorkspaceFolderState;
```

说明：

- `root_path/root_uri` 在 initialize 阶段直接折叠为标准 workspace folder 记录，不保留独立字段
- server 内部统一以 `workspace_folders[]` 作为工作区唯一真相源

### 主要改动

#### 2.1 统一 folder 生命周期

涉及文件：

- `src/app/lsp/xlsp_server.c`
- `src/app/lsp/xlsp_workspace.c`

要求：

- initialize 阶段把 `root_path/root_uri` 折叠成标准 folder 记录
- 所有 config load / initial scan / watched files 都基于 folder 列表驱动

#### 2.2 增加文件级 purge helper

涉及文件：

- `src/app/lsp/xlsp_workspace.c`
- analyzer 接口调用点

要求：

- 提供 `purge_file(uri)` / `purge_prefix(prefix_uri)` 一类 helper
- 文件删除、folder 删除、重新导入索引都统一走这条路径

#### 2.3 统一 deleted/created/changed 事件处理

涉及文件：

- `src/app/lsp/xlsp_server.c`

要求：

- 删除：清 analyzer + cache + workspace state
- 创建：进入后台索引/待分析队列
- 修改：重新分析并传播 dirty dependency

### 验收标准

- root 模式和 workspaceFolders 模式得到同等处理
- 文件删除后 definition/references/workspace symbol 不再命中已删文件
- 文件新建后能在后续索引中被发现
- folder 删除后 analyzer 无陈旧符号

---

### 5.4 Phase 3 — 后台索引与主线程响应性收敛

### 目标

1. 背景索引不能在 merge 阶段长时间阻塞主循环
2. 前台请求（didChange/completion/hover）优先级高于后台批量 reanalyze
3. 背景索引路径只保留一套清晰模型

### 短期设计（推荐先做）

**保持当前 index pool，但把 main-thread reanalyze 改成预算化 drain。**

即：

- worker 线程继续负责扫描/浅层符号/变更探测
- `xlsp_workspace_merge_index_results()` 不直接同步重分析所有文件
- 改为把文件加入 `pending_analysis_queue`
- 主循环每 tick 只处理有限文件数或固定时间预算

### 主要改动

#### 3.1 新增 `pending_analysis_queue`

涉及文件：

- `src/app/lsp/xlsp_server.h`
- `src/app/lsp/xlsp_server.c`
- `src/app/lsp/xlsp_workspace.c`

要求：

- 统一管理后台待分析文件
- 去重
- 支持 open doc 提升优先级

#### 3.2 主循环 drain 预算化

涉及文件：

- `src/app/lsp/xlsp_server.c`

要求：

- 每轮主循环按 “文件数预算” 或 “时间预算” drain 若干待分析任务
- 若 stdin 有请求输入、open doc 有 pending parse/diagnostics，应优先处理前台任务

#### 3.3 让 worker 结果用于“是否需要分析”的判定

> **前置**：实施前先审计 `XrLspIndexResult` 结构体当前返回什么数据，再决定增量判定策略（content_hash vs parse tree diff）。

涉及文件：

- `src/app/lsp/xlsp_index_pool.c`
- `src/app/lsp/xlsp_workspace.c`

要求：

- worker 结果至少携带 `content_hash` 或等价变更标识
- 若文件内容未变，主线程可直接跳过 analyzer 更新

#### 3.4 收敛遗留索引层

删除 `XrLspWorkspaceIndex` 遗留层。

`workspace_analyzer` 是唯一权威语义状态源，不保留任何半废弃的并行索引抽象。若 `workspace/symbol` 需要快速 lexical cache，在删除后按需新建一个职责明确的轻量缓存。

### 验收标准

- 大 workspace 初次索引时，completion/hover/didChange 响应不再明显卡顿
- 同一文件不会在一次索引周期中无意义地重复全量分析
- 旧遗留 workspace index 不再处于半废弃状态

### 后续可选优化（不作为本阶段必须项）

- 统计主线程每 tick 背景分析耗时
- 为 background indexing 增加 progress 指标与取消能力
- 按 folder 或 priority 分配分析预算

---

### 5.5 Phase 4 — 语义能力正确性收敛

### 目标

1. hover / definition / references / call hierarchy 的语义模型更一致
2. 文档内外行为不再明显两套逻辑
3. rename 和 formatting 的真实能力边界清晰可测

### 主要改动

#### 4.1 hover 优先走 position-aware 查询

涉及文件：

- `src/app/lsp/xlsp_analysis.c`（或拆分后的 navigation 模块）

要求：

- 优先 `xa_analyzer_lookup_at()`
- 仅在 analyzer 无位置信息时，才使用更保守的 fallback
- fallback 结果应标记为“低置信度路径”，避免覆盖更准结果

#### 4.2 references 跨文件搜索改为 analyzer 驱动优先

涉及文件：

- `src/app/lsp/xlsp_analysis.c`
- `src/app/lsp/xlsp_call_hierarchy.c`

目标顺序：

1. 当前文档：保持 scope-aware AST 搜索
2. 已打开文档：不再使用纯 lexer 名字匹配作为主路径
3. 未打开文档：优先使用 workspace analyzer / 明确的 symbol cache，而不是 open-doc 特判

#### 4.3 call hierarchy 补齐 unopened-file 路径

把当前 TODO 收敛成真实实现，避免“只要文件没打开就查不到 incoming/outgoing call”。

#### 4.4 rename 契约收敛

- 当前仅支持 single-document rename，修正文档/注释/测试以反映真实能力
- 等 references 工作区级语义收敛后，直接实现真实 cross-file rename
- 契约必须与实现一致，不做乐观承诺

#### 4.5 formatting 契约收敛

- range formatting：Phase 0 已关闭 capability，待 formatter 真正支持 range 后重新启用
- on-type formatting：明确定位为“缩进辅助”，不是完整 formatter 输出
- full document formatting：必须走统一 formatter 配置，不再使用 `xfmt_default_config`

### 验收标准

- shadowing 场景 hover/definition 结果不再明显错误
- references 在“当前文件 / 已打开文件 / 未打开文件”之间行为差距显著缩小
- call hierarchy 不再明显依赖“文件必须先打开”
- rename / formatting 的对外契约与真实行为一致

---

### 5.6 Phase 5 — 架构边界修复与大文件拆分

### 目标

1. `app/lsp` 不再依赖 `app/cli`
2. `xlsp_server.c` / `xlsp_analysis.c` 体量回落到安全区间
3. feature 模块职责清晰，后续修改不会持续堆到中心文件

### 主要改动

#### 5.1 下沉 formatter 到 `frontend/format`

涉及文件：

- `src/app/cli/xfmt.c`
- `src/app/cli/xfmt.h`
- `src/app/lsp/xlsp_analysis.c`
- `tests/unit/CMakeLists.txt`
- 相关 CMake 源文件列表

目标：

- `xfmt.[ch]` 迁移到 `src/frontend/format/`
- CLI/LSP/测试统一改为依赖新路径
- `tests/unit/lsp` 不再通过 `src/app/cli/xfmt.c` 才能编译

#### 5.2 拆分 `xlsp_server.c`

建议拆法：

- `xlsp_handlers_lifecycle.c`
- `xlsp_handlers_text_document.c`
- `xlsp_handlers_workspace.c`
- `xlsp_diagnostics.c`

要求：

- `xlsp_server.c` 只保留 create/free/run/dispatch table 主骨架
- 事件处理函数移出中心文件

#### 5.3 拆分 `xlsp_analysis.c`

建议拆法：

- `xlsp_navigation.c`
- `xlsp_rename.c`
- `xlsp_format.c`
- 诊断若仍过重，可进一步拆 `xlsp_diagnostics_analysis.c`

要求：

- 每个 feature 模块职责单一
- 方便单独加测试和做行为比对

#### 5.4 处理遗留 `XrLspWorkspaceIndex`

Phase 3 已删除 `XrLspWorkspaceIndex`。Phase 5 确认清理完毕，不残留相关头文件/声明/测试引用。

### 验收标准

- `app/lsp` 不再 include `app/cli` 头文件
- `xlsp_server.c` / `xlsp_analysis.c` 行数显著下降
- 新 feature 不再需要继续堆到两个中心文件
- 单测编译链路不再依赖错误层级的 formatter 文件

---

### 5.7 Phase 6 — 测试与回归体系补齐

### 目标

把 LSP 从“主要靠手工试用验证”的状态，提升到“有最低可信回归网”的状态。

### 主要改动

#### 6.1 扩充 unit tests

建议新增：

```text
tests/unit/lsp/
├── test_lsp_json.c                 # 现有
├── test_lsp_document.c             # 现有
├── test_lsp_diagnostics.c          # debounce / clear / disable / overflow
├── test_lsp_config.c               # xray.toml / runtime config / precedence
├── test_lsp_workspace.c            # workspace folder add/remove / purge / ignore
├── test_lsp_completion.c           # maxItems / ordering / config-driven clipping
├── test_lsp_format.c               # full formatting config behavior
└── test_lsp_navigation.c           # hover/definition/references basic semantic cases
```

#### 6.2 增加 LSP 回归测试入口

建议新建：

- `tests/regression/lsp/`
- `scripts/run_lsp_regression_tests.sh`

采用 stdio transcript 驱动：

- 启动 LSP server 子进程
- 发送 `initialize/didOpen/didChange/completion/definition/references/formatting` 等标准请求
- 校验 JSON-RPC 响应与 diagnostics 输出

#### 6.3 补充 architecture / contract 回归

至少覆盖：

- LSP capability snapshot
- formatter 依赖路径不再指向 `app/cli`
- multi-root add/remove folder 行为
- ignore 配置生效
- diagnostics enabled/disabled 切换

### 测试策略约束

- **unit test 优先测可隔离逻辑**：diagnostics queue、config precedence、ignore matching
- **高耦合 feature 优先用 transcript 回归**：navigation/completion/formatting 等
- 不追求 handler 级 mock 覆盖率，优先保证行为契约可回归

### 验收标准

- `tests/unit/lsp/` 至少覆盖 config / diagnostics / workspace / navigation / formatting
- LSP 有独立 regression 入口，而不是完全依赖手工 IDE 测试
- capability 与行为的关键契约有回归测试保护

---

## 6. 文件变更总览

### 6.1 新增文件（建议）

| 文件 | Phase | 说明 |
|------|-------|------|
| `docs/tasks/014-lsp-refactor.md` | now | LSP 实施文档 |
| `src/app/lsp/xlsp_diagnostics.c` | 0/5 | 诊断调度与发布 |
| `src/app/lsp/xlsp_handlers_lifecycle.c` | 5 | initialize / shutdown / exit |
| `src/app/lsp/xlsp_handlers_text_document.c` | 5 | didOpen / didChange / didClose 等 |
| `src/app/lsp/xlsp_handlers_workspace.c` | 5 | workspace 相关 handler |
| `src/app/lsp/xlsp_navigation.c` | 4/5 | hover / definition / references / highlight |
| `src/app/lsp/xlsp_rename.c` | 4/5 | rename |
| `src/app/lsp/xlsp_format.c` | 1/4/5 | formatting bridge |
| `tests/unit/lsp/test_lsp_diagnostics.c` | 0 | 诊断调度/禁用/清屏 |
| `tests/unit/lsp/test_lsp_config.c` | 1 | 配置模型 |
| `tests/unit/lsp/test_lsp_workspace.c` | 0/2 | multi-root / purge / ignore |
| `tests/unit/lsp/test_lsp_completion.c` | 1 | completion 配置与裁剪 |
| `tests/unit/lsp/test_lsp_format.c` | 1/4 | formatting 配置/契约 |
| `tests/unit/lsp/test_lsp_navigation.c` | 4 | hover / definition / references |
| `scripts/run_lsp_regression_tests.sh` | 6 | LSP transcript 回归 |

### 6.2 修改文件（核心）

| 文件 | Phase | 改动 |
|------|-------|------|
| `src/app/lsp/xlsp_server.c` | 0-5 | 诊断队列、主循环预算化、handler 拆分、capability 收敛 |
| `src/app/lsp/xlsp_server.h` | 0-5 | server/config/workspace 状态结构调整 |
| `src/app/lsp/xlsp_workspace.c` | 0-3 | multi-root、ignore、生删改 purge、索引调度 |
| `src/app/lsp/xlsp_index_pool.c` | 3 | worker 结果中增加变更标识/元数据 |
| `src/app/lsp/xlsp_analysis.c` | 1/4/5 | 配置接线、navigation/rename/format 抽离 |
| `src/app/lsp/xlsp_call_hierarchy.c` | 4 | unopened-file 支持 |
| `src/app/lsp/xlsp_completion.c` | 1 | maxItems / config-driven behavior |
| `src/app/lsp/xlsp_config.c` | 0/1 | `xray.toml` 支持面扩展 |
| `tests/unit/CMakeLists.txt` | 5/6 | LSP 测试接入、formatter 新路径 |
| `docs/engineering/README.md` | now | 增加文档入口 |
| `CMakeLists.txt` | 5/6 | 新源文件与测试目标接入 |

### 6.3 删除/下线（候选）

| 项目 | Phase | 原因 |
|------|-------|------|
| `documentRangeFormattingProvider` capability | 0 | 直接关闭，当前是全文替换 |
| `completion_auto_import` config | 1 | 未实现，直接删除 |
| `log_level` config | 1 | 未实现，直接删除 |
| `XrLspWorkspaceIndex` 老抽象 | 3 | 删除半废弃遗留层 |

---

## 7. 验证基线

若涉及代码修改，每个 Phase 完成后至少执行：

```bash
cmake --build build -j8
ctest --output-on-failure --test-dir build
scripts/run_regression_tests.sh
scripts/check_architecture.sh
```

若已补充 LSP 专项回归，则额外执行：

```bash
scripts/run_lsp_regression_tests.sh
```

### LSP 专项最低用例

1. **diagnostics**
   - 大于原 `MAX_PENDING_DIAG` 数量的文档批量修改
   - `diagnostics.enabled = false` 后清屏

2. **workspace**
   - initialize 带多个 `workspaceFolders`
   - 动态 add/remove folder
   - 删除文件后 definition / workspace symbol 不再命中

3. **config**
   - `completion.maxItems`
   - `format.tabSize`
   - `format.insertSpaces`
   - `ignore`

4. **navigation**
   - shadowing 场景 hover / definition
   - open doc / unopened file references
   - call hierarchy 跨文件

5. **formatting contract**
   - range formatting capability 已关闭，客户端 capability snapshot 应同步反映
   - full formatting 输出受 config 控制

---

## 8. 风险与缓解

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| P0 修复范围散，容易把 hotfix 做成大重构 | 中 | 高 | Phase 0 只修 correctness / contract，不混入文件拆分 |
| multi-root 清理 analyzer 时接口不够用 | 中 | 高 | 先补文件级 purge helper，再做 folder/prefix purge |
| 背景索引预算化后总索引耗时上升 | 中 | 中 | 先保证前台响应性；后续再做并行与优先级优化 |
| formatter 迁移引发 CLI/LSP 双侧回归 | 中 | 中 | 迁移前先补 formatter 相关单测和 LSP formatting 回归 |
| references / call hierarchy 语义收敛涉及 analyzer 能力边界 | 高 | 中 | 先统一 contract 和 fallback，再逐步扩大 analyzer 路径 |
| 大文件拆分时 link/include 回归 | 中 | 中 | 先抽 header/内部声明，再按 feature 拆，不跨 phase 混改 |

---

## 9. 回滚策略

- 每个 Phase 独立提交
- Phase 0 不与 Phase 5 混合
- formatter 下沉单独提交，不与 navigation/diagnostics 改动混在一起
- 若某阶段失败，直接回退整个阶段，不保留“半新半旧”的桥接状态

原则：**发现设计错误就直接删除重写，不通过兼容层维持。**

---

## 10. 推荐实施顺序（给后续新对话直接用）

如果后续要在新对话中按收益最大化推进，建议严格按下面顺序执行：

1. **先做 Phase 0**
   - 修 diagnostics overflow / enabled 开关
   - 修 ignore 接线
   - 修 multi-root init/add/remove 的 correctness
   - 下线错误 capability 宣称

2. **再做 Phase 1**
   - 收敛 config schema
   - 接通 `completion_max_items` / formatter config
   - 删除未实现的假配置项

3. **然后做 Phase 2**
   - workspace folder 状态结构化（`XlspWorkspaceFolderState`）
   - 统一 folder 生命周期、purge helper、事件处理

4. **接着做 Phase 3**
   - 先上 `pending_analysis_queue` + budgeted drain
   - 让后台索引不再阻塞前台

5. **接着做 Phase 4**
   - 收敛 hover/references/call hierarchy/rename contract

6. **最后做 Phase 5**
   - formatter 下沉
   - `xlsp_server.c` / `xlsp_analysis.c` 拆分
   - 确认 `XrLspWorkspaceIndex` 已在 Phase 3 删除，清理残留引用

7. **全过程同步做 Phase 6 的测试补齐**
   - 每完成一个行为面，就补对应 unit/regression test

---

## 11. 结论

`src/app/lsp` 当前的主要矛盾不是“没有功能”，而是：

- **correctness 没完全收口**
- **contract 有漂移**
- **workspace 状态不统一**
- **索引并行化没有真正换来稳定响应性**
- **架构边界和代码体量正在逼近失控点**

因此，最佳策略不是继续补 feature，而是按本文的顺序：

**先修 correctness 和 contract，再收敛 workspace/indexing，再做语义完善和架构拆分。**

每个阶段都直接采用最佳设计，不做兼容层，不留技术债务。LSP 模块将逐步收敛到“行为可信、结构清晰、后续可扩展”的状态。
