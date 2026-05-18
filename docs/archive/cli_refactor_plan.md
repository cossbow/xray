# CLI 模块重构实施文档（`src/app/cli`）

> **原则：不考虑向后兼容性，直接采用最佳设计。**
>
> - 删除旧接口，不做兼容层，不保留 deprecated 路径
> - 只保留语义正确、结构清晰、可验证的最终实现
> - 未完成能力不对外暴露，禁止“占位成功”命令
> - 每个阶段独立可合并、独立可回滚；阶段间严格前置

## 1. 背景

`src/app/cli` 目前已经承担了 Xray 的脚本执行、REPL、测试、格式化、检查、构建、包管理、LSP/DAP/MCP 启动等全部入口职责，但整体成熟度差异很大：

- `run` / `repl` / `test` 基本可用，但存在解析与行为一致性问题
- `check` / `fmt` / `deps` 可用但基础设施重复、边界条件不统一
- `build` 的帮助文案和真实能力不一致
- `pkg` 明显仍处于“壳子已搭好、后端未闭环”的阶段
- CLI 没有被当成稳定接口来做系统测试，导致真实 bug 长时间停留在主干

当前状态已经不适合继续小修小补。最佳策略不是继续补丁式修复，而是**按统一模型直接重建 CLI 层**。

---

## 2. 当前问题总览

### 2.1 路由与参数解析模型不可靠

当前 `xcli.c` 用 `commands[] + arg_offset` 做分发，多个命令内部再各自调用 `getopt_long()`。

这带来两个系统性问题：

1. **调用约定不一致**
   - 有的 handler 认为 `argv[0]` 是程序名/命令名占位
   - 有的 handler 直接把 `argv[0]` 当第一个真实参数
   - `arg_offset` 设计使 `repl/check/fmt` 一类命令极易出现跳参 bug

2. **解析逻辑分散**
   - `xcli.c` 做一层特殊判断
   - 各命令各自手写 `getopt_long()` 或手写字符串扫描
   - `help`、错误提示、未知参数处理、退出码都不统一

这不是局部 bug，而是**CLI 协议本身没有被建模**。

### 2.2 模块边界混乱

典型问题：`xfmt.[ch]` 放在 `src/app/cli/` 下，但 `src/app/lsp/xlsp_analysis.c` 直接复用它。

这说明：

- `xfmt` 实际上是通用“前端格式化能力”
- 但物理位置却挂在 `app/cli`
- 导致 `app/lsp -> app/cli` 的 sibling coupling

这类问题会持续扩散，最终让 `app/cli` 变成“顶层杂物间”。

### 2.3 命令成熟度不一致，且存在错误承诺

- `build --native` 的帮助文本承诺了完整 native/AOT 能力，但实际实现并不匹配
- `pkg remove` / `pkg update` 未实现却返回成功
- `pkg install` 是占位流程
- `pkg add` 不修改 `xray.toml`

CLI 不应该向用户暴露“看起来有、实际上没有”的能力。

### 2.4 公共基础设施缺失，重复逻辑大量存在

目前 `xcli_utils.[ch]` 只提供了很薄的一层工具函数，无法覆盖 CLI 真正重复的部分：

- 文件树遍历
- ignore 规则
- 输出/颜色/quiet 策略
- isolate 配置模板
- 原子写文件
- 命令错误报告
- 帮助文案生成
- 参数校验与互斥规则

于是 `check` / `fmt` / `test` / `build` / `run` 都在重复发明自己的轮子。

### 2.5 测试覆盖严重不足

当前几乎没有“把 CLI 当产品接口”来做的测试：

- 路由正确性
- 选项解析
- `--help` / `--version`
- 退出码
- `--fail-fast`
- 目录扫描与 ignore 规则
- 命令帮助与实际行为是否一致

缺乏这层测试，CLI 的真实行为就会长期依赖手工试用。

---

## 3. 目标

本次重构的最终目标不是“修几个命令”，而是建立一套**统一、可测试、可扩展、无技术债**的 CLI 架构。

### 3.1 最终目标

1. **统一命令协议**
   - 所有命令都通过同一套命令描述表注册
   - 所有命令都由同一套解析器解析
   - 所有帮助文案由命令描述自动生成

2. **统一 handler 输入模型**
   - 子命令不再直接操作原始 `argc/argv`
   - 子命令接收结构化的 invocation/request 对象
   - 子命令不再自己解析参数

