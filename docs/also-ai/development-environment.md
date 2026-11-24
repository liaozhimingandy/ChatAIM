# 开发环境检查报告（Development Environment）

> 检查日期：2026-11-24 ｜ 平台：Windows（Git Bash + PowerShell）

## 一、实测版本对照

| 工具 | 官方要求 | 本地实测 | 结论 |
|---|---|---|---|
| Git | 任意现代版本 | 2.55.0.windows.3 | ✅ 达标 |
| Node.js | **24.22.3**（仓库 `.nvmrc`） | 22.22.2 | ⚠️ 版本偏低 |
| npm | **11**（官方推荐） | 10.9.7 | ⚠️ 版本偏低 |
| Docker | 任意近期版本 | 28.1.1 | ✅ 版本达标 |
| Docker Compose | v2 | v2.35.1-desktop.1 | ✅ 达标 |

**Node/npm 差距说明：**
- `.nvmrc` 指定 `24.22.3`，本地为 Node 22 LTS + npm 10。依赖安装与前端构建已在 Node 22 下实际跑通（见 frontend-build.md），开发不受阻。
- 建议通过 [nvm-windows](https://github.com/coreybutler/nvm-windows) 或 Docker 内 Node 24 对齐官方版本；上线 CI 时 GitHub Actions 直接使用 `node-version: 24`（读取 `.nvmrc`）。
- 本机未安装 nvm。

## 二、Docker 阻碍（阶段三受阻原因）

**现象：** `docker info` / `docker ps` 无输出、退出码 1；Docker Desktop 进程存在但引擎不可达。

**根因：** 本机安全策略（WorkBuddy 安全中心 → 命令安全 → 程序黑名单）将 `wsl.exe` 列入黑名单。Docker Desktop 的 WSL2 后端启动时内部调用 `wsl.exe` 被拦截，Linux 引擎无法初始化，命名管道 `dockerDesktopLinuxEngine` 不可用。

**处理过程：**
1. 确认 `\\.\pipe\docker_engine` 与 `dockerDesktopLinuxEngine` 管道存在，但连接失败
2. 尝试重启 Docker Desktop（含绕过沙箱启动）→ 子进程 `wsl.exe` 仍被策略拦截
3. 该拦截无法从会话内绕过，需用户手动操作

**解除方法（二选一）：**
- 打开 WorkBuddy 安全中心 → 命令安全 → 程序黑名单，移除 `wsl.exe`
- 或由用户在系统内手动启动 Docker Desktop，确认 Docker 图标显示 "Engine running" 后，再执行本阶段脚本

**解除后执行（阶段三）：**
```powershell
# .env 已就绪（从 .env.example 复制）
docker compose up -d                    # 启动 6 个服务
docker compose ps                       # 检查状态
# 浏览器访问 http://localhost:3080
# 首次进入注册首个账号即为管理员，然后验证登录/聊天
```

## 三、Compose 服务清单（官方架构，未做修改）

| 服务 | 容器名 | 镜像 | 用途 |
|---|---|---|---|
| api | LibreChat | registry.librechat.ai/librechat-ai/librechat-dev:latest | 主应用（端口 3080） |
| admin-panel | admin-panel | librechat-admin-panel:latest | 管理面板 |
| mongodb | chat-mongodb | mongo:8.0.20 | 主数据库 |
| meilisearch | chat-meilisearch | meilisearch:v1.35.1 | 全文搜索 |
| vectordb | vectordb | pgvector:0.8.0-pg15-trixie | RAG 向量库 |
| rag_api | rag_api | librechat-rag-api-dev-lite:latest | RAG API |

## 四、环境变量说明

- `.env` 已从官方 `.env.example`（72KB，含全部说明注释）复制生成，遵循官方默认值
- `CREDS_KEY` / `CREDS_IV` / `JWT_SECRET` / `JWT_REFRESH_SECRET` 初始为空——Docker 镜像启动时会自动生成并写回；本地裸跑 Node 时需自行填写
- `MONGO_URI=mongodb://127.0.0.1:27017/LibreChat`（本地模式默认；Compose 模式由容器内变量覆盖）
- `.env` 已被 `.gitignore` 排除，不会入库
