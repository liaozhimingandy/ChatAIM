# ChatAIM 项目总览（Project Overview）

> 更新日期：2026-11-24 ｜ 基线：LibreChat upstream/main @ `4700c44eb`

## 一、项目定位

ChatAIM 是基于 [LibreChat](https://github.com/LibreChat-AI/LibreChat) 官方开源项目建立的独立代码仓库（Fork：[liaozhimingandy/ChatAIM](https://github.com/liaozhimingandy/ChatAIM)）。

**目标：**
1. 完整保留 LibreChat 原生聊天功能
2. 建立独立的代码维护体系（develop 分支工作流）
3. 支持同步 LibreChat 官方上游更新
4. 本地 Windows 环境可正常开发
5. Docker Compose 可完整启动
6. 后续支持前端独立构建与 GitHub Pages 自动部署
7. 后续接入 Also Account 与 NewAPI

**当前阶段原则：**
- 不修改 LibreChat 核心业务代码、不删除原有功能、不改 MongoDB 数据结构
- 不引入 Vue/TDesign 改造 React UI
- 所有自定义修改必须可追踪（独立分支 + 独立目录 `docs/also-ai/`）
- 不提交 `.env`、API Key、密码等敏感信息

## 二、当前仓库状态

| 项目 | 状态 |
|---|---|
| origin | `git@github.com:liaozhimingandy/ChatAIM.git` |
| upstream | `https://github.com/LibreChat-AI/LibreChat.git` |
| main 分支 | 与 upstream/main 完全同步（0 差异） |
| develop 分支 | 已从 main 建立，推送至 origin |
| 本地 .env | 已从 `.env.example` 复制创建（不提交） |
| 工作树 | 干净，无未提交修改 |

## 三、上游分支用途

- **upstream/main**：LibreChat 发布分支，保持 fast-forward 同步，不做 backport PR
- **upstream/dev**：官方开发分支，PR 合入目标，当前领先 main 184 个提交
- **ChatAIM 分支策略**：
  - `main` 跟随 upstream/main（发布基线）
  - `develop` 自有开发分支，自定义功能（Also Account、NewAPI 等）在此累积
  - 自定义功能分支从 `develop` 拉出，PR 目标为 `develop`

## 四、仓库结构速览

```
ChatAIM/
├── api/                # 旧版 Express 装配层（CJS，仅路由注册，不写业务）
├── packages/
│   ├── api/            # 新版后端业务逻辑（TypeScript，新行为一律写这里）
│   ├── chat/           # 聊天消息组件库（@librechat/chat，不反向依赖应用）
│   ├── client/         # 共享 UI 原语（主题、字体、组件）
│   ├── data-provider/  # 前后端共享 API 类型与服务（librechat-data-provider）
│   ├── data-schemas/   # 数据库方法层（Mongoose 边界收敛在此）
│   └── channel-gateway/# Agent Channel 网关
├── client/             # React 前端应用（Vite + rolldown）
├── config/             # 运行时配置辅助
├── docker-compose.yml  # 官方编排（api/admin-panel/mongodb/meilisearch/vectordb/rag_api）
├── docs/also-ai/       # ChatAIM 自有技术文档（本目录）
└── .env.example        # 官方环境变量模板（.env 不入库）
```

## 五、里程碑规划

| 阶段 | 内容 | 状态 |
|---|---|---|
| 一 | 源码初始化、origin/upstream 配置、develop 分支 | ✅ 完成 |
| 二 | 环境检查（Git/Node/npm/Docker） | ✅ 完成 |
| 三 | Docker Compose 完整启动验证 | ⏸ 受阻（见 development-environment.md） |
| 四 | 前端独立构建研究与实际构建 | ✅ 完成（见 frontend-build.md） |
| 五 | GitHub Actions 自动化设计 | 📋 设计完成（见 github-pages-deployment.md） |
| 六 | Also Account / NewAPI 接入 | 未开始 |
