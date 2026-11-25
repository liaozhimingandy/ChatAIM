# 开发路线图（Development Roadmap）

> 2026-11-24 制定

## 阶段总览

```
✅ 阶段一：仓库初始化
✅ 阶段二：环境检查
⏸ 阶段三：Docker Compose 完整启动验证      ← 受阻：wsl.exe 黑名单（见 development-environment.md）
✅ 阶段四：前端独立构建研究与实际构建
📋 阶段五：GitHub Actions 自动化设计（方案就绪，待阶段三验证后启用）
⬜ 阶段六：Also Account / NewAPI 接入
⬜ 阶段七：套餐、充值、支付、用户运营（暂缓，明确不在本阶段开发）
```

## 近期待办（按优先级）

### P0 — 解除 Docker 阻碍，完成阶段三验收
1. 用户在安全中心移除 `wsl.exe` 黑名单（或手动启动 Docker Desktop）
2. `docker compose up -d` 启动 6 服务
3. 验证 MongoDB → 前端访问 → 首个账号注册（管理员）→ 登录 → 聊天
4. 验收标准：完整走通"登录 + 聊天"

### P1 — 前端独立构建落地
1. 确认 `client/dist` 产物已在 Node 22 下构建成功（已完成，见 frontend-build.md）
2. 实现 GitHub Pages 最小改造方案（见 github-pages-deployment.md，预计 ~20 行核心改动）
3. 建立独立后端的跨域 CORS 配置（后端侧）

### P2 — GitHub Actions 发布流水线
1. 先建验证工作流（build-only，不发布）：PR/push 触发，构建 + 产物校验
2. 稳定后加 Pages 部署工作流：main 提交触发，构建失败不发布
3. Node 版本锁 `.nvmrc`（24.x），不把任何后端密钥写入静态前端

### P3 — 上游同步机制运转
1. 每周 `fetch upstream` 一次，main fast-forward、develop merge
2. 首次同步时观察冲突面，校准"自定义代码隔离纪律"

## 中期（阶段六）

- **Also Account 接入**：OIDC/OAuth 对接（LibreChat 原生支持 OIDC 提供方，优先走 `librechat.yaml` 的 `openid` 配置而非改代码）
- **NewAPI 接入**：作为 OpenAI 兼容端点接入（LibreChat 支持 custom endpoint + baseURL），零代码改造即可用；如需品牌定制再考虑 configSchema 扩展

## 长期（暂缓）

- 套餐/充值/支付/用户运营——明确暂不开发；届时评估是否以独立服务 + LibreChat 数据库视图方式实现，避免侵入核心

## 约束清单（全程有效）

1. 不修改核心业务代码；修改必须可追踪
2. 不删除原有功能、不重写聊天 API、不改 MongoDB 结构
3. 不引入 Vue/TDesign 改造 React UI
4. 保持与上游兼容，同步优先
5. 不提交 .env / API Key / 密码等敏感信息
