# Nuxt（官网 / SEO）

只用于官网、营销页、内容站等需要搜索引擎收录与 SSR/SSG 的场景，独立放在 `apps/website`，独立构建与部署；admin、web 保持 Vue3 + Vite SPA，不因引入 Nuxt 而迁移。

```bash
pnpm dlx nuxi@latest init apps/website
```

## 渲染模式

| 场景 | 配置 |
|---|---|
| 官网 / 内容站（默认） | `ssr: true`，服务端渲染，SEO 最优 |
| 纯静态托管 | `ssr: true` + `nuxi generate`，产出静态 HTML |
| 同应用内混有登录后页面 | 用 `routeRules` 逐路由声明 |

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/': { prerender: true },        // 首页预渲染
    '/docs/**': { swr: 3600 },       // 内容页增量静态化
    '/dashboard/**': { ssr: false }, // 登录后页面走 CSR
  },
})
```

不要一上来就 `ssr: false` —— 那等于放弃了引入 Nuxt 的唯一理由。

## SEO 必备项

站点级默认值放 `nuxt.config.ts`，页面级用 `useSeoMeta`。

```ts
export default defineNuxtConfig({
  app: {
    head: {
      htmlAttrs: { lang: 'zh-CN' },
      titleTemplate: '%s | 站点名',
    },
  },
})
```

```vue
<script setup lang="ts">
useSeoMeta({
  title: '产品名',
  description: '一句话描述，控制在 80 字内',
  ogTitle: '产品名',
  ogDescription: '分享卡片描述',
  ogImage: 'https://example.com/og.png',
  twitterCard: 'summary_large_image',
})
</script>
```

清单（缺一项基本等于没做 SEO）：

- **每页唯一 `title` + `description`**，禁止全站共用一份。
- **canonical**：多域名 / 带参 URL 场景必配，避免被判重复内容。
- **`htmlAttrs.lang`** 与 i18n 语言联动。
- **sitemap**：`@nuxtjs/sitemap`，随构建产出 `sitemap.xml`。
- **robots**：`@nuxtjs/robots`；**测试环境必须 `disallow: /`**，否则测试稿会被收录，后期清理极麻烦。
- **OG / Twitter 卡片**：分享到 IM 时的呈现，`ogImage` 必须是绝对 URL。
- **JSON-LD 结构化数据**：产品 / 文章 / 面包屑。
- **301 重定向**：改版换 URL 时在 `routeRules` 配 `redirect`，不要让旧链接直接 404。

## 国际化（SEO 视角）

官网统一用 `@nuxtjs/i18n`（底层就是 vue-i18n，与前端既有约定一致）：

- **语言与 SPA 端同一基线**：不保留"先中文站、后补英文"的过渡态。
- 每种语言**独立 URL**（如 `/en/` 前缀）。不要做纯客户端语言切换——搜索引擎索引不到 JS 切换出来的内容。
- 用 `useLocaleHead()` 产出 `hreflang` 与 `og:locale`。
- sitemap 需按语言生成 alternate 互链。
- 翻译文件放 `i18n/locales/*.json`；与 SPA 端共用的部分抽到 `packages/`。

## SSR 水合陷阱（Nuxt 最容易翻车的地方）

- **服务端没有 `window` / `document` / `localStorage`**。浏览器 API 一律放进 `onMounted`，或用 `<ClientOnly>` / `import.meta.client` 守卫。
- **跨请求状态污染**：模块级变量在服务端被所有请求共享，必须用 `useState()` 承载请求间共享状态。这是 SSR 下最严重的一类 bug（A 用户的数据串给 B 用户）。
- **持久化**：沿用 VueUse 方案，但 `useLocalStorage` 在服务端不可用——
  ```ts
  // 仅客户端生效，避免 SSR 阶段访问 localStorage
  const token = import.meta.client
    ? useLocalStorage<string | null>('auth:token', null)
    : ref<string | null>(null)
  ```
  注意：**参与 SEO 的主体内容不要包进 `<ClientOnly>`**，只把纯交互区域这么做。
- **pinia**：用 `@pinia/nuxt`，store 内同样禁止模块级可变状态。
- **登录后内容不要 SSR**：`ssr: false`，避免缓存串号。

## 图标

沿用 `@iconify/tailwind4` + `@iconify-json/carbon`，与其他前端保持一致。

**坑**：Nuxt 生态默认推荐的 `@nuxt/icon` 默认走 Iconify API 远程拉取，与「离线」要求冲突。若要使用它，必须显式安装 `@iconify-json/carbon` 并开启本地 bundle（`serverBundle` / `clientBundle`，选项名随版本可能调整）；否则直接用 Tailwind 插件方案。

## 样式

官网属 C 端定位，用 daisyUI + tailwindcss，与其他 C 端一致，**不要引入 element-plus**。

## 内容与性能

- 博客 / 文档类内容用 `@nuxt/content`（Markdown → 静态）。
- 图片用 `@nuxt/image`，**显式给 `width` / `height`**，避免 CLS。
- 字体本地化 + `font-display: swap`，避免字体阻塞 LCP。
- 上线前用 Lighthouse 过一遍 Core Web Vitals（LCP / CLS / INP）。

## 共享代码

- zod schema 从 `packages/schemas` 引，与后端、SPA 同一份。
- 请求走 `packages/http`，不要在 Nuxt 里另起一套 axios 封装。
- SSR 下服务端取数用绝对地址（`useRuntimeConfig().public.apiBase`），相对 URL 会失败。

## 部署

- Node 服务：Nitro 默认 `node-server` preset，产物 `.output/`，`node .output/server/index.mjs`。
- 静态托管：`nuxi generate` 产出 `.output/public/`，交给 nginx / CDN。
- Docker 多阶段构建见 [devops.md](devops.md)；nginx 反代必须正确转发 `Host` 与 `X-Forwarded-*`，否则 Nuxt 生成的 canonical / og URL 会变成内网地址。
- 环境变量用 `NUXT_PUBLIC_*` 在运行时注入，不要写死在 `nuxt.config.ts` 后打进客户端产物。

## 命令

```bash
pnpm --filter website dev        # 本地开发
pnpm --filter website build      # 构建（Node 服务）
pnpm --filter website generate   # 生成静态站
pnpm --filter website preview    # 本地预览产物
```
