# 数据层 — ORM 与数据库

## 选型（主推 drizzle-orm）

| 场景 | 用 |
|---|---|
| **默认 / 新项目 / 关系型 PostgreSQL** | **drizzle-orm** + drizzle-kit |
| 需要复杂实体图、UoW、装饰器实体（既有 NestJS 项目） | mikro-orm |
| MongoDB 文档模型 | mongoose |
| 全文检索 / 日志分析 / 复杂聚合 | Elasticsearch（**只读索引，不作主库**） |

- 主库是 PostgreSQL，MongoDB 用于非结构化/日志类数据，ES 用于搜索与分析。
- 默认 drizzle；用户点名 mikro-orm / mongoose 时才切换，并保持项目内一致。

## drizzle-orm

Schema 定义（`packages/db/src/schema/`）：

```ts
import { pgTable, uuid, varchar, numeric, timestamp, index, pgEnum } from 'drizzle-orm/pg-core'

export const orderStatus = pgEnum('order_status', ['pending', 'paid', 'shipped', 'closed'])

export const orders = pgTable('orders', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull(),
  amount: numeric('amount', { precision: 12, scale: 2 }).notNull(),   // 金额用 numeric
  status: orderStatus('status').notNull().default('pending'),
  remark: varchar('remark', { length: 500 }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  index('orders_user_created_idx').on(t.userId, t.createdAt),
])

export type Order = typeof orders.$inferSelect
export type NewOrder = typeof orders.$inferInsert
```

约定：

- 表名蛇形复数，字段蛇形，TS 侧驼峰（drizzle 自动映射）。
- 主键统一 `uuid` + `defaultRandom()`。
- 时间列一律 `timestamp with time zone`。
- 金额列 `numeric(12,2)`，读出来是 string，交给 decimal.js，禁止转 number。
- 每张业务表带 `createdAt` / `updatedAt`；软删用 `deletedAt timestamp`。
- 索引按查询模式建，外键列、高频过滤 + 排序列建复合索引。不要给每个列都建索引。
- 关系用 `relations()` 声明，查询默认用手写 join，避免 N+1；确实需要嵌套时用 `db.query.xxx.findMany({ with })`。

配置与迁移：

```ts
// drizzle.config.ts
export default defineConfig({
  schema: './packages/db/src/schema/index.ts',
  out: './packages/db/drizzle',
  dialect: 'postgresql',
  dbCredentials: { url: process.env.DATABASE_URL! },
})
```

- 迁移走 `drizzle-kit generate` + `drizzle-kit migrate`；生产**只跑 migrate**，禁止 `push`。
- 迁移文件提交进仓库并评审；破坏性变更（drop/rename）必须单独说明并提供回滚方案。

事务与查询：

```ts
await db.transaction(async (tx) => {
  const [order] = await tx.insert(orders).values({ userId, amount }).returning()
  await tx.insert(orderItems).values(items.map((i) => ({ orderId: order.id, ...i })))
})
```

- 批量写入用 `insert().values([])`，不要循环单条 insert。
- 分页用 `limit/offset`；深分页改游标（`where id > cursor`）。
- 列表查询只 select 需要的列，禁止无脑 `select *`。

## mikro-orm（备选）

- 装饰器实体 `@Entity()`，`@PrimaryKey({ type: 'uuid' })`，`MikroORM.init()` 单例注入 NestJS。
- 迁移用 `mikro-orm migration:create/up`，同样禁止生产 `schema:update`。
- 与 drizzle 不同时用在一个服务里。

## mongoose（备选，MongoDB）

- Schema 显式声明、开 `timestamps: true`、`toJSON` 转换 `_id` → `id`。
- 查询一律 `.lean()` 除非要实例方法。
- 索引在 schema 里 `index: true` 或 `schema.index()`，同时保证生产已建索引。
- 事务需副本集；跨库一致性用 outbox 模式，不硬拆分布式事务。

## Elasticsearch

- 只作搜索/分析索引，源数据以 PG/Mongo 为准，通过消息或定时同步写入 ES。
- 索引设计：`index-template` 管理 mapping，`dynamic: strict` 防止脏字段。
- 别名 + 滚动索引（`orders-000001` → alias `orders`），便于重建。
- 禁止把 ES 当唯一数据源做业务写入。

## 多数据源共存

- 每个数据源一个独立 client 单例，导出 `db`（PG）、`mongo`、`es`，不混用一个连接对象。
- 连接字符串走环境变量 + zod 校验：`DATABASE_URL`、`MONGO_URL`、`ES_NODE`。
- 健康检查分别探测三个数据源，任一不可用要在 `/health` 中体现。
