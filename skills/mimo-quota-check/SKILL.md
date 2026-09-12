---
name: mimo-quota-check
description: >-
  用内置浏览器巡检小米 MiMo 开放平台的 Token Plan 套餐余量，产出结构化结果并按告警规则决定是否提醒。
  Use when 需要查询、巡检或定期监控 MiMo 配额、套餐额度、余额或重置时间，或要为此建定时巡检。
version: 1.0.0
type: procedural
risk_level: low
status: enabled
tags:
  - mimo
  - quota
  - browser
  - monitoring
metadata:
  author: mimo-model-quota-monitor
  updated_at: '2026-09-12'
---

# MiMo 配额巡检 SOP

站点事实（域名分工、接口白/黑名单、登录表单选择器、时间换算）全部记在
site-pattern 记忆 `platform.xiaomimimo.com` 里，执行前先读它，**不要在这里重复一份**——两份会漂移。

本技能目录下的 `references/` 有两份结构定义，按需 `Read`（路径 = 本 SKILL.md 所在目录 + 下列相对路径）：

| 文件 | 用途 |
|---|---|
| `references/result.schema.json` | 每轮巡检结果（`latest.json` 与 `history/` 下每个文件）的字段定义 |
| `references/accounts.schema.json` | 账号 ↔ 浏览器 space 映射表 `accounts.json` 的字段定义 |

工作目录里的数据布局（相对 Agent 的工作目录）：

```
mimo-quota/
├── accounts.json                 账号 ↔ space 映射表（唯一真源）
└── <credentialId>/
    ├── latest.json               最近一轮的最终结果（每轮最后一步覆盖写）
    └── history/
        └── <采集时刻>.json        每轮一份，如 20260912T083105+0800.json
```

## 先分清本轮是哪种运行

- **定时运行（无人值守）**：由调度触发，提示里写明无人值守，或系统提示里有「无人值守运行」规则。没有人会回答提问。
- **对话**：用户在对话里让你查。用户在场，可以请用户处理登录验证，再接着做。

下文凡是两种运行做法不同的地方都会分开写。

## 硬性边界（每次执行都适用）

1. **只读**。只允许对白名单路径发 GET。绝不访问
   `purchase` / `subscription/order` / `deductRedirect` / `cancel` /
   `apiKey/reset` / `apiKey/raw` / `logout`——它们会花钱或造成破坏。
2. **遇到人机验证立即停止**，请求人工介入。**严禁尝试绕过。**
3. **不推测、不编造**。拿不到就报拿不到。所有差值、百分比都用 `MathCalc` 算；时刻之间的小时数与时区换算照抄下方「时间计算」一节的表达式交给 `MathCalc`，不要心算。
4. 落盘文件**不得**包含凭据、Cookie、完整 API Key 或完整手机号。
5. 账号密码只以 `{{secrets.<id>.username}}` / `{{secrets.<id>.password}}` 占位符形式写进 BrowserAct，**不用 ManageCredentials get 取明文**（用 list 找 id 就够了）。
6. **定时运行里没人回答提问。** 不要问「需要我继续吗」「是否要……」，按本 SOP 一口气走完；
   确需人工介入的情况按下文落 `blocked` 再结束。
7. **平台报错不是 space 坏了。** 任何 `BROWSER_*` 平台错误都**不得**导致新建 space 或修改 `accounts.json`，
   落 `blocked` 后跳到第 7 步（见「失败分支」）。曾有一次浏览器平台故障期间，因为把平台错误当成 space 损坏而每轮新建 space，
   攒出 30 多个没有登录态的空 space，还把账号映射指到了空环境上。
8. **只做巡检这一件事。** 本任务不需要创建脚本、安装依赖、访问 MiMo 以外的站点或别的目录。
   发现自己正在做与配额巡检无关的事（写代码、建测试、截图、处理「用户的其他请求」），立即停下回到本 SOP。
