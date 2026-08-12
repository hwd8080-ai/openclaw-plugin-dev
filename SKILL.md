---
name: openclaw-plugin-dev
description: openclaw 插件（extension）从搭建、构建、部署、调试到 ClawHub 发布的一站式开发指南。覆盖通用项目结构、API 路由与 auth、前端 UI 内联、node:sqlite 兼容性、访问 openclaw 原生数据（JSONL/多 agent/trajectory 去重）、iframe sandbox 竞态与主题/国际化平台限制，以及 ClawHub 发布的字段要求、版本号与 git rebase 等避坑经验。当用户需要创建/修改/调试/发布任意 openclaw 插件，或遇到构建、部署、API、SQLite、前端内联、ClawHub 发布等问题时使用。
agent_created: true
---

# openclaw 插件开发一站式指南

基于多个 openclaw 插件的真实开发与发布经验总结。**适用于任意 openclaw 插件**（后台页面、API、定时任务、数据同步等），不绑定具体业务。按"新建 → 开发 → 调试 → 发布"一条线组织。

> **总原则（最高优先级）：绝不修改 openclaw 主程序 / 源码。** openclaw 是成熟的宿主程序，其 sandbox、权限、iframe 策略等都是刻意的安全设计，不是 bug。做插件的目的就是用扩展机制在宿主**外面**做事，绝不可 fork 官方源码、重新打包 control-ui、改动 `embedSandboxMode` / plugin-page / 安全逻辑来"修"宿主 UX。宿主若有问题（如冷刷新的 sandbox 竞态），只能报到 openclaw 官方由其修复。branding 插件对 control-ui 文件做 logo / 品牌字替换，是**插件既定目的（运行时换肤）**，与"修改宿主源码 / 安全逻辑"是两回事，不可混淆——后者一律禁止。

## 一、插件项目结构（通用模板）

```
extension-name/
├── openclaw.plugin.json   # 插件元数据（名称、版本、adminPage 等）
├── package.json           # npm 包配置（ClawHub 有强制字段，见第九章）
├── build.mjs              # esbuild 构建脚本（自包含 bundle）
├── index.ts               # 后端入口：API 路由、定时任务、插件注册
├── ui.ts                  # 前端页面：HTML + 内联 JS/CSS（无 UI 可省略）
├── db.ts / sync.ts        # 数据层 / 同步逻辑（按需拆分）
└── dist/
    └── index.mjs          # esbuild 构建产物（运行时实际加载）
```

## 二、构建与部署（铁律：每次改代码必走）

```bash
# 1. 构建：esbuild 打包为自包含 bundle（openclaw 运行时不会 npm install）
NODE_PATH=<npm_workspace>/node_modules <node22+> node build.mjs

# 2. 部署到【运行时】扩展目录（注意：不是 openclaw 源码仓库的 extensions/ 目录！）
#    运行时只扫描 ~/.openclaw/extensions/<id>/；源码仓库的 extensions/ 不会被加载。
EXT=~/.openclaw/extensions/<name>
mkdir -p "$EXT/dist"
cp openclaw.plugin.json "$EXT/"
cp index.mjs "$EXT/"            # 必须：运行时加载的是扩展根目录的 index.mjs
cp dist/index.mjs "$EXT/dist/"  # 也带上 dist 副本，保持与 build 产物一致

# 3. 在 openclaw.json 中显式启用（否则插件不会被加载，日志报 plugin not found）
#    plugins.entries 下加一行："<name>": { "enabled": true }
python3 - <<'PY'
import json
p=os.path.expanduser("~/.openclaw/openclaw.json")
d=json.load(open(p)); d.setdefault("plugins",{}).setdefault("entries",{})["<name>"]={"enabled":True}
json.dump(d,open(p,"w"),ensure_ascii=False,indent=2)
PY

# 4. 重启 daemon 加载新代码
openclaw daemon restart

# 5. 验证（见第七章）
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/plugins/<name>
```

- 必须用打包工具：openclaw 运行时**不执行 `npm install`**，所有依赖须 bundled 进 `index.mjs`。
- 源码仓库**只含 TS 源码**：根 `index.mjs` 与 `dist/index.mjs` 是 esbuild 产物，**不入库**（`.gitignore` 配 `dist/`；根 `index.mjs` 也别提交）。外部 clone 后必须先 `node build.mjs` 再部署——README「安装」章节务必含此步，否则 clone 用户拿不到运行时文件。
- **重启 daemon 前先清旧端口占用**：`openclaw daemon restart` 偶发起重进程失败（旧 gateway 仍占 `18789`，LaunchAgent 起重进程时旧进程还在服务旧代码）。稳妥写法：`kill $(lsof -iTCP:18789 -sTCP:LISTEN -t) 2>/dev/null; sleep 2; openclaw daemon restart; sleep 3`。
- **运行时目录是 `~/.openclaw/extensions/<id>/`，不是 openclaw 源码仓库的 `extensions/`**（后者的几百个目录是仓库自带源码，运行时不加载）。
- **插件运行时状态文件只放插件自己的扩展目录**（`~/.openclaw/extensions/<id>/` 下），**绝不往 openclaw 系统路径**（`~/.openclaw` 配置树、`agents/`、`sessions/`、daemon data 等）**里写**。例如换肤状态 `branding-state.json` 由 `path.dirname(fileURLToPath(import.meta.url))` 解析到扩展目录（若落在 `dist/` 再取父目录），并在 `.gitignore` 排除——纯运行时生成、随插件卸载一起消失、不污染 openclaw 本身。
- **必须同时部署根目录 `index.mjs` 和 `dist/index.mjs`**：openclaw 默认加载扩展根目录的 `index.mjs`；只放 `dist/` 会导致 `plugin not found`（热重载日志报 `stale config entry ignored`）。注意 `dist/index.mjs` 要拷到扩展目录的 `dist/` 子目录下，而不是和根 `index.mjs` 平级；拷错位置会让 openclaw 继续serve旧的 `dist/index.mjs`。
- **部署后立即双验**：`grep` 扩展目录的 `index.mjs` 与 `dist/index.mjs`，确认关键字符串已更新；再 curl  served HTML/JS 验证。
- **必须在 `openclaw.json` 的 `plugins.entries` 显式启用**：未启用的扩展即使文件就位也不会被加载（日志直接报 `plugin not found`）。
- Node ≥ 22.5（`node:sqlite` 要求，若插件用 SQLite）。
- stderr 被丢到 /dev/null（见 plist `StandardErrorPath`），插件加载期报错看不到，排错看 `~/.openclaw/...` 不行时改用 `~/Library/Logs/openclaw/gateway.log`（stdout）。

