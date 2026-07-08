---
title: 为什么选 Fumadocs：Node.js 项目接入开箱即用文档站
date: 2026-07-08 23:30:00
description: 给 Midway.js 后端服务搭文档站的选型过程：为什么在 Vitepress / Docusaurus 里最终选了 Fumadocs，它开箱即用的颜值从何而来，fumadocs-openapi 如何自动从 OpenAPI spec 生成接口文档，以及怎么用 Next.js 静态导出 + egg-static 把它嵌进 Midway 项目、一个命令同时跑服务端和文档站。
categories:
  - [AI Infra]
tags:
  - Fumadocs
  - Node.js
  - Midway.js
  - OpenAPI
  - 文档站
  - Next.js
  - Tailwind CSS
cover: /images/fumadocs-cover.webp
---

后端服务到了一定规模，文档就成了刚需。不是给外部用户的 README，而是给自己人的：新成员怎么跑起来、接口入参是什么、数据库里几张表是怎么关联的。

我负责的这个 Node.js 后端服务跑着一套自动化平台，随着接口越来越多，"靠脑子记"已经不够了。于是我决定正经做一个内嵌文档站，要求只有一个：**好看，且不要花时间调样式**。

## 一句话总结

Fumadocs 是目前我见过最开箱即用的 Node.js 文档框架——接入 OpenAPI 两行命令就能出接口文档，颜值不需要额外调，跟 Midway.js / Egg.js 静态托管的结合也很自然。

---

## 选型过程：为什么不选其他的

我评估了以下几个方案：

| 框架 | 技术栈 | OpenAPI 支持 | 颜值开箱 | 备注 |
|---|---|---|---|---|
| **VitePress** | Vue 3 | 无官方插件 | 中等 | 适合纯文档，定制自由度高但要花时间 |
| **Docusaurus** | React | docusaurus-plugin-openapi-docs | 一般 | 生态成熟，但 OpenAPI 插件渲染效果不够精致 |
| **Nextra** | Next.js | 有，需要配置 | 中等 | 和 Next.js 绑定深，学习成本适中 |
| **Mintlify** | SaaS | 一流 | 最好看 | 需要联网托管，私有项目不适合 |
| **Fumadocs** | Next.js | fumadocs-openapi，官方维护 | **一流** | 可自部署，颜值对标 Mintlify |

Mintlify 的 OpenAPI 渲染是业内标杆，但它是 SaaS——文档得推到他们的服务器，对内部项目不合适。

Fumadocs 是我找到的唯一一个**可以自部署、OpenAPI 一流、不需要调样式就好看**的方案。

---

## Fumadocs 的颜值从何而来

### 设计系统内置

Fumadocs UI 本身是 Tailwind CSS v4 驱动的设计系统，所有颜色、排版、间距都基于 CSS 变量。导入一行 CSS 就能用：

```css
/* app/globals.css */
@import 'fumadocs-ui/style.css';
@import 'tailwindcss';
```

不需要写主题配置，深色模式自动支持，字体和间距比例都对。

### 文档布局开箱可用

侧边栏、TOC（目录）、面包屑、代码高亮——这些在大多数框架里都要自己拼，Fumadocs 全部内置：

```tsx
// app/docs/layout.tsx
import { DocsLayout } from 'fumadocs-ui/layouts/docs';
import { source } from '@/lib/source';

export default function Layout({ children }) {
  return (
    <DocsLayout tree={source.pageTree} nav={{ title: '我的项目' }}>
      {children}
    </DocsLayout>
  );
}
```

这就是文档布局的全部代码。侧边栏自动从 `content/docs/` 的目录结构生成，`meta.json` 控制排序和分组。

### MDX 组件

`<Cards>`、`<Card>`、`<Callout>`、`<Steps>` 这些常见 MDX 组件全部内置，写文档时直接用，不需要自己实现。

---

## fumadocs-openapi：两行命令出接口文档

这是我最看重的功能。维护一份 `openapi.yaml`，然后：

```bash
# 安装
npm install fumadocs-openapi

# 从 YAML 生成 MDX 页面
npx fumadocs-openapi generate -i openapi/my-api.yaml -o content/docs/api
```

生成的每个接口都是一个独立的 MDX 页面，包含：

- 请求方法 + 路径（带颜色标签）
- 路径参数 / Query 参数 / 请求体的字段说明和类型
- 响应示例，支持多状态码
- 交互式 "Try it" 面板（可选）

效果对标 Mintlify 和 Swagger UI Pro，但完全自部署。

OpenAPI spec 的骨架是这样的：

```yaml
# openapi/my-api.yaml
openapi: 3.0.3
info:
  title: My API
  version: 1.0.0
paths:
  /api/skills:
    get:
      summary: 获取技能列表
      tags: [Skills]
      parameters:
        - name: projectKey
          in: query
          schema:
            type: string
      responses:
        '200':
          description: 技能列表
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Skill'
```

写一次 spec，既能生成文档，将来也能用于接口测试和 Mock Server——比手写 Markdown 维护成本低。

---

## 怎么和 Midway.js / Egg.js 结合

这里有个有意思的架构问题：文档站是 Next.js 项目，服务端是 Midway.js（Egg.js 底层）。两个进程，怎么变成"一个项目"？

