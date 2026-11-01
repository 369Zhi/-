# 通义千问（qianwen.com）原始回答采集

- 采集时间：2026-11-01 15:18–15:33（Asia/Shanghai）
- 站点/页面：https://www.qianwen.com/ （`?source=tongyigw`；发送后跳 `/chat/42e4c58be6804e62a84025ac6de889f7`、`/chat/a35867ce16c94571a625ea9b280db53a`，两次均回落 `/chat`）
- 浏览器：用户已登录的调试 Edge（CDP 端口 9222，profile `.edge-ai`），标签页 id `E4FACEAA25672B93A8F8291A8768D18B`
- 模型显示名称：输入框上方模型按钮「**Qwen-快速**」；侧栏账号/工作区按钮「Qwen6405」；页脚「内容由AI生成，可能不准确，请注意核实」
- 采集方式：CDP 原生 WebSocket（`Runtime.evaluate` / `Input.insertText` / `Input.dispatchKeyEvent` / `Input.dispatchMouseEvent`），未使用 `scripts\ai-ask.ps1`（其 `Input.insertText` + Enter 路径在该站点未产生回答，改用按钮点击触发）
- **结果：三题均未取得回答 —— 站点侧请求异常（空回答 + 「对话不存在」）。以下为问题原文与实测观察，无任何改写。**

---

## Q1

**问题原文**

> 在 Windows 上用 PowerShell 5.1 通过 CDP 驱动 Edge：Input.insertText 与 Input.dispatchKeyEvent 有什么区别？为什么 insertText 有时往 contenteditable 里插不进内容？

**回答全文**

> 未获取。发送成功（进入 `停止回答` 生成态），但回答气泡 innerText 始终为 0 字符；约 4.5 分钟后页面提示会话失效，再次刷新出现「对话不存在」。

**观察**

1. 输入框为 Slate 编辑器：`div[data-slate-editor="true"][contenteditable="true"]`，`data-placeholder="向千问提问"`。
2. `Input.insertText` 在该 contenteditable **可用**：133 字问题一次写入，回读 `innerText` 与原文逐字相等（`match=True`）。
3. 清空方式：`Range.selectNodeContents` 选中 + `Input.dispatchKeyEvent` Backspace（keyDown/keyUp, vkey 8）有效。
4. 触发发送：`button[aria-label="发送消息"]` 的 JS `.click()` 有效（首问、新会话各生效一次）；`Input.dispatchMouseEvent` 按同一坐标点按在本页面**未生效**（发送按钮坐标随窗口状态变化：949,627 → 965,643 → 923,688/923,716）。
5. 发送后 URL 变为 `/chat/<id>`，用户消息气泡正常渲染（133–267 字符），生成态按钮显示 `停止回答`。
6. 回答气泡（`.chat-room-outer-wrap` 的最后一个子节点，`<section>`）始终 `innerText` 长度 0；`[class*="mainContent"]` 文本停在 225–476 字（全是侧栏/工具条/免责声明），无任何正文。
7. 轮询约 100s + 96s 两轮，长度恒定（`stable` 递增到 19/12），排除慢速流式；`break` 条件 `len -gt 300` 从未满足。
8. 点击「停止回答」后按钮短暂消失、2s 后恢复，编辑器内容仍保留 133 字（未被清空，与该站正常流程不一致）。

---

## Q2

**问题原文**

> PowerShell 5.1 执行 .ps1 时，如果文件是 UTF-8 无 BOM 且含中文，会出现什么现象？根因是什么？

**回答全文**

> 未获取 —— 站点恢复前按指令停手，避免反复发送触发风控（Q1 已发送 2 次仍未出字）。

---

## Q3

**问题原文**

> 用「三个不同厂商的大模型交叉验证同一个结论」这种方法，最大的失效场景是什么？给出反例。

**回答全文**

> 未获取 —— 同上，未发送。

---

## 站点故障证据（原文摘录）

| 时间点 | 页面文字（原样） | 说明 |
| --- | --- | --- |
| 15:26 首次发送后 ~100s | `停止回答` 常显、回答气泡 0 字 | 生成卡死 |
| 15:31 点击停止后 | 页面内无错误文案，回答气泡仍 0 字 | 静默失败 |
| 15:32 刷新后 | `对话不存在`（`body.innerText` 末尾） | 会话被服务端丢弃 |
| 15:33 新会话再发一次 | `path=/chat/a35867ce…`，`stop=1`，回答气泡 `len=0`，随后正文出现 `对话不存在` | 同一故障复现 |

- 队长巡场同时观察到页面提示「请求异常，请检查网络连接」；本地两次 CDP 抓取中该 toast 未在 `body.innerText` 稳定出现（出现即消失），故此处只记为旁证。

---

## 可复用结论（供流水线参考）

1. 千问站点的输入写入**不需要** `dispatchKeyEvent` 逐字输入，`Input.insertText` 一次即可（133 字无丢失、无乱码）。
2. 发送按钮必须显式点击，且 `aria-label="发送消息"`；Enter 未验证成功；按钮在生成态下被替换为 `aria-label="停止回答"`，此时 `querySelector('button[aria-label="发送消息"]')` 返回 `null`——这解释了 `ai-ask.ps1` 在千问上「插入了但没发出去」的观感。
3. 就绪判定不能只看 `document.body.innerText.length`：该站 body 里常驻大量侧栏/推广文案（约 440–560 字且会波动），必须定位回答气泡节点（`.chat-room-outer-wrap` 末子节点）再判长度。
4. 该站会静默失败：`停止回答` 显示但正文 0 字，需设「无首字超时」（建议 15s 内无正文字符即判定失败并停止生成），否则会像本次一样白等 4 分钟。
5. 本轮千问源**不可用于三源对照**：本次 3 题 0 回答，需等站点恢复后重采。