## 三、插件元数据

### openclaw.plugin.json
```json
{
  "name": "<extension-name>",
  "version": "1.0.0",
  "description": "...",
  "main": "dist/index.mjs",
  "adminPage": { "path": "/plugins/<name>", "label": "管理页面" }
}
```
- `adminPage` 在 openclaw 后台注册一个页面（可选；纯 API 插件可省略）。

### package.json（ClawHub 强制字段见第九章）

## 四、API 路由与 auth

```typescript
api.registerRoute({
  path: "/plugins/<name>/api/*",
  handler: (req, res) => { /* ... */ return true; },
  auth: "plugin"   // plugin：iframe 直连后台，免二次 token，最方便
});
```

页面路由（如有 UI）：
```typescript
if (pathname === "/plugins/<name>" || pathname === "/plugins/<name>/") {
  res.setHeader("content-type", "text/html; charset=utf-8");
  res.end(ADMIN_PAGE_HTML);
  return true;
}
```

- **必须 `return true`** 告知已处理该请求，否则请求继续流向后续中间件，导致页面/接口不生效或 404。
- `auth` 可选策略：`plugin`（免二次认证，推荐给后台 iframe 用）、按需求用其他策略。

### ⚠️ 跨域 CORS 坑（插件页在 sandbox iframe 里，origin 为 "null"）
插件页被 openclaw 用 `<iframe sandbox="allow-scripts">` 嵌入，**无 `allow-same-origin`**，所以页内 fetch 后台 API 是跨域（origin = `"null"`），必须由后端返回 CORS 头，否则浏览器直接拦。

- **前端 fetch 不要带 `content-type: application/json` 这类"非简单头"**——它会触发浏览器 OPTIONS 预检，而很多人只在正式响应里加 `Access-Control-Allow-Origin`、忘了预检响应，于是报 `Request header field content-type is not allowed by Access-Control-Allow-Headers in preflight response`。**最稳做法：fetch 不显式设 content-type 头**（字符串 body 时浏览器自动设为 `text/plain`，属简单请求，不再预检）。后端本来就 `JSON.parse(原始 body)`，不设该头完全不影响解析。
- **后端 handler 必须统一返回 CORS 头**（每个响应 + OPTIONS 预检）：
  ```typescript
  function setCors(res: any) {
    res.setHeader("Access-Control-Allow-Origin", "*");
    res.setHeader("Access-Control-Allow-Methods", "GET, POST, OPTIONS");
    res.setHeader("Access-Control-Allow-Headers", "*"); // 覆盖任意头，预检必过
  }
  // handler 开头：setCors(res);
  // if (req.method === "OPTIONS") { res.statusCode = 204; res.end(); return true; }
  ```
- 验证技巧：用 `curl -H "Origin: null" -X OPTIONS ...` 模拟预检，确认响应含 `Access-Control-Allow-Headers`；用 egobrowser 打开页面触发真实 fetch，确认不再报 CORS。

## 五、前端 UI 编写约定

### 内联 JS 的引号转义（最易踩坑）
ui.ts 常用 `js.push("...")` 行拼接浏览器端 JS：
```typescript
// 外层双引号 → 内层 HTML 属性双引号必须写成 \"
js.push("  box.innerHTML = '<div class=\"loading\">加载中...</div>';");
```
- 用 Python 批量改 ui.ts 时，文件里是 `\"`（反斜杠+双引号），Python 字符串需 `\\\"`。最稳做法：Python 原始三引号 `r'''...'''` 逐行重写。

### 服务端渲染（SSR）降级（强烈推荐，应对第七章平台坑 #1）
把数据直接渲染进 HTML，翻页/筛选/进详情用 `<a>` 链接，导出/同步用链接参数（`?dl=1` / `?sync=1`）触发服务端响应。脚本仅作增强（SPA 接管）。这样 strict 沙盒下首屏也完整可用。

### 复用 openclaw 原生 CSS（仅当插件 UI 涉及对话/聊天渲染时）
- 对话气泡：`chat-group` / `chat-avatar` / `chat-bubble`。
- 工具调用：`.chat-tool-msg-summary`（配 `<summary>`）+ `.chat-tool-msg-collapse`（默认折叠）。

### 主题与国际化限制（重要，避免白做）
- **插件当前无法跟随 openclaw 的明暗主题 / 语言切换**（平台限制，见第七章坑 #2）。
- 若确实需要暗色 / 多语言，只能读**操作系统**偏好（`prefers-color-scheme` / `navigator.language`），这**不等于** openclaw 应用内开关。

### 五·之二：对齐 openclaw 官方菜单的 UI 视觉规范（调色板与布局基线）
两个真实插件（session-label-from-sender 会话管理、openclaw-branding 换肤）反复打磨出来的「看起来就像 openclaw 自家菜单」的基线。**新插件直接复用这套数值**，别自己拍脑袋配色——用户一眼能看出「不像原生的」。