### 静态导出 + egg-static 托管

Next.js 支持 `output: 'export'`，构建后输出纯静态文件：

```js
// client/next.config.mjs
const config = {
  output: 'export',      // 输出静态 HTML/JS/CSS，不需要 Node.js 运行时
  images: { unoptimized: true },
};
```

Egg.js 内置 `egg-static` 插件，开启后可以托管静态目录：

```ts
// src/config/config.default.ts
import { join } from 'path';

static: {
  prefix: '/',
  dir: [join(appInfo.appDir, 'client', 'out')],  // Next.js 导出目录
  dynamic: true,
  preload: false,
},
```

```ts
// src/config/plugin.ts
static: {
  enable: true,
  package: 'egg-static',
},
```

请求流程：

```
HTTP 请求
  ↓
egg-static（命中 client/out/ 静态文件 → 直接返回 HTML/JS/CSS）
  ↓
Midway 路由（没有匹配的静态文件 → 走 API 控制器）
```

也就是说，`GET /` 返回文档站首页，`GET /api/skills` 走 Midway 控制器——静态文件和 API 共用同一个端口，完全无感。

### 本地开发：一个命令同时跑两个进程

生产环境用静态导出。本地开发时，Next.js 需要一个 dev server 做热更新，不能用静态文件。用 `concurrently` 解决：

```json
// package.json（项目根目录）
{
  "scripts": {
    "dev": "concurrently --names 'server,docs' --prefix-colors 'cyan,magenta' --kill-others-on-fail 'npm run dev:server' 'npm run dev:docs'",
    "dev:server": "cross-env NODE_ENV=local mwtsc --watch --run @midwayjs/mock/app.js",
    "dev:docs": "npm --prefix client run dev"
  }
}
```

`npm run dev` 之后，终端里：
- `cyan` 前缀的输出是 Midway 后端（`:7001`）
- `magenta` 前缀的输出是 Fumadocs（`:3000`）

Home controller 做一个简单的判断：本地环境没有 `client/out/`，就重定向到 `:3000`；其他环境如果 build 缺失就显示一个保底状态页。

```ts
// src/controller/home.ts
const CLIENT_INDEX = join(__dirname, '../../client/out/index.html');

@Get('/')
async home() {
  if (existsSync(CLIENT_INDEX)) {
    this.ctx.type = 'html';
    return readFileSync(CLIENT_INDEX, 'utf-8');
  }
  if (this.ctx.app.config.env === 'local') {
    this.ctx.redirect('http://localhost:3000');
    return;
  }
  // 返回保底状态页
}
```

### 目录结构

```
my-server/
├── src/              # Midway.js 后端
│   ├── controller/
│   ├── service/
│   └── model/
├── client/           # Fumadocs 文档站（Next.js）
│   ├── app/
│   ├── content/docs/ # MDX 文档内容
│   ├── openapi/      # OpenAPI spec
│   └── out/          # next build 产物（被 egg-static 服务）
└── package.json      # dev/build 统一入口
```

---

## 版本对齐的坑

Fumadocs 各包的 peer dependency 比较严格，截至写作时：

| 包 | 版本 | 备注 |
|---|---|---|
| `fumadocs-core` | `^16.9.0` | |
| `fumadocs-mdx` | `^15.0.0` | 注意不是 ^12，v15 才支持 core v16 |
| `fumadocs-ui` | `^16.0.0` | |
| `fumadocs-openapi` | `^10.0.0` | 要求 core ^16.9、ui ^16.9 |
| `tailwindcss` | `^4.0.0` | v16 ui 内置 Tailwind v4，不再用 v3 preset |

`fumadocs-mdx` 对 Next.js 16 有 optional peer dep，安装时加 `--legacy-peer-deps` 即可，不影响 Next.js 15 运行。

CSS 配置从 Tailwind v3 的 `tailwind.config.ts` 改成 v4 的 `@import` 方式：

```css
@import 'fumadocs-ui/style.css';  /* Fumadocs 设计系统 + Tailwind v4 */
@import 'tailwindcss';             /* 补充 utility classes */
@source '../app/**/*.{ts,tsx}';
@source '../content/**/*.{md,mdx}';
```

---

## 要不要用 / 我的判断框架

**适合用 Fumadocs 的场景：**
- 需要 OpenAPI 接口文档，且对渲染效果有要求
- 项目本身是 Node.js，可以把文档站代码一起放在 repo 里
- 团队熟悉 React / Next.js（文档站本身就是 Next.js 项目）
- 不想花时间在文档样式上，要开箱即用

**不适合的场景：**
- 项目是纯静态网站 / 博客，VitePress 更轻量
- 团队技术栈是 Vue，Vitepress 更顺手
- 需要公开托管 + 自定义域名，Mintlify 的 SaaS 体验更好
- 对 Next.js 包体积有严格限制（文档站打出来还是有一定体积的）

**我的判断**：值得上。特别是如果你的项目已经在写 OpenAPI spec，fumadocs-openapi 自动生成这一步能省很多时间，比手写 Markdown 维护成本低，比 Swagger UI 好看一个量级。
