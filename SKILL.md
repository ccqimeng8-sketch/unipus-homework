---
name: unipus-homework
description: 在U校园（Unipus）自动完成英语课程必修练习。当用户要求"做U校园作业"、"完成Unit X练习"、"刷U校园"时使用。需要 browser-use 插件和用户手动登录。
argument-hint: "[课程目录URL] [要做的单元，如 unit1]"
---

# U校园作业自动化

在 U校园（u.unipus.cn）完成指定 Unit 的**必修练习**（黄色星标），跳过视频、课文文本和单词卡列表，以及非必修小节。

## 前置条件

- 加载 `browser-use:control-browser` skill，全程通过 `mcp__node_repl__js` 驱动内置浏览器。
- **每个 JS 调用都要重新 bootstrap**（fresh kernel），并按已验证的 tab id 用 `browser.tabs.get(id)` 恢复标签页。

## 工作流程

### 1. 打开课程目录页并处理登录

用用户提供的目录 URL（形如 `https://u.unipus.cn/user/student/mycourse/courseCatalog?courseId=...&school_id=...&eccId=...&classId=...&coursetype=0`）。

- 若跳转到 `sso.unipus.cn/sso/login` → 未登录。**密码只有用户知道**：让用户在内置浏览器里手动登录，然后轮询 `tab.url()` 直到 URL 不再包含 `sso.unipus.cn/sso/login`（每3秒查一次，用长 timeout 的调用）。
- 登录成功后重新 `goto` 目录 URL。

**⚠️ 两个站点会话各自独立且都会过期。** `u.unipus.cn`（目录）和 `ucontent.unipus.cn`（做题内容页）是两套 CAS 会话：目录页登录后，点开练习仍可能跳到 `sso.unipus.cn/sso/login?service=...ucontent.unipus.cn...`。此时同样让用户手动登录一次即可，登录后会自动跳回原地址。

- 用户登录后若 `browser.tabs.list()` 返回空（tab 被释放为 userTab），用 `browser.user.openTabs()` 找到它再 `browser.user.claimTab(info)` 接管。
- **长时间作业期间会话容易过期**，中途 `goto` 目录或点练习时随时可能跳回登录页；每次跳转后都要检查 `url.includes("sso/login")` 再继续。

### 2. 解析目录，确定必修练习清单

截图确认页面布局（每行左侧橙色"必修"标签 = 必修小节；右侧一排活动图标）。然后用 evaluate 解析图标 class：

| class | 含义 |
|---|---|
| `icon-bixiu` | 必修小节标记 |
| `icon-lianxi` | 练习（要做） |
| `icon-wenben` | 课文文本（跳过） |
| `icon-check` | 单词卡列表（跳过） |
| `icon-shipin` | 视频（跳过） |
| 裸 `iconfont` 且颜色 `rgb(255, 129, 0)` | **黄色星标 = 必修活动** |

带星活动的定位器（按行文本收窄，行数必须=1）：

```js
const row = tab.playwright.locator("li").filter({ hasText: "1-3 iExplore 1: Building your language" });
const starLinks = row.locator("a:has(.iconfont:not(.icon-lianxi):not(.icon-wenben):not(.icon-check):not(.icon-shipin))");
// starLinks.nth(k) 依目录显示顺序排列
```

注意：图标顺序 ≠ 活动标签顺序。按类型映射（练习/文本/词表/视频），别猜位置对应关系。

### 3. 进入活动并处理弹窗

点击星标链接后进入 `ucontent.unipus.cn/_pc_default/pc.html#...` 内容页。

**每次页面/活动加载、"继续学习"跳转、翻页后都可能弹出"本单元学习时间"对话框**（是否必修：必修/非必修），挡住一切点击。Playwright 常规点击会报 covered 错误，用"动态定位+坐标点击"关闭：

```js
async function dismissDialog() {
  const p = await tab.playwright.evaluate(() => {
    const btns = [...document.querySelectorAll("button")].filter(b => (b.innerText||"").trim() === "确定" && b.getBoundingClientRect().width > 0);
    if (!btns.length) return null;
    const r = btns[0].getBoundingClientRect();
    return { x: Math.round(r.x + r.width/2), y: Math.round(r.y + r.height/2) };
  });
  if (p) { await tab.cua.click(p); await tab.playwright.waitForTimeout(600); }
  return !!p;
}
```

内容页顶部有活动标签页（如 `Viewing | Reading in detail | Vocabulary | Understanding`），可用 `getByRole("link", { name: "标签名" })` 直接切换活动。

### 3.5 透明遮罩残留：按钮"能命中但点了没反应"

