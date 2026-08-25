---
title: 让 AI 助手“主动一点”：OpenClaw 下的 proactive 巡检实践
feedId: 34698
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

大多数 Agent 工作流仍是被动响应式：用户发一条消息，Agent 执行一次任务。但在运维、协作和自动化场景里，很多动作本来就不该等人开口——比如“每天 9 点查看昨天未关闭的高优先级 issue”“监控某个部署状态，失败后通知负责人”“每周一生成上周变更摘要”。

这类需求如果硬塞进聊天窗口，就变成用户每天手动提醒 Agent 干活。更实际的做法，是让 Agent 具备 proactive 能力：由时间、事件或外部信号触发，不等用户开口，主动完成“检测-判断-执行”的最小闭环。

## 问题

Proactive 的难点不是“能不能主动”，而是主动之后如何可控、可回滚、可审计。如果直接给 Agent 一个 cron 加上写工具权限，很容易出现误判、重复执行、写操作不可逆、失败无人感知等问题。

所以真正要解决的，是一个工程化边界：哪些触发源可信？哪些动作可以自动执行？哪些必须人工确认？执行结果如何追踪？

## 做法/步骤

下面是我在 OpenClaw 环境里实践 proactive 巡检任务的一套最小框架，核心链路是：**调度器触发 → headless Agent 决策 → 结构化输出 → 受控执行 → 审计记录**。

### 1. 先选一个足够窄的触发源

不要一上来就做复杂事件驱动。建议从一个固定的定时任务开始，比如每 30 分钟或每天 9 点。触发方式用系统 cron、systemd timer 或 CI 定时任务都行。

例如：

```bash
*/30 * * * * /usr/local/bin/openclaw-run proactive-issue-check
```

### 2. 准备 MCP 工具，明确读写边界

通过 MCP server 暴露两类工具：

- **只读工具**：查询 GitHub issue、查询部署状态、读取监控指标。
- **写工具**：打标签、发送 Slack 通知、创建 issue 评论。

写工具最好放在独立的 MCP server 里，并使用最小权限 token。这样即使 Agent 决策失误，影响面也被限制在少数几个低风险动作内。

工具描述要写清“只读”还是“写”，例如：

```yaml
name: list_stale_issues
description: List open issues not updated in the last 24 hours. Read-only.
```

不要让 Agent 自己猜工具的副作用。

### 3. 设计决策 prompt 和结构化输出

给 Agent 的 system prompt 要明确三件事：

- 你只能使用哪些工具；
- 判定标准是什么；
- 默认不做任何写操作，除非满足特定条件。

同时要求输出 JSON，而不是自由文本。例如：

```json
{
  "should_act": true,
  "action": "notify_slack",
  "targets": ["issue #123"],
  "reason": "high priority issue not updated for 26 hours"
}
```

程序解析这个 JSON 后，再决定是否调用对应写工具。这样 Agent 不会直接拿着写工具乱跑，执行权还在外层。

### 4. 受控执行，默认 dry-run

首轮实现只让 Agent 返回“建议动作”，不直接执行写操作。比如先发一个摘要到运维频道，或者写入本地日志。跑稳定一周后，再对低风险动作放开自动执行。

对于写操作，可以在外层做一次规则引擎校验：只有 action 在白名单内、targets 数量小于阈值、reason 长度符合要求时，才真正调用写工具。

### 5. 失败通知与审计

Proactive 任务最大的问题是“没人盯着”。所以每个定时任务都要：

- 检查退出码；
- 失败时发送告警；
- 记录每次运行的 trace id、触发时间、Agent 输出、最终执行动作。

这样问题发生后能回溯，而不是只知道“昨天好像有个任务没跑”。

## 踩坑点

### 1. 模型会“猜”

如果上下文里没有实时数据，Agent 可能根据记忆编造 issue 状态或部署结果。必须强制 Agent 通过工具获取真实数据，不要让它凭感觉生成结果。

### 2. 写操作不可逆

自动打标签、自动发通知这类操作一旦误触发，虽然能撤回，但会造成打扰。一定要默认 dry-run，生产环境写工具单独隔离。

### 3. 重复触发和并发

定时任务可能重叠执行，导致同一个 issue 被处理两次。可以用文件锁、数据库唯一键或调度器自身保证单实例运行。

### 4. 上下文噪声

不要把整个公司的监控都塞进一个 proactive 任务。任务域要窄，上下文越干净，Agent 决策越稳定。

### 5. 失败无人关注

定时任务失败往往几天后才发现。必须配失败告警，并且告警通道要能触达真人，而不是又回到 Agent 自己。

## 可复用建议

- **采用“检测-建议-执行”三级**：只读检测可以全自动，写动作默认人工确认或规则引擎确认。
- **所有 proactive 任务输出统一 schema**，方便后续统计误报率、动作分布和失败原因。
- **从低频、只读任务开始**，例如每天生成摘要，验证稳定后再逐步放开自动写操作。
- **把 trigger、agent session、tool call 日志关联到同一个 trace id**，排查问题会快很多。
- **工具描述要清晰**，写清参数、返回值和是否只读，能明显降低 Agent 误调用概率。

## 总结

Proactive 能力不一定需要复杂的多 Agent 协作或事件流平台。一个定时调度器 + 一个 headless Agent 会话 + 一套受控 MCP 工具，就能覆盖不少日常巡检和提醒场景。

关键是克制：先让 Agent“看见并说”，再逐步让它“动手”。边界清晰、审计完整、失败可感知，这个 proactive 助手才值得长期跑下去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/8408a2ade699c7bd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/5df17da8faf01d3e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/0becfad2af12f4eb.png)

