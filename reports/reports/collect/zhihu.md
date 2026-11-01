# 知乎采集：3 主题高赞实战回答

> **性质：经验源（工程踩坑），置信度上限 0.7。不得当作权威结论，仅供与开源实现 / 本地实测交叉参考。**
> 采集方式：已登录 Edge（CDP 端口 9222，profile `.edge-ai`）→ `scripts\cdp.ps1 nav/eval`，取 DOM `innerText`/`textContent`。
> 采集时间：2026-11-01 16:0x（Asia/Shanghai）。采集者：collector-zhihu（t12）。
> 口径：每主题 1 个问题页、前 3 条高赞回答；只摘可核查的技术断言（命令 / 报错 / 常量 / 版本 / 现象），丢弃情绪与营销。
> 搜索消耗 6 次（达上限）；失败记录见文末「采集偏差」。

---

## Z1 · CDP / 浏览器自动化实战踩坑

**问题页**：<https://www.zhihu.com/question/562879512>（「浏览器自动化工具 Selenium，Playwright，Puppeteer 做爬虫有哪些弊病？」关注 242 / 浏览 83,187）
> 说明：知乎搜索里不存在专门讨论 `insertText` 输入失败的问题页（见文末偏差）；该问题页是本主题下最高热度的落点，内容为 CDP 层自动化踩坑。

### A1 · 貘吃馍香 ｜ 赞同 555 ｜ 发布于 2024-04-16 02:35
链接：<https://www.zhihu.com/question/562879512/answer/3467303740>
- **WebGL 纹理上限**：全页截图存在 `MAX_TEXTURE_SIZE`（作者写作 `1<<14`）限制，超出可能**崩渲染器**；拉伸视口截图的高度也可能远小于该上限。
- **滚动对象判断**：页面滚动条可能属于某个 `div` 容器而非视口，直接设 `scroll` **可能无效**；替代做法是用 **CDP 合成事件拖拽**滚动（代价是慢）。
- **加载时机**：`onload` 时（前端渲染场景）页面信息**不完整**，不能用它判定采集完成。
- **URL 不是一跳到终点**：中间页可能加载后跳走，或倒计时 N 秒再跳，需另设「是否已到最终地址」判据。
- **对抗不止 UA**：还会识别视口高度/屏幕高度差异、ARM 移动内核、是否有 GPU、是否 Linux 服务器系统，甚至针对特定 webview 的 jsb 探针。
- **耗时量级**：代理不佳时单次抓取 20s–60s 乃至 >100s；调优后 2s–10s。
- 可核查性：`1<<14`=16384 与 GPU `MAX_TEXTURE_SIZE` 语义一致，**可核**；其余为经验陈述，**不可逐条核**。

### A2 · 老葛 ｜ 赞同 59 ｜ 发布于 2025-11-26 15:39
链接：<https://www.zhihu.com/question/562879512/answer/1977038711209349177>
- 唯一可核查断言：Playwright / Puppeteer 的取舍是「不用做逆向、但速度慢」，其自述方案为 **Playwright + Next.js + Prisma**，仓库 <https://github.com/g089h515r806/nextcrawler>。
- 其余为自述项目功能清单（readability 正文识别、定时任务、代理、模板导入导出）→ **营销成分 > 技术细节，弃用**。

### A3 · 许良 ｜ 赞同 35 ｜ 发布于 2025-07-30 17:35
链接：<https://www.zhihu.com/question/562879512/answer/1933943789292098224>
- 唯一可核查断言：**Chrome MCP Server 走原生 CDP 通信**（可访问事件生命周期、性能指标、WebSocket 流），**OpenDia 走浏览器插件注入 + JS 执行**（CDP 能力为封装层，抓包需扩展）。
- 其余为对比表与比喻排版 → **营销，弃用**。
- 备注：第 4 高赞「朱卫军」（赞同 21，2025-08-05）含检测规避要点，按「前 3 条」规则未纳入，如需可复核 answer/1936124020237010601。

---

## Z2 · Windows 中文乱码 / 编码踩坑

**问题页**：<https://www.zhihu.com/question/2069471280651310305>（「准大一软件工程新生，求大佬指点怎么解决终端里的中文乱码（vs code）？」）
> **偏差**：该问题页覆盖面为通用「终端/编译期编码」，**不含 PowerShell 5.1 奇数非 ASCII 字节吞字符、UTF-8 BOM 的具体断言**；ExecutionPolicy 方向的候选问题页 0 回答（见文末）。

### A1 · 小孩子娃娃 ｜ 赞同 1 ｜ 发布于 2026-08-09 13:45
- 处理手法：在输出中文前执行 `system("chcp 65001 > nul");`（切控制台代码页为 UTF-8，`> nul` 隐藏命令自身输出）。
- 归因：GCC(MinGW/Linux) 默认执行字符集 UTF-8，MSVC 默认 GBK；Windows 终端默认 GBK → 解码不匹配即乱码（ASCII 因最小字符集要求永远不乱）。
- 备选（作者不推荐）：① 改系统区域设置（全局代码页，可能影响其它软件）；② 改 IDE 配置（有时不生效，需禁用伪终端）；③ 编译期 `gcc -c -fexec-charset=GBK main.c -o gbk.o`，或 CMake `set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fexec-charset=GBK")`；④ `SetConsoleOutputCP(CP_UTF8); SetConsoleCP(CP_UTF8);`（需 `<windows.h>`）。
- 代价自述：`system()` 会起子进程，频繁调用影响性能。可核查性：命令与参数**可核**。