9. **本任务不执行任何命令。** 不调用 `Bash`、`PowerShell`、终端类工具，也不用 `BrowserScript`：读文件用 `Read`，写文件用 `Write`，计算用 `MathCalc`，浏览器只用 `BrowserManage` / `BrowserAct` / `BrowserSnapshot`。
   想执行命令就说明跑偏了，立即停下回到本 SOP。真机上发生过模型巡检做完后凭空声称「换了新模型」，
   接着执行命令启动带调试端口的浏览器进程、探测本机端口、列出进程。
10. **不要写站点经验。** 定时运行里不调用 `SitePatternWrite`；对话里只在用户明确要求时才写。
    写入内容只要出现 `cookie` / `sessionid` / `csrf` / `token`（包括「Token Plan」）/ `bearer` / `jwt` 这些英文单词，
    或手机号、邮箱，就会被自动降级写到用户私有层，而读取时用户层优先，这份片段会遮住整份站点经验。
11. **写文件被拒就停。** `Write` 被审批拒绝时，不换工具、不换路径重试；本轮开过会话的仍按第 9 步关闭，然后把原因写进最后回复，本轮结束。

巡检结果只写第 7、10 步的文件。平台要求的 Plan 文件照平台规则处理，它不算巡检结果文件。

## 流程

### 1. 激活浏览器能力
先 `Skill(skill='web-access')`。站点经验工具 `SitePatternRead` 默认隐藏，激活后才可用。
web-access 技能里「发现新陷阱就写站点经验」的要求，在本任务里以上面第 10 条为准。

### 2. 确定本轮账号，读上一轮结果，选定 space

**本 SOP 一轮只巡检一个账号。** 先 `ManageCredentials action='list'`，然后：

- **调度或用户指定了 `credentialId`**（账号库里该账号条目的 id）→ 在 `list` 结果里逐字找 `id` 等于它、且 `site` 含 `xiaomimimo` 的条目。
  找不到：
  - **对话里**：直接告诉用户账号库里没有这个 id（可能是打错了），请到 资源 → 算力 → 账号 里核对，站点 / 服务要填 platform.xiaomimimo.com。**不落盘**，本轮到此为止。
  - **定时运行里**：落 `blocked.reason='credential_missing'`，`detail` 写明上面这句话，结果写在 `mimo-quota/<指定的 id>/` 下。
    **不建 space、不开会话**，读完上一轮结果（见下）就跳到第 7 步。
- **没有指定**：
  - `site` 含 `xiaomimimo` 的条目**恰好一个** → 用它；
  - 一个都没有、或多于一个 → 对话里直接告诉用户（多个时问要巡检哪个），**不落盘**，本轮到此为止；
    定时运行的 prompt 必须指定账号，没指定就只在最后回复里说明原因，不落盘。**不要自己遍历全部账号。**

❌ 绝不凭印象编造 id，也不照抄本文档里的示例。曾发生过编造一个库里根本没有的 id，导致每步 `BROWSER_SECRET_NOT_CONFIGURED`、整轮空转的事故。

**读上一轮结果（必须在本轮写 `mimo-quota/` 下任何文件之前）**：`Read mimo-quota/<credentialId>/latest.json`。
- 文件不存在 → 本账号第一次巡检，没有上一轮；
- 读到了、是合法 JSON → 记住它的 `outcome`、`blocked.reason`、`alerts` 和 `lastOk`，第 4b、7 步要用；
- 读取报错（不是「文件不存在」），或内容不是合法 JSON → 不要当成没有上一轮。本轮照常巡检，第 7 步把所有成立的告警项都当新出现（宁可多提醒一次），并在汇报里说明读取失败。

⚠️ **一个账号必须独占一个 persistent space。** 两个小米账号的 Cookie 同处一个浏览器分区时会互相覆盖，
后登录的会把先登录的挤掉——这不是策略选择，是浏览器的既定行为。

然后 `SitePatternRead` 读 `platform.xiaomimimo.com` 的站点经验，再 `Read` 读 `mimo-quota/accounts.json`。
按本轮 `credentialId` 查出 `spaceId`，`BrowserManage(action='list_spaces')` 后**按那个 id 选**。
**不要按名字取第一个**——多个账号的 space 可能同名，选错就会拿到别的账号的登录态
（更糟：拿到没登录态的那个，白白重登一次并卡在短信验证）。

