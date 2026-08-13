# 架构与加载机制参考

> 本文件汇总 OpenClaw 插件的四层架构、加载流水线、注册表模型、清单优先原则、缓存边界、Gateway HTTP 路由与 SDK 导入路径。偏"为什么这样设计"，配合 `SKILL.md` 的"怎么跑起来"。

---

## 1. 四层架构

> 来源：https://docs.openclaw.ai/zh-CN/plugins/architecture

1. **清单 + 发现（manifest + discovery）**：扫描候选根目录、读清单与包元数据、拒绝不安全项。
2. **启用 + 校验（enable + validate）**：按 `plugins.enabled` / `allow` / `deny` / `entries` / `slots` / `load.paths` 规范化配置，决定是否启用。
3. **运行时加载（runtime load）**：加载已启用的原生模块，调用 `register(api)`，把注册项收集进插件注册表。
4. **接口消费（interface consume）**：命令/运行时界面从注册表读取能力并暴露。

---

## 2. 加载流水线（启动时的实际步骤）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/architecture-internals

1. 发现候选插件根目录。
2. 读取原生或兼容捆绑包清单 + 包元数据。
3. **拒绝不安全候选项**（安全门控在运行时执行前）：
   - 解析后的入口**逸出插件根目录**；
   - 路径（或根）**可由所有用户写入**；
   - 非内置插件路径**所有权与当前 uid（或 root）不匹配**。
   - 世界可写的内置目录：门控会先尝试就地 `chmod` 修复再重新检查；**内置来源完全跳过所有权检查**。
   - 被阻止项若已知插件 ID，诊断信息仍携带该 ID（含从会被拒目录内清单解析出的 ID），便于定位"被阻止"而非"未知插件"。
4. 规范化插件配置（`plugins.enabled` / `allow` / `deny` / `entries` / `slots` / `load.paths`）。
5. 决定每个候选项是否启用。
6. 加载已启用的原生模块（构建后内置模块用原生加载器；第三方本地 TS 用应急 Jiti 回退）。
7. 调用 `register(api)`，收集注册项进插件注册表。
8. 向命令/运行时界面公开注册表。
9. **安全门控**在运行时执行（第 3 步是发现期门控，此处是运行时门控）。

---

## 3. 注册表模型（单向加载）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/architecture-internals

- 已加载插件**不直接改任意核心全局变量**，而是注册进中央 **PluginRegistry**（跟踪插件记录：身份/来源/源类型/状态/诊断），以及每种能力的数组：工具、旧式钩子与类型化钩子、渠道、提供商、Gateway RPC 处理程序、HTTP 路由、CLI 注册器、后台服务、插件自有命令，以及数十种类型化提供商系列（语音/嵌入/图像视频音乐生成/Web 获取搜索/Agent harness/会话操作等）。
- 核心功能从注册表读取，而非直接与插件模块通信。加载保持单向：
  - 插件模块 → 注册表注册
  - 核心运行时 → 注册表消费
- 意义：核心界面只需一个集成点"读注册表"，无需"为每个插件特殊处理"。

---

## 4. 清单优先（Manifest-first）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/architecture-internals

清单是**控制平面事实来源**，用于：
- 识别插件；
- 发现已声明的渠道 / Skills / 配置 Schema / 捆绑能力；
- 校验 `plugins.entries.<id>.config`；
- 补全 Control UI 标签/占位符；
- 展示安装/目录元数据；
- 保留轻量 `activation` / `setup` 描述符，**而不加载运行时**。

关键：**`activation` 与 `setup` 块只是控制平面元数据描述符，不能取代运行时 `register(...)` 或 `setupEntry`**。实时激活规划利用清单中的命令/渠道/提供商提示，在广泛注册表实体化前缩小加载范围（如 CLI 加载缩到拥有主命令的插件、渠道设置缩到拥有渠道 ID 的插件）。

激活规划原因分两类（兼容性边界）：
- `activation-*-hint`（来自 `activation.*` 提示）
- `manifest-*-owner`（来自 `channels` / `commandAliases` / `providers` / `setup.providers` / `hooks` / `contracts.tools` 等所有权）

---

## 5. 缓存边界（不要加隐藏持久缓存）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/architecture-internals

