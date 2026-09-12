---
id: site_platform_xiaomimimo_com_quota
domain: platform.xiaomimimo.com
aliases:
  - mimo.mi.com
  - account.xiaomi.com
  - token-plan-cn.xiaomimimo.com
type: site-pattern
pinned: true
confidence: high
learned_at: 2026-09-01
updated_at: 2026-09-12
---

## 本文件的维护规则（先读）

本文件只记**站点事实**：域名、路径、接口、登录流程、选择器、错误码含义。它**不是运行日志**。

- 每轮巡检的结果、平台故障、登录态观察一律写 `mimo-quota/<credentialId>/` 下的 `latest.json` 与 `history/`，**不要**用 `SitePatternWrite` 追加到这里。
- 账号 ↔ space 映射表**不在这里**，在 Agent 工作目录下的 `mimo-quota/accounts.json`（结构见 `mimo-quota-check` 技能的 `references/accounts.schema.json`）。
- **定时运行里不调用 `SitePatternWrite`；对话里只在用户明确要求时才写。** 平台按英文单词不分大小写识别敏感片段：
  `cookie` / `sessionid` / `csrf` / `xsrf` / `token`（「Token Plan」也算）/ `bearer` / `jwt`，以及手机号、邮箱。
  命中任何一个都会自动降级写到用户私有层，而读取时用户层优先，那份片段会把本文件整个遮住——写这个站的经验几乎一定会提到 Token Plan 或 Cookie，
  所以默认不写。真机上发生过：映射表里带完整手机号，每轮写回都落到用户层，攒出 500 多行运行流水，
  本文件的接口白名单、登录判定、错误码应对全部读不到了。

## 域名分工

| 域名 | 角色 |
|------|------|
| `platform.xiaomimimo.com` | **控制台**，所有配额数据在此 |
| `mimo.mi.com` | 只是产品文档/营销站，**不是控制台** |
| `account.xiaomi.com` | 小米账号统一登录 |
| `token-plan-cn.xiaomimimo.com` | Token Plan 的**推理 API 网关**，不是查配额的地方（`/user/balance` 恒 404） |

## 控制台路径

- `/console/plan-manage` —— **Token Plan 套餐：档位、额度、已用、到期时间**（主目标页）
- `/console/balance` —— 账户余额（余额 / 现金余额 / 赠送余额）
- `/console/usage` —— 用量明细

## 接口（base：`https://platform.xiaomimimo.com/api/v1`）

鉴权走**登录 Cookie**（不是 Bearer、不是 API Key），由 space 的持久化分区自动携带，**不要手工设置任何 header**。
响应信封 `{ code, data }`，`code !== 0` 视为失败。

### ✅ 本任务只 GET 这四个（与 principles 一致）

```
/userProfile        仅用于判定登录态；其余字段是 PII，只在内存里用，不落盘、不进汇报
/tokenPlan/detail   套餐元信息：planCode / planName / currentPeriodEnd（到期时间，UTC）/ expired / enableAutoRenew 等，不含额度数字
/tokenPlan/usage    套餐用量：额度数字从这里取，字段名以实际返回为准
/balance            余额 / 现金 / 赠送
```

控制台页面自己还会调 `/usage`、`/tokenPlan/subscription/status`、`/tokenPlan/list`、`/tokenPlan/price`、
`/tokenPlan/managementUrl`、`/auth/nonMainlandUserStatus` 等，**本任务不需要，不访问**。

### 🚫 绝对禁止访问（任何方法）

```
/tokenPlan/purchase               会花钱
/tokenPlan/subscription/order     会花钱
/tokenPlan/deductRedirect         会花钱
/tokenPlan/cancel                 取消订阅
/tokenPlan/apiKey/reset           毁掉专属 API Key
/tokenPlan/apiKey/raw             读出明文凭据
/logout                           毁掉登录态
/balanceAlertConfig               改用户告警配置（读写一律不碰）
```

> **为什么要专门列黑名单**：只读查询与购买、退订、重置密钥**共用同一个 base**，路径只差几个字符。
> 判据是白名单：**不在上面四个里的一律不碰**。

## ⭐ 怎么读接口（免审批做法，必须照做）

