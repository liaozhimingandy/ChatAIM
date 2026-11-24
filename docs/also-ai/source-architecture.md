# 源码架构分析（Source Architecture）

> 基于 upstream/main @ `4700c44eb`（2026-11-24）

## 一、Monorepo 总体结构

npm workspaces 管理，turbo 编排构建（根 `package.json` 的 `build` = `npx turbo run build`）。

```
client/                 # React 前端应用（Vite/rolldown，产出静态 SPA）
packages/
├── api/                # 后端业务逻辑（TypeScript, tsdown 构建）
├── chat/               # 聊天消息组件库（@librechat/chat，独立 npm 包形态）
├── client/             # 共享 UI 原语：主题 token、字体、通用组件
├── data-provider/      # 前后端共享 API 层（类型 + 请求封装 + 端点定义）
├── data-schemas/       # 数据库方法层（Mongoose 全部收敛在此）
└── channel-gateway/    # Agent Channel 网关（Slack webhook 等）
api/                    # 旧版 Express 装配（CJS）——只做 require/路由注册，
                        # 任何新业务逻辑都必须写进 packages/api
```

## 二、关键边界规则（来自官方 AGENTS.md）

1. **`/api` 只放装配**：分支、辅助函数、校验等行为一律进 `packages/api`（TS）
2. **数据库契约归 `packages/data-schemas`**：`FilterQuery`/`Types.ObjectId` 等 Mongoose 类型不得泄漏到 `packages/api`、`data-provider`、`client` 的导出签名
3. **前后端共享逻辑归 `packages/data-provider`**，用 `npm run build:data-provider` 构建根目录
4. **新配置项必须进 `configSchema`**（`packages/data-provider/src/config.ts`），可在 `librechat.yaml` 设置，默认值复现当前行为
5. **依赖注入**：模块接收配置/数据库方法/客户端，不 import 应用单例；集成（SDK、存储、向量库）通过调用方提供的接口进入
6. `packages/api`、`packages/client`、`packages/chat`、`packages/data-schemas` 用 tsdown 构建——**只产出不查类型**，必须另跑 `npx tsc --noEmit`

## 三、前端请求链路

**API 地址解析（`packages/data-provider/src/api-endpoints.ts`）：**

```ts
let BASE_URL = '';
if (browser) {
  BASE_URL = document.querySelector('base')?.getAttribute('href') || '/';
}
// 所有端点 = `${BASE_URL}/api/...` —— 同源相对路径
```

- **开发模式**：Vite dev server（3090）把 `/api`、`/oauth` 代理到 `http://localhost:3080`（`client/vite.config.ts` server.proxy）
- **生产模式**：Express 同源托管 `client/dist`，前端用相对路径直连同源后端
- **要点**：默认**没有任何绝对 API 地址注入机制**——跨域部署（如 GitHub Pages）需要最小改造（见 github-pages-deployment.md）

**认证：**
- 登录后 Cookie 会话 + `refreshToken` 端点轮换（`packages/data-provider/src/request.ts`）
- 请求封装基于原生 `fetch`，401 后自动刷新重试
- OIDC/OAuth 登录走 `/oauth` 路由（开发期同样由 Vite 代理）

**SSE：**
- 流式回复走 `text/event-stream`（`packages/data-provider/src/upload.ts` 等封装）
- SSE 为同源 HTTP 流，静态托管无影响；**前提是 API 地址可达**

**文件上传/下载：**
- 上传：`POST /api/files`（multipart，见 `packages/data-provider/src/upload.ts`）
- 下载：`GET /api/files/{fileId}` 同源流式
- 跨域部署时上传下载同样依赖 API 地址可达 + 服务端 CORS

## 四、SPA 路由与静态托管相关机制

| 机制 | 位置 | 对静态托管的影响 |
|---|---|---|
| `createBrowserRouter` | `client/src/routes/index.tsx` | 需要 404.html 回退（GH Pages 惯用做法） |
| `base: ''` | `client/vite.config.ts` | 资源相对引用，可在任意子路径托管 |
| PWA Service Worker（vite-plugin-pwa） | `client/vite.config.ts` | `navigateFallback: null`，无需 SPA fallback SW；`useCredentials: true` 依赖同源 Cookie |
| 运行时 index.html 改写 | 服务端按请求注入主题/语言/subpath | 静态托管拿不到注入，需确认静态产物自带 boot script 兜底 |
| `<base>` 元素读取 | `data-provider/src/api-endpoints.ts` | 跨域 API 的潜在注入点 |
| 环境变量前缀 | `VITE_/SCRIPT_/DOMAIN_/ALLOW_/REACT_APP_THEME_`，`envDir: '../'` | 构建期变量从仓库根 `.env` 读取 |

## 五、构建产物形态

- `client` 构建输出 `client/dist/`：`index.html` + `assets/`（哈希命名 js/css/字体）+ `assets/*.png`（PWA 图标/logo）+ `robots.txt` + `sw.js` + `sw-heal.js` + gzip/brotli 预压缩文件
- `copyPublicAssets` 插件在构建内复制 `public/assets` 与字体许可证（publicDir 生产期关闭，必须走插件）
- 分包策略：mermaid/katex/monaco/sandpack 等重依赖全部懒加载 chunk，boot chunk 有 guard 插件防止泄漏（`lazyChunkGuard`）
