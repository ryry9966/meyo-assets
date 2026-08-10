---
title: 当 AI 学会“主动出击”：Proactive Agent 的落地与避坑指南
feedId: 32425
source: 综合讨论
publishedAt: 2026-08-10
---

# 当 AI 学会“主动出击”：Proactive Agent 的落地与避坑指南

## 背景：助手再聪明，不开口也只是摆设

在 OpenClaw 生态里，我们已经能通过 Agent + MCP 工具链完成不少自动化：查天气、发邮件、操作数据库，甚至把多个工具编排成工作流。但这些能力依然遵循“请求-响应”模式——必须由你说出第一句话，AI 才会动。

现实中有大量场景其实不需要你开口：服务磁盘快满了、关注的 GitHub 仓库有新的安全 Issue、日报没填被机器人催了……被动助手对这类需求无能为力。于是，**Proactive（主动式）能力**便成了 Agent 工程下一道必须跨过的坎：让 AI 在不被打扰的前提下，替你先把事情办了。

## 问题：如何让 Agent 安全、克制地“自己动手”？

与被动回答不同，主动 Agent 要同时解决三个难题：

1. **何时动**：怎么定义触发条件？定时轮询、事件回调还是条件规则？
2. **动什么**：怎么让 LLM 决定该不该执行工具，而不是凡事都来一遍？
3. **怎么控**：如何避免消息轰炸、错误操作或上下文失控？

这些难题不解决，Proactive 只会变成骚扰，甚至闯祸。

## 做法：基于 OpenClaw + MCP 的 Proactive 管道

我们在 OpenClaw 环境中实现了一条轻量 Proactive 管道，大致分为三层：**触发器 → 决策引擎 → 执行管道**。下面以“服务器磁盘空间主动告警”为例，展示具体步骤。

### 1. 触发器：用 Cron 插件作为最简入口

OpenClaw 允许挂载定时任务插件，这里我们可以注册一个每 30 分钟执行一次的任务：

```yaml
# cron-job.yaml
triggers:
  - name: disk-check
    schedule: "*/30 * * * *"
    action: agent_disk_check
```

触发器只负责到点唤起决策链，不包含任何逻辑判断。

### 2. 决策引擎：让 LLM 做一次“是否值得行动”的轻量 triage

触发器启动一个专用 Agent，该 Agent 会调用 MCP server 提供的 `check_disk` 工具，拿到当前磁盘使用率，然后依据系统 prompt 判断是否需要发出告警。示例 prompt（关键片段）：

```
You are a system health triage assistant.
Call the `check_disk` tool to get the current disk usage percentage.
If usage > 85%, respond with only the string "ALERT".
If usage <= 85%, respond with "OK".
Do not add extra explanation.
```

这样做的好处是：LLM 只负责高频的、极简的分类任务，成本低且速度快。同时你也可以在 prompt 里加入“过去一小时内已推送过告警则跳过”的规则，避免重复告警。

### 3. 执行管道：通过 MCP 工具链完成具体动作

当 LLM 返回 `"ALERT"` 后，外层流程抓取该信号，接着调用另一个 Agent 或直接调度 `send_message`（钉钉/微信/邮件等 MCP 工具）推送格式化后的告警卡片。推送内容可以由 LLM 再生成一句简短摘要，例如“CVM-01 磁盘使用率 92%，建议清理”。至此一次主动告警闭环完成。

### 代码片段（精简版）

```python
# Pseudo: proactive agent pipeline in OpenClaw
from openclaw import trigger, get_agent

@trigger("cron:disk-check")
def disk_check_job():
    agent = get_agent("triage")
    result = agent.run(prompt="Check disk and decide.")
    if "ALERT" in result.stdout:
        msg_agent = get_agent("notifier")
        msg_agent.run(prompt=f"Send alert: disk usage high. Details: {result.data}")
```

## 踩坑实录

### 1. LLM 判断阈值不稳定
最初我们让 LLM 自行根据数值判断是否“过高”，结果偶尔 80% 的磁盘也被判定为紧急，或者文字输出格式不统一。**解决办法**：将决策改为规则+LLM 双保险——先由 `check_disk` 工具直接返回数字，规则层判断是否 >85%，若是再让 LLM 生成自然语言摘要。LLM 仅负责文案，不做阈值决策。

### 2. 轮询成本与频率矛盾
每 5 分钟一次 check 对磁盘这类指标而言收益极低，但紧急场景又希望尽快通知。**建议**：能走事件驱动的绝不轮询。如果上游提供 webhook（如 GitHub、监控系统），优先在 OpenClaw 里注册 HTTP 触发器，把频率问题交给事件源。

### 3. 通知疲劳与冷却缺失
早期版本某次误判导致连续发送 7 条“磁盘满了”的消息。后续加入了 **cooldown 机制**：记录每一个 (trigger_id, target) 的上次通知时间，同一告警 2 小时内只发一次，并允许用户回复“snooze”临时屏蔽。

### 4. 上下文无限增长
Agent 在循环触发中容易把历史决策结果全量保留，导致 token 爆炸。**强制策略**：每次触发后清空对话上下文，仅通过工具返回的当前快照做判断，避免“翻旧账”。

## 可复用建议

- **定义清晰的触发维度**：将场景分为时间驱动（日报/周报）、事件驱动（GitHub push）、条件驱动（磁盘>阈值），优先实现有明确 ROI 的。
- **降级链路必备**：MCP 工具调用失败时，告警本身不能淹死在异常里。设计一条兜底通知（如通过简单 HTTP 请求发到健康检查通道）。
- **让用户可控制**：提供 `/proactive off` 这样的指令，或在推送消息中附带“关闭此提醒”的链接。克制比炫耀能力更重要。
- **日志与审计**：每次决策（是否告警、为何跳过）都写日志，方便回溯误判，也便于调优 triage prompt。

## 总结

Proactive 能力的落地，与其说是技术炫技，不如说是一场对工程纪律和用户共情的双重考验。借助 OpenClaw 的定时/事件触发器、MCP 工具链以及 LLM 的轻量判断能力，我们完全可以构建出一个“不请自来却恰到好处”的主动助手。但它要真正可用，离不开规则兜底、冷却策略和用户可控性。

当你的 Agent 学会在隐患刚冒头时就伸手拉你一把，而不是等你摔倒了才问“需要帮助吗”，这个助手才算真正融入了工作流。

---

