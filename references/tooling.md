# 工程管理 — pnpm monorepo / oxlint / oxfmt / hooks

工具链是**固定**的：oxlint 取代 eslint，oxfmt 取代 prettier，simple-git-hooks + lint-staged 取代 husky + lint-staged。不要引入被替代的工具，也不要同时保留两套。

## pnpm monorepo

`pnpm-workspace.yaml` 示例（**只给结构，不写版本号**——版本由 catalogs 承载，约定见下）：

```yaml
packages:
  - 'apps/*'
  - 'packages/*'

# 单版本依赖：全仓库共用同一个版本
catalog:
  vue:
  typescript:
  zod:
  dayjs:
  decimal.js:
  axios:

# 分组版本：按用途分段；同名包在不同段可各用不同版本
catalogs:
  frontend:
    vue-router:
    pinia:
    element-plus:
    '@wot-design-uni/components':
  backend:
    '@nestjs/core':
    hono:
    drizzle-orm:
    pino:
  tooling:
    oxlint:
    oxfmt:
    lint-staged:
    simple-git-hooks:
```

> 示例中的值为空，表示「版本以 `pnpm add` 实际解析结果为准」。**本规范刻意不锁定任何版本号**：文档里的版本号无法跟随上游更新，只会腐坏，而且极易被照抄成装不上的错误版本。需要锁定版本或复现构建，靠 `pnpm-lock.yaml`。

- 子包引用统一 `"vue": "catalog:"`、`"drizzle-orm": "catalog:backend"`，**不写死版本号**。
- 升级依赖改 catalogs 一处，全局生效。
- **文档里不出现具体版本号**；版本只存在于 `pnpm-workspace.yaml` 的 catalogs 与 `pnpm-lock.yaml`。例外：区分机制差异的大版本号要保留（`Vue 3`、`Tailwind v4/v3`、`tauri v2`），删了就无法判断该用哪套配置。
- 根 `package.json` 用 `pnpm -r` / `pnpm --filter <pkg>` 编排脚本。
- `pnpm-workspace.yaml` 之外不出现第二个 workspace 定义（不用 lerna/nx/turbo 替代 pnpm workspace；如需任务编排再加 turbo，但 workspace 定义仍归 pnpm）。

常用命令：

```bash
pnpm install
pnpm --filter admin dev            # 只跑 admin
pnpm --filter @repo/api start:dev  # 只跑 api
pnpm -r build                      # 全量构建
pnpm --filter @repo/schemas build  # 先构建被依赖包
```

## TypeScript 项目引用

- 根 `tsconfig.base.json` 放公共编译选项（`strict`、`verbatimModuleSyntax`、`moduleResolution: bundler`）。
- 子包 `extends` 根配置，只覆盖 `paths`/`outDir`。
- 共享包用 `tsdown` 或 `vite build --lib` 产出 `dist` + `types`，`exports` 字段暴露入口。
- 跨包引用走包名（`@repo/schemas`），**禁止** `../../packages/x/src/...` 深链源码。

## oxlint

`.oxlintrc.json`：

```json
{
  "plugins": ["typescript", "unicorn", "import", "promise", "vue"],
  "categories": { "correctness": "error", "suspicious": "warn", "pedantic": "off" },
  "rules": {
    "no-console": ["error", { "allow": ["error", "warn"] }],
    "typescript/no-explicit-any": "error",
    "typescript/consistent-type-imports": "error",
    "import/no-cycle": "error"
  },
  "ignorePatterns": ["dist", "node_modules", "drizzle", "src-tauri/target", "unpackage"]
}
```

- 跑 `pnpm oxlint`；CI 用 `pnpm oxlint --deny-warnings`。
- 不做类型检查（oxlint 只 lint）；类型检查单独 `pnpm -r typecheck`（`vue-tsc --noEmit`）。
- 不为通过 lint 而 `// oxlint-disable`，确需禁用要写明原因。

## oxfmt

- 跑 `pnpm oxfmt`（格式化）/ `pnpm oxfmt --check`（CI 校验）。
- 配置 `oxfmtrc.json`：`semi: false`、`singleQuote: true`、`printWidth: 100`、`trailingComma: 'all'`（跟随项目既有配置，首次创建时用这套）。
- 格式化与 lint 分离：oxfmt 管风格，oxlint 管正确性，不要互相越界。

## git hooks

`package.json`：

```json
{
  "scripts": {
    "lint": "oxlint",
    "format": "oxfmt",
    "typecheck": "pnpm -r typecheck",
    "prepare": "simple-git-hooks"
  },
  "simple-git-hooks": {
    "pre-commit": "pnpm lint-staged",
    "commit-msg": "node scripts/check-commit-msg.mjs $1"
  },
  "lint-staged": {
    "*.{ts,tsx,vue,js,mjs}": ["oxlint --fix", "oxfmt"],
    "*.{json,md,yml,yaml}": ["oxfmt"],
    "*.{scss,css}": ["oxfmt"]
  }
}
```

- `simple-git-hooks` 在 `pnpm install` 时自动写入 `.git/hooks`，不需要手动 `install`。
- secret 检测（gitleaks）另行配置，不塞进 lint-staged。
- `commit-msg` 用 Conventional Commits（`feat:` / `fix:` / `chore:`），脚本自校验。

## 发布与版本

- 共享包版本用 changesets 管理（如需）；仅内部消费可保持 `private: true` 不发布。
- 提交前确保 `lint + typecheck` 通过；CI 对 PR 跑同样命令。
