---
name: vue-fullstack
description: Opinionated Vue 3 fullstack conventions for this user's stack. Use when building, configuring, reviewing or scaffolding Vue3, Nuxt, NestJS, Hono, Electron, uni-app or pnpm monorepo work, including 创建项目、官网 SEO 与前后端和 ORM 开发. Do not use for unrelated stacks.
---

# Vue Fullstack

下表是决定，不是建议：不得替换成其他方案，也不得"顺手升级"。备选列写明了唯一可切换的条件。

## 选型与细则

| 层 | 主选 | 备选（仅在此条件下） | 细则 |
|---|---|---|---|
| 前端框架 | Vue 3 + TypeScript + Vite | — | [frontend.md](references/frontend.md) |
| 官网 / SEO | Nuxt（SSR / SSG） | — | 仅官网、营销页、内容站 · [nuxt.md](references/nuxt.md) |
| 路由 / 状态 | vue-router + pinia | — | [frontend.md](references/frontend.md) |
| 请求 / 国际化 | axios + vue-i18n | — | [frontend.md](references/frontend.md) |
| 工具库 | vueuse、lodash-es、dayjs、decimal.js | — | [frontend.md](references/frontend.md) |
| 图表 / 动效 / 3D | echarts、animate.css、three.js、vue-bits | — | [frontend.md](references/frontend.md) |
| UI 库 | element-plus（后台）· daisyUI（C 端）· WotUI（小程序） | — | [ui.md](references/ui.md) |
| 图标 | `@iconify/tailwind4` + `@iconify-json/carbon` | — | [ui.md](references/ui.md) |
| 后端框架 | NestJS（业务 / 管理端 API） | HonoJS（轻量、边缘、BFF） | [backend.md](references/backend.md) |
| 校验 / 日志 | zod + pino | — | [backend.md](references/backend.md) |
| ORM | drizzle-orm + drizzle-kit | mikro-orm、mongoose（用户点名时） | [data.md](references/data.md) |
| 数据库 | PostgreSQL | MongoDB、Elasticsearch（ES 不作主库） | [data.md](references/data.md) |
| 缓存 / 队列 | Redis | Valkey（需要宽松许可时） | [data.md](references/data.md) |
| 对象存储 | RustFS | MinIO | [storage.md](references/storage.md) |
| 桌面端 | electron | tauri（极小体积或需 Rust 侧沙箱） | [desktop-mobile.md](references/desktop-mobile.md) |
| 小程序 | uni-app + WotUI | — | [desktop-mobile.md](references/desktop-mobile.md) |
| 测试 | vitest | — | [testing.md](references/testing.md) |
| 工程 | pnpm monorepo + catalogs、oxlint、oxfmt、simple-git-hooks、lint-staged | — | [tooling.md](references/tooling.md) |
| 运维 | docker、nginx、Loki | — | [devops.md](references/devops.md) |
| AI Agent | langchain、langgraph、deepAgent + Python FastAPI | — | [agent.md](references/agent.md) |

只读当前任务需要的那一份，不要一次全读。

## 铁律

1. **UI 分工不越界**：同一页面只用一套组件库。确需在后台嵌入 C 端页面，用 iframe 或独立路由隔离。
2. **校验单一来源**：zod schema 定义一次，前端表单、后端入参、DB 类型推导（drizzle `$inferSelect`）都从它派生。
3. **金额禁用 number**：运算用 decimal.js、存储用 PG `numeric`、传输用 string，禁止经手浮点数。
4. **统一响应体**：`{ code, data, message }`，`code` 为业务码，`0` 成功、其余失败，失败时 `data` 恒为 `null`。成败只看 `code`，不看 HTTP 状态码；业务码集中登记，不复用、不改含义。格式与 HTTP 分工见 [backend.md](references/backend.md)。
5. **中英双语基线**：起步即 `zh-CN` + `en`，两种语言同时交付、可随时切换，默认语言与 `fallbackLocale` 固定 `zh-CN`，基线语言只增不减。
6. **工具链固定**：oxlint 取代 eslint，oxfmt 取代 prettier，simple-git-hooks + lint-staged 取代 husky，vitest 取代 jest。被替代的工具不得引入，也不得两套并存。
7. **依赖版本收敛在 catalogs**：npm 依赖版本只在 `pnpm-workspace.yaml` 的 catalogs 声明，子包用 `catalog:` 引用；catalog 值写 caret 范围，精确锁定交给 `pnpm-lock.yaml`。新增依赖先查已有版本，不重复声明。
8. **Monorepo 边界**：跨包只通过包名的公开导出引用，不深链 `src` 内部路径。
9. **TS 严格**：`strict: true`；禁止 `any`（用 `unknown` + zod 收窄）；禁止 `@ts-ignore`，确需时用 `@ts-expect-error` 并注明原因。
10. **交付可验证**：同时给出 pnpm filter 形式的运行命令；跨端/跨包改动说明落在哪个 app/package、如何验证。
11. **产出代码不写规范元信息**：注释只解释代码本身；不出现「铁律 N」「按规范 / 按 skill 要求」「见 SKILL.md / references/*.md」。出处写进 PR 描述或 commit message。

## 默认目录骨架

```
<repo>/
├── apps/
│   ├── admin/        # element-plus 中后台 (Vue3 + Vite)
│   ├── web/          # daisyUI C 端 (Vue3 + Vite)
│   ├── website/      # Nuxt 官网（官网/SEO，SSR/SSG）
│   ├── server/       # NestJS 主服务
│   ├── bff/          # Hono 轻量/边缘服务（可选）
│   ├── desktop/      # electron 桌面端（可选）
│   ├── miniapp/      # uni-app + WotUI 小程序（可选）
│   └── agent/        # Python FastAPI + LangGraph（可选）
├── packages/
│   ├── ui/           # 跨端基础组件与设计 token
│   ├── schemas/      # zod schema（前后端共用）
│   ├── db/           # drizzle 表定义与迁移
│   ├── http/         # axios 实例与拦截器（无业务判断）
│   ├── utils/        # 通用工具（dayjs/decimal 封装）
│   ├── locales/      # 跨端共享的 i18n 语言包
│   └── config/       # 共享 oxlint/ts/tailwind 配置
├── pnpm-workspace.yaml
├── .oxlintrc.json
├── docker-compose.yml
└── package.json
```

按需增删目录；用户已有结构时服从其结构，不强行重排。
