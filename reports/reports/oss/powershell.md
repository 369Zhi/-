# 开源检索：PowerShell 5.1 编码坑与 Windows 脚本项目

检索人：oss-crawl-ps ｜ 日期：2026-11-01 ｜ 范围：PowerShell 5.1 下 UTF-8 无 BOM 脚本含中文、调用外部 exe、CDP 客户端、WAV 处理
证据分级：本地实测 > 官方文档 > 上游 issue 讨论 > 第三方 repo

---

## 0. 结论先行（TL;DR）

1. **本机就是坑上作业**：`powershell.exe` = Windows PowerShell 5.1.26100.9711，ANSI 代码页 **936 (gb2312)**，`$OutputEncoding` = **us-ascii**。
2. **`.ps1` 里出现中文 + 存成 UTF-8 无 BOM = 必踩**。轻则乱码，重则解析报错，**最坏是静默改坏字符串**（exit 0，路径/URL 已经被吃掉一个字符）。
3. **规避成本极低**：`.ps1` 一律存 **UTF-8 with BOM**（或纯 ASCII + 中文外置到数据文件）。PS7 存无 BOM 也行，但要兼容 5.1 就必须带 BOM。
4. 中文 prompt 不要硬编码进 `.ps1`，放 `payload/*.txt`（UTF-8 无 BOM）用 `Get-Content -Encoding UTF8` 读——**这样同一个脚本既能被 5.1 跑，也不会把中文写进代码**。

---

## 1. 本地实测（最强证据，可复现）

环境：PS 5.1.26100.9711 / zh-CN / `[Text.Encoding]::Default` = gb2312 (936) / `[Console]::OutputEncoding` = utf-8 / `$OutputEncoding` = **us-ascii**
方法：用 .NET 精确控制 BOM 写入 `.ps1`，再 `powershell.exe -NoProfile -File` 执行。

| 脚本内容（UTF-8 无 BOM） | 结果 |
| --- | --- |
| `$s = '中文'` | exit 0，输出 `涓枃`（乱码但能跑） |
| `$s = '中'` | **exit 1** `The string is missing the terminator: '` |
| `$s = '中文测'` | **exit 1** 同上 |
| `$s = '中文测试'` | exit 0（乱码） |
| `$s = '北京大'` / `"中"` / `'https://x/?q=北京大'` | **exit 1** 引号被吃掉，字符串未闭合 |
| `if($true){ $s='中'; }` | **exit 1** `Missing closing '}' in statement block` |
| `@{ 中文 = 1 }` | **exit 1** `Missing '=' operator after key in hash literal` |
| `$p = 'C:\新建文件夹\a.txt'` | **exit 0，但路径变成 `C:\鏂板缓鏂囦欢澶筡a.txt`（长度 16，反斜杠被吞）** ← 最危险 |
| `Write-Output 'C:\Users\倪晖\...'`（本机 home 含中文） | **exit 1** `PathNotFound`：脚本里的 home 路径直接失效 |
| 同上，改为 UTF-8 **with BOM** | 全部 exit 0、中文正确 |