关掉"本单元学习时间"弹窗后，页面常残留 **`z-index: 1400 / 1500` 的全屏透明 div**。它不在 snapshot 里显示（`opacity:0` 或纯空 div），但会吞掉所有鼠标与键盘事件。

症状对照：
- `document.elementFromPoint(按钮中心)` 返回的是 `DIV`（非按钮）→ 有遮罩
- 按钮 `isConnected: true`、`getBoundingClientRect()` 正常、`el.click()` 也无反应
- 提交/翻页反复"点了没反应"，且没有 toast、没有弹窗

检测 + 处置（每次提交前都可顺手做一遍，React 重渲染会恢复遮罩，所以随时重做）：

```js
const hit = await tab.playwright.evaluate(() => {
  const b = [...document.querySelectorAll("button")].filter(x => (x.innerText||"").trim() === "提交" && x.getBoundingClientRect().width > 0).pop();
  const r = b.getBoundingClientRect();
  const top = document.elementFromPoint(r.x + r.width/2, r.y + r.height/2);
  return { hittable: top === b || (top && b.contains(top)), topTag: top?.tagName };
});
// 关闭拦截：把空的高层级 fixed div 的 pointer-events 置 none
await tab.playwright.evaluate(() => {
  let n = 0;
  [...document.querySelectorAll("div")].forEach(d => {
    const s = getComputedStyle(d);
    const zi = parseInt(s.zIndex || "0");
    if (zi >= 1000 && d.innerText.trim() === "" &&
        !d.querySelector("button, textarea, input")) {
      d.style.pointerEvents = "none"; n++;
    }
  });
  return n;
});
```

处置后立刻执行那一次点击（别等太久，遮罩可能被重渲染恢复）。

### 4. 作答（关键：必须真实键盘输入 + 必须真实失焦）

**⚠️ 教训一：`fill()` 对很多题型不生效。** 界面上值显示了，但提交时报"还有N道题没作答"——应用状态没记录。必须用真实键盘事件：

```js
// 点击输入框中心 → Ctrl+A 全选 → Backspace 清空 → 逐键输入
await tab.cua.click({ x, y });            // 框的位置由 evaluate getBoundingClientRect 取得
await tab.cua.keypress({ keys: ["Control", "a"] });
await tab.cua.keypress({ keys: ["Backspace"] });
await tab.cua.type({ text: answer });
```

**⚠️ 教训二：答案在"失焦"时才写入应用状态，不是打字时。**
表现为：框里明明有字，翻页回来就空了 / 提交报"还有N道题"。每个框打完字后必须制造一次**真实失焦**——把焦点移到另一个输入框（不要只靠 `el.blur()`，也不要打完就翻页）：

```js
// 打完第 k 个框后，聚焦相邻框触发它失焦保存
await tab.playwright.evaluate((idx) => {
  const inputs = [...document.querySelectorAll("input[type=text], textarea")]
    .filter(el => el.getBoundingClientRect().width > 0);
  if (inputs[idx]) inputs[idx].focus();
}, k + 1);
await tab.playwright.waitForTimeout(600);
```

**⚠️ 教训三：`native setter` 直接写值会追加，不会替换。**
`Object.getOwnPropertyDescriptor(HTMLInputElement.prototype,"value").set.call(el, txt)` 单独用会让文本重复叠加（如 340 字符的重复句），且应用状态里也是脏的。必须分两步——先置空并派发 input，再写目标值：

```js
const r = await tab.playwright.evaluate((txt) => {
  const el = [...document.querySelectorAll("input[type=text], textarea")]
    .filter(x => x.getBoundingClientRect().width > 0)[idx];
  const proto = el.tagName === "TEXTAREA" ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
  const setter = Object.getOwnPropertyDescriptor(proto, "value").set;
  el.focus();
  setter.call(el, "");                                  // ① 先清空
  el.dispatchEvent(new Event("input", { bubbles: true }));
  setter.call(el, txt);                                 // ② 再写目标值
  el.dispatchEvent(new Event("input", { bubbles: true }));
  el.dispatchEvent(new Event("change", { bubbles: true }));
  return el.value;
}, answer);
// 写完后仍要真实失焦（见教训二）
```

> 注意：`el.select()` / `setSelectionRange(0,len)` 选中的文本，被 `cua.type()` 打字时**不会被替换而是追加**。清理脏数据只能走上面的两步 setter。

单选题（`input[type=radio]`）用 DOM click 最可靠：

```js
await tab.playwright.evaluate(() => {
  const el = document.querySelector('input[name="single-1"][value="A"]');
  if (el && !el.checked) el.click();
});
```