**为什么必须自己内联样式（关键约束）**：插件 iframe 沙盒（`scripts` 模式，无 `allow-same-origin`）**读不到 openclaw 的 stylesheet**（DOM/localStorage/cookie 全隔离），不能 `@import` 或 `<link>` openclaw 的 CSS。只能把下面这套数值**复制到插件自己的 `<style>` 里复刻**。权威值源文件（供查）：`ui/src/styles/layout.css`（`--accent`/`.content-header`/`.page-title`）、`components.css`（`.card`）、`config-quick.css`（`.settings-workspace`）。

**官方调色板（暖色浅色主题，实测取自官方 CSS 变量）**：
| 变量 | 值 | 用途 |
|------|-----|------|
| `--ink` | `#252421` | 主文字 |
| `--muted` | `#78746d` | 副文字/占位 |
| `--line` | `#e8e3da` | 卡片/输入框边框 |
| `--paper` | `#fffefa` | 卡片背景 |
| `--canvas` | `#f5f2ec` | 页面背景 |
| `--red` | `#d84a38` | 强调色（错误红边/聚焦）朱红 |
| `--teal` | `#168f89` | 次强调（成功/链接） |
| `--field-border` | `#ded8ce` | 输入框边框 |
- **标题色 ≠ 强调色**：`.page-title` 用砖红 `#bd4531`（偏深），按钮/错误用朱红 `--red #d84a38`。两者都是「红」但色相不同，别混用。

**布局基线（对照 `/skills` `/agents` 菜单实测）**：
- 头部 `.content-header`：padding `4px 8px`；标题**左上角、偏上**——**不要用居中容器**（`width:min(820px,...);margin:auto` 会让标题飘中间，不像官方）。
- 标题 `.page-title`：`font-size:22px; font-weight:650; letter-spacing:-.03em; line-height:1.2; color:#bd4531`。
- 副标题 `.page-sub`：`font-size:13px; font-weight:400; color:var(--muted)`。
- 卡片容器（插件侧 `.body`，对应官方 `.settings-workspace`）：`display:flex; flex-direction:column; gap:20px`。
- 卡片 `.card`：`background:var(--paper); border:1px solid var(--line); border-radius:14px; padding:18px`；**默认无 box-shadow，hover 才出**。
- 表单字段标签 `.fld>span`：`font-size:13px; font-weight:600; color:var(--ink)`。
- 输入框 `.control`：`height:42px; border:1px solid var(--field-border); border-radius:9px; background:#fff; padding:0 12px`；宽度 `max-width:340px`（单字段）或配 `path-row{flex:1}` 自适应。**两个并列输入框要等长就统一 max-width**。
- 主按钮 `.btn.primary`：背景 `#e87a66`（暖珊瑚红，比 `--red` 浅 ~25%，更柔和）hover `#d66853`；次按钮描边 `--teal`。**错误/聚焦红边保留更深 `--red`**（不要和按钮一样浅，否则弱化错误信号）。

**对齐实操要点（踩过的坑）**：
- 改巨型 CSS 单行字符串（`const CSS="..."` 一行）时，HTML 元素改名（如 `h1`→`div`）**必须同步改 CSS 选择器**（`.top h1`→`.top .page-title`），否则样式不生效。
- 多 `js.push("...")` 拼接浏览器 JS 时**所有行必须配对**：IIFE `})();` 关闭后，下一行必须重新显式写 `function setMode(m){`，否则函数声明整段丢失，运行时点击才报 `setMode is not defined`。build 后用 `node --check` 抓线上 served `<script>` 块语法。
- **操作后用户无感知是 iframe 插件通病**：不要只把结果打到隐藏较深的区块，要在操作按钮附近首屏可见处弹醒目反馈（成功绿/失败红），并明确告知下一步动作（如「按 Cmd+Shift+R 硬刷新」）。自动刷新父页面留作 best-effort（`window.top.location.reload()` 多数部署被 sandbox 静默拒绝）。
- 视觉验证降级：openclaw 的 `sw.js` 把插件页 HTML 也 Cache Storage 缓存，egobrowser 截图常被旧 SW 挡；改用 `curl` 抓线上 HTML + `node --check` 文本核验最可靠（用户 `Cmd+Shift+R` 硬刷一次看真实效果）。

### 五·之三：客户端抽屉要渲染 markdown / 用 npm 库——打包成 IIFE 内联（SSR 直接 import）

插件 UI 常见两类渲染路径，对 npm 库的接入方式完全不同：

1. **服务端渲染（SSR）/ 后端运行时**：`ui.ts` 在 Node 里跑，可直接 `import markdownit from "markdown-it"` 用 `md.render(t)`。最简单，无额外打包。
2. **客户端抽屉（浏览器端内联 JS）**：浏览器脚本在 `<iframe sandbox="allow-scripts">` 里跑，**不能 `import` npm 模块**（没有打包器、没有 module 解析）。要把 npm 库送进浏览器，只能**用 esbuild 单独打包成 IIFE，挂到 `globalThis`，再内联进页面 `<script>`**。

**双路径落地（以 markdown-it 为例，小包、XSS 安全）**：

- 建一个浏览器入口 `mdClientEntry.ts`：
  ```typescript
  import markdownit from "markdown-it";
  const md = markdownit({ html: false, linkify: true, breaks: true, typographer: false });
  (globalThis as unknown as { MD: typeof md }).MD = md;  // 挂到 window.MD
  ```