- OpenClaw **不会**把"发现结果 / 直接清单注册表"缓存在挂钟时间窗口后。安装、清单编辑、加载路径更改必须在下次显式元数据读取或快照重建时可见。
- 清单解析器维护**有界文件签名缓存**（键 = 打开的清单路径 + 设备/inode + 大小 + mtime/ctime），仅避免重解析未改字节——**不得**缓存发现/注册表/所有者/策略答案。
- 安全的元数据快速路径是**显式对象所有权**，不是隐藏缓存。Gateway 热路径应传递当前 `PluginMetadataSnapshot` / `PluginLookUpTable` / 显式清单注册表。
- 持久缓存层是**运行时加载**（如 `PluginLoaderCacheState`、jiti/模块缓存、已安装工件文件系统缓存）——这些是数据平面实现细节，**不得**回答"哪个插件拥有此提供商"之类的控制平面问题。

---

## 6. Gateway HTTP 路由

> 来源：https://docs.openclaw.ai/zh-CN/plugins/architecture-internals

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",          // 必填："gateway" 或 "plugin"
  match: "exact",          // 可选："exact"(默认) 或 "prefix"
  handler: async (_req, res) => {
    res.statusCode = 200; res.end("ok");
    return true;           // 已处理必须 return true
  },
  // handleUpgrade?: 处理同路由 WebSocket 升级
  // replaceExisting?: 允许同插件替换自己的路由注册
});
```

规则：
- **`api.registerHttpHandler(...)` 已被移除**，用它会导致加载错误，改用 `registerHttpRoute`。
- `auth` **必填**。`"gateway"` = 常规 Gateway 身份验证；`"plugin"` = 插件管理身份验证 / webhook 验证。
- 完全相同的 `path + match` 冲突会被拒绝；一个插件**不能**替换另一个插件的路由（除非 `replaceExisting:true` 且是同插件）。
- 不同 `auth` 级别的重叠路由会被拒绝；`exact`/`prefix` 回退链只能用**同一** auth 级别。
- `auth:"plugin"` 路由**不会**自动获得操作员运行时权限范围（仅用于 webhook/签名验证）。
- `auth:"gateway"` 路由默认保守（`gatewayRuntimeScopeSurface:"write-default"`，仅 `operator.write`）；可加入 `"trusted-operator"` 表面以遵循 `x-openclaw-scopes`。
- 处于准备/重启状态的 Gateway 在处理程序前返回 `503`（少数 `trusted-operator` 路由例外）。

---

## 7. 插件 SDK 导入路径（用细粒度子路径）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/architecture-internals

核心子路径（**不要**从单体 `openclaw/plugin-sdk` 根聚合导出）：

- `openclaw/plugin-sdk/plugin-entry` — 插件注册原语
- `openclaw/plugin-sdk/channel-core` — 渠道入口/构建辅助
- `openclaw/plugin-sdk/core` — 通用共享辅助与总括约定

渠道插件可细分导入：`channel-setup` / `setup-runtime` / `setup-tools` / `channel-pairing` / `channel-contract` / `channel-feedback` / `channel-inbound` / `channel-outbound` / `command-auth` / `secret-input` / `webhook-ingress` / `channel-targets` / `channel-actions`。

运行时/配置辅助在 `*-runtime` 子路径（`approval-runtime` / `agent-runtime` / `lazy-runtime` / `directory-runtime` / `text-runtime` / `runtime-store` 等）；优先用 `config-contracts` / `plugin-config-runtime` / `runtime-config-snapshot` / `config-mutation`，而非宽泛的 `config-runtime`。

**弃用兼容性层（新代码勿用）**：`channel-lifecycle` / `config-runtime` / `infra-runtime`。

铁律：
- **外部插件只能导入 `openclaw/plugin-sdk/*` 子路径**；
- **切勿**从核心或别的插件包导入 `src/*`；
- 仓库内部入口（`index.js` / `api.js` / `runtime-api.js` / `setup-entry.js`）仅供内置插件使用。

---

## 8. 信任模型（Trust model）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/architecture

- `plugins.allow` 信任**插件 ID**；原生插件**进程内**运行、**不沙箱**。
- 配置键：`plugins.allow` / `plugins.deny` / `plugins.load.paths` / `plugins.slots`。
- 这与安装覆盖（`references/install-and-dependencies.md` 第 3 节）不同：信任模型管"谁被加载/信任"，覆盖管"怎么装"。

---

## 9. 插件形态分类（按架构角色）

> 来源：https://docs.openclaw.ai/zh-CN/plugins/architecture

- **纯能力型（plain-capability）**：只注册一种能力（如纯工具 / 纯提供商）。
- **混合能力型（hybrid-capability）**：同时注册多种能力（如工具 + HTTP 路由 + 钩子）。
- **纯钩子型（hook-only）**：只挂钩子、不改能力表。
- **非能力型（non-capability）**：仅设置/引导描述符，不注册运行时行为。

> 本 skill 的实测插件（branding / session-label）属**混合能力型**（管理页 + API + 钩子）。
