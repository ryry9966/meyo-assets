---
title: 让 AI 助手主动干活：在 OpenClaw 里跑通“只读优先”的 proactive 闭环
feedId: 34412
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw、Agent、MCP 这类工具链里，大多数助手仍然是被动的：你给 prompt，它执行。很多日常事务其实可以被提前处理，比如仓库异常巡检、依赖更新提醒、数据质量波动、定时摘要。但“主动”一旦放开，风险也随之放大。

真正的 proactive 不是让 Agent 自动写生产环境，而是把“察觉、建议、执行”分层。先让 AI 稳定地发现问题、给出建议，再逐步放开动作权限。

## 问题

直接让 Agent 主动执行，常见的失控点有：

- 误判：AI 把正常波动当成异常，触发不必要的操作。
- 权限过大：proactive 任务复用了默认 session 的写权限。
- 重复触发：定时器和 webhook 同时命中同一个任务。
- 确认疲劳：每条低风险事件都问一次，最终没人看。
- 循环触发：动作产生的新事件又唤起另一个 proactive 任务。

要让 AI 主动干活，必须先解决边界、幂等、审计和可回滚。

## 做法/步骤

我在 OpenClaw 上跑的最小闭环是：

**定时触发 → 只读上下文 → 结构化决策 → 分级动作 → 通知回流**

### 1. 定义只读触发源

先用 cron 或固定 webhook 作为触发，避免事件风暴。比如工作日早上 9 点巡检 GitHub 仓库，或者每 30 分钟检查一次数据表延迟。

### 2. 给 proactive 任务独立 session 和系统提示

不要复用日常对话 session。给 proactive 任务单独配置系统提示，明确只读边界。例如：

> 你是仓库巡检助手。仅允许调用 read_only MCP 工具。不要修改任何资源。输出 JSON 格式决策结果。

### 3. 输出结构化决策

要求 AI 输出 JSON，而不是自然语言。字段至少包含：

```json
{
  "need_action": true,
  "severity": "medium",
  "summary": "近 24h 新增 3 个带 bug 标签的 issue",
  "proposed_action": "notify maintainer",
  "risk": "low"
}
```

这样后续处理不需要再解析自然语言，也方便插件消费。

### 4. 动作分级

- 低风险：只通知，例如日报摘要。
- 中风险：dry-run 或创建待办任务，不直接执行。
- 高风险：进入确认队列，等待人工批准。

### 5. 通知回流

默认静默，只有 `need_action == true` 才推送。通知走 IM、OpenClaw 通知通道或 webhook。

一个配置示意（实际字段按你的调度器实现调整）：

```yaml
proactive_jobs:
  - name: repo-health-check
    schedule: "0 9 * * 1-5"
    allowed_tools:
      - github.list_issues
      - github.list_pull_requests
    action_policy: read_only
    decision_format: json
    notify:
      only_if: "decision.need_action == true"
```

运行后先让它只输出 JSON，不执行动作。人工观察几天，确认判断靠谱，再逐步放开 dry-run 或低风险动作。

## 踩坑点

### 重复触发

同一个任务可能被定时器和 webhook 各触发一次。方案：加幂等键，按“任务名 + 日期”写入 store，处理前先查是否已执行。

### 上下文过载

巡检时把所有 issue 或日志拉进来，会导致 token 爆炸、判断漂移。方案：先过滤，只取近 24h 更新、带 critical/bug 标签或状态变化的数据。

### 工具权限过宽

如果 proactive session 复用了默认 token，很容易获得写权限。方案：单独创建 token，或仅授权 read_only MCP 工具。

### 确认疲劳

每条低风险事件都问一次，用户最终会忽略所有通知。方案：低风险合并成一条日报，中高风险再单独提醒。

### 循环触发

通知动作可能产生新事件，又唤起另一个 proactive 任务。方案：设置冷却时间、来源标记，并在触发条件里排除由 proactive 动作产生的事件。

## 可复用建议

- **先只读一周，再考虑写操作。** 稳定后，写操作仍建议 human-in-the-loop。
- **所有主动行为记录审计日志。** 包括触发原因、上下文摘要、决策结果、最终动作。
- **用 feature flag 控制 proactive 任务。** 可以一键关闭，不必改配置或重启。
- **把决策结果结构化。** 后续 MCP 工具、插件、前端面板都能直接消费。
- **把 proactive 逻辑封装成插件。** 配置模板、工具白名单、通知策略都放在插件里，复用成本更低。

## 总结

Proactive 的价值不是“它替你做了多少事”，而是把关注成本降下来。先跑“只读巡检 + 低风险通知”，把幂等、权限、审计、冷却做好，再逐步放开动作范围。

一个可控的主动助手，比一个四处乱动的自动化脚本更有工程价值。工程化地限制边界，AI 的主动性才能真正落地。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/a31f12379bfbdfbe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/88cfcc268f4db149.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/f312ebe8bfe8366c.png)

