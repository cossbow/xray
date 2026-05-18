# Xray 包管理与 Registry 实施计划

> 日期：2026-04-24
> 状态：Draft
> 涉及仓库：`xray`（客户端）、`xray-server`（官方 Registry）、`xray-sqlite`（示例三方包）

---

## 1. 现状评估

### 1.1 客户端 (`xray/src/module`) — 基础设施 ~70%

| 模块 | 文件 | 行数 | 状态 |
|------|------|------|------|
| semver 解析 | `xsemver.[ch]` | ~12k+2k | ✅ 完成：parse/compare/constraint match (`^`,`~`,`>=` 等) |
| lockfile | `xlockfile.[ch]` | ~14k+3k | ✅ 完成：`xray.lock` 序列化、SHA256 校验 |
| 依赖解析器 | `xresolver.[ch]` | ~17k+3k | ✅ 完成：`XrDepGraph` 有向图、环检测、拓扑排序 |
| 项目配置 | `xproject.[ch]` | ~10k+2k | ✅ 完成：`xray.toml` 解析 (`[project]`/`[package]`、`[dependencies]`、path 依赖) |
| HTTP 客户端 | `xpkg_client.[ch]` | ~23k+3k | ✅ 完成：搜索/下载/安装/发布/认证 |
| CLI 命令 | `xcmd_pkg.c` | ~580 | ⚠️ 部分：`init`/`add`(不写 toml)/`tree`/`login`/`publish` 可用；`install` 骨架未接通 resolver；`remove`/`update` stub |

**关键缺口**：
- `PKG_REGISTRY_URL` 在 `xpkg_client.h:29` 硬编码为 `https://pkg.xray-lang.org`，无法配置私有仓库
- `cmd_pkg_add` 成功后不会自动写回 `xray.toml`
- `cmd_pkg_install` 打印 "waiting for registry deployment" 后直接退出，未接通 `xr_resolve_dependencies()`
- Token 只保存到 `~/.xray/credentials` 单文件，不支持多 registry 分别认证
- `xpkg_client.c` 内的 JSON 解析是手写 `strstr` 实现，健壮性差

### 1.2 服务端 (`xray-server`) — 框架 ~60%

| 层 | 文件 | 状态 |
|----|------|------|
| 路由 | `cmd/server/main.go` | ✅ 完成：20+ 路由，公开/认证分组 |
| 数据模型 | `internal/model/model.go` | ✅ 完成：`User`/`Package`/`Version`/`DownloadStat`，PostgreSQL |
| 存储 | `internal/storage/storage.go` | ⚠️ `LocalStorage` 可用；`MinIOStorage` 全是 TODO |
| Handler | `internal/handler/handler.go` | ⚠️ 671 行，CRUD 基本完成但有缺陷 |
| 认证 | `internal/middleware/middleware.go` | ⚠️ 只检查 token 存在性，不验签 |
| 配置 | `internal/config/config.go` | ✅ 完成：viper 加载、环境变量覆盖 |

**Handler 具体缺陷**：
- `GitHubCallback`: 只返回 `code`，不兑换 access_token、不创建/查询 User
- `PublishPackage`: 不解析上传 tarball 中的 `xray.toml`，不计算 `tarball_sha256`，`dependencies` 字段为空
- `RequireAuth`: token 直接从 header 取出传递给 handler，handler 中 `db.Where("api_token = ?", ...)` 每次都查库，无缓存
- `Download` 路由 `/download/:owner/:name/:version` 与客户端 `xpkg_client.c:487` 的 URL 格式 `/api/packages/:owner/:name/:version/download` **不匹配**
- `Search` 的 `= ANY(keywords)` 不做 lower 匹配，keyword 大小写敏感

### 1.3 示例三方包 (`xray-sqlite`)

- `xray.toml` 使用 `[package]` + `native = true` 标识原生共享库包
- CMake 构建 `.dylib/.so`，用 `-undefined dynamic_lookup` 解析宿主符号
- 导出标准 `xr_module_load(XrModuleRegistry *)` 入口
- 无 `[dependencies]` section（纯原生包，不依赖其他 xray 包）
- 可作为包管理闭环测试的金丝雀

---

## 2. 设计目标

1. **官方源 + 私有源并存**：用户可配置多个 registry，按包名或 scope 路由
2. **私有源零门槛自建**：最简可以是一个静态文件目录（Nginx/S3/Git），无需数据库
3. **CLI 端到端闭环**：`pkg init` → `pkg add` → `pkg install` → `pkg publish` 全链路可用
4. **安全性**：SHA256 完整性校验、token 分仓库存储、tarball 解析在服务端完成
5. **渐进式**：每个 Phase 独立可用，不阻塞其他 Phase

---

## 3. 协议设计

### 3.1 Registry API 协议 (v1)

两种 Registry 模式通过 `GET /registry.json` 区分：

#### 动态 Registry（官方 xray-server）

