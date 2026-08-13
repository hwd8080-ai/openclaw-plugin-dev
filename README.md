# openclaw-plugin-dev

OpenClaw 插件（extension）**开发生产线 + 核心难点**指南 skill。

**生产线**（`SKILL.md` 第一章）：明确目的 → 文件组成 → 配置（manifest/package.json/openclaw.json）→ 开发入口 → 构建产物 → 本地验证 → 先发 GitHub（要求）→ 发 ClawHub → 发完验证。

**核心难点概要**（第二章）：CORS/sandbox iframe、sandbox 竞态、主题/i18n 不跟随、模态框失效、node:sqlite 坑、原生数据去重、定时任务不自动停、SW 缓存三层、视觉对齐。

版本号用 `YYYY.M.D-rc.N`（对齐官方 `<base>-<tag>.<n>`，支持同天多次发布）。

> `SKILL.md` 是核心内容；`references/sandbox-noscript-guide.md` 提供可直接粘贴的 sandbox `<noscript>` 实现。官方文档：https://docs.openclaw.ai/plugins/building-plugins

基于多个真实 openclaw 插件的踩坑沉淀，通用不绑业务。

## 如何使用本 Skill（路由与查阅）

模型拿到本 skill 后，`SKILL.md` 始终全量加载（含实战流水线与参考索引表），`references/` 下 6 个文件按需读取——只加载命中主题的那一个，其余不进上下文，省 token 且避免污染。

不同问题具体落在哪：

| 用户问 | 路由到 |
|---|---|
| 我要做工具插件 / 下一步怎么走 | `SKILL.md` 流水线，必要时读 `references/plugin-forms-and-manifest.md` |
| manifest 字段怎么填 / 四种形态区别 | `references/plugin-forms-and-manifest.md` |
| 怎么加 hook / `before_tool_call` 怎么拦截 | `references/hooks-and-permissions.md` |
| 插件怎么装上 / 依赖报缺 | `references/install-and-dependencies.md` |
| 插件加载失败 / 架构怎么回事 | `references/architecture-and-loading.md` |
| 怎么发到 ClawHub / 构建报错 | `references/build-and-publish.md` + `SKILL.md` 流水线后半段 |
| sandbox 里脚本跑不了 | `references/sandbox-noscript-guide.md` |

每个 reference 段落都标注了官方文档出处链接（`docs.openclaw.ai/zh-CN/plugins/...`）。
