# 插件形态与清单（Manifest）参考

> 本文件汇总 OpenClaw 插件"有哪几类、各自长什么样、清单怎么写"。属于**官方文档的系统性梳理**，与 `SKILL.md` 里基于真实插件（openclaw-branding / session-label）的实测流水线互补。
> 约定：所有"来源"链接均为官方文档页面；`openclaw.plugin.json` 字段以官方为准，`SKILL.md` 的 v2026.7.1 实测值仅作示例。

---

## 1. 插件与 Skill 是两套平行机制

- **Plugin（插件）**：进程内扩展，能注册工具 / 渠道 / 提供商 / CLI 后端 / 钩子 / HTTP 路由，并拥有自己受信的代码边界。能读写宿主运行时、注册能力。
- **Skill（技能）**：轻量指令包（含 `SKILL.md` + 可选 `references/`、`scripts/`），只改变 agent 的**行为/知识**，不注册宿主能力、不加载运行时模块。
- 二者**不是包含关系**，是平行的两套机制。本 skill 目录（`openclaw-plugin-dev`）本身是一个 Skill，用来教人**开发** Plugin。

> 来源：https://docs.openclaw.ai/zh-CN/tools/plugin （插件概述，解释 Plugin vs Skill 的定位）

---

## 2. 四种插件形态（Forms）

| 形态 | 用途 | 入口 helper | 来源 |
|---|---|---|---|
| **Tool 插件** | 给 agent 注册可调用的工具（含可选工具、工厂工具） | `defineToolPlugin` / `definePluginEntry` | https://docs.openclaw.ai/zh-CN/plugins/tool-plugins |
| **Channel 插件** | 接入新的消息渠道（WhatsApp / 钉钉 / 飞书等），含入站/出站/配对/反馈 | `defineChannelPluginEntry`（来自 `openclaw/plugin-sdk/channel-core`） | https://docs.openclaw.ai/zh-CN/plugins/sdk-channel-plugins |
| **Provider 插件** | 接入新的模型/推理提供商（含 40+ 运行时钩子、目录、用量） | `defineSingleProviderPluginEntry` / `definePluginEntry` + `api.registerProvider` | https://docs.openclaw.ai/zh-CN/plugins/sdk-provider-plugins |
| **CLI 后端插件** | 注册新的 CLI 后端（命令执行/沙箱目标） | `definePluginEntry` + `api.registerCliBackend` | https://docs.openclaw.ai/zh-CN/plugins/cli-backend-plugins |

> 来源总览：https://docs.openclaw.ai/zh-CN/plugins/building-plugins （"构建插件"开篇即列四种形态）
> 能力所有权（plugin = 所有权边界，capability = 共享核心契约）：https://docs.openclaw.ai/zh-CN/plugins/adding-capabilities

---

## 3. 原生插件 vs 兼容捆绑包（Bundle）

- **原生插件（Native plugin）**：`openclaw.plugin.json` + 构建后的运行时模块（如 `dist/index.mjs`），**进程内**加载、不沙箱。能注册工具/渠道/提供商等全部能力。
- **兼容捆绑包（Bundle）**：Codex / Claude / Cursor 格式的**内容型**包，信任边界更窄，通常只携带指令/配置，**不注册进程内能力**。适合"给某个 harness 一段提示词/配置"的场景。

> 来源：https://docs.openclaw.ai/zh-CN/plugins/bundles
> 原生插件构建与加载边界：https://docs.openclaw.ai/zh-CN/plugins/building-plugins

---

## 4. 入口 API 速查

| 入口 helper | 适用 | 导入路径 |
|---|---|---|
| `definePluginEntry` | **非通道**插件（provider / tool / hook / 管理页+API）通用入口 | `openclaw/plugin-sdk/plugin-entry` |
| `defineChannelPluginEntry` | 通道插件，自动处理 discovery 分支 | `openclaw/plugin-sdk/channel-core` |
| `defineToolPlugin` | 纯工具插件（少 boilerplate） | `openclaw/plugin-sdk/tool-plugin` |
| `defineSingleProviderPluginEntry` | 单提供商插件 | `openclaw/plugin-sdk/...`（见 provider 文档） |
| `defineSetupPluginEntry` | 仅做设置/引导、不注册运行时行为的插件 | `openclaw/plugin-sdk/...` |

> 来源：https://docs.openclaw.ai/zh-CN/plugins/building-plugins
> 通道入口：`https://docs.openclaw.ai/zh-CN/plugins/sdk-channel-plugins`
> 提供商入口：`https://docs.openclaw.ai/zh-CN/plugins/sdk-provider-plugins`

### `register(api)` 的通用铁律（来自实测 + 官方）

- 必须按 `api.registrationMode` 分支：discovery / tool-discovery 模式下 OpenClaw 会执行入口模块建快照，**顶层 import 必须无副作用**（禁止顶层起网络客户端/子进程/DB/监听器/读凭据）。重活放进 handler 或按 mode 跳过。
- `manifest.id` 必须与入口 `id` 一致；`contracts.tools` 必须与 `api.registerTool` 注册名一致（否则 tool discovery 不加载）。
- `handler` 必须 `return true`，否则请求漏到后续中间件 → 404。
- 钩子 `api.registerHook(events, handler, opts)` 第三参必须带 `name`（如 `{ name: "<id>:hook" }`），否则 `requireRegistrationValue` 抛 `hook registration missing name`，**中断整个 `register(api)`**。
- 对话类钩子（`inbound_claim` / `message_received` 等）对非 bundled 插件需 `plugins.entries.<id>.hooks.allowConversationAccess:true`（注入回复还要 `allowPromptInjection:true`），否则**静默拦截**。

