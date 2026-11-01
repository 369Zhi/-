# 三源对照 · 最终交付

- 生成：2026-11-01 ｜ Lead 收口 ｜ 团队 `ai-multisource`（8 成员 / 8 任务，全部终止）
- 输入：`reports/collect/*.md`（3 份采集）、`reports/oss/*.md`（3 份开源检索）、`reports/review/cross-check.md`、`reports/review/oss-check.md`
- 证据分级：本机实测 > 官方协议/文档 > 上游源码与 issue > 模型说法

---

## 0. 一句话结论

**本轮没有形成任何「三源共识」—— 9 格数据只有 1 格有效（豆包 Q1）。** 但这轮产出的真东西不是模型的答案，而是把「网页版模型为什么采不到」和「CDP 输入到底该怎么写」变成了可执行代码与判定表。

| 指标 | 数值 |
| --- | --- |
| 计划采集 | 3 站 × 3 题 = 9 格 |
| 有效回答 | **1 格**（豆包 Q1，3598 字符） |
| 部分/残片 | 1 格（DeepSeek Q1，站点清了会话，仅剩片段，置信 0.4） |
| 完全缺失 | 7 格（千问站点故障 0/3；豆包 Q2/Q3；DeepSeek Q2/Q3） |
| 开源侧交付 | 3 份（27 / 95 / 72 行），21 个仓库全量 `gh api` 复核 |
| 复核剔除 | 10 个（超期 2 个、archived 2 个、AGPL 2 个、GPL 1 个、无 LICENSE 2 个、NOASSERTION 2 个 – 部分重叠） |

**最大收获（反直觉）**：交叉验证最大的失效场景不是「三家一起答错」，而是**可用性不对称导致的伪共识** —— 如果跳过源可用性校验直接聚合，单源就会被写成「三源一致」。

---

## 1. Q1 · CDP 输入：最终可用结论

### 1.1 输入宿主分派（核心表，直接照用）

| 宿主类型 | 判定方法 | 写入配方 | 为什么另一种失效 |
| --- | --- | --- | --- |
| 受控 `<textarea>`（豆包） | `textarea[data-testid=chat_input_input]` 存在 | 关掉/隐藏 `dialogMask` → 原生 `HTMLTextAreaElement.prototype.value` setter → 派发 `input` → 点发送或 Enter | textarea 无 Range 语义；insertText 只改 DOM，React 受控状态不同步 → 按钮变可用也提交不了 |
| `contenteditable`（千问 Slate） | `div[data-slate-editor][contenteditable=true]` | 聚焦编辑宿主（上溯到最外层 contenteditable）→ `Range.selectNodeContents` 清场 → **整段一次** `Input.insertText` → 回读 `innerText` 断言 | 原生 value setter 对 contenteditable 无效 |
| 未知（DeepSeek 等） | 先探测 `textarea` / `[contenteditable]` 哪一个是可见的 | 按前两行分派 | 站点会在回答后切新会话、旧正文清空 → 每轮答完立刻落盘 |

### 1.2 四条经实测/开源双支撑的策略

| # | 策略 | 置信 | 依据 |
| --- | --- | --- | --- |
| 1 | 中文提问走 `Input.insertText` **整段一次**，不逐字符 | 0.90 | 千问实测：133 字一次写入、回读逐字相等；puppeteer 对非键盘字符也一律 `sendCharacter→insertText` |
| 2 | 写入前显式聚焦编辑宿主 + `Range.selectNodeContents` 清场，写入后回读 `innerText` | 0.85 | playwright `injectedScript.ts:939-970`；playwright#39492（未聚焦时 `fill` 变追加） |
| 3 | 受控 `<textarea>` 必须用原生 value setter + `input` 事件 | 0.90 | 豆包实测：insertText 能写入、按钮变可用，但 Enter/点击都不提交 |
| 4 | 发送优先点按钮，Enter 作兜底；生成态用「停止回答」按钮存在性判活，**不要用 `body.innerText.length`** | 0.85 | 千问实测：Slate 上 Enter 不提交；侧栏常驻 440–560 字导致长度判据失真 |

### 1.3 被推翻的模型说法（不得进 skill 正文）