3. **统一错误模型**
   - usage error、命令执行失败、功能不可用、内部错误有一致退出码
   - 所有用户可读错误统一经 CLI 诊断层输出

4. **统一公共基础设施**
   - 路径遍历、ignore、输出、isolate profile、写文件、时间统计统一下沉

5. **清理模块边界**
   - `xfmt` 从 `app/cli` 移出，成为通用前端能力
   - `app/cli` 只保留“顶层交互层”职责

6. **不再暴露未完成命令**
   - 任何未闭环能力，直接删除或标记为 unavailable
   - 不保留“空实现 + 成功返回”的路径

### 3.2 非目标

以下内容**不在本次重构范围**：

- shell completion
- 多语言帮助文案
- 复杂主题/颜色配置系统
- 包管理协议本身的重新设计（仅定义 CLI 入口契约，求解器算法另行设计）
- `build --native` 背后的 AOT 编译器深度设计（CLI 只负责正确暴露实际能力）

---

## 4. 目标 CLI 语法

重构后采用**显式子命令优先**的 CLI 语法。

### 4.1 顶层语法

```bash
xray --help
xray --version

xray run <entry.xr> [-- script args...]
xray eval <code>
xray repl
xray test [options] <file-or-dir...>
xray check [options] <file-or-dir...>
xray fmt [options] <file-or-dir...>
xray compile [options] <entry.xr>
xray build [options] <entry.xr>
xray deps [options] <entry.xr>
xray pkg <subcommand> [options]
xray info
xray lsp [options]
xray dap [options]
xray mcp-server [options]
```

### 4.2 允许保留的快捷语法

```bash
xray app.xr [-- script args...]
```

该形式被视为**一等语法**，等价于：

```bash
xray run app.xr [-- script args...]
```

这是为了脚本语言的常见使用体验而保留的快捷入口，不是兼容层。

快捷路径的歧义消解规则：

1. 以 `-` 开头的参数 → 视为 global flag 或传递给 `run`
2. 与已注册命令名完全匹配 → 走命令路由
3. 以 `.xr` 结尾 → 走 `run` 快捷路径
4. 其它 → 报未知命令错误（**不再**尝试将无扩展名的参数当作脚本文件执行）

理由：当前代码中 `cli_file_exists(cmd)` 的兜底会导致文件名与未来命令名冲突时行为不可预测。

### 4.3 明确删除的旧语法

以下路径在重构中直接移除：

- `xray -e 'code'` → 改为 `xray eval 'code'`
- `xray init` / `xray add` / `xray remove` / `xray install` / `xray update` / `xray tree` / `xray login` / `xray publish`
  - 统一改为 `xray pkg <subcommand>`
- 所有依赖“隐式 `arg_offset`”才能工作的子命令转发方式

原则：**只保留一种主路径，不保留平行入口。**

---

## 5. 目标架构

### 5.1 顶层职责拆分

重构后 `src/app/cli` 内部分为两层，同时将通用能力抽到更低的模块：

#### A. CLI 核心基础设施（`xcli_spec/parser/dispatch/help/diag/fs/isolate/output`）

负责：

- 命令注册与子命令树
- 参数解析
- 诊断输出与退出码
- 帮助文案自动生成
- 文件树遍历与 ignore 规则
- isolate profile 工厂
- 公共输出策略（颜色、quiet、verbose、json）
- 信号处理（crash handler）
- 命令模糊匹配建议

#### B. 命令实现层（`cmd/xcli_cmd_*.c`）

负责：

- `run/eval/repl/test/check/fmt/compile/build/deps/pkg/info/lsp/dap/mcp-server` 的业务逻辑
- 只消费结构化 `XrCliInvocation`，不自己解析参数

#### C. 通用能力抽取（不属于 CLI 层）

- `xfmt` 移出 `src/app/cli`
- 新位置：`src/frontend/format/xfmt.[ch]`
- LSP 与 CLI 共同依赖该模块
- 这不是 CLI 的第三层，而是架构边界修正——将误放在 L8 的能力下沉到 L4

---

### 5.2 目标目录结构

