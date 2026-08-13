# 钩子（Hooks）与权限请求参考

> 本文件汇总 OpenClaw 插件的钩子机制（typed hooks）与权限模型（插件权限请求 / 可选工具 / Exec 审批 / Codex 原生权限）。与 `SKILL.md` 第 4 节的实战坑（钩子 name 必填、对话访问门控）互补。

---

## 1. 两类钩子注册

| 方式 | 用途 | 说明 |
|---|---|---|
| `api.on(name, handler, opts)` | **类型化钩子**（推荐） | 受类型约束、可声明 `opts`；用于 agent 回合、对话、工具、消息、会话、子 agent、生命周期等 |
| `api.registerHook(name, handler)` | 内部/旧式钩子注册 | 第三参必须带 `name`（如 `{ name: "<id>:hook" }`），缺失会被 `requireRegistrationValue` 拦截并**中断整个 `register(api)`** |

> 来源：https://docs.openclaw.ai/zh-CN/plugins/hooks
> 实测坑（name 缺失导致 register 中断、管理页 404）：`SKILL.md` 第 4 节

---

## 2. 钩子目录（Catalog）

类型化钩子按主题分为几大类（详见官方"钩子"页）：

- **Agent 回合（agent turn）**：回合开始/结束、推理前/后。
- **对话（conversation）**：`inbound_claim`、`message_received` 等 —— 注入/改写消息需 `allowPromptInjection`。
- **工具（tools）**：`before_tool_call` 等（见第 3 节）。
- **消息（messages）**：消息落库前后。
- **会话（sessions）**：会话创建/切换。
- **子 agent（subagent）**：子 agent 启动/完成。
- **生命周期（lifecycle）**：插件启用/禁用、daemon 启动。

> 来源：https://docs.openclaw.ai/zh-CN/plugins/hooks

---

## 3. `before_tool_call` —— 工具调用前拦截

返回对象可携带以下字段，控制工具是否放行：

- `requireApproval: true` — 要求用户/操作员审批后才执行。
- `block: true` — 直接阻断该次工具调用。
- `params` — 改写/补充传给工具的参数。

> 来源：https://docs.openclaw.ai/zh-CN/plugins/hooks（"工具钩子"小节）
> 与"插件权限请求"区别见第 4 节。

---

## 4. 权限模型：四种机制不要混

> 来源：https://docs.openclaw.ai/zh-CN/plugins/plugin-permission-requests 与 https://docs.openclaw.ai/zh-CN/plugins/adding-capabilities

| 机制 | 触发点 | 适用 |
|---|---|---|
| **插件权限请求（Plugin permission requests）** | 插件运行时向宿主请求一次性/持久授权（如访问某 surface） | 插件需要宿主级能力但默认没给 |
| **可选工具（`toolMetadata.<tool>.optional` + `tools.allow`）** | 工具按需由用户开启 | 非默认启用的工具 |
| **Exec 审批** | 执行 shell / 命令前审批 | 命令执行类动作 |
| **Codex 原生权限** | Codex harness 自身的权限体系 | Codex 捆绑/原生上下文 |

要点：不要为"可选工具"走"插件权限请求"，二者是不同通道。需要持久授权时优先用可选工具 + `tools.allow`；需要临时/一次性宿主授权才用插件权限请求。

---

## 5. 对话访问门控（非 bundled 插件必看）

对话类钩子（`inbound_claim` / `message_received` 等）对非 bundled 插件**额外要求信任授权**，否则被**静默拦截**（只 push warn、不注册、不抛异常）：

- `plugins.entries.<id>.hooks.allowConversationAccess: true` — 允许插件读取/绑定对话。
- 若还要注入回复 / 改写消息：额外 `allowPromptInjection: true`。

> 来源：https://docs.openclaw.ai/zh-CN/plugins/architecture-internals（"对话绑定回调"与加载门控）
> 实战踩坑：`SKILL.md` 第 4 节"坑#（对话类钩子静默拦截）"——mock-import 测试发现不了，必须真 daemon 加载看 gateway 日志。

---

## 6. 对话绑定回调

绑定对话的插件可在审批解决后收到回调：

```ts
api.onConversationBindingResolved(async (event) => {
  if (event.status === "approved") {
    console.log(event.binding?.conversationId);
    return;
  }
  // 被拒：清理本地待处理状态
  console.log(event.request.conversation.conversationId);
});
```

- 载荷：`status`（`approved`/`denied`）、`decision`（`allow-once`/`allow-always`/`deny`）、`binding`、`request`。
- **仅用于通知**，不改变谁有权绑定对话，且跑在核心审批处理完成后。

> 来源：https://docs.openclaw.ai/zh-CN/plugins/architecture-internals
