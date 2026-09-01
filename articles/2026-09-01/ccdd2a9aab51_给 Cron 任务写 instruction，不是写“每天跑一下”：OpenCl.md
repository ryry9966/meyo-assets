---
title: 给 Cron 任务写 instruction，不是写“每天跑一下”：OpenClaw 定时任务 Prompt 工程化实战
feedId: 35665
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

OpenClaw 的 cron 任务会定时拉起 Agent，用同一段 instruction 无监督执行。交互式对话里，指令模糊还能追问；定时任务没有追问机会，一次理解偏差就可能重复推送、写坏状态文件、污染频道。

所以 cron instruction 不能按普通提示词写。它更像一个无人值守的控制程序：输入、工具、状态、输出、失败策略都要显式定义。

## 问题

常见失败有三类：

1. 只写目标，不写边界，例如“每天检查更新并推送”。Agent 不知道从哪里读更新源、推送哪个 channel、怎么判断“更新过”。
2. 没有状态和幂等。cron 重试、补跑或任务重叠时，同一条内容会被重复推送。
3. 输出随意。成功了不返回结构化结果，失败时还可能继续编造“已推送”，排查时没有现场。

## 做法：把 instruction 写成 I/O 合约

不是让 instruction 更长，而是把执行过程固定下来。下面是一个监控发布并推送通知的示例：

```text
You are a scheduled release sentinel.
Trigger: cron 0 9 * * * (UTC).

# INPUTS
- State file: ./state/release_sentinel.json
- Read config from ./config/repos.yaml at each run.
- Only use these MCP tools:
  - mcp.list_releases(repo)
  - notify.push(channel, payload)
  - fs.read_json(path), fs.write_json(path, data)

# EXECUTION
1. Read state. If missing, create {seen: {}, last_run: null}.
2. For each repo in config.repos:
   a. Call list_releases once; retry twice on temporary error, wait 5s.
   b. Compare latest release id with state.seen[repo].
   c. If id exists, skip.
   d. If new id, build payload with repo, version, url.
   e. Call notify.push exactly once with dedup_key=repo+id.
   f. Only after notify.push returns success, update state.seen[repo]=id.
3. Write state with last_run=UTC now.

# OUTPUT CONTRACT
Return only JSON:
{"status":"ok|no_op|partial_error","processed":n,"notified":n,"next_cursor":...}

# CONSTRAINTS
- MUST NOT call notify.push when RUN_MODE=dry_run.
- MUST NOT fabricate release ids if list_releases returns empty or error.
- If a repo tool fails after retries, record error and continue other repos.
- Never ask for confirmation.
```

核心是“状态文件 + dedup_key + 成功后再写状态”。这不能做到 exactly once，但能把重复降到可恢复、可排查的程度。

## 踩坑点

1. **“没更新就安静”**：这样会导致 no_op 没有日志，排障时不知道任务到底跑没跑、为什么没动作。每次运行都应返回结构化状态，包括 no_op。
2. **状态写反**：先更新状态再推送，推送失败后下一轮会认为已处理，造成漏报。正确顺序应该是在推送成功后更新游标。
3. **工具失败后自由发挥**：如果不显式禁止，Agent 可能用内部知识补一个“结果”。要明确写：空结果合法，MUST NOT fabricate。
4. **时区不统一**：cron 触发时间和 instruction 里的“当前时间”可能不一致。固定 UTC，写清 trigger 时区，避免本地时差误判。
5. **软表达太多**：关键路径用 MUST / MUST NOT，比“尽量、建议、你可以”更不容易被 Agent 在长指令里忽略。

## 可复用模板

```text
# ROLE
Scheduled task: <一句话>

# TRIGGER / TIMEZONE
<cron> (UTC)

# INPUTS
- state: <path>
- config: <path>
- env: <vars>

# TOOLS
- <read tool>
- <write/send tool>

# EXECUTION STEPS
1. ...
2. ...

# OUTPUT
Return JSON...

# FAILURE
retry policy, continue/stop, error object

# CONSTRAINTS
- Idempotent
- Dry-run safe
- No fabrication
```

## 总结

Cron instruction 的本质，是把“希望每天自动发生的事”编译成 Agent 能无监督执行的确定性流程。难点不在 prompt 技巧，而在补齐状态、幂等、输出契约和失败策略。先加 dry-run 跑几轮，再上 cron，比事后从频道里删重复消息、恢复错乱状态要便宜得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/4c88ef553812f394.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/1938ada79cd436fe.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/1b64dcc69d476d1a.png)

