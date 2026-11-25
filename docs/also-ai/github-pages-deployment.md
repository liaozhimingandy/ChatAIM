# GitHub Pages 部署方案（GitHub Pages Deployment）

> 2026-11-24 ｜ 基于源码级分析（构建验证待磁盘释放后补做）

## 一、可行性结论

**GitHub Pages 可以托管前端静态产物，但不能"开箱即用"直接工作。**

| 环节 | GH Pages 直接托管 | 说明 |
|---|---|---|
| 静态资源（js/css/字体/图标） | ✅ 可行 | `base: ''` 相对引用，`*.github.io/chataim` 子路径可用 |
| SPA 深链接路由 | ✅ 可行（需 1 个技巧） | 复制 `index.html` → `404.html` 实现回退 |
| PWA Service Worker | ⚠️ 需验证 | `navigateFallback: null` 无 SPA 回退需求；`useCredentials: true` 跨域下行为需回归 |
| **API 调用** | ❌ 不可行 | 所有请求 `${BASE_URL}/api/*` 同源相对路径 → 打到 github.io 域名 404 |
| 认证 Cookie | ❌ 需后端配合 | 跨域需 `SameSite=None; Secure` + CORS credentials |
| 服务端 index.html 运行时改写 | ❌ 丧失 | 主题/语言注入退化为静态兜底，影响小 |

**判定：需要最小改造方案（约 3 项、核心改动 < 20 行），改造后前端可在 GH Pages 独立托管，后端另行部署（Docker 服务器 / 云主机）。**

## 二、最小修改方案

### 改造 1：API 地址可配置（核心，~5 行）

`packages/data-provider/src/api-endpoints.ts` 现状读取 `<base>` 元素。改为：

```ts
let BASE_URL = '';
if (typeof import.meta !== 'undefined' && import.meta.env?.VITE_API_BASE_URL) {
  BASE_URL = import.meta.env.VITE_API_BASE_URL.replace(/\/$/, '');   // 新增
} else if (typeof process === 'undefined' || process.browser === true) {
  BASE_URL = document.querySelector('base')?.getAttribute('href') || '/';
}
```

构建时 `VITE_API_BASE_URL=https://api.chataim.example.com` 注入（`envPrefix` 已含 `VITE_`，`envDir` 已指向仓库根，无需改 vite 配置）。

> 这是本方案唯一触碰核心包的改动，单文件 ~5 行，upstream 合并冲突面极小。

### 改造 2：404.html 回退（构建后处理，0 行代码）

GitHub Pages 不支持服务端 rewrite。CI 中构建后：

```bash
cp client/dist/index.html client/dist/404.html
```

`createBrowserRouter` 深链接（如 `/chat/xxx`）刷新时由 404.html 接管，React Router 启动后自愈到正确路由。

### 改造 3：后端 CORS（后端侧配置，不动前端）

后端部署域名与 GH Pages 不同源，需要：
- 允许来源 `https://liaozhimingandy.github.io`
- `Access-Control-Allow-Credentials: true`
- Cookie 属性 `SameSite=None; Secure`（HTTPS 前提下）
- 确认官方是否已有 CORS 配置项；没有则需在 `packages/api` 加一个 configSchema 驱动的 CORS 中间件（可配置，符合官方"新配置项进 configSchema"纪律）

### 明确不做的事

- ❌ 不用 HashRouter 重写路由（破坏兼容）
- ❌ 不把任何后端密钥、JWT、数据库连接写入静态前端
- ❌ 不改 vite.config.ts 的分包/插件体系

## 三、GitHub Actions 工作流设计

**两阶段：先验证（build-only），稳定后再发布（deploy）。**

### 工作流 A：`frontend-build.yml`（验证，先上线）

```yaml
name: Frontend Build Check
on:
  push: { branches: [main, develop], paths: ['client/**', 'packages/**'] }
  pull_request: { branches: [develop] }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version-file: '.nvmrc', cache: 'npm' }   # Node 24
      - run: npm ci                      # 固定版本依赖（lockfile）
      - run: npm run build:data-provider
      - run: cd client && npm run build
      - uses: actions/upload-artifact@v4 # 产物校验
        with: { name: frontend-dist, path: client/dist }
      - name: Verify output              # 构建结果验证，失败不发布
        run: |
          test -f client/dist/index.html || exit 1
          test -d client/dist/assets || exit 1
```

### 工作流 B：`frontend-deploy.yml`（发布，验证工作流稳定后启用）

```yaml
name: Deploy Frontend to GitHub Pages
on:
  push: { branches: [main] }            # main 分支提交自动触发
permissions: { contents: read, pages: write, id-token: write }
concurrency: { group: pages, cancel-in-progress: true }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version-file: '.nvmrc', cache: 'npm' }
      - run: npm ci
      - run: npm run build:data-provider
      - run: cd client && npm run build
        env:
          VITE_API_BASE_URL: ${{ vars.VITE_API_BASE_URL }}  # 仓库 Variables 配置，非 Secret
      - run: cp client/dist/index.html client/dist/404.html
      - name: Verify output             # 构建失败不发布
        run: test -f client/dist/index.html && test -d client/dist/assets
      - uses: actions/upload-pages-artifact@v3
        with: { path: client/dist }
  deploy:
    needs: build                        # build 失败则不部署
    runs-on: ubuntu-latest
    environment: { name: github-pages }
    steps:
      - uses: actions/deploy-pages@v4
```

**安全要点：**
- `VITE_API_BASE_URL` 是公开的后端地址，放 repo **Variables**（非 Secrets 也无妨，但绝不放任何密钥）
- 后端 API Key / JWT_SECRET / 数据库凭据永远只在后端服务器 `.env`，静态前端无密钥
- `concurrency` 防并发部署；`needs: build` 保证构建失败不发布

## 四、落地顺序

1. ✅ 工作流已上线并首次部署成功（2026-11-25，Run #1 全绿，构建+部署约 2 分钟）
2. ✅ 自定义域名 chataim.cn 已启用（工作流内固化 CNAME 文件，commit cb41c17bf）
3. ⏳ DNS 配置：chataim.cn A 记录 → 185.199.108.153 / 185.199.109.153 / 185.199.110.153 / 185.199.111.153，www.chataim.cn CNAME → liaozhimingandy.github.io
4. ⏳ DNS 生效后 GitHub 自动签发 HTTPS 证书（Settings → Pages 可见进度）
5. ⏳ 待办：`VITE_API_BASE_URL` 改造 PR → 后端 CORS → 端到端验证登录/聊天/上传

> 注意：仓库 Pages 已配置自定义域名后，`liaozhimingandy.github.io/ChatAIM` 会 301 到 chataim.cn，这是 GitHub 官方行为。

## 五、域名访问路径

| 地址 | 说明 |
|---|---|
| https://chataim.cn | 主域名（DNS 配置后生效） |
| https://www.chataim.cn | www 子域（可选，CNAME 指向 liaozhimingandy.github.io） |
| https://liaozhimingandy.github.io/ChatAIM | 默认域（配置自定义域名后 301 到 chataim.cn） |
