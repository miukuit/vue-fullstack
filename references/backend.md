# 后端 — NestJS / Hono + zod + pino

## 选型规则

| 场景 | 用 |
|---|---|
| 主业务 API、管理后台接口、需要 DI/守卫/装饰器/队列 | **NestJS** |
| 轻量 BFF、边缘函数、Serverless、聚合多个上游 | **HonoJS** |
| AI Agent 服务 | Python FastAPI（见 [agent.md](agent.md)） |

同一个仓库可以并存：`apps/serve`（NestJS）+ `apps/bff`（Hono）。不要用 NestJS 写一次性轻服务，也不要用 Hono 硬扛复杂业务。

## NestJS 约定

- 模块化：`modules/<domain>/{dto,entities,service,controller,module}.ts`。
- 分层：Controller 只做参数解析与响应包装，业务在 Service，数据访问在 Repository（drizzle 封装）。
- 全局管道：

```ts
app.useGlobalPipes(new ZodValidationPipe())   // 自定义，底层用 zod
app.useGlobalFilters(new AllExceptionsFilter())
app.useGlobalInterceptors(new TransformInterceptor())  // 统一响应体
```

- zod 做入参校验，替代 class-validator。DTO 用 zod schema 推导：`type CreateUserDto = z.infer<typeof createUserSchema>`。
- 鉴权用 Passport JWT + `@UseGuards(JwtAuthGuard)`；权限用自定义 `@RequirePermission('user:create')` 装饰器 + Guard。
- 响应体一律走统一结构，见下方「统一响应体」；**不要在 Controller 里手写 `c.json({...})` 式的裸响应**。
- 配置用 `@nestjs/config` + zod 校验 schema，缺环境变量启动即失败。

## 统一响应体

**所有接口（成功与失败）响应体固定三段：**

```ts
{ code: number; data: T | null; message: string }
```

| 字段 | 含义 |
|---|---|
| `code` | **业务码**。`0` 表示成功，**其余任何值都表示失败** |
| `data` | 成功时为业务数据；失败时**固定 `null`** |
| `message` | 面向开发者 / 兜底文案，前端正常路径**不直接展示** |

zod 定义一次，前后端共用（铁律 5）：

```ts
// packages/schemas/src/response.ts
import { z } from 'zod'

export type ApiOk<T> = { code: 0; data: T; message: string }
export type ApiErr = { code: number; data: null; message: string }

export const failSchema = z.object({
  code: z.number().int().refine((c) => c !== 0, '失败码不能为 0'),
  data: z.null(),
  message: z.string(),
})
```

### 规则

- **失败时 `data` 必须是 `null`，不能省略字段**。JSON 会丢掉 `undefined`，省略后前端无法区分「无数据」与「字段缺失」，可判别联合类型直接失效。
- **`message` 不作为业务判断依据**，禁止 `if (message === '...')`。判定成败**只看 `code`**。
- **业务码集中登记**，禁止随手写魔法数字。放 `packages/schemas/src/codes.ts`，前端 i18n 按 code 映射文案（见 [frontend.md](frontend.md) 国际化一节）。
- **业务码全局唯一且不复用**。废弃的码保留占位并标 `@deprecated`，绝不改其原有含义。

### 业务码分段

| 段位 | 含义 | 示例 |
|---|---|---|
| `0` | 成功 | — |
| `1xxxx` | 通用 / 参数校验 | `10001` 参数缺失、`10002` 格式非法 |
| `2xxxx` | 认证与授权 | `20001` 未登录、`20002` token 过期、`20003` 无权限 |
| `3xxxx` | 业务规则拒绝 | `30001` 余额不足、`30002` 当前状态不允许该操作 |
| `4xxxx` | 资源 | `40001` 资源不存在、`40002` 资源已存在 |
| `5xxxx` | 系统 / 下游依赖 | `50001` 内部错误、`50002` 下游超时 |

段位划分可按项目调整，但**一旦发布不得重排**。

### HTTP 状态码与业务码的分工

两者**并存，不互相替代**：

- **可预期的业务失败** → HTTP `200` + `code ≠ 0`（余额不足、状态冲突等）。
- **框架级 / 基础设施错误** → 保留语义化 HTTP 状态码，响应体仍是同一结构：401 未认证、403 无权限、404 路由不存在、429 限流、500 未捕获异常。

**不要把 HTTP 状态码一律设成 200。** 那会让网关路由、监控告警、CDN 缓存、客户端重试策略全部失去判断依据——「统一响应体」统一的是**格式**，不是 HTTP 语义。

### 分页

```ts
{ code: 0, data: { list: T[], total: number, page: number, pageSize: number }, message: 'ok' }
```

