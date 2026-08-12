---
title: Prompt 工程实战：如何为 Cron 任务编写可靠的 Agent Instruction
feedId: 32803
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景

在 OpenClaw / Agent + MCP 的自动化工作流里，越来越多的实践者开始用 LLM 接管定时任务，比如每天早上抓取财报摘要、每小时巡检服务健康状态、每周整理待办并推送到频道。理想很自然：用户只需用自然语言说一句“每天早上 9 点帮我总结 RSS”，Agent 就能记住、解析、注册，并准时执行。

但在真实落地中，这只是第一步——真正的挑战在于 **instruction 怎么写**，才能让 Agent 不把 cron 任务执行成一场薛定谔的闹钟：有时候不响，有时候响两次，有时候拿错工具，甚至直接回复“好的，我记下了”之后再也没有下文。

本文面向已经接触过 OpenClaw 或类似 Agent 框架，并在构建定时任务插件的工程师。我们不再重复“cron 是什么”，而是聚焦 **如何给 Agent 写指令，才能让定时任务在生产环境里不翻车**。

## 问题拆解：LLM 与时间的天然矛盾

LLM 没有时间概念。即便你在系统 prompt 里塞入 `{{current_time}}`，也仅仅是给了一个上下文标签。Agent 很容易在以下环节发生漂移：

1. **理解偏差**：将“每天 9 点”解读为“0 9 * * *”是理想情况，但模型可能悄悄补上一个非 UTC+8 的时区，或者把“每两小时”写成错误的 cron 表达式。
2. **触发判断不可靠**：如果让 Agent 自己在一个循环里读时间、判断是否触发，本质是在赌模型是否会正确比大小。结果往往是时区错乱、浮点秒数导致跳过、甚至因温度参数引入随机丢弃。
3. **执行阶段啰嗦且不可控**：真正触发后，Agent 会惯性输出“好的，现在为您执行……”，这些废话不但浪费 token，还可能在多轮对话里被当成新请求，产生幻觉链。
4. **静默失败**：工具调用报错后，如果没有显式错误处理指令，Agent 可能会输出“任务完成”，实际上什么都没发生。

解决以上问题的核心原则是：**不要用 LLM 做调度判断，让 Agent 只负责“被触发后干活”和“注册任务时输出结构化描述”**。

## 做法：两阶段 Instruction 设计

我们采用“配置时”与“触发时”分离的架构，由外部调度器（系统 cron、OpenClaw 内置调度或其他 runtime）负责真正的时间触发。Agent 只暴露两个能力：接受自然语言意图并注册任务；在收到触发信号后执行任务。下面给出这两个阶段的指令设计步骤。

### 1. 系统级环境注入

首先确保 Agent 每次推理都能拿到真实且明确的当前时间与默认时区。通常在系统 prompt 里写死：

```
Current time: {{current_utc_iso}} (UTC)
Default timezone for schedule: Asia/Shanghai
```

这里的关键是**不要只给一个时间戳就放手**，要同步声明时区，并强调“所有用户提供的时间若无时区则按默认时区处理”。同时，避免使用混合格式（如“现在是 2025-03-17”），优先提供 ISO 8601 字符串。

### 2. 注册阶段的 Instruction

当用户说“每天早上 9 点查天气”，Agent 的行为必须被指令约束为：
- 识别意图为 **schedule**
- 将自然语言时间映射为 cron 表达式 + 时区
- 输出结构化 JSON，绝不直接执行

推荐在 prompt 中写入如下规则：

```
When you need to schedule a task:
- Output a JSON object with exactly these keys: 
  action: "schedule" | "cancel" | "list"
  cron_expression: string (standard 5-field cron)
  timezone: string (IANA identifier, default "Asia/Shanghai")
  task_description: string (plain text of what to do when triggered)
  tool_name: string (the MCP tool to call, optional)

- If user doesn't specify time, default to 09:00.
- For "every day", use "0 9 * * *". For "every hour", use "0 * * * *". 
- Always ask confirmation before using midnight ("0 0 * * *") to avoid unintended late-night execution.
- Do NOT call any tool at this stage; only output the JSON.
```

依靠这个约束，Agent 不再自行决定何时运行，而是生成一份供调度器消费的任务描述。如果模型能力足够，还可以让它附带一个 `next_run` 作为给调度器的友善提示，但这不是关键。