```
GET  /registry.json → {"api": 1, "name": "pkg.xray-lang.org"}
GET  /api/packages/{owner}/{name}                → 包详情 + latest version
GET  /api/packages/{owner}/{name}/versions       → 版本列表
GET  /api/packages/{owner}/{name}/{ver}/download → 下载 tarball
POST /api/packages                               → 发布（multipart: tarball + metadata）
GET  /api/search?q={query}&page=1&size=20        → 搜索
POST /api/auth/token                             → 换取/刷新 API token
```

#### 静态 Registry（自建最简模式）

```
GET  /registry.json → {"static": 1, "name": "internal"}
GET  /{owner}/{name}/metadata.json → {"versions":["1.0.0","1.1.0"],"latest":"1.1.0","description":"..."}
GET  /{owner}/{name}/{version}.tar.gz → 下载 tarball
```

发布流程（静态源）：
```bash
# 手动或 CI 脚本：
tar czf mylib-1.0.0.tar.gz -C mylib .
cp mylib-1.0.0.tar.gz /registry/alice/mylib/1.0.0.tar.gz
# 更新 metadata.json（脚本自动 or 手动编辑）
```

### 3.2 xray.toml 扩展

```toml
# === 应用项目 ===
[project]
name = "my-app"
main = "src/main.xr"

# === 或者：可发布包 ===
[package]
name = "alice/mylib"
version = "1.0.0"
description = "My library"
license = "MIT"
# native = true              # 含 C 共享库的原生包

# === Registry 配置 ===
[registries]
# default 不写则使用内置官方源 https://pkg.xray-lang.org
# default = "https://pkg.xray-lang.org"
company = "https://pkg.internal.company.com"

# === 依赖声明 ===
[dependencies]
# 简写 → 使用 default registry
xray/redis = "^1.0.0"

# 指定 registry
company/auth-sdk = { version = "^2.0.0", registry = "company" }

# 本地路径依赖（开发阶段，不上传 registry）
utils = { path = "../utils" }

# Git 依赖（未来可选，Phase 3 考虑）
# bob/helper = { git = "https://github.com/bob/helper.git", tag = "v1.0.0" }
```

### 3.3 全局用户配置 `~/.xray/config.toml`

```toml
[registries]
default = "https://pkg.xray-lang.org"
company = "https://pkg.internal.company.com"

[tokens]
# 按 registry 名称存 token（替代当前的 ~/.xray/credentials 单 token）
default = "xr_xxxxxxxxxxxxxxxx"
company = "xr_yyyyyyyyyyyyyyyy"
```

加载优先级：项目 `xray.toml [registries]` > `~/.xray/config.toml [registries]` > 内置默认值。

---

## 4. 分阶段实施

### Phase 0：跑通最小闭环（官方源端到端）

**目标**：用 xray-sqlite 作为金丝雀，跑通 `publish` → `install` → `import` 全流程。

#### 0.1 服务端：修复 Download 路由不匹配

**问题**：客户端请求 `/api/packages/:owner/:name/:version/download`，服务端注册的路由是 `/api/download/:owner/:name/:version`。

**文件**：`xray-server/cmd/server/main.go`
**改动**：增加一条兼容路由：
```go
api.GET("/packages/:owner/:name/:version/download", h.Download)
```

#### 0.2 服务端：补完 GitHub OAuth 回调

**文件**：`xray-server/internal/handler/handler.go` — `GitHubCallback()`
**改动**：
1. 用 `code` 换取 GitHub access_token（POST `https://github.com/login/oauth/access_token`）
2. 用 access_token 获取用户信息（GET `https://api.github.com/user`）
3. 在 `users` 表中 upsert（by `github_id`），生成 API token
4. 返回 API token 给 CLI（或重定向到前端带 token）

**新增依赖**：无（用标准库 `net/http` 发请求即可）

#### 0.3 服务端：发布时解析 tarball + 计算 SHA256

**文件**：`xray-server/internal/handler/handler.go` — `PublishPackage()`
**改动**：
1. 保存 tarball 到临时文件
2. 计算 SHA256，写入 `version.TarballSHA256`
3. 解压 tarball，读取其中的 `xray.toml`：
   - 自动填充 `name`/`version`/`description`/`license`/`keywords`
   - 解析 `[dependencies]` 写入 `version.Dependencies` (JSONB)
4. 删除临时文件，tarball 已由 storage 保存

#### 0.4 客户端：`cmd_pkg_install` 接通 resolver

**文件**：`xray/src/app/cli/xcmd_pkg.c` — `cmd_pkg_install()`
**改动**：
```
1. 遍历 project->dependencies hashmap
2. 对每个依赖，调用 xr_depgraph_add_root(graph, name, version_constraint)
3. 调用 xr_resolve_dependencies(graph, registry_fetch_info, lockfile, NULL)
4. 遍历 resolve_result，对每个包调用 xr_pkg_client_install()
5. 更新 lockfile，保存 xray.lock
```

需要实现的回调：
```c
static XrPackageInfo* registry_fetch_info(const char *name, void *user_data) {
    // 拆分 owner/name，调用 xr_pkg_client_get_info()
}
```

#### 0.5 客户端：`cmd_pkg_add` 写回 xray.toml

