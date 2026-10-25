# 华大万事通 · AI学长

> CCNU 1024 Hackathon 2026 · **Agent 应用赛道**
> 一个会自己调工具的校园 AI 学长 —— **模型决定调什么，代码只负责让它调得到、拦得住、看得见**。

**在线 Demo**：<DEMO_URL>
**演示视频**：`preview/演示视频.mp4`（3 分 23 秒）
**提交版本**：v5.0 ｜ 团队：<队名>

---

## 一分钟看懂

页面里没有一张「问句 → 答案」的映射表。

用户说一句话，模型（通义 / DeepSeek / 智谱 / 硅基流动，OpenAI 兼容接口）拿到的是 **18 个工具的 JSON Schema**，它自己决定：调哪个、参数填什么、要不要连着调好几个、什么时候可以答了。代码只做四件事：

1. 把工具清单交给模型；
2. 执行模型点名要调的工具，把结果原样塞回对话；
3. 在执行前过一道**安全网关**——红线不写在提示词里，写在真正执行的位置；
4. 把每一轮 `tool_calls` 画在气泡下方，让「它在想什么」看得见。

```
用户：明天下午找个能坐 30 人、有插座的自习室
   ↓ 模型返回 tool_calls: classroom_query({date:"2026-10-25", slot:"13:00-18:00", capacity:30, hasSocket:true})
   ↓ SAFETY_GATE 放行 → 执行 → {rooms:[…], total:3}
   ↓ 模型读工具结果，组织答案
学长：七号楼 7201（40 人）和九号楼 905（36 人）都空着，插座在靠窗那排……
      [执行轨迹 · 1 次工具调用]  classroom_query  {...}  38ms
```

这就是 Agent 与「聊天框套壳」的区别：**判断权在模型，证据在页面**。

---

## 截图

| | |
| --- | --- |
| ![执行轨迹](preview/17-执行轨迹.png) | ![安全拦截](preview/19-安全拦截.png) |
| ![填 Key](preview/18-填Key弹层.png) | ![数据从哪来](preview/20-数据来源.png) |

上排左：模型自己调 `classroom_query`，轨迹里能看到参数、命中条数和耗时。
上排右：越界调用在执行前被拦下（`card_query {"scope":"other"}` → 不能查他人隐私），模型改口给办理路径。

更多见 `preview/`（20 张）。

---

## 怎么跑

### 1. 只想看看界面（不需要任何东西）

双击 `index.html`。所有页面、图表、办事指南、通知全文都能看。此时对话处于**离线直连**模式：它只做关键词匹配，页面上会明写「离线直连 · 关键词」，不会假装自己是模型。

### 2. 让它真的思考（30 秒）

1. 打开页面 → 左下角「设置」→「接个真模型」；
2. 选服务商 → 点「去拿 Key」→ 粘贴 → 保存（**只存在你这台浏览器的 localStorage，不上传任何服务器**）；
3. 右上角变成「已接上 qwen-plus」，随便问一句试试。

| 服务商 | 模型 | 浏览器直连 | 备注 |
| --- | --- | --- | --- |
| 通义千问（默认） | `qwen-plus` | ✅ 可直接调 | 有免费额度，响应里带 `Access-Control-Allow-Origin: *` |
| DeepSeek | `deepseek-chat` | ✅ | 回显 Origin |
| 硅基流动 | `Qwen/Qwen2.5-7B-Instruct` | ✅ | 有免费模型 |
| 智谱 GLM | `glm-4-flash` | ❌ 需代理 | 智谱响应不带 CORS 头，浏览器直连会被拦；用下面的 `dev-server.js` 走 `/api/llm` |

智谱的 CORS 限制是用 `curl -X OPTIONS` 实测出来的（HTTP 200，但没有 `Access-Control-Allow-Origin`），不是猜的。所以页面默认选通义，并给智谱准备了本地代理。

### 3. 本地服务（推荐演示时用）

```bash
node dev-server.js          # → http://localhost:8080
PORT=9000 node dev-server.js
```

它做两件事：静态托管（让 `data/*.json` 能被 `fetch` 到），以及把 `/api/llm` 转发到智谱。代理**不保存 Key**，只是把浏览器带来的 `Authorization` 头转给上游。

---

## Agent 是怎么工作的

```
agentLoop(userText)
  messages = [systemPrompt] + 最近 12 轮 + 本轮
  for step in 1..6:
      resp = callLLM({ messages, tools: TOOL_SCHEMAS })     ← 模型在这里做决定
      if resp 没有 tool_calls:  return 答案 + trace          ← 模型认为可以答了
      for tc in resp.tool_calls:
          if SAFETY_GATE.check(tc): trace.push(blocked); 把拒绝理由回给模型
          else: result = TOOLS[tc.name].execute(tc.args);  trace.push(done)
          messages.push({ role:'tool', tool_call_id, content: result })
  超过 6 轮还没答 → 带着已有结果收口
```

### 18 个工具

