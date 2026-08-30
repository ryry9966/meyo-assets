---
title: 不主动，是因为缺一条事件循环：给 OpenClaw 加上有限 proactive 能力
feedId: 35427
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景：Agent 不主动，不一定是模型不行

很多 OpenClaw 用户的体验卡在同一个位置：你问一句，Agent 动一下；你不问，它就一直等着。

这种“被动应答”模式下，Agent 的 MCP 工具、插件、自动化脚本都只能被用户消息触发。但工程里大量事务本来就不需要等人开口：

- 证书快过期，提前 14 天创建续期任务；
- 依赖仓库有新的安全公告，自动打开一条检查 Issue；
- 磁盘使用率连续 30 分钟超过 85%，先执行清理预演；
- 某个分支 30 天没有活动，自动创建归档提醒。

这些场景的特点是：信号明确、动作边界清楚、风险可控。实现不了 proactive，通常不是模型推理能力不够，而是缺少一条“监听信号 → 过滤策略 → 执行动作 → 反馈闭环”的事件循环。

## 问题：无限 proactive 比被动更危险

如果让 Agent 随时可以自主行动，会出现几类典型问题：

- 通知风暴：每个噪声信号都触发一次告警，很快没人看；
- 重复动作：同一个信号触发两次，创建一堆重复 Issue/PR；
- 破坏性动作：模型直接决定执行 `rm`、`drop`、`restart`；
- 成本不可控：后台频繁调用 LLM 或外部 API；
- 状态污染：Agent 记住了错误判断，后续持续带偏。

所以 proactive 能力的关键不是“让它多动”，而是“有限自治”：动作范围提前划定，执行路径可审计，风险动作必须干跑或审批。

## 做法：一条四层 proactive loop

我建议把 proactive 能力拆成四层，而不是写一个巨大的自主 Agent。

### 第一层：信号源

信号来源通常有四种：

- 时间触发：cron / schedule；
- 外部事件：webhook、GitLab/GitHub event、MQ 消息；
- 内部状态：任务队列长度、缓存命中率、Agent 记忆变化；
- 环境指标：CPU、磁盘、证书剩余天数、依赖漏洞数。

在 OpenClaw 里，比较稳妥的做法是用 scheduler 插件定时拉取，或者用 webhook 插件接收外部事件。不要一上来就做复杂的流式事件处理。

### 第二层：过滤与策略

拿到信号后，先过滤噪声，再决定是否行动。

策略至少包含这几项：

```yaml
policy:
  threshold:
    disk_usage: 85%
    cert_days_left: 14
  cooldown:
    same_fingerprint: 6h
  max_actions_per_day: 5
  require_approval: ["restart", "delete", "force_push"]
  allowlist_tools:
    - github.create_issue
    - github.create_draft_pr
    - notify.im
```

这里的核心是“fingerprint + cooldown”：同一个信号在冷却时间内只处理一次。比如一个分支已经提醒过，就不要每周重复创建同一个 Issue。

### 第三层：动作层

动作层只允许调用预定义工具，不让模型现场发明命令。

更严格一点，可以拆成 planner 和 executor：

- planner 可以自由读数据、做判断；
- executor 只能调用固定工具，参数必须通过 schema 校验；
- 破坏性动作一律先干跑或等待人工确认。

### 第四层：反馈闭环

每次 proactive 动作都要留下记录，并给用户一个反悔入口。

推荐记录 JSON 审计日志：

```json
{
  "trigger": "schedule:weekly_stale_branch_check",
  "signal": "branch:feature/old-flow",
  "decision": "create_archive_issue",
  "action_id": "issue-1823",
  "status": "done",
  "fingerprint": "branch-archive:feature/old-flow:20250614"
}
```

同时通知里带上“暂停此规则”“撤销此动作”的链接。没有反馈闭环的 proactive，很快会变成不可维护的自动化垃圾。

## 一个可复现的最小实现

以“每周一检查 stale branch，超过 30 天未更新则创建归档提醒”为例。

**步骤一：接入只读数据源**

先给 OpenClaw 配一个 Git MCP server，只用列分支、看最近提交时间、查是否存在开放 PR 的能力。不要给写权限。

**步骤二：注册定时触发**

在插件或配置里加一个 cron：

```yaml
trigger:
  type: schedule
  cron: "0 9 * * 1"
  timezone: "Asia/Shanghai"
```

注意用绝对时区，不要依赖容器默认时区。

**步骤三：写检查工具**

工具内部先做只读检查，返回一个候选列表。比如：

```json
[
  {
    "branch": "feature/old-flow",
    "last_commit_days": 37,
    "open_pr": false
  }
]
```

这个工具不要直接创建 Issue。

**步骤四：策略引擎决定动作**

匹配规则：

- `last_commit_days > 30`
- `open_pr == false`
- 同一 fingerprint 在最近 7 天内未执行过

满足条件后，才调用 `github.create_issue`，标题使用固定模板，正文附上分支名、最后提交时间、建议。

**步骤五：通知与撤销**

创建成功后，用 IM 插件发送一次性通知。通知里带规则名称和关闭链接。记录 fingerprint 到本地 SQLite，避免下次重复创建。

## 踩坑点

1. **时区问题**：容器里 cron 默认可能是 UTC。把时区显式写进配置，避免周五下午变成周六凌晨执行。
2. **重复动作**：不要用“分支名”做唯一键，分支可能被删后重建。用 `规则名 + 分支名 + 日期粒度` 做 fingerprint，或者用内容哈希。
3. **通知风暴**：只通知最终动作，不通知每次轮询。冷却时间必须覆盖一个轮询周期以上。
4. **模型自由发挥**：agent 很可能会在判断时应激调用额外工具。把 executor 的工具列表锁死，LLM 只能调 approved tools。
5. **动作不可回滚**：不要直接删除或强推。优先创建 Issue、Draft PR、tag、归档标记。写操作要可撤销。
6. **脚本路径与权限**：定时任务通常以服务用户运行，shell 工具里的相对路径、环境变量、Git token 都可能和交互环境不一致。先跑一次 shadow mode 验证。

## 可复用建议

- **先做 shadow mode**：只记录“如果执行会做什么”，跑一周再看误报率；
- **一次只接一个信号一个动作**：不要一开始就做多信号联动；
- **每个 proactive 规则都要有 kill switch**：一个环境变量或配置开关能立即停用；
- **动作结果必须回流到状态**：创建过 Issue 的分支，下次轮询要跳过，不能只靠外部状态；
- **从读后写开始**：展示信息 > 创建记录 > 修改状态 > 执行命令。按这个顺序逐步放开权限；
- **把 proactive 当运维任务看**：不是“让 AI 替你决定”，而是“用规则让 AI 在边界内替你跑腿”。

## 总结

Proactive 能力并不等于模型更聪明。它更像给 OpenClaw 加了一个受控的后台任务系统：有信号源、有过滤策略、有动作白名单、有反馈与撤销。

最难的不是“让 Agent 动起来”，而是让它动得有边界、可停止、可追溯。一个真正实用的 proactive Agent，往往不是最激进的，而是最像靠谱值班员：不打扰你，但该做的事会提前做了，并且做了会告诉你。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/292691f6650c9299.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/6ae723793f33f36d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/be3391d818b24b48.png)

