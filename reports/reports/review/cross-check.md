# 交叉核对与取舍裁决（t7）

- 核对人：cross-reviewer ｜ 日期：2026-11-01 ｜ 输入：`reports/collect/{doubao,deepseek,qwen}.md`（均 2026-11-01 采集）+ `reports/oss/{cdp,powershell,method}.md`（2026-11-01 快照）
- 判定口径：模型说法 < 生产级开源实现 < 官方协议/文档 < 本机实测。缺数据一律标「缺失」，不做补写。
- 本轮证据量（先摆明，防止下游误读）：**3 站点 × 3 题 = 9 格，有效回答 1 格**（豆包 Q1）。DeepSeek 仅剩 1 段置信 0.4 的片段；千问 0/3。
- 记号：`[豆包]` `[DS]` `[千问]` 为站点采集；`[实测]` 为采集轮内本机观察；`[OSS-xx]` 为开源报告章节。
- 采集侧证据的效力：`[千问]` 的观察来自 CDP 实测（回读 innerText 比对），属「本机实测」级；`[DS]` 的片段是其自身产出的残句，只作方向旁证，不作裁决依据。

---

## 1. 逐题对照表

### Q1 · CDP `Input.insertText` 与 `Input.dispatchKeyEvent` 的区别 / insertText 为何常插不进 contenteditable

| 站点 | 有效回答 | 核心结论（可核查断言，已压缩修辞） | 自述不确定点 |
|---|---|---|---|
| 豆包 | 1/1，全文 3598 字符未截断 | 见 §2 的 A1–A11，共 11 条可核查断言 | 未自述任何不确定；结尾反问「要不要给 PS 5.1 示例」，无置信标注 |
| DeepSeek | 缺失（原文被站点清空） | 仅 1 句残片：「insertText 向当前焦点插入文本，焦点不在编辑器内命令即失败」`[DS]` | 自述原文无法取回、置信 0.4 |
| 通义千问 | 缺失（0/1） | 无 | 无 |

### Q2 · PowerShell 5.1 执行 UTF-8 无 BOM + 中文的 .ps1

| 站点 | 状态 | 说明 |
|---|---|---|
| 豆包 | 未获取 | 回答后 `textarea` 从 DOM 消失，注入无效，未重试（`[豆包]` 2026-11-01） |
| DeepSeek | 未获取 | 会话丢失后不再提问（`[DS]` 2026-11-01） |
| 通义千问 | 未获取 | 站点故障期，按不重复触发风控停手（`[千问]` 2026-11-01） |

**结论：模型侧 0/3，本题不存在任何模型交叉证据。**

### Q3 · 「三家异构模型交叉验证」的最大失效场景

| 站点 | 状态 | 说明 |
|---|---|---|
| 豆包 | 未获取 | 同 Q2（`[豆包]`） |
| DeepSeek | 未获取 | 同 Q2（`[DS]`） |
| 通义千问 | 未获取 | 同 Q2（`[千问]`） |

**结论：模型侧 0/3。**

---

## 2. 判定：豆包 Q1 的 11 条断言逐条裁决

### 2.1 标签计数（按任务要求的三档）

| 档位 | 条数 | 说明 |
|---|---|---|
| 三源共识 | **0** | 千问 0 答、DeepSeek 仅 1 片段，三家中从未有两家以上对同一断言独立表态 → 结构上不可能构成 |
| 两正一反 | **1**（A9），另 1 条部分（A6） | 「反」方是豆包模型，「正」方是本机实测 + 开源生产实现；不是三家模型互搏 |
| 三源冲突 | **0** | 无三家同时表态的断言 |

### 2.2 逐条裁决

