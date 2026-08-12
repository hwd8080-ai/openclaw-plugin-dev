---
name: openclaw-plugin-dev
description: openclaw 插件开发流水线与核心难点指南。覆盖从明确目的、文件组成、配置、官方文档、构建产物、先发 GitHub、发 ClawHub 到发完验证的完整开发生产线，以及常见问题类型与解决思路。当用户要创建/调试/发布 openclaw 插件时使用。
agent_created: true
---

# openclaw 插件开发流水线与核心难点

基于真实插件（openclaw-branding、session-label-from-sender）在 v2026.7.1 实测可跑的指南。两部分：① **开发生产线**；② **问题类型 + 解决思路**（概要）。

> **总原则（最高优先）**：绝不修改 openclaw 主程序 / 源码。sandbox、权限、iframe 策略是宿主刻意的安全设计，插件用扩展机制在宿主外面做事。宿主问题报官方修。branding 对 control-ui 做 logo/品牌字替换是插件既定目的（运行时换肤），与改宿主源码是两回事。

> **官方文档（schema/SDK 先查）**：开发 https://docs.openclaw.ai/plugins/building-plugins ｜ manifest https://docs.openclaw.ai/plugins/manifest ｜ SDK 入口 https://docs.openclaw.ai/plugins/sdk-entrypoints ｜ ClawHub https://docs.openclaw.ai/clawhub 。本 skill 是 v2026.7.1 实测流水线，字段以官方为准。

## 一、开发生产线

### 1. 明确插件目的
定清楚插件做什么、属哪类 capability：后台管理页 + API（branding）、会话数据同步（session-admin）、agent 工具、channel、provider、定时任务等。目的决定文件组成与 `contracts`。

### 2. 插件文件组成
```
extension-name/
├── openclaw.plugin.json   # manifest（id/activation/contracts/configSchema）
├── package.json           # npm 包 + openclaw 块（extensions/build/compat/install/release）
├── build.mjs              # esbuild 构建脚本
├── index.ts               # 入口：definePluginEntry + register(api)
├── ui.ts                  # 前端页面 HTML/内联 JS/CSS（无 UI 可省）
├── dist/index.mjs         # 构建产物（运行时加载，gitignored）
├── CHANGELOG.md / LICENSE / README.md
└── .gitignore             # 排除 node_modules/ dist/ *.bak index.mjs <state>.json reports/
```

### 3. 配置怎么写

**manifest `openclaw.plugin.json`**：
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
- `id` 是唯一标识（不是 `name`）；`contracts` 声明受信 surface（http 路由用 `gatewayMethodDispatch`，agent 工具用 `tools`）；`configSchema` 即使无配置也要给空对象。

**`package.json` 的 `openclaw` 块**：
```json
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
```
- `extensions` 指向**构建产物** `./dist/index.mjs`（不能指 `.ts` 源码）；`build.openclawVersion` 写具体版本不加 `>=`；`install.clawhubSpec` 必填。

**`~/.openclaw/openclaw.json` 启用**（源码构建本地测试手动加）：
```json
{ "plugins": { "entries": { "<id>": { "enabled": true } } } }
```
- `enabledByDefault:true` 只是运行时判定，**不落盘**；源码 clone 后需手动加 `entries` 才加载。