- `build.mjs` 加一步 IIFE 构建（注意 `nodePaths` 指向 managed workspace 的 `node_modules`，否则 esbuild 解析不了 `markdown-it`；见下方坑）：
  ```javascript
  await build({
    entryPoints: [path.join(__dirname, "mdClientEntry.ts")],
    bundle: true, format: "iife", platform: "browser", target: "es2020",
    nodePaths: [path.resolve(process.env.HOME, ".workbuddy/binaries/node/workspace/node_modules")],
    outfile: path.join(distDir, "md-client.js"),
  });
  ```
- SSR：列表/详情页直接 `mdServer.render(t)` 输出 HTML（同时 `<style>` 注入 markdown 样式）。
- 客户端：内联 `<script>` 把 `dist/md-client.js` 内容塞进 `<head>`，抽屉里用 `window.MD.render(t)`；并保留一个正则 fallback 防库未加载：
  ```javascript
  function md(t){
    if (!t) return '';
    try { if (window.MD && typeof window.MD.render === 'function') return window.MD.render(t); } catch (e) {}
    var s = esc(t); /* 回退：换行→<br>、URL→<a> */ return s;
  }
  ```
  - ⚠️ 内联脚本里的 `</script>` 必须转义成 `<\/script>`，否则提前闭合标签。
- **`.bubble` 样式切换**：从正则 `pre-wrap`（保留换行）切到 markdown-it 后，要把 `.bubble{white-space:pre-wrap}` 改成 `white-space:normal`——markdown-it 输出 block 元素（`<p>`/`<ul>`/`<li>`），pre-wrap 会额外多出空行。配套补一段 markdown 专属 CSS（`.bubble p/ul/ol/li/h1-4/blockquote/a/table/pre/code/img`）。
- **XSS 安全**：markdown-it 配 `{html:false}`，默认转义原始 HTML（`<script>`、`<img onerror>` 等被转义成文本），客户端/服务端都安全，无需手动 sanitize。
- **build.mjs 坑**：esbuild 默认只搜当前目录 `node_modules`，而 `markdown-it` 装在 **managed workspace**（`~/.workbuddy/binaries/node/workspace/node_modules`），插件目录里没有。必须把该路径加进 `nodePaths`（且**每个 build 调用都要带**，共用 config 对象时别漏）。装包：`cd ~/.workbuddy/binaries/node/workspace && npm install markdown-it`。

### 五·之四：无限滚动分页渲染——必须增量插入，禁止整列表 innerHTML 重渲染

聊天/列表页做"滚动到底/到顶自动加载下一页"时，**绝不能在每次分页返回后 `box.innerHTML = renderAll(currentList)` 整列表重渲染**。快速滚动会连续触发多次加载，每次都销毁并重建全部 DOM 节点 + 重设 scrollTop → 视觉**闪屏/抖动**；往回滚一下不触发加载就"正常"了，正是这个特征。

**正确做法：增量 DOM 插入，只插入新批次的节点，保留已有节点。**
- 客户端抽屉（浏览器端）用 `element.insertAdjacentHTML('beforeend'|'afterend'|'beforebegin', batchHtml)` 把新批次 HTML 拼进对应位置，**不碰已有节点**。
- 维护日期分割线等需要跨批次连续性的状态：用 `topDate` / `bottomDate` 记录已渲染首尾消息的日期，新批次从对应日期续接（`buildBatchHtml(msgs, startLastDate)` 返回 `{html, lastDate}`）。
- **顶部续接技巧（向上加载更早、prepend 到顶部）**：把新批次插到列表里**第一个 `.date-divider` 之前**（`firstDiv.insertAdjacentHTML('beforebegin', html)`）。这样无论新批次与现有顶部是否同一天，顶部那条分割线始终留在最顶，既不会出现跨页同日的"中间分割线"，也不会重复。若没有分割线则插到 `#msgTop` 之后。
- 插入后用滚动高度差锚定视口，避免跳动：`prevH = box.scrollHeight; prevTop = box.scrollTop; /* 插入 */ box.scrollTop = prevTop + (box.scrollHeight - prevH);`（顶部 prepend 时尤其要，否则视口会跟着内容被顶下去）。
- 底部 append 时若用户已在底部（`scrollTop + clientHeight >= scrollHeight - 2`）才 `scrollTop = scrollHeight` 钉到底；否则保持 `scrollTop` 不动，新内容在折叠线以下、无跳动。
- `msgLoading` 守卫仍要有（加载中忽略滚动事件），保证分页请求串行、不重叠。
- 首屏（offset 0）仍是整列表 `innerHTML` 一次（清空 loading 态），没问题——闪屏只来自后续每页的重复全量重渲染。

## 六、数据层：node:sqlite (DatabaseSync) 的坑

Node ≥ 22.5 的 `DatabaseSync` 为实验 API，实测关键坑如下。