| 豆包断言 | 判定 | 反证 |
| --- | --- | --- |
| 「框架只监听 keydown」是 insertText 失效的**最高发**原因 | ❌ 不成立 | 千问同为 Slate，`insertText` 一次写入 133 字、回读一致；真实机制是 **焦点 + 选区 + composition** 三因 |
| insertText 与「输入法上屏同一条路」 | ⚠️ 半对 | 中文能插入 ✅；但不产生 `compositionstart/end`，不等价于 IME 合成 |
| 用 `execCommand`/`setRangeText` 清场 | ⚠️ 结论对、手段作废 | 采 `Range.selectNodeContents + removeAllRanges/addRange` |
| 「PS 5.1 必须顺序 await，乱序 CDP 会页面错乱」 | ⛔ 无佐证 | 本机多轮串行 id 匹配实跑无此现象 |
| 「Edge 与 Chrome 有 insertText 差异 / AutomationControlled 干预」 | ⛔ 无佐证 | 未做 A/B，无出处 |

### 1.4 协议级事实（devtools-protocol v1.3 原文，本地核）

- `Input.insertText` 定义：*"emulates inserting text that doesn't come from a key press, for example an emoji keyboard or an IME"*，且 **`experimental: true`**。
- 现行协议**只有 `imeSetComposition`**，**没有 `imeCommitComposition` 方法**（后者只出现在描述里）→ 「两步提交 IME」不可依赖。

---

## 2. Q2 · PowerShell 5.1 编码：最终结论

本机环境：PS `5.1.26100.9711`、ANSI 代码页 `936`、`$OutputEncoding = us-ascii`。

