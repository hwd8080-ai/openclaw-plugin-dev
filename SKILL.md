---
name: openclaw-plugin-dev
description: openclaw 插件开发流水线与核心难点指南。覆盖从明确目的、文件组成、manifest/配置、官方文档参考、构建产物、先发 GitHub、发 ClawHub 到发完验证的完整开发生产线，以及开发中常见问题类型与解决思路。当用户要创建/调试/发布 openclaw 插件时使用。
agent_created: true
---

# openclaw 插件开发流水线与核心难点

基于真实插件（openclaw-branding 换肤、session-label-from-sender 会话归档）在 v2026.7.1 实测可跑的实操指南。两部分：① 完整**开发生产线**；② 开发中遇到的**问题类型 + 解决思路**（概要，给方向不展开）。

> **总原则（最高优先）**：绝不修改 openclaw 主程序 / 源码。sandbox、权限、iframe 策略是宿主刻意的安全设计。插件用扩展机制在宿主**外面**做事。宿主问题（如冷刷新 sandbox 竞态）报官方修。branding 对 control-ui 做 logo/品牌字替换是插件既定目的（运行时换肤），与改宿主源码/安全逻辑是两回事。

> **官方文档（权威参考，schema/SDK 问题先查）**：
> - 插件开发总指南：https://docs.openclaw.ai/plugins/building-plugins
> - manifest 字段：https://docs.openclaw.ai/plugins/manifest
> - SDK 入口点：https://docs.openclaw.ai/plugins/sdk-entrypoints
> - ClawHub 发布：https://docs.openclaw.ai/clawhub
>
> SDK 在演进，manifest 字段以官方文档为准；本 skill 记录的是 v2026.7.1 实测流水线。

## 一、开发生产线

### 1. 明确插件目的
先定清楚插件做什么、属于哪类 capability：后台管理页 + API（如 branding）、会话数据同步（如 session-admin）、agent 工具、channel、provider、定时任务等。目的决定文件组成与 `contracts` 声明。

### 2. 插件文件组成
```
extension-name/
├── openclaw.plugin.json   # manifest（id/activation/contracts/configSchema）
├── package.json           # npm 包配置 + openclaw 块（extensions/build/compat/install/release）
├── build.mjs              # esbuild 构建脚本（自包含 bundle）
├── index.ts               # 后端入口：definePluginEntry + register(api)
├── ui.ts                  # 前端页面 HTML/内联 JS/CSS（无 UI 可省）
├── dist/index.mjs         # 构建产物（运行时实际加载，gitignored）
├── CHANGELOG.md           # 发布 changelog 的事实源
├── LICENSE / README.md    # 仓库必备
└── .gitignore             # 排除 node_modules/ dist/ *.bak index.mjs <state>.json reports/
```

### 3. openclaw 插件配置怎么写

**manifest `openclaw.plugin.json`**（实测样例）：
```json
{
  "id": "openclaw-branding",
  "enabledByDefault": true,
  "activation": { "onStartup": true },
  "name": "OpenClaw Branding",
  "description": "...",
  "contracts": { "gatewayMethodDispatch": ["authenticated-request"] },
  "configSchema": { "type": "object", "additionalProperties": false, "properties": {} }
}
```
- `id` 是插件唯一标识（不是 `name`）；`activation.onStartup` 控制 gateway 启动时加载；`contracts` 声明受信 surface（注册 http 路由用 `gatewayMethodDispatch`，注册 agent 工具用 `tools`）；`configSchema` 即使无配置也要给空对象。完整字段见官方 manifest 页。

**`package.json` 的 `openclaw` 块**：
```json
{
  "name": "@<owner>/<plugin-name>",
  "version": "2026.8.14",
  "type": "module",
  "openclaw": {
    "extensions": ["./dist/index.mjs"],
    "build": { "openclawVersion": "2026.7.1" },
    "compat": { "pluginApi": ">=2026.7.1" },
    "install": {
      "npmSpec": "@<owner>/<plugin-name>",
      "clawhubSpec": "@<owner>/<plugin-name>",
      "localPath": "extensions/<plugin-name>",
      "defaultChoice": "clawhub",
      "minHostVersion": ">=2026.7.1"
    },
    "release": { "publishToClawHub": true, "publishToNpm": false }
  }
}
```
- `extensions` 指向**构建产物** `./dist/index.mjs`（发布插件必须指 built JS，不能指 `.ts` 源码）；`build.openclawVersion` 写具体版本不加 `>=`；`install.clawhubSpec` 是 ClawHub 识别安装目标的必填项。

