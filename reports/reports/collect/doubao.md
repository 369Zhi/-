# 豆包（doubao.com）原始回答采集

- 采集时间：2026-11-01 15:26–15:30（Asia/Shanghai）
- 站点/页面：https://www.doubao.com/chat/38450791904530946 （由 /chat/ 新对话自动跳转生成）
- 浏览器：用户已登录的调试 Edge（CDP 端口 9222，profile `.edge-ai`）
- 模型显示名称：页面左上工作区标签「豆包」；输入框右侧模型按钮「快速」（`data-testid=chat_input_action_model`）；页头提示「AI 生成可能有误 请核实」
- 采集方式：CDP（`Runtime.evaluate` / `Input.dispatchMouseEvent` / `Input.dispatchKeyEvent` / 原生 value setter + `input` 事件），逐题整段抄录 `[data-testid=receive_message]` 的 `message_text_content` innerText，未做摘要或改写

---

## Q1

**问题原文**

> 在 Windows 上用 PowerShell 5.1 通过 CDP 驱动 Edge：Input.insertText 与 Input.dispatchKeyEvent 有什么区别？为什么 insertText 有时往 contenteditable 里插不进内容？

**回答全文**（原样，页面渲染顺序）

▸ 原文开始
一句话总览
Input.insertText：高层接口，告诉浏览器插入一段文本字符；浏览器内部走编辑器输入逻辑，不派发 keydown/keypress/keyup，适合普通文本，不适合快捷键、方向键、退格。
Input.dispatchKeyEvent：底层键盘事件模拟，派发完整 keydown / keypress / keyup DOM 键盘事件序列，更贴近真实按键；可以发控制键、组合键，但不会自动处理输入法 / 编辑器合成文本。
Edge 和 Chrome CDP 这套 Input 协议是同源的，PowerShell 5.1 只是 WebSocket 调用者，差异不在 PS，在 CDP 接口本身。
详细区别
1. Input.insertText
参数就一个 text，非常简单：
json
{"method":"Input.insertText","params":{"text":"hello"}}
✅ 特点
目标要求：焦点必须已经在可编辑区域（input/textarea/contenteditable）
语义：文本插入，走浏览器内部编辑引擎（和粘贴、输入法上屏走同一条路）
不会触发 keydown /keypress/keyup，很多前端监听键盘事件的逻辑收不到通知
天然支持 Unicode、emoji、复杂字符，不需要拆按键
不能发送控制按键：Backspace、Enter、箭头、Ctrl+C 这些都做不了
会遵循选区：在光标位置插入；如果有选中文字，默认替换选中内容
❌ 局限：很多富文本编辑器、自定义 contenteditable 自己接管了输入管线，不监听原生文本插入，只监听键盘事件 → 这就是你遇到的「插不进去」最常见原因。
2. Input.dispatchKeyEvent
可以单独下发 keyDown / keyUp / rawKeyDown，完整模拟键盘事件：
json
{"method":"Input.dispatchKeyEvent","params":{"type":"keyDown","key":"a","code":"KeyA"}}
✅ 特点
派发标准 DOM KeyboardEvent，页面 JS onkeydown/onkeyup 可以捕获
支持组合键：Ctrl、Shift、Alt、方向键、退格、回车都可以
适合那些自己接管输入、只认键盘事件的富文本框架（如很多基于 ProseMirror、Slate、Quill 的编辑器）
缺点：它只是发事件，不等于一定会打出字符；输入法、IME 合成、浏览器字符映射不自动处理；中文、复杂字符要自己处理 IME 或者逐个字符模拟。
一个常见误区：连续 dispatchKeyEvent 发 a、b、c，不等于一定能在编辑器里出现 abc；编辑器可以选择忽略事件。
表格
维度	insertText	dispatchKeyEvent
底层行为	浏览器编辑层插入文本	派发 DOM KeyboardEvent
keydown/keyup 事件	❌ 不触发	✅ 触发
适合普通文字	✅ 简单高效	⚠️ 麻烦，要循环
适合快捷键 / 退格 / 方向	❌ 不行	✅ 支持
contenteditable 兼容性	原生简单 div 好用；自定义富文本经常失效	自定义富文本更稳
中文 / IME	✅ 原生支持文本	❌ 不自动走输入法合成
重点：为什么 insertText 往 contenteditable 里插不进去？
按概率排序：
① 编辑器框架拦截输入，只监听键盘事件（最高发）
现代富文本（Slate、ProseMirror、Draft.js、Quill、Notion 类编辑器）不依赖浏览器原生编辑行为：
它们自己维护一套虚拟文档、选区 state
只监听 keydown 事件，自己计算文本变更，忽略浏览器原生插入文本行为
insertText 是告诉浏览器内核去改 DOM，但框架状态不同步 → 字看起来没进去，或者一闪消失
现象：你 CDP 返回成功，没有报错，但是页面看不到文字。
👉 对策：改用 dispatchKeyEvent 逐个字符模拟按键；或者用 Runtime.evaluate 在页面 JS 层面调用编辑器暴露的 API（优先 evaluate，比模拟键盘稳定）。
② 焦点不在正确的编辑宿主 / Shadow DOM 隔离
insertText 要求：当前焦点（document.activeElement）就是那个可编辑宿主。
contenteditable 在 closed shadow root 里面，焦点捕获行为特殊
点击后焦点跑到外层宿主，实际编辑子节点不是 activeElement
或者页面有 blur()、自动失焦脚本抢焦点
👉 排查：CDP 调用 Runtime.evaluate 执行 document.activeElement，确认是不是目标节点；必要时先用 Input.dispatchMouseEvent 点击内部元素，再输入。
③ 选区（Selection/Caret）丢失、被锁定
contenteditable 依赖浏览器 Selection：
有些编辑器在渲染循环里强制重置选区、清空 range
你在插入时选区不在编辑区域
insertText 是在当前光标选区处插入，没有有效选区就可能无效果。
👉 验证：evaluate 查看 window.getSelection()；可以先用 JS 把选区定位好再 insertText。
④ 页面有 beforeinput /input 事件拦截、preventDefault
insertText 会触发 beforeinput + input 事件（注意：不是 keydown）。
如果页面脚本：
js
el.addEventListener('beforeinput', e=>e.preventDefault())
直接阻止了文本输入流，insertText 就无效。
区分：beforeinput 拦 insertText；拦不住单纯 dispatchKeyEvent（除非自己监听 keydown）。
⑤ Edge 版本 CDP 实现差异 + 自动化检测干扰
Edge 某些版本对 contenteditable 的 insertText 行为和 Chrome 细微差异；
如果启动参数不带 --disable-features=AutomationControlled，页面自动化检测脚本会额外干预输入行为。
⑥ 时序问题（PowerShell 5.1 WebSocket 很容易踩）
PowerShell 5.1 用 WebSocket 发 CDP 是同步 / 手动等待的坑大户：
点击元素 → 立刻发 insertText
页面富文本异步挂载、异步初始化状态还没完成
命令发早了，焦点和编辑器还没就绪
👉 对策：加延迟，或者 evaluate 轮询判断编辑器就绪再发送；不要连续无间隔轰炸 CDP 指令。
实操选型建议（你在 PS5.1 + Edge CDP）
简单原生 <div contenteditable="true"> /input：优先 insertText，最快最简单
富文本编辑器（Slate/ProseMirror 这类）：
首选：Runtime.evaluate 注入 JS，调用编辑器 API 或者直接操作 Selection + execCommand / setRangeText（兼容性取舍）
次选：dispatchKeyEvent 模拟按键；长文本要循环，注意加间隔
需要回车、退格、Ctrl+A：只能 dispatchKeyEvent
混合场景：先用 dispatchKeyEvent 发光标 / 控制键，再 insertText 输入大块文本（前提编辑器支持 insertText）
一个 PowerShell 5.1 CDP 调用的小提醒
PS5.1 没有内置 WebSocket 客户端，一般是用 System.Net.WebSockets.ClientWebSocket；一定要顺序 await / 接收返回消息，CDP 是异步双向协议，乱序发送很容易出现：命令返回成功，但页面行为错乱，你误以为是 insertText 本身 bug。
如果你想要，我可以给一段可直接跑的 PowerShell 5.1 示例：
启动 Edge 远程调试
WebSocket 连接 CDP
演示：insertText 版本 和 dispatchKeyEvent 逐字符输入版本，对比 contenteditable。
▸ 原文结束

