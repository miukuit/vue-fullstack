# vue-fullstack

一套面向 **Vue 3 全栈** 的通用 AI 编程规范包，同时支持 **Trae · Claude Code · OpenAI Codex · DeepSeek Harness · Cursor · GitHub Copilot · Gemini CLI** 等主流编码代理工具。固化技术选型与工程约定，让 AI 规范产出代码，而不是每次给出它自己的默认方案。

## 覆盖范围

| 层 | 技术 |
|---|---|
| 前端 | Vue 3 · TypeScript · Vite · vue-router · vue-i18n · pinia · vueuse · axios · dayjs · decimal.js · echarts · lodash-es · animate.css · three.js · vue-bits · iconify |
| 官网 / SEO | Nuxt（SSR / SSG，官网、营销页与内容站） |
| UI 分工 | element-plus（后台业务）· daisyUI + tailwindcss（C 端）· WotUI（uni-app 小程序端） |
| 后端 | NestJS · HonoJS · zod · pino |
| ORM | drizzle-orm（主推）· mikro-orm · mongoose |
| 数据库 | PostgreSQL · MongoDB · Elasticsearch |
| 对象存储 | RustFS（主推）· MinIO |
| 跨端 | tauri（桌面主推）· electron · uni-app |
| 测试 | vitest · @vue/test-utils · @vitest/coverage-v8 · happy-dom |
| 工程 | pnpm monorepo + catalogs · oxlint · oxfmt · simple-git-hooks · lint-staged |
| 运维 | docker · nginx · Loki |
| AI Agent | langchain · langgraph · deepAgent · Python · FastAPI |

## 多工具入口

不同工具读取的入口文件不同，本仓库为两类机制各准备了一份入口，**内容核心一致，共用同一份 `references/`**：

| 入口文件 | 机制 | 适用工具 |
|---|---|---|
| [SKILL.md](SKILL.md) | Agent Skills（`SKILL.md` + `name`/`description` frontmatter） | Trae、Claude Code，以及任何支持 Agent Skills 的工具 |
| [AGENTS.md](AGENTS.md) | AGENTS.md 约定（纯 markdown，无 frontmatter） | OpenAI Codex、DeepSeek Harness、Cursor、GitHub Copilot、Gemini CLI、Google Jules、Zed、Windsurf、Aider、Amp、Devin 等 30+ 工具 |

一份规范，两种形态：选错入口文件不要紧，按上表认领即可。

## 目录结构

```
vue-fullstack/
├── SKILL.md                      # 技能入口（Trae / Claude Code）：技术栈总览、任务路由表、12 条铁律
├── AGENTS.md                     # 通用入口（Codex / DeepSeek Harness 等）：同一套规范的 AGENTS.md 版本
├── README.md                     # 本文件（安装教程）
├── references/                   # 按需加载的细分规范（agent 只读当前任务需要的那一份）
│   ├── frontend.md               # Vue3 写法、路由、状态、请求、i18n、图表、3D
│   ├── nuxt.md                   # Nuxt 官网、SSR/SSG、SEO、i18n hreflang、Nitro 部署
│   ├── ui.md                     # element-plus / daisyUI / WotUI 分工与配置
│   ├── backend.md                # NestJS / Hono、zod 共享 schema、pino 日志
│   ├── data.md                   # drizzle/mikro/mongoose、PG/Mongo/ES、迁移
│   ├── storage.md                # RustFS/MinIO、预签名上传、bucket 安全
│   ├── desktop-mobile.md         # tauri / electron、uni-app 小程序
│   ├── tooling.md                # monorepo、pnpm catalogs、oxlint/oxfmt、git hooks
│   ├── testing.md                # Vitest 单测/组件测试、mock、覆盖率
│   ├── devops.md                 # Dockerfile、compose、nginx、Loki
│   └── agent.md                  # LangGraph / DeepAgent + FastAPI
├── .gitignore
└── .gitattributes                # 统一 LF 换行，跨平台协作不抖动
```

**本仓库根目录就是规范目录本身**，`SKILL.md` / `AGENTS.md` 均位于仓库顶层。

内容**只有文档，不含可执行脚本**，安装不会执行任何代码。

## 安装

下文用 `<本仓库地址>` 指代本仓库的 git 地址。

### 一、Trae（技能机制）

目录名必须与 `SKILL.md` 的 `name` 一致，即 **`vue-fullstack`**。

```bash
# 全局（推荐）— Trae CN
git clone <本仓库地址> ~/.trae-cn/skills/vue-fullstack

# 全局 — Trae International
git clone <本仓库地址> ~/.trae/skills/vue-fullstack

# 项目级（只对当前项目生效）
git clone <本仓库地址> <你的项目>/.trae/skills/vue-fullstack
```

> 注意：全局目录 CN 版是 `~/.trae-cn/`、国际版是 `~/.trae/`；**项目级目录两个版本都固定是 `<项目>/.trae/`**，不是 `.trae-cn`。

Windows PowerShell：

```powershell
git clone <本仓库地址> "$env:USERPROFILE\.trae-cn\skills\vue-fullstack"
```

### 二、Claude Code（技能机制）

Claude Code 的技能格式与 Trae 相同，同为 `SKILL.md` + `name`/`description`，直接克隆即可：