`accounts.json` 里没有本轮账号，或记的 id 不在 `list_spaces` 里：
- **定时运行里**：不建 space。落 `blocked.reason='space_missing'`，`detail` 写明「该账号还没有可用的浏览器环境，请在对话里让本 Agent 查一次」，跳到第 7 步。
  `list_spaces` 只列出本 Agent 有权访问的 space，授权被撤销时看起来也像「不在列表里」，这种情况必须由人确认。
- **对话里**：`BrowserManage(action='create_space', name='mimo-quota-<credentialId>', persistence='persistent')`，
  并立刻按 `references/accounts.schema.json` 写入或更新 `accounts.json` 里**对应那一行**（文件不存在就新建，
  `phoneTail` 只写后 4 位，不知道就不写）。创建 space 需要审批；没通过就告诉用户，不落盘。

**不要**因为某个 space 报错就另建 space——`start_session` 失败、`BROWSER_PROVIDER_UNAVAILABLE`、
`journal is not ready` 这类都是平台问题，落 `blocked` 后跳到第 7 步。

> **必须在主对话上下文里自己创建，不得委派子智能体。**
> Grant 主体按 `run → job → team → agent` 取最具体的一个；子智能体创建会让 Grant
> 变成 run 级，下次运行看不见该 space（`BROWSER_TOOL_SPACE_FORBIDDEN`），登录态在磁盘上却永远够不着。

### 3. 开会话
`BrowserManage(action='start_session', spaceId=<id>, url='https://platform.xiaomimimo.com/console/plan-manage')`

> **开了就得自己关**（见第 9 步）。平台不会因为本轮结束就替你关会话：租约 TTL 是 1 小时，
> 期间这个 space 上的**下一轮巡检会被你自己这条没关的会话挡住**，报
> `BROWSER_SPACE_LEASE_HELD_BY_SAME_PRINCIPAL`，白等一小时。

### 读接口的唯一正确姿势（第 4、5 步都用它）

**不要用 `BrowserAct action='page.evaluate'`。** `browser.page.evaluate` 属于必须经过审批的高危浏览器能力，
定时运行容易卡在审批上；而下面的导航与读文本都不需要审批。

`/api/v1/*` 返回 JSON，浏览器当纯文本页面渲染，所以「读接口」＝「导航 + 读文本」：

```
BrowserAct      action='tab.navigate'  params.url='<接口完整 URL>'
BrowserSnapshot mode='text'            # 返回的整页文本就是 JSON
```

Cookie 由持久化 space 自动携带，**不要手工设置任何 header**。
同一 tab 换 url 重复两步即可依次取完各接口。**每次导航后都要紧跟一次读文本**，不要连续导航几次再回头读。

### 4. 登录态自检（确定性判定）
按上述两步读 `https://platform.xiaomimimo.com/api/v1/userProfile`：
- `code === 0` → 继续第 5 步
- `code === 401` → 走第 4b 步。响应体自带 `loginUrl`。

**不要靠猜页面文本判断是否登录。** 该接口其余字段是 PII，只在内存判定，不落盘、不进汇报。

### 4b. 未登录时：先免填表恢复，再用账号库登录

**先免填表试一次**：`BrowserAct action='tab.navigate' params.url=<401 响应里的 loginUrl>`，再按「读接口」两步读一次
`userProfile`。分区里的小米 Cookie 链常能经 sts 免填表恢复登录（实测多次如此）；
`code === 0` 就回到第 5 步取数。中间的 sts 页正文可能是空的，不要在它上面反复快照。仍是 401 才考虑填表。

**定时运行里，上一轮已经是 `human_verification` 或 `not_logged_in` 时不再填表**：`blocked.reason` **沿用上一轮的原因**（改成别的会被当成新情况再提醒一次），
`detail` 写明「上一轮起就需要人工登录，本轮只尝试了免填表恢复」，跳到第 7 步。每小时重复填表会触发小米风控。

