# openclaw-plugin-dev

OpenClaw 插件（extension）从搭建、构建、部署、调试到发布的**一站式开发指南** skill。

内容覆盖：

- **项目结构与构建部署**：`index.ts` / `ui.ts` / `build.mjs` / `openclaw.plugin.json` 标准布局；运行时只加载 `~/.openclaw/extensions/<id>/`，必须用 esbuild 打包（运行时不 `npm install`），`node build.mjs` 后清空旧端口占用 + `openclaw daemon restart` 生效
- **API 路由与 auth/CORS**：插件页在 sandbox iframe（origin `null`），后端统一返回 `Access-Control-Allow-Origin/Methods/Headers:*`，前端 fetch 不显式设 `content-type` 免 OPTIONS 预检
- **前端 UI**：SSR 降级（strict 沙盒首屏可用）、内联 JS 引号转义、对齐 openclaw 官方视觉基线（暖色浅色调色板）、客户端用 esbuild 打包 IIFE 引入 npm 库、无限滚动增量插入
- **node:sqlite 坑**：`DatabaseSync` 无 `.transaction()`、UPDATE 命名参数绑定失效改 `db.exec()` 内联、SELECT 必须 `prepare`
- **访问 openclaw 原生数据**：JSONL / trajectory 两种格式、multi-agent 须传 agentId、trajectory 用 snapshot 下标去重、近实时 registry 同步 + 后台全量兜底
- **定时任务**：extension 型无 `onDisable` 钩子，定时器删除/停用不自动停，回调自查退出或告知用户 `openclaw daemon restart`
- **平台级坑**：iframe sandbox 竞态（SSR 根治、不改宿主）、主题/国际化不跟随、模态框失效改页面内 non-modal 提示
- **浏览器缓存与 SW**：三层独立缓存（HTTP / SW / favicon）、`activate` 删全部旧缓存、fetch 失败直接 502 不回退旧值；顽固问题用 ego-browser 真实浏览器实测闭环
- **ClawHub 发布**：必填字段、`clawhub package validate/publish` 流程、清理硬编码本地路径与敏感数据

> 本仓库即 WorkBuddy 加载的 skill 本体——`SKILL.md` 是核心内容；`references/sandbox-noscript-guide.md` 提供可直接粘贴的 sandbox `<noscript>` 提示实现；`scripts/` 为预留目录。

基于多个真实 openclaw 插件的踩坑沉淀，通用不绑具体业务。
