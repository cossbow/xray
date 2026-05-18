# stdlib/path 分析与优化建议

## 模块职责

`stdlib/path` 提供纯字符串路径处理能力：

- `join`
- `dirname`
- `basename`
- `extname`
- `normalize`
- `isAbsolute`
- `resolve`
- `relative`
- `parse`
- `format`
- 常量 `sep` / `delimiter`

该模块应只处理路径字符串语义，不应执行文件系统访问，除非 API 明确要求读取当前工作目录。当前只有 `resolve()` 需要 `getcwd`。

## 源码结构

| 文件 | 职责 |
|---|---|
| `stdlib/path/path.c` | native binding、路径算法、loader |
| `stdlib/path/path.h` | loader 声明与模块说明 |
| `stdlib/ctxbuf.h` | `resolve/format` 使用的动态 buffer |
| `src/os/os_fs.h` | `resolve()` 获取当前工作目录 |
| `tests/regression/10_stdlib/1170_path_basic.xr` | path 基础回归测试 |
| `src/base/xfileio.c` | base 层也有 dirname/join/basename 的简化实现 |

## 当前行为契约

### POSIX 默认语义

现有回归测试主要按 POSIX 路径行为验证：

- `path.join("foo", "bar") == "foo/bar"`
- `path.normalize("/usr//local///bin") == "/usr/local/bin"`
- `path.relative("/usr/local/bin", "/usr/local/lib") == "../lib"`

### Windows 支持

代码里有 `XR_OS_WINDOWS` 分支，定义了：

- `PATH_SEP = '\\'`
- `PATH_DELIMITER = ';'`
- `IS_SEP(c)` 同时接受 `/` 和 `\\`

但实际算法并没有完整实现 Windows drive / UNC 语义。

## 依赖与架构边界

### 问题 1：`path.c` include VM internal 头

`path.c` include `src/vm/xvm_internal.h`，但当前实现没有直接使用 VM internal 类型。

影响：

- 标准库 path 模块不必要依赖 VM internal。
- AOT runtime、动态标准库或模块边界收敛会受阻。

建议：

- 删除 `xvm_internal.h` include。
- 只保留 map/string/module/binding 所需的稳定头。

### 问题 2：`path.h` 依赖过重

`path.h` 只需要声明 loader，却 include：

- `src/module/xmodule.h`
- `src/vm/xvm.h`
- `src/runtime/object/xstring.h`

建议：

- 改为 forward declare `XrayIsolate` / `XrModule`。
- 或统一由 `xmodule_loaders.h` 提供 loader 声明，模块 header 只保留内部需要的声明。

### 问题 3：loader 缺少统一可见性修饰

`xr_load_module_path()` 定义没有 `XR_FUNC`，与 `xmodule_loaders.h` 中声明风格不一致。

建议纳入 stdlib loader 统一修复。

## 内存与生命周期

### 问题 4：`path_normalize()` 泄漏临时 result buffer

`path_normalize()` 分配：

```c
char *result = (char *) xr_malloc(len + 2);
```

随后：

```c
XrValue ret = xrs_string_value_c(X, result);
xr_free(seg_buf);
return ret;
```

`result` 在 intern/copy 后没有释放。影响范围：

- 直接调用 `path.normalize()` 每次泄漏一次。
- `path.resolve()` 调用 `path_normalize()`，也间接泄漏。
- `path.relative()` 对 from/to 分别调用 normalize，会泄漏两次。

建议：

- 在返回前释放 `result`。
- 增加回归或 ASAN 测试覆盖 normalize/resolve/relative 高频调用。

### 问题 5：多个长度计算缺少 overflow 检查

`path_join()` 通过累加每个 part 的 `strlen(s) + 1` 计算 total length，没有检查 `size_t` 溢出。

`path_relative()` 计算 `result_size = up_count * 3 + to_rest_len + 2`，也没有显式 overflow 检查。

建议：

- 使用 `XrCtxBuf` 替代手工预估长度。
- 或增加统一 size overflow helper。

### 问题 6：`path_parse()` 每次重复 intern map keys

`root/dir/base/name/ext` 每次 parse 都通过 `xrs_string_value_c()` 生成 key。成本不大，但这是高频纯函数，可纳入 per-isolate key cache。

建议：

- 复用 `stdlib_cache` 增加 path key cache。
- 或等横切 cache API 收敛后统一处理。

## Windows 与跨平台语义

### 问题 7：Windows absolute 判断不准确

`path_isAbsolute()` 当前将 `C:foo` 视为 absolute，因为只检查第二个字符是 `:`。但 Windows 语义里 `C:foo` 是 drive-relative，不是绝对路径；`C:\foo` 才是绝对路径。

建议：

- drive absolute 必须匹配 `^[A-Za-z]:[\\/]`。
- `\\server\share` UNC 路径需要单独处理 root。

### 问题 8：`normalize()` 不支持 Windows drive / UNC root

`path_normalize()` 的 absolute 判断只看 `IS_SEP(path[0])`，因此：

- `C:\foo\..\bar` 不会被识别为 drive-root absolute。
- UNC 路径 root 语义会丢失。
- 输出 separator 硬编码为 `/`，没有使用 `PATH_SEP`。

这与文件头“cross-platform path operations”的定位不一致。

建议：

- 把 path 解析拆成 root + segments。
- root 支持 POSIX `/`、Windows drive root、UNC root。
- 输出统一使用平台 separator，或明确提供 posix/win32 双命名空间。

### 问题 9：`resolve()` 的 absolute 判断与 `isAbsolute()` 不一致

`resolve()` 只用 `IS_SEP(part[0])` 判断 absolute。Windows 下 drive absolute path 不会重置累计 buffer。

建议：