否则用第 2 步核对过的 `credentialId` 自己登录，**整轮最多填一次表**。
**不要因为本轮账号登录不上就改用另一个账号**——那会把两个账号的登录态搅进同一个 space。

**填字段一律用 `page.element{op:'fill'}`，不要用 `input.text`**——
`input.text` 走 CDP `Input.insertText`，只插字符、不触发页面（Ant Design）的 React onChange，
表单校验不过、登录按钮不激活；`page.element{op:'fill'}` 用原生 value setter + 派发 input/change 事件，
是能驱动受控输入的方式，且 `value` 支持 `{{secrets.*}}` 占位符（明文不进上下文、不进会话记录）：

```
BrowserAct action='tab.navigate'   params.url=<401 响应里的 loginUrl>
# 填账号（CSS 选择器不需要 snapshotId）
BrowserAct action='page.element'   params.op='fill'  params.selector='loc=css:input[name="account"]'                     params.value='{{secrets.<id>.username}}'
# 填密码
BrowserAct action='page.element'   params.op='fill'  params.selector='loc=css:input[type="password"][name="password"]'  params.value='{{secrets.<id>.password}}'
# 勾选用户协议（必须先勾，否则登录按钮不激活）；check 是 page.element 的写 op
BrowserAct action='page.element'   params.op='check' params.selector='loc=css:.ant-checkbox-input'
# 点登录：input.click 只认 ref 或坐标（不认 selector），先取语义快照拿到提交按钮的 e<序号>
BrowserSnapshot                                              # 默认 mode='semantic'，回执含 elements 与 snapshotId
BrowserAct action='input.click'    params.ref='e<提交按钮序号>'  params.snapshotId=<该回执的 snapshotId>  params.wait={until:'networkidle'}
# 读当前页面，按下表判定
BrowserSnapshot mode='text'
```

**提交之后留在当前页面判定，不要马上导航去读 `userProfile`**——导航会把验证页和报错原话冲掉，既判断不了分支，
用户打开浏览器面板也看不到验证页。

| 当前页面 | 判定 | 下一步 |
|---|---|---|
| 弹出「友情提示」对话框，有「同意并继续」 | 小米的协议确认，正常流程 | 取语义快照，`input.click` 点「同意并继续」并同样带 `params.wait={until:'networkidle'}`，再读当前页面文本，重新对照本表 |
| 「小米账号安全验证」、`verifyPhone`、滑块、图形码、短信验证码 | 人机验证 | 见下方「人机验证」 |
| 仍是登录表单，页面上有报错（密码错误、账号被锁等） | 登录失败 | 落 `blocked.reason='not_logged_in'`，`detail` 逐字照抄报错，跳到第 7 步 |
| 已经离开 `account.xiaomi.com`，回到 `platform.xiaomimimo.com` | 登录完成 | 按「读接口」两步读 `userProfile`：`code === 0` 进入第 5 步；仍 401 落 `not_logged_in`，跳到第 7 步 |
| 以上都不是（正文为空、跳转中间页、仍是登录表单但没有报错） | 还在跳转 | 等一次跳转：`BrowserAct action='page.wait' params={until:'url', url:'https://platform.xiaomimimo.com/**', timeoutMs:15000}`，再读一次当前页面文本，按上面四行判定。只等这一次；仍判断不了落 `not_logged_in`，`detail` 写「提交后没有跳转」，跳到第 7 步 |

> 若 `input.click` 返回 `BROWSER_COMMAND_OUTCOME_UNKNOWN`，**不要重试同一条命令**，落 `blocked.reason='outcome_unknown'` 后跳到第 7 步。

**❗提交之后禁止使用 `Sleep` 等跳转**，用上面的 `params.wait` 与读当前页面文本代替。
`Sleep` 那一轮不产生任何新信息，模型醒来时手里和睡前一样，极易就此偏离 SOP。
曾有真机运行在填表、勾选、点登录、点「同意并继续」全部成功、页面已跳到 `verifyPhone` 之后，
模型 `Sleep` 醒来输出一句无意义文本就结束了整轮，**既没继续判定，也没落盘**，调度却记为成功。

