# stdlib 横切基础层分析与优化建议

## 范围

本轮只分析标准库的横切基础层，不修改实现代码。范围包括：

- `stdlib/common.h`
- `stdlib/common_io.h`
- `stdlib/common_parser.h`
- `stdlib/common_writer.h`
- `stdlib/ctxbuf.h`
- `stdlib/stdlib_cache.h`
- `src/module/xmodule.c` 中的 stdlib 注册表
- `src/module/xmodule_loaders.h`
- `CMakeLists.txt` 中的 stdlib 源文件分组

## 模块职责

横切基础层承担四类职责：

1. **native 绑定样板统一**：参数读取、字符串返回、C 函数导出。
2. **共享 I/O 与序列化工具**：文件读写、动态缓冲、格式化 writer。
3. **parser 共享约定**：错误 map schema、配置读取、默认嵌套深度。
4. **注册与生命周期 glue**：loader 声明、模块注册表、per-isolate 缓存。

这些能力是后续逐库分析的公共前提。这里的问题如果不先统一，会在每个库里重复扩散。

## 源码结构

| 文件 | 当前职责 | 备注 |
|---|---|---|
| `common.h` | native 函数导出宏、字符串参数与返回值工具 | 直接依赖 runtime/module/vm 相关内部类型 |
| `common_io.h` | `parseFile` / `writeFile` 的同步文件读写 helper | 当前会阻塞 worker |
| `common_parser.h` | parser 错误 map、配置读取、默认深度常量 | 通过 `stdlib_cache.h` 依赖 isolate 内部结构 |
| `common_writer.h` | serializer writer 包装 | 与 `ctxbuf.h` 有部分重复格式化逻辑 |
| `ctxbuf.h` | header-only 动态字节缓冲 | 缺少容量溢出保护 |
| `stdlib_cache.h` | per-isolate stdlib cache | 懒初始化无并发保护 |
| `xmodule_loaders.h` | stdlib loader 声明 | 条件编译表与 CMake、注册表重复维护 |
| `xmodule.c` | stdlib 注册表 | 注册失败未显式处理 |
| `CMakeLists.txt` | stdlib 源文件分组 | 与 loader 注册表存在漂移风险 |

## 当前行为契约

### native export

- `XRS_EXPORT` 注册普通 C 函数。
- `XRS_EXPORT_SLOW` 标记阻塞函数，将 C function class 设为 `XR_CFUNC_SLOW`。
- `XRS_EXPORT_YIELDABLE` 注册可 yield 的 C 函数。

问题是三种导出形式只在调用点体现，当前没有统一检查“可能阻塞的函数必须是 SLOW 或 yieldable”。`common_io.h` 的 `parseFile` / `writeFile` 使用同步 I/O，但调用模块目前用普通 `XRS_EXPORT` 暴露。

### parser 错误

`common_parser.h` 约定错误 map 使用 `{ type, line, row, column, message }` schema，并通过 `XrStdlibErrKeys` 缓存 key。

当前各库失败返回仍不统一。例如同步读文件失败时：

- YAML / XML 返回 `null`
- CSV 返回空数组
- TOML 返回空 map

这会让脚本层难以区分“空内容”和“读取失败”。

### per-isolate cache

`stdlib_cache.h` 通过 `isolate->stdlib_cache` 懒加载缓存，`isolate_cleanup_full()` 在释放 module registry 之前调用 `xr_stdlib_cache_free()`。

释放顺序基本合理，但初始化路径和错误路径不够严谨。

## 依赖与架构边界

### 问题 1：stdlib helper 直接接触 isolate 内部结构

`stdlib_cache.h` include `src/runtime/xisolate_internal.h` 并直接访问 `isolate->stdlib_cache`。`common_parser.h` include `stdlib_cache.h`，因此所有使用 parser helper 的标准库模块都会间接依赖 isolate 内部布局。

影响：

- stdlib 与 runtime isolate 内部结构耦合。
- 后续 isolate 生命周期、字段迁移、AOT runtime 复用都会被该 helper 阻碍。
- 头文件依赖方向不清晰，难以做模块化构建或拆分动态标准库。

建议：

- 将 cache get/free 的实现移入 `src/module` 或 runtime 可见的 `.c` 文件。
- 对 stdlib 暴露不透明 API，而不是让 header 直接访问 `XrayIsolate` 字段。
- `XrStdlibCache` 的具体结构可以留在 stdlib 私有实现中，头文件只暴露访问函数和必要的局部 cache accessor。

### 问题 2：`xmodule_loaders.h`、`xmodule.c`、CMake 三份表重复维护

当前同一份模块分组信息分散在：

- `CMakeLists.txt` 的 `STDLIB_*_SRC`
- `src/module/xmodule_loaders.h` 的 loader 声明
- `src/module/xmodule.c` 的 `stdlib_*` 注册表

已发现一个具体漂移风险：