**不要用 `BrowserAct action='page.evaluate'` 去 fetch 接口。**
`browser.page.evaluate` 属于必须经过审批的高危浏览器能力，定时运行容易卡在审批上；导航与读文本都不需要审批。

`/api/v1/*` 返回 JSON，浏览器把它当纯文本页面渲染，所以「读接口」＝「导航 + 读文本」：

```
1. BrowserAct       action='tab.navigate'  params.url='https://platform.xiaomimimo.com/api/v1/userProfile'
2. BrowserSnapshot  mode='text'
      → 返回的整页文本就是那段 JSON
```

同一个 tab 里换 url 重复两步即可依次取完四个接口。

## 登录态判定（确定性，不要靠猜页面文本）

未登录时访问**任一** `/api/v1/*` 返回：

```
HTTP 401
{"code":401,"loginUrl":"https://account.xiaomi.com/pass/serviceLogin?callback=..."}
```

⇒ 固定先读 `/api/v1/userProfile`：`code === 0` 已登录；`code === 401` 未登录，响应体自带 `loginUrl`。

**401 之后先直接导航 `loginUrl` 一次，再读一次 `userProfile`。** 分区里残留的小米 Cookie 链常能让
`serviceLogin` 经 sts 免填表完成登录（多次实测）。中间页 sts 可能是空白正文，
不要在它上面反复快照判断，直接再导航 `userProfile` 看 `code`。仍是 401 且出现登录表单，才按 SOP 第 4b 步填表。

## 登录表单

### 账号从哪来（账号库）

用户在 资源 → 算力 → **账号** 面板保存的条目，结构是 `{id, label, site, username, password}`。
`ManageCredentials action='list'` 能看到 id / label / site / username（**不含密码**）。
填表时写占位符，平台在最后一刻注入真实值：

| 写法 | 取值 |
|---|---|
| `{{secrets.<id>.username}}` | 账号 |
| `{{secrets.<id>.password}}` | 密码 |

占位符只能用在 `BrowserAct page.element{op:'fill'}` 的 `value`（和 `input.text` 的 `text`）；
写到别的参数会被拒（`BROWSER_SECRET_PLACEHOLDER_UNSUPPORTED`）。
**这个站的登录表单必须用 `page.element{op:'fill'}`**——`input.text` 不触发 Ant Design 受控输入的 onChange，
表单校验不过、登录按钮不激活。

**密码登录页**：`https://account.xiaomi.com/fe/service/login/password?sid=passport&_locale=zh_CN`

| 元素 | 选择器 | 备注 |
|------|--------|------|
| 账号 | `input[name="account"]` | 邮箱 / 手机号 / 小米 ID |
| 密码 | `input[type="password"][name="password"]` | |
| 协议勾选 | `.ant-checkbox-input` | **必须先勾**，否则登录按钮不激活 |
| 提交 | `button[type="submit"]`（文本「登录」） | |

**短信登录**：`input[name="phone"]`、`input[name="ticket"]`，验证码倒计时约 60s。

⚠️ 页面加载了 `miverify` 人机验证组件，小米随时可能弹滑块或图形码。
> **红线**：遇到人机验证**立即停止并请求人工介入**，严禁尝试自动绕过。

点「登录」「同意并继续」时给 `input.click` 带上 `params.wait={until:'networkidle'}`，然后**留在当前页面**读文本判定：
出现安全验证是人机验证，表单上有报错是登录失败，回到 `platform.xiaomimimo.com/console/...` 才算登录成功。
判断不了时用 `BrowserAct action='page.wait' params={until:'url', url:'https://platform.xiaomimimo.com/**', timeoutMs:15000}` 等一次跳转再读，不要连续快照。
不要一提交就导航去读 `userProfile`——那会把验证页和报错原话冲掉。

## 时间换算

页面与接口给的到期/重置时间是 **UTC**（形如 `2026-09-02 23:59:59 (UTC)`）。
汇报时**必须同时给出 UTC 原文和换算后的北京时间**（+8h）。跨日、跨月时容易算错，按 `mimo-quota-check` 技能「时间计算」一节的表达式交给 `MathCalc`。

## 取数策略