### UPDATE 参数绑定 bug
`prepare() + run()` 的命名参数对 UPDATE 有时**不写入**（实测踩过）。改用 `db.exec()` 内联 SQL，并手动转义字符串单引号防注入：
```typescript
db.exec(`UPDATE t SET c = ${val} WHERE k = '${key.replace(/'/g, "''")}'`);
```

### SELECT 必须用 prepare
`db.exec("SELECT ...")` 返回 `undefined`。SELECT 用 `db.prepare() + stmt.all()/get()`。

### 混合策略
| 操作 | 方法 | 原因 |
|------|------|------|
| SELECT | `prepare() + all()/get()` | 参数绑定正常，返回格式正确 |
| INSERT | `prepare() + run()` 或 `INSERT OR IGNORE` | 可正常工作 |
| UPDATE | `db.exec()` 内联 SQL（手动转义） | prepare 参数绑定有 bug |

### 去重 / 幂等
`PRIMARY KEY (col1, col2)` + `INSERT OR IGNORE` 保证增量同步重复写入被忽略。
- 注意 `DatabaseSync` **没有** `.transaction()` 方法，批量写用循环 `run()` 即可。

## 七、访问 openclaw 原生数据（插件常需读会话/转录）

### 转录文件两种格式
- 标准 `.jsonl`：每行 `{type:"message", message:{role, content}, timestamp}`，消息在 `message` 信封；`role` ∈ `user/assistant/tool/toolResult`。
- 轨迹 `.trajectory.jsonl`：事件流，对话在 `model.completed.data.messagesSnapshot[]`；注意过滤 `system` 等非消息角色。

### ⚠️ 坑 A：multi-agent 同步必须传 agentId
openclaw 每 agent 的会话独立存于 `<stateDir>/agents/<agentId>/sessions/`（各自 `sessions.json` + `*.jsonl`）。`listSessionEntries()` 不传 `agentId` 时默认只返回 `main`，**子 agent（work/code 等）会话永远拉不到**。

```typescript
import fs from "node:fs";
import path from "node:path";
function listAgentDirs(stateDir: string): string[] {
  const root = path.join(stateDir, "agents");
  if (!fs.existsSync(root)) return ["main"];
  return fs.readdirSync(root, { withFileTypes: true })
    .filter(e => e.isDirectory() && fs.existsSync(path.join(root, e.name, "sessions")))
    .map(e => e.name);
}
for (const agentId of listAgentDirs(stateDir)) {
  const entries = api.runtime.agent.session.listSessionEntries({ agentId });
  // ... 逐 agent 处理
}
```
- sessionKey 形如 `agent:<id>:<rest>`；按 agent 维度筛选直接从自己 SQLite `GROUP BY agent_id` 取，**别依赖 `openclaw.json`**（配置会变，历史数据会失联）。

### ⚠️ 坑 B：trajectory 去重 —— seq 不能当消息 ID
`.trajectory.jsonl` 中**同会话多个 `model.completed` 事件，seq 不唯一，且 `messagesSnapshot[]` 是累计超集**（后事件含前事件全部）。若消息 ID 用 `${sessionId}-e${seq}-m${i}`，相同消息会因 seq 前缀不同而重复插入（实测翻倍）。

**正确做法**：用 snapshot **下标**作 ID（累计超集下标天然对齐，去重到最终唯一集合）：
```typescript
for (let i = 0; i < snapshot.length; i++) {
  const role = snapshot[i].role || "unknown";
  if (!["user", "assistant", "tool", "toolResult"].includes(role)) continue;
  const id = `${sessionId}-m${i}`;   // 下标，不依赖 seq
  insertMessage(db, { id, /* ... */ });  // INSERT OR IGNORE 去重
}
```
- 验证计数时过滤真实消息角色，别直接数 `snapshot.length`（含 system 会偏高）。

### 增量同步
按文件**字节偏移**游标增量读取，避免每次全量扫描：
```typescript
const stream = fs.createReadStream(path, { start: cursor });
// readline 逐行解析，完成时 updateCursor(byteOffset)
```
- ⚠️ 若上游旋转了文件名（如 `/new` 新建会话文件），游标需随新文件**重置为 0**，否则 `startOffset >= fileSize` 会被判空跳过。

### ⚠️ 近实时同步最佳实践（强烈推荐）
若插件要"展示 openclaw 实时数据"，**不要只依赖定时同步**——用户会感觉延迟。正确做法：在**读数据的 API/页面 handler 返回前，先跑一次轻量 registry 同步**（只发现新条目、更新元数据，不全量扫消息），再返回。这样用户一打开页面/刷新列表，新数据立刻出现，体感"瞬间同步"；而定时的全量消息同步只作后台兜底。

```typescript
// 列表/查询接口返回前先同步 registry
if (pathname === "/plugins/<name>/api/items" && req.method === "GET") {
  syncRegistry(api);              // 轻量：发现新条目 + 更新元数据
  const items = listItems(getDb(), {...});
  res.end(JSON.stringify(items));
  return true;
}
```
- 区分两层：**registry 同步**（轻量，近实时，让新行立刻出现）vs **全量内容同步**（重，定时/手动触发，拉消息体）。两者频率不同，别混为一谈。

## 八、定时任务

```typescript
const timer = setInterval(doWork, intervalMinutes * 60 * 1000);
```

> openclaw 的 extension 型插件**没有 `onDisable`/`onUnload` 生命周期钩子**（channel 型插件才有 `lifecycle.onAccountRemoved` 等）。删除插件或 disable 后 `setInterval` **不会自动停**——定时器只在 daemon 进程退出时自然释放。
>
> **安全的做法**：在 `setInterval` 回调里自己判断是否需要停止（例如检查文件是否存在、配置是否变化），或明确告知用户「修改插件配置后需 `openclaw daemon restart`」。不要在模块顶层写无法自行退出的无限循环定时器。

## 九、发布到 ClawHub

### ⚠️ 前置：版本号与文件可能不一致
- ClawHub 按 `package.json` 的 `version` 建版本；但若之前用 `--version` 旗标发版，文件里的 `version` 可能落后线上。
- **发版前必做**：`clawhub package inspect @<owner>/<name>` 查线上 Latest，再定一个**更高**的版本号，否则被拒或重复。

### package.json 强制字段
```json
{
  "name": "@<owner>/<plugin-name>",
  "version": "2026.8.9",
  "type": "module",
  "repository": { "type": "git", "url": "https://github.com/<owner>/<repo>" },
  "openclaw": {
    "extensions": ["./dist/index.mjs"],
    "build": { "openclawVersion": "2026.7.1" },   // 具体版本，不要加 >=
    "compat": { "pluginApi": ">=2026.7.1" },       // 这里可用范围
    "install": {
      "npmSpec": "@<owner>/<plugin-name>",
      "clawhubSpec": "@<owner>/<plugin-name>",     // 必填，ClawHub 识别安装目标
      "minHostVersion": ">=2026.7.1"
    }
  }
}
```
必填：`openclaw.build.openclawVersion`、`openclaw.compat.pluginApi`、`install.clawhubSpec`。

### 发布流程
1. **清理硬编码本地绝对路径**：`/Users/xxx` → `$HOME` 或通用示例（`$REPO`/`/path/to`）。**不只 ClawHub validate 扫源码，public GitHub 仓库也会暴露你的本机用户名路径**；README「安装」里的本地路径示例同样参数化，否则既泄露隐私又对 clone 用户不可用。token/邮箱一并清理。ClawHub 扫工作目录**不读 .gitignore**，发布前 `rm -rf reports`（validate 产物）。
2. 改 `package.json` 版本号 + 必填字段。
3. 构建：`node build.mjs`。
4. 本地部署 + **daemon 重启 + 实际验证页面/接口**（发布前必须本地验证，确认无误）。
5. 验证：`clawhub package validate .` → 必须 **0 错 0 警**。
6. **写 changelog**：维护 `CHANGELOG.md`（作为单一事实源），并提炼一段适合版本页的 release note。
7. 发布：`clawhub package publish . --family bundle-plugin --owner <handle> --changelog "<release-note>" --source-repo <repo> --source-commit $(git rev-parse HEAD)`（先加 `--dry-run` 确认，再正式）。**`--changelog` 不是可选项，必须带；ClawHub 不允许修改已发布版本的 release note，漏填只能再升一个版本重发**。
8. 提交 GitHub：注意 remote 名（`github` / `origin` 可能指向不同平台，别推错）；推送后 `clawhub package inspect` 确认 Latest = 新版本、Scan clean、moderation 通过（live）。
   - 刚发布那几分钟 `Scan: suspicious` 是正常过渡态，约 10~15 分钟后会自动转 `clean`，和上一版行为一致；不必惊慌，但要复查确认。
- 从零建公开仓库一步到位：`gh repo create <name> --public --source=. --remote=origin --push`（要求 `gh` 已登录且带 `repo` scope）；仓库只含源码，运行时 bundle 不入库（见第二章）。

### 常见 validate 警告
| 警告 | 修复 |
| `package-install-metadata-incomplete` | 补 `install.clawhubSpec` |
| `package-min-host-version-drift` | `build.openclawVersion` 不要加 `>=` 前缀 |

### ⚠️ 发布 Git 避坑：rebase 的 ours/theirs 陷阱
推送到远端被拒后 rebase，冲突解决里 **`--ours` 指目标基底（远端），`--theirs` 才是你的改动**。曾误用 `--ours` 把自己的提交判成空提交丢掉，靠 `git reflog` 找回重做。牢记：**要保留自己的改动用 `--theirs`**。

### 镜像多远端
若 GitHub（`github`）与 coding.net（`origin`）等都要同步，注意各自默认分支名可能不同（如 `main` vs `master`）。先 `git fetch` 查共同祖先确认是 fast-forward，再 `git push <remote> <localBranch>:<remoteBranch>`，避免误用 `--force` 覆盖。

## 十、平台级坑（插件侧改不了，需知晓）

### 坑 #1：iframe sandbox 竞态（所有插件都会遇到）
- **现象**：硬刷新插件页报 `Blocked script execution ... allow-scripts is not set`；切走再切回正常。
- **根因**：openclaw WebUI 用 `<iframe sandbox>` 嵌插件页。sandbox 来自 `embedSandboxMode`：`strict→""`(禁脚本)、`scripts→allow-scripts`、`trusted→allow-same-origin`。WebUI 初始默认 `strict`，真正配置（gateway 默认 `scripts`）**异步**到达。首屏用 strict 建 iframe，配置到达后**已加载文档不追溯生效**（HTML 规范）→ 脚本被禁。切回 = 重建 iframe，配置已就位 → 正常。
- **根因在 openclaw 主程序**（`ui/src/pages/plugin/plugin-page.ts` 用 `subscribe:false` 不订阅 config 更新），插件侧改不了。
- **根治（推荐，见第五章 SSR）**：服务端渲染数据进 HTML，导航/筛选/导出/同步全用 `<a>` 链接或链接参数，脚本仅增强。strict 沙盒只禁脚本，静态 HTML + 链接导航可用，首屏硬刷新也完整可用。
- **不要做**：`<meta http-equiv="refresh">` 自愈——strict 沙盒连 meta 刷新导航也拦，死路。
- **绕过（不改代码）**：从菜单进入 / 切走再切回即可，功能正常只是首屏需一下。
- **铁律：绝不修改 openclaw 主程序 / 源码**。sandbox 是 openclaw 刻意的安全设计（防插件逃逸、隔离宿主），不是 bug；即使有 UX 瑕疵，也由 openclaw 官方修复，插件侧不碰。改 `embedSandboxMode` 默认值、改 plugin-page 逻辑、重新打包 control-ui —— 都属于 fork 官方源码、越界且背离"用插件扩展而非改宿主"的初衷，绝不做。本插件（branding）对 control-ui 文件做 logo / 品牌字替换，是**插件既定目的（运行时换肤）**，与"修改宿主源码 / 安全逻辑"是两回事，不可混淆。
- **无害残留**：strict 首屏 console 仍打一条 Blocked script 警告，页面功能不受影响。
- **插件侧可做的缓解（推荐，不改宿主）**：在插件页 HTML 顶部加 `<noscript>` 提示块。机制——strict 沙箱（无 `allow-scripts`）下浏览器把该文档视为"脚本禁用"，`<noscript>` 内容**正好此时渲染**；正常沙箱下脚本跑起来，提示自动消失。于是冷刷新那个"静默坏页"窗口被变成一条清晰引导：「若按钮无响应，点左侧其他菜单再回到本页即可」。这等价于用户产品思路①（引导走菜单进入）的插件侧实现，不碰宿主、不破坏任何东西。注意：console 那条浏览器安全报错本身无法从插件消除（浏览器行为），只能靠 openclaw 官方修复；`<noscript>` 解决的是"坏页无提示"这一 UX，不是消除 console 噪声。
- **⚠️ noscript 方案的致命实现坑（已踩过，必读）**：配套要用 `.app{display:none}` + 末尾 reveal 脚本 `getElementById('app').style.display='flex'`。**两个必错点**：
  1. **被显示的容器必须真的有 `id="app"`**。SSR 里 `<main class="app">` 只写 `class` 没写 `id` → `getElementById('app')` 返回 `null` → 脚本抛 `Uncaught TypeError: Cannot read properties of null (reading 'style')` → reveal 没跑 → `.app` 永远 `display:none` → **整页空白**。务必写 `<main class="app" id="app">`。
  2. **reveal 脚本一律防御式写**：`var _app=document.getElementById('app'); if(_app){_app.style.display='flex';}`。绝不要裸 `getElementById('app').style...`，否则任何拿不到元素的情况都整页崩。
  - 做法总结：SSR 两页（列表/详情）都在 `<main>` 加 `id="app"` 并在 `</body>` 前放防御式 inline `<script>` reveal；客户端 bundle 末尾也放同款防御式 reveal。沙箱禁脚本时 `.app` 保持隐藏 + `<noscript>` 提示呈现，脚本可用时正常显示。
  - **📦 直接复用现成代码**：`references/sandbox-noscript-guide.md` 提供了**可直接粘贴**的 CSS + noscript HTML（含「温馨提示」标题与**纯 CSS 中英文切换**，沙盒禁脚本时也能切语言，无需 JS）+ 防御式 reveal 脚本，以及 6 条致命坑清单。开发任何 openclaw 插件页时，把这套代码和逻辑贴进去即可，不必重新推导。

### 坑 #2：主题与国际化无法跟随 openclaw
- 插件 iframe 由 openclaw 嵌入，其 `src` 仅带插件路径，**不传 theme/lang 参数**，也无 postMessage；沙盒 `scripts` 模式无 `allow-same-origin`，插件读不到 openclaw 的 DOM/localStorage/cookie。
- 结论：插件**不能**跟随 openclaw 的明暗 / 语言。要跟必须改 openclaw 主程序把主题 / 语言传入 iframe（参数或 postMessage）。插件侧只能跟 OS 偏好（`prefers-color-scheme` / `navigator.language`），不等于 openclaw 开关。

### 坑 #3：sandbox 禁用模态对话框（alert/confirm/prompt 全部失效）
- **现象**：页内调用 `alert()` 不弹窗，console 报 `Ignored call to 'alert()'. The document is sandboxed, and the 'allow-modals' keyword is not set.`（`confirm`/`prompt` 同理）。
- **根因**：插件 iframe 是 `sandbox="allow-scripts"`（无 `allow-modals`），浏览器一律忽略模态对话框调用。这是平台安全设计，插件侧改不了。
- **替代方案（插件侧必须做）**：所有用户提示改用页面内非模态元素——错误用已有的 `.error-banner`（如 `#errorBox`，设置 `textContent` + `style.display='block'`），成功/提示用按钮附近的首屏可见 inline 文字。绝不在校验、确认、输入等处依赖 `alert/confirm/prompt`。
- 排查技巧：`grep` 源码确认全程无 `alert(/confirm(/prompt(` 残留。

