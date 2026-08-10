# openclaw-plugin-dev

OpenClaw 插件（extension）从搭建、构建、部署、调试到发布的**一站式开发指南** skill。

内容覆盖：

- **项目结构**：`index.ts` / `ui.ts` / `build.mjs` / `openclaw.plugin.json` 的标准布局与加载机制（根 `index.mjs` 与 `dist/index.mjs` 双份的坑）
- **构建部署铁律**：必须用打包工具（运行时 `npm install`）、源码仓库不含构建产物、`node build.mjs` 后用 `kill` 旧端口 + `openclaw daemon restart` 生效
- **API 路由与 auth/CORS**：插件页在 sandbox iframe（origin `null`），后端 `setCors` 返回 `Allow-Origin/Methods/Headers:*`
- **前端 SSR 内联**：巨型 CSS/JS 单行字符串的引号转义、`js.push` 拼接必须配对函数声明（否则运行时 `setMode is not defined`）
- **UI 视觉规范**：对齐 openclaw 官方菜单的颜色/大小/位置基线（砖红 `#bd4531` 标题、珊瑚红 `#e87a66` 按钮、`340px` 输入框、`14px` 圆角卡片等）
- **node:sqlite 坑**：`DatabaseSync` 无 `.transaction()`、批量写循环 `run()`、`UNIQUE` 去重 + `renumberSeq`
- **访问 openclaw 原生数据**：JSONL 会话、多 agent、trajectory 去重、`/new` 后按 `session_key` 归并
- **平台级坑**：iframe sandbox 竞态、主题/国际化限制（插件无法跟随 openclaw 明暗主题）
- **发布**：ClawHub 流程（`clawhub package validate/publish`、rebase 用 `--theirs`）、清理硬编码本地路径与敏感数据、`gh repo create` 一步建公开仓库

> 本仓库就是 WorkBuddy 加载的 skill 本体——`SKILL.md` 是唯一实质内容，`references/`、`scripts/` 为预留目录。

基于两个真实插件（`openclaw-branding` 换肤、`session-label-from-sender` 会话归档）的踩坑沉淀。