| 情形 | 结果 |
| --- | --- |
| UTF-8 **无 BOM** + 中文，非 ASCII 字节数**偶数** | 只乱码，能跑（exit 0） |
| UTF-8 **无 BOM** + 中文，非 ASCII 字节数**奇数** | **吞掉后继字符**：吞 `'` → `TerminatorExpectedAtEndOfString`；吞 `}` → `Missing closing '}'`；吞 `\` → **静默改坏路径**（exit 0！实测 `'C:\新建文件夹\a.txt'` 变成 `C:\鏂板缓鏂囦欢澶筡a.txt`） |
| UTF-8 **with BOM** | 正常 |

**规避（两条任选）**：
1. `.ps1` 存 **UTF-8 with BOM**；
2. 脚本**纯 ASCII**，中文放外部 `payload/*.txt`，用 `Get-Content -Encoding UTF8 -Raw` 读入。

**附带三条 5.1 默认坑**：`Get-Content` 默认按 ANSI 读（必须 `-Encoding UTF8`）；`Out-File -Encoding UTF8` 写出**带 BOM**（写 JSON 用 `[IO.File]::WriteAllText(path, text, UTF8Encoding($false))`）；`$OutputEncoding=us-ascii` 导致中文经管道进原生 exe 变 `????`（中文当命令行参数正常）。

**重要限定**：模型侧 0/3，本条**无模型交叉验证**，不得在报表里计为「三源一致」。置信 0.90（本机可复现）。

---

## 3. Q3 · 交叉验证方法论：最终结论

| 结论 | 置信 | 依据 |
| --- | --- | --- |
| 辩论类多智能体常打不过 CoT / self-consistency，且更贵；**模型异构才是有稳定增益的因素** | 0.70 | arXiv 2502.08788（标题《Stop Overvaluing Multi-Agent Debate》，5 MAD × 9 benchmark × 4 基座）只读摘要 |
| 交叉验证最大失效场景 = **可用性不对称导致的伪共识** | 0.95 | 本轮一手反例：9 格仅 1 格有效 |
| 源可用性门禁：**有效源 < 2 就禁止输出「一致」**，只报「有效源数 + 缺失原因」 | 0.90 | 同上 |

**不做**：同模型多轮自辩论（成本高、收益不稳）、整段文本余弦相似度当一致度、逆向网页接口绕验证码。

---

## 4. 候选与取舍（21 个仓库，全量 `gh api` 复核）

**可采纳（借思路，不引依赖）**：puppeteer（Apache-2.0）、playwright（Apache-2.0）、devtools-protocol（BSD-3-Clause，协议原文）、chrome-remote-interface（MIT，仅 attach 思路）、PowerShell #26642/#25698。

**观望**：go-rod（MIT，顺序范本）、ichrome（MIT）、promptfoo（MIT，矩阵报表形态）、LLM-PeerReview（MIT）、NAudio（本机 `voice.ps1` 已够用）、PowerShell-Docs（**NOASSERTION**，只引用不复制）。

**排除（写明原因）**：

| 仓库 | 原因 |
| --- | --- |
| karpathy/llm-council | **无 LICENSE**（保留所有权利），流程可借、代码不可碰 |
| composable-models/llm_multiagent_debate | 无 LICENSE + 19 个月无提交 |
| Skytliang/Multi-Agents-Debate | **GPL-3.0** 红线 |
| zendriver / jshookmcp | **AGPL-3.0** 红线 |
| databricks/judges / ProseMirror/prosemirror | **已 archived** |
| togethercomputer/MoA（2025-01-07）、selfcheckgpt（2024-06-26） | **超 12 个月无提交**，不满足 G2 |
| browserless | NOASSERTION，许可不明 |

**关键立场**：puppeteer/playwright 都要求自建 profile，与「复用已登录 Edge」直接冲突 → 本机继续维持「attach 现有调试实例，不引依赖」。

---

## 5. 落地物（本轮真正交付的代码与文档）

| 文件 | 作用 |
| --- | --- |
| `scripts/webask.ps1` | 单站提问脚本，内置三站适配器（textarea 受控组件 / contenteditable / 探测回退）+ 遮罩处理 + 回读校验 |
| `scripts/cdp.ps1` | 最小 CDP 客户端（list / nav / eval / close），无外部依赖 |
| `skills/tri-source-research/SKILL.md` | 三源对冲检索流程（含网页版接入、轮换规则、降级规则） |
| `skills/retrieval-drill/SKILL.md` | 信息检索训练手册（四步法、gh 限定符、止损五条、证据卡） |
| `scripts/gh-net.ps1` | GitHub 链路自检/修复（注意：`proxy443 FAIL` 是假阴性） |

---

## 6. 待实测清单（必须本机实测才能定）

| # | 问题 | 最小复现 | 判定 |
| --- | --- | --- | --- |
| T1 | insertText 到底触发哪些事件、是否产生 `composition*` | 挂监听 → 提问 → 读 `window.__ev` | 出现 `beforeinput/insertText` 且无 `composition*` → 「同一条路」不成立 |
| T2 | 豆包受控 textarea 写入与提交能否分离观测 | 写入后读 URL → 单独发 Enter → 再读 URL | URL 不变 → 必须走 value setter + input |
| T3 | 千问就绪判定改用「停止回答」按钮 + 15s 无首字超时 | 只读检查 stop/send 两个 aria-label | 故障态 `stop=true 且 ans=0` |
| T4 | 无 BOM 中文脚本行为复验 | 写两个 `.ps1`（BOM/无 BOM）各跑一次 | 无 BOM 组 exit 1 |
| T5 | 三源可用性门禁 | 每站检查编辑器节点/发送按钮/上一轮回答长度 | 有效源 < 2 → 只报单源 |
| T6 | Edge vs Chrome 的 insertText 差异 | 两个调试实例跑 T1 比对 | 事件序列一致 → 删除该说法 |

---

## 7. 风险与红线

1. **许可证**：GPL / AGPL / 无 LICENSE 一律只读不抄；NOASSERTION 只引用原文不复制。
2. **依赖冲突**：引入 puppeteer/playwright 会毁掉「复用已登录 profile」这一前提。
3. **时效**：star ≠ 活跃；本表所有 star/pushed 均为 2026-11-01 快照，复核命令 `gh api repos/OWNER/REPO`。
4. **工具坑**：gh 2.102.1 的 `--jq '(... // "NONE")'` 会报 `function not defined`，改裸 JSON + `ConvertFrom-Json`；`gh` 不在 PATH（真实路径 `%LOCALAPPDATA%\copilot-desktop-gh-2.102.1\gh.exe`）。
5. **语义风险**：把「框架只认 keydown」写进 skill 会诱导对 Slate 站点改逐字符输入（每字一次往返、仍可能被吞）—— 必须删除该归因。
6. **可证伪**：重跑 `scripts/webask.ps1 -Site doubao` 若纯 insertText 后也能提交，则 §1.1 的受控组件结论错。

---

## 8. 下一轮怎么更快（本轮教训）

| 教训 | 改法 |
| --- | --- |
| 采集 agent 一失败就整站空手 | 每题独立子任务，失败只影响该题 |
| 「一答完立刻写盘」没写进指令 | 写进任务描述，且禁止空内容覆盖 |
| 站点在回答后会清会话 | 采集完立即取文本，不依赖回访旧会话 |
| 模型侧 0/3 仍跑了 25 分钟 | 先做源可用性预检，不可用直接跳过 |
| 三源 vs 实际 | 现实做法：**1 个稳定模型源 + GitHub 开源 + 本地实测**，模型只当假设生成器 |
