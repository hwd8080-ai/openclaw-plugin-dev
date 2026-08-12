---
name: openclaw-plugin-dev
description: openclaw 插件（extension）开发 SOP 与核心难点指南。覆盖构建部署流水线、API/CORS、前端内联与 sandbox 限制、node:sqlite、原生数据访问、定时任务、ClawHub 发布，以及 sandbox 竞态/主题/模态框/SW 缓存等平台级坑的概要。当用户要创建/调试/发布 openclaw 插件时使用。
agent_created: true
---

# openclaw 插件开发 SOP 与核心难点

基于多个插件真实经验。**通用**，不绑具体业务。本 skill 只做两件事：① 跑通插件的 SOP 流水线；② 核心难点/报错**概要**描述（给方向，不展开长代码）。

> **总原则（最高优先）**：绝不修改 openclaw 主程序 / 源码。sandbox、权限、iframe 策略都是宿主刻意的安全设计。插件用扩展机制在宿主**外面**做事。宿主问题（如冷刷新 sandbox 竞态）报官方修。branding 对 control-ui 做 logo/品牌字替换是插件既定目的（运行时换肤），与改宿主源码/安全逻辑是两回事。

## 一、SOP 流水线（每次改代码必走）

```bash
# 1. 构建：esbuild 自包含 bundle（运行时不执行 npm install）
NODE_PATH=<npm_workspace>/node_modules <node22+> node build.mjs

# 2. 部署到【运行时】扩展目录（注意：不是 openclaw 源码仓库的 extensions/）
EXT=~/.openclaw/extensions/<name>
mkdir -p "$EXT/dist"
cp openclaw.plugin.json index.mjs "$EXT/"      # 根 index.mjs 必带
cp dist/index.mjs "$EXT/dist/"                 # dist/index.mjs 拷到 dist/ 子目录，别平级

# 3. openclaw.json 显式启用（否则插件不加载，日志报 plugin not found）
plugins.entries["<name>"] = { "enabled": true }

# 4. 重启 daemon（先清旧端口占用更稳）
kill $(lsof -iTCP:18789 -sTCP:LISTEN -t) 2>/dev/null; sleep 2
openclaw daemon restart; sleep 3

# 5. 验证
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/plugins/<name>
```

要点：
- 运行时只扫描 `~/.openclaw/extensions/<id>/`；源码仓库的 `extensions/` 不被加载。
- 运行时 bundle（根 `index.mjs` + `dist/index.mjs`）不入库（`.gitignore` 配 `dist/` 与根 `index.mjs`）；README 安装步必含 `node build.mjs`，否则 clone 用户拿不到运行时文件。
- Node ≥ 22.5（node:sqlite 要求）。stderr 丢 /dev/null，排错看 `~/Library/Logs/openclaw/gateway.log`（stdout）。

## 二、插件元数据

- `openclaw.plugin.json`：`name`/`version`/`description`/`main`(`dist/index.mjs`)/`adminPage`(注册后台页，纯 API 可省)。
- `package.json`：ClawHub 强制字段见第六章。

## 三、API 路由与 auth

`api.registerRoute({ path:"/plugins/<name>/api/*", handler:(req,res)=>{ /* ... */ return true; }, auth:"plugin" })`。

- **必须 `return true`**，否则请求漏到后续中间件 → 页面/接口不生效或 404。
- `auth:"plugin"`：iframe 直连后台免二次 token，最方便。

**CORS 坑（sandbox iframe，origin="null"）**：插件页被 `<iframe sandbox="allow-scripts">` 嵌入（无 `allow-same-origin`），页内 fetch 后台即跨域。
- 前端 fetch **不显式设 `content-type`**（字符串 body 浏览器自动 `text/plain`，属简单请求，免 OPTIONS 预检）。
- 后端每个响应 + OPTIONS 预检统一返回 `Access-Control-Allow-Origin/Methods/Headers: *`。

## 四、前端 UI 要点

- **内联 JS 引号转义**：外层双引号 → 内层 HTML 属性双引号写成 `\"`；Python 批量改用原始三引号 `r'''...'''`。
- **SSR 降级（强烈推荐）**：数据直接渲染进 HTML，导航/筛选/导出用 `<a>` 链接或链接参数，脚本仅增强。strict 沙盒下首屏也完整可用（应对第九章坑#1）。
- **主题/i18n 限制**：插件无法跟随 openclaw 明暗/语言（沙盒无 allow-same-origin，读不到宿主 DOM/localStorage）。要暗色/多语只能读 OS 偏好（`prefers-color-scheme`/`navigator.language`）。
- **视觉对齐**：iframe 读不到 openclaw stylesheet，必须自己内联样式复刻。官方暖色浅色调色板：--ink `#252421`、--muted `#78746d`、--line `#e8e3da`、--paper `#fffefa`、--canvas `#f5f2ec`、--red `#d84a38`、--teal `#168f89`；标题砖红 `#bd4531`；卡片圆角 14px 默认无阴影 hover 出；输入框 `max-width:340px`。照搬这套数值，别自己拍配色。
- **模态框失效**：sandbox 禁 alert/confirm/prompt，改用页面内 non-modal 元素提示（见第九章坑#3）。
- **客户端用 npm 库**：浏览器脚本不能 `import`，用 esbuild 单独打包 IIFE 挂 `globalThis` 再内联（esbuild 的 `nodePaths` 指向 managed workspace 的 `node_modules`）。
- **无限滚动**：必须增量插入（`insertAdjacentHTML`），禁止整列表 `innerHTML` 重渲染（否则闪屏抖动）。

