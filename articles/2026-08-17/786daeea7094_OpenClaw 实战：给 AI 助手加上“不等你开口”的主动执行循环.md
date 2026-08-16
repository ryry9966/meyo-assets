---
title: OpenClaw 实战：给 AI 助手加上“不等你开口”的主动执行循环
feedId: 33515
source: 综合讨论
publishedAt: 2026-08-17
---

# OpenClaw 实战：给 AI 助手加上“不等你开口”的主动执行循环

## 背景

我们熟悉的 AI 助手大多是被动响应式：你问一句，它答一句。OpenClaw 生态里，MCP、插件和 Agent 能力很强，但很多实践仍停留在“人等模型回复”的阶段。

实际工作中，有些事不应该靠人主动开口：

- 每天早上 9 点汇总未关闭的 issue；
- CI 失败 10 分钟后定位最近的提交者；
- 数据库磁盘使用率超过 80% 时提前告警；
- 定时检查证书是否快过期。

这些需求如果全靠人肉触发，既不稳定也不及时。更合理的做法是让 Agent 具备 proactive 能力：到点、到事件、到阈值时自己跑，跑完把结果带回来。

## 问题

很多人一听到 proactive，就以为是模型变得“更主动”。工程上真正缺的不是模型能力，而是四个环节：

1. 触发从哪里来；
2. 动作如何安全执行；
3. 什么时候直接做，什么时候必须人确认；
4. 执行结果如何可回溯、可打扰控制。

在 OpenClaw 体系里，可以用 scheduler、MCP tools 和 policy gate 组合出一个最小可行的主动执行循环，不需要引入复杂平台。

## 做法：五个环节搭一个最小闭环

### 1. 定义触发源

触发源不要一上来就接很多，先选两类：

- 定时触发：适合日报、周报、巡检；
- 事件触发：通过 webhook 打入 OpenClaw 的任务接口。

示例配置：

```yaml
triggers:
  - name: morning-digest
    schedule: "0 9 * * 1-5"
    intent: summarize_open_issues
    timezone: Asia/Shanghai
    ttl: 15m
  - name: disk-watch
    schedule: "*/30 * * * *"
    intent: check_disk_usage
    threshold: 80
```

每条 trigger 建议固定三个字段：`intent`、`payload`、`priority`。`intent` 决定后面调哪些工具，`payload` 是上下文，`priority` 影响策略门禁。

### 2. 把动作封装成 MCP 工具

触发之后，Agent 需要通过 MCP 工具来执行具体动作。建议强制区分只读工具和写工具。

例如先封装这些只读工具：

- `list_open_issues`
- `get_disk_usage`
- `list_recent_ci_failures`

每个工具声明里标记副作用：

```json
{
  "name": "list_open_issues",
  "side_effect": "read",
  "risk": "low"
}
```

写操作，比如 `restart_service`、`create_ticket`、`merge_pr`，单独放一组工具，默认不进入自动批准白名单。

### 3. 加一层 policy gate

这是 proactive agent 能不能落地的关键。不要直接让模型自由决定“该不该做”，而是用策略引擎做分诊：

```yaml
policy:
  rules:
    - if: tool.side_effect == "read" and trigger.priority != "critical"
      action: auto_approve
    - if: tool.side_effect == "write"
      action: propose
    - if: env == "prod" and tool.risk == "high"
      action: block
```

只读低风险任务可以自动执行；写操作必须通过 IM 或审核队列推给人确认；生产环境高风险动作直接 block，转人工处理。

### 4. 执行并回传结果

主动任务执行后不能静默结束。建议把结果回传到原始会话或通知频道，并用 `run_id` 关联。

对于需要确认的 `propose` 动作，可以推送一个带“批准/拒绝”按钮的消息，设置 10 分钟 TTL。过期自动关闭，避免任务悬挂。

### 5. 记录审计日志

每次触发至少记录这些字段：

```text
trigger_id / intent / policy_decision / tool_call / result / elapsed / user_action
```

没有审计日志，proactive agent 出问题后基本没法排查。

## 踩坑点

### 1. 触发过频，变成骚扰

一开始最容易翻车的是主动性过强。每天早上推十条“我帮你检查了以下内容”，用户很快就会关掉。

建议设置 quiet hours，例如工作日 9:00–19:00 之外不推非紧急主动消息；同时限制每日主动消息上限。

### 2. 写操作没卡住，误伤生产

把 `restart_service` 放进 auto approve，等于把生产环境交给一个不可控的循环。前期至少两周只跑只读工具，写操作全部走 propose。

### 3. 循环触发

proactive action 可能触发 webhook，webhook 又产生新 event，形成死循环。给每条 trigger 加幂等键，例如：

```text
idempotency_key = intent + object_id + date_hour
```

同一对象在同一周期内只执行一次。

### 4. 上下文膨胀

主动任务很容易把大量工具输出塞进模型上下文。建议工具输出先做摘要，原始日志只存文件或对象存储，把引用地址给模型即可。

### 5. 时区和 cron 的坑

容器默认 UTC，导致“早上 9 点”实际不是本地时间。cron 配置里显式指定 timezone，或者统一使用本地时区偏移。

### 6. 失败重试与通知

webhook 失败、工具超时、MCP server 短暂不可用都很常见。需要有指数退避和独立的 failure channel，不能只依赖主通知通道。

## 可复用建议

- 从 1 个只读场景开始，例如“每天早上汇总失败流水线”。
- 建立风险分级：`read-only/low`、`read+summarize`、`write/low`、`write/prod`，不同级别走不同策略。
- 主动不等于自动。前两周只开 propose 模式，观察误报率和用户的批准率。
- 工具接口尽量设计成幂等，方便安全重试。
- 用结构化日志贯穿始终，别只记自然语言。
- 给危险动作留 cancel 或 undo 的口子。

## 总结

Proactive 能力的关键不是模型突然“更主动”，而是把触发、工具、策略、通知、审计五个环节做成闭环。OpenClaw 作为编排底座，配合 MCP 工具和策略门禁，适合小团队落一个低风险、可回滚的主动执行循环。

先稳定地做对一件小事，再逐步扩大自动执行范围。这个顺序比一次接十个主动任务重要得多。

---

