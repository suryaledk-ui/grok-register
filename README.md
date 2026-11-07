# Grok Register

FastAPI + React + Playwright 浏览器适配层的 Web 注册管理工具。默认 Camoufox，可切换 CloakBrowser。支持注册任务、账号管理、CPA / Grok2API 授权文件生成。

部署文档: DEPLOYMENT.md · Web 说明: WEB.md

上游 grok.com 已不再下发 `bfs` 标记，本注册机无法据此判断风控或降智。账号级降智检测需另行接入 GrokIQ（https://github.com/kaibush/grok-iq）。

## 环境要求

宿主机只需安装 Docker 与 Docker Compose。无需桌面环境，容器内使用 Xvfb + 有头浏览器，Docker 模式强制关闭无头模式。

## Docker 快速启动

```bash
git clone https://github.com/kaibush/grok-register.git
cd grok-register
cp .env.example .env
docker compose build
docker compose up -d
```

访问 `http://服务器IP:8787`。

查看状态和日志：

```bash
docker compose ps
docker compose logs -f grok-register
curl http://127.0.0.1:8787/api/health
```

首次访问会要求创建唯一管理员账号。

默认后端为 Camoufox，可在设置页切换 CloakBrowser。

若代理为 `127.0.0.1:7897`，Compose 会自动映射到宿主机代理；宿主机代理软件需监听 `0.0.0.0` 或 Docker 网桥地址。

完整说明见 DEPLOYMENT.md。

## 可选 OutlookEmail 邮箱池

Compose 已集成 `ghcr.io/assast/outlookemail:latest`，默认不随主服务启动。需要时在 `.env` 修改登录密码与 `SECRET_KEY`，然后启动可选 profile：

```bash
docker compose --profile outlookemail up -d
```

访问地址：

```text
Grok Register: http://服务器IP:8787
OutlookEmail:  http://服务器IP:5000
```

`5000` 默认映射到所有网卡。主容器内 API Base 使用 `http://outlook-email:5000`。

Docker 首次生成 `data/config.json` 时会预填该内部地址；已有配置可在“系统设置 → Outlook 邮箱池”填写。

OutlookEmail 数据保存在 `outlookemail-data/`，已被 Git 与 Docker 构建上下文忽略。完整配置见 DEPLOYMENT.md。

## 可选 GrokIQ 联动

复制环境变量模板，并为两端设置相同的联动 Token：

```bash
cp .env.example .env

# 编辑 .env
GROKIQ_WEBHOOK_TOKEN=替换为随机长字符串
```

用两个 Compose 文件启动注册机、GrokIQ 后端与前端：

```bash
docker compose -f compose.yaml -f compose.grokiq.yaml pull
docker compose -f compose.yaml -f compose.grokiq.yaml up -d
```

默认访问地址：

```text
Grok Register: http://服务器IP:8787
GrokIQ:        http://服务器IP:8091
```

GrokIQ 前端通过容器内 Nginx 将 `/api` 转发至 `grokiq-backend:8090`，后端端口无需暴露。`8091` 默认监听所有网卡；使用反向代理时在 `.env` 设置 `GROKIQ_WEB_BIND=127.0.0.1`。

验证联动服务：

```bash
docker compose -f compose.yaml -f compose.grokiq.yaml ps
curl http://127.0.0.1:8091/api/health
docker compose -f compose.yaml -f compose.grokiq.yaml logs -f grokiq-backend grokiq-frontend
```

首次启动后，在 GrokIQ“系统设置 → 联动与启动项”保存联动 Token、首次探针方案和出口目标；再到 Grok Register“系统设置 → Grok2API”启用联动并填写相同 Token。

统一 Compose 中 `GROKIQ_REGISTER_PROBE_STABILIZATION_SECONDS` 控制新账号事件后等待多久创建首次探针，默认 `15`，设为 `0` 关闭。

## 配置文件

本机运行读取根目录 `config.json`：

```bash
cp config.example.json config.json
```

Docker 运行读取宿主机 `data/config.json`：

```bash
mkdir -p data
cp config.json data/config.json
docker compose restart grok-register
```

也可在 Web“系统设置”中修改配置。

Docker 配置中的授权目录建议保持：

```json
{
  "cpa_auth_dir": "data/cpa_auth",
  "grok2api_auth_dir": "data/grok2api_auth",
  "grok2api_remote_url": "https://grok2api.example.com",
  "grok2api_remote_username": "admin",
  "grok2api_remote_password": "change-me",
  "grok2api_auto_import": true,
  "grok2api_auto_import_build": true,
  "grok2api_auto_import_web": false,
  "grok2api_auto_import_console": false
}
```

## 本机运行

要求 Python 3.10+、Node.js 22+。

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/python -m camoufox fetch

cd front
npm install
npm run build
cd ..

cp config.example.json config.json
./start-web.sh
```

访问 `http://127.0.0.1:8787`。

Windows 启动：