## 五、数据层 node:sqlite 坑

Node ≥ 22.5 的 `DatabaseSync` 为实验 API：
- **UPDATE 命名参数绑定有时不写入** → 改用 `db.exec()` 内联 SQL + 手动转义单引号（`'`.replace(/'/g,"''")）。
- **SELECT 必须用 `prepare() + all()/get()`**（`db.exec("SELECT")` 返回 undefined）。
- 无 `.transaction()`，批量写循环 `run()`。
- 去重：`PRIMARY KEY(...)` + `INSERT OR IGNORE`。

## 六、访问 openclaw 原生数据

- 转录两种格式：标准 `.jsonl`（`message` 信封）/ 轨迹 `.trajectory.jsonl`（`model.completed.data.messagesSnapshot[]`，过滤 `system`）。
- **坑 A：multi-agent 必须传 agentId**。`listSessionEntries()` 不传只回 `main`，子 agent（work/code 等）会话拉不到 → 遍历 `agents/<id>/sessions/`。
- **坑 B：trajectory 去重**——同会话多个 `model.completed` 的 seq 不唯一、snapshot 是累计超集 → 用 snapshot **下标** `${sessionId}-m${i}` 作 ID + `INSERT OR IGNORE`（用 seq 会翻倍）。
- **近实时同步**：读数据的 handler 返回前先跑轻量 registry 同步（发现新行即现），全量内容同步只作后台兜底。两层频率不同，别混。

## 七、定时任务

`const timer = setInterval(doWork, n)`。extension 型插件**无 `onDisable`/`onUnload` 钩子**（channel 型才有），删除/disable 定时器**不自动停**（仅 daemon 退出释放）。→ 在回调里自查退出条件（如文件/配置是否变化），或明确告知用户改配置后需 `openclaw daemon restart`。

## 八、ClawHub 发布

- **必填字段**：`openclaw.build.openclawVersion`（具体版本，不加 `>=`）、`openclaw.compat.pluginApi`、`install.clawhubSpec`；`repository` 填 git url。
- **流程**：清理硬编码本地路径（`/Users/xxx` → `$HOME`）→ 升 version → `node build.mjs` → 本地部署 + daemon 重启 + **实际验证页面/接口** → `clawhub package validate .`（必须 0 错 0 警）→ `clawhub package publish . --family bundle-plugin --owner <h> --changelog "<note>" --source-repo <r> --source-commit $(git rev-parse HEAD)`（先 `--dry-run`）。
- `--changelog` 必带；已发布版本不可改 release note（漏填只能再升版本重发）。
- 发布后 `clawhub package inspect` 确认 Latest=新版本；刚发布几分钟 `Scan: suspicious` 是过渡态，约 10~15 分钟自动转 `clean`。

## 九、平台级坑（插件侧改不了，需知晓方向）

- **坑#1 iframe sandbox 竞态**：硬刷新报 `Blocked script execution ... allow-scripts is not set`，切走再切回正常。根因在宿主（config 异步到达、已加载文档不追溯生效），插件改不了。根治 = SSR 降级（第四节）；**绝不改宿主**。可用 `<noscript>` 提示作缓解：`.app{display:none}` + 末尾防御式 reveal 脚本 `var _a=document.getElementById('app'); if(_a)_a.style.display='flex'`（容器必须真有 `id="app"`，否则整页空白）。
- **坑#2 主题/i18n 不跟随**：iframe `src` 不传 theme/lang、沙盒读不到宿主状态 → 插件不能跟随。只能跟 OS 偏好。
- **坑#3 模态框失效**：sandbox 无 `allow-modals` → alert/confirm/prompt 全失效，console 报 `Ignored call to 'alert()'...`。改用页面内 non-modal 提示（查源码确认无 `alert(/confirm(/prompt(` 残留）。

## 十、浏览器缓存与 SW（核心难点概要）

> **经验法则**：顽固浏览器侧问题（缓存/SW/渲染/鉴权/跨域）直接用 ego-browser 真实浏览器实测闭环，别只在 DevTools 猜。

「URL 没变就命中旧字节」来自**三层独立缓存**，任一没管住都复发：
1. **HTTP 缓存**：`cache:'no-store'` 能管，不覆盖下两类。
2. **SW fetch 缓存（Cache Storage）**：SW 写入的旧缓存，新 SW 不删就永久沉积。
3. **favicon 独立缓存**：不走 Cache-Control/fetch cache，**唯一根治 = URL 变化（`?v=stamp`）**。

关键教训：
- 不同资源是独立字节通道（如 chat 头像 vs branding 图标），只修一处覆盖不到另一处。
- SW `CACHE_PREFIX` 别假设都按你前缀开头 → `activate` 删【所有】非当前缓存（`cacheKeys.filter(k=>k!==CACHE_NAME)`）。
- SW fetch 失败**绝不 fallback 到 `caches.match`**（否则刷新偶吐旧值）→ 直接 `502`。
- 验证：用 ego-browser 构造旧缓存 → 复现 → 注入修复 → 验证清除（形成闭环证据）。

## 十一、调试技巧

- 构建：`grep dist/index.mjs` 确认新字符串落地（中文可能被转 `\uXXXX`，Python 解码核对）。
- API：`curl http://127.0.0.1:18789/plugins/<name>/api/<x>`。
- DB：Python `sqlite3` 读 `.db`。
- 前端：DevTools Console + `Cmd+Shift+R` 强刷。
- 进程：`openclaw daemon restart` 加载新 bundle。
