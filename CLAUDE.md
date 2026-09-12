# MiMo 模型余量检测 · 开发与维护指引

> 本文件给**接手维护这个 Agent 的智能体或开发者**看。`AGENTS.md` 与 `CLAUDE.md` 内容相同，两份必须保持同步。
> DesireCore 运行时**不读取**本仓库里的这两个文件，所以这里写的是维护指引，不是 Agent 的行为规则——行为规则在 `persona.md`、`principles.md` 和 `skills/mimo-quota-check/SKILL.md`。

---

## 1. 这个仓库是什么

DesireCore 官方市场里「MiMo 模型余量检测」Agent 的**内容仓库**。它用内置浏览器巡检小米 MiMo 开放平台 Token Plan 的套餐额度，把结果结构化落盘，并在余量偏低或登录失效时提醒用户。

能力全部来自 DesireCore 平台自带的工具（内置浏览器、账号库占位符填表、文件工具、`SendUserMessage`）和本仓库自带的 1 个私有技能，**不依赖任何第三方 CLI**。前置条件是授权类的：账号库里的 MiMo 账号、一次真人短信验证、一个持久化浏览器环境。

## 2. 两个仓库的关系（改动前必须懂）

```
desirecore-agent/mimo-model-quota-monitor   ← 本仓库：Agent 的全部内容
        ↑ 被指向（pin 到具体 commit）
desirecore/market
  └── agents/mimo-model-quota-monitor/
        ├── entry.json                 ← 卡片：写着「内容在本仓库，版本 <SHA>」
        └── catalog-metadata.v1.json   ← 审核信息（证据六项）
```

**市场侧只有 pointer，不能放 `agent.json`。** 校验器要求 `agent.json`（inline）与 `entry.json`（pointer）**恰好存在一个**。

**改内容 → 改本仓库；发新版 → 改市场卡片的 pin。** 用户装到的永远是卡片上 pin 的那个 commit，不是本仓库最新 `main`。

发新版时 pin 要**三处同时改**，漏一处 CI 必红：

| 文件 | 字段 |
|---|---|
| `entry.json` | `source.ref`（同时改 `latestVersion`） |
| `catalog-metadata.v1.json` | `provenance.content.ref` |
| `catalog-metadata.v1.json` | `governance.compliance.reviewedRef`（同时改 `release.version`、各 `timestamps` 与 `compliance.reviewedAt`） |

自检不要按路径逐个查，全文扫 40 位 hex，要求整个条目只剩一个 SHA：

```bash
grep -oE "[0-9a-f]{40}" agents/mimo-model-quota-monitor/*.json | sort | uniq -c
```

最低客户端版本 `requiredClientVersion` 同时写在 `entry.json` 顶层与 `catalog-metadata.v1.json` 的 `compatibility` 里，两处必须一致。
Agent 用到新发版才有的平台能力或修复时要跟着提高（1.0.0 依赖 10.0.156 里的调度完成判据、内置浏览器操作记录上限修复、调度暂停恢复不丢工具）。

推之前在 market 仓库本地跑目录校验器，只看 `N error(s)`（warning 多为存量条目）：

```bash
uv run --quiet scripts/catalog/validate_catalog_metadata.py --require-complete
```

## 3. 目录结构与各文件职责

```
agent.json          AgentFS 运行时配置（纯运行时字段，pointer 形态下由 agentConfigSchema 直接校验，
                    根级 additionalProperties: false，不要加市场展示字段）
persona.md          人格（L0/L1/L2 三层）
principles.md       执行纪律：只读边界、不执行命令、接口白名单、登录态判定、提醒原则、落盘
skills/mimo-quota-check/
├── SKILL.md                      巡检 SOP（10 步 + 失败分支 + 告警规则 + 建定时巡检 + 汇报格式）
└── references/
    ├── result.schema.json        每轮结果的字段定义
    └── accounts.schema.json      账号 ↔ 浏览器环境映射表的字段定义
memory/site-patterns/platform.xiaomimimo.com.md   站点事实（随 Agent 安装，scope=agent）
heartbeat/HEARTBEAT.md            只有一行标题（见 §4「心跳」一行）
USAGE.md / USAGE.zh-CN.md / USAGE.en-US.md         使用说明（市场详情页取无后缀的 USAGE.md）
README.md / CHANGELOG.md / LICENSE
```

**`references/` 里的文档必须在 `SKILL.md` 正文里显式索引。** `Skill` 工具只返回 SKILL.md 正文，不列出目录——
没被索引的文件 Agent 看不见。新增 reference 时照现有表格补一行。