| ID | 断言（豆包原文压缩） | 佐证 | 反证 | 判定 | 置信 |
|---|---|---|---|---|---|
| A1 | insertText 不派发 keydown/keypress/keyup | `[OSS-cdp]` §二.3：只产生 `beforeinput`/`input`(`inputType=insertText`)，无 keydown/compositionend | 无 | **成立**（源码级证据，非本机实测） | 0.85 |
| A2 | insertText 触发 beforeinput+input，页面 `preventDefault` 可使其无效 | `[OSS-cdp]` §二.3 同源 | 无 | **成立** | 0.80 |
| A3 | insertText 要求焦点在编辑宿主（activeElement 即该宿主） | `[DS]` 残片同向；`[OSS-cdp]` §三.1 playwright#39492（未聚焦时 `fill` 变追加，根因是 selectAll 依赖焦点） | 无 | **成立**；三家口径里唯一「豆包 + DeepSeek + 开源」同向的点，但 DeepSeek 仅 0.4 片段，**不升级为三源共识** | 0.85 |
| A4 | insertText 遵循选区，选中文本时替换；无有效选区可能无效；建议用 `execCommand`/`setRangeText` | `[OSS-cdp]` §二.2：playwright 用 `Range.selectNodeContents` + `selection.removeAllRanges/addRange` 再插入 | 手段：`execCommand` 属受限/废弃 API，开源路线更稳 | **结论成立，手段作废**：采 Range+Selection，不采 execCommand | 0.80 |
| A5 | insertText 不能发控制键（Backspace/Enter/方向/Ctrl+C） | `[实测-千问]` 清空编辑器用的是 `dispatchKeyEvent` Backspace(vkey 8)，insertText 做不到 | 无 | **成立** | 0.90 |
| A6 | insertText 对中文/emoji 天然支持，且「和粘贴、输入法上屏走同一条路」 | `[实测-千问]` 133 字中文一次写入，回读逐字相等（`match=True`）→ 中文可插入成立 | 协议定义 + `[OSS-cdp]` §三.2（puppeteer#4981）：insertText **不产生** `compositionstart/end`，「与输入法上屏同一条路」不成立 | **部分成立**：中文可一次性插入 ✅；等价于 IME 合成 ❌ | 0.85 / 0.75 |
| A7 | dispatchKeyEvent 派发完整 keydown/keypress/keyup，可发控制键与组合键 | `[OSS-cdp]` §二.1 puppeteer `Input.ts:69-81` 的 `press(char)` 路径 | 边界：`[OSS-cdp]` §三.2 #27130——`type()` 对需修饰键的字符并不真按键 | **成立**（限「控制键/单字符」，组合键语义需自测） | 0.80 |
| A8 | dispatchKeyEvent 不自动处理 IME；逐字符发 a,b,c 不保证出现 abc | `[OSS-cdp]` §二.1：非键盘字符走 `sendCharacter`→insertText，正是绕开逐字符的原因 | 无 | **成立** | 0.85 |
| A9 | 富文本（Slate/ProseMirror/Draft/Quill）「只监听 keydown、自己改文档」，是 insertText 失效的**最高发**原因 | 千问站点确为 Slate（`div[data-slate-editor][contenteditable=true]`）`[实测-千问]`；且 `[OSS-cdp]` §二.1 主流生产实现（puppeteer/playwright）对非键盘字符一律整段走 insertText | **直接反例**：同一 Slate 上 insertText 一次写入 133 字成功 `[实测-千问]`；`[OSS-cdp]` §三.2 指出真实机制是 composition 状态机吞输入，而非「只监听 keydown」 | **不成立（作为「最高发原因」的排序）**：本轮唯一真冲突，见 §2.3 | 0.75 |
| A10 | PS 5.1 必须顺序 await，乱序发 CDP 会「返回成功但页面行为错乱」 | 无（本机 `cdp.ps1`/`ai-ask.ps1` 按 `id` 匹配串行等待，多轮实跑未见此现象） | 无 | **不可用**：无正例无实验，不作为结论 | — |
| A11 | Edge 与 Chrome 对 contenteditable 的 insertText 有细微差异；不带 `--disable-features=AutomationControlled` 会被页面检测干预输入 | 无 | 无（未做 Edge/Chrome A/B，亦无 OSS 出处） | **不可用** | — |

### 2.3 冲突点（A9）的分歧结构

- 豆包主张：编辑器框架拦截 → 应按「概率排序」用 dispatchKeyEvent 逐字模拟或调编辑器 API。
- 千问实测：同框架（Slate）、同接口（`Input.insertText`）、133 字一次成功、回读一致。
- 开源：生产库对中文这类非键盘字符**只能**走 insertText（逐字符对中文零收益，每字一次 CDP 往返），并给出失效的真实机制（composition / 焦点 / 选区），不是「只监听 keydown」。
- 裁决：采「焦点 + 选区 + composition」三因，弃「框架只认键盘事件」这一泛化归因。留边界：只反证了 Slate 一例，ProseMirror/Quill/Draft 未测。
- 本机可直接验的最小复现：见 §4 的 T1（挂事件监听后插入中文，看是否出现 `composition*`/`keydown`）。

### 2.4 采集轮内新增、模型未覆盖的实测事实（比模型说法更硬）