**文件**：`xray/src/app/cli/xcmd_pkg.c` — `cmd_pkg_add()`
**改动**：安装成功后，读取 `xray.toml` 内容，在 `[dependencies]` section 末尾追加 `owner/name = "version"`，写回文件。

**实现策略**（简单可靠）：
1. 逐行读取 `xray.toml`
2. 找到 `[dependencies]` section
3. 在该 section 最后一行（下一个 `[` 或 EOF 之前）插入新行
4. 写回文件

#### 0.6 验证清单

```bash
# 1. 启动服务端
cd xray-server && make run

# 2. 注册/登录
xray pkg login                        # 获取 token

# 3. 发布 xray-sqlite
cd xray-sqlite && xray pkg publish    # tarball → registry

# 4. 搜索
xray pkg search sqlite                # 应返回 xray/sqlite

# 5. 新项目安装
mkdir /tmp/test-app && cd /tmp/test-app
xray pkg init
xray pkg add xray/sqlite@^1.0.0      # 下载 + 写 xray.toml
xray pkg install                      # resolve + install from lockfile
xray pkg tree                         # 显示依赖树

# 6. 使用
echo 'import sqlite from "xray/sqlite"' > src/main.xr
echo 'print(sqlite)' >> src/main.xr
xray src/main.xr                      # 应能加载模块
```

---

### Phase 1：私有 Registry 支持

**目标**：`xray.toml` 可声明多个 registry，CLI 按配置路由请求。

#### 1.1 客户端：Registry 配置加载

**新文件**：`xray/src/module/xpkg_config.[ch]`
**职责**：

```c
typedef struct XrPkgRegistry {
    char *name;          // "default", "company", ...
    char *url;           // "https://..."
    char *token;         // nullable
    bool is_static;      // auto-detected via /registry.json
} XrPkgRegistry;

typedef struct XrPkgConfig {
    XrPkgRegistry *registries;
    int registry_count;
    char *default_registry;  // name, defaults to "default"
} XrPkgConfig;

// Load from xray.toml [registries] + ~/.xray/config.toml
XR_FUNC XrPkgConfig* xr_pkg_config_load(const char *project_root);
XR_FUNC void xr_pkg_config_free(XrPkgConfig *config);

// Resolve which registry to use for a given dependency
XR_FUNC const XrPkgRegistry* xr_pkg_config_resolve(
    const XrPkgConfig *config,
    const char *dep_name,       // "xray/redis"
    const char *dep_registry    // from xray.toml, nullable → use default
);
```

**加载优先级**：
1. 项目 `xray.toml` 的 `[registries]` → URL
2. `~/.xray/config.toml` 的 `[registries]` → URL 补缺
3. `~/.xray/config.toml` 的 `[tokens]` → token
4. 内置默认值 `default = "https://pkg.xray-lang.org"`

#### 1.2 客户端：`xpkg_client` 支持运行时 URL

**文件**：`xray/src/module/xpkg_client.h`
**改动**：
```c
// 删除: #define PKG_REGISTRY_URL "https://pkg.xray-lang.org"
// 改为: 通过 xr_pkg_client_set_config() 设置 registry_url
// tls_config.registry_url 默认值改为 "https://pkg.xray-lang.org"（运行时可覆盖）
```

所有 `cmd_pkg_*` 函数启动时，从 `XrPkgConfig` 获取当前 registry URL 和 token，调用 `xr_pkg_client_set_config()`。

#### 1.3 客户端：静态 Registry 支持

**文件**：`xray/src/module/xpkg_client.c`
**改动**：在 `xr_pkg_client_get_info()` / `xr_pkg_client_install()` 中增加分支：

```c
if (is_static_registry(tls_config.registry_url)) {
    // GET /{owner}/{name}/metadata.json
    // Parse JSON → XrPackageInfo
} else {
    // 现有动态 API 路径
}
```

静态 registry 检测：首次访问时 `GET /registry.json`，缓存结果到 `tls_config`。

#### 1.4 客户端：多 registry token 管理

**文件**：`xray/src/module/xpkg_client.c` — `xr_pkg_client_save_token()` / `xr_pkg_client_load_token()`
**改动**：
- 当前：写 `~/.xray/credentials` 单 token 文件
- 改为：写 `~/.xray/config.toml` 的 `[tokens]` section，key 为 registry name

**文件**：`xray/src/app/cli/xcmd_pkg.c` — `cmd_pkg_login()`
**改动**：增加 `--registry <name>` 参数支持。

#### 1.5 `xproject.c` 解析 `[registries]` section

**文件**：`xray/src/module/xproject.[ch]`
**改动**：在 `XrProject` 中增加 `XrHashMap *registries`（name → url），`xr_project_load()` 中解析 `[registries]` section。

`XrDependency` 增加 `char *registry` 字段：
```c
typedef struct XrDependency {
    char *name;
    char *version;
    char *path;
    char *registry;   // NEW: nullable, defaults to "default"
    bool is_local;
} XrDependency;
```

#### 1.6 验证清单