```text
src/app/cli/
├── xcli.c                  # main，仅负责启动 CLI 核心
├── xcli_spec.h
├── xcli_spec.c             # 命令/选项声明
├── xcli_parser.h
├── xcli_parser.c           # 通用参数解析器
├── xcli_dispatch.h
├── xcli_dispatch.c         # 顶层分发
├── xcli_help.h
├── xcli_help.c             # 帮助文案生成
├── xcli_diag.h
├── xcli_diag.c             # 错误/警告/退出码
├── xcli_fs.h
├── xcli_fs.c               # 路径遍历、ignore、原子写文件
├── xcli_isolate.h
├── xcli_isolate.c          # isolate profile 工厂
├── xcli_output.h
├── xcli_output.c           # 颜色/quiet/json 等输出策略
└── cmd/
    ├── xcli_cmd_run.c
    ├── xcli_cmd_eval.c
    ├── xcli_cmd_repl.c
    ├── xcli_cmd_test.c
    ├── xcli_cmd_check.c
    ├── xcli_cmd_fmt.c
    ├── xcli_cmd_compile.c
    ├── xcli_cmd_build.c
    ├── xcli_cmd_deps.c
    ├── xcli_cmd_pkg.c
    ├── xcli_cmd_info.c
    ├── xcli_cmd_lsp.c
    ├── xcli_cmd_dap.c
    └── xcli_cmd_mcp.c
```

对应下沉模块：

```text
src/frontend/format/
├── xfmt.h
└── xfmt.c
```

### 5.3 删除的旧文件 / 旧设计

以下内容直接删除，不保留兼容壳：

- `xcli_utils.[ch]`
- `xcli.c` 中的 `arg_offset` / `is_pkg_alias` 模型
- 子命令私有的 `getopt_long()` 参数解析逻辑
- `xcli.h` 中为帮助文案暴露的大量 `print_*_help()` 声明
- 顶层 `pkg` alias 入口

---

### 5.4 核心数据模型

### 全局上下文

```c
typedef struct {
    bool color;             // resolved: --color/--no-color/auto
    bool verbose;
    bool quiet;
    bool json_output;
    const char *program;    // argv[0], for diagnostics
} XrCliContext;
```

`XrCliContext` 在 `xr_cli_main()` 入口处从 global flags 解析一次，全程只读传递给每个 handler。

### 命令规格

```c
typedef enum {
    XR_CLI_VALUE_NONE,
    XR_CLI_VALUE_BOOL,
    XR_CLI_VALUE_INT,
    XR_CLI_VALUE_STRING,
} XrCliValueKind;

typedef struct {
    const char *long_name;
    int short_name;
    XrCliValueKind value_kind;
    bool required;
    bool repeatable;
    const char *value_name;
    const char *help;
} XrCliOptionSpec;

typedef struct XrCliCommandSpec XrCliCommandSpec;
struct XrCliCommandSpec {
    const char *name;
    const char *summary;
    const XrCliOptionSpec *options;
    int positional_min;
    int positional_max;         // -1 = unlimited
    bool allow_passthrough;     // support `--` separator
    int (*handler)(const XrCliInvocation *inv);
    // Subcommand support (for `pkg` etc.)
    const XrCliCommandSpec *subcommands;  // NULL-terminated array, or NULL
    int subcommand_count;
};
```

子命令模型说明：

- 平坦命令（`run/test/check/...`）：`subcommands = NULL`，dispatch 直接调用 `handler`
- 有子命令的命令（`pkg`）：`subcommands` 指向子命令表，dispatch 先查找子命令再调用对应 `handler`
- 只支持一级子命令，不做无限嵌套（YAGNI）
- `pkg` 的 `handler` 字段为 NULL，子命令各自有自己的 `handler`

### 选项查询模型

```c
// Flat array indexed by option position in spec.
// CLI option count is always small (<20), so linear scan is fine.
typedef struct {
    const XrCliOptionSpec *spec;  // back-pointer to spec array
    int count;                    // number of options in spec
    bool *present;                // present[i]: option i was given
    const char **values;          // values[i]: string value (NULL if not given)
} XrCliOptionMap;
```

设计理由：CLI 选项数量始终很少（< 20），用 spec 索引的平坦数组比哈希表更简单、更可预测、零分配开销。查询时按 `long_name` 或 `short_name` 在 spec 中找索引，再用索引取 `present[i]` / `values[i]`。

### Invocation 模型

```c
typedef struct {
    const XrCliContext *ctx;
    const XrCliCommandSpec *spec;
    XrCliOptionMap options;
    int positional_count;
    const char **positionals;
    int passthrough_argc;
    char **passthrough_argv;
} XrCliInvocation;
```

### 退出码模型