获取全部输入框位置的方法（注意 720px 视口，y>660 的框要先 `cua.scroll` 再重新定位）：

```js
const targets = await tab.playwright.evaluate(() => {
  const inputs = [...document.querySelectorAll("input[type=text], textarea")].filter(el => el.getBoundingClientRect().width > 0);
  return inputs.map((el, i) => { const r = el.getBoundingClientRect(); return { i, x: Math.round(r.x+r.width/2), y: Math.round(r.y+r.height/2), inView: r.y > 60 && r.y < 660 }; });
});
```

### 5. 多页活动与提交流程

- 右下角有分页指示（如 `1/2`）和 **下一题** 按钮；**每一页都可能有习题**（第1页不一定只是视频，有的活动第1页就是练习）。逐页检查 `input/textarea/radio` 数量再决定作答。
- **提交按整个活动计**：做完所有页后点一次 提交，一次性批改全部页的题目。
- 按钮都要"动态定位中心点 + `cua.click`"（页面滚动/弹窗会导致坐标漂移，固定坐标会点空）。

提交链（每一步后等 1.5~2.5 秒）：

```
提交 → 确认弹窗("小心只记录你第一次答题的得分") → [确认] → 结果弹窗(得分/查看答案/继续学习) → [继续学习]
```

- 客观题立即出分（**只记第一次成绩**，提交前想清楚答案）；主观题显示"已完成，暂无评分"，由老师批阅。
- **"继续学习"会按课程顺序跳到下一个活动**——如果下一个正好是同节的下一练习，直接链式做下去，省去回目录页。

#### 提交前的完整自检（踩过的坑全覆盖）

1. 关掉所有 `确定` 弹窗 → 2. 检查并解除透明遮罩 → 3. 逐框确认值长度正确（没有重复文本）→ 4. 翻页往返一次确认没丢答案 → 5. 点提交 → 6. 出现"确认"弹窗才算成功。

失败时的排查顺序：
- **出现 toast「还有N道题没有作答」** → 该页有输入没写进状态。翻页回去看哪个框空了（空的就是没注册的），用"真实键盘 + 真实失焦"重打；若是脏值则用两步 setter 清理。
- **完全没反应（无 toast、无弹窗）** → 99% 是透明遮罩（见 3.5），解除后立刻重点。
- **点了之后页面白屏 / DOM 全空** → 应用崩了，`tab.reload()` 重来；答案一般在服务器，刷新后在空框里补即可。

#### ⚠️ 视口尺寸变了之后，CUA 坐标全部失效

`tab.setViewportSize({ width, height })` 改过尺寸后，`tab.cua.click({x,y})` 的坐标空间**不会同步**（仍按旧的 720 高度算），导致所有坐标点击落空，但键盘输入、`el.click()` 正常。

- 尺寸改过就不要再用 CUA 坐标点击：改用 `evaluate` 里 `el.click()`，或关键词 `tab.cua.keypress` / `cua.type`（不受影响）。
- 或者把视口改回最初的值再等 1~2 秒，坐标才恢复。

### 6. 核对完成状态

回目录页，检查各行状态 class：`icon-yiwancheng-`=已完成、`icon-jinhangzhong`=进行中、`icon-todo`=未做。或截图目视（完成的图标从灰变蓝）。

## 拖拽配对题（react-dnd 排序组件）解法

某些练习（如 Vocabulary learning strategies）是 HTML5 拖拽题：左列修饰语，中列为词池卡片（`draggable=true`，class `sequence-pc-card--item`），拖动卡片到目标行交换位置。

1. **配对答案从课文找**：题目要求"Read through the text"——先去对应单元的 Reading in detail 页（`getByRole("link", { name: "Reading in detail" })`），用正则 `new RegExp(word + " [a-zA-Z]+", "gi")` 搜索修饰语后接的中心词。
2. **拖拽用真实鼠标**：`tab.cua.drag({ path: [起点, ...中间点20个, 终点] })`，起点=源卡片中心、终点=目标卡片中心（卡片既是源也是目标，拖 A 到 B 上=交换位置）。
3. **坐标必须现读现用**：每次拖拽前 evaluate 取卡片中心，不要缓存——之前失败的原因大多是用了一次性前的旧坐标。
4. **关键坑：第一次拖拽后后端状态会坏**（后续拖拽全部静默失败，无报错无变化）。可靠流程：**每完成一次拖拽验证结果，若失败则 `tab.reload()` 重置**（答案自动保存在服务端，排列不会丢），刷新后做下一次拖拽。每次刷新后的第一次拖拽成功率最高。
5. 拖完所有配对后注意：活动可能有**多页**（如 3 页：视频/拖拽/填空），提交按钮只在最后一页出现，翻到 `N/N` 页再点提交。

