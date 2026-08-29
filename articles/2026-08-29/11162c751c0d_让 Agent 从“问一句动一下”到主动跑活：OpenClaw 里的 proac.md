---
title: 让 Agent 从“问一句动一下”到主动跑活：OpenClaw 里的 proactive 实践
feedId: 35216
source: 综合讨论
publishedAt: 2026-08-29
---

# 让 Agent 从“问一句动一下”到主动跑活：OpenClaw 里的 proactive 实践

## 背景

目前多数 agent 还停留在 turn-based：用户开口，模型才动。很多运维和协作场景中，问题不是“不会做”，而是“没人记得去做”。比如证书快过期、某个 GitHub issue 三天没回复、每日站会前的变更汇总。这类任务规则明确、频率固定，很适合交给 proactive agent。

但 proactive 不是让 AI 自由发挥。真正可用的主动能力，应该像定时巡检脚本加了一层自然语言决策，而不是一个随时可能打扰人的“智能管家”。在 OpenClaw 这类 agent 框架里，可以通过 MCP 工具 + 定时触发 + 白名单策略，做成一个受控的主动执行链路。

## 问题

做之前先想清楚四件事：

1. 触发源来自哪里？定时器、webhook，还是轮询状态？
2. 如何避免误动作？默认只读，写操作需要确认。
3. 如何避免打扰？通知要聚合，并且有静默窗口。
4. 状态存哪里？不能只靠对话上下文，必须有外部存储或明确的输入快照。

## 做法/步骤

第一步，把能力收成 MCP server。比如写一个只读 MCP，暴露 `list_domains`、`get_cert_expiry`、`list_recent_issues`。工具描述写清楚返回结构和限制，让 agent 不要乱猜。只读工具是 proactive 的边界。

第二步，用定时触发跑巡检。OpenClaw 里如果有 scheduler，可以用内部定时任务；没有就用外部 cron 调用 CLI 或 skill。建议先做每日一次的低频任务。触发后不直接让模型“看看有什么要做”，而是给它一个明确的 input snapshot，例如：“以下是今天到期的证书列表，请判断是否需要提醒”。

第三步，把策略写成规则，不靠模型临时决定。比如：

```yaml
if cert.days_left < 7 and cert.env == 'production':
    notify_owner(severity='high')
```

模型只负责解释和生成通知文案，不负责决定“要不要发”。这样可控性高很多。写操作可以接入 OpenClaw 的审批机制，或者发通知后让人手动确认。

第四步，通知聚合与审计。一个任务只发一条汇总消息，不要每个异常都单独发。每次触发记录一条 audit log，包含 trigger、输入摘要、决策结果、执行动作、消息 ID。这样出问题能回查。

示意配置：

```yaml
triggers:
  - name: expiring_certs
    schedule: "0 8 * * *"
    task: check_expiring_certs
    policy:
      - if: cert.days_left < 7
        action: notify_owner
        require_approval: false
```

这只是一个结构示意，具体要按 OpenClaw 的插件/脚本接口调整。

## 踩坑点

- **循环触发**：agent 处理后产生的新事件可能再次触发它。比如它监控消息列表，自己发的通知又被当成输入。必须过滤来源，比如排除自己的 bot 账号或消息类型。
- **重复执行**：定时器和 webhook 可能同时触发同一任务。给每次任务生成幂等键，例如 `cert_check:domain:2025-01-01`。
- **上下文膨胀**：不要把所有历史通知塞回 prompt。每次只给当前输入和必要的规则，否则几周后 prompt 会失控。
- **权限过大**：不要让 proactive agent 默认拥有删除、发布、合并等权限。先做“只读 + 提醒”，等人确认后再由另一个受限流程执行。
- **时区问题**：定时任务用 UTC 存储，展示时按用户时区转换。别在 prompt 里用“明天”这种模糊词。
- **误报疲劳**：阈值太敏感会天天提醒，最后没人看。设置聚合窗口和严重等级，低优先级只进汇总，不单独打扰。

## 可复用建议

1. 从只读巡检+通知开始，跑两周，再考虑写操作。
2. 策略写成可测试函数或规则文件，不要藏在 prompt 里。
3. 通知里带触发原因和原始数据链接，方便用户快速判断。
4. proactive agent 的默认值应该是“不动作”而不是“动作”。不确定就不发，或转人工。
5. 给 proactive 系统本身加监控：如果太久没触发，或触发频率异常，要能告警。

## 总结

proactive 能力真正有用的场景，是那些规则稳定、价值明确、但人容易忘的低频任务。用 MCP 限制工具边界，用定时器或 webhook 提供触发，用白名单和审批控制写操作，用 audit log 保留可追溯性。这样做出来的 agent 不会显得“自作主张”，反而像一个可靠的巡检同事：它不替你拍板，但会提前把该看的东西推到你面前。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/81333e60447dc26b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/d3e43c246e4df859.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/3eb076d6042f7b79.png)