```c
typedef enum {
    XR_CLI_EXIT_OK = 0,
    XR_CLI_EXIT_FAIL = 1,
    XR_CLI_EXIT_USAGE = 2,
    XR_CLI_EXIT_UNAVAILABLE = 3,
    XR_CLI_EXIT_INTERNAL = 4,
} XrCliExitCode;
```

规则：

- **usage 错误**：参数缺失、未知选项、互斥冲突 → `2`
- **命令执行失败**：测试失败、格式检查失败、语法错误、网络失败 → `1`
- **功能暂不可用**：编译时未启用某功能、尚未支持的 backend → `3`
- **内部错误**：分配失败、状态不变量被破坏 → `4`

禁止：

- 未实现命令返回 `0`
- 同一类错误在不同命令返回不同退出码

---

### 5.5 解析与分发流程

顶层只允许一套分发逻辑：

```text
main()
  -> install signal handlers (SIGSEGV/SIGBUS crash_handler)
  -> xr_cli_main(argc, argv)
     -> argc == 1 (no args)?  -> default to "repl" command
     -> parse global flags (--help/--version/--color/--no-color)
     -> build XrCliContext from global flags
     -> detect routing target:
        1. argv[1] matches registered command name -> use that command
        2. argv[1] matches registered command with subcommands -> consume argv[2] as subcommand
        3. argv[1] ends with ".xr" -> implicit "run" command
        4. otherwise -> fuzzy-match suggest_command(), report unknown command error
     -> load command spec
     -> parse command options into XrCliInvocation
     -> validate arity / mutual exclusion / passthrough
     -> call handler(inv)
```

关键约束：

1. **所有 handler 都接收结构化 invocation**
2. **任何命令都不得再手工解析 `argc/argv`**
3. **帮助文案只从 command spec 生成**
4. **未知参数的报错只能由 parser 产生，不能由子命令各自打印半套错误**

信号处理归属：

- **crash handler**（SIGSEGV/SIGBUS）：保留在 `xcli.c` 薄入口的 `main()` 中
- **SIGINT handler**：仅由 `xcli_cmd_repl.c` 在 REPL 会话期间安装/卸载
- 其它命令不应安装自己的信号处理

命令建议（"did you mean?"）：

- 保留当前 Levenshtein 距离匹配逻辑
- 移入 `xcli_dispatch.c`，由分发层在路由失败时统一调用
- 只对 `commands[]` 表中非隐藏命令做匹配

---

### 5.6 公共基础设施设计

### A. `xcli_fs`

统一提供：

- 递归遍历
- 统一 ignore 规则
- `.xr` 过滤
- 原子写文件
- stdin / file 读取

统一 ignore 规则至少覆盖：

- `.git/`
- `.svn/`
- `.hg/`
- `build/`
- `build-asan/`
- `build-release/`
- `node_modules/`
- `.xray/`
- 隐藏目录（默认跳过，命令可显式覆盖）

可扩展性决策：

- 本次重构**暂不支持** `.xrayignore` 或 `xray.toml` 中的自定义 ignore 规则
- 但 `xcli_fs` 的 API 应预留参数位（例如接受额外的 ignore pattern 数组），以便将来无痛添加
- 默认规则以编译期常量数组形式存在，不做运行时配置解析

### B. `xcli_isolate`

提供 profile 化创建接口：

- `XR_CLI_ISOLATE_RUN`
- `XR_CLI_ISOLATE_EVAL`
- `XR_CLI_ISOLATE_PARSE`
- `XR_CLI_ISOLATE_ANALYZE`
- `XR_CLI_ISOLATE_TEST`
- `XR_CLI_ISOLATE_REPL`

这样：

- `run` / `test` / `check` / `fmt` 不再各自手拼初始化流程
- JIT / trace / dump-bytecode / worker 数等统一从 profile + option 覆盖得到

### C. `xcli_diag`

统一提供：

- usage error
- command error
- unavailable error
- internal error
- 带 command name 的错误前缀
- 建议命令输出

### D. `xcli_output`

统一提供：

- color on/off/auto
- quiet
- verbose
- json/plain
- terminal capability 检测

这样 `test` / `fmt` / `check` / `info` 的输出策略将统一，不再各自决定颜色和摘要格式。

### E. JSON 输出