**人机验证**（`blocked.reason='human_verification'`）：**立即停止操作页面，严禁尝试绕过。** 小米对新浏览器环境强制短信验证，只能真人完成。
- **对话里**：**不关会话、不落盘**。回复用户：在浏览器面板里点「发送验证码」并输入，完成后点面板上的「恢复 Agent」，再回复你。
  **回复完就停下等用户，不执行第 5–10 步。** 用户回复后：
  - 先重新 `Read` 一次 `mimo-quota/<credentialId>/latest.json` 作为上一轮结果（用户可能隔了很久才回复，期间定时巡检可能已经写过）；
  - 会话还在 → 按「读接口」两步读 `userProfile`；
  - 会话已失效（`BROWSER_TOOL_SESSION_TERMINATED` 等）→ 回到第 3 步重开会话再读；
  - `code === 0` → 进入第 5 步继续；仍是 401 → 落 `human_verification`，进入第 7 步。
- **定时运行里**：`detail` 写明「需要真人完成短信验证：在对话里让本 Agent 用 <credentialId> 查一次，按提示在浏览器面板完成验证」，进入第 7 步。

### 5. 取数
按同样两步依次读：`/api/v1/tokenPlan/detail` 取套餐档位与到期时间（它**不含额度数字**），`/tokenPlan/usage` 取额度数字，`/balance` 取余额。
**不要假设字段名，按实际返回的 JSON 取。**

- `detail` 与 `usage` 都读到、且返回里有总量与已用 → `source = 'api'`
- `/balance` 读不到只影响 `wallet`：整个不写 `wallet`，不回落页面
- `detail` 或 `usage` 读不到，或返回里找不到总量与已用 → 回落 `BrowserSnapshot(mode='text')` 读 `/console/plan-manage`（页面上有「已用 / 总量」），`source = 'dom'`

### 6. 校验
- `creditsRemaining === creditsTotal - creditsUsed`
- `remainingPercent ∈ [0, 100]`
- `resetAtUtc` 可解析且在未来

任一不过 → `dataQuality = 'suspect'`，在汇报里说明疑点，**不得静默采用**。

### 7. 算出本轮结果，写入本轮历史文件（**每条路径都要做，阻塞也不例外**）

**本轮已经写过 history 文件就不要重做这一步**（例如平台续跑、或你忘了自己做到哪）：沿用那份结果，直接从第 8 步往下补。

1. `collectedAt` 取提示里的当前本地时间：`Today's date` 的日期加上 `Local time` 的时分秒与时区，写成 `2026-09-12T08:31:05+08:00`
   （续跑或用户回复后，用平台消息里给的当前本地时间）。时区 `+0800` 写成 `+08:00`。不要换算成 UTC，也不要编造秒数。
   定时运行时这个时间就是本轮触发的时刻，会比真正取到数据早几分钟；告警都按小时计算，不受影响，照写即可。
2. 本轮取到了数据、且上一轮结果里有 `lastOk` → 用 `MathCalc` 算 `deltaFromLast`：剩余量变化、剩余百分比变化、间隔小时数（用「时间计算」的小时数表达式）、每小时消耗。
3. 按下方「告警规则」算出本轮 `alerts`。
4. 定 `notify`：先求**对比基准** = 上一轮的 `alerts` ∪ 上一轮 `lastOk.alerts`（没有上一轮、或字段缺失都按空列表；第 2 步读取报错时基准为空）。
   **本轮 `alerts` 里有基准没有的项 → `notify=true`，否则 `false`。**
5. 定 `lastOk`：本轮取到了数据且 `dataQuality='ok'` → 写本轮自己的 `collectedAt`、`creditsRemaining`、`remainingPercent`、`alerts`；
   本轮阻塞或数据存疑 → 原样照抄上一轮结果里的 `lastOk`（上一轮没有就不写）。存疑的数字不能当下一轮的对比基准。
6. 按 `references/result.schema.json` 组好整份结果，`Write` 到 `mimo-quota/<credentialId>/history/<文件名>.json`。
   文件名 = `collectedAt` 去掉 `-` 和 `:`（`2026-09-12T08:31:05+08:00` → `20260912T083105+0800.json`）。