```bash
# 1. 创建静态 registry 目录
mkdir -p /tmp/my-registry/alice/mylib
echo '{"static":1,"name":"test"}' > /tmp/my-registry/registry.json
echo '{"versions":["1.0.0"],"latest":"1.0.0"}' > /tmp/my-registry/alice/mylib/metadata.json
cp mylib-1.0.0.tar.gz /tmp/my-registry/alice/mylib/1.0.0.tar.gz

# 2. 启动简易 HTTP 服务
python3 -m http.server 9000 -d /tmp/my-registry &

# 3. 项目配置
cat > xray.toml << 'EOF'
[project]
name = "test-app"
main = "src/main.xr"

[registries]
local = "http://localhost:9000"

[dependencies]
alice/mylib = { version = "^1.0.0", registry = "local" }
EOF

# 4. 安装
xray pkg install   # 应从静态 registry 下载
```

---

### Phase 2：服务端加固

**目标**：xray-server 生产级可用，支持自建部署。

#### 2.1 SQLite 后端支持（降低自建门槛）

**文件**：`xray-server/internal/database/database.go`
**改动**：增加 SQLite driver 分支：

```go
import (
    "gorm.io/driver/sqlite"
    "gorm.io/driver/postgres"
)

func Connect(cfg config.DatabaseConfig) (*gorm.DB, error) {
    switch cfg.Driver {
    case "sqlite":
        return gorm.Open(sqlite.Open(cfg.SQLitePath), ...)
    default:
        return gorm.Open(postgres.Open(dsn), ...)
    }
}
```

**配置**：
```yaml
database:
  driver: "sqlite"              # "postgres" (default) or "sqlite"
  sqlite_path: "./xray-pkg.db"  # SQLite 模式下的数据库文件路径
```

**model 兼容性**：
- `pq.StringArray` (PostgreSQL text[]) → SQLite 下改为 JSON string（`gorm:"type:text"`）
- `gen_random_uuid()` → 用 Go 端 `uuid.New()` 预生成（已有 `BeforeCreate` hook）
- `pgcrypto` extension → SQLite 跳过

#### 2.2 MinIO/S3 存储补完

**文件**：`xray-server/internal/storage/storage.go` — `MinIOStorage`
**改动**：用 `github.com/minio/minio-go/v7` 实现 `Save`/`Get`/`Delete`/`Exists`。

#### 2.3 认证加固

**文件**：`xray-server/internal/middleware/middleware.go`
**改动**：
1. `RequireAuth` 中验证 JWT 签名（当前只检查 token 存在）
2. 增加 token 缓存（LRU，避免每次查库）
3. `GitHubCallback` 补完 OAuth 全流程

#### 2.4 Publish 增强

**文件**：`xray-server/internal/handler/handler.go` — `PublishPackage()`
**改动**：
1. tarball 大小限制（默认 50MB）
2. 解压 tarball 读取 `xray.toml` 提取 metadata
3. SHA256 写入 `Version.TarballSHA256`
4. 验证包名格式 `owner/name`，`owner` 必须匹配当前用户的 username

#### 2.5 Docker 一键部署

**新文件**：`xray-server/Dockerfile`

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=1 go build -o /xray-pkg-server ./cmd/server

FROM alpine:3.19
RUN apk add --no-cache ca-certificates
COPY --from=builder /xray-pkg-server /usr/local/bin/
COPY config.yaml /etc/xray-pkg-server/
EXPOSE 8080
CMD ["xray-pkg-server"]
```

**新文件**：`xray-server/docker-compose.yml`

```yaml
version: "3.8"
services:
  server:
    build: .
    ports: ["8080:8080"]
    environment:
      - DATABASE_DRIVER=sqlite
      - DATABASE_SQLITE_PATH=/data/xray-pkg.db
    volumes:
      - ./data:/data
      - ./packages:/packages
```

自建用户只需：
```bash
git clone https://github.com/xray-lang/xray-server
cd xray-server
docker compose up -d
```

#### 2.6 验证清单

```bash
# SQLite 模式
DATABASE_DRIVER=sqlite DATABASE_SQLITE_PATH=./test.db make run

