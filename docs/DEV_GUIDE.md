# 开发指南 (DEV_GUIDE)

本文档介绍如何在本地搭建开发环境、编译项目以及进行调试。

---

## macOS 特别说明

> 本项目在 macOS（Intel 和 Apple Silicon 均支持）上开发体验良好，但有以下几点需要注意。

### 工具安装（推荐 Homebrew）

```bash
# 安装 Homebrew（如未安装）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 安装 Go
brew install go

# 安装 Bun
brew install bun
# 或官方脚本：curl -fsSL https://bun.sh/install | bash

# 安装 Redis（本地开发可选）
brew install redis
brew services start redis   # 后台启动

# 安装 air（热重载，可选）
go install github.com/air-verse/air@latest
```

安装 Go 后，确保 `$GOPATH/bin` 在 PATH 中（zsh 默认配置文件是 `~/.zshrc`）：

```bash
echo 'export PATH="$PATH:$(go env GOPATH)/bin"' >> ~/.zshrc
source ~/.zshrc
```

### Apple Silicon (M1/M2/M3) 说明

本机编译无需任何额外配置，Go 默认输出 `arm64` 原生二进制，性能最佳。

**交叉编译 Linux amd64**（用于部署到 x86 服务器）：

```bash
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 \
  go build -ldflags "-s -w" -o new-api-linux-amd64
```

> 注意：macOS 上 **不能** 使用 `-extldflags '-static'`（该选项仅 Linux 下有效）。
> 如需完整静态链接的 Linux 二进制，建议用 Docker 交叉编译（见下方）。

**通过 Docker 编译 Linux 静态二进制**：

```bash
docker run --rm -v "$(pwd)":/app -w /app golang:1.25 \
  go build -ldflags "-s -w -extldflags '-static'" -o new-api-linux
```

### 常用 macOS 诊断命令

```bash
# 查看端口占用（替代 Linux 的 netstat）
lsof -i :3000

# 杀掉占用端口的进程
kill -9 $(lsof -ti :3000)

# 一键在浏览器打开
open http://localhost:3000
```

### 防火墙提示

首次运行 `./new-api` 时，macOS 可能弹出防火墙对话框询问是否允许网络访问，选择"允许"即可。

---

## 环境要求

| 工具 | 最低版本 | 说明 |
|------|---------|------|
| Go | 1.25.1+ | 后端语言 |
| Bun | 最新版 | 前端包管理器（推荐），也可用 npm/yarn |
| Node.js | 18+ | 前端运行时（Bun 已内置，仅在不用 Bun 时需要） |
| Git | 任意 | 版本管理 |
| Redis | 可选 | 启用分布式缓存时需要 |
| MySQL/PostgreSQL | 可选 | 默认使用 SQLite，生产建议换用外部数据库 |

---

## 项目结构速览

```
new-api/
├── main.go              # 入口，注意通过 //go:embed web/dist 将前端嵌入二进制
├── go.mod
├── .env.example         # 环境变量示例，复制为 .env 后修改
├── web/                 # React 前端（Vite + Semi Design）
│   ├── package.json
│   ├── vite.config.js   # 开发代理：/api /mj /pg → localhost:3000
│   └── src/
├── router/              # HTTP 路由
├── controller/          # 请求处理
├── service/             # 业务逻辑
├── model/               # 数据模型（GORM）
├── relay/               # AI 上游代理适配器
├── middleware/          # 中间件
├── common/              # 共享工具（JSON、Redis、环境变量等）
├── dto/                 # 数据传输对象
└── docker-compose.yml   # 快速启动完整环境
```

---

## 快速开始（开发模式）

### 第一步：克隆仓库

```bash
git clone https://github.com/QuantumNous/new-api.git
cd new-api
```

`.env` 文件**不是必须的**。项目所有配置项均有默认值，不存在 `.env` 时会静默跳过，
本地开发直接启动即可（默认：SQLite 数据库、端口 3000、无 Redis）。

仅当需要覆盖默认值时才创建（如连接 MySQL/PostgreSQL、启用 Redis、修改端口）：