## 十一、浏览器缓存与 Service Worker 调试（实战：branding 头像闪烁）

> **经验法则（用户强调，最高优先）**：遇到实在查不动的浏览器侧问题（缓存、Service Worker、渲染、鉴权态、跨域等），**不要只在 DevTools 看 Network 干猜，直接用 ego-browser 真实浏览器实测**——它能构造缓存/复现症状/验证清除，是这类顽固问题的决定性武器。本次头像闪烁 bug 前两轮都因「没用真实浏览器验证 SW 缓存是否被清」而误判已修好，第三轮靠 ego-browser 才彻底坐实修复。
>
> openclaw-branding「换头像后切会话正常、刷新闪回旧头像」bug 的完整复盘就是范例。**核心方法论可迁移到任意 web 缓存 / SW 问题**：用真实浏览器构造旧缓存 → 复现 → 注入修复 → 验证清除，形成闭环证据。

### 三条彼此独立的缓存通道（必须分别处理）

「URL 没变就命中旧字节」可能来自任意一层，任何一层没管住都会复发：

1. **HTTP 缓存（Cache-Control / fetch cache 选项）**：常规 `cache:'no-cache'|'reload'` 能管，但**不覆盖下面两类**。
2. **Service Worker fetch cache（Cache Storage）**：SW `fetch` handler 写入的 `caches.open(...).put(...)`。**SW 接管前写入的旧缓存，新 SW 不删就永久沉积**。
3. **浏览器 favicon 独立缓存**：`<link rel="icon">` / `<link rel="apple-touch-icon">` 走**浏览器内置 favicon 缓存，不遵守 Cache-Control 与 fetch cache 选项**——URL 不变就显示历史字节。**唯一根治手段是让 URL 变化（`?v=stamp`）**。