| 域 | 工具 | 模型什么时候会用 |
| --- | --- | --- |
| 空间 | `classroom_query` `classroom_book` | 问自习、开会、找空教室、占教室 |
| 信息 | `notice_search` `notice_read` `feed_search` | 问通知、政策、招标、校园新闻 |
| 生活 | `canteen_query` `library_query` `bus_query` `weather_read` | 吃啥、有没有座、校车几点、要不要带伞 |
| 个人 | `card_query` `todo_read` `calendar_write` `schedule_read` | 余额、待办、加提醒、课表 |
| 事务 | `howto_guide` `doc_draft` `event_read` `stats_read` `clock_read` | 办事流程、文书骨架、倒计时、当前时间 |
| 元数据 | `data_source` | 「这数据哪来的 / 准不准」，给出来源、日期与模拟字段 |

工具只返回**结构化数据**，不返回写死的句子。答案里的每一句人话，都是模型读完数据自己组织的。回归测试里专门有一条：工具源码里不准出现「宝子」「冲鸭」这类文案。

`weather_read` 是真公网调用（`api.open-meteo.com`，无需 Key），断网时它会失败，模型会照实说查不到——这比编一个 26℃ 诚实。

### 安全网关（执行前拦截）

```js
const RED_LINES = [
  /代(写|做|考|签|刷课)|替我(考|写|做|签|上课|答到)/,     // 学术诚信
  /(别人|他人|室友|同学)的(成绩|消费|流水|隐私|身份证)/,   // 他人隐私
  /(导出|发给).{0,10}(成绩单|成绩|消费流水|身份证)/,       // 数据外带
  /(帮(我|他|她)).{0,6}(交|缴)(学费|住宿费|报名费|罚款)/     // 资金操作
];
```

关键在位置：它拦在 `TOOLS[name].execute()` **之前**，不在提示词里。提示词可以被绕过、被注入、被模型忘了；执行前的判断不会。被拦的调用会作为一条 `blocked` 轨迹显示出来（🛑 + 理由），模型收到拒绝理由后改用合规路径回答（「这个需要你本人在办事大厅办理，带上……」）。

演示脚本里有一条固定动作：问「帮我把成绩单导出发给导师」，让评委看到 🛑。

### 记忆

工具返回的结果会以 `role: 'tool'` 留在消息里，下一轮追问直接带着走。

> 「明天下午有空教室吗」→ `classroom_query({date:'2026-10-25'})`
> 「那七号楼呢」→ `classroom_query({date:'2026-10-25', building:'七号楼'})`，`capacity` 等上下文被模型自己补回

### 无 Key / 断网时

不报错、不假装。降级为**离线直连**：关键词命中几个最常用的工具，答案上方明写「离线直连 · 关键词」，设置页写着「接上模型后才是真 Agent」。评委不填 Key 也能看到界面和数据，填了才能看到真正的工具调用循环——这一点我们写在页面上，不藏。

---

## 评委怎么验证（2 分钟）

| # | 动作 | 应该看到 |
| --- | --- | --- |
| 1 | F12 → Network，问「找个 30 人的空教室」 | 一条 `chat/completions` 请求，请求体里有 `tools` 数组；响应里有 `tool_calls` |
| 2 | 同上 | 请求体第二条消息 `role: "tool"`，内容是教室数据 |
| 3 | 问「明天下午…」再问「那七号楼呢」 | 第二次请求携带上一轮工具结果，第二次 `tool_calls` 带上 `building` |
| 4 | 问「把成绩单导出发给导师」 | 轨迹里出现 🛑 `blocked`，回答给办理路径 |
| 5 | 问「武汉今天要带伞吗」 | 请求体里 `weather_read`，Network 里有 `open-meteo.com` |
| 6 | 点「清掉 Key」再问一句 | 状态变「离线直连 · 关键词」，不冒充模型 |
| 7 | 问「这些数据是哪来的 / 准不准」 | 调用 `data_source`，列出出处、日期与哪些字段是模拟 |
| 8 | 刷新页面 | Key 与对话历史还在（localStorage），页面里搜不到任何硬编码 Key |

---

## 技术栈

| 层 | 选型 |
| --- | --- |
| 结构 / 样式 | 单文件 HTML5 + Tailwind（`vendor/` 本地优先，CDN 兜底） |
| 交互 | 原生 ES6，无框架、无构建 |
| 图表 | ECharts 5.5.1（本地 `vendor/`） |
| 图标 | Lucide |
| Agent | OpenAI 兼容 `tools` / `tool_calls` 循环，最多 6 步 |
| 数据 | `data/*.json`（HTTP 部署时 fetch）+ `mockData` 内嵌兜底（`file://` 双击可用） |
| 部署 | 静态托管（Netlify / GitHub Pages），零服务端；可选 Node 代理仅用于智谱 |

**为什么还是单文件**：评委双击就能跑、断网也能看、不用 `npm install`。模块化在这里换不来任何评分，换来的是一堆构建步骤。代码内部已经按「数据层 / 工具层 / Agent 循环 / 渲染层」分块，注释标了序号。

---

## 本地运行

```bash
# 方式一：双击 index.html（离线模式，界面全可用）
# 方式二：静态服务（数据走 data/*.json）
python3 -m http.server 8000     # → http://localhost:8000
# 方式三：带智谱代理
node dev-server.js              # → http://localhost:8080
```

