# TGBot — Bot 运行时（tgbot-go）+ 管理面板（frontend）

Telegram Bot 的运行时服务 + Web 管理面板，负责消息处理、业务交互、内部管理 API 和与商城 API 的数据同步。

## 架构概览

```
┌──────────────────────────────────────────────────────────┐
│  管理面板 (Vue 3 SPA)                                     │
│  https://bot.example.com/tgbot                            │
│  开发端口: 5175  │  生产: nginx 静态文件服务               │
└───────────┬──────────────────────────────────────────────┘
            │ /api/v1/* (JWT)
            ▼
┌──────────────────┐    ┌──────────────────────────────────┐
│  核心 API          │◄───│  Bot 运行时 (tgbot-go)            │
│  (dujiao-next)     │    │  管理端口: 8015                   │
│  端口: 8088        │    │  健康检查: /health                │
│  /health           │    │  服务状态: /api/v1/overview/     │
│                    │    │           service-health (公开)   │
└──────────────────┘    └──────────────────────────────────┘
```

| 项目 | 值 |
|------|-----|
| **tgbot-go 管理端口** | `8015`（可通过 `TGBOT_PORT` 环境变量修改） |
| **核心 API 地址** | `http://127.0.0.1:8088`（通过 `TGBOT_API_URL` 环境变量修改） |
| **技术栈** | Go + GORM + SQLite + Vue 3 + TypeScript + Vite |
| **进程管理** | PM2 / systemd / start.sh |
| **启动入口** | `start.sh`（开发） / `deploy.sh`（生产） |
| **数据库** | 与核心 API 共享 `dujiao.db`，或使用独立 `tgbot.db` |

## 快速开始

### 前置条件

- Go 1.23+
- Node.js 20+（前端开发）
- 核心 API（dujiao-next）已运行
- SQLite 数据库已初始化

### 1. 配置

```bash
cp .env.example .env
# 编辑 .env，填入你的配置：
#   TGBOT_APP_SECRET       — AES-256 加密密钥（必需）
#   TGBOT_DB_PATH          — 数据库路径（必需）
#   TGBOT_API_URL          — 核心 API 地址
#   TGBOT_PORT             — 管理面板端口
#   TGBOT_JWT_SECRET       — JWT 密钥（需与核心 API 一致）
#   TGBOT_CHANNEL_KEY      — Channel 凭据（有 Bot 时必需）
#   TGBOT_CHANNEL_SECRET   — Channel 凭据（有 Bot 时必需）
```

> **无 Bot 模式**：如果还没有配置任何 Bot，可以只提供 `TGBOT_APP_SECRET` 和 `TGBOT_DB_PATH`，tgbot-go 将以"仅管理面板"模式启动，健康检查和服务状态页面正常可用。

### 2. 启动 Bot 运行时

```bash
# 开发环境
./start.sh

# 生产环境
./deploy.sh
```

### 3. 验证

```bash
# 健康检查
curl http://127.0.0.1:8015/health
# → {"checks":{"db":"ok"},"status":"ok"}

# 服务健康状态（公开，无需认证）
curl http://127.0.0.1:8015/api/v1/overview/service-health
# → {"database":{"connected":true,...},"api":{"connected":true,...},"cache":{...}}
```

### 4. 启动前端（开发）

```bash
cd frontend
npm install
npm run dev -- --host 0.0.0.0
# 打开 http://localhost:5175/tgbot/login
```

### 5. 生产部署

```bash
npm run build     # 产出 frontend/dist/
# 将 dist/ 部署到 nginx，参考下方 nginx 配置
```

## 管理面板 API 参考

| 端点 | 认证 | 说明 |
|------|------|------|
| `/health` | 无 | Bot 运行时健康检查 |
| `/api/v1/overview/service-health` | 无 | 数据库 + API + 缓存三服务状态 |
| `/api/v1/overview/error-logs` | JWT/HMAC | 错误日志查询 |
| `/api/v1/tgbot-runtime/bot-users` | JWT/HMAC | Bot 用户列表 |
| `/api/v1/tgbot-runtime/stats` | JWT/HMAC | 统计数据 |
| `/api/v1/support/messages` | JWT/HMAC | 客服消息 |

Bot 管理（CRUD/激活）通过核心 API 的 `/api/v1/admin/channel-clients` 端点。

## 第三方安装注意事项

1. **核心 API 必须先运行**，tgbot-go 依赖它来获取 Bot 配置
2. **JWT_SECRET 必须一致**：tgbot-go 和核心 API 使用相同的 JWT 密钥
3. **服务健康状态端点无需认证**：方便初始安装时验证连通性
4. **零 Bot 可启动**：首次安装无需配置 Bot，管理面板可直接使用
5. **缓存服务可选**：Redis 未配置时显示"未启用"，不影响正常使用
6. **端口选择**：如果同一服务器运行多个 tgbot-go 实例，通过 `TGBOT_PORT` 区分

## 开发

```bash
# 编译 tgbot-go
GOGC=20 GOMEMLIMIT=3500MiB go build -buildvcs=false -o tgbot-go ./cmd/server/

# 运行测试
go test ./...
go test -race ./...
```

## Nginx 反向代理示例

> **重要**：管理面板需要同时访问两个后端 — 核心 API (`dujiao-next`，默认 `:8088`) 和 Bot 运行时 (`tgbot-go`，默认 `:8015`/`8016`)。
> Nginx 必须按路径将请求分流到正确的后端，否则会出现 404 或"资源不存在"错误。

```nginx
server {
    listen 443 ssl http2;
    server_name bot.example.com;

    # ── tgbot-go 专属 API（必须在通用 /api/ 之前，更长前缀优先匹配）──
    location /api/v1/tgbot-runtime/ {
        proxy_pass http://127.0.0.1:8016;   # tgbot-go 管理端口
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_http_version 1.1;
    }
    location /api/v1/overview/       { proxy_pass http://127.0.0.1:8016; }
    location /api/v1/support/        { proxy_pass http://127.0.0.1:8016; }
    location /tgbot-runtime/         { proxy_pass http://127.0.0.1:8016; }
    location /health                 { proxy_pass http://127.0.0.1:8016; }
    location /internal/              { proxy_pass http://127.0.0.1:8016; }

    # ── 通用 API → 核心 API（dujiao-next）──
    location /api/ {
        proxy_pass http://127.0.0.1:8088;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_http_version 1.1;
    }

    # ── 上传文件 → 核心 API ──
    location /uploads/ {
        proxy_pass http://127.0.0.1:8088;
    }

    # ── 静态资源（长缓存）──
    location /assets/ {
        root /path/to/tgbot/frontend/dist;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # ── SPA 前端（所有其他路径回退到 index.html）──
    location / {
        root /path/to/tgbot/frontend/dist;
        try_files $uri $uri/ /index.html;
    }
}
```

> **注意**：
> - 如果 tgbot-go 使用不同端口（通过 `TGBOT_PORT` 配置），请将上面 8016 改为实际端口。
> - `location /api/v1/tgbot-runtime/` 等 tgbot-go 专属路径**必须放在** `location /api/` **之前**，否则请求会被通用 `/api/` 规则拦截并错误转发到核心 API。
> - 前端使用 HTML5 History 模式，`try_files` 回退到 `index.html` 是必须的，否则刷新页面会 404。

## 相关文档

- [根目录 CLAUDE.md](./CLAUDE.md) — 开发者参考
- [前端 CLAUDE.md](./frontend/CLAUDE.md) — 前端架构详解
- [.env.example](./.env.example) — 完整环境变量列表