## 视频/音频/词表/课文的"完成"机制

带星的视频（shipin）、课文（wenben）、词表（check）活动即使不做习题也需要"过一遍"才算完成（小节才会变"已完成"）：

- **课文/词表**：只要打开过页面就会自动记为完成（无需翻页/提交）。
- **音频**：必须用**播放器自己的播放按钮**（DOM click `[class*=aplayer-play]`）以 **1倍速**播完；JS 直接 `av.play()` 或 2倍速播放不计入进度。
- **视频**：先确认缓冲完成（`video.readyState === 4`），再 cua 点击 `.vjs-big-play-button`，**1倍速静音播完**（10分钟视频需分多次调用轮询 `video.ended`）。
- **⚠️ 播放前必须关掉"本单元学习时间"弹窗**——弹窗遮挡时播放器会立刻自动暂停（play 后 ~10ms 被暂停，且 HTMLMediaElement.pause 不被调用，无调用栈）。播放后若 3 秒内 paused=true，先检查弹窗是否没关。
- **⚠️ 页面必须是可见状态（`document.visibilityState === "visible"`）否则播放器静默拒绝播放。** 内置浏览器面板被遮挡/隐藏时，点播放按钮毫无反应（`currentTime` 恒为 0、`paused` 恒为 true、无任何报错）。这**无法用 JS 绕过**——`document.visibilityState` 是不可 redefine 的浏览器属性，原型链补丁也无效。处置：请用户把 ZCode 窗口切到前台、确保能看到内置浏览器画面，再点播放。

```js
const vis = await tab.playwright.evaluate(() => document.visibilityState);
if (vis !== "visible") { /* 停手，请用户切到前台 */ }
```

顺带：页面初次加载时还会弹 **「录音机初始化错误: Permission denied」**（无麦克风权限），会反复弹出挡住操作。可用假麦克风流压住它：

```js
await tab.playwright.evaluate(() => {
  try {
    const canvas = document.createElement("canvas"); canvas.width = 2; canvas.height = 2;
    const cs = canvas.captureStream(2);
    const Ctx = window.AudioContext || window.webkitAudioContext;
    const ctx = new Ctx(); const osc = ctx.createOscillator();
    const dst = ctx.createMediaStreamDestination(); osc.connect(dst); osc.start();
    const tracks = [dst.stream.getAudioTracks()[0], cs.getVideoTracks()[0]].filter(Boolean);
    navigator.mediaDevices.getUserMedia = async () => new MediaStream(tracks);
  } catch (e) {}
});
```
- 视频未缓冲完时点击播放无效（页面有加载动画），等 `readyState===4` 再点。
- 完成判定回目录页看小节状态：`icon-yiwancheng-`=已完成 / `icon-jinhangzhong`=进行中 / `icon-todo`=未做；活动图标是否完成看有无 `passstyle` class。

## 测试与计时说明

- 单元测试直接可进，无次数/倒计时页面；客观题首次提交即锁定成绩（确认弹窗提示"小心只记录你第一次答题的得分"）。
- 各小节有自己的"板块学习时间"窗口（如 Unit 2 为 2026-09-28~10-18），不影响作答，仅提示。
- 主观题（翻译/简答）提交后显示"已完成，暂无评分"，由老师批阅。

## 经验与坑

