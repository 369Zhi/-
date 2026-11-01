# 开源侧核对 + skill 候选清单（t8）

- 核对人：oss-reviewer ｜ 复核时间：2026-11-01 15:40–16:05（Asia/Shanghai）｜ 账号：gh `369Zhi`
- 对象：`reports/oss/{cdp,powershell,method}.md`（t9/t10/t11）的全部候选与引用
- 证据分级：本机实测 > 官方协议/文档 > 上游源码与 issue > 模型说法。star/LICENSE/pushed_at 为 **2026-11-01 快照**，会变
- 方法：`gh api repos/OWNER/REPO` 取 `stargazers_count / license.spdx_id / pushed_at / archived` → 不合格直接剔除；再 `contents` 抓源码原文比对行号；issue 走 API 取 `title/state/created_at/closed_at`
- 门槛（AGENTS.md G2）：近 6–12 个月有 push、LICENSE 明确、非 archived

## 0. 结论先行

1. **主证据全部站得住**：puppeteer `Input.ts:169-194`、playwright `injectedScript.ts:939-970` 与 `dom.ts:625-649`、rod `element.go:261-280`、playwright#39492、puppeteer#4981、PowerShell #26642/#25698、arXiv 2502.08788 —— 逐条抓原文复核，语义一致（3 处日期/编号需修，见 §2）。
2. **两份候选被门槛淘汰**：`togethercomputer/MoA`（2025-01-07）、`potsawee/selfcheckgpt`（2024-06-26）都超过 12 个月无提交，method.md 的候选表需降级为「仅方法论引用」。
3. **豆包「受控 textarea + dialogMask」与 cdp.md「click 聚焦 + Range + insertText」不冲突**——它们分属不同输入宿主。开源实现本身就把 input/textarea 与 contenteditable 分两条路（playwright `injectedScript.ts:943-955` 对前两者用 `select()`，只有 contenteditable 走 Range）。cdp.md 的「三站点统一」才是要改的表述（§4）。
4. **模型侧只有豆包 Q1 一个有效源**：A1/A2/A3/A5/A7/A8 与开源侧一致；A4 结论成立但手段（execCommand）作废；**A9 与开源 + 实测冲突**；A6 半对半错；A10/A11 无佐证。Q2/Q3 模型 0/3，开源侧只有单源佐证，**不构成任何「三源投票」**。

## 1. 候选表（21 个仓库全部复核）

### 1.1 可采纳（借鉴思路 / 协议原文，不引入依赖）

