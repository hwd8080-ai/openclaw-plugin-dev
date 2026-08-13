# 安装来源与依赖解析参考

> 本文件汇总插件的安装来源、依赖解析规则、安装覆盖（override）与社区/市场。发 ClawHub 的实操流水线见 `SKILL.md` 第 7–9 节与 `references/build-and-publish.md`。

---

## 1. 安装来源（Install sources）

OpenClaw 支持多种来源前缀 / 形式：

| 来源 | 写法 | 说明 |
|---|---|---|
| **ClawHub** | `clawhub:@<owner>/<plugin>` | 官方插件市场；发布流水线目标 |
| **npm** | `npm:@<scope>/<pkg>` | 从 npm 安装 |
| **git** | `git:<url>` | 从 git 仓库安装（克隆到 `~/.openclaw/git`） |
| **npm-pack** | `npm-pack:<...>` | 用于**包验收测试**（验证打包后能否正常安装/加载），发布前 sanity check |
| **本地路径** | 本地目录路径 | 源码本地开发测试 |
| **Marketplace** | 市场条目 | 经市场分发的插件 |

> 来源：https://docs.openclaw.ai/zh-CN/plugins/manage-plugins（管理/安装插件）
> 社区与市场：https://docs.openclaw.ai/zh-CN/plugins/community
> npm-pack 验收：https://docs.openclaw.ai/zh-CN/plugins/dependency-resolution

---

## 2. 依赖解析：运行时依赖必须进 `dependencies`

> 来源：https://docs.openclaw.ai/zh-CN/plugins/dependency-resolution （核心规则）

- **运行时依赖必须写在 `dependencies` 或 `optionalDependencies`，绝不能只写 `devDependencies`**——发布后 `devDependencies` 不会被安装，导致运行时 `import` 失败。
- 安装根（install roots）：
  - npm 项目：`~/.openclaw/npm/projects/<encoded-package>`
  - git：`~/.openclaw/git`
- `npm-pack:` 来源专为**打包验收**设计：把包打出来后在隔离环境安装验证，确认 `dependencies` 齐全、入口可加载，再正式发布。
- 构建产物（`dist/index.mjs` 由 esbuild 自包含）通常不依赖运行时 `npm install`；但若插件在运行时 `import` 第三方包（而非 bundled），则该包必须在 `dependencies`。

> 实战提醒（`SKILL.md` 第 5 节）：esbuild 把 `index.ts` + 依赖打成自包含 `dist/index.mjs`，运行时**不 npm install**——若用 esbuild bundle，则 `dependencies` 仅用于构建期；若运行时动态 import，则必须进 `dependencies`。

---

## 3. 安装覆盖（Install overrides）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/install-overrides

- 可在宿主配置里对插件安装行为做覆盖（override），例如覆盖来源、版本、路径解析等，用于：
  - 把某个插件临时指向本地/特定 git 提交调试；
  - 固定/锁定某版本；
  - 绕过默认来源选择。
- 覆盖是**安装期**配置，不影响清单本身。
- 注意与 `plugins.allow` / `plugins.deny` / `plugins.load.paths` / `plugins.slots`（见 `references/architecture-and-loading.md` 信任模型）区分：前者管"怎么装"，后者管"谁被信任/加载"。

---

## 4. 管理插件（启用 / 信任 / 拒绝）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/manage-plugins

- `plugins.entries.<id>.enabled` — 启用/禁用某个插件（源码 clone 后必须手动加才加载，`enabledByDefault` 不落盘）。
- `plugins.entries.<id>.config` — 插件配置（受 `configSchema` 校验）。
- `plugins.entries.<id>.hooks` — 钩子级授权（`allowConversationAccess` / `allowPromptInjection`）。
- `plugins.entries.<id>.subagent.allowModelOverride` / `allowedModels` — 子 agent 模型覆盖授权。

---

## 5. 社区与市场

> 来源：https://docs.openclaw.ai/zh-CN/plugins/community

- 社区/市场是用户发现与安装插件的入口；发布到 ClawHub 后插件即进入可发现状态。
- 刚发布几分钟 `Scan: suspicious` 是过渡态，约 10~15 分钟自动转 `clean`（见 `SKILL.md` 第 9 节）。
