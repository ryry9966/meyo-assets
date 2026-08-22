---
title: OpenClaw proactive 工程化：从“等指令”到“条件触发”的受控实践
feedId: 34198
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

OpenClaw 里大多数 Agent 还是被动响应：用户发消息 → 模型规划 → 调 MCP 工具 → 返回结果。这套在对话场景够用，但在运维和研发自动化里明显不够。很多事是“条件满足就该做”：CI 挂掉、依赖发布新版本、日志出现异常模式、issue 超时未响应。如果每个都要人手动触发，助手就没省下多少事。

## 问题

直接把 proactive 能力塞进主对话循环会有几个问题：

- 巡检结果频繁灌入上下文，token 消耗快，还干扰正常对话；
- 主动执行误操作风险高，越权调用工具很难控制；
- 没有闭环，做了没人知道，或者重复执行；
- 事件重复触发导致死循环。

所以不能只给模型一个“你可以主动做事的 prompt”，需要工程化的边界。

## 做法/步骤

我的做法是把 proactive 能力独立成“事件源 → 策略匹配 → 动作执行 → 通知”四步流水线，不侵入主对话循环。

**1. 独立 worker，不侵入对话主循环**

在 OpenClaw 里起一个独立进程或定时任务，只做上面的四步。聊天会话照常响应，proactive worker 只负责条件触发。

**2. 事件源通过 MCP 接入**

比如 GitHub MCP server 暴露 `list_workflow_runs`；HTTP MCP 轮询 Prometheus/日志接口；或自己写一个 poller 插件。事件源只负责产出结构化事件，不做判断。

**3. 策略层用声明式配置，不用每次都过 LLM**

示意配置：

```yaml
sources:
  - name: ci_status
    type: mcp
    server: github
    method: list_workflow_runs
    interval: 5m

policies:
  - name: ci_failed_on_main
    when:
      source: ci_status
      match:
        conclusion: failure
        branch: main
    actions:
      - tool: slack.post_message
        args:
          channel: "#ci"
          text: "CI failed on main: {{ run.name }}"
    cooldown: 15m
    dedup_key: "ci_failed:{{ run.repo }}:{{ run.id }}"
```

大部分过滤在策略层完成，只有需要语义判断的事件才交给 LLM，成本可控，也能减少误判。

**4. 动作走白名单工具**

执行层只允许调用提前注册的低风险工具。写操作默认 dry-run 或需要人工确认。OpenClaw 的 MCP client 可以做一层 tool ACL，proactive worker 不直接拿到完整工具集合。

**5. 闭环反馈**

执行后写回状态，生成简短摘要通过 IM/邮件通知用户，用户可以确认、回滚或忽略。同时把事件指纹写入状态库，避免重复处理。

## 踩坑点

- **循环触发**：处理动作本身产生新事件，比如自动创建 issue 又触发 issue 事件。解决：给每个事件算指纹，处理前查状态库，已处理直接丢弃；动作工具带幂等键。
- **上下文膨胀**：不要把所有巡检结果都塞给 LLM。先让策略层过滤，只把少数需要语义判断的事件交给模型；确定性动作直接执行，不走 LLM。
- **权限过大**：proactive worker 不要复用日常对话的完整 tool 集合。给最小权限，写操作强制 dry-run 或 approval。
- **静默失败**：异常被吞后任务卡住，下次也不会重试。需要超时告警、死信队列和重试退避。
- **时区问题**：定时任务用 UTC，界面展示再做本地化；cron 表达式写清时区。

## 可复用建议

- 先做通知型 proactive，跑稳定后再放开执行型动作。
- 每个事件必须有 `dedup_key`，动作必须幂等。
- 冷却时间和速率限制要配，避免同类事件风暴。
- 状态外部化，用 SQLite/Redis 存已处理事件和任务状态，不要依赖进程内存。
- 结构化日志记录 `trigger → decision → action → result` 全链路，出问题能追。
- 在非生产环境至少跑 1–2 周再上生产。

## 总结

Proactive 不是让模型“更主动”，而是把判断和执行拆开，用事件源、策略、白名单工具和状态管理构成一条受控流水线。OpenClaw 的 MCP/插件生态刚好能承载这种结构，但护栏要自己做。先把它当成一个条件触发执行器，而不是一个随时会自己干活的“数字人”，落地会更稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/a7cb5e7339bb5715.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/a9b15c17227fb0a6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/a7152fa493ec6a00.png)