---

## 数据有多真

这一版把能量化的都换成了**能核对的口径**，查不到的就老实标成模拟。每个工具返回都带 `meta`：

```json
{"kind":"official","source":"后勤保障部《食堂服务》","url":"https://houqin.ccnu.edu.cn/fwzn/stfw.htm",
 "asOf":"2026-10-24","simulated":["queue 排队指数","people 排队人数"]}
```

| 领域 | 真实部分（有出处） | 模拟部分（页面标明） |
| --- | --- | --- |
| 食堂 | 8 家食堂的名称、楼层、面积、座位、**营业时间**、订餐电话，取自[后勤保障部《食堂服务》](https://houqin.ccnu.edu.cn/fwzn/stfw.htm) | 排队指数、排队人数 |
| 图书馆 | 总馆 8:00-22:00（**周五 14:00 闭馆**）、借阅册数与天数、逾期费 0.1 元/册/天、古籍室开放时段，取自[图书馆](https://lib.ccnu.edu.cn) | 各层剩余座位 |
| 校车 | 摆渡车 7:00-21:30、1 元/人次刷一卡通、高峰 5 分钟一班、南湖食堂→南湖东门→桂子山南门→博物馆，取自[长江日报报道](http://news.cjn.cn/whpd/yw_19947/202605/t5290792.htm) | 下一班到站分钟数 |
| 教学楼 | 七号楼公共教学、八号楼文科大楼与智慧教室、九号楼计算机等学院、三号楼南楼为教室，取自[桂子山教学楼图鉴](https://m.sohu.com/a/271725818_100005812/) | 具体教室占用状态 |
| 办事 | 校园卡管理中心（田家炳楼 8 楼，027-67868564）、挂失自助电话 67868484、补卡工本费 20 元、水电报修 027-67861703、南湖综合楼自助补卡 8:30-17:30，取自[信息化办公室](https://bwc.ccnu.edu.cn/xxhbgs2024/ITfw/xykfw.htm)与后勤指南 | — |
| 一卡通余额 / 待办 / 课表 | — | 全部（真实数据需要本人登录，我们不去碰） |
| 天气 | open-meteo 实时，无 Key | — |

页面「设置 → 数据从哪来」把这张表直接摆在用户面前；轨迹里每次调用也会打上 `官方 / 混合 / 实时 / 模拟` 标签。模型被要求在回答里区分口径——模拟数据必须说「演示数据」，官方数据要带出处和日期。

> 为什么不全用真实数据？空教室占用、食堂排队、一卡通余额在学校的教务与一卡通系统里，需要本人登录，没有公开接口。把模拟值包装成「实时」是最容易翻车的做法，所以我们把它标出来。

## 质量保障

两套浏览器内回归，全部 async，直接跑在真实页面上：

```bash
agent-browser open "file://$PWD/index.html"
agent-browser eval "$(cat tools/回归测试/t_full.js)"    # 107 项：数据自洽 / 渲染 / 交互 / Agent 循环 / 安全 / 降级
agent-browser eval "$(cat tools/回归测试/t_tools.js)"   # 53 项：18 个工具的声明、执行、筛选、真实数据口径、来源标记
```

当前结果：**107 / 107**、**33 / 33**。

`t_full.js` 的 I 段用 mock 模型（`window.__MOCK_LLM__`）跑完整循环：验证工具 Schema 确实发给了模型、第二轮确实带上了 `role: "tool"`、工具结果确实进了下一轮上下文、安全网关确实拦下了越界调用。

---

## 目录结构

```
.
├── index.html              # 全部产品（数据 / 工具 / Agent 循环 / 渲染）
├── data/                   # 工具数据接口（classrooms / notices / feed / courses / canteen / library / bus / howto）
├── dev-server.js           # 本地静态服务 + 智谱 /api/llm 代理
├── vendor/                 # Tailwind / ECharts / Lucide 本地副本
├── assets/                 # 图标与静态资源
├── preview/                # 截图 + 演示视频
├── tools/
│   ├── 回归测试/            # t_full.js / t_tools.js
│   └── 演示录制/            # 截图与录像脚本
├── 交付计划.md              # 完整设计说明
└── 黑客松提交执行手册.md     # 提交时间表与检查清单
```

---

## 边界

| 能直接干 | 先问你 | 不干 |
| --- | --- | --- |
| 查空教室、课表、通知、食堂、图书馆、校车、天气、算倒计时、给办事流程 | 涉及你的成绩、课表变动、要替你提交的表单 | 代写论文、替考、导出成绩发给别人、代缴费用、查他人隐私 |

越界请求不会硬拒：安全网关拦下 → 模型收到理由 → 给合规路径（去哪儿办、带什么、多久）。

---

## 团队

| 角色 | 成员 |
| --- | --- |
| <角色 1> | <姓名 1> |
| <角色 2> | <姓名 2> |
| <角色 3> | <姓名 3> |
| <角色 4> | <姓名 4> |

---

## License

MIT
