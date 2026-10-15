# SlideFlow 本地运行指南

以下步骤面向 Windows PowerShell 和 Python 后端。本地需要 PostgreSQL、Redis、API、Worker 与前端共同运行，其中 API、Worker 和前端分别占用一个终端。

## 环境与端口

后端要求 Python 3.12 或以上，使用 uv 管理依赖。2026 年 10 月 15 日的验证环境为 Python 3.13、Node.js 24、PostgreSQL 16 和 Redis 7。

| 服务 | 本地地址或端口 |
| --- | --- |
| 前端 | `http://127.0.0.1:39173` |
| FastAPI | `http://127.0.0.1:39800` |
| PostgreSQL | `localhost:39432` |
| Redis | `localhost:39379` |

在项目根目录检查基础工具及容器：

```powershell
python --version
uv --version
node --version
npm --version
docker --version
docker compose ps
```

Docker Desktop 启动完成后，运行：

```powershell
docker compose up -d
docker compose ps
```

等待 `postgres` 和 `redis` 的状态变为 `healthy`。

## 安装后端与配置模型

```powershell
# 从项目根目录开始
cd backend
uv sync --locked
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
notepad .env
```

已有 `.env` 时保留现有值。默认本地基础设施对应的配置如下：

```dotenv
DATABASE_URL=postgresql+asyncpg://aippt:aippt@localhost:39432/aippt
REDIS_URL=redis://localhost:39379/0
LLM_API_KEY=replace-with-your-key
LLM_BASE_URL=https://api.deepseek.com
LLM_MODEL=deepseek-flash
```

`deepseek-flash` 是本次试验使用的模型名；新账号应根据供应商实际可用模型设置。`backend/.env.example` 和代码默认模型名可能与账号可用模型不同，因此需要核对 `LLM_MODEL`。

首次验证可保留图片生成和图库密钥为空，出现占位图属于降级结果。`.env` 已被 Git 忽略，密钥只保存在本地。对外部署前应单独配置 `JWT_SECRET` 等环境参数。

```powershell
uv run alembic upgrade head
uv run alembic current
```

本次迁移头为 `e5f6a7b8c9d0 (head)`。没有新增迁移日志且显示 `head`，表示当前表结构已经是该版本。

## 安装前端

先在运行 Vite 的终端按 `Ctrl+C` 停止服务，再重新安装依赖。Windows 会锁定被运行进程加载的原生模块文件。

```powershell
# 当前位于 backend
cd ../frontend
npm ci
npm run build
```

`npm ci` 按锁文件安装依赖，`npm run build` 检查 TypeScript 并生成静态文件。构建命令结束后还需要单独启动开发服务器。

## 启动三个服务

终端一进入 `backend`：

```powershell
uv run uvicorn app.main:app --reload --host 127.0.0.1 --port 39800
```

看到 `Application startup complete` 表示 API 启动完成。

终端二进入 `backend`：

```powershell
uv run arq app.worker.settings.WorkerSettings
```

看到 `Starting worker for 2 functions: generate_outline, generate_deck` 以及 Redis 版本信息，表示 Worker 已连接队列。

终端三进入 `frontend`：

```powershell
npm run dev
```

浏览器打开前端地址，并访问健康检查：

```text
http://127.0.0.1:39173
http://127.0.0.1:39800/api/v1/health
```

正常健康检查返回：

```json
{"status":"ok","database":"ok","redis":"ok"}
```

修改 `.env` 后重启 API 和 Worker，使两个进程重新读取配置。Python API 与 Java API 共用端口，本指南运行 Python 服务。

## 第一次生成试验

1. 注册账号并创建 5 页项目，选择中等文字量和灵活排版。
2. 输入一段包含明确事实的材料，生成大纲并检查页数、标题、要点和来源覆盖。
3. 确认大纲，生成页面；观察前端进度及 Worker 的 `generate_deck` 日志。
4. 修改一页内容，查看修改预览后应用；必要时重新排布。
5. 打开导出质量报告，处理 error，记录 warning，再下载 PPTX。
6. 在 PowerPoint 或 WPS 中检查换行、位置与原生对象编辑能力。

ARQ 日志中的 `delayed=...` 表示任务排队延迟；结束行中的耗时是该次任务执行时间。任务函数返回后仍需检查网页中的页面状态，确认是否存在失败页。

## 自动化验证

默认 Docker Compose 环境下，可以在项目根目录使用辅助脚本运行隔离测试：

```powershell
./scripts/dev.ps1 test
```

该操作创建或复用 `aippt_test` 数据库、使用 Redis 的 15 号逻辑库，迁移后执行测试。开发环境继续使用 `aippt` 数据库和 Redis 的 0 号库。直接运行 `pytest` 会读取当前配置，因此先确认数据库目标。

下载度量字体并运行固定语料回归，在 `backend` 目录执行：

```powershell
uv run python scripts/fetch_fonts.py
uv run python scripts/run_regression.py
uv run ruff check .
```

服务运行后，冒烟脚本默认只检查健康状态：

```powershell
uv run python scripts/smoke_test.py
```

需要真实模型试验时显式加 `--generate`。它会创建独立测试账号和 5 页项目，执行生成、SSE、过期 revision 拒绝与导出检查，消耗模型额度。报告、账号信息和文件保存在被 Git 忽略的 `tmp/` 中。

## 常见问题

| 现象 | 原因与处理 |
| --- | --- |
| `npm ci` 报 Rolldown `.node` 文件 `EPERM unlink` | 先停止已运行的 Vite，再执行 `npm ci`；本次实际通过该方式恢复安装 |
| 端口已占用 | 检查已有服务终端，使用 `Ctrl+C` 停止需要重启的进程 |
| 生成进度停在 0% | 检查 Worker 是否启动并连接正确的 Redis；API 启动不会自动启动 Worker |
| API 出现 `422` | 查看页面校验提示或响应 detail，核对邮箱、密码和项目参数 |
| 大纲或页面生成失败 | 检查网页错误、Worker 日志、模型密钥、模型名和网络；不要仅凭 API 健康检查判断模型可用 |
| 出现占位图 | 图片服务或图库未配置、请求失败时可能降级，文字生成与图片凭证分开配置 |
| 导出显示 warning | 记录对应页，允许继续导出后核对；存在 error 时必须先处理 |
| Vite 提示大文件 | 构建已通过，后续可评估路由懒加载和图表依赖拆分 |

只读检查前后端端口：

```powershell
Get-NetTCPConnection -State Listen |
    Where-Object { $_.LocalPort -in 39800,39173 } |
    Select-Object LocalAddress,LocalPort,OwningProcess
```

依赖安全提示使用 `npm audit` 查看具体包及修复范围，再决定升级方案。Docker 生产编排与 Nginx 配置已在仓库中提供；本次验证范围为本地环境。