- 所有 API 共用同一个 `path_is_absolute_raw()` helper。
- `join/resolve/normalize/relative/parse` 共享 root 解析逻辑。

### 问题 10：`parse()` / `format()` 忽略 Windows root 和 format root 字段

`path_parse()` 的 root 只判断 `path[0] == '/'`。

`path_format()` 当前只关注 `dir/base/name/ext`，基本忽略 `root` 字段。

影响：

- `path.format(path.parse(p))` 不是可靠 round-trip。
- Windows path 组件语义不完整。

建议：

- 定义 parse map schema：`root/dir/base/name/ext` 的单位和优先级。
- format 按 `dir > root`、`base > name+ext` 的明确规则实现。
- 增加 round-trip 测试。

## POSIX 边界语义

### 问题 11：root 和特殊 dotfile 边界未充分测试

当前测试未覆盖：

- `basename("/")`
- `basename("////")`
- `dirname("////")`
- `extname("..")`
- `extname("...")`
- `extname("file.")`
- `normalize("../../a")`
- `relative("/", "/a")`
- `relative("/a", "/")`

其中 `extname("..")` 当前很可能返回 `"."`，这通常不是用户期望。

建议：

- 明确采用 Node.js path、Python pathlib，还是自定义 Xray 语义。
- 按语义补全测试矩阵。

### 问题 12：`join()` 不做 normalize

`join()` 只拼接 segment 和处理 absolute reset，不解析 `.` / `..`。

这可以是合法设计，但需要明确：

- `join()` 是否应该等价于 normalize(joined path)
- 还是只做轻量拼接

当前测试没有覆盖 `path.join("a", "..", "b")`。

## API 与类型声明

### 问题 13：`parse` / `format` 类型声明与实现不一致

`path_parse` 实际返回 `XrMap`，但 builtin signature 是 `(path: string): Json`。

`path_format` 实现要求 `XR_IS_MAP(args[0])`，但 signature 是 `(obj: Json): string`。

建议：

- 统一 Map/Json 在标准库 API 中的使用。
- 如果 parse 返回 Map，signature 应表达 Map 或对象类型。
- 如果要返回 Json，需要构造 XrJson 而不是 XrMap。

### 问题 14：非字符串参数被静默忽略或返回默认值

例如：

- `join` 忽略非字符串 segment。
- `dirname/basename/extname/normalize` 对非字符串返回默认值。
- `parse` 对非字符串返回空 map。

建议：

- 标准库统一 type error policy。
- 至少在文档中说明非字符串参数行为。

## 与 base 层重复

`src/base/xfileio.c` 已有简化版：

- `xr_path_dirname`
- `xr_path_join`
- `xr_path_basename`

这些是 base 层内部工具，`stdlib/path` 又有独立实现。短期可以接受，但应避免语义漂移。

建议：

- base 层只保留构建/内部用的简单 POSIX helper。
- stdlib/path 保留用户可见完整语义。
- 如果共享，必须保证 base 不依赖 runtime/module。

## 测试覆盖

现有 `1170_path_basic.xr` 覆盖：

- 常量
- join 基础
- dirname/basename/extname 基础
- normalize 基础
- isAbsolute POSIX
- resolve 基础
- relative 基础和 prefix mismatch
- parse/format 基础

缺口：

1. normalize 内存泄漏没有 ASAN/循环测试。
2. Windows drive / UNC 语义没有测试。
3. root、dotfile、双点路径边界不足。
4. parse/format round-trip 不足。
5. resolve 的 cwd 超长或失败路径未覆盖。
6. 非字符串参数行为未覆盖。
7. join 是否 normalize 未定义。
8. path header self-contained 编译未覆盖。

## 问题清单

| 严重度 | 问题 | 影响 | 建议 |
|---|---|---|---|
| 严重 | `path_normalize()` 泄漏 `result` | normalize/resolve/relative 高频调用泄漏系统堆内存 | intern 后释放 result，补 ASAN 循环测试 |
| 高 | Windows drive/UNC 语义不完整 | 跨平台承诺不成立 | 引入 root parser，统一 isAbsolute/normalize/resolve |
| 高 | `path.c` include VM internal | 标准库与 VM 内部耦合 | 删除不必要 include |
| 高 | `path.h` include 过重 | header 边界污染 | forward declare 或统一 loader 声明 |
| 中 | `parse/format` signature 与实现不一致 | 类型系统、文档、运行时漂移 | 统一 Map/Json 契约 |
| 中 | 长度计算无 overflow 检查 | 极端输入可能越界或分配错误 | 使用 ctxbuf 或 size helper |
| 中 | `extname` 特殊 dot 边界未定义 | 用户可见行为不稳定 | 明确语义并补测试 |
| 中 | 非字符串参数静默忽略 | 错误隐藏 | 统一 type error policy |
| 低 | map key 重复 intern | 高频 parse 成本偏高 | path key cache |
| 低 | 与 base path helper 语义可能漂移 | 维护成本增加 | 明确 base/helper 与 stdlib 的边界 |

## 后续实施建议

建议先做低风险 correctness 修复，再设计跨平台语义：

1. 修复 `path_normalize()` 临时 buffer 泄漏。
2. 清理 `path.c/path.h` include 和 loader 修饰。
3. 补 POSIX 边界测试：root、dotfile、relative root、normalize dotdot。
4. 明确 Map/Json 契约并修正 builtin signature。
5. 设计统一 root parser，覆盖 Windows drive / UNC。
6. 决定 `join()` 是否自动 normalize。
7. 增加 ASAN 循环测试和 header self-containment 测试。

完成后验证：

```bash
cmake --build build -j 8
ctest --output-on-failure
scripts/run_regression_tests.sh
```
