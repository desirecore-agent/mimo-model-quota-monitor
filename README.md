# MiMo 模型余量检测

定时巡检小米 MiMo 开放平台 Token Plan 的套餐额度：总量、已用量、剩余量与重置时间，余量偏低或登录态失效时主动提醒。

需要 DesireCore 10.0.156 或更高版本。安装前的前置条件、定时巡检的配法与能力边界，见 [USAGE.zh-CN.md](./USAGE.zh-CN.md)（[English](./USAGE.en-US.md)）。

## 仓库结构

| 路径 | 内容 |
|---|---|
| `agent.json` | Agent 配置（AgentFS `agent-config` schema） |
| `persona.md` | 人格描述 |
| `principles.md` | 执行纪律：只读边界、不执行命令、接口白名单、登录态判定、提醒原则、落盘规则 |
| `skills/mimo-quota-check/` | 私有技能：巡检 SOP，`references/` 下是结果与账号映射两份 JSON Schema |
| `memory/site-patterns/platform.xiaomimimo.com.md` | 站点经验：域名分工、接口白/黑名单、登录表单选择器、错误码应对 |
| `heartbeat/HEARTBEAT.md` | 只有一行标题：心跳刻意不用，巡检走定时调度 |
| `USAGE.md` | 使用说明兜底版（内容同 `USAGE.zh-CN.md`），市场详情页渲染的就是这个文件 |
| `USAGE.zh-CN.md` / `USAGE.en-US.md` | 使用说明双语版 |
| `CHANGELOG.md` | 变更历史 |
| `AGENTS.md` / `CLAUDE.md` | 维护指引（两份内容相同，运行时不读取） |

技能随 Agent 一起安装到它的私有技能目录，不进全局技能目录。

## 工作方式

每轮只巡检一个账号：用该账号独占的持久化浏览器环境打开 MiMo 控制台，以「导航 + 读取页面文本」的方式读取登录状态、套餐详情、用量与余额四个接口，校验后与上一轮结果对比，把本轮结果写进工作目录下的 `mimo-quota/<credentialId>/history/`，出现新的告警项时主动发消息，关闭浏览器会话，最后覆盖写 `latest.json`。同一情况持续期间只提醒一次。整个巡检不执行任何命令。

登录由 DesireCore 账号库以占位符方式代填，密码不进入模型上下文；遇到滑块、图形码或短信验证一律停下请用户处理。

## 市场条目

本仓库是内容事实源，市场条目 [`desirecore/market`](https://github.com/desirecore/market)
的 `agents/mimo-model-quota-monitor/` 只保留 pointer，按不可变 commit ref 指向这里。

改动流程：本仓库合并 → 市场条目重新 pin 到新 ref。详见 `AGENTS.md`。

## 许可

[MIT](./LICENSE)
