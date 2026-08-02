---
title: Prompt 工程实战：Cron 任务应该怎么写 instruction
feedId: 31371
source: 综合讨论
publishedAt: 2026-08-02
---

# Prompt 工程实战：Cron 任务应该怎么写 instruction

## 背景：Agent 有了定时能力，然后呢？

无论你是在 OpenClaw 里用内置的 Cron 调度器，还是通过 MCP server 把定时触发接入 Agent，亦或是在插件维度实现自动化巡检，有一件事绕不过去——**你写进 prompt 里的 instruction，会直接决定定时任务能不能跑对**。

一个典型的场景是：你给 Agent 配了一条 cron `0 9 * * 1-5`，下一条 system message 写着“每周工作日上午 9 点检查网站健康状态，如果异常就把错误码发到钉钉群里”。看起来没问题，但一周之后你会发现：

- 任务在 UTC 09:00 执行，等于北京时间下午 5 点；
- Agent 每次触发时都要重新理解“健康状态”到底是 HTTP 200 还是响应时间低于 500ms；
- 某天请求超时，Agent 连续 fail 了 6 次，往群里刷了一屏幕同样的报警；
- 上下文窗口里塞满历史调度信息，Agent 把三天前的失败记录当成了当前任务的一部分。

问题不是 cron 表达式不对，是 prompt 没把调度行为的边界和契约说清楚。

## 问题拆解：你的 instruction 少了什么

如果你把 Cron 任务视为“定时往对话里塞一条消息”，那 Agent 的表现天然是不稳定的。因为在缺少约束的情况下，它每次都会用自己的角色知识去重新解释任务，而这种重新解释会在边缘条件（时区、夏令时、网络抖动、历史上下文污染）下产生偏差。

具体到 instruction 层面，常见缺漏包括：

1. **时区信息没有显式声明** — Agent 的默认时区可能是 UTC、服务器本地时间，或跟着对话上下文走，这会导致触发时刻和预期错位；
2. **任务语义描述过于自然语言化** — “早上检查一下数据库”在 10 次执行里可能产生 4 种不同的行为；
3. **缺少幂等与去重逻辑** — 调度框架可能因异常重试而短时间多次触发，没有去重就会产生告警风暴；
4. **没有定义输出裁剪规则** — 每次执行都保留完整上下文，窗口会被无效历史撑爆，Agent 行为逐渐漂移；
5. **错误处理模糊** — 一句“失败就告警”没有给出重试次数、退避策略和 silence 规则，导致故障识别混乱。

指令的本质不是下任务，而是**定义一个可复现的执行边界**。

## 做法与步骤：为 Cron 任务设计结构化 instruction

最好的思路是把 Cron 任务的 instruction 当成一个配置文件来写，而不是一句人类对话。在 OpenClaw 或类似 Agent 环境里，我推荐在 system prompt 中为每个调度任务引入一个独立的模块，用 YAML 风格的注释或 Markdown 代码块来承载。

下面是一个可以直接用的模板（以“工作日早上 9 点（北京时间）检查核心 API 健康状态”为例）：

