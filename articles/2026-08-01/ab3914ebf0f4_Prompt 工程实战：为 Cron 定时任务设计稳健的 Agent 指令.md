---
title: Prompt 工程实战：为 Cron 定时任务设计稳健的 Agent 指令
feedId: 31150
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：为什么 Cron 任务的 instruction 值得单独讨论

在 OpenClaw、Agent 编排、MCP 插件等自动化场景里，经常需要让 Agent 按固定周期执行任务：每日生成行业简报、每半小时检查 API 健康状态、凌晨拉取数据并写入数据库。这类需求通常通过 **Cron 触发器** 调用 Agent 实现。  
但与传统代码编写的 Cron Job 不同，Agent 每次触发只拿到一段静态的 `system instruction`（以及平台注入的少量变量），没有历史会话、没有记忆。意味着这段 instruction 必须同时完成**任务描述、上下文注入、输出约束、错误处理**，否则 Agent 很容易变得不可靠。

## 常见问题

- Agent 忘记当前时间或时区，做出错误判断  
- 因为没有状态，每次运行都重复处理同一条数据，或反之遗漏增量  
- 输出格式飘移：今天是纯文本，明天是 Markdown，下游解析失败  
- 访问外部 API 失败后，Agent 开始“合理编造”数据（幻觉）  
- 指令过长导致模型忽略关键约束，只执行前几条要求  

这些问题归根结底源于一个事实：**Cron Agent 是一个无状态批处理任务，而不是对话机器人。** instruction 的设计必须充分防御性。

## 做法与步骤：逐个拆解 instruction 的要素

### 1. 先定义清晰的任务边界与角色
example:
```
You are a scheduled monitoring agent. Your only job is to call the status API, compare the result with the threshold, and produce a JSON report. Do not add commentary or ask questions.
```
避免“你是一个有帮助的助手”这种泛化开场，它会鼓励模型发散。

### 2. 强制结构化输出
在 instruction 中锁定输出格式，最好附带一个 **最小示例**。
```
You MUST output a single JSON object with exactly these keys: "status", "checked_at", "summary". Example:
{"status":"OK","checked_at":"2025-04-10T09:00:00Z","summary":"All endpoints healthy."}
```
如果怕 Agent 在 JSON 前后加说明文字，可以加上：
```
Reply ONLY with the JSON object. No preamble, no markdown code fences.
```

### 3. 注入动态上下文（核心）
Cron 任务缺少环境感知，所以要通过平台变量或工具主动提供：
- **时间与时区**：使用 `{{NOW}}` 或 `{{CURRENT_DATETIME_UTC}}` 等变量，并明确时区。
- **上一次运行状态**：如果平台支持持久化变量，注入 `{{LAST_RUN_STATUS}}` 和 `{{LAST_PROCESSED_ID}}`，避免重复处理。
- 若平台不支持变量，可在 instruction 中要求 Agent 通过 Shell 工具执行 `date -u` 获取时间，或使用 MCP 文件系统工具读取 `.last_run` 标记文件来获得上次处理位置。

### 4. 设计错误处理策略
example:
```
If the API call fails or times out, set status to "ERROR" and summary to "API_UNREACHABLE". Do NOT guess any data. If the file to read doesn't exist, treat it as first run.
```
要求 Agent 在特殊情况下返回统一定义的错误码，便于下游告警。

### 5. 控制长度，防止输出截断
一旦 Agent 输出过长可能被平台截断或超时。明确指示：
```
Keep the summary under 200 characters. Do not include raw API response.
```

### 6. 使用 MCP 工具实现状态持久化
如果任务需要增量处理（如邮件读取），让 Agent 把已处理 ID 列表写进文件，并每次都先读取。instruction 示例：
```
Use the MCP filesystem tool to read /data/processed_ids.txt. Skip any IDs found there. After processing, append new IDs to the file.
```
这样即使 Agent 无记忆，也能在工作区文件系统上维持状态。

## 踩坑记录

- **指令内容冲突**：一边要求“总是输出JSON”另一边说“如果状态异常就发邮件说明”，模型容易既输出 JSON 又生成邮件内容。统一成单一输出通道。
- **时区陷阱**：`{{NOW}}` 不带时区信息，Agent 会假定为 UTC，而业务时间是东八区，导致“每日”任务在错误的日期执行。解决方案是 instruction 中硬编码 `All times are Asia/Shanghai (UTC+8)`。
- **并发触发**：如果上一次任务执行时间过长，Cron 下次触发会再起一个实例，两个实例同时写文件可能损坏状态。需在 instruction 中结合文件锁（如 `flock`）或平台并发策略设置 `maximum concurrency: 1`。
- **幻觉结果**：Agent 可能为满足输出要求而虚构一个看起来合理的“正常”值。务必在 instruction 中强调数据来源，例如 `summary MUST be derived from the actual API response. If you didn't receive a response, use "NO_DATA".`

## 可复用建议

1. **制作 instruction 模板**，用占位符抽出可变部分，如 `{{TASK_DESCRIPTION}}`、`{{OUTPUT_FORMAT}}`，配合脚本或平台变量渲染。  
2. **将输出监控起来**：把 Agent 输出定向到日志文件或 Webhook，再用简单的脚本检查 JSON 中的 `status` 字段，发现连续 ERROR 就告警。  
3. **先干跑再上线**：利用平台的“手动触发”或 dry-run 功能，反复测试至少 5-10 次，观察输出的一致性和边界。  
4. **保持 instruction 精简**：超过 500 词后很多模型会开始丢失约束，把长指令拆分成 “角色+核心规则+示例”，其余通过外部引用或工具文档补充。  
5. **把 Cron Agent 看作“无状态函数”**：每次运行都应能从环境（变量、文件、API）重建全部所需上下文，而不依赖任何内部记忆。

## 总结

为 Cron 任务写 instruction 本质上是在 **用自然语言定义批处理作业的契约**：给定输入（变量、工具），产生确定性的输出，并明确异常行为。守住“无状态、结构化、显式上下文”三条原则，就能大幅降低 Agent 在生产中失控的概率。在 OpenClaw 这类 Agent 编排平台上，稳扎稳打的 instruction 设计比堆叠复杂工作流更能提升自动化可靠性。

---