`meta.credentialId` 必填；`meta.spaceId` 在选定 space 之后才有，之前就阻塞的不写。

**阻塞同样必须落盘**——只有 8 个字段必填
（`schemaVersion` / `collectedAt` / `outcome` / `notify` / `alerts` / `source` / `dataQuality` / `meta`），阻塞时还要有 `blocked`，配额数字全部可选。
阻塞情形填 `source='none'`、`dataQuality='unavailable'`、`outcome='blocked'`，
并把卡点写进 `blocked` 对象（`reason` + `detail`，登录相关时附 `loginUrl`）。

读不到的对象（`plan` / `quota` / `wallet` / `deltaFromLast`）整个不写，**不要写 `null`**。

**绝不为了凑齐字段编造数值。** 一次「没取到，原因是 X」的记录是有价值的；
一份看着完整、数字是猜的记录会让人基于错误余量做决策。

### 8. 需要提醒时通知用户
`notify === true` 时：
- **定时运行里**：调用 `SendUserMessage(status='proactive')`。
- **对话里**：直接在回复里说，不另发 `SendUserMessage`。

第一行就是新出现的那项结论（例如「MiMo 账号 mimo-main 剩余额度 18%，已低于 20%」或「MiMo 账号 mimo-main 登录已失效，需要你处理」），
阻塞时写清需要用户做什么，其余按下方「汇报格式」。`notify === false` 时定时运行里**不发消息**。

### 9. 关闭本轮开的会话（**强制，成功失败都要做**）
`BrowserManage(action='close_session', sessionId=<本轮 start_session 拿到的 id>)`

**这一步不能省，也不能因为「反正要结束了」跳过。** 判据是「本轮开过会话就必须关」——
包括取数失败、被阻塞、页面改版等任何提前收场的路径。只关一次；已经关过就不要再关。本轮没开会话就跳过这一步。
`close_session` 本身报错时不要重试，照常进入第 10 步。

漏关的代价是确定的：租约 1 小时 TTL 内该 space 被锁死，下一轮拿到
`BROWSER_SPACE_LEASE_HELD_BY_SAME_PRINCIPAL` 直接失败，中间的巡检全部丢失。

### 10. 最后一步：写 `latest.json`（**强制，成功失败都要做**）
把第 7 步的同一份结果 `Write` 覆盖到 `mimo-quota/<credentialId>/latest.json`。**这必须是本轮最后一个写入动作，排在关闭会话之后。**

建调度时把它设成完成判据（见下方「建定时巡检」）：平台会核对本轮有没有写入它，没写就在同一会话里要求你把活干完，
续跑后仍没写则记为失败并通知用户。这是为了发现半途收场——真机上发生过模型写完结果后突然声称
「这是一次新的对话」就结束，既没关会话也没留下完整记录。

**定时运行的最后回复只写一行摘要**（例如「mimo-main：剩余 95.26%，无新告警」或「mimo-main：登录失效，已提醒」）。
这条回复会作为执行回执显示在对话里，每轮一条，写长了会刷屏。**不要回头重做前面已经完成的步骤。**

## 失败分支（必须显式处理，不许含糊过去）

所有阻塞都 `outcome='blocked'`、`source='none'`、`dataQuality='unavailable'`，从第 7 步往下走；是否提醒由「告警规则」决定。

