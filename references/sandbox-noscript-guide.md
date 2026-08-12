# 沙盒 noscript 提示页（含纯 CSS 中英文切换）

openclaw 把插件页塞进 `<iframe sandbox>`：冷刷新时首屏常处于 strict 沙箱（无 `allow-scripts`），脚本被禁，页面空白、console 报 `Blocked script execution ... allow-scripts is not set`。本文件提供一套**可直接复用的代码与逻辑**：任何 openclaw 插件页只要把这套 CSS + noscript HTML + reveal 脚本贴进去，就能在脚本被禁时显示一条清晰的「温馨提示」引导，并支持**纯 CSS 中英文切换**（沙盒禁脚本时也能切，无需 JS）。

> 设计原则：`.app{display:none}` 默认隐藏真实 UI，只有脚本成功运行（非 strict 沙箱）时才 reveal；脚本被禁时 `<noscript>` 内容正好渲染。中英文切换用 `radio + :checked` 兄弟选择器，纯 CSS、零 JS 依赖。

---

## 1. CSS（贴进页面 `<style>`，或加进项目的 CSS 常量）

```css
/* 沙盒兜底：默认隐藏真实 UI，仅当脚本成功运行后才显示 */
.app{display:none}

/* noscript 提示页（脚本被禁时显示） */
.noscript-guide{position:fixed;inset:0;display:flex;align-items:center;justify-content:center;padding:24px;z-index:1}
.noscript-guide-inner{max-width:440px;width:100%;background:var(--paper,#fffefa);border:1px solid var(--line,#e8e3da);border-left:4px solid var(--teal,#168f89);border-radius:var(--radius,14px);padding:28px 32px;box-shadow:var(--shadow,0 2px 10px rgba(86,75,56,.06))}
.noscript-guide h2{margin:0 0 12px;font-size:16px;font-weight:700;color:#0f7a74;letter-spacing:-.01em;display:flex;align-items:center;gap:10px}
.noscript-guide h2 .dot{display:inline-block;width:9px;height:9px;border-radius:50%;background:var(--teal,#168f89);box-shadow:0 0 0 4px rgba(22,143,137,.18);animation:pulse 1.6s ease-in-out infinite}
.noscript-guide p{margin:0;font-size:13.5px;line-height:1.75;color:#3a3833}

/* 纯 CSS 中英文切换（无需 JS，strict 沙盒下也能用） */
.noscript-guide .lang-tab{display:inline-block;cursor:pointer;padding:4px 16px;border:1px solid var(--line,#e8e3da);border-radius:999px;font-size:12.5px;color:var(--muted,#78746d);user-select:none;background:var(--paper,#fffefa)}
.noscript-guide input[name=sa-lang]{position:absolute;opacity:0;width:0;height:0;pointer-events:none}
.noscript-guide #sa-zh:checked~.lang-tab[for=sa-zh],
.noscript-guide #sa-en:checked~.lang-tab[for=sa-en]{color:#fff;background:var(--teal,#168f89);border-color:var(--teal,#168f89)}
.noscript-guide .lang-block{display:none}
.noscript-guide #sa-zh:checked~.lang-zh{display:block}
.noscript-guide #sa-en:checked~.lang-en{display:block}
.noscript-guide .lang-zh p,.noscript-guide .lang-en p{margin:0;font-size:13.5px;line-height:1.85;color:#3a3833}

@keyframes pulse{0%,100%{opacity:1;transform:scale(1)}50%{opacity:.45;transform:scale(.85)}}
```

> CSS 变量（`--paper` 等）若你的主题未定义，上面已给 `var(--x, 回退值)`，可原样使用；或把回退值直接写死。

---

## 2. noscript HTML（放在 `<body>` 顶部，真实 UI 之前）

> **结构铁律**：`input#sa-zh` / `input#sa-en`、两个 `.lang-tab` label、两个 `.lang-block` **必须全部是同一父元素（`.noscript-guide-inner`）的直接子节点且依次排列**，`#sa-zh:checked ~ .lang-zh` 这类兄弟选择器才能生效。

### 形式 A：直接写进 HTML 模板（属性用双引号）

