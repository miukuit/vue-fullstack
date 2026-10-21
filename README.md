# vue-fullstack

一套面向 **Vue 3 全栈** 的通用 AI 编程规范包，以 Agent Skills（`SKILL.md`）为唯一入口，适用于 **Trae · Claude Code** 等支持该机制的工具。固化技术选型与工程约定，让 AI 规范产出代码，而不是每次给出它自己的默认方案。

## 覆盖范围

| 层 | 技术 |
|---|---|
| 前端 | Vue 3 · TypeScript · Vite · vue-router · vue-i18n · pinia · vueuse · axios · dayjs · decimal.js · echarts · lodash-es · animate.css · three.js · vue-bits · iconify |
| 官网 / SEO | Nuxt（SSR / SSG，官网、营销页与内容站） |
| UI 分工 | element-plus（后台业务）· daisyUI + tailwindcss（C 端）· WotUI（uni-app 小程序端） |
| 后端 | NestJS · HonoJS · zod · pino |
| ORM | drizzle-orm（主推）· mikro-orm · mongoose |
| 数据库 | PostgreSQL · MongoDB · Elasticsearch |
| 缓存 / 队列 | Redis（主推）· Valkey · BullMQ |
| 对象存储 | RustFS（主推）· MinIO |
| 跨端 | electron（桌面主推）· tauri · uni-app |
| 测试 | vitest |
| 工程 | pnpm monorepo + catalogs · oxlint · oxfmt · simple-git-hooks · lint-staged |
| 运维 | docker · nginx · Loki |
| AI Agent | langchain · langgraph · deepAgent · Python · FastAPI |

## 入口

唯一入口是 [SKILL.md](SKILL.md)（Agent Skills：`SKILL.md` + `name`/`description` frontmatter），适用于 Trae、Claude Code 以及任何支持 Agent Skills 的工具；细分规范按需从 `references/` 加载。

## 目录结构

```
vue-fullstack/
├── SKILL.md                      # 技能入口：选型与细则索引、11 条铁律、目录骨架
├── README.md                     # 本文件（安装教程）
├── references/                   # 按需加载的细分规范（agent 只读当前任务需要的那一份）
│   ├── frontend.md               # Vue3 写法、路由、状态、请求、i18n、图表、3D
│   ├── nuxt.md                   # Nuxt 官网、SSR/SSG、SEO、i18n hreflang、Nitro 部署
│   ├── ui.md                     # element-plus / daisyUI / WotUI 分工与配置
│   ├── backend.md                # NestJS / Hono、zod 共享 schema、pino 日志
│   ├── data.md                   # drizzle/mikro/mongoose、PG/Mongo/ES、迁移
│   ├── storage.md                # RustFS/MinIO、预签名上传、bucket 安全
│   ├── desktop-mobile.md         # electron / tauri、uni-app 小程序
│   ├── tooling.md                # monorepo、pnpm catalogs、oxlint/oxfmt、git hooks
│   ├── testing.md                # Vitest 单测、mock
│   ├── devops.md                 # Dockerfile、compose、nginx、Loki
│   └── agent.md                  # LangGraph / DeepAgent + FastAPI
├── .gitignore
└── .gitattributes                # 统一 LF 换行，跨平台协作不抖动
```

**本仓库根目录就是规范目录本身**，`SKILL.md` 位于仓库顶层。

内容**只有文档，不含可执行脚本**，安装不会执行任何代码。

其中 `SKILL.md`、`references/` 是技能运行需要的文件；`README.md`、`.gitignore`、`.gitattributes` 属仓库自用，安装技能时无需带入（见下节）。

## 安装

下文用 `<本仓库地址>` 指代本仓库的 git 地址。

安装只取技能运行需要的文件（`SKILL.md`、`references/`）；仓库自用的 `README.md`、`.gitignore`、`.gitattributes` 与 `.git/` 不会进入技能目录。做法是浅克隆到临时目录，再拷贝所需文件：

### 一、Trae（技能机制）

目录名必须与 `SKILL.md` 的 `name` 一致，即 **`vue-fullstack`**。

macOS / Linux：

```bash
tmp="$(mktemp -d)"
git clone --depth 1 <本仓库地址> "$tmp"

# 全局（推荐）— Trae CN
dst=~/.trae-cn/skills/vue-fullstack
# 全局 — Trae International：dst=~/.trae/skills/vue-fullstack
# 项目级（只对当前项目生效）：dst=<你的项目>/.trae/skills/vue-fullstack

mkdir -p "$dst"
cp "$tmp/SKILL.md" "$dst/"
cp -R "$tmp/references" "$dst/"
rm -rf "$tmp"
```

Windows PowerShell：

