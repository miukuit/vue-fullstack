# 测试 — Vitest

**统一用 Vitest，不引 Jest。** 前端、后端、共享包共用同一套 runner，配置收敛在 `packages/config`。

## 分层定位

| 层 | 测什么 | 依赖 |
|---|---|---|
| `packages/schemas` | zod schema 的边界与类型推导 | vitest（纯函数） |
| `packages/utils` | dayjs / decimal 封装 | vitest |
| `apps/*`（前端） | composable、pinia store | vitest |
| `apps/api` / `apps/bff` | service、controller、业务码分支 | vitest |
| `apps/website` | Nuxt 组合式函数、utils | vitest + @nuxt/test-utils |

**不为覆盖率写测试。** 优先测纯逻辑、边界条件、以及曾经出过 bug 的地方；不测展示细节。

## 配置

根 `vitest.config.ts`（或由 `packages/config` 导出共享配置）：

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    globals: true,              // 免去 import { describe, it, expect }
    setupFiles: ['./vitest.setup.ts'],
    exclude: ['**/node_modules/**', '**/dist/**', '**/.output/**'],
  },
})
```

- **文件命名与位置**：`*.test.ts`，与被测文件**同目录**（`useAuth.test.ts` 挨着 `useAuth.ts`），不建集中式 `__tests__` 目录。
- **环境默认 node**（vitest 默认值），测纯逻辑不需要 DOM 环境。
- **不引 jest**：`jest.fn()` → `vi.fn()`，`jest.mock()` → `vi.mock()`，`jest.useFakeTimers()` → `vi.useFakeTimers()`。不要两套测试框架并存。

## 前端约定

- **pinia store**：`beforeEach(() => setActivePinia(createPinia()))`。**每个用例新建 pinia 实例**，复用会导致状态串味、用例相互污染。
- **请求层**：mock `packages/api-client` 的导出，不要真发请求。需要拦截 HTTP 时用 MSW，不要在业务代码里加测试开关。
- **金额断言比较字符串**（decimal.js 的 `toString()`），不要断言浮点数——浮点比较必然不稳定。
- **i18n**：测试里固定 locale，不要依赖系统语言。

## 后端约定

```ts
describe('OrderService', () => {
  it('余额不足抛业务码 30001', async () => {
    const svc = new OrderService(mockRepo)
    await expect(svc.create({ amount: '999999' }))
      .rejects.toThrowError(expect.objectContaining({ code: 30001 }))
  })
})
```

- **`new` 直接实例化 Service 并注入 mock repository**，只有整模块联调才用 `Test.createTestingModule`。起 Nest 应用会让单测慢一个数量级。
- **断言业务码，不断言 `message` 文案**。文案会改，业务码才是契约（见 [backend.md](backend.md) 统一响应体）。
- **数据库用真实测试库**（docker compose 起临时 PG）。不要 mock 到把 SQL 全绕过去——drizzle 的 schema、迁移、约束错误只有真库能发现。
- **Hono 用 `app.request('/path', init)` 直接打请求**，无需起 HTTP server。
- 登录态用测试夹具签发的 token，不要绕开鉴权 Guard（否则权限回归测不出来）。

## 命令

`package.json` 里固定两个脚本：

```json
{ "scripts": { "test": "vitest run", "test:watch": "vitest" } }
```

```bash
pnpm --filter <pkg> test            # 单包（一次性，CI 用）
pnpm -r test                        # 全仓
pnpm --filter <pkg> test:watch      # 本地 watch
```

## 纪律

- **`.only` / `.skip` 合并进主干前必须清掉**。CI 上加检查或靠 review 拦。
- **不用 `setTimeout` 等异步**，用 `vi.waitFor()` 或 `await flushPromises()`。
- **时间相关一律 mock 系统时间**（`vi.setSystemTime`）。不要写依赖真实日期的断言（如「今天」），跨天/跨时区必挂。
- **随机数 / uuid 也要 mock**，否则断言不稳定。
- **测试之间不许有顺序依赖**，每个用例自己准备数据、自己清理。