**`~/.openclaw/openclaw.json` 启用**（源码构建本地测试时手动加）：
```json
{ "plugins": { "entries": { "<id>": { "enabled": true } } } }
```
- `enabledByDefault:true` 只是运行时判定，**不落盘**写 openclaw.json；源码 clone 构建后需手动加 `entries` 才加载。

### 4. 开发入口（index.ts）
```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "<plugin-id>",
  name: "...",
  description: "...",
  register(api) {
    const a = api as { session: { controls: any }; registerHttpRoute: any };
    // ① 注册后台管理页（control UI 多一个 tab）
    a.session.controls.registerControlUiDescriptor({
      surface: "tab", id: "<tab-id>", label: "菜单名",
      description: "...", path: "/plugins/<id>", icon: "palette",
      group: "control", order: 50,
    });
    // ② 注册 HTTP 路由：页面 + API
    a.registerHttpRoute({ path: "/plugins/<id>",     auth: "plugin", match: "exact", handler: pageHandler });
    a.registerHttpRoute({ path: "/plugins/<id>/api", auth: "plugin", match: "prefix", handler: apiHandler });
  },
});
```
- 入口必须用 `definePluginEntry` 包裹；路由用 `a.registerHttpRoute`（不是旧的 `api.registerRoute`）。
- `handler` 返回 `Promise<boolean>`，**必须 `return true`** 表示已处理，否则请求漏到后续中间件 → 404。`match:"exact"` 用于页面、`"prefix"` 用于 API 子路径。
- 后台管理页**不是** manifest 字段，而是在代码里 `registerControlUiDescriptor` 注册。
- CORS 仍要自己加（见第二部分坑#1）。

### 5. 构建产物（发布前必生成）
```bash
NODE_PATH=<npm_workspace>/node_modules <node22+> node build.mjs
```
- esbuild 把 `index.ts` + 依赖打成自包含 `dist/index.mjs`（openclaw 运行时**不 npm install**，所有依赖须 bundled）。
- `dist/index.mjs` 就是 `package.json openclaw.extensions` 指向的运行时入口——**这是发布到 ClawHub 必须生成的文件**。
- `dist/` 与根 `index.mjs` gitignore 不入库；但 ClawHub `publish` 扫工作目录**不读 .gitignore**，所以发布前必须先 `node build.mjs` 让 dist/ 存在。
- Node ≥ 22.5（用 node:sqlite 时）。

### 6. 本地验证（开发期迭代）
```bash
# 部署到运行时扩展目录（不是 openclaw 源码仓库的 extensions/）
EXT=~/.openclaw/extensions/<id>
mkdir -p "$EXT/dist"
cp openclaw.plugin.json index.mjs "$EXT/"
cp dist/index.mjs "$EXT/dist/"
# openclaw.json 加 entries（见 3）→ 重启 daemon（先清旧端口更稳）
kill $(lsof -iTCP:18789 -sTCP:LISTEN -t) 2>/dev/null; sleep 2
openclaw daemon restart; sleep 3
# 验证
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/plugins/<id>
```
- 运行时只扫 `~/.openclaw/extensions/<id>/`；根 `index.mjs` 与 `dist/index.mjs` 都要部署（位置别拷错）。
- stderr 丢 /dev/null，排错看 `~/Library/Logs/openclaw/gateway.log`。

### 7. 先发 GitHub（要求）
发布 ClawHub 前必须先发 GitHub（源码仓库作为 `--source-repo`）。
```bash
gh repo create <name> --public --source=. --remote=origin --push   # 首次
git push origin main                                                # 后续
```
- 仓库只含源码，`dist/`/`index.mjs`/`node_modules` gitignore 不入库。
- README 写清「解决了什么问题 + 怎么用」，安装步含 `node build.mjs`（否则 clone 用户拿不到运行时文件）。
- 清理硬编码本地路径（`/Users/xxx` → `$HOME`）与 token/邮箱。

### 8. 发 ClawHub
```bash
# 版本号必须高于线上：先查
clawhub package inspect @<owner>/<name>
# 改 package.json version → node build.mjs（生成 dist）→ 本地验证 → 校验
clawhub package validate .                # 必须 0 错 0 警
# 先 --dry-run 再正式（--changelog 必带，已发布版本不可改 note）
clawhub package publish . --family bundle-plugin --owner <handle> \
  --changelog "<release-note>" --source-repo <repo-url> --source-commit $(git rev-parse HEAD) --dry-run
clawhub package publish . --family bundle-plugin --owner <handle> \
  --changelog "<release-note>" --source-repo <repo-url> --source-commit $(git rev-parse HEAD)
```
- 必填字段：`openclaw.build.openclawVersion`、`openclaw.compat.pluginApi`、`install.clawhubSpec`。
- `--changelog` 不是可选项，漏填只能再升版本重发。

