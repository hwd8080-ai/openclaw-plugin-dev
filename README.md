# openclaw-plugin-dev

OpenClaw 插件（extension）**开发生产线 + 核心难点**指南 skill。

**生产线**（`SKILL.md` 第一章）：明确目的 → 文件组成 → 配置（manifest/package.json/openclaw.json）→ 开发入口 → 构建产物 → 本地验证 → 先发 GitHub（要求）→ 发 ClawHub → 发完验证。

**核心难点概要**（第二章）：CORS/sandbox iframe、sandbox 竞态、主题/i18n 不跟随、模态框失效、node:sqlite 坑、原生数据去重、定时任务不自动停、SW 缓存三层、视觉对齐。

版本号用 `YYYY.M.D-rc.N`（对齐官方 `<base>-<tag>.<n>`，支持同天多次发布）。

> `SKILL.md` 是核心内容；`references/sandbox-noscript-guide.md` 提供可直接粘贴的 sandbox `<noscript>` 实现。官方文档：https://docs.openclaw.ai/plugins/building-plugins

基于多个真实 openclaw 插件的踩坑沉淀，通用不绑业务。