项目中 LSP / DAP 层已有 JSON 序列化设施。CLI 层的 `--json` 输出应复用现有 JSON writer（例如 `xjson.[ch]`），而不是新建一套。如果现有设施不在可依赖层级内，则先将其下沉到 `base/` 或 `runtime/value/`，再由 CLI 引用。

禁止：命令自己用 `printf` / `sprintf` 手写 JSON 转义。

---

## 6. 命令级目标设计

### 6.1 `run` / `eval`

### 目标

- `run` 只负责执行脚本文件或 stdin 脚本
- `eval` 只负责执行一段代码字符串
- 两者都走同一套执行上下文构建逻辑

### 设计决策

- 删除顶层 `-e`
- 新增显式命令 `eval`
- `run` 负责 `args/__file__/__dir__`、模块系统、script info 的完整初始化
- `eval` 也应构建明确的虚拟 script identity（例如 `<eval>` / `<stdin>`），避免执行路径与文件路径模式差异过大

### 验收标准

- `xray run app.xr -- a b`
- `xray app.xr -- a b`
- `echo 'print(1)' | xray run -`
- `xray eval 'print(1)'`

四种路径都由同一套 parser 与同一套 runtime setup 驱动。

### 6.2 `repl`

### 目标

- 保留现有多行输入、history、`.load`、`.time`
- 但参数解析与输出策略并入 CLI 核心

### 设计决策

- `repl` 不再自己解析 `--help`
- history 路径、颜色策略、`--no-color` 统一走通用 option
- `.reset` 的 isolate 重建必须走 `xcli_isolate`，而不是手工复制初始化代码

### 6.3 `check` / `fmt`

### 目标

- 两者共享同一套目录遍历和 ignore 规则
- 两者都支持统一的 `--quiet` / `--verbose` / `--json` 策略（如需要）

### 设计决策

- `check` / `fmt` 的“按目录递归扫描”全部下沉到 `xcli_fs`
- `fmt` 使用的 `xfmt` 已移到 `frontend/format`
- `fmt` 写回必须使用原子写文件流程
- `check` 与 `fmt` 都必须对显式传入的非 `.xr` 文件给出明确错误，而不是静默尝试

### 6.4 `test`

### 目标

- 保留“每个文件一个 isolate”的核心模型
- 但补齐真正的 CLI 行为契约

### 设计决策

- `--fail-fast` 在并行模式下必须有真实语义：
  - 一旦出现首个失败，停止调度新文件
  - 已经启动的 worker 允许自然收敛
- 输出格式通过 `xcli_output` 统一
- 文件收集与 ignore 规则通过 `xcli_fs` 统一

### 验收标准

- `xray test -j8 --fail-fast tests/` 在首个失败后不再领取新任务
- `xray test --quiet` 不输出人类摘要，只保留退出码语义

### 6.5 `compile` / `build` / `deps`

### 目标

- 重新建立清晰、可信的产物语义
- CLI 文案必须与真实能力完全一致

### 设计决策

#### `compile`

- 只负责生成中间产物：bytecode / C source / header
- 不承担“链接可执行文件”的职责

#### `build`

- 只负责生成可执行文件
- backend 必须显式化，例如：
  - `--backend=vm`
  - `--backend=aot`

若某 backend 尚未具备真实能力：

- **不暴露**该 backend
- 或者返回 `XR_CLI_EXIT_UNAVAILABLE`
- 帮助文案不得继续宣传“完整可用”

#### `deps`

- 只负责报告依赖图/依赖清单/可机器消费的结果
- 不应继续承担“生成一段半成品 shell script”的模糊职责
- JSON 输出必须由统一的 JSON writer 负责，而不是命令自己手写不完整转义

### 6.6 `pkg`

### 原则

包管理命令必须满足“**真实执行、真实失败、真实落盘**”三项要求，否则不对外暴露。

### 阶段性决策

在真正的 resolver/lockfile/install pipeline 完成前：

- 移除返回成功但实际未做事的命令
- `pkg` 命令表只注册真实可用子命令

最终目标：

- `pkg init`：真实创建项目骨架
- `pkg add/remove`：真实修改 `xray.toml`
- `pkg install/update`：真实解析、下载、写 lockfile
- `pkg tree`：基于真实 lockfile 或解析结果输出
- `pkg login/publish`：真实认证与发布

禁止：

- 打印“not yet implemented”但返回成功
- 网络失败被吞掉
- 声称安装成功但 `xray.toml` / lockfile 未发生变化

### 6.7 `lsp` / `dap` / `mcp-server`