```bash
cp .env.example .env
# 编辑 .env，按需修改
```

### 第二步：安装前端依赖并构建

> **注意**：Go 的 `main.go` 通过 `//go:embed web/dist` 将前端打包进二进制，
> 因此 **每次修改前端后都需要重新 build 前端，再重新编译 Go**。

```bash
cd web
bun install          # 安装依赖
bun run build        # 构建到 web/dist/
cd ..
```

### 第三步：编译并运行后端

```bash
go mod download      # 下载 Go 依赖（首次或更新后执行）
go build -o new-api  # 编译（会嵌入 web/dist）
./new-api            # 运行，默认监听 :3000
```

浏览器访问 `http://localhost:3000`，初次启动会自动创建 SQLite 数据库文件。

---

## 日常开发启动方式

### 情况一：只改后端代码

直接 `go run main.go` 即可，访问 `:3000`。
后端服务会把 `web/dist`（上次构建好的前端产物）一并提供，无需单独启动前端。

```bash
cd ~/Desktop/code/api/new-api
go run main.go
# 访问 http://localhost:3000
```

若使用 air 实现后端热重载（改 Go 文件自动重启）：

```bash
cd ~/Desktop/code/api/new-api
air
```

### 情况二：需要同时修改前端代码

`go run main.go` 服务的是 `web/dist` 里**已构建好的静态文件**，修改前端源码后必须重新
`bun run build` 才能看到变化，效率低。

此时需要同时开启 Vite dev server，它带有 **HMR（热模块替换）**，改完 React 组件浏览器立刻刷新。

**终端 1：启动后端**（提供 API）

```bash
cd ~/Desktop/code/api/new-api
go run main.go
```

**终端 2：启动前端 dev server**（提供页面 + HMR）

```bash
cd ~/Desktop/code/api/new-api/web
bun run dev
```

访问 `http://localhost:5173`（不是 3000）。
Vite 会把 `/api`、`/mj`、`/pg` 请求自动代理到 `:3000` 的后端：

| 前端路径 | 代理目标 |
|---------|---------|
| `/api/*` | `http://localhost:3000` |
| `/mj/*`  | `http://localhost:3000` |
| `/pg/*`  | `http://localhost:3000` |

> 首次启动前需确保 `web/dist` 存在（`go run main.go` 编译时需要），
> 执行一次 `cd web && bun run build` 即可，之后只改前端时不必重复构建。

---

## 后端热重载（使用 air）