| ID | 事实 | 来源 | 对模型说法的修正 |
|---|---|---|---|
| L1 | 豆包输入框是**受控 `<textarea>`**（`data-testid=chat_input_input`）：insertText 能写入且发送按钮变可用，但 Enter 与点击发送均不提交；生效组合 = 原生 `HTMLTextAreaElement.prototype.value` setter + `input` 事件 + Enter | `[实测-豆包]` 2026-11-01 15:26 | 豆包答案只讲「写入」，未讲**受控组件状态同步**；写入成功 ≠ 可提交 |
| L2 | 千问输入框是 Slate contenteditable，insertText 整段可用 | `[实测-千问]` 15:18–15:33 | 推翻 A9 的「最高发」归因 |
| L3 | 清空编辑器：`Range.selectNodeContents` + `dispatchKeyEvent` Backspace 有效 | `[实测-千问]` | 支持 A5、A4 的 Range 路线 |
| L4 | 发送：`button[aria-label="发送消息"].click()` 有效；Enter **未验证成功**；生成态按钮变 `停止回答` 致选择器返回 null | `[实测-千问]` | 与 `[OSS-cdp]` §四「按钮发送时点按钮更稳」一致；豆包未提此路径 |
| L5 | DeepSeek 站点在回答后把会话切到新 ID，旧 URL 正文不保留 | `[DS]` 2026-11-01 | 采集可行性事实，非 Q1 技术结论 |

---

## 3. 取舍：每题的「可用结论 + 置信度 + 未覆盖边界」

### Q1（唯一有模型素材的题）

**不采用豆包整段答案，只采用以下 4 条，且每条都由实测或开源支撑：**

1. 中文提问走 `Input.insertText` 整段一次，不逐字符（`[实测-千问]` 133 字一致 + `[OSS-cdp]` §二.1/§四）。**置信 0.90**
2. 写入前必须显式聚焦编辑宿主 + `Range.selectNodeContents` 清场；写入后回读 `innerText` 断言（`[OSS-cdp]` §二.2/§三.1）。**置信 0.85**
3. 受控 `<textarea>` 站点（豆包）必须用原生 value setter + 派发 `input` 事件让框架状态同步，否则写入成功也不可提交（`[实测-豆包]` L1）。**置信 0.90**
4. 发送优先点按钮，Enter 作兜底；生成态用 `停止回答` 按钮存在性判活，不用 `body.innerText.length`（`[实测-千问]` L4 + `[OSS-cdp]` §四）。**置信 0.85**

**未覆盖边界**：ProseMirror/Quill/Draft 未测；真 IME（`imeSetComposition`/`imeCommitComposition`）未实测；Edge vs Chrome 差异（A11）未测；CDP 乱序响应（A10）无实验；千问 Enter 发送是否可用无定论；A1/A2/A3/A5/A7/A8 均为「源码/协议级成立但本机未跑对照实验」，故不标 ≥0.9。

### Q2

- **模型侧：不可用（0/3，无任何模型证据）。**
- 可用结论只能来自本机实测，且必须标注「非模型产出」：本机 PS 5.1.26100.9711 / 代码页 936 下，UTF-8 无 BOM 的 .ps1 含中文 → 偶数汉字乱码可跑（exit 0）、**奇数汉字吞掉后继字符**，表现为引号未闭合/缺大括号/`PathNotFound`，最坏是**静默改坏路径**（exit 0 但 `'C:\新建文件夹\a.txt'` 变成 `C:\鏂板缓鏂囦欢澶筡a.txt`）。规避：存 UTF-8 **with BOM**，或脚本纯 ASCII + 中文外置 `Get-Content -Encoding UTF8`。**置信 0.90**（可复现命令见 `[OSS-powershell]` §4）。
- **未覆盖边界**：只在本机 GBK/936 + 5.1 成立；Pwsh 7 路径未实测（`[OSS-powershell]` §5 自标 0.6）；本结论**无模型交叉验证**，不得在任何报表里计为「三源一致」。

### Q3

- **模型侧：不可用（0/3）。**
- 可用结论来自开源/论文证据 + 本轮一手反例，分两档：
  1. 方法论（`[OSS-method]` §2，arXiv 2502.08788 / 2311.17371，2026-11-01 快照）：辩论类多智能体常打不过 CoT/self-consistency 且成本更高；**异构**是唯一稳定增益；自洽性只对有唯一答案的题有效。**置信 0.70**（论文未复现，只读摘要）。
  2. 本轮一手反例（**置信 0.95**，证据即本仓库 3 份采集文件）：最大失效场景不是「三家一起答错」，而是**可用性不对称导致的伪共识**——本轮 9 格只有 1 格有效，若跳过可用性校验直接聚合，会把单源当共识。**采集侧必须先做「源可用性门禁」，三源 <2 家有效即禁止输出「一致」。**