### 目标

- 这些命令虽然功能简单，但仍必须纳入统一 parser/help/exit code 模型

### 设计决策

- 删除各自手写参数扫描
- 所有 server 命令统一支持：
  - `--help`
  - `--version`（如适用）
  - 未知参数报错
- 编译期未启用时返回 `XR_CLI_EXIT_UNAVAILABLE`

### 当前状态说明

- `lsp`：功能完整，已有独立模块 `src/app/lsp/`，CLI 层只负责启动
- `dap`：功能部分完整，已有独立模块 `src/app/dap/`，CLI 层只负责启动和 transport 选择
- `mcp-server`：当前状态待确认——可能仅有 `#ifdef XR_HAS_MCP` 的条件编译桩。Phase 6 工作量应根据其实际完成度调整

### 6.8 `info`

### 目标

- 打印版本、环境信息、配置路径等诊断信息
- 纳入统一 parser/help/exit code 模型

### 设计决策

- 保留当前 `cmd_info` 的核心职责（版本、缓存路径、注册表信息）
- 输出通过 `xcli_output` 统一，支持 `--json` 格式
- 不再在 `info` 内部硬编码 `printf` 格式

---

## 7. 实施计划

### 阶段依赖图

```text
Phase 1 → Phase 2 → Phase 3 (execution)
                     → Phase 4 (artifact)    ← 3 和 4 可并行
            Phase 2 → Phase 5 (pkg)          ← 5 只依赖 1+2
                     → Phase 6 (server+hardening) ← 最后收口
```

Phase 3 和 Phase 4 **互不依赖**，可以并行开发。Phase 5 只依赖 Phase 1+2（`pkg` 子命令模型和公共基础设施），不依赖 Phase 3/4。

---

### Phase 1 — CLI Core Rewrite（解析、分发、帮助生成）

### 目标

用 declarative spec + 统一 parser **直接替换**当前 `arg_offset + raw argc/argv + 每命令各自解析` 模型。不做 adapter 桥接——无外部用户，代码量小，直接一步到位。

### 改动

1. 新建：
   - `xcli_spec.[ch]`
   - `xcli_parser.[ch]`
   - `xcli_dispatch.[ch]`
   - `xcli_help.[ch]`
   - `xcli_diag.[ch]`
2. 重写 `xcli.c`：只保留 `main()` 与 `xr_cli_main()` 薄入口
3. 删除：
   - `arg_offset` / `is_pkg_alias` 模型
   - 顶层 `pkg` alias 命令（`init/add/remove/install/update/tree/login/publish`）
   - 所有 `print_*_help()` 函数
   - 旧 `xray -e` 入口
4. 引入显式 `eval` 命令
5. 所有命令帮助改为从 spec 自动生成
6. 零参数路径默认到 `repl`
7. 命令建议（`suggest_command`）迁移到 `xcli_dispatch.c`

### 验收

- `repl/check/fmt/test/run` 的选项解析不再依赖 `argv` 偏移
- `xray help <cmd>` 与 `<cmd> --help` 输出来自同一套 spec
- `xray eval 'print(1)'` 工作正常
- `xray init` 等旧 alias 已不可用
- 新分发逻辑覆盖零参数 → REPL、`.xr` 快捷路径、未知命令建议
- 新增单测：
  - `tests/unit/test_cli_parser.c`
  - `tests/unit/test_cli_dispatch.c`
  - `tests/unit/test_cli_help.c`

### Phase 2 — Shared Infrastructure + Boundary Cleanup

### 目标

消除 CLI 内重复逻辑，并修正模块边界。

### 改动

1. 新建：
   - `xcli_fs.[ch]`
   - `xcli_isolate.[ch]`
   - `xcli_output.[ch]`
2. 删除 `xcli_utils.[ch]`
3. `xfmt.[ch]` 迁移到 `src/frontend/format/`
4. `src/app/lsp/xlsp_analysis.c` 改为依赖 `frontend/format`
5. `check/fmt/test` 的目录遍历统一改用 `xcli_fs`
6. `run/repl/test/check/fmt` 的 isolate 初始化统一改用 `xcli_isolate`

### 验收

- `app/lsp` 不再 include `app/cli` 的 formatter 头
- `check/fmt/test` 的 ignore 行为一致
- CLI 层不再保留直接覆盖原指针的 `xr_realloc` 写法

### Phase 3 — Execution Commands Rewrite

### 目标

