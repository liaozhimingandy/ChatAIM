# Docker 部署指南（Docker Deployment）

> 适用：本地 Windows 开发环境 ｜ 编排：官方 `docker-compose.yml`（未做任何修改）

## 一、架构

官方 compose 定义 6 个服务 + 2 个卷（`pgdata2`、`librechat-data`），全部使用官方镜像，不本地构建。

```
浏览器 → localhost:3080
           └─ api (LibreChat 主应用)
                ├─ mongodb (chat-mongodb:27017)      # 会话/消息/用户
                ├─ meilisearch (7700, 内部)           # 消息全文搜索
                ├─ vectordb (pgvector)               # RAG 向量
                │    └─ rag_api (8000, 内部)
                └─ admin-panel (内部)
```

## 二、启动步骤

```powershell
# 1. 确保 Docker Desktop 引擎已运行（见 development-environment.md 的 wsl.exe 说明）

# 2. 创建环境文件（首次）
Copy-Item .env.example .env

# 3. 启动全部服务
docker compose up -d

# 4. 查看状态与日志
docker compose ps
docker compose logs -f api
```

## 三、验证清单

| 步骤 | 方法 | 预期 |
|---|---|---|
| 容器状态 | `docker compose ps` | 6 个服务均为 running/healthy |
| MongoDB | `docker compose exec mongodb mongosh --eval "db.version()"` | 8.0.x |
| 前端访问 | 浏览器 `http://localhost:3080` | 出现登录页 |
| 管理员初始化 | 注册页面创建首个账号 | 首个注册用户自动获得管理员角色 |
| 登录 | 使用注册账号登录 | 进入聊天界面 |
| 聊天功能 | 配置至少一个模型源后发起对话 | 流式回复正常 |

## 四、管理员说明

LibreChat 默认开放注册，**首个注册账号即管理员**（可通过 `librechat.yaml` 的 `registration` 段控制后续注册策略）。管理员面板走独立的 admin-panel 服务入口。

## 五、常用运维命令

```powershell
docker compose stop                 # 停止（保留卷）
docker compose down                 # 移除容器（保留卷）
docker compose down -v              # 连同数据卷一并清除（危险，数据丢失）
docker compose pull && docker compose up -d   # 升级到最新镜像
```

## 六、注意事项

1. **不要修改官方 compose 架构**。如需本地覆盖（改端口等），使用 `docker-compose.override.yml`（模板见 `docker-compose.override.yml.example`），它会被 compose 自动加载且不入库
2. `.env` 中的密钥（JWT_SECRET 等）为空时镜像会自动生成并写回 `.env`——确认该文件不入 git
3. 数据持久化：MongoDB/Meilisearch/pgvector 数据均在 named volumes，`down -v` 会清空全部数据
4. 升级上游版本后重新 `docker compose pull` 即可镜像同步（应用代码打包在镜像内，本地源码仅用于开发）
5. 模型源（OpenAI/Anthropic/NewAPI 等）在界面设置或 `librechat.yaml` 中配置；`.env` 只放框架级变量
