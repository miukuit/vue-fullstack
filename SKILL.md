---
name: vue-fullstack
description: Opinionated Vue 3 fullstack conventions for this user's stack. Use when building, configuring, reviewing or scaffolding Vue3, Nuxt, NestJS, Hono, Electron, uni-app or pnpm monorepo work, including 创建项目、官网 SEO 与前后端和 ORM 开发. Do not use for unrelated stacks.
---

# Vue Fullstack

Apply this user's fixed stack and defaults instead of suggesting alternatives they did not pick. Tech choices below are decisions, not options — do not "upgrade" them.

## Stack map

| 层 | 主选 | 备选 | 说明 |
|---|---|---|---|
| 前端框架 | Vue 3 + TypeScript + Vite | — | 一律 `<script setup lang="ts">` |
| 官网 / SEO | **Nuxt**（SSR / SSG） | — | 仅用于官网、营销页、内容站，见 [nuxt.md](references/nuxt.md) |
| 路由 / 状态 | vue-router + pinia | — | pinia 用 setup 风格；持久化走 vueuse，不用 pinia-plugin-persistedstate |
| 请求 / 国际化 | axios + vue-i18n | — | 统一封装，禁止散落裸 axios；基线语言固定 `zh-CN` + `en` |
| 工具库 | vueuse、lodash-es、dayjs、decimal.js | — | 金额一律 decimal.js |
| 图表 / 动效 / 3D | echarts、animate.css、three.js、vue-bits | — | 见 [frontend.md](references/frontend.md) |
| 后台业务 UI | **element-plus** | — | 管理端、B 端中后台 |
| C 端 UI | **daisyUI + tailwindcss** | — | 面向消费者的页面 |
| 小程序 UI | **WotUI** | — | uni-app 小程序端 |
| 图标 | iconify（`@iconify/tailwind4` + `@iconify-json/carbon`） | — | 官方 Tailwind4 方案，编译期转 CSS，离线可用，禁用 `@iconify/vue` |
| 后端框架 | NestJS（业务/管理端 API） | HonoJS（轻量/边缘/BFF） | 见 [backend.md](references/backend.md) |
| 校验 / 日志 | zod + pino | — | zod 为唯一 schema 源 |
| ORM | **drizzle-orm** | mikro-orm、mongoose | 主推 drizzle，见 [data.md](references/data.md) |
| 数据库 | PostgreSQL | MongoDB、Elasticsearch | pg 为主库 |
| 对象存储 | **RustFS** | MinIO | S3 兼容，见 [storage.md](references/storage.md) |
| 桌面端 | **electron** | tauri | 默认 electron，见 [desktop-mobile.md](references/desktop-mobile.md) |
| 跨端小程序 | uni-app + WotUI | — | — |
| 测试 | **vitest** | — | 不用 jest，见 [testing.md](references/testing.md) |
| 工程 | pnpm monorepo + catalogs、oxlint、oxfmt、simple-git-hooks、lint-staged | — | 不用 eslint/prettier/husky |
| 运维 | docker、nginx、Loki | — | 日志走 Loki，见 [devops.md](references/devops.md) |
| AI Agent | langchain、langgraph、deepAgent + Python FastAPI | — | 见 [agent.md](references/agent.md) |

## 路由表 — 按任务读对应 reference

| 任务 | 读取 |
|---|---|
| 页面/组件、组合式函数、路由、状态、请求封装、图表、3D、国际化 | [references/frontend.md](references/frontend.md) |
| 官网/SEO、SSR/SSG、useSeoMeta、sitemap/robots、i18n hreflang、Nitro 部署 | [references/nuxt.md](references/nuxt.md) |
| 选 UI 库、tailwind/daisyUI 配置、element-plus 主题、WotUI 用法 | [references/ui.md](references/ui.md) |
| NestJS / Hono 接口、统一响应体与业务码、zod 校验、pino 日志、鉴权 | [references/backend.md](references/backend.md) |
| 单元测试、vitest 配置、mock | [references/testing.md](references/testing.md) |
| drizzle/mikro/mongoose、PG/Mongo/ES、迁移、索引 | [references/data.md](references/data.md) |
| 文件上传下载、签名 URL、RustFS/MinIO | [references/storage.md](references/storage.md) |
| electron/tauri 桌面、uni-app 小程序、平台差异 | [references/desktop-mobile.md](references/desktop-mobile.md) |
| monorepo 结构、pnpm catalogs、lint/format、git hooks | [references/tooling.md](references/tooling.md) |
| Dockerfile、nginx、compose、日志采集 | [references/devops.md](references/devops.md) |
| LangGraph/DeepAgent 服务、FastAPI 接口 | [references/agent.md](references/agent.md) |