把 `run/eval/repl/check/fmt/test/info` 全部改为 typed invocation handler。

### 改动

1. 新建/改写：
   - `cmd/xcli_cmd_run.c`
   - `cmd/xcli_cmd_eval.c`
   - `cmd/xcli_cmd_repl.c`
   - `cmd/xcli_cmd_check.c`
   - `cmd/xcli_cmd_fmt.c`
   - `cmd/xcli_cmd_test.c`
   - `cmd/xcli_cmd_info.c`
2. 删除这些命令内部的参数解析代码
3. `test --fail-fast` 改成真正的全局 stop-scheduling 语义
4. `fmt` 改成原子写文件
5. `run` / `eval` 统一 script identity 和 runtime setup

### 验收

- `xray eval` 替代 `xray -e`
- `xray test -jN --fail-fast` 语义正确
- `xray fmt --check` / `xray check -v` / `xray repl --no-color` 均由统一 parser 驱动

### Phase 4 — Artifact Pipeline Rewrite

### 目标

重建 `compile/build/deps` 的 CLI 契约，使对外承诺与真实能力一致。

### 改动

1. 改写：
   - `cmd/xcli_cmd_compile.c`
   - `cmd/xcli_cmd_build.c`
   - `cmd/xcli_cmd_deps.c`
2. `build` backend 语义显式化
3. 未完成 backend 不暴露或返回 unavailable
4. `deps` 去掉半成品 shell script 生成职责
5. 复用现有 JSON writer（如需下沉层级则先进行），禁止手写 JSON 输出

### 验收

- 帮助文案与真实能力一一对应
- `build` 不再宣传未实现能力
- `deps --json` 结果由统一 writer 生成

### Phase 5 — Package Manager Re-introduction

### 目标

只在真实后端能力完成后重新引入完整 `pkg` 命令族。

### 改动

1. 先下线假实现子命令
2. 重写：
   - `cmd/xcli_cmd_pkg.c`
3. 按真实后端能力分批注册：
   - `init`
   - `add/remove`
   - `install/update`
   - `tree`
   - `login/publish`
4. 所有落盘修改通过统一文件更新流程完成

### 验收

- `pkg add/remove` 真实修改 `xray.toml`
- `pkg install/update` 真实生成/更新 lockfile
- 未实现子命令不出现在 help 中

### Phase 6 — Server Commands + Hardening

### 目标

把 `lsp/dap/mcp-server` 收口到统一 CLI 规范，补齐回归测试与文档，做最终硬化。

### 改动

1. 改写：
   - `cmd/xcli_cmd_lsp.c`
   - `cmd/xcli_cmd_dap.c`
   - `cmd/xcli_cmd_mcp.c`
2. 所有 server 命令使用统一 parser/help/exit code
3. 新增 CLI 回归测试
4. 更新 `docs/engineering/README.md`

### 验收

- `--help` / 未知参数 / 编译期开关关闭场景行为一致
- CLI 文档与实现无偏差

---

## 8. 文件变更总览

### 8.1 新增文件

| 文件 | Phase | 说明 |
|------|-------|------|
| `src/app/cli/xcli_spec.h` / `.c` | 1 | 命令与选项规格（含子命令树） |
| `src/app/cli/xcli_parser.h` / `.c` | 1 | 通用参数解析器 |
| `src/app/cli/xcli_dispatch.h` / `.c` | 1 | 顶层分发（含命令建议） |
| `src/app/cli/xcli_help.h` / `.c` | 1 | 帮助文案自动生成 |
| `src/app/cli/xcli_diag.h` / `.c` | 1 | 诊断与退出码 |
| `src/app/cli/xcli_fs.h` / `.c` | 2 | 文件系统与遍历 |
| `src/app/cli/xcli_isolate.h` / `.c` | 2 | isolate profile 工厂 |
| `src/app/cli/xcli_output.h` / `.c` | 2 | 输出策略 |
| `src/app/cli/cmd/xcli_cmd_*.c` | 3-6 | 各子命令实现 |
| `src/frontend/format/xfmt.h` / `.c` | 2 | 通用格式化模块 |
| `tests/unit/test_cli_parser.c` | 1 | parser 单测 |
| `tests/unit/test_cli_dispatch.c` | 1 | dispatch 单测（含零参数/快捷路径/命令建议） |
| `tests/unit/test_cli_help.c` | 1 | help 自动生成单测 |
| `tests/unit/test_cli_fs.c` | 2 | 文件遍历/ignore 单测 |
| `tests/regression/90_cli/*` | 3-6 | CLI 回归测试 |