**观察**

- 是否截断：否，整段回答完整（innerText 3598 字符），结尾停在模型主动追问「要不要给一段可直接跑的 PS 5.1 示例」。
- 拒答 / 验证码：无拒答，未出现验证码或风控提示。
- 附属内容：回答前有「搜索 6 个关键词，参考 4 篇资料」的联网检索标记（属页面渲染的一部分，已随原文保留位置）。
- 页面异常（重要，供下游脚本参考）：
  1. 用户原标签页是一条**已结束的任务流程会话**（`https://www.doubao.com/chat/38450674007697666`），其发送按钮 `id=flow-end-msg-send`、`disabled` 恒为 true，任何输入 + Enter 都无法发送；因此改在同一标签页打开 `/chat/` 新对话。
  2. 新对话页弹出「每日签到，免费用豆包工作」弹层（`div[class*=dialogMask]`，z-index 2000），覆盖整页、吞掉所有鼠标点击 —— 这是「点击发送按钮无效」的直接原因；需先点弹层关闭按钮（`button[aria-label="关闭"]`）或隐藏 mask。
  3. 输入框为受控 `<textarea data-testid=chat_input_input>`（不是 contenteditable）。`Input.insertText` 能把文本写进 DOM 且发送按钮变可用，但 Enter 与点击发送均不触发提交；最终生效的组合是：原生 `HTMLTextAreaElement.prototype.value` setter + 派发 `input` 事件 + `Input.dispatchKeyEvent` Enter —— 发出后 URL 跳转为 `/chat/38450791904530946`。
  4. 会话标题被站点自动改写为「PowerShell 5.1 Edge CDP Input 方法区别与问题」（非原问题）。

---

## Q2

**问题原文**

> PowerShell 5.1 执行 .ps1 时，如果文件是 UTF-8 无 BOM 且含中文，会出现什么现象？根因是什么？

**回答全文**：未获取

**观察**：未获取。原因：Q1 回答结束后在同一 textarea 注入 Q2 时，页面的 `textarea` 已不在 DOM 中（`document.querySelector('textarea')` 返回 null，疑似页面重渲染中），注入与 Enter 均未生效；按「不追加追问、不重试」的约束未继续重试，故该题无回答。

---

## Q3

**问题原文**

> 用「三个不同厂商的大模型交叉验证同一个结论」这种方法，最大的失效场景是什么？给出反例。

**回答全文**：未获取

**观察**：未获取。原因同上（Q2 未成功发送后会话输入区未恢复，未再发起新的发送尝试）。

---

## 采集侧小结（仅事实，不含解读）

- 成功采集 1/3 题（Q1），全文无删改。
- 豆包网页版与 CDP 自动化的三处硬障碍：已结束的任务流程会话锁死发送、营销弹层遮挡点击、受控 textarea 对 `insertText` + Enter 不响应。
- 可复用的发送配方（本机实测有效）：关闭 `dialogMask` → 原生 value setter 写入 → 派发 `input` 事件 → `Input.dispatchKeyEvent` Enter。