## 4. 行为设计里不能随手改的约束（每条都来自真机事故）

| 约束 | 为什么 |
|---|---|
| 读接口只用 `tab.navigate` + `BrowserSnapshot(mode='text')`，禁止 `page.evaluate` | `browser.page.evaluate` 在 `alwaysHumanGate` 名单里，每次都要一条审批结论，定时运行容易卡在审批上；导航与读文本不需要审批 |
| `BrowserAct` 的动作参数一律放 `params` 里（如 `params.url`），语义快照用 `BrowserSnapshot` | `BrowserAct` 入参只收 `action` / `sessionId` / `tabId` / `params`，多余字段直接拒绝；它没有 `snapshot.semantic` 这个动作 |
| 定时运行里不提问、不问「要不要继续」 | 没有人回答；模型曾在做到一半时停下来问，这一轮就静默漏掉了 |
| 浏览器平台报错（`BROWSER_PROVIDER_UNAVAILABLE` 等）只记 `blocked`，不新建 space、不改映射表 | 平台故障期间模型曾每轮新建 space，攒出 30 多个没登录态的空环境，映射被指向空环境 |
| 账号映射放工作目录 `mimo-quota/accounts.json`，不放站点经验 | `SitePatternWrite` 遇手机号等敏感片段会自动降级写用户层，而读取时用户层优先只取第一个命中，一份流水副本就能遮住整份站点经验 |
| 定时运行里不调用 `SitePatternWrite`，对话里只在用户明确要求时写 | 平台按英文单词识别敏感片段，`token` 会命中「Token Plan」、`cookie` 会命中「Cookie」（主仓 `site-pattern-write.ts`），写这个站的经验几乎必然降级到用户层；而 web-access 技能会主动引导模型写站点经验 |
| 一个账号独占一个持久化 space，按 id 选、不按名字选 | 小米账号 Cookie 在同一分区里会互相覆盖；多个 space 同名是常态 |
| 401 先导航 `loginUrl` 免填表恢复；定时运行里上一轮已需要人工登录时不再填表 | 分区里残留的小米 Cookie 链常能经 sts 直接恢复，免去一次登录；每小时重复填表会触发风控 |
| 自动登录最多尝试 1 次，人机验证立即停 | 不得绕过验证；反复重试会触发风控 |
| 提交登录表单后禁止 `Sleep`；点击带 `params.wait`，然后留在当前页面读文本判定 | `Sleep` 空轮次让模型丢掉方向直接收场；一提交就导航去读 `userProfile` 会把验证页和报错原话冲掉，既判不了分支，用户也看不到验证页 |
| 收尾顺序：写 `history/<采集时刻>.json` → 需要时 `SendUserMessage` → 关会话 → 最后写 `latest.json`；调度完成判据只列 `latest.json` | 模型曾写完结果就「失忆」收场，漏关会话、漏记历史；把固定路径的 `latest.json` 放最后并列入完成判据，半途收场就会被平台发现并续跑。历史按采集时刻一轮一个文件、同名覆盖，重复执行收尾步骤也不会产生重复记录（旧设计追加 `history.jsonl` 时发生过） |
| 会话必须自己关 | 平台不替 run 收会话，租约 1 小时内同一 space 的下一轮会被挡住 |
| 首次登录在对话里完成：人机验证时对话里不关会话、不落盘，回复用户后停下等回复；定时运行不建 space | 定时运行收尾必关会话，用户去看时页面已经没了；`list_spaces` 只列有授权的 space，授权被撤销时看起来也像「不在列表里」，后台建 space 会留下没登录态的空环境 |
| 建 space 前先在 `ManageCredentials list` 里核对本轮账号 | 账号不存在时建出来的 space 永远登录不上，还会留下垃圾 space |
| 提醒只看新出现的告警项：对比基准 = 上一轮 `alerts` ∪ `lastOk.alerts`；可能自愈的阻塞连续两轮才算 | 余量低、临近重置是持续状态，每轮都提醒会每小时打扰一次；只和上一轮比时，中间阻塞一轮会让余量低再提醒一次；额度重置回满曾会被误判成异常消耗；平台每轮本来就会在对话里写执行回执 |
| 上一轮结果只读 `latest.json`，而且在本轮写任何文件之前读；最近一次成功巡检由结果里的 `lastOk` 逐轮带下去 | 不用翻历史就能算变化与对比基准；`latest.json` 是每轮最后一步才写的，读到的一定是上一轮的最终结果 |
| 全程不执行任何命令（Bash / PowerShell / 终端 / BrowserScript） | 真机上模型巡检做完后凭空声称「换了新模型」，经 AI 审批执行命令启动带调试端口的进程、探测本机 9222 端口、列进程。审批员看不到任务目标，这类命令会被当成「无破坏性」放行。另外 Windows 没装 Git Bash 时 Bash 由 cmd.exe 执行，`tail`、单引号、heredoc 都不可用。DesireCore 的 `Write` / `Edit` 不能追加，所以历史改成一轮一个文件 |
| 定时运行的最后回复只写一行摘要 | 调度每轮都把最后回复作为执行回执写进 Agent 对话，长回复会刷屏 |
| 数值一律用 `MathCalc` 算；时刻差与时区换算照抄 SKILL「时间计算」的表达式 | 模型曾在汇报里心算错距重置天数；`MathCalc`（mathjs）不认日期字符串，平台也没有取时间的工具，不给表达式就只能心算或执行命令。两条表达式已用真实 `MathCalc` 对照 JS Date 随机验证 |
| `collectedAt` 取提示里的 `Today's date` + `Local time`，不换算 UTC | 模型拿不到「读到数据的时刻」，唯一的时钟是提示里的当前本地时间；自己换算 UTC 在 0–8 点会跨日出错 |
| 定时运行不再填表时 `blocked.reason` 沿用上一轮 | 从 `human_verification` 改记 `not_logged_in` 会被当成新告警项，同一次登录失效提醒两次 |
| 数据存疑时不更新 `lastOk` | 存疑的数字当下一轮基准会误报消耗过快 |
| 只做巡检这一件事 | 轻量模型曾凭空编出「给某项目写测试」「给 README 加文档」的任务并尝试执行脚本 |
| 心跳关闭，`HEARTBEAT.md` 只留标题 | 心跳上下文只有 `HeartbeatRespond` 一个工具，不加载技能、记忆和工作目录，做不了巡检；而内容非空时每轮心跳都会白调一次模型。平台把「只有标题、空列表项、空行」判为空（主仓 `heartbeat/content-gate.ts`），HTML 注释不算空 |