# Docker 模式
docker compose up -d
curl http://localhost:8080/health
curl http://localhost:8080/api/stats
```

---

### Phase 3：CLI 补完

**目标**：补齐 `remove`/`update`，完善 `install` 的错误处理和离线模式。

#### 3.1 `cmd_pkg_remove`

**文件**：`xray/src/app/cli/xcmd_pkg.c`
**逻辑**：
1. 解析参数 `owner/name`
2. 读取 `xray.toml`，从 `[dependencies]` section 删除对应行
3. 从 `xray.lock` 中删除对应条目
4. 删除 `~/.xray/packages/{owner}/{name}/` 目录
5. 打印结果

#### 3.2 `cmd_pkg_update`

**文件**：`xray/src/app/cli/xcmd_pkg.c`
**逻辑**：
1. 如果指定了包名，只更新该包；否则更新全部
2. 忽略 lockfile 中的版本锁定
3. 重新 resolve（获取满足约束的最新版本）
4. 下载新版本，更新 lockfile

#### 3.3 离线/缓存模式

**改动**：`xr_pkg_client_install()` 安装前先检查 `~/.xray/cache/` 是否有缓存 tarball（按 SHA256 匹配）。有则跳过下载。

`xray pkg install --offline` 模式：只使用缓存，不发网络请求。

#### 3.4 `cmd_pkg_info`（新增）

显示单个包的详细信息：
```bash
xray pkg info xray/redis
# Name:        xray/redis
# Version:     1.2.0
# License:     MIT
# Description: Redis client for Xray
# Versions:    1.0.0, 1.1.0, 1.2.0
# Registry:    https://pkg.xray-lang.org
```

#### 3.5 JSON 解析改进

**问题**：`xpkg_client.c` 中的 `json_get_string()` / `json_get_string_array()` 是手写 strstr 解析，不支持转义字符、嵌套对象等。

**改动**：复用项目已有的 JSON parser（`stdlib/json/` 或 `xr_json_parse()`），替换手写实现。

---

### Phase 4：高级能力

**目标**：原生包构建、workspace protocol、scoped 包名。

#### 4.1 原生包构建支持

xray-sqlite 类的原生包需要在安装时构建共享库。

**设计**：`xray.toml` 中 `native = true` 的包：
```toml
[package]
name = "xray/sqlite"
native = true

[build]
system = "cmake"           # "cmake" | "make" | "script"
script = "scripts/build.sh" # 仅 system = "script" 时使用
```

`xray pkg install` 安装原生包时：
1. 下载 + 解压
2. 检测构建系统（`CMakeLists.txt` → cmake, `Makefile` → make）
3. 执行构建
4. 将产物 `.dylib`/`.so` 安装到 `~/.xray/packages/{owner}/{name}/{version}/lib/`

#### 4.2 Workspace Protocol

大型 monorepo 中多个 xray 包共存：
```
my-monorepo/
├── xray.toml              # workspace root
├── packages/
│   ├── core/xray.toml
│   ├── web/xray.toml
│   └── cli/xray.toml
```

Workspace `xray.toml`：
```toml
[workspace]
members = ["packages/*"]
```

`xray pkg install` 在 workspace root 执行时，为所有成员解析并安装依赖，共享 lockfile。

#### 4.3 Git 依赖

```toml
[dependencies]
bob/helper = { git = "https://github.com/bob/helper.git", tag = "v1.0.0" }
```

实现：`git clone` → checkout tag → 当作本地路径依赖处理。

---

## 5. 文件变更清单

### Phase 0（最小闭环）

| 仓库 | 文件 | 变更类型 | 改动量(估) |
|------|------|---------|-----------|
| xray-server | `cmd/server/main.go` | 修改 | +1 行 (路由) |
| xray-server | `internal/handler/handler.go` | 修改 | +80 行 (OAuth + SHA256 + tarball 解析) |
| xray | `src/app/cli/xcmd_pkg.c` | 修改 | +120 行 (install 接通 + add 写回 toml) |

### Phase 1（私有 Registry）

| 仓库 | 文件 | 变更类型 | 改动量(估) |
|------|------|---------|-----------|
| xray | `src/module/xpkg_config.[ch]` | **新建** | ~300 行 |
| xray | `src/module/xpkg_client.h` | 修改 | ~10 行 (删除 hardcode, 增加 API) |
| xray | `src/module/xpkg_client.c` | 修改 | +100 行 (静态 registry + 多 token) |
| xray | `src/module/xproject.[ch]` | 修改 | +40 行 (registries + dep.registry) |
| xray | `src/app/cli/xcmd_pkg.c` | 修改 | +60 行 (--registry 参数, config 加载) |

### Phase 2（服务端加固）

| 仓库 | 文件 | 变更类型 | 改动量(估) |
|------|------|---------|-----------|
| xray-server | `internal/database/database.go` | 修改 | +30 行 (SQLite driver) |
| xray-server | `internal/config/config.go` | 修改 | +10 行 (driver/sqlite_path) |
| xray-server | `internal/model/model.go` | 修改 | +20 行 (SQLite 兼容) |
| xray-server | `internal/storage/storage.go` | 修改 | +80 行 (MinIO 实现) |
| xray-server | `internal/middleware/middleware.go` | 修改 | +40 行 (JWT 验签) |
| xray-server | `internal/handler/handler.go` | 修改 | +50 行 (publish 增强) |
| xray-server | `Dockerfile` | **新建** | ~20 行 |
| xray-server | `docker-compose.yml` | **新建** | ~15 行 |

### Phase 3（CLI 补完）

| 仓库 | 文件 | 变更类型 | 改动量(估) |
|------|------|---------|-----------|
| xray | `src/app/cli/xcmd_pkg.c` | 修改 | +200 行 (remove/update/info/offline) |
| xray | `src/module/xpkg_client.c` | 修改 | +40 行 (缓存检查) |

---

## 6. 竞品对照

| 特性 | npm | Cargo | Go modules | pip | **Xray (目标)** |
|------|-----|-------|-----------|-----|----------------|
| 包名格式 | `@scope/name` | `crate-name` | `module/path` | `name` | `owner/name` |
| 配置文件 | `package.json` | `Cargo.toml` | `go.mod` | `pyproject.toml` | `xray.toml` |
| Lock 文件 | `package-lock.json` | `Cargo.lock` | `go.sum` | `requirements.txt` | `xray.lock` |
| 私有源 | `.npmrc` registry | `[registries]` | `GOPROXY` | `--index-url` | `[registries]` in toml |
| 静态源 | verdaccio (需服务) | sparse index | `GOPROXY=file://` | `--find-links` | 静态目录 + `registry.json` |
| 自建门槛 | 中 (verdaccio/nexus) | 低 (git index) | 低 (Athens/GOPROXY) | 低 (devpi) | **低** (SQLite + Docker) |
| 原生包 | node-gyp | build.rs | cgo | wheel/sdist | `native = true` + cmake |
| Workspace | npm workspaces | Cargo workspace | go.work | N/A | `[workspace]` (Phase 4) |

