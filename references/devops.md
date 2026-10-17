# 运维 — Docker / nginx / Loki

## 镜像

前端（多阶段 + nginx）：

```dockerfile
FROM node:24-alpine AS build
WORKDIR /app
RUN corepack enable
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY . .
RUN pnpm install --frozen-lockfile && pnpm --filter web build

FROM nginx:alpine
COPY deploy/nginx/web.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/apps/web/dist /usr/share/nginx/html
```

后端（多阶段 + 非 root）：

```dockerfile
FROM node:24-alpine AS build
WORKDIR /app
RUN corepack enable
COPY . .
RUN pnpm install --frozen-lockfile && pnpm --filter @repo/serve build && pnpm prune --prod

FROM node:24-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=build --chown=app:app /app .
USER app
EXPOSE 3000
CMD ["node", "apps/serve/dist/main.js"]
```

- 必须多阶段，最终镜像不含源码与 devDependencies。
- 必须 `--frozen-lockfile`。
- 以非 root 运行；不加 `--privileged`。
- `.dockerignore` 排除 `node_modules`、`dist`、`out`、`release`、`.git`、`src-tauri/target`、`unpackage`。

## compose

整个项目**只保留根目录一份 `docker-compose.yml`**：中间件、应用、日志组件都在这一份里。它同时是中间件定义与版本 tag 的唯一来源。

```yaml
services:
  postgres:
    image: postgres:<stable>-alpine     # 初始化时取最新稳定版并锁定，勿用浮动 tag
    healthcheck: { test: ['CMD-SHELL', 'pg_isready -U app'], interval: 10s }
    volumes: ['pgdata:/var/lib/postgresql/data']
  redis:
    image: redis:<stable>               # 有状态，同样写死具体 tag
    command: ['redis-server', '--appendonly', 'yes']
    healthcheck: { test: ['CMD', 'redis-cli', 'ping'], interval: 10s }
    volumes: ['redisdata:/data']
  rustfs:
    image: rustfs/rustfs:<stable>       # 同上
    command: ['server', '/data', '--address', '0.0.0.0:9000']
    volumes: ['rustfsdata:/data']
  serve:
    build: { context: ., dockerfile: apps/serve/Dockerfile }
    env_file: [.env]
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
      rustfs: { condition: service_started }
    healthcheck:
      test: ['CMD', 'wget', '-qO-', 'http://localhost:3000/health']
      interval: 15s
  loki:
    image: grafana/loki:3
    command: -config.file=/etc/loki/local-config.yaml
volumes: { pgdata: {}, redisdata: {}, rustfsdata: {} }
```

- **不要复制成多份 compose**：dev / staging / prod / 本地 / CI 都用这同一份，环境差异走 `.env`（必要时叠加 `docker-compose.override.yml` 做局部覆盖），禁止 dev / prod 各 fork 一份。
- 数据卷必须具名 volume，不用 bind mount 到宿主目录（生产）。
- 服务间用 compose 网络的服务名互访：`DATABASE_URL=postgres://app:pwd@postgres:5432/app`、`REDIS_URL=redis://redis:6379`、`S3_ENDPOINT=http://rustfs:9000`。
- 生产密钥走 `.env`（不进仓库）或密钥管理，不用 `environment` 明文写密码。
- **版本单一来源**：中间件 tag 只写在这一份 `docker-compose.yml` 里。各自初始化必然版本漂移：dev 与 prod 落到不同 PG 大版本，而 PG 跨大版本不能复用 `pgdata`，备份回灌直接失败。
- **初始化取值**：中间件镜像**默认取各官方仓库的当前最新稳定版**，把解析到的具体版本写进 `docker-compose.yml` **锁定**。
- **有状态服务禁用浮动 tag**（`latest`、只带 `-alpine` 的 `postgres:alpine` 同样浮动）。`docker compose pull` 会把大版本顶上去；其余有状态组件也有各自的升级前置条件。
- **Redis 镜像许可**：主选用官方 `redis`（8.x 起为 AGPLv3）；需要宽松许可时换 `valkey/valkey`（BSD-3，协议与客户端兼容）。二者都放进同一份 `docker-compose.yml`，切换只改 tag。
- **规范本身不写死版本号**（会腐坏，见 [tooling.md](tooling.md) 的版本约定）。样例里的 `<stable>` 是占位符，初始化时按官方仓库现行稳定版替换并落成具体 tag。
- 无状态 / 构建类镜像只在影响写法与配置时才带大版本：`node:24`（决定可用语法与工具链，须与本地 `.nvmrc` 一致，见 [tooling.md](tooling.md)）、`grafana/loki:3`（3.x 与 2.x 的 config 语义不同）保留大版本，补丁与次版本交给部署侧浮动。

## nginx

```nginx
server {
  listen 80;
  server_name _;

  location / {
    root /usr/share/nginx/html;
    try_files $uri $uri/ /index.html;   # SPA 必须
  }

  location /api/ {
    proxy_pass http://serve:3000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Request-Id $request_id;
    proxy_http_version 1.1;
  }

  location /s3/ {                        # 公开资源回源到对象存储
    proxy_pass http://rustfs:9000/;
  }

  gzip on;
  gzip_types text/css application/javascript application/json image/svg+xml;
  location ~* \.(js|css|woff2|png|webp|svg)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
  }
}
```

- `index.html` 不做长缓存（`Cache-Control: no-cache`），带 hash 的静态资源长缓存。
- API 透传 `X-Request-Id`，与后端 pino 的 `requestId` 对齐。
- WebSocket 需 `proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade";`。
- 大文件上传设 `client_max_body_size`。

## 日志 — Loki

- 容器日志走 stdout（JSON），由 Promtail / Alloy 采集推 Loki；禁止写文件日志。
- pino 输出即是 Loki 可解析的 JSON（见 [backend.md](backend.md)），字段名保持稳定。
- 标签只用低基数字段：`{app="serve", env="prod", level="error"}`；**不要把 requestId / userId 当标签**（高基数），放日志内容里查。
- 前端错误上报单独走 Sentry 或自建端点，不混进 Loki 应用日志。

## 环境与发布

- 三套环境：`dev` / `staging` / `prod`，用同一镜像，靠环境变量区分（`NODE_ENV`、`S3_BUCKET`、`LOG_LEVEL`）。中间件同理——共用同一份根目录 `docker-compose.yml`，大版本在环境间不得分叉（见上方 compose 一节的「版本单一来源」）。
- 迁移在发布时执行：`docker compose run --rm serve pnpm --filter @repo/db migrate`，先迁移后切流量。
- 健康检查 `/health` 必须探测 DB / Redis / 存储连通性。
- 回滚 = 回退镜像 tag，不依赖"再改一次代码"。