- `xmodule.c` 在 filesystem 分组里注册 `test_yield`
- `xmodule_loaders.h` 也在 filesystem 条件下声明 `xr_load_module_test_yield`
- 但 modular CMake 的 filesystem 分组只包含 `stdlib/io/*.c` 和 `stdlib/os/*.c`

因此当 `XR_STDLIB_FULL=OFF` 且 filesystem 打开时，注册表可能引用未编译进来的 `test_yield` loader。

建议：

- `test_yield` 不应作为普通 filesystem 标准库注册。
- 用单一 X-macro 或生成表统一 source list、loader declaration、registry entry。
- modular build 至少要增加一个配置矩阵测试，覆盖 `XR_STDLIB_FULL=OFF` 的常用组合。

### 问题 3：stdlib 子目录普遍使用深层相对 include

`stdlib/*/*.c` 和 `.h` 中大量使用 `../../src/...` include。短期能编译，但会让目录层级和 include 路径强绑定。

建议：

- 依赖项目 include path，改成稳定的模块头路径。
- 将 stdlib 可用的内部 API 收敛为少量标准入口。
- 对测试模块和生产模块设置不同 include 规则。

## 内存与生命周期

### 问题 4：`xr_stdlib_cache_get()` 的 OOM 契约不一致

注释说“几乎不会返回 NULL，OOM 与 xmalloc 策略一致”，但实现是：

- `xr_malloc(sizeof(XrStdlibCache))`
- 如果失败则 `return NULL`

部分调用点检查 NULL，部分调用点直接解引用。典型路径：

- `common_parser.h` 的 `xrs_err_keys_get()` 直接使用 `&c->err_keys`
- `xml` 的 key cache 直接使用 `&c->xml_keys`
- `log_state_get()` 也依赖 cache 可用

建议：

- 要么严格 abort，并让注释与实现一致。
- 要么所有调用点统一处理 NULL，并把错误返回暴露给 native binding。
- 对 parser error path，推荐采用 fail-fast，避免在错误上报路径再次产生未定义行为。

### 问题 5：per-isolate cache 懒初始化无并发保护

`xr_stdlib_cache_get()` 对 `isolate->stdlib_cache` 做无锁检查与赋值。如果同一个 isolate 的多个 worker 同时首次调用 stdlib parser/log/xml/io，就可能出现：

- 重复分配
- 后写覆盖先写
- 某些 cache 字段初始化丢失
- log state 初始化与析构函数设置竞态

建议：

- 在 isolate 初始化时创建 stdlib cache，避免运行时懒初始化。
- 或使用 isolate 级 mutex / once primitive 保护初始化。
- 对 cache 内部子结构也需要明确线程安全策略。

### 问题 6：`ctxbuf` 容量增长缺少溢出防护

`xr_ctxbuf_reserve()` 用倍增策略增长容量，但没有检查：

- `b->len + extra + 1` 溢出
- `ncap *= 2` 溢出

对于 parser / serializer，输入可能来自用户文件或网络。容量溢出会导致分配小 buffer 后越界写。

建议：

- 增加 `SIZE_MAX` 上界检查。
- 统一返回错误或 fail-fast，不允许静默截断。
- writer 层保留最大输出大小选项，防止 stringify 巨型对象耗尽内存。

### 问题 7：`ctxbuf.h` 使用 `abort()` 但没有显式 include 标准声明

`ctxbuf.h` 调用 `abort()`，但当前 include 列表里没有 `<stdlib.h>`。这依赖其他头间接包含，属于 header 自包含性问题。

建议：

- header 自己 include 所需标准头。
- 后续增加“每个 stdlib header 可单独 include 编译”的单测或脚本检查。

## 并发、阻塞与协程语义

### 问题 8：`common_io.h` 是同步 I/O，但调用模块普通导出

`xrs_file_read_all_sync()` / `xrs_file_write_all_sync()` 使用 `fopen`、`fread`、`fwrite`、`fclose`。调用者包括：

- `csv.parseFile` / `csv.writeFile`
- `toml.parseFile` / `toml.writeFile`
- `yaml.parseFile` / `yaml.writeFile`
- `xml.parseFile` / `xml.writeFile`

这些 binding 当前使用普通 `XRS_EXPORT`，会在 worker 上同步阻塞。

建议：

- 短期：将同步文件 API 标记为 `XRS_EXPORT_SLOW`，至少让调度器能把阻塞调用交给专门线程。
- 中期：增加统一的 yieldable file job helper，把 parseFile/writeFile 改为可 yield native binding。
- 长期：让 parser 本体只处理内存 buffer，文件 I/O 由 `io` 模块或 runtime async I/O 层负责。

### 问题 9：export 宏没有参数签名或阻塞属性元数据

当前 `XRS_EXPORT` 只注册函数指针，缺少：

- 参数个数
- 参数类型
- 返回类型
- 是否阻塞
- 是否需要 isolate / coroutine / runtime capability