| 情况 | 判据 | `blocked.reason` | 动作 |
|------|------|------|------|
| **账号库里没有账号** | 第 2 步在 `list` 里找不到指定的 id | `credential_missing` | 不建 space、不开会话，`detail` 写明去哪里新增。 |
| **没有可用的浏览器环境** | 定时运行里 `accounts.json` 没有本账号，或记的 id 不在 `list_spaces` 里 | `space_missing` | 不建 space，请用户到对话里查一次。 |
| **未登录** | 免填表恢复后仍 401，且自动登录失败或本轮不再填表 | `not_logged_in` | 附 `loginUrl`。**不要反复尝试。** |
| **人机验证** | 页面出现 verifyPhone / miverify / 滑块 / 图形码 / 短信验证 | `human_verification` | **立即停止，严禁绕过。** 对话里请用户处理后继续；定时运行里收尾（见第 4b 步）。 |
| **space 够不着** | `BROWSER_TOOL_SPACE_FORBIDDEN` | `space_forbidden` | 报告 Grant 主体问题。**不要重复建新 space**（会累积垃圾 space）。 |
| **会话被用户暂停** | `BROWSER_AGENT_PAUSED_BY_USER` (409) | `agent_paused` | **不要重试、不要绕过。** 对话里请用户点「恢复 Agent」后继续，不收尾；定时运行里落盘收尾。 |
| **space 被占用** | `BROWSER_SPACE_LEASE_HELD_BY_SAME_PRINCIPAL` (409) / `BROWSER_SPACE_SINGLE_WRITER_VIOLATION` (409) | `space_busy` | 错误消息里带着占用方信息，照抄进 `detail`。**不要重试、不要新建 space、不要试图「恢复」对方的会话**。 |
| **会话已终止** | `BROWSER_TOOL_SESSION_TERMINATED` (409) | `session_terminated` | 对话里回到第 3 步重开一次；定时运行里落盘，下一轮重新开会话。 |
| **浏览器平台不可用** | `BROWSER_PROVIDER_UNAVAILABLE` / `journal is not ready` / `authority is sealed` / `BROWSER_JOURNAL_INTEGRITY_FAILED` | `platform_unavailable` | **不新建 space、不改 `accounts.json`**，下一轮自然重试。 |
| **结果未知** | `*_OUTCOME_UNKNOWN` | `outcome_unknown` | **不要重放同一条命令**，按错误消息里的指引对账。 |
| **网络失败** | 导航失败 / 快照超时 | `network` | 重试**一次**；仍失败再落阻塞，记录原始错误。 |
| **页面改版** | 接口与页面都拿不到预期字段 | `page_changed` | **绝不推测数值。** |
| **其他** | 以上都不是 | `other` | `detail` 写清经过。 |

## 告警规则

每轮算出一个 `alerts` 列表，记录本轮成立的告警项：

| 告警项 | 成立条件 |
|------|------|
| `low_quota` | 剩余百分比 ≤ 20 |
| `reset_soon` | 从 `collectedAt` 到 `resetAtUtc` 不超过 72 小时（用「时间计算」的小时数表达式，`resetAtUtc` 的时区填 0） |
| `fast_burn` | 与上一次成功巡检（`lastOk`）相比，剩余百分比**下降** ≥ 10 个百分点，且 `lastOk.collectedAt` 到本轮 `collectedAt` 不超过 24 小时。上升说明额度已重置，不算 |
| `suspect` | `dataQuality === 'suspect'` |
| `blocked:<reason>` | 本轮阻塞。需要人处理的原因（`credential_missing` / `space_missing` / `not_logged_in` / `human_verification` / `space_forbidden` / `agent_paused` / `page_changed` / `other`）本轮就记；可能自行恢复的原因（`space_busy` / `session_terminated` / `platform_unavailable` / `network` / `outcome_unknown`）只有**上一轮也是同一原因阻塞**时才记 |

**`notify` 只看有没有新出现的项**：对比基准是上一轮的 `alerts` 加上最近一次成功巡检的 `alerts`（第 7 步第 4 点）。所以：
- 剩余一直在 15%，只在第一次跌破时提醒；中间偶尔阻塞一轮，恢复后也不会重复提醒；
- 额度重置回满后告警项自然消失，不会误报；
- 网络偶尔失败一轮不打扰，连续两轮才提醒。

## 建定时巡检

用户要求定时巡检时：