### 4. 开发入口（index.ts）
```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
export default definePluginEntry({
  id: "<plugin-id>", name: "...", description: "...",
  register(api) {
    const a = api as { session: { controls: any }; registerHttpRoute: any };
    // ① 后台管理页（control UI 多一个 tab，不是 manifest 字段）
    a.session.controls.registerControlUiDescriptor({
      surface: "tab", id: "<tab-id>", label: "菜单名",
      description: "...", path: "/plugins/<id>", icon: "palette", group: "control", order: 50,
    });
    // ② HTTP 路由：页面 + API
    a.registerHttpRoute({ path: "/plugins/<id>",     auth: "plugin", match: "exact",  handler: pageHandler });
    a.registerHttpRoute({ path: "/plugins/<id>/api", auth: "plugin", match: "prefix", handler: apiHandler });
  },
});
```
- 入口 helper 按类型选：非通道插件（provider/tool/hook/管理页+API）用 `definePluginEntry`；**通道插件**用 `defineChannelPluginEntry`（`openclaw/plugin-sdk/channel-core`，自动处理 discovery 分支）；纯工具插件可用 `defineToolPlugin`（`openclaw/plugin-sdk/tool-plugin`）。
- **`register(api)` 必须按 `api.registrationMode` 分支**：discovery/tool-discovery 模式下 OpenClaw 会执行入口模块建快照，**顶层 import 必须无副作用**（禁止顶层启动网络客户端/子进程/DB 连接/监听器/读凭据），重活放进 handler 或按 mode 跳过。
- `handler` 返回 `Promise<boolean>`，**必须 `return true`**，否则请求漏到后续中间件 → 404。
- `manifest id` 必须与入口 `id` 一致；`contracts.tools` 必须与 `api.registerTool` 注册的名字一致（否则 tool discovery 不加载）。
- **注册钩子 `api.registerHook(events, handler, opts)` 第三参 `opts` 必须带 `name`**（如 `{ name: "<id>:my-hook" }`）。缺失 name 时内部 `requireRegistrationValue` 抛 `hook registration missing name`，该异常会**中断 `register(api)`**，导致之后的 `registerHttpRoute` / `registerControlUiDescriptor` 全部不执行 → 管理页 404、gateway 启动列表（"http server listening (N plugins: …)"）不出现本插件。踩坑实例：openclaw-tweaks 首版只用 `api.registerHook("inbound_claim", handler)` 漏了 name，daemon 加载后插件 `inspect` 显示 loaded、但路由 404，直到在 SDK 源码 `registry.ts` 的 `registerHook` 里定位到 name 必填才修好。
- **对话类钩子（`inbound_claim`/`message_received` 等）对非 bundled 插件额外要求信任授权**：必须在 `openclaw.json` 的 `plugins.entries.<id>.hooks` 设 `allowConversationAccess:true`（注入回复/改写消息还要 `allowPromptInjection:true`），否则被**静默拦截**（只 push warn 不注册、不抛异常）。bundled 插件不受此限。
- **mock-import 测试（stub `openclaw` SDK）发现不了上述两类问题**：stub 不校验 name、也不做 conversation-access 拦截，钩子永远"成功"。验证钩子类插件必须**真 daemon 加载**（看 gateway 日志的 "http server listening" 列表是否含本插件 + 有无 `blocked`/`missing name` warn），或读 `openclaw/plugin-sdk` 源码确认签名与门控。
- import 用窄路径 `openclaw/plugin-sdk/<subpath>`，别混用、别从 SDK 路径 import 自己的插件。
- CORS 仍要自己加（见第二部分坑#1）。

### 5. 构建产物（发布前必生成）
```bash
NODE_PATH=<npm_workspace>/node_modules <node22+> node build.mjs
```
- esbuild 把 `index.ts` + 依赖打成自包含 `dist/index.mjs`（运行时**不 npm install**，依赖须 bundled）。
- `dist/` 与根 `index.mjs` gitignore 不入库；但 ClawHub `publish` 扫工作目录**不读 .gitignore**，发布前必须先构建让 dist/ 存在。
- Node ≥ 22.22.3（或 24.15+/25.9+，官方要求）；用 node:sqlite 时需 ≥22.5。

