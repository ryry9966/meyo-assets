---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 32217
source: 综合讨论
publishedAt: 2026-08-09
---

# Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问

## 背景

Chat-based agent 有种天然的被动感——你不发消息，它就永远沉默。但很多工程场景需要 agent 主动执行例行任务：早晨抓取依赖库更新、定时检查日志异常、每晚生成变更摘要、每 30 分钟巡检一次 MCP 服务的可用性。把这一切都留给人类记忆和手动触发，既不现实也不符合“助手”的定位。

常见的解法是引入专门的调度系统（Airflow、Temporal）或用 cron 调用无状态的 agent 任务。但这类方案常带来“状态外挂”的问题：agent 不知道上次做了什么、上下文在重启后丢失、任务逻辑和调度规则割裂。有没有更轻量且与 agent 心智模型更契合的做法？

## 问题

我们想要一种**心跳机制**，让 agent 能周期性地“醒来”，自主查看待办、评估条件、执行任务并记录状态，而不是永远等待下一个 `@agent`。希望满足三点：

1. **自文档化**：人能一眼看懂 agent 在定期做什么、上次执行情况。
2. **agent-native**：复用 agent 的推理能力，而不是写死脚本逻辑。
3. **轻量**：不引入外部调度数据库，不依赖特定平台。

## 做法：HEARTBEAT.md

核心方案是在项目仓库中放一个 `HEARTBEAT.md` 文件，它既是 agent 的周期任务定义文件，又是执行状态的持久化层。任一可运行 agent CLI 的环境都能配合 cron 变成主动式 agent。

### 1. 定义 HEARTBEAT.md 结构

文件使用 YAML front matter 保存元信息，正文用 markdown 表格列出周期性任务。示例：

```markdown
---
last_heartbeat: 2025-03-11T10:15:00Z
agent: openclaw-v2
heartbeat_interval_minutes: 15
---
# Heartbeat Tasks

| id | task | every | last_run | status | result_summary |
|----|------|-------|----------|--------|----------------|
| hb01 | 检查 GitHub 依赖更新 | 4h | 2025-03-11T08:00Z | ok | no updates |
| hb02 | 扫描 error 日志 (last 1h) | 1h | 2025-03-11T10:00Z | ok | 0 errors |
| hb03 | 生成每日摘要 (仅20:00执行) | 24h@20:00 | 2025-03-10T20:00Z | ok | sent to Slack |
```

约定：
- `every`：支持 `Nh`、`Nm`、`24h@20:00`（每日指定时刻）等简单表达。
- `last_run`：ISO 时间，agent 负责更新。
- `status`：`ok` / `error` / `skipped`。
- `result_summary`：一句话，方便人类扫读。

### 2. agent 的“心跳循环”

编写一个**心跳脚本**，例如 `heartbeat.sh`：

```bash
#!/bin/bash
set -euo pipefail
HEARTBEAT_FILE="./HEARTBEAT.md"
openclaw run \
  --prompt "Read $HEARTBEAT_FILE, decide which tasks are due (compare every + last_run vs now), execute them using available tools, and update the file accordingly. Only touch due tasks. Keep result_summary concise." \
  --allowed-tools fs.read,fs.write,github.search,slack.send,log.search \
  --context-file $HEARTBEAT_FILE
```

关键设计：
- **context-file** 将 HEARTBEAT.md 作为前置上下文注入，agent 看到自己的“心跳清单”。
- **prompt** 明确要求只处理到期任务，避免重复执行。
- 工具权限仅限完成任务所需，防止写入无关文件。

用 cron 每 15 分钟调用此脚本，与 `heartbeat_interval_minutes` 匹配即可。

### 3. agent 的执行与更新

agent 推理过程：
1. 解析 front matter 和表格，计算当前 UTC 时间。
2. 比对每个任务的 `last_run` + `every`，选出到期任务。
3. 对每个任务，调用对应工具（如 `github.search` 检查新 release，`log.search` 查询错误日志）。
4. 根据工具返回结果，更新该行：`last_run` 改为当前时间，`status` 设为 `ok`/`error`，`result_summary` 写入简短摘要。
5. 如果有通知任务，顺便推送 Slack/邮件。
6. 更新 front matter 中的 `last_heartbeat`，用 `fs.write` 覆盖文件。

整个过程**由同一个 agent 上下文完成**，不需要额外状态数据库。

## 踩坑点

在实践中遇到了几个值得注意的问题：

- **并发写入冲突**：如果 cron 间隔小于任务执行时间，可能上一个 heartbeat 还没结束，下一个又触发。解法：脚本开头加文件锁（`flock`）或使用唯一 run-id 写入临时文件再原子替换。
- **agent 幻觉修改**：有时 agent 会“顺手”修改无关行或篡改 `every` 表达式。限制 prompt 措辞（“Only touch due tasks”）有一定效果，更稳的解法是用结构化中间表示（如 JSON）让 agent 先输出变更计划，再由脚本校验后应用——但这会增加复杂度。初期可接受人工偶尔校对。
- **上下文增长失控**：如果日志结果很长，agent 可能把大段内容塞进 `result_summary`，导致表格膨胀。建议 prompt 明确要求“不超过 80 字符”，或结果存入单独文件，表格只放链接。
- **时区与时间解析**：agent 默认可能按本地时间理解，front matter 最好强制使用 UTC，prompt 中标注“all times are UTC”。
- **工具不可用时的错误处理**：当外部 API 限流或 MCP 服务宕机，agent 可能卡死。可限定每个任务超时，agent 在超时后设为 `error` 并记录原因，避免阻塞整个心跳循环。

## 可复用建议

- **轻量起步**：先用两个任务跑通闭环（如“每小时写一个心跳时间戳”），再扩展。
- **把 HEARTBEAT.md 当作 agent 的外部海马体**：除了例行任务，也可以记录 agent 上次值得记住的上下文片段，让主动执行和记忆结合。
- **结合 MCP 工具**：如果你的 agent 通过 MCP 获取日志、监控数据，这个模式格外适用——MCP 连接器就是 agent 的感官，心跳驱动其定期采样。
- **监控心跳健康度**：可再建一个 watcher 任务，检查 `last_heartbeat` 是否超过 2 倍间隔未更新，若是则报警（agent 自己宕了）。
- **版本控制友好**：HEARTBEAT.md 放在 Git 里，状态变更就是一次清晰的审计记录。

## 总结

`HEARTBEAT.md` 模式不算颠覆性创新，但恰到好处地把**定时驱动**和**agent 自主推理**捏在了一起。它让 agent 从“应答机器”变成一支悄无声息的持久进程，用人类可读的 markdown 完成了调度定义、状态持久化和执行摘要三件事。对于 OpenClaw 相关的技术栈，这是一种低摩擦的主动化方案，值得一试。

---