1. 先确认这个账号在对话里成功巡检过（`mimo-quota/<credentialId>/latest.json` 存在且 `outcome='ok'`）。没有就先在对话里查一次，把首次登录走通。
2. 调用 `ManageSchedule action='create'`，每个账号一个调度：
   - `display_name`：`MiMo 巡检 · <credentialId>`
   - `trigger_type='interval'`、`trigger_value='PT1H'`（用户另有要求时照用户说的；用 cron 时带上用户所在时区的 `timezone`）
   - `prompt`：`执行一次 MiMo Token Plan 配额巡检：调用 Skill(skill=mimo-quota-check) 并严格按其 SOP 走完。这是无人值守的定时运行，没有人会回答提问：不要询问是否继续，按 SOP 一口气走完。本轮账号：credentialId = '<credentialId>'。`
   - `completion_artifacts=['mimo-quota/<credentialId>/latest.json']`
3. 告诉用户建好了，以及出现哪些情况会提醒。

## 只查看上次结果

用户问「上次巡检结果是什么」这类问题时，只 `Read mimo-quota/<credentialId>/latest.json`，按下方「汇报格式」回答并说明采集时刻。
**不打开浏览器、不写任何文件。** 用户明确要求「现在查一次」才走上面的流程。

## 时间计算（照抄，不要心算）

`MathCalc` 不认识日期字符串，时刻要拆成数字放进 `scope`：年 `y`、月 `mo`、日 `d`、时 `h`、分 `mi`、秒 `s`、时区小时数 `tz`（`+08:00` 填 8，`+05:30` 填 5.5，`-05:00` 填 -5，UTC 填 0）。
表达式原样照抄，变量名不要改（mathjs 里 `i`、`e`、`to`、`in` 有特殊含义）。下面两条表达式都以同一个前缀开头：

```
dn(y,mo,d) = 365*(y-(mo<=2)) + floor((y-(mo<=2))/4) - floor((y-(mo<=2))/100) + floor((y-(mo<=2))/400) + floor((153*(mo+(mo>2?-3:9))+2)/5) + d - 719469;
```

**两个时刻相差多少小时**（第二个减第一个，负数表示第二个更早）。`scope` 填 `y1,mo1,d1,h1,mi1,s1,tz1,y2,mo2,d2,h2,mi2,s2,tz2`，表达式是「前缀」接上：

```
round((dn(y2,mo2,d2)*86400 + h2*3600 + mi2*60 + s2 - tz2*3600 - (dn(y1,mo1,d1)*86400 + h1*3600 + mi1*60 + s1 - tz1*3600)) / 3600, 4)
```

例：从 `2026-09-12T08:31:05+08:00` 到 `2026-10-01 23:59:59 (UTC)`，结果 `[479.4817]` 小时。

**换算到另一个时区**（用于 `resetAtBeijing`）。`scope` 填 `y,mo,d,h,mi,s,tz,tzOut`，结果依次是年、月、日、时、分、秒，表达式是「前缀」接上：

```
t = dn(y,mo,d)*86400 + h*3600 + mi*60 + s - tz*3600 + tzOut*3600; z = floor(t/86400) + 719468; era = floor(z/146097); doe = z - era*146097; yoe = floor((doe - floor(doe/1460) + floor(doe/36524) - floor(doe/146096))/365); doy = doe - (365*yoe + floor(yoe/4) - floor(yoe/100)); mp = floor((5*doy+2)/153); mon = mp < 10 ? mp+3 : mp-9; sod = t - floor(t/86400)*86400; [yoe + era*400 + (mon<=2), mon, doy - floor((153*mp+2)/5) + 1, floor(sod/3600), floor(mod(sod,3600)/60), mod(sod,60)]
```

例：`2026-09-30 23:59:59 (UTC)` 换算到北京时间（`tz=0`、`tzOut=8`），结果 `[[2026, 10, 1, 7, 59, 59]]`。

## 汇报格式

结果先行，再附依据：

```
账号：<credentialId>（手机尾号 <后 4 位>）
套餐：<planName>
剩余：<creditsRemaining> / <creditsTotal>（<remainingPercent>%）
重置：<resetAtBeijing>（北京时间）｜ <resetAtUtc>（UTC 原文）
较上次成功巡检：<deltaFromLast 摘要>
数据来源：<api | dom>｜采集时间：<collectedAt>
```

数据质量存疑时，把疑点写在第一行。
