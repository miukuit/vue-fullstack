# 对象存储 — RustFS（主）/ MinIO（备）

两者都是 **S3 兼容**，代码层统一用 `@aws-sdk/client-s3`（或 `@aws-sdk/s3-request-presigner`）。差别只在 endpoint、region、path-style。

## 统一客户端

```ts
import { S3Client } from '@aws-sdk/client-s3'

export const s3 = new S3Client({
  endpoint: process.env.S3_ENDPOINT!,          // RustFS: http://rustfs:9000
  region: process.env.S3_REGION ?? 'us-east-1',
  forcePathStyle: true,                        // 自建 S3 必须开
  credentials: {
    accessKeyId: process.env.S3_ACCESS_KEY!,
    secretAccessKey: process.env.S3_SECRET_KEY!,
  },
})
```

环境变量固定：`S3_ENDPOINT` / `S3_REGION` / `S3_ACCESS_KEY` / `S3_SECRET_KEY` / `S3_BUCKET` / `S3_PUBLIC_BASE_URL`。
**禁止在代码里写死 MinIO 或 RustFS 的地址，也禁止把 bucket 名散落在业务代码**。

## Bucket 规划

| bucket | 用途 | 访问策略 |
|---|---|---|
| `public` | 公开静态资源（头像、商品图） | 公共读 |
| `private` | 用户上传、合同、导出文件 | 私有，仅签名 URL |

- 对象 key 规则：`<env>/<biz>/<yyyy>/<MM>/<uuid>.<ext>`，例如 `prod/avatar/2026/10/9f1c....webp`。
- 文件名用 uuid，不用原始文件名（防注入、防冲突）；原始名存在数据库。
- 上传前校验 MIME + 后缀白名单 + 体积上限；图片做压缩转 webp。

## 上传

- 小文件（<5MB）：后端代理上传 `PutObjectCommand`，或直接返回 `PutObjectCommand` 的预签名 URL 由前端直传。
- 大文件：`@aws-sdk/lib-storage` 的 `Upload`（自动分片）：

```ts
import { Upload } from '@aws-sdk/lib-storage'
await new Upload({ client: s3, params: { Bucket, Key, Body: stream, ContentType } }).done()
```

- 前端直传流程：前端请求后端拿预签名 PUT URL → 前端 `axios.put(url, file)` → 前端回调后端登记 key。不把 AK/SK 下发到前端。

## 下载 / 预览

- 私有对象通过预签名 URL 访问，有效期默认 10 分钟：

```ts
import { getSignedUrl } from '@aws-sdk/s3-request-presigner'
import { GetObjectCommand } from '@aws-sdk/client-s3'

const url = await getSignedUrl(s3, new GetObjectCommand({ Bucket, Key }), { expiresIn: 600 })
```

- 公开资源走 `S3_PUBLIC_BASE_URL` 拼接，经 nginx/CDN 回源，不直连存储内网地址。
- 大文件下载响应加 `Content-Disposition` 控制内联/附件。

## 删除与生命周期

- 逻辑删除为主：DB 标记 `deletedAt`，对象由生命周期规则清理（RustFS/MinIO 均支持 S3 lifecycle）。
- 批量删除用 `DeleteObjectsCommand`（一次最多 1000）。
- 生命周期策略：临时导出文件 7 天，回收站 30 天，日志归档 180 天。

## 安全

- Bucket 默认**私有**，公开读必须显式配置且只对 `public` bucket。
- 生产禁止开启匿名 ListObjects。
- 签名 URL 不落库、不写日志（日志脱敏 `*.X-Amz-Signature`）。
- 上传文件名/内容做类型校验，防止上传 html/svg 导致 XSS；svg 一律转 png 或强制下载。
