---
title: 让 AI 助手主动干活：一个可落地的 proactive 最小闭环
feedId: 34720
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

现在多数 Agent 的交互模式仍然是“人先开口”：你问一句，它动一下。真正有工程价值的是 proactive 能力——不等用户输入，根据时间、事件或状态变化主动发起工作。但在 OpenClaw / MCP / 插件这套体系里，直接给 Agent 加一个 cron 和写权限，翻车率很高。

我们团队在 OpenClaw 上跑了几周 proactive 任务，结论是：能落地，但前提是把“何时醒、能看什么、能做什么、做错怎么退”拆开设计，而不是让模型自发行动。

## 问题

最初的做法很直接：在 system prompt 里写“每天检查仓库状态，有问题就处理”，然后给 Agent 一个定时触发和几个写工具。很快出现三类问题：

1. **参数臆造**：信息不足时，模型会自行补全参数。比如把未确认的 PR 当成已通过，或者在 Slack 里 @ 错人。
2. **重复执行**：时区、夏令时或 webhook 重试导致同一个任务跑两次，产生重复通知或重复 issue。
3. **不可回滚**：直接开放写操作，一次误判就需要人工修数据，而且没有任何审计记录。

## 做法/步骤

**第一步：只做低风险、可逆、幂等的动作。**  
通知、创建 issue、打标签、更新看板列属于低风险；合并、删除、发布等动作先排除。proactive 的初期目标不是“自动完成业务”，而是“把人工巡检和提醒省掉”。

**第二步：把 proactive 任务显式化，不要藏在 prompt 里。**  
每个任务用配置定义清楚触发条件、前置检查、动作、回滚策略和通知对象。示意 YAML：

```yaml
id: pr-review-reminder
trigger: cron(0 9 * * 1-5, tz=Asia/Shanghai)
precondition:
  read: mcp.github.list_open_prs
  assertion: open_review_requested_count > 3
actions:
  - tool: mcp.slack.post_message
    params: { channel: "#eng", template: "pr_reminder" }
rollback:
  tool: mcp.slack.delete_message
notify: [owner]
```

**第三步：用 MCP 把“读”和“写”分开。**  
读工具可以放开给模型自由组合；写工具只暴露白名单方法，并在 MCP server 层做严格参数校验。不要让模型直接拿到底层 API 的完整权限。

**第四步：每次运行注入固定上下文。**  
包含当前时间、任务 id、上次运行摘要、失败记录。否则长周期任务里模型很容易忘记自己在哪个任务中，把 proactive 运行当成普通对话。

**第五步：先跑 shadow mode。**  
执行写工具前只记录日志、不真正调用。把模型决策和人工判断做对比，误报率稳定后再切真实执行。

## 踩坑点

- **时区**：cron 表达式不带时区会踩夏令时。统一用 UTC，或显式带 IANA 时区。
- **重复触发**：webhook 源可能重试。任务入口要加幂等键，例如 `任务id + 日期` 去重。
- **工具超时 / token 过期**：MCP server 的 token 过期后，模型会把错误信息当上下文继续编。工具层应返回明确错误码，Agent 检测到 `auth_error` 直接终止运行。
- **上下文漂移**：模型有时会输出大段解释而不调用工具。可以强制要求第一轮必须产出结构化计划，或限制非工具输出的长度。
- **成本**：高频 proactive 非常烧 token。用只读小模型做预筛，复杂判断再上大模型；或者把周期拉长。

## 可复用建议

- 建立 proactive 注册表，每个任务有人工 owner、回滚策略、通知渠道。
- 给每个任务配置 `max_run_duration` 和 `max_tool_calls`，防止跑飞。
- 保留最近 N 次运行记录，排障时先看日志而不是重跑。
- 渐进式放开权限：先通知类 → 再状态变更类 → 最后才考虑带副作用的动作。
- 监控误报率、执行成功率、回滚次数，不要只看“跑了多少任务”。

## 总结

Proactive 能力不是给模型更多自主权，而是把调度、工具边界、审批链路和回滚机制设计清楚。在 OpenClaw / MCP 体系里，最小可用的 proactive 闭环是：显式触发器 + 受控 MCP 工具 + 幂等动作 + shadow/approval 模式。先跑通知和巡检，别一上来就让它自主修改外部系统。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/19e7905da72bb1df.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/2d222fcdba360328.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/bc1e22593a98941a.png)