```html
<noscript><div class="noscript-guide"><div class="noscript-guide-inner">
  <h2><span class="dot"></span>温馨提示</h2>
  <input type="radio" name="sa-lang" id="sa-zh" checked>
  <label class="lang-tab" for="sa-zh">中文</label>
  <input type="radio" name="sa-lang" id="sa-en">
  <label class="lang-tab" for="sa-en">English</label>
  <div class="lang-block lang-zh"><p>本插件受 openclaw 沙箱安全策略限制，每次刷新时页面脚本可能未能正常执行，导致功能暂时不可用。请点击左侧任意菜单（如「概览」「活动」），再回到本页即可恢复正常。</p></div>
  <div class="lang-block lang-en"><p>This plugin is restricted by openclaw's sandbox security policy. Each refresh may fail to run the page scripts, leaving the page temporarily unusable. Please click any left-side menu (e.g. "Overview" / "Activity"), then return to this page to restore normal function.</p></div>
</div></div></noscript>
```

### 形式 B：数组 join 拼 HTML 的项目（如 session-admin / branding 的 `lines.push("...")`）

属性改用**单引号**，整串作为数组元素（注意英文文案里的双引号要保留在 HTML 属性外，或用 `&quot;`）：

```js
lines.push("<noscript><div class='noscript-guide'><div class='noscript-guide-inner'><h2><span class='dot'></span>温馨提示</h2><input type='radio' name='sa-lang' id='sa-zh' checked><label class='lang-tab' for='sa-zh'>中文</label><input type='radio' name='sa-lang' id='sa-en'><label class='lang-tab' for='sa-en'>English</label><div class='lang-block lang-zh'><p>本插件受 openclaw 沙箱安全策略限制，每次刷新时页面脚本可能未能正常执行，导致功能暂时不可用。请点击左侧任意菜单（如「概览」「活动」），再回到本页即可恢复正常。</p></div><div class='lang-block lang-en'><p>This plugin is restricted by openclaw's sandbox security policy. Each refresh may fail to run the page scripts, leaving the page temporarily unusable. Please click any left-side menu (e.g. &quot;Overview&quot; / &quot;Activity&quot;), then return to this page to restore normal function.</p></div></div></div></noscript>");
```

> 中文非 ASCII 字符在 esbuild 打包后会被转成 `\uXXXX` 转义——**运行时浏览器会自动还原**，无需处理；只有你在 `node --check` 或 grep 时看到的是转义形态。

---

## 3. 真实 UI 容器 + reveal 脚本（两处都要）

真实 UI 包在 `<main>` 里，**必须同时带 `id="app"` 和 `class="app"`**：

```html
<main id="app" class="app"> ... 真实 UI ... </main>
```

reveal 脚本**必须防御式**——放在客户端 bundle 末尾，以及服务端 `</body>` 前各一份。脚本被禁时它不执行（`.app` 保持隐藏，显示 noscript）；脚本可用时显示真实 UI：

```js
// 注意 display 值：flex（纵向布局容器）或 block（普通块），按 .app 实际布局选
var _app=document.getElementById('app'); if(_app){_app.style.display='flex';}
```

> 服务端 `</body>` 前若不能注入脚本（纯 SSR 静态页），把 reveal 放进页面内联 `<script>` 即可，效果相同。

---

## 4. 致命坑（已踩过，必读）

1. **`<main>` 必须有 `id="app"`**：reveal 用 `getElementById('app')`。漏写 → 拿到 `null` → `null.style` 抛 `Cannot read properties of null (reading 'style')` → reveal 不执行 → 整页空白。
2. **reveal 必须防御式**：先判空再设 `display`，否则任意空引用都会崩页。
3. **`:checked ~` 兄弟选择器**：radio、label、lang-block 必须同一父、依次排列；任一嵌套错位切换失效。
4. **中英文切换用纯 CSS**：strict 沙箱禁脚本，JS 切换在那一刻不可用；`radio + :checked` 是浏览器原生能力，禁脚本也照常工作。
5. **两套文案写死在 HTML**：禁脚本环境无法动态注入文本，中文 + English 都直接放进 `<noscript>`。
6. **`.app{display:none}` 与 reveal 成对**：缺一不可——有隐藏没 reveal 会永久空白；有 reveal 没隐藏则 noscript 与真实 UI 叠在一起。