## 5. 修改与发布流程

1. 改本仓库内容；行为变化同步 `SKILL.md` / `principles.md` / 站点经验 / `references/result.schema.json` 的一致性（尤其是告警项、`blocked.reason` 取值与落盘路径），以及 USAGE 中英文与市场条目描述
2. 递增 `agent.json#version` 与 `SKILL.md` frontmatter 的 `version`，写 `CHANGELOG.md`；用到新平台能力时同步提高市场条目的 `requiredClientVersion`
3. 用 DesireCore 主仓的 `validateAgentConfig`（`packages/schemas/src/agent/index.ts`）校验 `agent.json`，用技能 frontmatter 解析器校验 `SKILL.md`（`type` 只能是 `procedural` / `conversational` / `meta`）
4. 提交并推送本仓库 `main`，记下新 commit SHA
5. 在 `desirecore/market` 开分支，按 §2 改 pin 三处与版本、时间字段，本地跑校验器，提 PR
6. market PR 的 Copilot review 线程也要处理完，checks 全绿仍 `BLOCKED` 多半是线程没 resolve

## 6. 真机验证建议

- 用独立的 dev 实例（`npm run dev -- --home <path>`），账号库放一个测试账号，首次登录由真人过短信验证
- 让 Agent 在对话里建一个每小时的调度（完成判据 `latest.json`），连续观察多轮
- 每轮检查：`latest.json` 已更新、`history/` 增加且只增加一个文件、浏览器会话已关闭、没有调用 Bash / PowerShell 或巡检用不到的工具
- 余量不变时连续两轮只提醒一次；手动制造一次网络失败，确认第一轮不提醒、连续第二轮才提醒；余量低期间插入一轮阻塞，恢复后不重复提醒
- 对话里走一次首次登录：停在验证页不关会话，用户验证并回复后接着取数
- 故意制造一次 401（清掉 Cookie 或等过期），确认先走免填表恢复
- 核对 `/tokenPlan/usage` 实际返回里的额度字段名，补进站点经验「接口」一节（2026-09-12 实测 `/tokenPlan/detail` 不含额度数字，`usage` 的字段结构尚未核实）
- 核对 `/api/v1/balance` 当前行为：2026-09-12 两轮定时运行里模型分别报告它返回 404、导航时出现 terminated，原文未留存，需真机确认后更新站点经验
