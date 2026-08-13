# 构建与发布参考

> 本文件汇总插件的构建方式、`clawhub` 发布命令、包验收（`npm-pack`）、内置插件清单与 bundle 形式。本 skill 的**实测发布流水线**（含版本号 semver 规则、发前必构建、本地真 daemon 验证）见 `SKILL.md` 第 5–9 节，本文件补充官方文档层面的命令与概念。

---

## 1. 构建（Build）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/building-plugins

- 用 esbuild（或等价 bundler）把 `index.ts` + 依赖打成**自包含** `dist/index.mjs`。运行时加载的是构建产物，不现场 `npm install`。
- `package.json` 的 `openclaw.extensions` 指向 `./dist/index.mjs`（不能指 `.ts` 源码）。
- 校验：`openclaw plugins build` / `openclaw plugins validate`（构建与清单校验）。
- Node 版本：官方要求 ≥ 22.22.3（或 24.15+/25.9+）；用 `node:sqlite` 需 ≥ 22.5。

> 实战细节（构建脚本、NODE_PATH、中文 `\uXXXX` 核对）：`SKILL.md` 第 5 节。

---

## 2. 发布到 ClawHub

> 来源：https://docs.openclaw.ai/zh-CN/clawhub（发布）与 https://docs.openclaw.ai/zh-CN/plugins/building-plugins

```bash
clawhub package inspect @<owner>/<name>   # 查线上 Latest，新版本必须 > 它
clawhub package validate .                # 必须 0 错 0 警
clawhub package publish . --family bundle-plugin --owner <handle> \
  --changelog "<note>" --source-repo <repo-url> --source-commit $(git rev-parse HEAD) --dry-run
# dry-run OK 后去掉 --dry-run 正式发
```

- 必填字段：`openclaw.build.openclawVersion`、`openclaw.compat.pluginApi`、`install.clawhubSpec`。
- `--changelog` 必带（已发布版本的 note 不可改，漏填只能升版本重发）。
- 发布前**必须先发 GitHub** 作为 `--source-repo`（见 `SKILL.md` 第 7 节）。

### 版本号规则（ClawHub 走严格 semver）

> 来源：实测（`SKILL.md` 第 8 节）；官方兼容性见 https://docs.openclaw.ai/zh-CN/plugins/architecture

- 预发布恒低于稳定版：`...-beta.1` < `...-beta.2` < `...-rc.1` < `...`(裸稳定)。
- **想用 `-beta`/`-rc` 风格，必须先把预发布发出来，再毕业到裸稳定**；一旦发了裸 `2026.8.14`，同核心的 `-rc.1` 都比它小 → 被拒。
- 当天首次 `2026.8.15-rc.1`；同天再发 `-rc.2`；次日 `2026.8.16-rc.1`（patch 更大，天然 > 前一天）。
- 新版本必须 > 线上 Latest，否则被拒（先 `inspect` 看 Latest）。
- 注意：GitHub tag 的 `v2026.7.1-1`/`v2026.7.1-2` 是 GitHub 自己的排序习惯，**不等于** ClawHub 优先级。

---

## 3. 包验收（npm-pack）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/dependency-resolution

- `npm-pack:` 来源专为**打包验收**设计：把包真正打出来后在隔离环境安装验证，确认 `dependencies` 齐全、入口可加载、无缺失运行时依赖。**正式发布前做一遍 sanity check**。
- 用法即在安装来源处用 `npm-pack:<...>` 触发隔离安装验证。

---

## 4. 内置插件清单（Plugin inventory）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/plugin-inventory

- 官方维护一份内置插件清单（按能力分类：渠道、提供商、工具、媒体、搜索等），用于查阅"哪些能力已有内置实现、可作为参考或避免重复"。
- 写新插件前先查清单，确认没有现成内置/社区插件覆盖你的能力，避免重复造轮子或命名冲突。

---

## 5. 兼容捆绑包（Bundle）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/bundles

- 捆绑包是 **Codex / Claude / Cursor** 格式的内容型包，信任边界更窄，**不注册进程内能力**，主要携带指令/配置。
- 适合"给某个 harness 一段提示词/配置"的场景；若需要注册工具/渠道/提供商，必须用**原生插件**而非 bundle。
- 与原生插件的关系见 `references/plugin-forms-and-manifest.md` 第 3 节。