### 层进式排查（4 个 root cause，逐个打穿）

症状：A→B→C 换头像（无还原）→ 切会话显示 C → **普通刷新闪回 B**。

| 轮次 | 当时假设 | 实际根因（本轮才看清） | 修复 |
|------|---------|----------------------|------|
| 1 | branding 4 图标缓存 | 漏了 chat 头像走**独立通道** `/avatar/{agentId}`，与 branding 4 图标无关 | SW 把 `/avatar/` 加 no-store 白名单 |
| 2 | favicon + `/avatar/` 都加 no-store | SW `activate` 只删 `openclaw-control-*` 前缀，openclaw 原始 SW 的 `CACHE_PREFIX` 不一定以该前缀开头 → 旧缓存清不掉 | `activate` 改为删【所有】非当前缓存 |
| 3 | 删了所有旧缓存 | 头像 fetch 失败 `.catch(()=>caches.match())` 回退 cacheStorage 吐旧字节 | 失败直接 `502`，**绝不读旧缓存** |

**关键教训**：

- **chat 头像 ≠ branding 图标，是两条独立字节通道**。chat 头像由 SPA bundle（如 `chat-page-*.js` 的 `Ai(e)`）先 `fetch('/avatar/{id}?meta=1')` 拿 `avatarUrl`，再 `fetch('/avatar/main')` 带鉴权 token → `response.blob()` → `URL.createObjectURL` → `<img src>`。它**完全不经过** branding 的 4 图标文件，也不走 `<link rel>`。**只修 branding 图标永远覆盖不到 chat 头像**。
- **SW `CACHE_PREFIX` 陷阱**：别假设历史缓存都以你插件前缀开头。openclaw 原始 SW 的 `CACHE_PREFIX` 可能用别的命名；只删自己前缀 = 清不掉沉积旧字节。正确做法：`cacheKeys.filter(k => k !== CACHE_NAME)` 删全部（除当前缓存外的所有缓存）。
- **SW fetch 失败绝不能 fallback 到 `caches.match`**：一旦失败回退旧缓存，就等于「刷新时偶尔吐旧头像」。favicon/avatar 这类**来源唯一、不应有旧值**的资源，fetch 失败就直接 `502`（让页面显示破图/占位），**绝不读旧缓存**。

