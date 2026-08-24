---
title: AI 助手的 proactive 实践：不等指令，先把巡检和提醒做稳
feedId: 34550
source: 综合讨论
publishedAt: 2026-08-24
---

# AI 助手的 proactive 实践：不等指令，先把巡检和提醒做稳

## 背景

多数 agent 仍然是被动执行：用户发指令，助手调用工具，返回结果。接入 OpenClaw/MCP/插件后，很多重复性工作其实不需要等用户开口——磁盘容量、证书过期、CI 失败、未读工单、异常日志，这些都可以由助手主动检查并汇报。

但 proactive 容易被做成一个“会自己动的黑盒”。它真正的工程难点不是“能不能定时跑”，而是：触发是否可控、判断是否可解释、动作是否可回滚、误报是否能收敛。否则，一个过于主动的助手很快会被关掉。

## 问题

直接把“定时任务 + LLM 判断 + 自动执行”拼起来，通常会遇到四类问题：

1. 触发方式单一：只有 crontab，漏掉事件驱动。
2. 重复触发：没有状态去重，同一个异常每小时报一次。
3. 误报和打扰：没有冷却、静默时段、干跑模式。
4. 权限过大：自动执行删除、发布、写库等高风险操作，回滚困难。

因此在工程上，我更倾向于把 proactive 拆成三层：Trigger、Policy、Action。

## 做法

### 1. 用配置声明触发和动作

在 OpenClaw 这类可挂 MCP 的 agent 项目里，不建议把 proactive 逻辑硬编码进插件。可以维护一份 `proactive.yml` 或 JSON，每个任务声明触发源、检查工具、命中条件、动作、冷却和静默时段。

示例：

```yaml
tasks:
  - name: check_disk_usage
    trigger:
      type: cron
      interval: "30m"
    check:
      tool: mcp.server.read_disk_usage
    policy:
      condition: "usage_percent > 85"
      cooldown: "6h"
      quiet_hours: "23:00-08:00"
    action:
      type: notify_only
      channel: "ops"
      message: "disk usage > 85%"
      dry_run: true
```

这个配置的关键点：检查工具只读，动作默认 `notify_only`，并先开 `dry_run`。

### 2. 事件优先，轮询兜底

能接 webhook 的优先事件驱动，例如 CI job 失败、issue 更新、文件变更。事件驱动延迟低，也不容易造成无意义轮询。轮询适合无法推送的状态，例如磁盘水位、证书剩余天数、服务健康检查。

不要用 1 分钟轮询去模拟实时事件。事件驱动的任务和轮询任务应分开管理，避免一个全局循环把系统拉高。

### 3. 状态与去重

proactive 最容易翻车的地方是“同一件事重复报”。不要只判断当前值，要给任务保存上次状态 fingerprint。可以是上一次异常值、hash、时间戳。只有状态发生变化，或超过 cooldown 才再次通知。

例如磁盘 86% 报警后，不能每 30 分钟再报一次。可以让 fingerprint 包含 `usage_percent` 分桶值，例如按 5% 分桶；或直接记录已通知状态，直到回落到阈值以下再重置。

### 4. 先干跑，再通知，最后动作

建议分阶段启用：

- 第一阶段：`dry_run=true`，只写日志“如果启用会做什么”；
- 第二阶段：`dry_run=false`，`action.type=notify_only`，只通知到群或频道；
- 第三阶段：低风险动作，例如刷新缓存、重建索引、创建工单；
- 最后才考虑自动写操作，并要求幂等、可审计、有 kill switch。

复杂判断建议只通知，不自动执行。例如“CI 失败且最近三次都失败且负责人不在线”，这种条件交给模型判断可以，但自动执行带有不确定性，应只给建议。

## 踩坑点

1. **先做全局开关**：每个任务要有 `enabled`，整个 proactive 系统要有一个总开关。否则半夜故障时只能去关服务。
2. **动作幂等**：自动创建的工单、发送的消息、触发的 job 都可能重复。动作侧要做唯一键或去重。
3. **静默时段不是摆设**：Ops 通知要默认尊重 23:00-08:00，紧急级别任务再破例。
4. **MCP 工具命名语义**：建议 `check_*`、`read_*` 只读，`act_*` 才是副作用动作。让配置里一眼看出风险。
5. **记录 run_id 和 trace**：每次检查要有 run id、触发时间、使用工具、输入输出摘要、最终 decision。问题排查时能还原“它为什么做了这个”。

## 可复用建议

- 把 proactive 任务当成 Git 仓库配置管理，变更走 review。
- 在群里先推“冷静通知”：一条消息包含检查值、阈值、建议动作，而不是“我已自动处理”。
- 灰度：先在个人频道或测试群跑 7 天，观察误报率，再开放到团队。
- 给模型设定明确的决策边界：低风险、可回滚、可解释的动作才允许自动执行。
- 保留一键暂停入口，例如 `/proactive off` 或环境变量开关，比去改配置更快。

## 总结

proactive 能力不是让模型“更主动”，而是用触发器、策略和边界把主动性关进可控的笼子。一个能稳定预报磁盘水位、CI 失败和异常日志的助手，远比一个自动做高风险的“智能代理”更有工程价值。先把“通知”做稳，再谈“代办”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/4bdfb9dcdfa78c2c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/624425fb3ec9daff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/43c609fbde798730.png)

