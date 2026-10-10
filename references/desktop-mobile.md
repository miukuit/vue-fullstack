# 桌面端与小程序

## 桌面端 — 主推 tauri

| 场景 | 用 |
|---|---|
| **默认桌面应用** | **tauri v2** |
| 既有 electron 项目 / 必须用 Node 主进程生态 | electron |

选 tauri 的理由：体积小、内存低、Rust 侧安全。除非明确需要 Node 主进程能力，不要建议 electron。

### tauri 约定

- 前端仍是 Vue3 + Vite（复用 `apps/admin` 或独立 `apps/desktop`），`src-tauri/` 放 Rust 侧。
- 前端调后端能力一律走 `@tauri-apps/api` 的 `invoke`，命令名与 Rust `#[tauri::command]` 一一对应。
- 权限：v2 用 capability 文件 `src-tauri/capabilities/*.json` 最小授权，不要图省事开 `core:default` 全开。
- 网络请求优先在前端 axios 发（走 WebView），确需绕 CORS / 用系统代理时才在 Rust 侧发。
- 本地存储：配置用 `tauri-plugin-store`，数据库用 `tauri-plugin-sql`（SQLite）或复用远端 API；不要在前端 localStorage 存敏感数据。
- 自动更新用 `tauri-plugin-updater`，签名密钥走 CI secret。
- 打包：`pnpm tauri build`，Windows 产出 msi/nsis，macOS dmg，Linux deb/AppImage。跨平台构建用 GitHub Actions matrix，不在本机交叉编译。
- 开发命令：`pnpm tauri dev`。

### electron（仅备选）

- 主进程/预加载/渲染进程严格隔离，`contextIsolation: true`、`nodeIntegration: false`。
- 渲染进程通过 `contextBridge.exposeInMainWorld` 暴露白名单 API。
- 打包用 electron-builder，配置在 `electron-builder.yml`。

## 小程序 / 跨端 — uni-app + WotUI

- 项目结构：`pages.json`（路由/分包/easycom）、`manifest.json`（各端配置）、`App.vue`、`pages/`、`components/`。
- UI 一律 WotUI（`@wot-design-uni`），见 [ui.md](ui.md)。
- 条件编译：

```
// #ifdef MP-WEIXIN
wx.login()
// #endif
// #ifdef H5
console.log('h5 only')
// #endif
```

- 分包：主包只放 tabBar 页与首屏，其余进 `subPackages`；主包 <2MB，单包 <2MB，总包 <20MB。
- 请求封装：`uni.request` 包成 promise，复用 `packages/api-client` 的类型；token 存 `uni.getStorageSync`，请求头统一注入。
- 登录：微信 `code` → 后端换 `openid`/session → 下发业务 token；不把 `session_key` 传到前端逻辑层之外。
- 支付：微信 `uni.requestPayment`，订单参数由后端签名，前端不参与签名。
- 图片走 CDN（可用 S3 + 预签名，见 [storage.md](storage.md)），配 `lazy-load` 与 `mode="aspectFill"`。
- 小程序不支持 echarts 原版，用 `lime-echart` 或 WotUI 内置图表；three.js 不支持，需要 3D 用 web-view 或 canvas 手写。
- 性能：`onReachBottom` 分页、长列表用虚拟列表、`setData` 类操作（uni 的响应式赋值）避免大对象整体替换。
- 发布：微信开发者工具上传 → 平台提审；H5 端产物丢给 nginx（见 [devops.md](devops.md)）。

## 多端复用原则

- 可复用：`packages/schemas`（zod）、`packages/api-client`（请求与类型）、`packages/utils`（纯函数：dayjs/decimal 封装）、`packages/ui` 的 token。
- 不复用：DOM 相关 composable、组件库组件、路由实现。
- 判断标准：能否在无 DOM、无 `window` 环境跑。跑不了就不下沉到共享包。