**踩坑注记**：部分 Agent 在看到 `task_description` 后会不自觉展开幻想的执行过程，比如在 JSON 里嵌套一个预执行结果。解决方案是在 instruction 里加一句：“task_description must be a concise instruction, not a simulation result.”

### 3. 触发阶段的 Instruction

调度器在约定时间拉起 Agent，并向其上下文注入一段明确的触发标记。触发时使用的系统 prompt 应该完全不同于对话模式，我建议单独维护一个“执行快照”：

```
[SYSTEM]
You are a task executor, not a conversational assistant.
You have been triggered by a scheduled task.
Task: {task_description}
Current time: {current_time}

Rules:
- Execute the task immediately using the specified tool. 
- Do not reply with "Sure, let me...", "I'll do that now", or any conversational wrapper. Only output the result or the tool call.
- If a tool call returns an error, retry exactly once after 2 seconds. If it fails again, output: "ERROR: {error_message}".
- Do not invent new tasks, ask questions, or produce summaries unless explicitly requested in the task description.
```

如果将这段 prompt 与对话 prompt 混合，Agent 极易被之前的对话历史带偏。例如刚才还在聊天，突然来一句“执行任务 A”，它可能回复：“我刚才已经告诉过你了呀”。所以强烈建议：**为定时任务执行单独维护上下文，或至少用 `<task trigger>...</task trigger>` 标签将历史隔离**。

### 4. 幂等与防重

调度器触发时，要向上下文写入一个唯一 `run_id`，并要求 Agent 在任务结果中包含该 ID。这一步不是为了 Agent，而是为了让外部调度器进行事后幂等校验。指令可写为：

```
Include the run_id "{run_id}" at the beginning of your response, like:
[run_id: xxx]
... result ...
```

外部调度器可据此判断是否重复，避免幽灵执行。

## 踩坑点汇总

在实际部署中，我们还踩过以下典型坑：

- **时区静默丢失**：模型在生成 cron 时省略了 timezone 字段，调度器默认用 UTC，导致任务在凌晨三点而不是早上九点执行。修复方式：在 prompt 中规定“若不填 timezone 则拒绝注册”，并提示“你用哪个时区？”
- **工具名幻觉**：Agent 注册的工具名与实际 MCP 暴露的名称不一致，触发时直接报错。除加强对 task_description 的约束外，可以在注册阶段做一次工具名校验（前端调度器校验）。
- **连续触发下的对话残留**：调度器每小时触发一次，但上下文未清空，Agent 把十次结果累积成一篇冗长的“执行报告”，超出 token 限制。解决方式：触发上下文只保留本次任务，不计入历史。
- **“收到”式的假完成**：未加“不要寒暄”指令，Agent 回复“任务已完成”但实际未调用工具。缓解方法：在触发 instruction 中明确要求 output 必须包含工具返回的数据，否则视为失败。

## 可复用建议

最终我沉淀出一份可复用的双模式 instruction 模板，经若干生产任务验证。使用时只需把 `{{}}` 占位符替换为实际值即可：

**注册模式 (用于对话)：**
```
You are a task scheduling assistant.
Current time: {{now_utc}} (Default TZ: {{tz}})
When user asks you to schedule a task, output a JSON object:
{
  "action": "schedule",
  "cron": "...",
  "timezone": "...",
  "task": "clear instruction for execution stage",
  "tool": "exact_tool_name"
}
Do not execute anything now. Only output JSON.
```

**执行模式 (用于调度器触发)：**
```
You are a no-talk executor.
Task: {{task}}
run_id: {{run_id}}
- Call the tool exactly as described in Task.
- Output only: [run_id: {{run_id}}] <tool result>
- On error: retry once, else output "ERROR: <msg>".
```

这份模板可以无缝嵌入到 OpenClaw 的自定义 agent 配置、MCP 插件的系统 prompt 或任何支持变量替换的框架中。

## 总结

让 Agent 执行 cron 任务，最根本的矛盾在于 LLM 不可预测的时间感知与精确调度要求之间的差距。务实方案不是让模型更“聪明”地读懂时间，而是把调度权彻底交出，由确定性的 runtime 保证，Agent 回归工具执行者的本分。这种“调度外移、指令收敛”的模式同样适用于 webhook 触发、消息队列消费等异步场景。

如果你正面对类似的定时任务设计，不妨从这份双阶段 instruction 模板开始，再根据你的工具生态和调度器能力做渐进扩展。把对 LLM 的期望，从“按时做事”校准到“被叫到时好好做事”，调度类的自动化才能从 Demo 走进日常运维。

---

