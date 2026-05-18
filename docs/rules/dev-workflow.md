# 开发工作流

## 文件存放

| 类型 | 目录 |
|------|------|
| 文档 | `docs/` |
| 单元测试 | `tests/unit/` |
| 回归测试 | `tests/regression/` |
| 临时测试 | `tests/tmp/` |

## 调试

### 字节码

```bash
./build/xray --dump-bytecode <file.xr>
```

### LLDB（**禁止直接调用 lldb**，用脚本）

```bash
scripts/lldb_debug.sh --bt tests/some_test.xr
XRAY_BUILD=build scripts/lldb_debug.sh --bt test.xr
```

### 内存检查

使用 ASan 构建: `cmake -B build-asan -DENABLE_ASAN=ON -DBUILD_TESTS=ON`

## 测试

```bash
# 快速单元测试
cd build && ctest --output-on-failure

# 完整回归
scripts/run_regression_tests.sh

# 指定测试
cd build && ctest -R test_name --output-on-failure

# 架构检查
scripts/check_architecture.sh

# 平台抽象层检查（R1/R3 hard, R2 advisory）
scripts/check_platform_layering.sh
```

## 标准库类型声明同步

```bash
python3 scripts/gen_stdlib_types.py
```

## CI

| 操作 | 触发 |
|------|------|
| `git push` / PR | 编译 + 回归测试 |
| `git tag v*` | 编译 + 测试 + 打包 + 发布 |

## 平台

| 平台 | 网络轮询 |
|------|----------|
| macOS (Intel + ARM) | kqueue |
| Linux | epoll |
| Windows | IOCP |

## Windows / Parallels Desktop VM 测试

Windows 平台的所有构建和测试一律走 `scripts/win_pd_test.sh`（基于 `prlctl exec`），不允许在 IDE/CI 里直接 `ssh xray-win 'cmd /c ...'` 跑长命令。详细行为/退出码/故障排查见 `.windsurf/rules/windows-testing.md`。

### 常用命令

```bash
# 全量：同步 + 构建 + ctest
scripts/win_pd_test.sh

# 单 ctest（跳过构建）
scripts/win_pd_test.sh --no-build -R test_vm_api -V

# 单 .xr 回归
scripts/win_pd_test.sh --no-build --xray-test tests/regression/11_coroutine/1131_linked_go_expr.xr

# 全量回归（VM 内并行 + 单测超时）
scripts/win_pd_test.sh --no-build --regression

# 看 VM 当前是否还有 cmake/ctest/ninja/cl/link/xray 在跑
scripts/win_pd_test.sh --status

# 清理残留构建/测试进程（解锁 build.ninja 锁定）
scripts/win_pd_test.sh --kill
```

### 关键设计

- **wall timeout**：构建默认 `1800s`（Release 模式 `xvm.c` 单文件在 4 vCPU VM 上需要 ~15min），测试默认 `360s`，超时返回 `124` 并自动清理残留进程。
- **heartbeat**：默认每 `20s` 报告一次进展；静默 `90s` 触发自动 VM 进程快照（看 CPU 是否还在涨）。
- **慢 vs 卡住**：日志字节数还在涨 = 慢；CPU 全停 = 卡住。两者都会等到 wall timeout。
- **构建前清理**：默认开启，避免被取消的旧 `ninja/cl` 锁住 `build.ninja`（`failed recompaction: Permission denied`）。
- **失败诊断**：测试失败时自动 dump VM 上 `LastTest.log` 末尾若干行；构建失败时打印进程快照。

### 退出码

| code | 含义 |
|------|------|
| `0` | 通过 |
| `1` | 测试失败（ctest / `xray test`） |
| `2` | 参数错 / VM 不存在 / 不在 running |
| `124` | wall timeout 命中（脚本主动 kill） |
| `127` | 缺 `prlctl` |

### 旧 SSH 脚本

`scripts/win_test.sh`、`scripts/win_build.sh` 仍然存在，但已变成 `win_pd_test.sh` 的瘦壳，保持兼容老链接；新脚本/CI/文档全部走 `win_pd_test.sh`。
