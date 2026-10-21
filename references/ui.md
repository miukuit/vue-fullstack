# UI 库分工与配置

## 分工（硬性）

| 端 | 库 | 场景 |
|---|---|---|
| 后台业务 / B 端中后台 | **element-plus** | 表格、表单、弹窗、权限后台、数据看板 |
| C 端（消费者 Web/H5） | **daisyUI + tailwindcss** | 落地页、营销页、用户中心、内容页 |
| uni-app 小程序端 | **WotUI** (`@wot-design-uni`) | 微信/支付宝/抖音小程序、跨端 H5 |

- 确需在后台嵌入 C 端页面，用 iframe 或独立路由隔离，不要同页混装。
- 选择依据是**面向谁**，不是技术偏好：给运营/管理员用 → element-plus；给终端用户用 → daisyUI。

## Tailwind + daisyUI

- Tailwind v4 用 `@tailwindcss/vite` 插件 + CSS `@import "tailwindcss"`；v3 用 `tailwind.config.ts` + postcss。跟随项目既有版本，不擅自升级。
- **Tailwind 是所有前端应用的布局层**（admin 后台的布局/间距/工具类同样用它），daisyUI 只是 C 端的组件主题层，后台不引 daisyUI。
- daisyUI 通过 `@plugin "daisyui"` 载入，主题用 `@plugin "daisyui/theme"` 或 `[data-theme]` 切换。
- 主题 token 与 i18n/暗色模式联动：暗色用 `data-theme` 而非 `dark:` 前缀堆砌。
- 业务样式优先用 daisyUI 语义类（`btn` `card` `badge` `alert`），避免长串原子类；复杂布局才写原子类。
- 自定义主题色统一写在 CSS 变量，禁止在组件里写死 `#rrggbb`。

```css
@import "tailwindcss";
@plugin "@iconify/tailwind4";
@plugin "daisyui" {
  themes: light --default, dark --prefersdark;
}
```

## element-plus（后台）

- 按需引入：`unplugin-vue-components` + `ElementPlusResolver`，不整包 `app.use(ElementPlus)`。
- 中文/多语言：`ElConfigProvider` 包根组件，`locale` 跟随 vue-i18n 语言。
- 主题：用 SCSS 变量或 CSS 变量覆写，集中放 `styles/element/index.scss`，不在组件里 `:deep` 到处改。
- 表格：统一封装 `ProTable`（分页、加载、空态、列配置），业务页只用封装后的组件。
- 表单：统一封装 `ProForm`，schema 用 zod 生成校验规则（见 [backend.md](backend.md) 的 schema 复用）。
- `ElMessage`/`ElMessageBox` 在请求拦截器里按端注入，不硬编码在 `packages/http`。

```ts
import { ElConfigProvider } from 'element-plus'
import zhCn from 'element-plus/es/locale/lang/zh-cn'
// <ElConfigProvider :locale="zhCn"><router-view /></ElConfigProvider>
```

## WotUI（uni-app 小程序）

- 安装 `@wot-design-uni`，在 `pages.json` 配 `easycom` 按需加载：

```json
{ "easycom": { "autoscan": true, "custom": { "^wd-(.*)": "@wot-design-uni/components/wd-$1/wd-$1.vue" } } }
```

- 主题用 WotUI 的 CSS 变量（`--wot-color-theme` 等），在 `App.vue` 或全局 scss 覆写。
- 小程序端限制：无 DOM 操作、无 `window`、单包体积受限。长列表用 `wd-list`/虚拟列表，图片走 CDN + 懒加载。
- 跨端条件编译用 `#ifdef MP-WEIXIN` / `#ifndef H5`，不要写运行时 `if (平台)` 判断。
- 尺寸用 `rpx` 或 WotUI 的 `px`（已做适配），不混用 rem。
- 小程序端不引 element-plus / daisyUI / three.js；图表用 `lime-echart` 等小程序兼容方案。

## 主题

三端都靠 CSS 变量出主题，**切换开关与持久化只允许一份**，不给每个端各写一套。

- **切换**：用 VueUse 的 `useDark`，并把属性映射到各端认的位置——daisyUI 认 `data-theme`，element-plus 认 `html.dark`，两个一起挂。

```ts
// composables/useTheme.ts —— web / admin 复用同一份
import { watchEffect } from 'vue'
import { useDark } from '@vueuse/core'

export const isDark = useDark({
  selector: 'html',
  attribute: 'data-theme', // daisyUI 靠它切主题
  valueDark: 'dark',
  valueLight: 'light',
  storageKey: 'app:theme', // 与 store 的 app: 前缀一致
})

// element-plus 只认 html.dark，与 data-theme 同步，不另开开关
watchEffect(() => {
  document.documentElement.classList.toggle('dark', isDark.value)
})
```