### 8.2 修改文件

| 文件 | Phase | 改动 |
|------|-------|------|
| `src/app/cli/xcli.c` | 1 | 变为薄入口（保留 crash handler） |
| `src/app/cli/xcli.h` | 1-3 | 删除旧 handler/help 声明，替换为新核心接口 |
| `src/app/lsp/xlsp_analysis.c` | 2 | 改用 `frontend/format/xfmt.h` |
| `CMakeLists.txt` | 2-6 | 新增 CLI 核心文件、formatter 新路径、CLI 测试 |
| `docs/engineering/README.md` | 6 | 增加文档入口 |

### 8.3 删除文件

| 文件 | Phase | 原因 |
|------|-------|------|
| `src/app/cli/xcli_utils.c` / `.h` | 2 | 工具层过薄且职责混杂，拆为专用模块 |
| 旧 `xcmd_*.c` 文件 | 3-6 | 被 `cmd/xcli_cmd_*.c` 替换 |
| `src/app/cli/xfmt.c` / `.h` | 2 | 迁移到 `frontend/format` |

---

## 9. 验证基线

每个阶段完成后至少执行：

```bash
cmake --build build -j8
ctest --output-on-failure --test-dir build
scripts/run_regression_tests.sh
scripts/check_architecture.sh
```

阶段内的最低要求：

- 新增单测必须覆盖该阶段引入的核心抽象
- 所有帮助文案与实现行为同步更新
- CLI 命令若改变用户可见语法，必须同步更新 `README.md` / `README_CN.md` / 相关脚本示例

### CLI 专项测试要求

至少覆盖以下用例：

1. **路由**
   - `xray`（零参数 → REPL）
   - `xray repl --no-color`
   - `xray check -v foo.xr`
   - `xray fmt --check src/`
   - `xray app.xr -- a b`
   - `xray nosuchcmd`（模糊匹配建议）
   - `xray myfile`（无扩展名，应报未知命令错误）

2. **错误语义**
   - 未知命令
   - 未知选项
   - 缺失参数
   - feature unavailable

3. **命令行为**
   - `test --fail-fast -jN`
   - `fmt` 原子写回
   - `deps --json`
   - `pkg` 未注册子命令不出现在帮助中

---

## 10. 风险与缓解

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 一次性改动过大，CLI 全面回归 | 中 | 高 | 按 phase 落地；Phase 1 范围明确（只解析/分发/帮助），不涉及子命令业务重写 |
| `build` / `pkg` 重构被后端未完成能力拖住 | 高 | 高 | CLI 先移除假能力，不等待后端补齐 |
| `xfmt` 迁移导致 LSP 格式化回归 | 中 | 中 | 先补 formatter API 单测，再迁移引用 |
| `test --fail-fast` 并行语义改动引入竞态 | 中 | 中 | 新增 stop flag 单测 + ASan/TSan 构建验证 |
| 文档与实现再次漂移 | 中 | 中 | 每 phase 完成时同步更新帮助快照和回归测试 |

---

## 11. 中止 / 回退策略

- 每个 Phase 单独提交。
- 阶段间不混改。
- 若 Phase N 失败，直接回退到 Phase N-1，不保留半成品兼容桥。
- **禁止**在新架构外再包一层“临时兼容适配器”。

原则：**回退是回退整个阶段，不是保留旧代码与新代码并存。**

---

## 12. 最终验收标准

CLI 重构完成后，应满足以下条件：

1. `src/app/cli` 不再存在原始 `argc/argv + getopt_long + arg_offset` 的分散模型
2. `app/lsp` 不再依赖 `app/cli` 的通用能力
3. 不存在“未实现但返回成功”的命令
4. 帮助文案全部自动生成，且与实现一致
5. `build` / `pkg` 的对外能力描述与真实能力完全一致
6. CLI 有独立 unit test 与 regression test，不再依赖手工试跑
7. 所有 CLI 代码遵守项目内存与可见性规则，不引入新的临时兼容层

---

## 13. 一句话决策

**这次不是修 CLI，而是重建 CLI：删掉旧协议、删掉假能力、删掉跨层耦合，用 declarative spec + typed invocation + 统一基础设施，把 `src/app/cli` 变成一个真正可维护的 L8 入口层。**