只读当前任务需要的那一份，不要一次全读。

## 铁律

1. **UI 分工不越界**：后台业务用 element-plus，C 端用 daisyUI，小程序用 WotUI。同一页面内不混用两套组件库。
2. **ORM 主推 drizzle-orm**：新项目默认 drizzle + drizzle-kit。除非用户点名 mikro-orm 或 mongoose，否则不换。
3. **对象存储主推 RustFS**：S3 兼容协议，代码用 `@aws-sdk/client-s3` 对接，endpoint 走环境变量，绝不在代码里写死 MinIO。
4. **桌面端主推 electron**：默认 electron；只有明确要求极小体积、低内存或 Rust 侧安全沙箱时才用 tauri。
5. **校验单一来源**：zod schema 定义一次，前端表单、后端入参、DB 类型推导（drizzle `$inferSelect`）都从它派生。
6. **金额禁用 number**：涉及金额/精度用 decimal.js 或 PG `numeric`，禁止浮点直接运算。
7. **工具链固定**：oxlint 取代 eslint，oxfmt 取代 prettier，simple-git-hooks + lint-staged 取代 husky，vitest 取代 jest。不要引入被替代的工具，也不要两套并存。
8. **依赖版本收敛在 catalogs**：monorepo 内所有包版本写进 `pnpm-workspace.yaml` 的 catalogs，子包用 `catalog:` 引用。
9. **Monorepo 边界**：跨包只通过 `packages/*` 的公开导出引用，不深链 `src` 内部路径。
10. **TS 严格**：`strict: true`，禁止 `any`（用 `unknown` + zod 收窄），禁止 `@ts-ignore`（必要时 `@ts-expect-error` 并注明原因）。
11. **Nuxt 只做官网/SEO**：仅用于官网、营销页、内容站等需要 SSR/SSG 与搜索引擎收录的场景。admin、web 等既有 SPA 不因引入 Nuxt 而迁移或改写。
12. **统一响应体**：所有接口返回 `{ code, data, message }`。`code` 是业务码，`0` 成功、其余任何值均失败；**判定成败只看 `code`，不看 HTTP 状态码**；失败时 `data` 固定为 `null`。业务码集中登记，不复用、不改含义。
13. **中英双语基线**：国际化起步就以 `zh-CN` + `en` 为基线，两种语言同时交付、随时可切换，默认语言与 `fallbackLocale` 固定 `zh-CN`；不允许"先只做中文、事后补英文"。其他语言在此基线上按需追加，不删减基线语言。

## 默认目录骨架

```
<repo>/
├── apps/
│   ├── admin/        # element-plus 中后台 (Vue3 + Vite)
│   ├── web/          # daisyUI C 端 (Vue3 + Vite)
│   ├── website/      # Nuxt 官网（官网/SEO，SSR/SSG）
│   ├── api/          # NestJS 主服务
│   ├── bff/          # Hono 轻量/边缘服务（可选）
│   ├── desktop/      # electron 桌面端（可选）
│   ├── miniapp/      # uni-app + WotUI 小程序（可选）
│   └── agent/        # Python FastAPI + LangGraph（可选）
├── packages/
│   ├── ui/           # 共享基础组件与 token
│   ├── schemas/      # zod schema（前后端共用）
│   ├── api-client/   # 生成的类型化请求客户端
│   ├── utils/        # 通用工具（dayjs/decimal 封装）
│   ├── locales/      # 跨端共享的 i18n 语言包
│   └── config/       # 共享 oxlint/ts/tailwind 配置
├── pnpm-workspace.yaml
├── .oxlintrc.json
├── docker-compose.yml
└── package.json
```

按需增删目录；用户已有结构时服从其结构，不强行重排。

## 交付约定

- 新增依赖前先查 `pnpm-workspace.yaml` catalogs 里是否已有版本，有就复用同一版本。
- 交付代码时同步给出运行命令（pnpm filter 形式，见 [tooling.md](references/tooling.md)），不要只说"已完成"。
- 涉及跨端/跨包改动，说明改动落在哪个 app/package 以及如何验证。

> 维护提示：根目录 `AGENTS.md` 是同一套规范的通用入口（供 Codex / DeepSeek Harness / Cursor / Copilot / Gemini CLI 等 AGENTS.md 系工具读取），修改技术选型或铁律时两处需同步。