1. **优先接口**：`/tokenPlan/detail` 取套餐档位与到期时间，`/tokenPlan/usage` 取额度数字，`/balance` 取余额
2. `/balance` 只补余额，读不到就不写余额，不影响取数来源
3. `detail` 或 `usage` 失败、或接口返回里找不到总量与已用时，回落 `BrowserSnapshot(mode='text')` 读 `/console/plan-manage`，页面上有「已用 / 总量」
   （2026-09-12 实测 `detail` 返回里没有任何额度字段；早先记录的「detail 含总量 / 已用」有误。不要假设字段名，按实际返回的 JSON 取）
4. **汇报必须标注数据来源**是 `api` 还是 `dom`

## 浏览器 space 约束

- 必须 `persistence: 'persistent'`（分区名带 `persist:` 前缀，Cookie 跨重启保留）；`ephemeral` 关闭即清空登录态
- 登录态存在磁盘 `<userData>/Partitions/desirecore-browser-<spaceId>-g1/`，**跨实例重启保留**
- **必须在主对话上下文里创建 space，不得委派子智能体**——子智能体创建会让 Grant 变成 run 级，
  下次运行报 `BROWSER_TOOL_SPACE_FORBIDDEN`（登录态在磁盘上却够不着）
- 浏览器工具默认隐藏，用前先激活 `web-access` 技能
- **一个账号独占一个 persistent space**；映射见 Agent 工作目录下的 `mimo-quota/accounts.json`，**按 id 选，永不按名字选**

## 已知错误码与应对

| 错误 | 含义 | 应对 |
|------|------|------|
| `BROWSER_PROVIDER_UNAVAILABLE`（含 `journal is not ready`、`authority is sealed`） | **浏览器平台**暂时不可用，与哪个 space 无关 | `blocked.reason='platform_unavailable'` 落盘后结束。**绝不新建 space、绝不改 accounts.json**——曾有一次平台故障期间因为把平台错误当成 space 坏了，前后新建了 30 多个没有登录态的 space，并把账号映射指到了空环境上 |
| `BROWSER_JOURNAL_INTEGRITY_FAILED:*` | 同上，平台级 | 同上 |
| `BROWSER_COMMAND_OUTCOME_UNKNOWN_DO_NOT_RETRY` / `BROWSER_SESSION_START_OUTCOME_UNKNOWN` | 命令**已下发**但结果读不回 | **不要重放同一条命令**；按错误消息里的指引对账，或落 `blocked.reason='outcome_unknown'` |
| `BROWSER_SPACE_LEASE_HELD_BY_SAME_PRINCIPAL` (409) | 本 Agent 的另一轮正占着这个 space | 不重试、不新建 space、不碰对方会话；`blocked.reason='space_busy'` 落盘，写明占用方 |
| `BROWSER_SPACE_SINGLE_WRITER_VIOLATION` (409) | space 上还有在席 writer 会话 | 平台会在 `start_session` 时回收租约已失效的僵尸会话；仍报错说明有**真在席**的会话——`blocked.reason='space_busy'` 并停止，**不要**新建 space |
| `BROWSER_TOOL_SPACE_FORBIDDEN` | Grant 主体够不着该 space | `blocked.reason='space_forbidden'` 报告，**不要**重复建新 space |
| `BROWSER_AGENT_PAUSED_BY_USER` (409) | 真人正在工作台里操作这个会话 | **不要重试、不要绕过**。对话里请用户点「恢复 Agent」后继续；定时运行里落 `blocked.reason='agent_paused'` |
| `BROWSER_TOOL_SESSION_TERMINATED` (409) | 会话已终止，正文带 `recoveryReason` | 读原因。对话里重开一次会话；定时运行里落 `blocked.reason='session_terminated'`，下一轮重新开会话 |
| 页面跳到 `account.xiaomi.com/.../verifyPhone`「小米账号安全验证」 | 密码已通过，小米要求短信验证 | 人机验证：`blocked.reason='human_verification'`。对话里保留会话，请用户在浏览器面板点「发送验证码」并输码、再点「恢复 Agent」并回复；定时运行里落盘收尾。**不要**自己点发送 |
| 点「登录」后弹「友情提示」对话框、页面不跳转 | 小米的协议确认弹框，正常流程 | 再取一次语义快照，点「同意并继续」（同样带 `params.wait`），之后才会跳转 |