---

## 7. 数据流图

### Publish 流程

```
用户                        CLI                      Registry Server
 │                          │                            │
 │ xray pkg publish         │                            │
 │─────────────────────────>│                            │
 │                          │ 读取 xray.toml             │
 │                          │ 验证 [package] section     │
 │                          │ 创建 tarball               │
 │                          │ 计算 SHA256                │
 │                          │ 加载 token                 │
 │                          │                            │
 │                          │ POST /api/packages         │
 │                          │ (multipart: tarball)       │
 │                          │───────────────────────────>│
 │                          │                            │ 验证 token
 │                          │                            │ 解压 tarball
 │                          │                            │ 读取 xray.toml
 │                          │                            │ 计算 SHA256
 │                          │                            │ 存储 tarball
 │                          │                            │ 创建/更新 Package
 │                          │                            │ 创建 Version
 │                          │        201 Created         │
 │                          │<───────────────────────────│
 │ Published xray/foo@1.0.0 │                            │
 │<─────────────────────────│                            │
```

### Install 流程

```
用户                        CLI                      Registry (may be multiple)
 │                          │                            │
 │ xray pkg install         │                            │
 │─────────────────────────>│                            │
 │                          │ 加载 xray.toml             │
 │                          │ 加载 xray.lock (if exists) │
 │                          │ 加载 registry config       │
 │                          │                            │
 │                          │ 构建 XrDepGraph            │
 │                          │ 对每个依赖:                │
 │                          │   确定 registry URL        │
 │                          │   GET /api/packages/o/n    │
 │                          │──────────────────────────>│
 │                          │   ← PackageInfo (versions) │
 │                          │<──────────────────────────│
 │                          │                            │
 │                          │ xr_resolve_dependencies()  │
 │                          │ (semver 约束求解)          │
 │                          │ (环检测 + 拓扑排序)        │
 │                          │                            │
 │                          │ 按拓扑序下载:              │
 │                          │   检查缓存                 │
 │                          │   GET .../download         │
 │                          │──────────────────────────>│
 │                          │   ← tarball               │
 │                          │<──────────────────────────│
 │                          │   校验 SHA256              │
 │                          │   解压到 ~/.xray/packages/ │
 │                          │                            │
 │                          │ 保存 xray.lock             │
 │ Installed 3 packages     │                            │
 │<─────────────────────────│                            │
```

---

## 8. 静态 Registry 详细规范

### 8.1 目录结构

```
<registry-root>/
├── registry.json                          # 必须，标识 registry 类型
├── <owner>/
│   └── <name>/
│       ├── metadata.json                  # 必须，包元数据 + 版本列表
│       ├── <version>.tar.gz               # 每个版本一个 tarball
│       └── <version>.tar.gz.sha256        # 可选，SHA256 校验文件
```

### 8.2 `registry.json`

```json
{
  "static": 1,
  "name": "my-company-registry",
  "description": "Internal package registry"
}
```

### 8.3 `metadata.json`

```json
{
  "name": "alice/mylib",
  "description": "My library",
  "license": "MIT",
  "latest": "1.2.0",
  "versions": {
    "1.0.0": {
      "published": "2026-01-15T10:00:00Z",
      "sha256": "abcdef1234567890...",
      "size": 12345,
      "dependencies": {
        "bob/utils": "^1.0.0"
      },
      "yanked": false
    },
    "1.2.0": {
      "published": "2026-03-20T14:30:00Z",
      "sha256": "fedcba0987654321...",
      "size": 15678,
      "dependencies": {
        "bob/utils": "^1.1.0"
      },
      "yanked": false
    }
  }
}
```

### 8.4 发布脚本示例

