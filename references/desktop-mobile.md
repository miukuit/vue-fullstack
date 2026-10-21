# 桌面端与小程序

## 桌面端 — 主推 electron

| 场景 | 用 |
|---|---|
| **默认桌面应用** | **electron** |
| 体积/内存敏感、要求 Rust 侧安全边界 | tauri v2 |

选 electron 的理由：Node 主进程生态完整（native 能力可直接复用 npm 包）、进程模型与调试工具链成熟、团队上手成本低。除非明确要求极小体积或 Rust 侧安全沙箱，不要建议 tauri。

### electron 约定

- 前端仍是 Vue3 + Vite（独立 `apps/desktop`），主进程 / 预加载 / 渲染进程三段严格隔离。
- **安全基线（不可放宽）**：
  - `contextIsolation: true`、`nodeIntegration: false`、`sandbox: true`
  - 渲染进程只能通过 `contextBridge.exposeInMainWorld` 暴露**白名单** API，禁止整体暴露 `ipcRenderer`
  - 禁用 `remote` 模块；不要为了省事关掉 `webSecurity`
- IPC 命名固定前缀，channel 名**集中登记在一个常量文件**、两端引用同一份，禁止散落字符串：
  - 渲染 → 主：`invoke('域:动作', payload)` ↔ `ipcMain.handle('域:动作', handler)`
  - 主 → 渲染：`webContents.send('域:事件', data)` ↔ `ipcRenderer.on(...)`
- **主进程不信任渲染进程**：IPC 入参一律用 zod 校验（复用 `packages/schemas`）。渲染进程是主要被攻击面。
- 网络请求在前端 axios 发（渲染进程）；只有需绕 CORS、走系统代理或读写本地文件时才在主进程发。
- 本地存储：配置用 `electron-store`，数据库用 SQLite（`better-sqlite3`）。token 存主进程并用 `safeStorage` 加密，不要放前端 localStorage。
- 自动更新用 `electron-updater`，签名密钥走 CI secret。
- **不在主进程做重 CPU 计算**（会卡死整个应用），用 `utilityProcess` 或 worker。
- 崩溃与异常上报：`render-process-gone`、`child-process-gone`、`uncaughtException` 统一采集，日志走 Loki（见 [devops.md](devops.md)）。
- 打包用 `electron-builder`，配置在 `electron-builder.yml`。Windows 产出 nsis/msi，macOS dmg（需签名+公证），Linux AppImage/deb。跨平台构建用 GitHub Actions matrix，不在本机交叉编译。
- 构建工具推荐 `electron-vite`（自带主/预加载/渲染三份配置与 HMR）：`pnpm dev` / `pnpm build`。

### tauri（备选）

- 体积/内存敏感或要求 Rust 侧安全沙箱时改用。前端调 Rust 走 `@tauri-apps/api` 的 `invoke`，命令名与 `#[tauri::command]` 一一对应。
- v2 用 capability 文件 `src-tauri/capabilities/*.json` 最小授权，不要图省事开 `core:default` 全开。
- 打包 `pnpm tauri build`，开发 `pnpm tauri dev`。

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
- 请求封装：`uni.request` 包成 promise，复用 `packages/http` 的拦截器与请求配置（类型来自 `packages/schemas`）；token 存 `uni.getStorageSync`，请求头统一注入。
- 登录：微信 `code` → 后端换 `openid`/session → 下发业务 token；不把 `session_key` 传到前端逻辑层之外。
- 支付：微信 `uni.requestPayment`，订单参数由后端签名，前端不参与签名。
- 图片走 CDN（可用 S3 + 预签名，见 [storage.md](storage.md)），配 `lazy-load` 与 `mode="aspectFill"`。
- 小程序不支持 echarts 原版，用 `lime-echart` 或 WotUI 内置图表；three.js 不支持，需要 3D 用 web-view 或 canvas 手写。
- 性能：`onReachBottom` 分页、长列表用虚拟列表、`setData` 类操作（uni 的响应式赋值）避免大对象整体替换。
- 发布：微信开发者工具上传 → 平台提审；H5 端产物丢给 nginx（见 [devops.md](devops.md)）。

## 多端复用原则

- 可复用：`packages/schemas`（zod）、`packages/http`（axios 实例与拦截器）、`packages/utils`（纯函数：dayjs/decimal 封装）、`packages/ui` 的 token。
- 不复用：DOM 相关 composable、组件库组件、路由实现。
- 判断标准：能否在无 DOM、无 `window` 环境跑。跑不了就不下沉到共享包。
