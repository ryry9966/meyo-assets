---
title: Prompt 工程实战：Cron 任务指令的正确写法
feedId: 31935
source: 综合讨论
publishedAt: 2026-08-07
---

# Prompt 工程实战：Cron 任务指令的正确写法

## 背景

在 OpenClaw、MCP 或各类 Agent 编排平台中，将周期性工作交给 Agent 自动执行已经成为常见需求——定时生成日报、轮询接口告警、定期汇总数据。这些场景依赖一个关键概念：**Cron 任务**。然而，与在服务器上直接写 crontab 不同，Agent 需要通过自然语言或结构化 prompt 来理解任务规则。很多开发者在写这类 instruction 时仍然沿用“每天上午发一份报告”这种模糊说法，导致 Agent 行为偏离预期：时区错误、重复触发、静默失败。本文结合真实踩坑经验，讨论如何为 Agent 设计可靠的 Cron 任务指令。

## 问题：模糊指令导致不可靠定时

以下是一个典型的反面案例：

> “每天早上给我发一封邮件，汇总昨天的订单数据。”

这个 instruction 看似清晰，实际藏着多个陷阱：

- **时间定义不清**：“早上”究竟是 7:00、8:00 还是 9:00？Agent 可能默认使用 UTC 零点，导致在北京时间早上 8 点才触发。
- **时区缺失**：调度器可能运行在 UTC 环境，而用户期望北京时间。如果不显式声明时区，定时一定偏移。
- **失败无感知**：如果数据源不可用或邮件发送失败，会不会重试？用户可能第二天才发现昨天没报表。
- **任务幂等性差**：Agent 若重复执行同一“昨天”的逻辑，在跨时区或调度抖动时可能发送多份报告。

要让 Agent 稳定执行周期性任务，必须将模糊指令转译成**包含触发器、环境、校验、失败策略的工程化描述**。

## 做法：编写工程化 Cron Instruction

### 1. 用显式 Cron 表达式替代自然语言

在 prompt 中直接提供标准 5 位或 6 位 cron 表达式，并要求 Agent 严格按照表达式执行。例如：

```
Cron Expression: 0 9 * * 1-5
```

并注明是否支持秒级字段，避免 Agent 自行“理解”成其他规则。若平台只支持自然语言触发，也要把语言固化为“每天 09:00 Asia/Shanghai”，减少歧义。

### 2. 绑定时区声明

在所有定时相关描述中加入：

```
All scheduled times are in Asia/Shanghai (UTC+8).
```

这会让 Agent 在解析和执行时将时区作为必须考虑的参数。某些 Agent 框架允许在任务定义中设置 timezone 属性，应优先使用该能力。

### 3. 结构化任务定义模板

将任务拆解成**预检、主任务、失败处理、通知**四个部分，像写函数一样写 prompt。以下是一个通用模板：

```markdown
## Scheduled Task Definition

- **Cron**: `0 8 * * *` (daily at 08:00)
- **Timezone**: Asia/Shanghai (UTC+8)
- **Pre-execution Checks**:
    - Verify target API health endpoint returns 200. If unhealthy, retry every 60s up to 3 times.
    - Ensure yesterday's date partition exists in the data warehouse.
- **Main Task**:
    1. Query yesterday's order metrics from the internal BI API.
    2. Generate a summary Markdown report with a table of top 10 products.
    3. Send the report via email to ops@yourdomain.com using the configured SMTP channel.
- **Failure Handling**:
    - If pre-checks fail after retries, skip this run and post a warning message to Slack #ops-alerts with the failure reason.
    - If main task fails, send an error notification and do NOT retry automatically to avoid duplicate emails.
- **Confirmation**: Before activation, output the next 3 scheduled run times in the specified timezone so I can manually verify.
```

将这个模板直接粘贴进 Agent 的 system prompt 或任务描述中，能极大减少误解空间。

### 4. 增加“自检确认”步骤

要求 Agent 在接收任务后、首次启动前，输出它理解后的执行计划，例如：

> “I will run this task at 2025-03-15 08:00:00 Asia/Shanghai, then 2025-03-16 08:00:00 …”

这相当于一次“干跑”验证，帮你提前发现 cron 含混、时区错误等问题。

### 5. 设计可测试路径

在正式启用 Cron 前，添加一条手动触发策略：

```markdown
For testing, I can say: "Run the scheduled task now (once, in test mode)" and you MUST execute the main task immediately, ignoring the cron schedule, but still respecting timezone and failure handling.
```

通过一次手动执行，你可以验证数据接口、权限、输出格式是否正常。

## 踩坑记录

以下是从实际排障中总结出的高频问题：

- **Agent 忽略秒级 cron**：很多 LLM 默认只处理 5 位 cron，当你提供 6 位时，它可能把第一位当作分钟，导致频繁触发。要么统一采用平台支持的标准，要么在提示中明确“This cron uses 5 fields: minute hour day month weekday”。
- **“每天”被误解为固定间隔**：让 Agent 自己计算“每 24 小时”，可能因执行延迟产生漂移。务必使用 wall-clock 触发的 cron 语义。
- **任务重叠**：如果任务执行时间可能接近调度周期（比如每小时跑一次，但任务耗时 70 分钟），需要在 instruction 中增加防并发机制：“If a previous instance is still running, skip this run and log a warning.”
- **依赖服务短时不可用**：来自动补数据或发送警报时，网络抖动会导致一次执行失败。没有重试策略会让定时任务形同虚设。但重试必须带间隔和上限，避免雪崩。
- **日志不可见**：Agent 的执行日志如果只存在于云端，排障会很痛苦。在 instruction 中要求 Agent 每次执行后记录简要日志，并通过通知渠道输出关键状态。

## 可复用建议

1. **为团队沉淀一个 Cron Prompt 库**：把常用任务（数据快照、报表、健康检查）的 instruction 模板保存在文档中，使用时仅修改业务逻辑部分。
2. **将 Cron 配置和任务逻辑解耦**：在 OpenClaw 等支持插件的环境中，保持 schedule 由平台配置，Agent 只负责执行单次任务；或反之，用 prompt 管理全部逻辑，但必须确保平台不会因相互冲突而重复调度。
3. **加入监控**：即使 Agent 按 cron 运行，仍要通过外部手段监测“任务是否真的执行了”。例如，让任务最后一步写入一个时间戳到数据库，再由外部监控这个时间戳的新鲜度。
4. **限制可变部分**：让 prompt 中只有数据源、收件人等参数可变，结构保持固定，降低 Agent 创造性修改调度规则的风险。

## 总结

把 Cron 任务交给 Agent 不等于说一句“每天自动做某事”就行。你需要像对待生产环境的 crontab 一样，显式定义**触发时间、时区、预检、主流程、失败策略和观测方式**。一个好的 Cron instruction 是一个**确定性系统的入口**——它用工程语言消除了 Agent 的猜测空间。当你下一次写定时任务 prompt 时，不妨拿出上面的模板，先确认这几点是否都覆盖到了。花 10 分钟打磨 instruction，可能帮你避免无数次凌晨的报警电话。

---

