# 开源检索：多模型交叉验证 / 集成方法（method）

- 检索时间：2026-11-01（star / 最近提交均为该日快照）
- 口径：`gh search repos` + `gh api repos/<repo>` 取 star / pushedAt / license，只读 README 与论文摘要；**不读源码、不读论文全文**
- 约束：2 轮检索、5 个候选仓库（README 之外只用来判定「值不值得抄」）
- 用途：给 t7（交叉核对）与 t8（skill 候选）提供方法论依据；不改动采集脚本

---

## 1. 候选表（5 个）

| # | 仓库 | Star | 最近提交 | LICENSE | 核心思路 | 适用规模 | 成本 | 失效场景 |
|---|------|------|----------|---------|----------|----------|------|----------|
| 1 | [karpathy/llm-council](https://github.com/karpathy/llm-council) | 26.1k | 2025-11-22（创建即当天，之后无更新） | **无（null）** | 三段式：①各模型独立作答 ②**匿名**互评排序 ③主席模型汇总成最终答 | 3–5 个模型、单轮决策题 | 3×N 次调用 + 汇总 | 主席模型偏见；无维护；作者自述「99% vibe coded，不做任何支持」 |
| 2 | [togethercomputer/MoA](https://github.com/togethercomputer/MoA) | 3.0k | 2025-01-07 | Apache-2.0 | 分层聚合：每层多模型互看前层输出，末层聚合器出终答（`moa.py` 50 行可读） | 需要 ≥3 层才有收益 | 层数×模型数（3 层×3 模型≈9 次） | 单层≈普通投票；层数越多越慢越贵 |
| 3 | [potsawee/selfcheckgpt](https://github.com/potsawee/selfcheckgpt) | 635 | 2024-06-26 | MIT | 同模型多次采样，句级打分（BERTScore / N-gram / MQAG / LLM 提示），高分=更可能编造 | 单模型、事实类长文本 | k 次采样（论文用 k≈5–20） | 开放式创作题无意义；需要 torch（CPU 可跑） |
| 4 | [promptfoo/promptfoo](https://github.com/promptfoo/promptfoo) | 26.7k | 2026-11-01（当天） | MIT | 声明式 YAML：多 provider × 多 prompt 矩阵跑测，自带断言、缓存、对比报表 | 任意规模，含 CI | 一次跑满矩阵，成本线性 | 需 Node 环境；要写 custom JS provider 才能接非 API 源 |
| 5 | [zeyuji/LLM-PeerReview](https://github.com/zeyuji/LLM-PeerReview) | 33 | 2026-09-11 | MIT | 同行评审式：互评打分（含 flipped-triple 抗位置偏见）→ 真值推断 → 选最优，**无监督**不需标准答案 | 3–8 个模型、有多个候选答案 | N×N 次互评 | 样本太少时推断不稳定；论文级代码，工程化程度一般 |

**没进表但看过的**：`composable-models/llm_multiagent_debate`（557★，**无 LICENSE**）、`Skytliang/Multi-Agents-Debate`（619★，**GPL-3.0**）、`databricks/judges`（340★，**已归档 archived=true**）——都列入「不推荐」，理由见第 4 节。

---

## 2. 关键结论（含可核查证据）

1. **对我们真正有用的不是「辩论」，是「模型异构」。** [arXiv 2502.08788](https://arxiv.org/abs/2502.08788)（5 种 MAD 方法 × 9 benchmark × 4 个基座模型）结论：MAD 常常打不过 CoT / self-consistency，且算力消耗显著更高；而**模型异构**是唯一稳定带来提升的因素。我们手头豆包 / DeepSeek / 通义千问正好是三家异构模型——这是本项目最大的方法论优势，别浪费在自己跟自己辩论上。
2. **辩论类方案性价比存疑。** [arXiv 2311.17371](https://arxiv.org/abs/2311.17371)：多智能体辩论「当前形态下不能稳定超越 self-consistency 与多推理路径集成」，且对超参极敏感、难调。
3. **自洽性（self-consistency）只对「有唯一正确答案」的题有效。** [arXiv 2203.11171](https://arxiv.org/abs/2203.11171)：GSM8K +17.9%、SVAMP +11.0%；机制是 k 条推理路径投票取多数。开放写作题没有可投票的等价类，投不出东西。
4. **匿名化互评是 llm-council 唯一值得抄的点**（README Stage 2：把回答去标识后让模型排序）。但它 **LICENSE 为空**，法律上是「保留所有权利」——**只能抄流程，一行代码都不能抄**。
5. **工程化的现成轮子存在，但现在不该上。** promptfoo 支持 custom JS provider（`callApi(prompt, context) => {output}`）与 HTTP provider，理论上可以把我们的 CDP 脚本包成 provider 拿到矩阵报表；代价是多一层 Node 依赖和调试面。先用手上已有的 PowerShell 跑通再考虑。

---

## 3. 适合本机的可行方案（无 GPU、仅网页版模型）

**约束先摆明**：每次网页提问 ≈ 10–60 s；每站一个标签页/一个登录态 → **并发上限 ≈ 3**；无 GPU 意味着一切打分要么是字符串级，要么是让模型自己判。
所以：**请求数就是成本**，方案按「3 次 / 6 次 / 9+ 次」分档，默认只用 P0。

### P0 · 三源对照 + 本地断言比对（3 次请求，必做）
1. 三家并行提问，**同一个模板**，强制输出结构化结果：
   `先给结论；再给 3–5 条「可核对断言」，每条一行，形如「实体|属性|值|单位」`。
2. 本地（PowerShell）做**断言级**比对，而不是整段文本相似度：
   - 先归一化（全角/半角、千分位、`万/亿`、日期格式、`m³/h` 之类单位）
   - 再对「实体+属性」做键匹配，值做**数值容差**或**字符串编辑距离**比对
   - 输出三色标记：`3/3 一致`（绿）｜`2/3 一致`（黄，标出少数派与原话）｜`1/1/1`（红，分歧）
3. 交付形态：一张对照表 + 每条的「一致度」+ 少数派原话摘录。**不做二次询问**。

> 为什么不是「让模型互评」：P0 阶段判据必须是**可机器复核**的。模型互评是主观分，出了争议无法仲裁，违反「证据分级：本地实测 > 官方文档 > 生产级开源实现 > 模型说法」。

### P1 · 匿名互评（+3 次请求，只在 P0 出现红色分歧时触发）
- 抄 llm-council Stage 2 的**流程**（不抄代码）：把三家回答**打乱顺序、去掉模型名**，分别问每家「按准确性与信息量排序，并指出哪一句最可疑」。
- 汇总成「被指可疑的句子清单」→ 只把这些句子回填给人看，**不自动改答案**。
- 触发条件写死：P0 里存在红色分歧，且问题属于事实类（数值/日期/人名/法规条款）。创作类、观点类**不触发**。

### P2 · 单层聚合（+1 次请求，可选）
- 只做 MoA 的**一层**：把三家答案 + 分歧清单作为参考，让其中一个模型输出一份「合并稿」，并要求它**标注哪些点只来自单一来源**。
- 不做多层（README 的 `--rounds = 层数-1`，3 层直接 ×3 成本，网页版延迟下不可接受）。

### 传输层怎么接（不新增依赖）
- 已有 `scripts/cdp.ps1`（CDP `Runtime.evaluate` / `Page.navigate`）与 `scripts/ai-ask.ps1`（`-Site doubao|deepseek|qianwen` 的 `ask` 原语）已经够用，**不要引入 Selenium / Playwright**：要下驱动、要管版本、还多一层 Node 依赖，收益为零。
- 三家并行：3 个 `Start-Job` / `Start-ThreadJob` 各管一个 `-Site`，主进程收 JSON。**不要**在一个 WebSocket 上串行等 3 个慢回答。
- 结算方式沿用现有「文本长度连续 N 轮不变即视为答完」的启发式；上游若提供「停止生成」按钮消失的信号，优先用它（比长度稳定更可靠）。

### 成本对照（本机）
| 档位 | 请求数 | 预估墙钟 | 适用 |
|------|--------|----------|------|
| P0 | 3 | 1–3 min（并行） | 默认，日常三源对照 |
| P0+P1 | 6 | 3–6 min | 出现事实类红色分歧 |
| P0+P1+P2 | 7 | 4–8 min | 需要一份成稿时 |
| 自洽性 k=5×3 源 | 15 | 5–15 min | 只在「答案可以投票」的题上才划算 |

---

## 4. 不推荐的做法及原因

1. **不要抄 llm-council 的代码。** LICENSE 为 null（GitHub API `license: null`），法律语义是「保留所有权利」，code 不能复制；且作者明说不再维护。→ 只可借鉴「匿名互评 + 主席汇总」的**流程思路**。
2. **不要引入 `composable-models/llm_multiagent_debate`。** 同样是 **无 LICENSE**（ICML 2024 论文参考实现，README 自称 preliminary implementation），且它绑定 OpenAI 接口、按论文 benchmark 组织目录，跟我们的网页版链路完全不搭。
3. **不要碰 `Skytliang/Multi-Agents-Debate`（MAD）。** **GPL-3.0**，属于 AGENTS.md 划的红线；即便只看思路，其「天使/恶魔双方辩论」还需要同模型扮演对立角色，正是 arXiv 2502.08788 指出性价比最差的那一类。
4. **不要上 `databricks/judges`。** `archived=true`（2025-07-31 后停止），归档库不跟进。
5. **不要做「同模型多轮自我辩论 / 自我反思」。** 三重浪费：网页版一次请求 10–60 s，多轮直接把延迟推到十分钟级；同模型多轮的多样性远低于跨厂商；且 ID 里看不到的 DoT（thought degeneration）问题在 MAD 论文里被明确点出。
6. **不要做 k=5 以上的自洽性采样。** 3 源 × k=5 = 15 次网页请求，换来的只是「同一家模型内部的稳定性」，而不是「跨厂商的独立性」——我们已经有更便宜的异构信号了。
7. **不要逆向网页接口 / 绕验证码 / 无头批量刷。** 违反站点 ToS，账号封禁风险落在用户自己头上；且这类方案一改前端就全废。已有 CDP 复用真实登录态、低频率、人工可介入，风险可控得多。
8. **不要现在就接 promptfoo。** 它是好轮子（MIT、当天还在提交），但要先写 custom provider 包我们的脚本，属于「先有 P0 再谈集成」；现在上只是把调试面从 1 层变 3 层。
9. **不要用「整段文本余弦相似度」当一致度。** 网页版模型的行文风格差异极大，同一事实换个说法相似度就崩；必须下沉到**断言级**比对（第 3 节 P0 第 2 步）。

---

## 5. 「值不值得抄」结论（一句话版）

| 对象 | 抄什么 | 明确不抄什么 |
|------|--------|--------------|
| llm-council | 三段流程、**匿名化**互评、主席汇总 | 任何代码（无 LICENSE） |
| MoA | 分层聚合的**思想**；只取单层 | 多层循环（成本 ×层数） |
| selfcheckgpt | 「多次采样 + 句级一致度」的**度量视角** | torch/BERTScore 依赖（本机无 GPU，用字符串级替代） |
| promptfoo | 矩阵化对比 + 声明式配置的**报表形态** | 现在不引入 Node 链路 |
| LLM-PeerReview | 无监督选优 + 抗位置偏见（flipped-triple）的**打分设计** | 论文级代码结构 |

**最终落点**：本项目做「**异构三源 + 断言级比对 + 分歧才互评**」，不做「同模型辩论」。前者 3 次请求、可机器复核；后者 9+ 次请求、判据主观。

---

## 6. 可证伪：怎么验证这套结论是错的

- 若在**事实类**题上 P0 的 `3/3 一致` 出现明显错误（即三家同时错同一处）→ 说明异构一致性≈共同偏见，需要引入外部检索源，本方案降级。
- 若 P1 匿名互评在 10 个红色分歧样本上**一次都没能指出真正错误的那句** → 互评环节应删掉，改为人看。
- 若单次网页提问稳定 < 10 s → 成本模型失效，可重新评估 k=3 的自洽性采样。
- 复核命令（可随时重跑，star/LICENSE/活跃度会变）：
  ```bash
  gh api repos/karpathy/llm-council --jq '{stars:.stargazers_count,pushed:.pushed_at,license:.license.spdx_id,archived:.archived}'
  gh api repos/togethercomputer/MoA --jq '{stars:.stargazers_count,pushed:.pushed_at,license:.license.spdx_id}'
  gh api repos/Skytliang/Multi-Agents-Debate --jq '{license:.license.spdx_id}'
  gh api repos/databricks/judges --jq '{archived:.archived}'
  ```