[air](https://github.com/air-verse/air) 可以在 Go 文件变动时自动重新编译并重启服务。

```bash
# 安装 air
go install github.com/air-verse/air@latest

# 在项目根目录运行
air
```

> **注意**：air 触发重编时同样需要 `web/dist` 存在，
> 如果只开发后端代码，保留一次 `bun run build` 生成的静态文件即可。

---

## 环境变量说明

项目从 `.env` 文件和系统环境变量读取配置（系统环境变量优先级高于 `.env`）。
`.env` 不是必须的，所有变量均有默认值，本地开发可按需覆盖。

### 本地开发最常用

```bash
PORT=3000
GIN_MODE=debug
DEBUG=true
SQL_DSN=root:password@tcp(127.0.0.1:3306)/new-api?parseTime=true  # 换 MySQL
REDIS_CONN_STRING=redis://localhost:6379
SESSION_SECRET=your-random-secret-here
```

### 完整变量列表

#### 服务器

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `PORT` | `3000` | 监听端口 |
| `GIN_MODE` | `release` | 设为 `debug` 开启 Gin 详细日志 |
| `NODE_TYPE` | `master` | 多节点时从节点设为 `slave` |
| `FRONTEND_BASE_URL` | — | 前端访问地址（反向代理场景） |

#### 安全

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `SESSION_SECRET` | 内置默认值 | Session 加密密钥，多实例部署必须设置，且不能填 `random_string` |
| `CRYPTO_SECRET` | 同 `SESSION_SECRET` | 数据库敏感字段加密密钥 |
| `TLS_INSECURE_SKIP_VERIFY` | `false` | 跳过上游 TLS 证书验证 |
| `TRUSTED_REDIRECT_DOMAINS` | — | 支付回调可信域名，逗号分隔，如 `example.com,myapp.io` |

#### 数据库

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `SQL_DSN` | — | 不填则用 SQLite。MySQL: `root:pwd@tcp(127.0.0.1:3306)/new-api?parseTime=true`；PostgreSQL: `postgresql://root:pwd@localhost:5432/new-api` |
| `LOG_SQL_DSN` | 同主库 | 日志单独存一个数据库时填，格式同上 |
| `SQLITE_PATH` | `./new-api.db` | SQLite 文件路径 |
| `SQL_MAX_IDLE_CONNS` | `100` | 最大空闲连接数 |
| `SQL_MAX_OPEN_CONNS` | `1000` | 最大打开连接数 |
| `SQL_MAX_LIFETIME` | `60` | 连接最大生命周期（秒） |

#### Redis / 缓存

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `REDIS_CONN_STRING` | — | 不填则禁用 Redis，用内存缓存。如 `redis://localhost:6379` |
| `MEMORY_CACHE_ENABLED` | `false` | 启用内存缓存（开启 Redis 后自动为 true） |
| `SYNC_FREQUENCY` | `60` | 数据库 → 内存缓存同步频率（秒） |

#### 超时 / 限流

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `RELAY_TIMEOUT` | `0`（不限） | 上游请求超时（秒） |
| `STREAMING_TIMEOUT` | `300` | 流模式无数据超时（秒），出现空补全时可调大 |
| `RELAY_MAX_IDLE_CONNS` | `500` | HTTP 连接池最大空闲连接 |
| `RELAY_MAX_IDLE_CONNS_PER_HOST` | `100` | 每个上游最大空闲连接 |
| `GLOBAL_API_RATE_LIMIT_ENABLE` | `true` | 全局 API 限流开关 |
| `GLOBAL_API_RATE_LIMIT` | `180` | 限流窗口内最大请求数 |
| `GLOBAL_API_RATE_LIMIT_DURATION` | `180` | 限流窗口（秒） |
| `GLOBAL_WEB_RATE_LIMIT_ENABLE` | `true` | Web 页面限流开关 |
| `GLOBAL_WEB_RATE_LIMIT` | `60` | Web 限流窗口内最大请求数 |
| `GLOBAL_WEB_RATE_LIMIT_DURATION` | `180` | Web 限流窗口（秒） |
| `CRITICAL_RATE_LIMIT_ENABLE` | `true` | 关键接口（登录等）限流开关 |
| `CRITICAL_RATE_LIMIT` | `20` | 关键接口窗口内最大请求数 |
| `CRITICAL_RATE_LIMIT_DURATION` | `1200` | 关键接口限流窗口（秒） |

#### 任务 / 功能

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `UPDATE_TASK` | `true` | 启用 Midjourney 等异步任务轮询 |
| `BATCH_UPDATE_ENABLED` | `false` | 批量写库（高并发场景减少 DB 压力） |
| `BATCH_UPDATE_INTERVAL` | `5` | 批量写库间隔（秒） |
| `CHANNEL_UPDATE_FREQUENCY` | — | 自动测试渠道余额频率（秒），不填则不启用 |
| `POLLING_INTERVAL` | `0` | 任务轮询间隔（秒） |
| `TASK_QUERY_LIMIT` | `1000` | 任务轮询单次查询数量 |
| `TASK_TIMEOUT_MINUTES` | `1440` | 异步任务超时（分钟），超时自动退款，`0` 表示不限 |
| `TASK_PRICE_PATCH` | — | 任务价格覆盖，逗号分隔 |

#### 计费 / Token 统计

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `FORCE_STREAM_OPTION` | `true` | 强制上游返回 usage 字段，用于精确计费 |
| `CountToken` | `true` | 是否统计 token 用量 |
| `GET_MEDIA_TOKEN` | `true` | 统计图片/媒体 token |
| `GET_MEDIA_TOKEN_NOT_STREAM` | `false` | 非流模式也统计图片 token |

#### 文件 / 请求大小限制

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `MAX_FILE_DOWNLOAD_MB` | `64` | 最大文件下载大小（MB） |
| `MAX_REQUEST_BODY_MB` | `128` | 最大请求体大小（MB，防 zip bomb） |
| `STREAM_SCANNER_MAX_BUFFER_MB` | `64` | 流式响应 scanner 缓冲区（MB） |

#### AI 提供商

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `GEMINI_SAFETY_SETTING` | `BLOCK_NONE` | Gemini 安全过滤级别 |
| `GEMINI_VISION_MAX_IMAGE_NUM` | `16` | Gemini 单次最大图片数 |
| `COHERE_SAFETY_SETTING` | `NONE` | Cohere 安全设置 |
| `AZURE_DEFAULT_API_VERSION` | `2025-04-01-preview` | Azure OpenAI 默认 API 版本 |
| `DIFY_DEBUG` | `true` | Dify 渠道是否输出工作流节点信息到客户端 |

#### 调试 / 监控

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `DEBUG` | `false` | 开启详细调试日志 |
| `ENABLE_PPROF` | `false` | 开启 pprof 性能分析，监听 `:8005` |
| `ERROR_LOG_ENABLED` | `false` | 将错误响应记录到日志 |
| `PYROSCOPE_URL` | — | Pyroscope 持续性能分析服务地址 |
| `PYROSCOPE_APP_NAME` | — | Pyroscope 应用名 |
| `PYROSCOPE_BASIC_AUTH_USER` | — | Pyroscope 认证用户名 |
| `PYROSCOPE_BASIC_AUTH_PASSWORD` | — | Pyroscope 认证密码 |

#### 前端分析（注入到 HTML）

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `GOOGLE_ANALYTICS_ID` | — | Google Analytics 4 测量 ID，如 `G-XXXXXXXXXX` |
| `UMAMI_WEBSITE_ID` | — | Umami 网站 ID |
| `UMAMI_SCRIPT_URL` | 官方地址 | 自建 Umami 时填自定义脚本 URL |

#### 其他

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `GENERATE_DEFAULT_TOKEN` | `false` | 初次启动时是否自动生成一个 token |
| `NOTIFY_LIMIT_COUNT` | `2` | 通知发送次数上限 |
| `NOTIFICATION_LIMIT_DURATION_MINUTE` | `10` | 通知限流窗口（分钟） |
| `VERSION` | — | 版本号（CI 构建时注入，本地一般不填） |

---

## 出包方式

### 方式一：GitHub Actions 自动出包（推荐）

推送一个 tag 到 `fork` 远端，CI 自动完成前端构建、Go 编译、打包、发布全流程。

```bash
git checkout main
git pull fork main          # 确保本地 main 与远端一致

git tag v0.12.6-custom      # 版本号自定，不能含 -alpha（会被过滤）
git push fork v0.12.6-custom
```

推上去后自动触发两条流水线：

| 流水线 | 产物 | 时间 |
|--------|------|------|
| `release.yml` | Linux amd64/arm64、macOS、Windows 二进制包 + checksums | ~10 分钟 |
| `docker-image-arm64.yml` | `arronlee/new-api:latest` + `arronlee/new-api:v0.12.6-custom` | ~15 分钟 |

产物位置：`github.com/reputationly/new-api/releases`

**产物文件名规则**（来自 release.yml）：

| 平台 | 文件名 |
|------|--------|
| Linux amd64 | `new-api-v0.12.6-custom` |
| Linux arm64 | `new-api-arm64-v0.12.6-custom` |
| macOS | `new-api-macos-v0.12.6-custom` |
| Windows | `new-api-v0.12.6-custom.exe` |

> **删除 tag**（打错了）：
> ```bash
> git tag -d v0.12.6-custom               # 删本地
> git push fork --delete v0.12.6-custom   # 删远端
> ```

---

### 方式二：本地手动编译

编译时注入版本号并压缩体积（与 CI 流程一致）：

```bash
# 先构建前端
cd web && bun install && VITE_REACT_APP_VERSION=v1.0.0 bun run build && cd ..

# macOS 本机编译（不加 -extldflags，macOS 不支持静态链接）
go build -ldflags "-s -w -X 'github.com/QuantumNous/new-api/common.Version=v1.0.0'" -o new-api

# 在 macOS 上交叉编译 Linux amd64（CGO_ENABLED=0 跳过 CGO）
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 \
  go build -ldflags "-s -w -X 'github.com/QuantumNous/new-api/common.Version=v1.0.0'" -o new-api-linux-amd64

# 在 macOS 上交叉编译 Linux arm64
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 \
  go build -ldflags "-s -w -X 'github.com/QuantumNous/new-api/common.Version=v1.0.0'" -o new-api-linux-arm64

# 需要完整静态链接时，使用 Docker（仅限 Linux 目标）
docker run --rm -v "$(pwd)":/app -w /app golang:1.25 \
  go build -ldflags "-s -w -extldflags '-static' -X 'github.com/QuantumNous/new-api/common.Version=v1.0.0'" -o new-api-linux-static

# Windows 编译
GOOS=windows GOARCH=amd64 CGO_ENABLED=0 \
  go build -ldflags "-s -w -X 'github.com/QuantumNous/new-api/common.Version=v1.0.0'" -o new-api.exe
```

---

## Docker 快速启动

如果只是想跑通完整环境（含 PostgreSQL + Redis），直接用 docker-compose：

```bash
docker-compose up -d
# 访问 http://localhost:3000
```

默认使用 PostgreSQL。如需 MySQL，参考 `docker-compose.yml` 中的注释进行切换。

---

## 调试技巧

### pprof 性能分析

```bash
ENABLE_PPROF=true ./new-api
# pprof 监听在 :8005
go tool pprof http://localhost:8005/debug/pprof/profile
```

### Gin 详细日志

```bash
GIN_MODE=debug ./new-api
```

### IDE 调试（GoLand / VS Code）

**VS Code** 示例 `.vscode/launch.json`：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch new-api",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}/main.go",
      "env": {
        "GIN_MODE": "debug",
        "DEBUG": "true",
        "PORT": "3000"
      },
      "args": []
    }
  ]
}
```

> **注意**：IDE 调试前必须先确保 `web/dist` 目录存在（`//go:embed web/dist` 在编译时会检查），
> 否则编译会报错。只需运行一次 `cd web && bun run build` 即可。

