# 前端 — Vue 3 技术栈

## 组件写法

统一 `<script setup lang="ts">` + 组合式函数。禁止 Options API（第三方示例除外）。

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import { useI18n } from 'vue-i18n'

const props = defineProps<{ id: string }>()
const emit = defineEmits<{ submit: [value: string] }>()

const { t } = useI18n()
const loading = ref(false)
const title = computed(() => t('page.title'))
</script>
```

- props/emits 用类型声明，不用 runtime 对象写法。
- 可复用逻辑抽到 `composables/useXxx.ts`，返回 `ref`/`computed`/函数，不返回组件实例。
- 组件文件名 PascalCase，composable 名 `useXxx.ts`。

## 路由 vue-router

- 路由表集中在 `src/router/routes.ts`，按模块拆分后聚合。
- 一律懒加载：`() => import('@/views/xxx/index.vue')`。
- 路由 meta 声明 `title`（i18n key）、`auth`、`layout`。
- 后台管理端路由由权限动态生成；C 端路由静态声明。

```ts
const routes: RouteRecordRaw[] = [
  {
    path: '/order/:id',
    name: 'OrderDetail',
    component: () => import('@/views/order/detail.vue'),
    meta: { title: 'route.orderDetail', auth: true },
  },
]
```

## 状态 pinia

setup 风格 store，一个业务域一个 store。

```ts
export const useUserStore = defineStore('user', () => {
  const profile = ref<User | null>(null)
  const isLogin = computed(() => !!profile.value)
  async function fetchProfile() {
    profile.value = await api.user.me()
  }
  return { profile, isLogin, fetchProfile }
})
```

- 持久化统一用 VueUse（`useLocalStorage` / `useSessionStorage`），只持久 token / 主题 / 语言等少量字段，不整库持久化。
- 不在 store 里直接调裸 axios，走 `api-client`。

```ts
import { useLocalStorage, useSessionStorage } from '@vueuse/core'

export const useAuthStore = defineStore('auth', () => {
  // 需跨刷新、跨标签保留 → localStorage
  const token = useLocalStorage<string | null>('auth:token', null)
  const theme = useLocalStorage<'light' | 'dark'>('app:theme', 'light')
  // 仅本标签页有效 → sessionStorage
  const tempKey = useSessionStorage<string | null>('auth:temp-key', null)

  const isLogin = computed(() => !!token.value)
  return { token, theme, tempKey, isLogin }
})
```

VueUse 返回的 storage ref 本身就是响应式 state，直接 `return` 出去即可，读写自动落盘，不需要再写 `$subscribe` 或 `watch` 做同步。

持久化约定：

- **只持久确实需要的字段**（token、主题、语言、用户偏好）。业务数据、列表缓存、接口响应一律不持久化。
- storage key 统一加前缀（`app:` / `auth:`），避免同域多个项目互相覆盖。
- **敏感数据不进 localStorage**：refresh token、PII 走 httpOnly cookie 或仅内存态。
- 登出/切换账号时必须显式清理（把对应 ref 置 `null`），避免残留串号。
- **uni-app 小程序端没有 `localStorage`**，`useLocalStorage` 不可用；改用 `useStorage` 并显式传入适配 `uni.getStorageSync` / `uni.setStorageSync` / `uni.removeStorageSync` 的 storage 对象。

## 请求 axios

统一实例 + 拦截器，放在 `packages/api-client` 或 `src/api/request.ts`。**统一响应体 `{ code, data, message }` 的解包与成败判定只在这一层做**（结构定义见 backend.md 的「统一响应体」）。

```ts
import axios, { type AxiosError } from 'axios'
import type { ApiErr } from '@repo/schemas'

export class BizError extends Error {
  constructor(readonly code: number, message: string) {
    super(message)
    this.name = 'BizError'
  }
}

const http = axios.create({ baseURL: import.meta.env.VITE_API_BASE, timeout: 15_000 })

http.interceptors.request.use((cfg) => {
  const token = useAuthStore().token
  if (token) cfg.headers.Authorization = `Bearer ${token}`
  return cfg
})

http.interceptors.response.use(
  (res) => {
    const body = res.data as { code: number; data: unknown; message: string }
    // 业务码：0 成功，其余一律失败（此时 HTTP 可能仍是 200）
    if (body.code !== 0) throw new BizError(body.code, body.message)
    // 解包，调用方直接拿到业务数据
    return body.data
  },
  (err: AxiosError) => {
    // 框架级错误（401/403/500…）同样带统一响应体，一并转成 BizError
    const body = err.response?.data as ApiErr | undefined
    if (body?.code) return Promise.reject(new BizError(body.code, body.message))
    return Promise.reject(new BizError(-1, err.message))
  },
)
```

- **业务码判定只发生在这一层**。页面里禁止再写 `if (res.code === 0)`——调用方拿到的是已解包的 `data`。
- **HTTP 200 也可能是失败**。判定成败只看 `code`，不要用 `res.status === 200` 判断业务结果。
- **401 / token 过期在此处统一处理**（清 token + 跳登录），不要在业务页面各写一遍。
- 错误提示**按端注入**：后台用 `ElMessage`，C 端用自己的 toast，**不写死在请求层**。
- `BizError.code` 交给 i18n 映射文案（见下方国际化一节），拦截器里不拼中文。
- 分页/列表接口的数据结构统一为 `{ list, total, page, pageSize }`（已含在 `data` 内）。

## 国际化 vue-i18n

- `legacy: false`，组合式 `useI18n()`。
- 语言包按模块拆分：`locales/zh-CN/order.ts`，`index.ts` 聚合。
- 文案 key 用 `模块.语义`，如 `order.status.paid`，禁止把中文当 key。
- 后端返回的**业务码** `code` 在前端映射文案，不直接展示后端 `message` 字符串（见上方请求一节的 `BizError`）。
- 日期/金额格式化跟随 locale：dayjs locale 与 i18n locale 同步切换。

装配（`locales/index.ts`）：

```ts
import { createI18n } from 'vue-i18n'