### 9. 发完验证
```bash
clawhub package inspect @<owner>/<name>            # 确认 Latest = 新版本
openclaw plugins install clawhub:@<owner>/<name>   # 真实安装测试
```
- 刚发布几分钟 `Scan: suspicious` 是过渡态，约 10~15 分钟自动转 `clean`，不必惊慌但要复查。

## 二、开发中遇到的问题类型 + 解决思路（概要）

> 真实踩过的坑，给方向不展开。

1. **CORS / sandbox iframe（origin="null"）**：插件页被 `<iframe sandbox="allow-scripts">` 嵌入（无 allow-same-origin），fetch 后台即跨域。→ 后端每个响应 + OPTIONS 预检统一返回 `Allow-Origin/Methods/Headers:*`；前端 fetch 不显式设 `content-type`（走简单请求免预检）。
2. **iframe sandbox 竞态（首屏禁脚本）**：硬刷新报 `Blocked script execution ... allow-scripts is not set`，切走再切回正常。根因在宿主（config 异步到达、已加载文档不追溯生效），插件改不了。→ SSR 降级（数据渲染进 HTML、导航用 `<a>` 链接，脚本仅增强）；**绝不改宿主**。`<noscript>` 提示作 UX 缓解。
3. **主题/i18n 不跟随**：iframe `src` 不传 theme/lang、沙盒读不到宿主状态。→ 只能跟 OS 偏好（`prefers-color-scheme`/`navigator.language`）。
4. **模态框失效**：sandbox 无 allow-modals → alert/confirm/prompt 全失效。→ 改页面内 non-modal 提示（按钮附近首屏可见）。
5. **node:sqlite 坑**：UPDATE 命名参数绑定有时不写入；SELECT 用 `db.exec` 返回 undefined；无 `.transaction()`。→ UPDATE 改 `db.exec()` 内联 SQL + 手动转义单引号；SELECT 用 `prepare()+all()/get()`；批量写循环 `run()`；去重用 `PRIMARY KEY` + `INSERT OR IGNORE`。
6. **访问原生数据**：multi-agent 须传 `agentId`（不传只回 main）；trajectory 的 seq 不唯一、snapshot 是累计超集 → 用 snapshot 下标 `${sessionId}-m${i}` 作 ID 去重。近实时：读数据 handler 返回前先跑轻量 registry 同步，全量同步后台兜底。
7. **定时任务不自动停**：extension 型无 `onDisable` 钩子，删除/disable 定时器不停（仅 daemon 退出释放）。→ 回调自查退出条件，或告知用户改配置后 `openclaw daemon restart`。
8. **浏览器缓存/SW 三层独立**：URL 没变就命中旧字节来自 HTTP 缓存 / SW Cache Storage / favicon 独立缓存三层，任一没管住都复发。→ SW `activate` 删【所有】非当前缓存（别只删自己前缀）；fetch 失败直接 502 不回退旧缓存；favicon 靠 URL 加 `?v=stamp` 失效。顽固问题用 ego-browser 真实浏览器实测闭环（构造旧缓存→复现→注入修复→验证清除）。
9. **视觉对齐**：iframe 读不到 openclaw stylesheet，必须自己内联样式复刻。官方暖色浅色调色板：--ink `#252421`、--muted `#78746d`、--line `#e8e3da`、--paper `#fffefa`、--canvas `#f5f2ec`、--red `#d84a38`、--teal `#168f89`；标题砖红 `#bd4531`；卡片圆角 14px 默认无阴影 hover 出；输入框 `max-width:340px`。
10. **构建/部署位置**：运行时加载扩展根目录 `index.mjs`，`dist/index.mjs` 要拷到 `dist/` 子目录（别平级）；拷错位置会继续 serve 旧字节。构建后 `grep dist/index.mjs` 确认新字符串落地（中文可能被转 `\uXXXX`，Python 解码核对）。

## 三、调试速查
- 构建：`grep dist/index.mjs` 确认新代码落地。
- API：`curl http://127.0.0.1:18789/plugins/<id>/api/<x>`。
- DB：Python `sqlite3` 读 `.db`。
- 前端：DevTools Console + `Cmd+Shift+R` 强刷。
- 进程：`openclaw daemon restart` 加载新 bundle。