1. **弹窗是最大干扰源**：任何导航后都先 `dismissDialog()` 再操作。
2. **Playwright 常规 click/fill 在该站经常失效**（遮罩覆盖、React 状态不接收合成事件），优先级：`el.click()`(radio) > 真实键盘输入(text) > cua 坐标点击(按钮) > Playwright fill(仅个别题型有效)。
3. 按钮/输入框位置每次都动态获取，不要缓存坐标。
4. 滚动后位置会变，先滚动、再重新 evaluate 位置、再点击。
5. 提交后若"确认"弹窗没出现，立即截图——很可能是"还有N道题没有作答"的 toast（输入没被状态记录），用真实键盘重打一遍。
6. 做题时认真分析题目再作答，主观题写结构完整、内容切题的答案（听力开放题不看音频也能合理作答）。
7. 新一代大学英语（提高篇）综合教程2 的目录参数：`courseId=654&school_id=8956&eccId=143590&classId=1058653`；换课程/学期时以用户给的 URL 为准。
8. **答案保存靠"失焦"触发**，打完字别急着翻页——先把焦点移到别的框，等 600ms。
9. **提交前必做"翻页往返验证"**：`下一题→上一题→下一题`，每页的值长度都没变才算答案真的进了状态。
10. **提交无反应先查透明遮罩**（`z-index>=1000` 的空 div），`pointer-events:none` 后立刻重点。
11. **清理脏数据只能用两步 setter**（置空+input → 写值+input），`select()` + `type()` 只会追加。
12. **改过 viewport 就别用 CUA 坐标**，改用 `el.click()`。
13. **视频/音频必须在页面可见时播放**，这是硬限制，只能请用户把窗口切前台。
14. 大段作业中途会周期性掉登录态（尤其 `ucontent.unipus.cn`），每次跳转都检查一下 URL。
15. **填空是精确文本匹配，答案取最短核心词，不要照搬整句。** 例：题干 "Imagination is the 2)" 标准答案仅 `root`（填 root of all of this 判错）；"a practical process that 3)" 标准答案 `puts imagination to work`（多写 your 判错）。提交前先看语法空格所需的最小片段；客观题首次提交即锁定，错了无法改。
16. **环境注意（Windows）**：`python` 可能默认指向 Doubao 沙箱副本而 `pip` 装到 Python313，导致 import 失败；用 `C:\Users\cqm87\AppData\Local\Programs\Python\Python313\python.exe` 全路径运行。HuggingFace 被墙时设 `HF_ENDPOINT=https://hf-mirror.com`；faster-whisper 1.2.1 需配 `av==13.1.0`（av 19 移除了 metadata_errors 参数）。
17. 听力填空若 Scripts 按钮 disabled，可下载对应 mp3 用 faster-whisper 本地转写拿原文；音频 URL 在 `audio.src`，用 `bu.download` 下载。
18. **每个标签（tab）通常要单独提交，不是整个活动一次提交。** 目录里每个图标对应一个标签，passstyle 按标签独立变蓝。做完一个标签就在该标签末页点"提交"，再"继续学习"跳下一个；回目录核对每个图标都变蓝，漏了就补提该标签。
19. **提交按钮坐标点击/合成 MouseEvent 无反应时，直接用 JS `b.click()` 最可靠**（无 toast 无弹窗但按钮可命中、未 disabled 的情况）。即 `[...document.querySelectorAll('button')].find(...).click()`，确认弹窗随即出现。
20. **两步 setter 填的值翻页/切标签后仍可能丢（React 状态没记），改用真实键盘**：`focus`→`Ctrl+A`→`Backspace`→`press_key` 逐字符→聚焦下一框触发真实失焦。填完做翻页往返验证值保留。
21. **长视频一旦在 visible 时开始播放，之后页面变 hidden 仍会继续推进到 ended**（开始播放那一下必须 visible）。期间用 `currentTime/ended` 轮询即可，不必全程前台。
22. **CDN 视频/资源浏览器外直接访问 404（有 CDN 保护）**：在页面内用同步 XHR（`overrideMimeType('text/plain; charset=x-user-defined')`，不能设 responseType）读完整内容→按 charCode 构造 Uint8Array→Blob→`createObjectURL`→a 标签触发下载，再 `bu.wait_for_download` 拿完整文件。
23. **切标签会让已填答案从 DOM 清空**（React 状态不跨标签/跨页保留），切回时框全空。做单个标签时别中途切走，用真实键盘填完该标签各页后立刻提交；若必须核对其它标签，切回后要重填。
24. **单选题直接 DOM click radio**：`document.querySelector('input[name="single-N"][value="X"]').click()`（N 从1开始），比坐标点选项可靠；作答后用 `input[name="single-N"]:checked` 回读校验。
25. **选词/搭配题注意变形**：名词可数时随 many/are 变复数（social→social services）、动词随主语三单（set→sets）、进行时（do→doing）、名词作定语（incubate→incubation，business incubation program）。先判空格所需词性再选形。
26. **`bu.js` 输出大对象/多行时常截断或干脆回空**（只回 "Computer-use Python code completed"）。回显要极简：单值或数组 `.join('|')`；连续两次空回显就 Wait 几秒再重试，或改用截图。
27. **提交结果弹窗的"继续学习"会自动链到同活动下一标签**，可顺势连做连提，省回目录导航；"查看答案"可看对错。结果文案读"平均分为N / 答对N道"判断。
28. **回目录核对完成度**：按 `li.innerText` 前缀（如 "4-9"）定位行，数该行含 `i` 的 `a`（每个 a=一个活动），看每个 a 内是否有 `i` 带 `passstyle`；全 1 串即该小节完成。
29. **Banked cloze 选词题**：从词框按搭配/语义选，每词只用一次，答案填选项字母（A–O），一次性提交整页；客观题首次即锁定。