admin 端另需引入 element-plus 的暗黑变量表：`import 'element-plus/theme-chalk/dark/css-vars.css'`。

- **首屏防闪烁**：暗色必须在首屏渲染前就落到 `<html>`，否则会先白闪一下。在 `index.html` 的 `<head>` 放一段同步脚本，先读持久化值写属性，再由 `useDark` 接管。

```html
<script>
  (function () {
    var t = localStorage.getItem('app:theme')
    if (!t) t = matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light'
    var el = document.documentElement
    el.dataset.theme = t
    el.classList.toggle('dark', t === 'dark')
  })()
</script>
```

- **跟随系统**：`useDark` 默认读 `prefers-color-scheme`；用户手动选择后写 localStorage，不再跟随系统。不要自己再写一套 `matchMedia` 监听。
- **token 分层映射**：原始色值只存在于 `packages/ui` 的语义变量，三端各自把组件库变量指过去，业务代码不碰色值。变量名要避开组件库自身命名（如不用 `--color-primary`），避免循环引用。

```css
/* packages/ui —— 唯一色值来源 */
:root {
  --color-brand: #4f46e5;
  --color-surface: #ffffff;
  --color-text: #18181b;
}
[data-theme='dark'] {
  --color-surface: #0b0b0f;
  --color-text: #f4f4f5;
}

/* web：daisyUI 主题变量指向 token（写在 @plugin "daisyui/theme" 内） */
--color-primary: var(--color-brand);
--color-base-100: var(--color-surface);

/* admin：element-plus 变量指向 token */
:root { --el-color-primary: var(--color-brand); }
html.dark { --el-bg-color: var(--color-surface); }

/* miniapp：WotUI 变量指向 token */
page { --wot-color-theme: var(--color-brand); }
```

## 图标 iconify

**首选官方 Tailwind 4 方案：`@iconify/tailwind4` + `@iconify-json/carbon`。图标在构建期被编译成 CSS（内联 SVG data URI），零运行时请求、可完全离线。禁用 `@iconify/vue`。**

```bash
pnpm add -D @iconify/tailwind4 @iconify-json/carbon
```

```css
/* styles/main.css */
@import "tailwindcss";
@plugin "@iconify/tailwind4";
```

```html
<span class="icon-[carbon--home]"></span>
<span class="icon-[carbon--edit] text-primary text-xl"></span>
```

**离线判定标准：插件本身不含任何图标数据。** 只要用到某个图标集，就必须把对应的 `@iconify-json/<collection>` 显式装成依赖。未安装的图标集会让构建期去请求 `api.iconify.design`，内网/离线环境直接失败——所以「图标集是否已装为依赖」就是离线与否的唯一判定依据。

为什么禁用 `@iconify/vue`：它是运行时按需从 Iconify API 拉取图标数据，离线必然失败，首屏还多一轮网络往返。而 `@iconify-json/*` 是纯数据包，配合插件在构建期就把图标固化进 CSS 产物。

约定：

- 类名格式固定 `icon-[{prefix}--{icon}]`，如 `icon-[carbon--home]`。
- **图标集默认只用 `carbon`**，不要每个页面换一套。确需其他图标集时，按同样方式加装 `@iconify-json/<collection>`，保持离线。
- 尺寸用 Tailwind 的 `text-*` / `size-*`，颜色用 `text-*`（图标跟随 `currentColor`），不要把颜色写死在图标里。
- 禁止引入 iconfont 字体包、禁止手工堆内联 `<svg>` 文件、禁止 `<img src="https://api.iconify.design/...">`。
- element-plus 后台同样用这套 class（Tailwind 在后台端同样启用）；`<el-icon>` 只在 element-plus 组件自身的插槽约定处使用。
- **唯一例外：uni-app 小程序端**。各小程序平台对 CSS `mask-image` 的支持不一致，需在目标平台实测；确认不可用时，用 `unplugin-icons` 从同一份 `@iconify-json/carbon` 离线生成静态资源，仍然不引入运行时 API 请求。

## 共享 UI 包

- `packages/ui` 只放**跨端通用**的基础组件与设计 token（颜色/间距/圆角/字号）。
- 端专属组件放各自 app 内，不下沉到 `packages/ui`。
- `packages/ui` 不依赖 element-plus / daisyUI / WotUI 任何一方；token 用原生 CSS 变量，各端自行映射。