> 以上实测细节见 `SKILL.md` 第 4 节；官方 loading 门控见 https://docs.openclaw.ai/zh-CN/plugins/architecture-internals

---

## 5. 清单 `openclaw.plugin.json` 字段参考

> 来源：https://docs.openclaw.ai/zh-CN/plugins/building-plugins 与 https://docs.openclaw.ai/zh-CN/plugins/architecture

| 字段 | 作用 | 备注 |
|---|---|---|
| `id` | **唯一标识**（不是 `name`） | 必须与会话/入口/契约一致 |
| `name` | 展示名 | 可与 `id` 不同 |
| `description` | 描述 | 用于 Control UI / 安装元数据 |
| `enabledByDefault` | 默认是否启用 | **仅运行时判定，不落盘**；源码 clone 后仍需 `plugins.entries.<id>.enabled:true` 才加载 |
| `activation.onStartup` | 启动即导入（Gateway 热路径） | 无启动元数据的插件仅按更窄的激活触发器加载 |
| `contracts.tools` | 声明受信的 agent 工具 surface | 注册名需与 `api.registerTool` 对齐 |
| `contracts.gatewayMethodDispatch` | 声明受信的 http 路由 surface | 后台管理页 + API 用此契约 |
| `contracts.agentToolResultMiddleware` | 工具结果中间件契约 | — |
| `contracts.trustedToolPolicies` | 受信工具策略契约 | — |
| `contracts.externalAuthProviders` | 外部认证提供商契约 | provider 插件声明 |
| `configSchema` | 插件配置 JSON Schema | 即使无配置也要给空对象 `{ "type":"object", "properties":{} }` |
| `toolMetadata` | 工具级元数据（标签/可选标记） | `toolMetadata.<tool>.optional:true` 标记可选工具 |
| `channels` | 声明拥有的渠道 ID | 用于设置发现"manifest-channel-owner" |
| `cliBackends` | 声明拥有的 CLI 后端 | — |
| `providers` / `setup.providers` | 声明拥有的提供商 / 设置描述符 | provider 插件用；含 `envVars` / `providerAuthAliases` / `providerAuthChoices` / `channelConfigs` |
| `modelSupport` | 模型支持声明 | — |
| `setup` | 设置/引导描述符（控制平面，仅元数据） | **不能取代**运行时 `register(...)` 或 `setupEntry` |

> 清单是**控制平面事实来源**：识别插件、发现已声明渠道/Skill/配置Schema/捆绑能力、校验 `plugins.entries.<id>.config`、补全 Control UI 标签、展示安装元数据、保留轻量激活/设置描述符而**不加载运行时**。

---

## 6. `package.json` 的 `openclaw` 块

> 来源：https://docs.openclaw.ai/zh-CN/plugins/building-plugins

```jsonc
"openclaw": {
  "extensions": ["./dist/index.mjs"],   // 指向【构建产物】，不能指 .ts 源码
  "build":     { "openclawVersion": "2026.7.1" },  // 写具体版本，不加 >=
  "compat":    { "pluginApi": ">=2026.7.1" },       // 兼容的插件 API 下限
  "setupEntry": "...",                  // 设置插件入口（如有）
  "channel":   "...",                   // 通道插件声明
  "providers": "...",                   // 提供商声明
  "install": {
    "npmSpec":     "@<owner>/<plugin>",
    "clawhubSpec": "@<owner>/<plugin>", // 必填
    "localPath":   "extensions/<plugin>",
    "defaultChoice": "clawhub",
    "minHostVersion": ">=2026.7.1"
  },
  "release": { "publishToClawHub": true, "publishToNpm": false }
}
```

- `extensions` 指向**构建产物** `./dist/index.mjs`；`build.openclawVersion` 写具体版本不加 `>=`；`install.clawhubSpec` 必填。

---

## 7. 能力（Capability）模型：插件是所有权边界，能力是共享契约

核心把每种能力收进中央 **PluginRegistry**，插件只往里"注册"，核心只从注册表"读取"。这样加载保持单向：插件模块 → 注册表注册；核心运行时 → 注册表消费。

常用注册 API（能力 = 共享核心契约，plugin = 所有权边界）：

- `api.registerProvider(...)` — 模型/推理提供商
- `api.registerCliBackend(...)` — CLI 后端
- `api.registerChannel(...)` — 渠道
- `api.registerEmbeddingProvider(...)` — 嵌入提供商
- `api.registerTool(...)` — 工具（见下）
- `api.registerHttpRoute(...)` — Gateway HTTP 路由
- `api.registerSpeechProvider` / `api.registerMediaUnderstandingProvider` / `api.registerVideoGenerationProvider` / `api.registerWebSearchProvider` — 各类媒体/搜索提供商

> 来源：https://docs.openclaw.ai/zh-CN/plugins/adding-capabilities
> 注册表模型：https://docs.openclaw.ai/zh-CN/plugins/architecture-internals

---

## 8. 工具注册要点（Tool 插件）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/tool-plugins 与 https://docs.openclaw.ai/zh-CN/plugins/adding-capabilities

- `api.registerTool({ ... })` 注册工具；**必须**声明 `outputSchema`（工具结果的可选输出结构）。
- **可选工具**：在 `toolMetadata.<tool>.optional:true` 标记，并配合 `tools.allow`（用户按需开启）使用；未开启时不占用工具表。
- **工厂工具（factory tools）**：按需动态生成工具实例。
- 工具名需同时出现在 `contracts.tools`，否则 tool discovery 阶段不加载。
- 与"插件权限请求"的关系见 `references/hooks-and-permissions.md`（可选工具 vs 权限请求 vs Exec 审批 vs Codex 原生权限）。
