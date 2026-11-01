# 开源检索：CDP / 浏览器自动化 + 「向 contenteditable 输入文本」

- 检索者：oss-crawl-cdp ｜ 时间：2026-11-01 ｜ 工具：`gh search repos / gh search issues / gh api`（gh 2.102.1，账号 369Zhi）
- 硬门槛：近 6–12 个月有 push、LICENSE 明确、非 archived
- 数据来源：GitHub REST（`repos/{owner}/{repo}` 的 `pushed_at` / `license.spdx_id`）、`gh search issues`、仓库内源码文件（puppeteer/playwright/rod 的关键文件已读，行号见下）

## 一、候选项目（5 个，全部 MIT / Apache-2.0，可安全借鉴思路）

| # | 仓库全名 | Star | 最近提交 (push) | LICENSE | 关键实现文件（可借鉴点所在） |
|---|---|---|---|---|---|
| 1 | [puppeteer/puppeteer](https://github.com/puppeteer/puppeteer) | 95.8k | 2026-11-01 | Apache-2.0 | [cdp/Input.ts](https://github.com/puppeteer/puppeteer/blob/main/packages/puppeteer-core/src/cdp/Input.ts) — `CdpKeyboard` |
| 2 | [microsoft/playwright](https://github.com/microsoft/playwright) | 99.0k | 2026-10-31 | Apache-2.0 | [server/input.ts](https://github.com/microsoft/playwright/blob/main/packages/playwright-core/src/server/input.ts)、[chromium/crInput.ts](https://github.com/microsoft/playwright/blob/main/packages/playwright-core/src/server/chromium/crInput.ts)、[injected/injectedScript.ts](https://github.com/microsoft/playwright/blob/main/packages/injected/src/injectedScript.ts) |
| 3 | [cyrus-and/chrome-remote-interface](https://github.com/cyrus-and/chrome-remote-interface) | 4.6k | 2026-02-09 | MIT | [index.js](https://github.com/cyrus-and/chrome-remote-interface/blob/master/index.js) — 薄协议客户端，`client.Input.insertText({text})` 直发 |
| 4 | [go-rod/rod](https://github.com/go-rod/rod) | 7.1k | 2026-08-11 | MIT | [input.go](https://github.com/go-rod/rod/blob/main/input.go)、[element.go](https://github.com/go-rod/rod/blob/main/element.go) — `Element.Input()` |
| 5 | [ClericPy/ichrome](https://github.com/ClericPy/ichrome) | 256 | 2026-04-29 | MIT | `ichrome/daemon.py`、`ichrome/pool.py` — **接管已开着的 Chrome**（不新起 profile，天然的"已登录"复用） |

> 排除：`cdpdriver/zendriver`、`vmoranv/jshookmcp` 为 **AGPL-3.0**，按项目红线不读源码、不抄。`ProseMirror/prosemirror` 已 archived，`browserless/browserless` LICENSE 为 `NOASSERTION`，均不入围。

## 二、3 条可直接借鉴的做法（重写要点，非拷代码）

1. **输入分两条路：能映射成按键的走 `dispatchKeyEvent`，其余走 `Input.insertText`。**
   puppeteer `Input.ts:177-194` 与 playwright `crInput.ts:111-122` 逻辑一致：逐字符，若该字符在 US 键盘表里 → `press(char)`（发 keyDown/keyUp，`type: text ? 'keyDown' : 'rawKeyDown'`，puppeteer `Input.ts:69-81`）；否则 → `sendCharacter` → `Input.insertText`（puppeteer `Input.ts:169-171`）。
   **重写要点**：中文提问整句都是"非键盘字符"，所以主流实现等于全靠 `Input.insertText`；逐字符循环对中文毫无收益（每字一次 CDP 往返），应当整段一次 `insertText`。

2. **contenteditable 的"替换而不是追加"靠 Range + Selection，不靠 `select()`。**
   playwright `injectedScript.ts:939-970` `selectText()`：先向上找到真正的编辑宿主（`while (el.isContentEditable && parent.isContentEditable) focusTarget = parent`）再 `focus()`，然后 `range.selectNodeContents(el)` + `selection.removeAllRanges()/addRange()`；随后 `dom.ts:625-649` 只在 `result === 'needsinput'` 时才 `keyboard.insertText(value)`，空值走 `press('Delete')`。
   **重写要点**：给豆包/DS 的输入框清场时用 `Range.selectNodeContents` 覆盖旧文，再一次性 `insertText` 覆盖，就能避免"越问越长"。

3. **写入后补一次事件 + 校验，别相信 `insertText` 一定生效。**
   rod `element.go:261-280` `Input()` 的顺序值得照抄：`Focus()` → `WaitEnabled()` → `WaitWritable()`（`states.go` 里按 `disabled/readonly/isContentEditable` 判断）→ `InsertText` → 再 `Evaluate` 补发一次 `InputEvent`（`lib/js` 的 `InputEvent` helper）。
   **重写要点**：`Input.insertText` 走的是浏览器编辑管线，**只产生 `beforeinput`/`input`（`inputType=insertText`），不产生 `keydown`/`keypress`/`compositionend`**——发完必须回读输入框 `innerText` 做断言，再决定补发 `input` 或降级。

## 三、2 个已知坑（issue 只取标题+结论）

1. **contenteditable 未聚焦时 `fill()` 会变成追加** — [microsoft/playwright#39492](https://github.com/microsoft/playwright/issues/39492)（closed，2026-08-03，1.56.0 + Chromium + Jodit 编辑器实测）：先 `fill('old text')`、点到 `body` 失焦、再 `fill('new text')` → 编辑器里是 `old textnew text`。根因是内部 selectAll 依赖焦点，编辑器的 focus handler 抢在前面。
   → 对策：每次输入前**显式 click 聚焦到编辑宿主本身**，再 select-all（Range），再 insertText，写完回读校验；不要指望"上一步还聚焦着"。

2. **IME / composition 事件至今没有官方支持，中文长句只能"整段灌入"** — [puppeteer/puppeteer#4981](https://github.com/puppeteer/puppeteer/issues/4981)（**仍 open**，11 条评论，2026-03-05 仍有更新）；配合协议原文：`Input.insertText` 的定义就是 *"emulates inserting text that doesn't come from a key press, for example an emoji keyboard or an **IME**"*（[devtools-protocol Input domain](https://github.com/ChromeDevTools/devtools-protocol/blob/master/json/browser_protocol.json)，已本地校验 `version 1.3`）。
   → 后果：靠 `compositionstart/end` 做"回车提交 / @ 菜单 / 拼音联想"的前端（含多家大模型网页）不会收到这些事件，`insertText` 可能被自己的 composition 状态机吞掉。需要真 IME 时只能自己走 `Input.imeSetComposition` + `Input.imeCommitComposition` 两步（协议里已提供，但两个库都未封装）。
   → 另一条同源坑（标题即结论，不再展开）：[#42125](https://github.com/microsoft/playwright/issues/42125) `fill()` 在聚焦中途漂移时会把文本写进**另一个元素**；[#27130](https://github.com/microsoft/playwright/issues/27130) `type()` 在需要按住修饰键的字符上并不真正"按键"。

## 四、对我们这条流水线的结论

- **不要引入 puppeteer 依赖**：三个站点都在用户已登录的 Chrome 里，`puppeteer.launch()` 会新开 profile（登录态全丢）。走 **attach 现有实例**：`chrome-remote-interface` 的 `CDP({host, port})` 或 ichrome 的接管思路（`--remote-debugging-port` + 复用 user-data-dir），思路可借鉴，代码自己写。
- **输入策略（三站点统一）**：click 聚焦编辑宿主 → `Range.selectNodeContents` 全选 → `Input.insertText(整段中文一次发)` → 100–200ms 后回读 `innerText` 校验 → 不合格则补 `input` 事件或降级为 `dispatchKeyEvent` 逐字符。
- **发送**：`Input.dispatchKeyEvent` 发 Enter（`keyDown`+`keyUp`，带 `windowsVirtualKeyCode:13`）；若站点是按钮发送，则用 `Runtime.evaluate` 点按钮，比按键更稳（避免 #27130 类陷阱）。
- **许可**：以上 5 个候选全为 MIT/Apache-2.0，只借鉴思路、按本文"重写要点"独立实现；AGPL 的 zendriver / jshookmcp 一行代码都不进仓库。