**GoLand**：在 Run Configuration 中设置 `Environment variables`，添加 `GIN_MODE=debug;DEBUG=true`。

### 运行测试

```bash
# 运行所有测试
go test ./...

# 运行特定包的测试
go test ./relay/...

# 带详细输出
go test -v ./...
```

---

## 国际化（i18n）开发

### 前端

```bash
cd web
bun run i18n:extract   # 从源码提取新的翻译键
bun run i18n:sync      # 同步翻译文件（补全缺失键）
bun run i18n:lint      # 检查翻译文件问题
bun run i18n:status    # 查看各语言翻译完成度
```

翻译文件位于 `web/src/i18n/locales/{lang}.json`（zh/en/fr/ru/ja/vi）。

### 后端

翻译文件位于 `i18n/` 目录，格式为 TOML，语言：en/zh。

---

## 常见问题

**Q: `go build` 报错 `pattern web/dist: no matching files found`**
A: 需要先构建前端。运行 `cd web && bun run build`。

**Q: 前端修改后页面没有更新**
A: 使用前端 dev server（`bun run dev`）时修改会自动热更新。
如果直接访问后端，需要 `bun run build` 后重新编译 Go。

**Q: 数据库初始化失败**
A: 默认使用 SQLite，无需额外配置。检查当前目录是否有写权限。
使用 MySQL/PostgreSQL 时，确认 `SQL_DSN` 格式正确，数据库已创建。

**Q: Redis 连接失败导致启动报错**
A: 不配置 `REDIS_CONN_STRING` 则自动使用内存缓存，本地开发无需 Redis。

**Q: 首次登录的管理员账号**
A: 首次启动访问 `http://localhost:3000`，系统会引导创建初始管理员账号。