```bash
#!/bin/bash
# publish-static.sh — 发布包到静态 registry
# Usage: ./publish-static.sh /path/to/registry

REGISTRY=$1
PKG_NAME=$(grep 'name' xray.toml | head -1 | sed 's/.*"\(.*\)"/\1/')
PKG_VERSION=$(grep 'version' xray.toml | head -1 | sed 's/.*"\(.*\)"/\1/')
OWNER=$(echo $PKG_NAME | cut -d/ -f1)
NAME=$(echo $PKG_NAME | cut -d/ -f2)

# 创建 tarball
TARBALL="/tmp/${NAME}-${PKG_VERSION}.tar.gz"
tar czf "$TARBALL" --exclude=.git --exclude=build .

# 计算 SHA256
SHA256=$(shasum -a 256 "$TARBALL" | cut -d' ' -f1)
SIZE=$(stat -f%z "$TARBALL" 2>/dev/null || stat -c%s "$TARBALL")

# 复制到 registry
PKG_DIR="$REGISTRY/$OWNER/$NAME"
mkdir -p "$PKG_DIR"
cp "$TARBALL" "$PKG_DIR/${PKG_VERSION}.tar.gz"
echo "$SHA256" > "$PKG_DIR/${PKG_VERSION}.tar.gz.sha256"

# 更新 metadata.json（简化版，实际应 merge）
echo "Published $PKG_NAME@$PKG_VERSION to $REGISTRY"
echo "SHA256: $SHA256"
```

---

## 9. 测试策略

### 9.1 单元测试（客户端）

| 测试文件 | 覆盖模块 | Phase |
|---------|---------|-------|
| `tests/unit/module/test_semver.c` | semver 解析、约束匹配 | 已有 |
| `tests/unit/module/test_lockfile.c` | lockfile 读写、SHA256 | 已有 |
| `tests/unit/module/test_resolver.c` | 依赖图、环检测、拓扑排序 | 已有 |
| `tests/unit/module/test_pkg_config.c` | **新建**：registry 配置加载、优先级、resolve | Phase 1 |

### 9.2 集成测试

```bash
# scripts/test_pkg_e2e.sh
# 需要：xray binary, 本地 registry server 或 static dir

# Test 1: Static registry round-trip
setup_static_registry /tmp/test-registry
xray pkg publish --registry-url file:///tmp/test-registry
xray pkg install --registry-url file:///tmp/test-registry
verify_installed "alice/mylib" "1.0.0"

# Test 2: Dynamic registry round-trip (需要 xray-server 运行)
xray pkg login --registry default
xray pkg publish
xray pkg search mylib
xray pkg add alice/mylib@^1.0.0
xray pkg install
xray pkg tree
xray pkg remove alice/mylib
```

### 9.3 服务端测试

```go
// internal/handler/handler_test.go
func TestPublishPackage(t *testing.T) { ... }
func TestDownloadPackage(t *testing.T) { ... }
func TestSearchPackages(t *testing.T) { ... }
func TestGitHubOAuthFlow(t *testing.T) { ... }
```

---

## 10. 里程碑与时间线

| Phase | 里程碑 | 预估工作量 | 验收标准 |
|-------|--------|-----------|---------|
| **0** | 最小闭环 | 2-3 天 | 纯 xray 包可 publish → install → import |
| **0N** | Native 包闭环 | 2-3 天 | xray-sqlite 可 publish → install(编译) → dlopen → import |
| **1** | 私有 Registry | 3-4 天 | 静态 registry + 多 registry 配置 + 多 token |
| **2** | 服务端加固 | 3-4 天 | SQLite 后端 + Docker 部署 + OAuth 完整 |
| **3** | CLI 补完 | 2-3 天 | remove/update/info/offline 全部可用 |
| **4** | 高级能力 | 5-7 天 | workspace + git 依赖 |

**总计**：~15-24 个工作日

---

## 11. 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| `xpkg_client.c` 手写 JSON 解析器被恶意 payload 攻破 | 安全性 | Phase 3 替换为项目已有的 JSON parser |
| PostgreSQL 依赖导致私有部署门槛高 | 推广 | Phase 2 增加 SQLite 后端 |
| 原生包跨平台构建复杂 | 用户体验 | Phase 4 先只支持 cmake，后续扩展 |
| semver 约束求解在大依赖图上性能差 | 可用性 | xresolver 已有拓扑排序，O(V+E)，短期无风险 |
| tarball 中可能包含恶意文件（路径穿越等） | 安全性 | 服务端解压时验证路径，禁止 `..` 和绝对路径 |

---

## 12. 进度追踪（2026-04-24 更新）

### 已完成

| 项目 | 状态 |
|------|------|
| publish 协议修复：client 发送 name/version/description/license + tarball(multipart) | ✅ |
| XrProject 扩展：解析 description/license 字段 | ✅ |
| xray-sqlite 已发布到 https://pkg.xray-lang.org (v1.0.0) | ✅ |
| server Download 路由兼容（两条路由均注册） | ✅ |

### Phase 0 剩余（纯 xray 包端到端）

- [ ] 0.4 `cmd_pkg_install` 接通 resolver（遍历 dependencies → depgraph → resolve → download → extract）
- [ ] 0.5 `cmd_pkg_add` 安装后写回 xray.toml
- [ ] `xr_module_resolve_path` 第三方包查找逻辑完善（当前只找 `latest/` 硬编码目录）
- [ ] 端到端验证：纯 xray 包 publish → add → install → import

### Phase 0N：Native 包端到端（新增）

**核心问题**：当前 native module 是编译时静态注册（`xr_module_register_native`），第三方 native 包不可能静态链接到 xray binary。需要 **dlopen 动态加载**。

