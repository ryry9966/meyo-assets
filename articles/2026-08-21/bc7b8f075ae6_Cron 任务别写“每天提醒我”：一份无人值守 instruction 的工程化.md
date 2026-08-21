---
title: Cron 任务别写“每天提醒我”：一份无人值守 instruction 的工程化拆解
feedId: 33991
source: 综合讨论
publishedAt: 2026-08-21
---

在 OpenClaw / Agent 自动化里，cron 任务从“定时触发”变成“定时执行一个有工具、有记忆、可能产生外部副作用的 agent”。这时 instruction 不是聊天开场白，而是唯一运行规范。

## 背景

定时任务触发时通常没有人在场，agent 无法像对话场景那样实时追问：“你说的最新消息是指 RSS 还是邮件？”“失败后要不要重试？”“输出发给谁？” 它只能用你写下的 instruction 做判断。因此，cron 任务的 prompt 必须按无人值守 runbook 标准来写，而不是按对话指令来写。

## 问题

实际使用中，cron 任务常见的失败可以归结为三类：

1. **目标含糊**：写“看看有没有重要更新”，agent 可能摘要、发消息、写文件都做，最后动作不可预期。
2. **边界不清**：没说明是否允许发网络请求、写数据库、调用 MCP 工具，导致越权操作或该做的不敢做。
3. **失败路径缺失**：没写失败怎么办，agent 可能无限重试刷屏，或者静默退出，任务结果无人感知。

这些多数不是模型能力不足，而是 instruction 没有按无人值守标准补齐关键信息。

## 做法 / 步骤

### 1. 先写触发与运行假设

在 instruction 顶部明确时区、触发频率、运行环境。例如：

```text
Runs daily at 08:00 Asia/Shanghai.
Assume no interactive user is present.
```

这能避免 agent 把“今天”“明天”这类相对时间当成真实计时依据。

### 2. 使用四段式结构

推荐把 cron instruction 拆成四段：Goal / Input / Constraints / Output。

示例：

```text
Goal: 抓取指定 RSS，生成中文摘要。
Input: 从环境变量 RSS_FEED_URL 读取源；如需 HTML 提取，使用已授权 MCP fetch 工具。
Constraints: 只读，不写本地文件；网络失败最多重试 2 次，间隔 30 秒；不做事实外推；单次运行超时 60 秒停止。
Output: 以 JSON 输出 {title, summary, link, pubDate}，写入标准输出；同时追加到 logs/cron-digest.jsonl。
```

这种结构让 agent 知道“做什么、用什么做、不能做什么、做完给什么”。

### 3. 加一个明确的停止规则

停止规则比成功路径更重要。比如：

```text
If RSS_FEED_URL is empty or returns 401/403, do not attempt fallback sources. Exit with SKIP.
```

这能防止 agent 在异常情况下自由发挥，比如自行去搜索引擎找替代源。

### 4. 上线前手动模拟

用一次手动触发跑同一条 instruction，观察它是否做了多余动作。重点检查三件事：工具调用是否符合 allowlist、输出格式是否稳定、失败分支是否正确退出。

## 踩坑点

- **相对时间**：写“明天早上”或“今天”会因时区、执行延迟产生偏差。所有时间用 ISO 8601 或明确时区。
- **无重试上限**：只写“失败重试”没有最大次数，可能死循环或产生大量请求。写“重试最多 N 次，之后标记失败并退出”。
- **输出格式未约束**：如果后续有插件/管道消费，要求固定 JSON schema，避免 agent 在关键结果前加“好的，我已经完成任务”这类自然语言。
- **把环境依赖写进 agent 记忆**：只在记忆里存 token、路径或频道 ID，换环境或清记忆就失效。最好用环境变量或插件配置注入。
- **权限太宽**：在 instruction 中显式列出允许的工具/动作，其他默认拒绝。例如：

```text
Only allowed tools: fetch, log.write. Do not send messages unless output_channel is set.
```

## 可复用建议

1. 维护一个 cron 任务卡模板，至少包含：ID、时区、触发时间、输入、允许工具、输出格式、失败策略、日志位置。
2. 每个任务第一次上线前做 dry-run，不接真实外部发送，只输出到日志检查。
3. 对简单任务不要强行上 agent。如果只是定时 HTTP 请求 + 模板渲染，用脚本更稳定；agent 适合需要摘要、判断、多步工具编排的任务。
4. 保留失败样本，每周看一两条，反向改 instruction。比如发现 agent 对空响应编造内容，就加一条：

```text
Empty input -> output {items: []} and stop.
```

5. 将 cron instruction 当代码管理，改动用 diff，环境变量与 prompt 分离。

## 总结

Cron 任务的 instruction 不是更详细的聊天指令，而是无人值守 runbook。把目标、输入、边界、输出、失败路径写清楚，比堆更多工具或换模型更有效。一次能稳定运行一周的 cron，通常不是 prompt 写得聪明，而是写得没有歧义。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/81cbdc5433a55074.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/0ca9552a60c55525.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/1d75a2f5cbe7adc0.png)