### A2 · 云端GPT ｜ 赞同 2 ｜ 发布于 2026-08-08 22:58
- `chcp 65001` → 输出 `Active code page: 65001`；VS Code 设置项 `terminal.integrated.defaultEncoding` = `UTF-8`。
- Python：`import sys; print(sys.stdout.encoding)` 查实际输出编码；C/C++：源文件保存 UTF-8。
- Linux/macOS：`locale` 检查 `LANG=*.UTF-8`。可核查性：设置项名与命令**可核**；无版本号。

### A3 · 云开霁散 ｜ 赞同 1 ｜ 发布于 2026-08-12 20:35
- 用 `L""` 宽字符串字面量 + `WideCharToMultiByte` 转换后 `WriteConsole` / `wprintf("%ls", ...)`；**作者自注限制：不能处理代理对，不适用于 ≥U+10000 的字符**。
- 可核查性：API 名与限制**可核**（代理对问题与 `WideCharToMultiByte`/`wprintf` 行为一致）。

---

## Z3 · 大模型交叉验证 / 多模型投票是否可靠

**问题页**：<https://www.zhihu.com/question/1918234098859742683>（「多个AI交叉验证可以有效防止幻觉吗？」仅 2 条回答、各 2 赞 —— **低热度，非高赞**）

### A1 · Aetline ｜ 赞同 2 ｜ 发布于 2025-06-17 09:11（页面标注「包含 AI 辅助创作」）
- 流程断言（可核查部分是流程而非事实）：三模型互验 = 主模型生成 → 辅模型查逻辑漏洞 → 辅模型查事实细节；冲突即人工复核；置信度分层（多模型一致+可溯源 / 部分矛盾 / 关键冲突）。
- **不可采信数字**（无出处、无法核查）：DeepSeek-R1 幻觉率「14.3%」、文心一言「约 3.9%」、腾讯 AI Lab「差异率 >10% 需 100% 人工复核」、LawGPT/Med-PaLM「低 40%」。→ 标记为**噪声**，不得引用。
- 有价值的一句：**所有模型基于同一份过时/错误数据时会集体误判**（即伪共识风险）。

### A2 · 中文房间 ｜ 赞同 2 ｜ 发布于 2026-05-23 21:09
- **MARCH**（阿里 Qwen，2026-03，`arXiv:2603.24579`）：Solver 生成 / Proposer 拆原子命题 / Checker 验证，**Checker 看不到 Solver 原始输出**，对「证据材料」逐条判定有据支持 / 证据反驳 / 证据不足 → 用信息不对称打断确认偏误（作者自注：**尚未检索到 AAAI 2026 正式录用**）。
- **InEx**（Yang et al., AAAI-26）：熵估计不确定性 + 编辑 Agent / 自反思 Agent 迭代，报告「4%–27% 相对提升」；作者明确该数字**属于 InEx 而非 MARCH**。
- **多视角一致性**（Kong et al., 2025, FITEE，国防科大）：同一问题多种表述提问，前后矛盾即标记可疑；黑盒、零资源、仅需 API。
- 自述实战（IFRS 9 预期信用损失模型）：**伪计算（Pseudo-computation）**检测三步 —— ① 执行顺序检查（结果在计算前出现即假）② 依赖链回溯（每个中间量须有真实上游）③ **扰动测试：`LGD` +0.01 但 `ECL` 不变 ⇒ 伪计算**。
- 四层过滤器：L1 格式验证（JSON Schema/正则，成本≈0）→ L2 自一致性（同题问 3 次，3× token）→ L3 交叉验证（不同模型 + 投票，N× token）→ L4 外部锚定（拆原子命题逐条查库）。
- 引用风险：arXiv 编号 `2603.24579` 与 2026-03 时间自洽，但**本次未独立核实**；采信前需自查 arXiv/AAAI。

---

## 采集偏差与未获取项（open questions）

1. **`insertText` 专项缺失**：搜索「Chrome DevTools Protocol 踩坑」「puppeteer 输入中文 失败」均回落到 question/562879512，知乎无该细分问题的高赞回答。受控 textarea / contenteditable 的输入配方仍以 `reports/FINAL-三源对照结论.md` 的实测为准。
2. **PS 5.1 BOM / 奇数非 ASCII 字节**：知乎检索无对应问题页；候选 `question/540603960`（「无法加载文件，因为在此系统上禁止运行脚本」）**0 回答**，只能放弃该页。Z2 仅覆盖通用编码，**BOM 断言不可从知乎取证**。
3. **「高赞」不达标**：Z2 赞同数 1/2/1，Z3 各 2 赞；仅 Z1 达到 555/59/35。低赞 ≠ 错误，但按契约降置信度至 ≤0.4 使用。
4. **搜索预算**：共 6 次（CDP / puppeteer / PS 中文乱码 / ExecutionPolicy / Windows 命令行乱码 / 大模型交叉验证），已用满，未继续扩展。

## 站点技术备注（供后续采集脚本复用）

- **赞同数不在按钮 `innerText` 里**，要读 `[aria-label^="赞同"]` 的 `aria-label`（如 `赞同 555`）；`button` 文本在部分布局为空 → 会误判为 0 赞。
- 回答正文取 `.RichContent-inner` 的 `textContent`（折叠 clamp 不影响取全文）；日期/编辑时间在 `.ContentItem-time`；作者在 `.AuthorInfo-name`。
- `.List-item` 里有 4 类噪声卡片（「邀请回答」「可能对该问题感兴趣」「更多推荐」），须按「存在 `.RichContent-inner`」过滤后才是真实回答。
- 知乎拒绝裸 curl，但已登录 Edge 正常 → 必须走 CDP（本报告全部走 `scripts\cdp.ps1`）。
