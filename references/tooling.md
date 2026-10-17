# 工程管理 — pnpm monorepo / oxlint / oxfmt / hooks

工具链是**固定**的：oxlint 取代 eslint，oxfmt 取代 prettier，simple-git-hooks + lint-staged 取代 husky + lint-staged。不要引入被替代的工具，也不要同时保留两套。

## pnpm monorepo

`pnpm-workspace.yaml` 示例（**只给结构，不写版本号**——版本由 catalogs 承载，约定见下）：

```yaml
packages:
  - 'apps/*'
  - 'packages/*'

# 单版本依赖：全仓库共用同一个版本；值一律 caret 范围 ^<resolved>
catalog:
  vue: ^<resolved>
  typescript: ^<resolved>
  zod: ^<resolved>
  dayjs: ^<resolved>
  decimal.js: ^<resolved>
  axios: ^<resolved>

# 分组版本：按用途分段；同名包在不同段可各用不同版本
catalogs:
  frontend:
    vue-router: ^<resolved>
    pinia: ^<resolved>
    element-plus: ^<resolved>
    '@wot-design-uni/components': ^<resolved>
  backend:
    '@nestjs/core': ^<resolved>
    hono: ^<resolved>
    drizzle-orm: ^<resolved>
    pino: ^<resolved>
  tooling:
    oxlint: ^<resolved>
    oxfmt: ^<resolved>
    lint-staged: ^<resolved>
    simple-git-hooks: ^<resolved>
```

> `^<resolved>` 是**占位符**：`^` 必须保留，`<resolved>` 换成实际解析出的版本。**本规范刻意不在文档里写任何真实版本号**：文档里的版本号无法跟随上游更新，只会腐坏，而且极易被照抄成装不上的错误版本。

- **catalog 值一律写 caret 范围，不写精确版本**：`xxx: ^1.2.3` ✅、`xxx: 1.2.3` ❌（`1.2.3` 只是形式示例，不是可照抄的版本）。catalog 表达的是**允许浮动到的范围**，补丁与次版本随上游走；把它写成精确版本就等于把全仓库钉死在一个补丁上，`pnpm update` 再也升不动。精确锁定由 `pnpm-lock.yaml` 负责，两者不冲突。
- **禁止凭记忆填版本号**：版本必须来自实际解析——`pnpm add <pkg>@latest`、`pnpm up` 或查 registry，拿到真实版本后补 `^` 写进 catalog。凭记忆写出的是对不上真实发布的版本（实测中 catalog 里出现过与 Vue 3 对不上的 `vue-router` / `pinia` 大版本号），一装就失败。
- `0.x` 依赖按 semver 走：`^0.4.1` 只允许补丁浮动，这是预期行为，不要用 `>=` 放宽。
- 例外：上游次版本会 breaking 又绕不开的包，才写精确值，并在 PR 里注明原因。
- **`.npmrc` 固化 add 行为**：`save-prefix=^`、`save-exact=false`，避免不同代理 / 兼容模式下被写成精确值。
- **升级命令**：`pnpm update` 在**范围之内**升到允许的最新；`pnpm update --latest` 会跨大版本，**大版本升降由用户确认**，不由 Agent 自行决定。升级只改 catalogs 这一处，全局生效。

- 子包引用统一 `"vue": "catalog:"`、`"drizzle-orm": "catalog:backend"`，**不写死版本号**。
- **文档里不出现具体版本号**；版本只存在于 `pnpm-workspace.yaml` 的 catalogs 与 `pnpm-lock.yaml`。例外：区分机制差异、支持下限与推荐值的版本号要保留（`Vue 3`、`Tailwind v4/v3`、`tauri v2`、`Node 24`、`TypeScript` 下限 5.0 / 推荐 6.x），删了就无法判断该用哪套配置。
- 根 `package.json` 用 `pnpm -r` / `pnpm --filter <pkg>` 编排脚本。
- `pnpm-workspace.yaml` 之外不出现第二个 workspace 定义（不用 lerna/nx/turbo 替代 pnpm workspace；如需任务编排再加 turbo，但 workspace 定义仍归 pnpm）。

常用命令：

```bash
pnpm install
pnpm --filter admin dev              # 只跑 admin
pnpm --filter @repo/serve start:dev  # 只跑 serve
pnpm -r build                        # 全量构建
pnpm --filter @repo/schemas build    # 先构建被依赖包
```

## Node 版本

- 本地与镜像必须是**同一个 Active LTS**：`.nvmrc`（或 `.node-version`）与 Dockerfile 基底镜像保持一致（见 [devops.md](devops.md)），否则会出现「本机跑得通、容器里跑不通」。
- 当前为 **24**（Node 22 已进入维护期）。Node 官方要求生产只用 Active / Maintenance LTS，**不要**用 Current（如 26）。
- 升 / 降大版本同 TypeScript 一节的约定：**由用户确认**，不自行升降。

## TypeScript 项目引用

- **版本下限**：Vue 3 官方工具链要求 **TypeScript ≥ 5.0**（`vue-tsc` / Vue - Official 的下限），低于该版本无法对 SFC 做类型检查。
- **推荐版本：`6.x`**。TS 6.0 是最后一个基于 JS 代码库的版本（微软明确不再出 6.1，只有补丁），语义稳定；且 Vue SFC 工具链通过 `@typescript/typescript6` 内嵌 TS 6 的编译器 API——**TS 7.0 未暴露稳定的 programmatic API，Vue / Volar 工作流目前用不了**（官方称待 7.1）。故本仓暂不升 7。
- catalogs 里的 `typescript` 以推荐值为准，**由用户确认后**确定并全仓共用，**不由 Agent 自行选版或升降大版本**。
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
  "ignorePatterns": ["dist", "out", "release", "node_modules", "drizzle", "src-tauri/target", "unpackage"]
}
```

- 跑 `pnpm oxlint`；CI 用 `pnpm oxlint --deny-warnings`。
- 不做类型检查（oxlint 只 lint）；类型检查单独 `pnpm -r typecheck`（`vue-tsc --noEmit`）。
- 不为通过 lint 而 `// oxlint-disable`，确需禁用要写明原因。

## oxfmt

- 跑 `pnpm oxfmt`（格式化）/ `pnpm oxfmt --check`（CI 校验）。
- 配置 `.oxfmtrc.json`：`semi: false`、`singleQuote: true`、`printWidth: 100`、`trailingComma: 'all'`（跟随项目既有配置，首次创建时用这套）。
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