export const SUPPORTED = ['zh-CN', 'en'] as const
export type AppLocale = (typeof SUPPORTED)[number]

const fallbackLocale: AppLocale = 'zh-CN'
// 初值优先级：持久化 → 系统语言 → 回退
const saved = localStorage.getItem('app:locale') as AppLocale | null
const locale: AppLocale =
  saved ?? SUPPORTED.find((l) => navigator.language.startsWith(l)) ?? fallbackLocale

export const i18n = createI18n({
  legacy: false,
  locale,
  fallbackLocale,
  messages: { 'zh-CN': zhCN, en },
})
```

- **locale 必须持久化**：key 用 `app:locale`（与 store 的 `app:` 前缀一致），否则刷新即回退；`fallbackLocale` 固定 `zh-CN`，缺失 key 时兜到回退语言而不是把 key 显示出来。
- **切换语言只走一个 `setLocale()`**，在一次调用里同步四处：vue-i18n 的 `locale`、`dayjs.locale()`、`<html lang>`、组件库语言（后台 `ElConfigProvider` 的 `locale`、小程序 WotUI 的 `Locale.use()`）。不要各页面自己散着改。
- **语言包跨端共享**：业务码文案、通用词这类**多端复用的文案放 `packages/locales`**（按模块拆的 TS 模块），各端 i18n 配置里 merge 进来。端专属文案（页面/组件私有）留在各自 app，不下沉。Nuxt 端 `i18n/locales/*.json` 只放官网专属文案，共享部分同样从 `packages/locales` 引入。
- **小程序端**：uni-app（Vue3）直接用 vue-i18n，语言包复用 `packages/locales`；持久化不能用 `localStorage`，改用 `useStorage` 适配 `uni.setStorageSync`（见上方持久化约定）。WotUI 组件自带文案用 `Locale.use(lang, langPack)` 切换，必须和项目 locale 一起在 `setLocale()` 里切，否则会出现「页面是英文、组件还是中文」。

## 工具库约定

| 场景 | 用什么 | 禁止 |
|---|---|---|
| 日期时间 | dayjs（含 utc/relativeTime/timezone 插件） | new Date 手算、moment |
| 金额精度 | decimal.js | 原生浮点 ±*/ |
| 集合/对象 | lodash-es（按需 import） | 引入全量 lodash |
| 通用响应式 | vueuse | 自己重复造 useDebounce 等 |
| 深拷贝 | `structuredClone` | JSON.parse(JSON.stringify()) |

金额示例：

```ts
import Decimal from 'decimal.js'
export const money = (v: string | number) => new Decimal(v).toDecimalPlaces(2, Decimal.ROUND_HALF_UP)
export const fmtCny = (v: Decimal.Value) => `¥${money(v).toFixed(2)}`
```

## 图表 echarts

- 用 `echarts/core` 按需引入 + 自写轻量 `useEcharts` composable，按需 `onMounted` 初始化、`onBeforeUnmount` dispose、`ResizeObserver` 自适应。
- 主题色从 tailwind/CSS 变量读取，不写死十六进制。
- 大数据量（>2 万点）开 `large: true` 或降采样。

## 动效与 3D

- animate.css：CSS 类进场动效；全局只 import 用到的动画类，避免整包。
- vue-bits：用于 C 端高级视觉效果（背景、文字动效），仅在 C 端 app 引用。
- three.js：按场景懒加载 `import('three')`，配 `@tresjs/core`（若已在用）或裸 three；离开页面必须 dispose geometry/material/renderer。
- 尊重 `prefers-reduced-motion`，动效可关。

## Vite / TS 约定

- 别名 `@` → `src`，`@pkg/*` → `packages/*`，在 `vite.config.ts` 与 `tsconfig.json` 双向同步。
- 环境变量 `VITE_` 前缀，类型定义在 `src/env.d.ts` 用 zod 校验启动即失败。
- 自动导入 `unplugin-auto-import` + `unplugin-vue-components`（element-plus 场景配 `ElementPlusResolver`）。
- 构建产物 sourcemap 只在 staging 打开；生产开 gzip/brotli。