```powershell
.venv\Scripts\python.exe -m backend.web.cli --host 127.0.0.1 --port 8787
```

## 主要配置

建议直接在 Web 设置页填写。

| 配置项 | 说明 |
| --- | --- |
| `email_provider` | 邮箱服务商 |
| `register_count` | 注册数量 |
| `register_workers` | 并发数量，默认 1 |
| `proxy` | 注册与 OAuth 使用的 HTTP(S) 代理；支持 `http://host:port` 和 `http://user:password@host:port`，凭据特殊字符需 URL 编码。风控名单在「账号中心 → 出口 IP 风控」查看 |
| `browser_engine` | `camoufox`（默认）或 `cloakbrowser` |
| `browser_headless` | 本机无头模式；Docker 中强制关闭 |
| `browser_low_traffic_mode` | 低流量模式，默认开启；注册与重新登录共用静态资源缓存 |
| `browser_traffic_savings_level` | 低流量节省级别，默认 `standard`；`more` 额外缓存 accounts.x.ai 静态资源 |
| `cpa_auto_add` | 注册后生成 CPA 授权 |
| `sso_detailed_risk_check` | 获取 SSO 后读取 `botFlagSource`（不能作为风控结论） |
| `cpa_registration_risk_check` | 注册后复查 `botFlag`；默认关闭 |
| `cpa_auth_dir` | CPA JSON 保存目录 |
| `cpa_remote_url` | CPA Management API 地址 |
| `cpa_management_key` | CPA 管理密钥 |
| `grok2api_auth_dir` | Grok2API JSON 保存目录 |
| `grok2api_remote_url` | 远程 Grok2API 站点根地址 |
| `grok2api_remote_username` | 远程管理员账号 |
| `grok2api_remote_password` | 远程管理员密码 |
| `grok2api_auto_import` | JSON 生成后自动登录并导入远程 Grok2API |
| `grok2api_auto_import_build` | 自动导入 grok_build，默认开启 |
| `grok2api_auto_import_web` | 自动导入 grok_web，默认关闭 |
| `grok2api_auto_import_console` | 自动导入 grok_console，默认关闭 |
| `grokiq_webhook_enabled` | 导入后发送 Webhook |
| `grokiq_webhook_url` | GrokIQ `account-imported` 接口地址 |
| `grokiq_webhook_token` | 请求头 `x-grokiq-token` |
| `grokiq_webhook_timeout_seconds` | 单次投递超时；失败由持久 Outbox 退避重试 |

配置模板见 `config.example.json`。

## 数据目录

```text
data/
├── config.json
├── web_auth.json
├── accounts/
├── cpa_auth/
├── grok2api_auth/
└── cloakbrowser-cache/

logs/
outlookemail-data/
```

`data/`、`logs/` 与本地 `config.json` 已被 Git 忽略。

## 常用命令

```bash
# 停止服务
docker compose down

# 更新本地构建
git pull
docker compose up -d --build

# 验证有头 Camoufox
docker compose run --rm grok-register python /app/docker/camoufox_smoke.py

# 验证有头 CloakBrowser（首次运行会下载并缓存 Chromium）
docker compose run --rm grok-register python /app/docker/cloakbrowser_smoke.py

# 后端测试
.venv/bin/python -m unittest discover -s backend/tests -v

# 前端构建
cd front && npm run build
```

Docker 镜像更新：

```bash
docker compose pull
docker compose up -d --force-recreate
```

## 常见问题

### Docker 修改配置后未生效

Docker 读取 `data/config.json`，不是根目录 `config.json`。修改后：

```bash
docker compose restart grok-register
```

### Camoufox 未安装

```bash
.venv/bin/python -m camoufox fetch
.venv/bin/python -m camoufox version
```

### CloakBrowser 首次启动

```bash
.venv/bin/python -m cloakbrowser install
```

Docker 会把缓存保存在 `data/cloakbrowser-cache/`。

### 公网 HTTPS 登录状态异常

`.env` 中设置：

```dotenv
GROK_WEB_COOKIE_SECURE=1
```

然后重建容器：

```bash
docker compose up -d --force-recreate
```

### CPA 没有出现新账号

检查 `cpa_auto_add`、`cpa_auth_dir`，或远程配置 `cpa_remote_url`、`cpa_management_key`，并查看日志中的 `[CPA]` 信息。

## 项目结构

```text
front/                  React 前端
backend/                Python 后端
  web/                  FastAPI、认证与任务调度
  registration/         注册编排、仓储和结果产物
  automation/           浏览器运行时与页面适配
  integrations/         代理、连通性和授权交换
  mailbox/              邮箱渠道适配
  shared/               公共路径等基础设施
backend/tests/          后端测试
docker/                 容器启动与浏览器验证
.github/workflows/      GitHub Actions
data/                   运行数据
logs/                   运行日志
outlookemail-data/      可选 OutlookEmail 数据
compose.yaml            Docker Compose 配置
compose.grokiq.yaml     GrokIQ 联动编排
```

License: MIT