- `total` 是**满足条件的总条数**，不是当前页条数。
- 空列表返回 `list: []`，不要返回 `null`。
- `page` 从 1 开始；`pageSize` 设服务端上限（如 100），超出直接钳制。
- 大数据量列表用游标分页 `{ list, nextCursor, hasMore }`，**不要与页码分页混用在同一接口族**。

### 金额与日期

- **金额一律用 `string` 传输**（如 `"12.34"`），前端 decimal.js 解析，后端 PG `numeric` 承接。JSON number 是 IEEE754 双精度，大额与高精度必然失真（铁律 6）。
- 日期时间一律 ISO8601（UTC）字符串，不传时间戳数字或本地化字符串。

### NestJS 落地

```ts
// 成功：拦截器统一包装，Controller 直接 return 业务数据
@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, ApiOk<T>> {
  intercept(_ctx: ExecutionContext, next: CallHandler<T>) {
    return next.handle().pipe(map((data) => ({ code: 0, data: data ?? null, message: 'ok' })))
  }
}

// 失败：业务异常
export class BizException extends HttpException {
  constructor(readonly code: number, message: string, status: HttpStatus = HttpStatus.OK) {
    super({ code, data: null, message }, status)
  }
}

// 全局过滤器：兜住一切未捕获异常
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(e: unknown, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse()
    if (e instanceof BizException) return res.status(e.getStatus()).json(e.getResponse())
    logger.error({ err: e }, 'unhandled')   // 详细堆栈只进 pino
    return res.status(500).json({ code: 50001, data: null, message: '服务器内部错误' })
  }
}
```

- **业务失败优先 `throw new BizException(30001, '余额不足')`**，不要 `return` 一个失败结构——否则调用方容易忘判、静默当成成功。
- **未捕获异常绝不把堆栈或原始错误信息返回给客户端**，只回通用 `message`，细节写 pino。

### Hono 落地

```ts
export const ok = <T>(c: Context, data: T) => c.json({ code: 0, data: data ?? null, message: 'ok' })
export const fail = (c: Context, code: number, message: string, status = 200) =>
  c.json({ code, data: null, message }, status)

app.onError((err, c) => {
  if (err instanceof BizError) return fail(c, err.code, err.message, err.status)
  logger.error({ err }, 'unhandled')
  return fail(c, 50001, '服务器内部错误', 500)
})
```

Hono 与 NestJS 必须返回**完全相同的结构**，前端才能只维护一个拦截器。

## Hono 约定

```ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'
import { logger } from 'hono/logger'

const app = new Hono()
app.use('*', logger())
app.get('/health', (c) => c.json({ ok: true }))
app.post('/orders', zValidator('json', createOrderSchema), async (c) => {
  const body = c.req.valid('json')   // 已类型化
  return c.json(await orderService.create(body))
})
export default app
```

- 校验用 `@hono/zod-validator`，与前端共用 `packages/schemas` 的 zod schema。
- 错误统一用 `app.onError` 处理，返回与 NestJS 相同的响应体结构，方便前端复用。
- 运行时：Node 用 `@hono/node-server`，边缘/Worker 直接 export default app；两边共用同一份路由代码。

## zod 共享 schema

- schema 放 `packages/schemas/src/<domain>.ts`，前端表单、后端入参、drizzle 类型三方共用。

```ts
export const createOrderSchema = z.object({
  productId: z.string().uuid(),
  quantity: z.number().int().positive().max(999),
  remark: z.string().max(500).optional(),
})
export type CreateOrderInput = z.infer<typeof createOrderSchema>
```

- 前端表单直接复用：`zodResolver(createOrderSchema)`（element-plus / vee-validate 均可用）。
- 日期入参统一 ISO8601 字符串，`z.coerce.date()` 或 `z.string().datetime()`，避免时区歧义。
- 金额入参用 `z.string().regex(/^\d+(\.\d{1,2})?$/)`，不要 number。

## pino 日志

- NestJS 用 `nestjs-pino`，Hono 用 `pino` + 中间件，都是**结构化 JSON 日志**，禁止 `console.log`。
- 开发环境 `pino-pretty`，生产环境裸 JSON 输出给 Loki（见 [devops.md](devops.md)）。
- 必带字段：`requestId`（trace id）、`userId`、`path`、`method`、`statusCode`、`durationMs`。
- 敏感字段脱敏：`redact: ['req.headers.authorization', 'req.body.password', '*.token']`。
- 错误日志带 `err` 字段（pino 自动序列化 stack），业务日志用 `logger.info({ orderId }, 'order created')`。

```ts
export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  redact: ['req.headers.authorization', 'req.body.password', '*.token'],
  ...(isDev ? { transport: { target: 'pino-pretty' } } : {}),
})
```

## 跨服务契约

- 前端类型从后端 schema/OpenAPI 生成，放 `packages/api-client`，禁止手写重复的接口类型。
- 版本变更同步更新 schema 包，PR 里说明是否为破坏性变更。
