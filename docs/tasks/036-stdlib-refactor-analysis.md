# 标准库逐库分析与重构计划

## 目标

`stdlib/` 当前约 5.8 万行，覆盖基础工具、数据格式、压缩、加密、文件系统、网络、WebSocket、集群等能力。这个分支的目标不是一次性重写，而是先逐个库完成源码分析，沉淀问题清单和优化方案，再统一安排实施。

本任务只定义分析顺序、检查标准和输出模板。具体代码修改应等待对应库的分析文档完成后再执行。

## 当前入口与构建事实

- **构建入口**：`CMakeLists.txt` 通过 `STDLIB_CORE_SRC` 和模块化选项收集 `stdlib/*/*.c`。
- **默认形态**：`XR_STDLIB_FULL=ON` 时递归编译全部标准库模块，排除 sqlite 相关源文件。
- **模块注册**：`src/module/xmodule.c` 的 `xr_module_register_stdlib()` 统一注册 native loader。
- **loader 声明**：`src/module/xmodule_loaders.h` 维护所有标准库 loader 声明。
- **公共绑定工具**：`stdlib/common.h` 提供字符串参数、字符串返回值和 export 宏；`common_io.h`、`common_parser.h`、`common_writer.h`、`ctxbuf.h`、`stdlib_cache.h` 是横切辅助层。
- **现有单测入口**：`tests/unit/stdlib/`、`tests/unit/http/`、`tests/unit/encoding/`、`tests/unit/regex/` 覆盖了部分模块，但覆盖并不完整。

## 库清单

### 共享基础

| 范围 | 文件 |
|---|---|
| 绑定与参数工具 | `stdlib/common.h` |
| I/O 辅助 | `stdlib/common_io.h` |
| parser/writer 抽象 | `stdlib/common_parser.h`, `stdlib/common_writer.h` |
| 上下文缓冲 | `stdlib/ctxbuf.h` |
| 缓存辅助 | `stdlib/stdlib_cache.h` |

### Native 模块

| 顺序 | 模块 | 类型 | 初始关注点 |
|---:|---|---|---|
| 1 | `time` | core | 时间源、精度、跨平台语义 |
| 2 | `math` | core | 数值边界、NaN/Inf、类型返回一致性 |
| 3 | `gc` | core | 暴露 API 是否符合 runtime 生命周期 |
| 4 | `path` | core | 路径规范化、Windows/POSIX 差异 |
| 5 | `base64` | core | 编解码边界、错误返回、测试覆盖 |
| 6 | `encoding` | core | Unicode/hex/base64 责任边界 |
| 7 | `url` | core | percent encode/decode、URL parser 完整性 |
| 8 | `datetime` | core | 日期解析、时区、格式化一致性 |
| 9 | `log` | core | 全局状态、并发输出、级别语义 |
| 10 | `json` | core | 与 runtime JSON/对象模型的边界 |
| 11 | `csv` | data | parser 状态机、流式处理、转义语义 |
| 12 | `toml` | data | 与 `src/base/xtoml` 的职责重复 |
| 13 | `yaml` | data | scanner/parser 安全性、格式覆盖 |
| 14 | `xml` | data | 节点生命周期、实体解析、安全默认值 |
| 15 | `regex` | core | NFA/DFA 边界、缓存、最坏情况行为 |
| 16 | `io` | filesystem | 阻塞 I/O 标记、错误模型、路径安全 |
| 17 | `os` | filesystem | 进程/环境/平台 API 边界 |
| 18 | `crypto` | crypto | 哈希/HMAC/随机数安全语义 |
| 19 | `compress` | compress | zlib 依赖、内存上限、流式接口 |
| 20 | `net` | network | socket 生命周期、TLS/DNS、协程阻塞语义 |
| 21 | `http` | network | parser/client/server/router/cookie/multipart 分层 |
| 22 | `ws` | network | 握手、frame、deflate、连接生命周期 |
| 23 | `cluster` | optional | 体量最大之一，需优先确认是否仍是标准库核心能力 |
| 24 | `test_yield` | test/support | 是否应保留在标准库构建和注册表中 |