**机制（实测 + 上游解释一致）**：5.1 把无 BOM 文件按 ANSI 代码页解码。本机是 GBK/936（双字节）：
一段连续非 ASCII 字节长度是**奇数**时，最后一个字节变成"等待后继"的 GBK 首字节，**吞掉紧跟的下一个字符**——
吞掉 `'`/`"` → 字符串未闭合；吞掉 `}` → 少大括号；吞掉 `\` → **静默错路径**。
上游 issue #26642 里 mklement0（2025-12-26）给出的 1252 版机制同类：UTF-8 字节被解码成 `0x91/0x92`（弯单引号）、`0x93/0x94`（弯双引号），而 PowerShell 把弯引号当 ASCII 引号的同义词 → 字符串被提前闭合。
一句话：**"奇数个汉字就炸"**（每个汉字 3 字节，奇数个 = 奇数字节）；偶数个只乱码不报错。

其它实测到的 5.1 默认值（都会咬人）：

| 操作 | 5.1 实测 | 说明 |
| --- | --- | --- |
| `Get-Content x.txt`（文件是 UTF-8 无 BOM） | 读出乱码 | 默认按 ANSI 读；必须 `-Encoding UTF8` |
| `'中文' \| Out-File -Encoding UTF8` | 前 3 字节 `239,187,191` | 5.1 的 `UTF8` = **带 BOM** |
| `[IO.File]::WriteAllText($p,$s,(New-Object Text.UTF8Encoding($false)))` | 前 3 字节 `228,184,173` | 5.1 写无 BOM UTF-8 的唯一稳法 |
| `'中文内容' \| more`（管道给原生 exe stdin） | `????` | `$OutputEncoding` 默认 us-ascii |
| `cmd /c echo 中文参数`（中文当**命令行参数**） | 输出 `中文参数` 正常 | 参数走 CreateProcess 宽字符，不经过 `$OutputEncoding` |

---

## 2. 候选清单（5 个，只记标题/时间/许可证/结论）

| # | 候选 | 时间 | 许可证 | 结论 / 可复用做法 |
| --- | --- | --- | --- | --- |
| 1 | [PowerShell/PowerShell#26642](https://github.com/PowerShell/PowerShell/issues/26642) `PowerShell fails to parse .ps1 files when using UTF-8 BOM + Chinese filename + OneDrive path (parser reports missing braces)` | 2025-12-23 开，Resolution-Answered 关闭 | 仓库 MIT | 报告人以为是 BOM+OneDrive+中文文件名的问题；**mklement0 结论：与 OneDrive/文件名无关，是"无 BOM 的 UTF-8 被 5.1 按 ANSI 解码"的 by-design 行为**。可直接抄的判定句：要兼容 5.1 就存 UTF-8 with BOM；只想给 PS7 跑就写 `#requires -version 7`。 |
| 2 | [PowerShell/PowerShell#25698](https://github.com/PowerShell/PowerShell/issues/25698) `pipe default OutputEncoding should match Console::OutputEncoding to avoid encoding issues with native console tools` | 2025-06-26 开，仍 open | 仓库 MIT | 官方确认：管道喂原生 exe 用的是 `$OutputEncoding`（5.1 默认 us-ascii），不是控制台编码 → 中文进 exe stdin 变 `????`。与本地实测吻合。做法：脚本开头显式设 `$OutputEncoding`/`[Console]::OutputEncoding`。 |
| 3 | [MicrosoftDocs/PowerShell-Docs · 5.1 `about_Character_Encoding.md`](https://github.com/MicrosoftDocs/PowerShell-Docs/blob/main/reference/5.1/Microsoft.PowerShell.Core/About/about_Character_Encoding.md) | 随文档持续更新 | 仓库 API 返回 **NOASSERTION**（未给 SPDX，实际为 CC-BY-4.0/MIT 混合）→ 引用需注明出处，**不可整段复制** | 原文照录："If you need to use non-Ascii characters in your scripts, save them as UTF-8 with BOM. Without the BOM, Windows PowerShell misinterprets your script as being encoded in the legacy ANSI codepage."；另给出各 cmdlet 默认编码清单（`New-Item -Type File -Value` 写无 BOM UTF-8、`Export-Csv` 写 ASCII 等）。 |
| 4 | [puppeteer/puppeteer](https://github.com/puppeteer/puppeteer)（CDP 客户端参考实现） | 活跃，pushed 2026-11-01，★95.8k | **Apache-2.0** | 三源脚本若走 CDP：用 `Runtime.evaluate` + `awaitPromise: true`、`returnByValue: true` 拿回答；`Page.navigate` 后必须等网络/文本稳定而非固定 sleep。**只借鉴协议用法，不复制源码**（Apache-2.0 与本项目需保留版权声明，且这里只需协议语义）。 |
| 5 | [naudio/NAudio](https://github.com/naudio/NAudio)（WAV 处理） | 活跃，pushed 2026-10-26，★6.3k | **MIT** | WAV 剪辑/播放的 .NET 参考实现（`WaveFileReader`/`WaveFileWriter`/`OffsetSampleProvider`）。本项目语音收尾已有 `scripts\voice.ps1`，**够用就不要引入依赖**；需要按秒裁剪时照它的"reader→skip/take→writer"思路自己实现。 |

> 另：`PowerShell/Windows-PowerShell` 仓库**不存在**（`gh repo view` 报 Could not resolve）；Windows PowerShell 5.1 的 issue 也收在 `PowerShell/PowerShell` 里，检索时别再找那个仓库。

---

## 3. 怎么写脚本才能避开这个坑（可执行结论）

### 3.1 旧版（Windows PowerShell 5.1，`powershell.exe`）——本项目默认目标

```powershell
# ① 文件层面（唯一硬要求）：.ps1 存 UTF-8 with BOM。
#    手工/程序化生成时用这条，不要用 Set-Content 的默认：
$enc = New-Object System.Text.UTF8Encoding($true)          # $true = 带 BOM
[System.IO.File]::WriteAllText($path, $code, $enc)
#    Out-File -Encoding UTF8 在 5.1 也带 BOM，可用；但 Set-Content 默认是 ANSI，禁止用来写脚本。

# ② 更稳的工程做法：.ps1 全部写成纯 ASCII，中文外置到 UTF-8 无 BOM 的数据文件
$q = Get-Content (Join-Path $PSScriptRoot 'payload\query.txt') -Encoding UTF8 -Raw   # 默认按 ANSI 读 → 必须 -Encoding UTF8
# 等价写法：[System.IO.File]::ReadAllText($p, [System.Text.Encoding]::UTF8)

# ③ 写结果文件别带 BOM（下游 JSON 解析器讨厌 BOM）
[System.IO.File]::WriteAllText($out, $json, (New-Object System.Text.UTF8Encoding($false)))

# ④ 调外部 exe：命令行参数里的中文没问题；管道/stdin 前必须显式设编码
$OutputEncoding = New-Object System.Text.UTF8Encoding($false)   # 5.1 默认 us-ascii
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$r = & curl.exe -s -X POST $url --data-binary "@$bodyFile"      # 走文件，别走管道拼中文
```

- 中文**不要**写进 `-Command "..."` 字符串再喂 `powershell.exe`（外层引号 + 内层中文 = 双重风险），改写临时 `.ps1`（带 BOM）再 `-File`。
- 路径里带中文（本机 home 就是 `C:\Users\倪晖`）：**不要硬编码**，用 `$env:USERPROFILE`；脚本一旦无 BOM，硬编码的 home 路径 100% 失效（实测 `PathNotFound`）。
- 需要"这个脚本只给 7 跑"时，首行 `#requires -version 7` 让 5.1 直接拒绝执行，而不是跑出乱码。

### 3.2 新版（PowerShell 7.x，`pwsh`）——可选兼容路径

```powershell
# 7+ 默认就是 utf8NoBOM：无 BOM 的 UTF-8 脚本能被正确解析，中文可以放心写在代码里。
# 但默认值仍要显式化，别依赖：
$OutputEncoding = [System.Text.Encoding]::UTF8      # 7 默认已是 utf8NoBOM
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8NoBOM'
Get-Content $p -Encoding utf8                       # 7 默认就是 utf8，写全更清楚
# 输出：Set-Content / Out-File 的 -Encoding utf8 在 7 = 无 BOM；5.1 的 utf8 = 带 BOM。同一份脚本跨版本时写明 utf8BOM / utf8NoBOM。
```

- **跨版本共存规则**：要同一份 `.ps1` 在 5.1 和 7 都能跑 → 存 **UTF-8 with BOM**（7 能正确识别 BOM，只是 Unix 工具可能不认）。
- **只跑 7** → 无 BOM 无妨，但 `.ps1` 里别依赖 5.1 行为（`$OutputEncoding`、`Get-Content` 默认编码、`Out-File -Encoding UTF8` 语义三处语义不同）。

### 3.3 一句话验收清单（写脚本后自查）

1. `(Get-Content .\x.ps1 -Encoding Byte -TotalCount 3) -join ','` → 必须是 `239,187,191`（5.1 目标脚本）。
2. `powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\x.ps1` 跑一遍（**本机默认 ExecutionPolicy 会直接拒跑，必须显式 Bypass**，与 `scripts\voice.ps1` 的调用方式一致），看有没有 `TerminatorExpectedAtEndOfString` / `Missing closing '}'` / `PathNotFound`——**尤其 PathNotFound 要当编码错误查，不要当路径写错**。
3. 脚本里 `grep -n '[^\x00-\x7F]'`（非 ASCII）为 0 最稳；不为 0 就必须有 BOM。
4. 输出 JSON：确认无 BOM（前 3 字节不是 `239,187,191`），否则下游 `ConvertFrom-Json`/Python `json.load` 可能报错。

---

## 4. 可证伪 / 怎么验证我错了

- 若在本机把 `'中'` 脚本存成 **无 BOM**，`powershell.exe -File` 却 exit 0 且输出正确 → 本文机制判断错误。
- 若把 `$OutputEncoding` 保持 us-ascii，`'中文' | <exe>` 仍能正确传递 → #25698 那套在 5.1 不成立。
- 若 `Out-File -Encoding UTF8` 在 5.1 写出无 BOM → 3.1① 的判断需修正。
- 复验命令（PowerShell 5.1，注意本机必须加 `-ExecutionPolicy Bypass`，否则报 "running scripts is disabled" 而不是编码错）：
  `$e=New-Object Text.UTF8Encoding($false); [IO.File]::WriteAllText("$env:TEMP\t.ps1","`$s='中'`r`nWrite-Output `$s",$e); powershell.exe -NoProfile -ExecutionPolicy Bypass -File "$env:TEMP\t.ps1"`
  把 `$e` 换成 `New-Object Text.UTF8Encoding($true)` 即对照组，应正常输出 `中`。

---

## 5. 风险与边界

- **最坏情况不是报错而是静默错串**（实测 `'C:\新建文件夹\a.txt'` exit 0 但反斜杠被吞）。所以"跑通了"不等于"编码对了"，必须回读关键字符串比对长度。
- 本文的"奇数字节"规则是 **GBK/936 环境下**的实测结论；在 1252/UTF-8 代码页机器上表现形态不同（弯引号假阳性，见 #26642），但规避手段（带 BOM）通用。
- 候选 #3 的仓库许可证为 NOASSERTION：文档结论可引用，**不可大段复制进本项目**。
- 未做：PowerShell 7 实机验证（本机没有 `pwsh`，`Get-Command pwsh` 为空）——3.2 的内容来自官方文档与默认值说明，未实测，置信度 0.6。