- **未覆盖边界**：未取得「三家同时错同一处」的真实案例（只有 `[OSS-method]` §6 的理论证伪路径）；未做匿名互评（P1）实验。

### 给下游 t8 的硬约束

- 本轮无任何「三源共识」，skill/报表**不得**出现「三源一致率」类统计；只允许输出「有效源数 / 缺失源 + 缺失原因」。
- 任何来自模型的说法，若在 §2.2 被判「不可用」或「部分成立」，不得进入 skill 的规范正文。

---

## 4. 待实测清单（必须本地实测才能定的点）

| # | 待定问题 | 最小复现命令 | 判定标准 | 成本/前置 |
|---|---|---|---|---|
| T1 | insertText 到底触发哪些事件？是否产生 `composition*`？（裁决 A1/A6/A9） | ①`powershell -File scripts\cdp.ps1 eval -Match qianwen -Expr "window.__ev=[];['keydown','keypress','beforeinput','input','compositionstart','compositionend'].forEach(function(t){document.addEventListener(t,function(e){__ev.push(t+'\|'+(e.inputType\|\|''))},true)});'hooked'"` ②`powershell -File scripts\ai-ask.ps1 -Site qianwen -Question "1+1=?" -WaitSec 30` ③`powershell -File scripts\cdp.ps1 eval -Match qianwen -Expr "window.__ev.join(',')"` | 出现 `beforeinput\|insertText`/`input\|insertText` 而**无** `composition*` → A1 成立、A6「同一条路」不成立；若出现 `compositionstart` → 反过来推翻 A1 | 1 次站点提问 |
| T2 | 豆包受控 textarea：写入与提交能否分离观测？（裁决 L1） | 需先给 `ai-ask.ps1` 加 `-Mode insert\|send` 与 `-NoSubmit` 开关；之后：写入 → 读 `location.href` 是否变化 → 再单独发 Enter → 再读 `location.href` | 单发 Enter 后 URL 不变 = L1 成立（必须走 value setter + input 事件） | **工具前置**：现有脚本无此开关，属 t8 改动范围 |
| T3 | 千问就绪判定能否改用 `停止回答` 按钮 + 首字超时（裁决完成信号） | `powershell -File scripts\cdp.ps1 eval -Match qianwen -Expr "JSON.stringify({stop:!!document.querySelector('button[aria-label=\"停止回答\"]'),send:!!document.querySelector('button[aria-label=\"发送消息\"]'),ans:(function(){var w=document.querySelector('.chat-room-outer-wrap');var l=w&&w.lastElementChild;return l?l.innerText.length:-1})()})"` | 生成态 `stop=true`；完成态 `stop=false` 且 `ans>0`；故障态 `stop=true` 且 `ans=0` → 需 ≤15s 无首字即判失败 | 0 次提问（只读） |
| T4 | UTF-8 无 BOM + 中文的真实行为（Q2 唯一结论的复验） | 照抄 `[OSS-powershell]` §4：`$e=New-Object Text.UTF8Encoding($false); [IO.File]::WriteAllText("$env:TEMP\t.ps1","`$s='中'`r`nWrite-Output `$s",$e); powershell.exe -NoProfile -ExecutionPolicy Bypass -File "$env:TEMP\t.ps1"`；对照组把 `$false` 改 `$true` | 无 BOM 组 exit 1（`The string is missing the terminator`）；有 BOM 组 exit 0 输出 `中` | 0 次站点请求，~1 s |
| T5 | 三源可用性门禁（Q3 结论落地） | `powershell -File scripts\ai-ask.ps1 -List` + 对每站跑 T3 的多路存在性检查（编辑器节点 / 发送按钮 / 上轮回答气泡长度） | 有效源 <2 → 报表输出「单源，不构成共识」；=3 才允许三档标签 | 0 次提问（只读） |
| T6 | Edge vs Chrome 的 insertText 差异、`AutomationControlled` 是否干预（裁决 A11） | 同一页面分别在调试 Edge 与调试 Chrome 上跑 T1 的事件日志，比对 | 事件序列一致 → A11 不成立，从 skill 中删除 | **暂缓**：需第二个调试实例；A11 当前判「不可用」，不阻塞 |