### stampSwCache 关键改动（index.ts）

```typescript
// activate：删掉除当前缓存外的【所有】缓存（覆盖任何原始前缀）
const controlKeys = cacheKeys.filter((key) => key !== CACHE_NAME);

// 头像/favicon 分支：识别 4 图标 或 url.pathname.startsWith("/avatar/")
if (__brandIsFavicon) {
  event.respondWith(
    fetch(event.request, { cache: "no-store" })
      .then((r) => {
        if (!r || !r.ok) return new Response("...", { status: r ? r.status : 502 });
        const h = new Headers(r.headers);
        h.set("Cache-Control", "no-store, max-age=0, must-revalidate");
        h.set("Pragma", "no-cache");
        return new Response(r.body, { status: r.status, statusText: r.statusText, headers: h });
      })
      .catch(() => new Response("avatar/favicon fetch failed", { status: 502 })), // 绝不 caches.match
  );
  return;
}
```

同时给 `index.html` 的 `<link rel>` href 注入 `?v=brandStamp`、给 SPA bundle 的 URL 拼装函数（如 `st()`）注入 `?v=window.__brandStamp`，让 favicon URL 变化失效浏览器内置缓存。

### ego-browser 真实浏览器实测闭环（决定性证据）

光改代码不算完，**必须用真实浏览器验证 SW 真的接管并清空了旧缓存**。用 `/egobrowser`（真实 Chromium）走闭环：

1. **复现**：页面内 `js()` 执行 `await caches.open('openclaw-control-stale-B').put('/avatar/main', new Response(fakePng))` 构造旧缓存 → `await caches.match('/avatar/main')` 返回命中（= 复现「刷新回 B」症状）。
2. **注入修复**：改磁盘 `sw.js`（升 stamp）触发新 SW 接管（`skipWaiting` + `clients.claim` 已有）。
3. **验证清除**：`cdp('Page.reload')` 重载 → 再查 `await caches.match('/avatar/main')` 返回 `false`、`caches.keys()` 已无 `stale-B`、SW 不再把 `/avatar` 写进缓存、`img.chat-avatar` 的 `naturalWidth > 0`（非 502）。
4. **确认 SW 内容**：`curl` 抓线上 `sw.js` 确认含 `key !== CACHE_NAME` / `avatar/favicon fetch failed`。

> **ego-browser 原生 API 提示**：`waitForTimeout` / `evaluate` 未导出，改用 `js()`（页内执行）/ `cdp()`（如 `Page.reload`）/ `sleep`。裸 `fetch('/avatar/main')` 会 401（需 SPA 鉴权 token，存于 `localStorage["openclaw.device.auth.v1"]`），所以验证头像要用**页面内带 token 的探针**，或直接在 DOM 上查 `img.naturalWidth`。

### 给用户的一次性硬复位指引（关键）

新 SW 必须**真正接管**才能清掉旧缓存。若单纯 `Cmd+Shift+R` 无效（旧 SW 仍控制页面）：

> 关掉所有 openclaw 标签页 → F12 → Application → Storage → 点「Clear site data」（或 Service Workers → Unregister）→ 重开页面。之后头像永远是新值。

## 十二、调试技巧
1. **构建验证**：`grep` `dist/index.mjs` 确认新函数/字符串已落地（注意 esbuild 可能把中文转成 `\uXXXX`，用 Python 解码核对）。
2. **API 测试**：`curl http://127.0.0.1:18789/plugins/<name>/api/<endpoint>`。
3. **DB 检查**：Python `sqlite3` 直接读 `.db` 文件。
4. **前端**：浏览器 DevTools Console（`Cmd+Shift+R` 强刷绕过缓存）。
5. **进程**：`openclaw daemon restart` 重启 gateway 加载新 bundle。