| 仓库全名 | ★ | 最近 push | LICENSE | archived | 一句话理由 |
|---|---|---|---|---|---|
| [puppeteer/puppeteer](https://github.com/puppeteer/puppeteer) | 95,776 | 2026-11-01 | Apache-2.0 | false | [Input.ts:169-194](https://github.com/puppeteer/puppeteer/blob/main/packages/puppeteer-core/src/cdp/Input.ts) 原文已核：键盘表内→`press`，否则→`sendCharacter`→`Input.insertText` |
| [microsoft/playwright](https://github.com/microsoft/playwright) | 99,007 | 2026-10-31 | Apache-2.0 | false | [selectText](https://github.com/microsoft/playwright/blob/main/packages/injected/src/injectedScript.ts#L939-L970) 上溯编辑宿主再 focus + Range；[fill](https://github.com/microsoft/playwright/blob/main/packages/playwright-core/src/server/dom.ts#L625-L649) 仅 `needsinput` 走 insertText |
| [cyrus-and/chrome-remote-interface](https://github.com/cyrus-and/chrome-remote-interface) | 4,558 | 2026-02-09 | MIT | false | 只借 `CDP({host,port})` 接管已开实例的思路；9 个月未提交，不适合作依赖 |
| [ChromeDevTools/devtools-protocol](https://github.com/ChromeDevTools/devtools-protocol) | 1,585 | 2026-10-31 | BSD-3-Clause | false | `browser_protocol.json` **version 1.3** 本地下载比对：`Input.insertText` = "emulates inserting text that doesn't come from a key press, for example an emoji keyboard or an IME"，`experimental: true` |
| [PowerShell/PowerShell](https://github.com/PowerShell/PowerShell) #26642 / #25698 | 55,982 | 2026-10-31 | MIT | false | #26642 closed（2026-01-07，policy bot 标 answered）；#25698 open（updated 2026-09-20，23 评论） |

### 1.2 观望（思路可用，不入依赖）

| 仓库全名 | ★ | 最近 push | LICENSE | archived | 一句话理由 |
|---|---|---|---|---|---|
| [go-rod/rod](https://github.com/go-rod/rod) | 7,143 | 2026-08-11 | MIT | false | [element.go:261-280](https://github.com/go-rod/rod/blob/main/element.go#L261-L280) 顺序范本成立；Go 库，本机不用 |
| [ClericPy/ichrome](https://github.com/ClericPy/ichrome) | 256 | 2026-04-29 | MIT | false | 接管已开 Chrome 同思路；256★ 维护面小 |
| [promptfoo/promptfoo](https://github.com/promptfoo/promptfoo) | 26,684 | 2026-11-01 | MIT | false | 矩阵报表形态可借；Node 依赖，先有 P0 再谈 |
| [zeyuji/LLM-PeerReview](https://github.com/zeyuji/LLM-PeerReview) | 33 | 2026-09-11 | MIT | false | 抗位置偏见的打分设计；33★，不引代码 |
| [naudio/NAudio](https://github.com/naudio/NAudio) | 6,259 | 2026-10-25 | MIT | false | 本机 `voice.ps1` 已够用 |
| [MicrosoftDocs/PowerShell-Docs](https://github.com/MicrosoftDocs/PowerShell-Docs) | 2,577 | 2026-10-28 | **NOASSERTION** | false | 5.1 编码文档原句可引用，**不可整段复制** |

### 1.3 排除（不合格，写明原因）

| 仓库全名 | ★ | 最近 push | LICENSE | archived | 排除原因 |
|---|---|---|---|---|---|
| [karpathy/llm-council](https://github.com/karpathy/llm-council) | 26,116 | 2025-11-22 | **NONE** | false | 无 LICENSE = 保留所有权利；**流程可借、代码不可碰** |
| [togethercomputer/MoA](https://github.com/togethercomputer/MoA) | 2,993 | **2025-01-07** | Apache-2.0 | false | 22 个月无提交，不满足 6–12 月门槛 |
| [potsawee/selfcheckgpt](https://github.com/potsawee/selfcheckgpt) | 635 | **2024-06-26** | MIT | false | 28 个月无提交，同上门槛 |
| [composable-models/llm_multiagent_debate](https://github.com/composable-models/llm_multiagent_debate) | 557 | 2025-04-24 | **NONE** | false | 无 LICENSE + 19 个月无提交 |
| [Skytliang/Multi-Agents-Debate](https://github.com/Skytliang/Multi-Agents-Debate) | 619 | 2025-12-16 | **GPL-3.0** | false | 许可证红线 |
| [databricks/judges](https://github.com/databricks/judges) | 340 | 2025-07-31 | Apache-2.0 | **true** | 已归档 |
| [cdpdriver/zendriver](https://github.com/cdpdriver/zendriver) | 1,466 | 2026-10-02 | **AGPL-3.0** | false | 许可证红线，源码未读、未抄 |
| [vmoranv/jshookmcp](https://github.com/vmoranv/jshookmcp) | 2,094 | 2026-11-01 | **AGPL-3.0** | false | 同上 |
| [ProseMirror/prosemirror](https://github.com/ProseMirror/prosemirror) | 8,694 | 2026-04-01 | MIT | **true** | 已归档 |
| [browserless/browserless](https://github.com/browserless/browserless) | 13,880 | 2026-10-30 | **NOASSERTION** | false | 许可不明 |

## 2. 引用逐条复核（13 条，3 处需修）

| 引用 | 复核结果 | 处置 |
|---|---|---|
| puppeteer `Input.ts:169-171 / 177-194` | 原文一致（`sendCharacter`=insertText；`type()` 逐字符分两路） | 保留 |
| playwright `injectedScript.ts:939-970` | 原文一致；input 分支 943-948、textarea 949-955 才用 `select()` | 保留并补分支说明 |
| playwright `dom.ts:625-649` | 原文一致（`needsinput`→`insertText`，空值→`press('Delete')`） | 保留 |
| rod `element.go:261-280` | 原文一致（Focus→WaitEnabled→WaitWritable→InsertText→补 InputEvent） | 保留 |
| playwright#39492 | 标题一致；**`closed_at=2026-03-03`**，2026-08-03 是 updated_at | **改日期** |
| puppeteer#4981 | open，created 2019-09-26、updated 2026-03-05、11 评论、标题「Composition Events」 | 保留 |
| #27130 | 实为 **microsoft/playwright#27130**（closed 2023-09-15，标题「type() … does not actually press keys」）；`puppeteer#27130` = 404 | **补仓库前缀** |
| playwright#42125 | closed 2026-08-05，标题「fill() 可写进另一个元素」一致 | 保留 |
| `Input.insertText` 协议原文 | version 1.3 原文一致，`experimental: true` | 保留 |
| `imeSetComposition + imeCommitComposition` | 现行 json 中只有 `imeSetComposition`；`imeCommitComposition` 仅出现在它的描述里，**无同名方法** | **判「不可依赖」**，走 IME 只能自己试 setComposition 后如何提交 |
| PowerShell#26642 | closed 2026-01-07（answered）；mklement0 的解释在 2025-12-25 UTC / 12-26 本地 | 保留，建议标双时区 |
| PowerShell#25698 | open，23 评论，updated 2026-09-20 | 保留 |
| arXiv [2502.08788](https://arxiv.org/abs/2502.08788) | 标题「Stop Overvaluing Multi-Agent Debate」；摘要逐句一致（5 MAD × 9 benchmark × 4 模型；异构是 universal antidote） | 保留 |

## 3. 与模型答案的一致 / 冲突（只对豆包 Q1 有素材）

| 豆包断言 | 开源侧证据 | 判定 |
|---|---|---|
| A1 insertText 不派发 keydown/keypress/keyup | 协议方法只有 `text` 参数，结构上无法产生键事件；puppeteer 对非键盘字符单走 insertText | 一致 |
| A2 触发 beforeinput/input，`preventDefault` 可拦截 | #4981（composition 无官方支持）+ 协议语义 | 一致（「只发这两个事件」属推论，未逐事件核对，置信 0.8） |
| A3 焦点必须在编辑宿主 | playwright 先上溯 `isContentEditable` 宿主再 `focus()`；#39492 根因是 selectAll 依赖焦点 | 一致 |
| A4 遵循选区、选中即替换；建议 execCommand/setRangeText | 开源路线是 `Range.selectNodeContents` + `removeAllRanges/addRange` | 结论一致、**手段作废** |
| A5 不能发控制键（Backspace/Enter/Ctrl+C） | 协议无 key/keyCode 参数 | 一致（结构级） |
| A6 中文/emoji 天然支持；「与粘贴、输入法上屏同一条路」 | 协议只说模拟「emoji keyboard or IME」的文本插入；#4981 明确无 composition 事件 | **半对半错**：文本插入 ✅ / IME 事件等价 ❌ |
| A9 框架只监听 keydown 是 insertText 失效最高发原因 | 无任何开源出处；playwright/puppeteer 对中文一律 insertText；千问 Slate 一次写入 133 字成功（实测） | **冲突，判不成立**（真因是焦点/选区/composition） |
| A10 PS 5.1 需顺序 await，乱序 CDP 会「返回成功但页面错乱」 | 无出处 | 不可用 |
| A11 Edge 与 Chrome 有 insertText 差异、AutomationControlled 会干预 | 无出处 | 不可用 |

- **Q2**：模型侧 0/3。开源侧单源佐证 = #26642（by-design 解释）+ #25698（$OutputEncoding 管道坑）+ 5.1 文档原句；与本机实测方向一致，**但这不是交叉验证**。
- **Q3**：模型侧 0/3。只有 2502.08788 / 2311.17371 的单源方法论（异构有增益、MAD 常打不过 CoT/self-consistency），**不能标成三源共识**。

## 4. 裁决：豆包受控 textarea vs cdp.md 的「click + Range + insertText」

| 输入宿主 | 写入配方 | 为什么另一条不适用 | 风险 |
|---|---|---|---|
| 受控 `<textarea>`（豆包） | 关/隐藏 `dialogMask` → 原生 `HTMLTextAreaElement.prototype.value` setter → 派发 `input` → 点发送或 Enter | textarea 无 Range 语义（playwright 自己走 `select()`）；insertText 只改 DOM，React 受控状态不同步 → 按钮变可用也提交不了 | 弹层吞点击 → 用 DOM `focus()` 代替坐标点击；textarea 会从 DOM 消失 → 每轮重查 |
| contenteditable（千问 Slate） | 聚焦编辑宿主 → `Range.selectNodeContents` 清场 → 整段一次 `insertText` → 回读 innerText | textarea 的原生 setter 对 contenteditable 无效 | 生成态按钮变「停止回答」，选择器返回 null |
| 未知（DeepSeek） | 先探测 `textarea` 还是 `[contenteditable]`，再按上两行分派 | — | 站点会切新会话清正文 → 每轮答完立刻落盘 |

**改法**：把 cdp.md §四的「三站点统一 click+Range+insertText」改成「**先探测输入宿主类型，再选配方**」；两条配方互补，不是二选一。

## 5. 开源侧 skill 候选清单

| # | 做法 | 来源 | 可独立重写？ | 置信 |
|---|---|---|---|---|
| 1 | 三源采集模板：问题单文件 + 每站一份 md + 观察四件套（截断/拒答/风控/页面异常） | 本轮 t4/t5/t6 + `skills/retrieval-drill` §四证据卡 | ✅ 自有流程 | 0.90 |
| 2 | 输入宿主分派表（contenteditable / input·textarea / 受控组件三条路） | playwright `injectedScript.ts:939-970`、`dom.ts:625-649` + 豆包实测 | ✅ 只借逻辑，代码自写（Apache-2.0 需保留声明） | 0.85 |
| 3 | 写入后回读断言 + 必要时补发 `input` 事件 | rod `element.go:277-279` + 千问 133 字回读一致 | ✅ | 0.85 |
| 4 | 完成信号判据：停止按钮存在性 + ≤15s 无首字超时 + 长度稳定兜底 | 千问实测 + rod `WaitEnabled/WaitWritable` 顺序 | ✅ | 0.85 |
| 5 | 源可用性门禁：有效源 <2 就禁止输出「一致」，只报「有效源数/缺失原因」 | cross-check §3 Q3 一手反例 + 2502.08788 | ✅ 自有 | 0.90 |
| 6 | 遮罩处理：先隐藏 dialogMask，再用 DOM `focus()` 而非坐标点击 | 豆包实测；开源侧无对应（playwright 遇遮挡只会报拦截） | ✅ 需再实测 | 0.70 |
| 7 | PowerShell 编码模板：代码全 ASCII + 中文外置 payload + 脚本带 BOM + JSON 写无 BOM + `-ExecutionPolicy Bypass` | powershell.md 实测 + #26642/#25698 | ✅ 自有 | 0.90 |
| 8 | 断言级对比表：`实体\|属性\|值\|单位` + 数值容差/编辑距离 + 三色标记 | method.md P0 + promptfoo 矩阵形态（MIT，不引入） | ✅ | 0.75 |
| 9 | 引用核验卡：每次先 `gh api` 四字段，issue 必须带 `owner/repo#n` | 本次复核发现的 3 处引用缺陷 | ✅ 自有 | 0.90 |

## 6. 风险与红线

- **许可证**：GPL-3.0（Skytliang）、AGPL-3.0（zendriver、jshookmcp）与无 LICENSE（llm-council、composable-models）一律只读不抄、不进仓库；NOASSERTION（PowerShell-Docs、browserless）只引用不复制。
- **依赖**：puppeteer/playwright 都要自建 profile，与「复用已登录 Edge」直接冲突——本机结论维持「attach 现有调试实例，不引依赖」。
- **时效**：star 高 ≠ 活跃。cyrus-and/chrome-remote-interface 已 9 个月无提交；MoA/selfcheckgpt 已超 12 个月。skill 若内嵌推荐列表，必须带复核日期。
- **工具坑（本机实测）**：gh 2.102.1 的 `--jq '(... // "NONE")'` 报 `function not defined: NONE/0`（gojq 兼容）→ 改裸 JSON + `ConvertFrom-Json`；`gh` 不在 PS PATH，真实路径 `%LOCALAPPDATA%\copilot-desktop-gh-2.102.1\gh.exe`；`scripts\gh-net.ps1 check` 的 `proxy443 FAIL` 是**假阴性**（hosts 直连可用，API 实测 200）。
- **语义风险**：A9 若写进 skill，会诱导对 Slate 站点改用逐字符 `dispatchKeyEvent`（每字一次往返、仍可能被 composition 吞）——必须在规范正文里删除该归因。

## 7. 未做 / 可证伪

- 未做：真 IME 实测（现行协议无配套 commit 方法）；Edge vs Chrome A/B；ProseMirror/Quill/Draft 站点未测；未运行 puppeteer/playwright 本体（只读源码）。
- 可证伪：重跑 `gh api repos/<repo>`，四字段任一与表不符即本表过期；打开所引行号若内容不同即引用漂移；若豆包**不经 value setter**、纯 insertText 后也能提交 → §4 的 L1 结论错。