```bash
# 个人级（所有项目可用）
git clone <本仓库地址> ~/.claude/skills/vue-fullstack

# 项目级（随项目提交，团队共享）
git clone <本仓库地址> <你的项目>/.claude/skills/vue-fullstack
```

> 补充：Claude Code 也读项目根的 `AGENTS.md`，但**仅当项目里没有 `CLAUDE.md` 时**才回退到它。所以 Claude Code 优先用上面的 `.claude/skills/` 方式。

### 三、OpenAI Codex / DeepSeek Harness / 其他 AGENTS.md 工具

这些工具读 **AGENTS.md**，有两种接法，任选其一。

**接法 A：项目级（推荐）** — 把本仓库的 `AGENTS.md` 放到你的项目根：

```bash
# 方式 1：直接复制内容（不与项目已有 AGENTS.md 冲突时）
curl -o AGENTS.md <本仓库 AGENTS.md 的 raw 地址>

# 方式 2：子目录 + 软链，保持与上游同步（推荐，便于 git pull 更新）
git clone <本仓库地址> tools/vue-fullstack-spec
# Linux / macOS
ln -s tools/vue-fullstack-spec/AGENTS.md AGENTS.md
# Windows PowerShell（需管理员或开发者模式）
New-Item -ItemType SymbolicLink -Path AGENTS.md -Target tools\vue-fullstack-spec\AGENTS.md
```

**接法 B：全局级** — 让规范对所有项目生效：

| 工具 | 全局入口 |
|---|---|
| OpenAI Codex | `~/.codex/AGENTS.md`（若存在 `AGENTS.override.md`，它优先级更高） |
| DeepSeek Harness | 项目根 `AGENTS.md`（跟随 AGENTS.md 约定） |
| Cursor / Copilot / Gemini CLI / Jules / Zed / Windsurf / Aider / Amp / Devin | 均原生读取项目根 `AGENTS.md`，按接法 A 即可，无需各自单独的规则文件 |

> 各工具对全局路径和优先级可能随版本调整，以你所装版本的官方说明为准。

### 四、不用 Git（下载压缩包）

1. 在本仓库页面点击 **Code → Download ZIP**；
2. 解压，把文件夹重命名为 `vue-fullstack`；
3. 按上面任一方式，把目录/文件放到对应位置。

### 五、让 Agent 自己装

**Trae**（走 skill-creator）：

> 用 skill-creator 从 <本仓库地址> 安装 vue-fullstack 技能

**通用**（把下面这段话发给任意一个 agent，让它自己落地）：

> 克隆 <本仓库地址>，把 SKILL.md 放到本工具的技能目录，并按本工具的约定把 AGENTS.md 接入到项目根或全局配置。

## 验证是否安装成功

- **Trae / Claude Code**：新开会话，可用技能列表中应出现 `vue-fullstack`；确认 `<skills 根目录>/vue-fullstack/SKILL.md` 存在。
- **Codex / DeepSeek Harness / 其他**：确认项目根（或全局）`AGENTS.md` 已被读取，可让它复述「后台业务用哪个 UI 库、ORM 主推哪个」来验证。
- **实际触发**：提一个命中触发词的需求，例如「按我的规范新建一个 vue3 monorepo 项目」，观察 AI 是否按 element-plus / drizzle-orm / oxlint 等约定作答。

## 更新

```bash
cd <安装位置>/vue-fullstack
git pull
```

若用「子目录 + 软链」接法，`git pull` 后无需再做任何事，入口自动跟随更新。

## 卸载

删除 `vue-fullstack` 目录，以及接入的 `AGENTS.md` / 软链即可，不影响项目运行。

## 触发词参考

规范会在以下类型的需求中被触发：

- 创建 / 初始化 Vue3、NestJS、Hono、Tauri、uni-app、pnpm monorepo 项目
- 官网 / SEO 站（Nuxt、SSR/SSG、useSeoMeta、sitemap、robots、hreflang）
- 前端页面与组件开发（element-plus 后台 / daisyUI C 端 / WotUI 小程序）
- 后端接口、zod 校验、pino 日志、鉴权
- 数据库与 ORM（drizzle-orm、迁移、索引、PG/Mongo/ES）
- 文件上传下载、对象存储（RustFS / MinIO）
- 工程配置（pnpm catalogs、oxlint、oxfmt、git hooks）
- 测试（vitest 单测与组件测试、mock、覆盖率）
- 部署（Docker、nginx、Loki）
- AI Agent 服务（LangGraph、DeepAgent、FastAPI）

## 自定义

- **改技术选型**：技术栈总览与铁律在 [SKILL.md](SKILL.md) 和 [AGENTS.md](AGENTS.md) 里各存一份，**两处需同步修改**，再同步更新对应 `references/*.md`。
- **加新领域**：在 `references/` 下新增文件，并在两个入口的「索引 / 路由表」中登记，保持引用为一级（`references/xxx.md`）。
- **调整触发范围**：`SKILL.md` 的 `description` 是技能机制下唯一的触发依据，改这一行即可；AGENTS.md 系工具则是全量读取，不受 description 影响。

## 许可

个人规范包，可自由复制、修改、分发。