这导致文档、类型声明、运行时行为分散在注释和手写代码里。

建议：

- 后续统一 native binding descriptor。
- loader 从 descriptor 自动注册，同时生成类型声明与测试表。
- 至少先把 blocking/yieldable 属性集中成显式表，避免漏标。

## 安全与鲁棒性

### 问题 10：文件读取没有大小上限

`xrs_file_read_all_sync()` 会根据 `ftell` 的大小一次性分配 `size + 1`。对于大文件或特殊文件，可能造成：

- 内存耗尽
- 长时间阻塞 worker
- parser 在没有输入上限时继续递归或分配

建议：

- 增加默认最大文件大小。
- 允许调用者传入 limit。
- 对 parseFile 默认返回结构化错误，而不是返回空容器或 null。

### 问题 11：配置读取会静默截断 fixed string

`xrs_cfg_get_fixed_str()` 当输入超过 `dst_cap - 1` 时直接截断。这对 delimiter、encoding、schema name 之类配置可能可以接受，但对路径、URL、tag name 等配置会隐藏错误。

建议：

- 返回值区分“成功完整复制”和“发生截断”。
- 调用方决定截断是错误、警告还是合法行为。

## 注释与代码规范风险

本轮只记录，不修改 C 注释。当前横切层和相邻模块中存在源码注释规则风险：

- `ctxbuf.h` 的注释引用了规则文档路径。
- `common_io.h` 中出现临时计划编号式说法。
- `net/tls.c`、`net/tls.h` 中出现类似临时实施编号的标题。

建议在后续真正修改 stdlib C 代码前，先跑注释检查脚本，并把这些问题作为独立清理项处理。

## 测试覆盖缺口

当前单测已有部分覆盖：

- `tests/unit/stdlib/`
- `tests/unit/http/`
- `tests/unit/encoding/`
- `tests/unit/regex/`

横切基础层缺少以下测试：

1. **modular build matrix**：至少覆盖 core-only、filesystem、network、data、full。
2. **header self-containment**：每个 stdlib public/internal header 单独 include 编译。
3. **ctxbuf overflow/large append**：模拟接近容量边界的 reserve 行为。
4. **common_io error contract**：不存在文件、目录路径、权限失败、超大文件。
5. **cache concurrency**：多线程或多 worker 同时首次访问 stdlib cache。
6. **blocking export audit**：静态检查所有使用同步 I/O 的 binding 是否是 SLOW/yieldable。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 严重 | modular build 下 `test_yield` 注册表与 CMake 源列表可能不一致 | `XR_STDLIB_FULL=OFF` 时潜在链接失败 | 将 test/support 模块移出生产 stdlib 或纳入显式 test 选项 |
| 高 | `xr_stdlib_cache_get()` 懒初始化无并发保护 | 多 worker 首次访问可能竞态 | isolate 初始化时创建或加 once/mutex |
| 高 | 同步 parseFile/writeFile 用普通 export | 文件 I/O 阻塞 worker | 短期标记 SLOW，中期 yieldable I/O |
| 高 | stdlib cache header 直接访问 isolate 内部字段 | 架构耦合，阻碍 runtime/AOT 边界收敛 | 改为不透明 API 与 `.c` 实现 |
| 高 | `ctxbuf` 容量增长无溢出保护 | 极端输入可能越界写 | 增加 size 上界检查 |
| 中 | 文件读取无大小上限 | 大文件导致 OOM 或长时间阻塞 | 增加默认 limit 和调用方配置 |
| 中 | parser 文件失败返回不统一 | 脚本层无法可靠处理错误 | 统一结构化错误返回 |
| 中 | 三份模块表重复维护 | 新增/删除模块容易漂移 | X-macro 或生成表 |
| 中 | header 自包含性不足 | include 顺序脆弱 | 单独 include 编译测试 |
| 低 | writer printf 逻辑与 ctxbuf 重复 | 后续行为可能漂移 | `common_writer.h` 直接调用 `xr_ctxbuf_appendf()` |

## 后续实施建议

建议先不直接进入大型库重构，而是按以下顺序处理基础问题：

1. **标准库注册表单一真相源**：统一 CMake、loader 声明、注册表。
2. **cache 生命周期收敛**：消除 `stdlib_cache.h` 对 isolate 内部布局的直接访问，并保证并发初始化安全。
3. **阻塞 API 标注**：先把同步文件 I/O binding 标记为 SLOW，再设计 yieldable 文件 I/O。
4. **buffer 安全基线**：为 `ctxbuf` 增加容量溢出防护和输出大小限制。
5. **错误返回统一**：parser 文件 API 返回统一错误对象或 strict/non-strict 双模式。
6. **测试矩阵补齐**：modular build、header self-containment、cache concurrency、blocking export audit。

完成这些横切基础项之后，再逐个进入小型 core 库分析，优先顺序建议仍按 `036-stdlib-refactor-analysis.md` 的路线执行。