#### 0N.1 Runtime: dlopen 加载第三方 native 模块

**文件**：`xray/src/module/xmodule.c` — `load_native_module()`

当前逻辑只查 `native_loaders` hashmap（编译时注册的 stdlib 模块）。需要增加 **dlopen 回退路径**：

```c
static XrModule* load_native_module(XrayIsolate *isolate, const char *module_name) {
    // 1. 先查静态注册的 loaders（标准库）
    void *loader_ptr = xr_hashmap_get(registry->native_loaders, module_name);
    if (loader_ptr) {
        // ... 现有逻辑 ...
    }

    // 2. 查找已安装的第三方 native 包
    //    ~/.xray/packages/{owner}/{name}/{version}/lib/libxray_{name}.{so|dylib}
    char *lib_path = find_installed_native_lib(isolate, module_name);
    if (!lib_path) return NULL;

    // 3. dlopen 加载
    void *handle = dlopen(lib_path, RTLD_LAZY);
    if (!handle) { ... error ... }

    // 4. dlsym 获取 loader 入口
    //    约定函数名：xr_module_load
    NativeModuleLoader loader = (NativeModuleLoader)dlsym(handle, "xr_module_load");
    if (!loader) { ... error ... }

    // 5. 调用 loader
    XrModule *module = loader(isolate);
    // ... 同现有逻辑 ...
}
```

**关键设计**：
- 入口函数约定：`xr_module_load(XrayIsolate *isolate)` — xray-sqlite 已有此签名
- 搜索路径：`~/.xray/packages/{owner}/{name}/{version}/lib/libxray_{name}.so`
- 需要 `#include <dlfcn.h>`（POSIX）/ `LoadLibrary`（Windows）

#### 0N.2 `xray.toml` 扩展 `[build]` section

```toml
[package]
name = "xray/sqlite"
version = "1.0.0"
native = true

[build]
system = "cmake"                 # "cmake" | "make" | "custom"
# cmake_args = "-DCMAKE_BUILD_TYPE=Release"  # optional extra args
# custom_command = "./build.sh"               # only for system = "custom"
```

**文件**：`xray/src/module/xproject.[ch]`
- `XrProject` 增加：`char *build_system`、`char *build_args`、`char *build_command`
- `xr_project_load()` 解析 `[build]` section

#### 0N.3 `cmd_pkg_install` native build 自动化

**文件**：`xray/src/app/cli/xcmd_pkg.c` 或新文件 `xcmd_pkg_build.c`

安装 native 包时的额外步骤（在 download+extract 之后）：

```
1. 检测 xray.toml 的 native = true
2. 读取 [build].system
3. 执行构建:
   - cmake: mkdir build && cd build && cmake .. -DXRAY_SDK_DIR=... && make
   - make:  make XRAY_SDK_DIR=...
   - custom: sh build_command
4. 查找产物: find build -name "*.so" -o -name "*.dylib"
5. 复制到: ~/.xray/packages/{owner}/{name}/{version}/lib/
```

**XRAY_SDK_DIR 来源**：
- 用户安装了 xray → `$(which xray)/../..` 或 `~/.xray/sdk/`
- 从源码构建 → `XRAY_SDK_DIR` 环境变量
- xray binary 可以 `xray info --sdk-path` 输出 SDK 路径

#### 0N.4 xray-sqlite 适配

**文件**：`xray-sqlite/xray.toml` 增加 `[build]` section
**文件**：`xray-sqlite/src/sqlite_module.c` — 确认导出 `xr_module_load` 符号

#### 0N.5 验证清单

```bash
# 1. 发布（已完成）
cd xray-sqlite && xray pkg publish

# 2. 新项目安装
mkdir /tmp/test-native && cd /tmp/test-native
xray pkg init
xray pkg add xray/sqlite@^1.0.0

# 3. 自动安装 + 编译
xray pkg install
# 应看到:
#   Downloading xray/sqlite@1.0.0...
#   Building native package xray/sqlite (cmake)...
#   Installed xray/sqlite@1.0.0

# 4. 使用
cat > src/main.xr << 'EOF'
import sqlite
let db = sqlite.open(":memory:")
print(db)
EOF
xray src/main.xr
# 应正常加载 dlopen 的 native 模块

# 5. 跨平台验证
# macOS: libxray_sqlite.dylib
# Linux: libxray_sqlite.so
```

### Phase 0N 依赖关系

```
Phase 0 (纯包闭环)
  └─► Phase 0N.1 (dlopen runtime)  ← 可独立开发
  └─► Phase 0N.2 ([build] 解析)    ← 可独立开发
  └─► Phase 0N.3 (native build)    ← 依赖 0.4 + 0N.2
  └─► Phase 0N.4 (sqlite 适配)     ← 依赖 0N.3
```

### 推荐实施顺序

```
Day 1:  0.4 install 接通 + 0N.1 dlopen runtime（并行）
Day 2:  0.5 add 写回 toml + 0N.2 [build] 解析
Day 3:  0N.3 native build 自动化
Day 4:  0N.4 sqlite 适配 + 端到端测试
Day 5:  bug fix + Phase 1 启动
```
