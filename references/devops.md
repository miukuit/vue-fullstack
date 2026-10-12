# 运维 — Docker / nginx / Loki

## 镜像

前端（多阶段 + nginx）：

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
RUN corepack enable
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY . .
RUN pnpm install --frozen-lockfile && pnpm --filter web build

FROM nginx:1.27-alpine
COPY deploy/nginx/web.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/apps/web/dist /usr/share/nginx/html
```

后端（多阶段 + 非 root）：

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
RUN corepack enable
COPY . .
RUN pnpm install --frozen-lockfile && pnpm --filter @repo/api build && pnpm prune --prod

FROM node:22-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=build --chown=app:app /app .
USER app
EXPOSE 3000
CMD ["node", "apps/api/dist/main.js"]
```

- 必须多阶段，最终镜像不含源码与 devDependencies。
- 必须 `--frozen-lockfile`。
- 以非 root 运行；不加 `--privileged`。
- `.dockerignore` 排除 `node_modules`、`dist`、`out`、`release`、`.git`、`src-tauri/target`、`unpackage`。

## compose

```yaml
services:
  api:
    build: { context: ., dockerfile: apps/api/Dockerfile }
    env_file: [.env]
    depends_on:
      postgres: { condition: service_healthy }
      rustfs: { condition: service_started }
    healthcheck:
      test: ['CMD', 'wget', '-qO-', 'http://localhost:3000/health']
      interval: 15s
  postgres:
    image: postgres:16-alpine
    healthcheck: { test: ['CMD-SHELL', 'pg_isready -U app'], interval: 10s }
    volumes: ['pgdata:/var/lib/postgresql/data']
  rustfs:
    image: rustfs/rustfs:latest
    command: ['server', '/data', '--address', '0.0.0.0:9000']
    volumes: ['rustfsdata:/data']
  loki:
    image: grafana/loki:3.1
    command: -config.file=/etc/loki/local-config.yaml
volumes: { pgdata: {}, rustfsdata: {} }
```

- 数据卷必须具名 volume，不用 bind mount 到宿主目录（生产）。
- 服务间用 compose 网络的服务名互访：`DATABASE_URL=postgres://app:pwd@postgres:5432/app`、`S3_ENDPOINT=http://rustfs:9000`。
- 生产密钥走 `.env`（不进仓库）或密钥管理，不用 `environment` 明文写密码。

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
    proxy_pass http://api:3000/;
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
- 标签只用低基数字段：`{app="api", env="prod", level="error"}`；**不要把 requestId / userId 当标签**（高基数），放日志内容里查。
- 前端错误上报单独走 Sentry 或自建端点，不混进 Loki 应用日志。

## 环境与发布

- 三套环境：`dev` / `staging` / `prod`，用同一镜像，靠环境变量区分（`NODE_ENV`、`S3_BUCKET`、`LOG_LEVEL`）。
- 迁移在发布时执行：`docker compose run --rm api pnpm --filter @repo/db migrate`，先迁移后切流量。
- 健康检查 `/health` 必须探测 DB / 缓存 / 存储连通性。
- 回滚 = 回退镜像 tag，不依赖"再改一次代码"。
