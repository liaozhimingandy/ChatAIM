# 前端独立构建研究（Frontend Build）

> 2026-11-24 ｜ 基线 upstream/main @ `4700c44eb`

## 一、结论先行

| 问题 | 结论 |
|---|---|
| 前端能否独立于 Node 后端构建？ | ✅ 可以。`client` 是标准 Vite SPA，产物为纯静态文件 |
| 构建入口 / 输出 | `client/package.json` 的 `build`（vite build）→ `client/dist/` |
| 构建是否已实际执行？ | ⚠️ **被磁盘空间阻碍**（详见第五节），架构结论全部来自源码级分析，构建验证待磁盘释放后补做 |
| GitHub Pages 能否直接托管？ | ⚠️ 半可行：静态资源可托管，但 **API 调用会 404**，需最小改造（见 github-pages-deployment.md） |

## 二、构建体系（源码级分析）

```
根 package.json
├── build:data-provider → packages/data-provider 构建（前置，client 依赖其产物）
└── build → npx turbo run build（全 workspace）

client/package.json
└── build = cross-env NODE_ENV=production NODE_OPTIONS=--max-old-space-size=8192 vite build
```

- Vite **8.3.5 + rolldown**（非传统 rollup，`build.rolldownOptions` 分包）
- `vite.config.ts` 关键配置：
  - `base: ''` —— 资源相对引用，可部署在任意子路径
  - `outDir: './dist'`
  - `envDir: '../'` + `envPrefix: ['VITE_', 'SCRIPT_', 'DOMAIN_', 'ALLOW_', 'REACT_APP_THEME_']`——构建变量从仓库根 `.env` 读取
  - minify: oxc，gzip+brotili 预压缩，PWA（vite-plugin-pwa）
  - `copyPublicAssets` 插件在构建内复制 `public/assets`（图标/logo）与字体许可证（生产期 `publicDir: false`，不能靠后处理脚本）
  - `lazyChunkGuard` 构建期校验懒加载泄漏，mermaid/katex/monaco/sandpack 等重依赖全部独立懒加载 chunk

**产物结构（预期）：**
```
client/dist/
├── index.html          # 含 boot script（语言/主题检测）
├── assets/             # 哈希命名 js/css/字体/图标
├── robots.txt
├── sw.js + sw-heal.js  # PWA service worker
└── *.gz / *.br         # 预压缩资源
```

## 三、前端运行时对后端的依赖（关键）

1. **API 地址**（`packages/data-provider/src/api-endpoints.ts`）：
   ```ts
   BASE_URL = document.querySelector('base')?.getAttribute('href') || '/';
   // 所有端点 = `${BASE_URL}/api/...`
   ```
   默认**同源相对路径**。GitHub Pages 域名下 `/api/*` 必然 404 ——这是静态托管的核心矛盾。

2. **运行时配置**：应用启动拉 `/api/config` 获取端点列表、标题、接口主题等。静态托管下需后端可达且允许跨域。

3. **认证**：Cookie 会话 + `/api/refreshToken` 轮换。跨域部署要求后端 Cookie `SameSite=None; Secure` + CORS `credentials: true`。

4. **SSE**：`fetch` 流式（`text/event-stream`），同源直连；跨域下走 CORS 即可，无需 EventSource。

5. **文件上传/下载**：`POST/GET /api/files`，跨域同上。

6. **index.html 服务端改写**：官方后端按请求注入 subpath/语言/主题（源码注释 "LibreChat mutates index.html per request"）。静态托管后该能力退化为静态 boot script 兜底，功能影响有限但需回归测试。

## 四、构建执行步骤（磁盘释放后照此操作）

```bash
npm install                      # 全 workspace 依赖（约 3–5GB 磁盘、10–20 分钟）
npm run build:data-provider      # 前置包
cd client && npm run build       # 前端产物 → client/dist/
```

验证产物：`npx serve client/dist` 起静态服务，确认页面加载（接口报错属预期，无后端）。

## 五、本轮实际执行记录

| 步骤 | 结果 |
|---|---|
| `npm install`（第 1 次，7 分钟） | ⚠️ 表面完成但 `rimraf` 等二进制缺失（沙箱 safe-delete 干扰 tar 解包） |
| `npm install`（重试，23 分钟） | ❌ **ENOSPC：磁盘空间不足**，且把首次的 node_modules 清空 |
| `npm run build:data-provider` | ❌ 失败于 clean（rimraf 缺失），根因同上 |

**磁盘现状（清理后）**：C 剩 0.12 GB / D 剩 1.76 GB。LibreChat 依赖约需 3–5GB。
**大头（已扫描报告，未动个人文件）**：微信数据 107GB（D:\data\WeChat 56GB + xwechat_files 51GB）、Docker vhdx 38GB（C 盘）、D:\temp 7.4GB（含 2.5GB 电影、1.2GB 安装包等）。

**待磁盘释放后重跑第四节步骤即可完成实际构建验证。**