## 分析顺序

1. **横切基础先行**：先审查公共 helper、CMake 收集、loader 注册、错误返回约定，避免每个库重复记录同一类问题。
2. **小型 core 库优先**：从 `time/math/gc/path/base64/encoding/url/datetime/log` 开始，快速建立统一模板和判定标准。
3. **数据格式库集中分析**：`json/csv/toml/yaml/xml` 共享 parser/writer、安全边界和 runtime value 转换问题，应连续处理。
4. **算法型库单独分析**：`regex` 体量较大且有独立编译/执行模型，单独成文。
5. **系统能力库后置**：`io/os/crypto/compress` 需要更多平台、依赖和安全检查。
6. **网络栈最后处理**：`net/http/ws/cluster` 体量大、涉及协程和阻塞语义，放在基础规范稳定后分析。

## 每库分析模板

每个库分析完成后，按以下结构形成独立优化文档：

```markdown
# stdlib/<module> 分析与优化建议

## 模块职责

说明该库应该提供什么能力，不应该承担什么能力。

## 源码结构

列出主要 `.c/.h` 文件、核心类型、loader、export API。

## 当前行为契约

记录公开函数、参数类型、返回值、错误返回、异常/诊断策略。

## 依赖与架构边界

检查 include 方向、是否跨层依赖高层、是否重复实现 base/runtime 已有能力。

## 内存与生命周期

检查分配 API、GC 交互、对象 ownership、缓存生命周期、释放路径。

## 并发、阻塞与协程语义

确认 native 函数是否需要 `SLOW` 或 yieldable 标记，是否可能阻塞 worker。

## 安全与鲁棒性

检查输入长度、整数溢出、parser 递归深度、资源上限、默认安全策略。

## 性能与二进制体积

识别热点路径、重复 buffer、可共享表、可选依赖、模块化构建收益。

## 测试覆盖

列出现有单测/回归测试，指出缺口和建议新增用例。

## 问题清单

按“严重 / 高 / 中 / 低”记录问题，每项包含现象、原因、影响和建议方向。

## 后续实施建议

只描述事实和目标，不在分析文档中直接改代码。
```

## 统一检查项

- **API 一致性**：导出命名、参数检查、错误返回、null 处理是否统一。
- **类型一致性**：native C 返回值是否与语言层语义一致。
- **内存安全**：禁止直接系统分配 API；所有分配失败路径必须可控。
- **架构边界**：低层不能 include 高层；内部头不能 include 公共总头。
- **阻塞语义**：文件、网络、压缩、DNS、TLS 等潜在阻塞操作必须明确调度语义。
- **资源上限**：parser、压缩、网络、cluster 必须有长度、深度、连接数或内存上限。
- **可测试性**：每个库至少要有 unit 或 regression 覆盖，复杂库需要错误路径和边界用例。
- **AOT/VM 兼容性**：若标准库 API 被 AOT 或 VM helper 依赖，需要记录 ABI/语义约束。
- **模块化构建**：确认 `XR_STDLIB_FULL` 与模块化选项下的源文件、loader、宏声明一致。

## 验证基线

分析文档本身不要求运行完整测试。后续只要修改代码，至少执行：

```bash
cmake --build build -j 8
ctest --output-on-failure
```

涉及架构边界时追加：

```bash
scripts/check_architecture.sh
```

涉及注释或 commit 内容时追加：

```bash
scripts/check_comment_rules.sh
```

涉及 parser、网络、压缩、加密或跨平台行为时，需要为对应库补充定向单测或回归测试。

## 交付策略

- **先文档后实施**：每个库先完成分析文档，再统一挑选实施任务。
- **逐库闭环**：一个库分析完成后，记录问题、优化点、测试缺口和建议顺序。
- **避免半成品扩散**：不在分析过程中顺手改实现，除非发现会阻塞构建或测试的明确错误。
- **保留统一视角**：横切问题集中记录，避免 24 个库各自修出不同风格。