```powershell
$tmp = "$env:TEMP\vue-fullstack-skill"
git clone --depth 1 <本仓库地址> $tmp

# 全局（推荐）— Trae CN
$dst = "$env:USERPROFILE\.trae-cn\skills\vue-fullstack"
# 全局 — Trae International：$dst = "$env:USERPROFILE\.trae\skills\vue-fullstack"
# 项目级（只对当前项目生效）：$dst = "<你的项目>\.trae\skills\vue-fullstack"

New-Item -ItemType Directory -Force -Path $dst | Out-Null
Copy-Item "$tmp\SKILL.md" $dst
Copy-Item "$tmp\references" $dst -Recurse
Remove-Item $tmp -Recurse -Force
```

> 注意：全局目录 CN 版是 `~/.trae-cn/`、国际版是 `~/.trae/`；**项目级目录两个版本都固定是 `<项目>/.trae/`**，不是 `.trae-cn`。

> 若想保留 `git pull` 增量更新，可改为直接 `git clone <本仓库地址> <技能目录>`（会一并带上 `.git/`、`README.md`、`.gitignore`、`.gitattributes`），或改用 `sparse-checkout`。

### 二、Claude Code（技能机制）

Claude Code 的技能格式与 Trae 相同，同为 `SKILL.md` + `name`/`description`，安装方式同上——浅克隆到临时目录，只取所需文件：

```bash
tmp="$(mktemp -d)"
git clone --depth 1 <本仓库地址> "$tmp"

# 个人级（所有项目可用）
dst=~/.claude/skills/vue-fullstack
# 项目级（随项目提交，团队共享）：dst=<你的项目>/.claude/skills/vue-fullstack

mkdir -p "$dst"
cp "$tmp/SKILL.md" "$dst/"
cp -R "$tmp/references" "$dst/"
rm -rf "$tmp"
```

### 三、不用 Git（下载压缩包）

1. 在本仓库页面点击 **Code → Download ZIP**；
2. 解压后只取 `SKILL.md` 与 `references/`（`README.md`、`.gitignore`、`.gitattributes` 是仓库自用文件，装技能不需要）；
3. 把文件夹重命名为 `vue-fullstack`，再按上面任一方式放到对应位置。

### 四、让 Agent 自己装

**Trae**（走 skill-creator）：

> 用 skill-creator 从 <本仓库地址> 安装 vue-fullstack 技能

**通用**（把下面这段话发给任意一个 agent，让它自己落地）：

> 从 <本仓库地址> 取 `SKILL.md` 与 `references/` 放到本工具的技能目录（目录名用 `vue-fullstack`；README.md、.gitignore、.gitattributes 不需要，不要整仓克隆）。

## 验证是否安装成功

- **Trae / Claude Code**：新开会话，可用技能列表中应出现 `vue-fullstack`；确认 `<skills 根目录>/vue-fullstack/SKILL.md` 存在。
- **实际触发**：提一个命中触发词的需求，例如「按我的规范新建一个 vue3 monorepo 项目」，观察 AI 是否按 element-plus / drizzle-orm / oxlint 等约定作答。

## 更新

技能安装目录不含 `.git`，更新方式是**重跑安装脚本**（会覆盖同名文件）：

macOS / Linux：

```bash
rm -rf <技能目录>
# 再执行「安装」一节中对应平台的脚本
```

Windows PowerShell：

```powershell
Remove-Item "$env:USERPROFILE\.trae-cn\skills\vue-fullstack" -Recurse -Force
# 再执行「一、Trae」中的 PowerShell 脚本
```

## 卸载

删除 `vue-fullstack` 技能目录即可，不影响项目运行。

## 触发词参考

规范会在以下类型的需求中被触发：

- 创建 / 初始化 Vue3、NestJS、Hono、Electron、uni-app、pnpm monorepo 项目
- 官网 / SEO 站（Nuxt、SSR/SSG、useSeoMeta、sitemap、robots、hreflang）
- 前端页面与组件开发（element-plus 后台 / daisyUI C 端 / WotUI 小程序）
- 后端接口、zod 校验、pino 日志、鉴权
- 数据库与 ORM（drizzle-orm、迁移、索引、PG/Mongo/ES）
- 文件上传下载、对象存储（RustFS / MinIO）
- 工程配置（pnpm catalogs、oxlint、oxfmt、git hooks）
- 测试（vitest 单测、mock）
- 部署（Docker、nginx、Loki）
- AI Agent 服务（LangGraph、DeepAgent、FastAPI）

## 自定义

- **改技术选型**：技术栈总览与铁律只在 [SKILL.md](SKILL.md) 里维护，修改后同步更新对应 `references/*.md`。
- **加新领域**：在 `references/` 下新增文件，并在 `SKILL.md` 的「选型与细则」表中登记，保持引用为一级（`references/xxx.md`）。
- **调整触发范围**：`SKILL.md` 的 `description` 是技能机制下唯一的触发依据，改这一行即可。

## 许可

个人规范包，可自由复制、修改、分发。