\`\`\`markdown
## Cron Task: health-check-api

- **cron_expression**: `0 1 * * 1-5`
- **timezone**: `Asia/Shanghai` (UTC+8)
- **trigger_time_comment**: 北京时间上午 9:00，周一到周五
- **task_id**: `health-check-api`

### Execution Rules
1. **Time gate**: 执行前先获取当前北京时间（可通过工具 `get_current_time("Asia/Shanghai")`），确认小时字段为 9，否则直接输出 `schedule_skip: true` 并忽略本次调度。
2. **Pre-check**: 检查是否存在上次同 task_id 执行结果，如果上次执行在 5 分钟内且状态为 `completed`，则跳过本次调度，输出 `deduplication: true`。
3. **Core steps** (顺序执行):
   - 步骤 1：调用 `http_get("https://api.example.com/health")`，超时 5 秒。
   - 步骤 2：判断返回码 200 且响应体 `status` 字段为 `ok`，记 `healthy=true`。
   - 步骤 3：如果 `healthy=false`，调用 `send_alert(tool="dingtalk", content=...)`，最多重试 2 次，间隔 10 秒。
4. **Output format**: 任务结束后，你只能输出以下 JSON 结构，不能有任何多余自然语言：
   ```json
   {
     "task_id": "health-check-api",
     "timestamp": "<ISO8601>",
     "healthy": true/false,
     "status_code": <number>,
     "alert_sent": true/false,
     "retry_count": <number>,
     "deduplication": false,
     "schedule_skip": false
   }
   ```
5. **Error handling**: 如果工具调用失败且重试耗尽，设置 `healthy=false`，`alert_sent=true`，并在输出中完成报警，同时设置 `error_message` 字段（仅在此场景下允许附加该字段）。
6. **Context management**: 本 task 的执行输出在生成后立刻完成作用，后续对话中除非明确引用，否则你必须忽略历史中该 task 的所有输出，避免干扰新的调度。
\`\`\`

我给的这个模板里，几个关键设计是：

- **时间闸门（time gate）**：主动校对时间，避免时区问题和调度器误触发；
- **去重窗口（5 分钟）**：利用存储工具或上下文标记，防止同任务重复跑；
- **结构化输出**：把自然语言驱离出执行结果，减少歧义，也方便外部系统解析；
- **显式忽略历史**：在 system prompt 中明确指示 Agent 忽略过去调度记录，避免上下文爆炸。

## 踩坑点与实战经验

在实际跑了大半年之后，有几个坑值得单独拿出来说：

**坑 1：把 cron_expression 解释权交给 Agent**  
如果你写 “每周一 9 点”，Agent 或者调度框架可能会做本地化解释，甚至引入夏令时偏移。永远在 instruction 里提供明确时区，并通过工具获取绝对时间做 gate，不要靠 crontab 所在的宿主环境保证。

**坑 2：告警去重靠“人工记忆”**  
Agent 不知道上次发过告警，除非你明确给了它去重逻辑。一种方案是在 system prompt 中写死“两次告警间隔不少于 30 分钟”，另一方案是让 Agent 在执行前检查存储中的上次告警时间。后者更可靠，但在无状态环境里需要引入外部 KV 存储（可通过 MCP server 实现）。

**坑 3：输出混进对话导致功能退化**  
如果你允许每次执行结果以自然语言流回对话，很快上下文里全是“任务已完成”、“报警已发送”之类的噪音。Agent 甚至会在处理其他问题时引用这些历史。强制使用 JSON 输出并标记 `task_id`，配合 system prompt 里的“忽视历史调度输出”，是一个低成本解法。

**坑 4：不区分首次执行和恢复执行**  
容器重启后某一时刻可能积压多个调度，Agent 会被同时投喂多条 cron 消息。如果不做 id 去重和时间窗保护，可能出现并发执行脏数据。在上面的模板里，`task_id` 和 5 分钟窗口起了隔离作用。

## 可复用建议

如果你维护多个 Agent 或者要交付自动化能力给团队，可以考虑下面三个通用实践：

1. **提取 Cron Prompt 模板库**：把上面的 Markdown 块沉淀成一个通用模板，团队新增定时任务时只需填写 `cron_expression`、`timezone`、`steps` 等字段，其他规则一律固化。
2. **为调度指令引入专门的校验工具**：写一个简单的 MCP tool 叫 `validate_cron_instruction`，格式不符合模板时直接拒绝注入 system prompt，减少因人类随意粘贴导致的漂移。
3. **监控调度产出而非调度日志**：把 Agent 输出的 JSON 落到时序数据库或日志系统里，监控 `healthy` 字段和 `alert_sent` 频率，而不是在聊天界面里寻找定时任务的健康度。把 Agent 当成一个函数来看待。

## 总结

Cron 任务的 instruction 本质上是在 Prompt 层写一段“无副作用、高确定性”的执行脚本。只有当你把时区、去重、输出格式、错误契约、上下文隔离这五个维度都约束死后，Agent 的定时行为才会从“偶尔惊喜”变成“永不出奇”。

好的调度 prompt 读起来不会像人类的请求，而更像一份交付给执行器的规格书。**那份规格书，就是你日后不头痛的保险单。**

---