### 6. 本地验证 + 调试
```bash
EXT=~/.openclaw/extensions/<id>
mkdir -p "$EXT/dist"
cp openclaw.plugin.json index.mjs "$EXT/"
cp dist/index.mjs "$EXT/dist/"            # 拷到 dist/ 子目录，别平级（拷错会 serve 旧字节）
# openclaw.json 加 entries（见 3）→ 重启 daemon
kill $(lsof -iTCP:18789 -sTCP:LISTEN -t) 2>/dev/null; sleep 2
openclaw daemon restart; sleep 3
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/plugins/<id>
```
- 运行时只扫 `~/.openclaw/extensions/<id>/`（不是源码仓库的 `extensions/`）；根 `index.mjs` + `dist/index.mjs` 都要部署。
- 构建后 `grep dist/index.mjs` 确认新字符串落地（中文可能被转 `\uXXXX`，Python 解码核对）。
- 排错：stderr 丢 /dev/null，看 `~/Library/Logs/openclaw/gateway.log`；API 用 `curl http://127.0.0.1:18789/plugins/<id>/api/<x>`；DB 用 Python `sqlite3`；前端 DevTools + `Cmd+Shift+R`。

### 7. 先发 GitHub（要求）
发 ClawHub 前必须先发 GitHub（作为 `--source-repo`）。
```bash
gh repo create <name> --public --source=. --remote=origin --push   # 首次
git push origin main                                                # 后续
```
- 仓库只含源码（`dist/`/`index.mjs`/`node_modules` gitignore）。
- README 写「解决了什么问题 + 怎么用」，安装步含 `node build.mjs`。
- 清理硬编码本地路径（`/Users/xxx` → `$HOME`）与 token/邮箱。

### 8. 发 ClawHub
```bash
clawhub package inspect @<owner>/<name>   # 查线上 Latest，新版本必须 > 它
# 改 package.json version → node build.mjs → 本地验证 → 校验
clawhub package validate .                # 必须 0 错 0 警
clawhub package publish . --family bundle-plugin --owner <handle> \
  --changelog "<release-note>" --source-repo <repo-url> --source-commit $(git rev-parse HEAD) --dry-run
# dry-run OK 后去掉 --dry-run 正式发
```
- 必填字段：`openclaw.build.openclawVersion`、`openclaw.compat.pluginApi`、`install.clawhubSpec`。
- `--changelog` 必带（已发布版本不可改 note，漏填只能再升版本重发）。

**版本号规则（对齐官方 `<base>-<tag>.<n>` 模式）— 用 `YYYY.M.D-rc.N` / `-beta.N`**：
- **semver 优先级（ClawHub 实测走严格 semver，不要用 GitHub tag 截图判断）**：同一核心版本下，预发布恒低于稳定版——
  `2026.8.13-beta.1` < `2026.8.13-beta.2` < `2026.8.13-rc.1` < `2026.8.13`(裸稳定)。
  所以 **想用 `-beta`/`-rc` 风格，必须先把预发布发出来，再毕业到裸稳定**；一旦发了裸 `2026.8.13`，就再也回不去 `2026.8.13-*` 的任何后缀（更小 → 被拒）。官方 OpenClaw 仓库的 `v2026.7.1-1`/`v2026.7.1-2` 是 GitHub 自己的 tag 排序习惯，**不等于 ClawHub 优先级**，实测 `-rc.1` 后缀会被当成更小值拒绝。
- 当天首次：`2026.8.15-rc.1`；同一天再发：`-rc.2`、`-rc.3`（prerelease 段递增，semver 内更大）。
- 次日：`2026.8.16-rc.1`（patch 更大，天然 > 前一天所有 rc）。
- **不要先发裸日期** `YYYY.M.D`：发了裸 `2026.8.14` 后，同核心的 `-rc.1`/`-beta.1` 都比它小 → 被 ClawHub 拒，被迫跳到明天版本 → 不合实际。
- 新版本必须 > 线上 Latest，否则被拒（用 `clawhub package inspect` 先看 Latest）。

### 9. 发完验证
```bash
clawhub package inspect @<owner>/<name>            # 确认 Latest = 新版本
openclaw plugins install clawhub:@<owner>/<name>   # 真实安装测试
```
- 刚发布几分钟 `Scan: suspicious` 是过渡态，约 10~15 分钟自动转 `clean`。

## 二、问题类型 + 解决思路（概要）

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
