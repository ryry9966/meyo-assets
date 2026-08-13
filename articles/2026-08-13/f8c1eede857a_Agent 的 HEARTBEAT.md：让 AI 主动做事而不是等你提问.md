---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 32965
source: 综合讨论
publishedAt: 2026-08-13
---

## 背景

很多 OpenClaw/Agent 实践仍然停留在“人在回路”：打开对话框、发指令、等回复。想让 Agent 值班，常见做法是在 cron 里塞一句“帮我检查服务状态”，但很快会发现不可控：每次执行没有共同记忆，Agent 不知道上次做了什么，输出散在通知里，状态没有落盘。

我曾经把一堆检查逻辑硬编码在 shell 里，让 Agent 只做总结。后来发现真正需要的不是更聪明的 prompt，而是一个稳定的“心跳文件”：让 Agent 每次被唤醒时，先读一份状态文件，再决定做什么。

## 问题

没有固定入口的定时唤醒，通常会出现：

- 任务列表写在 crontab 注释里，难以维护；
- Agent 每次从零开始，重复检查或漏掉任务；
- 执行状态不持久，失败后不知道上次死在哪；
- Agent 自由发挥，把环境改乱；
- 日志和通知混在一起，无法追踪。

## 做法：定义 HEARTBEAT.md

我把项目级的 `HEARTBEAT.md` 当作 Agent 的唯一入口。大致长这样：

```markdown
# HEARTBEAT.md
last_run: 2025-01-01T00:00:00Z
owner: ops-agent
state: idle
notify: critical

## Tasks
- [ ] disk_usage: every 1h, threshold 80%, action df -h, notify warn
- [ ] cert_expiry: every 24h, threshold 15d, action check_ssl, notify critical
- [ ] git_dirty: every 6h, action git status --short, notify info
```

配合调度器（cron / systemd timer / GitHub Actions）定时调起 OpenClaw 的 headless 入口，让它执行固定流程：

1. 读取 `HEARTBEAT.md` 和上次运行日志；
2. 按 `every` 和当前时间判断到期任务；
3. 对到期任务执行只读检查；
4. 根据阈值决定是否升级通知；
5. 把结果追加到 `heartbeat.jsonl`，更新 `last_run`；
6. 仅当有 critical/warn 时才发通知。

我用 MCP 包了两个工具：`read_heartbeat` 和 `update_heartbeat`。Agent 只能通过这两个工具访问心跳状态，不能直接改文件内容。这样把“状态机”和“任务执行”分开。

## 踩坑点

1. **并发重复执行**：cron 重叠或容器重启导致两个 Agent 同时跑。必须在心跳文件里加 `state: running`，并写入 pid/lock 文件；超过超时时间才允许抢锁。

2. **上下文膨胀**：如果让 Agent 每次读全部历史日志，很快会耗尽上下文。日志只追加 JSONL，心跳文件只保存指针、状态、下一次检查时间。

3. **无限自我触发**：Agent 很容易把“我该做 X”写回任务列表，形成自激。约定 Agent 只能更新 `last_run`、`state`，不允许新增/修改 `## Tasks` 区块；任务变更必须人工提交。

4. **误报疲劳**：阈值太紧，Agent 每小时发一条“磁盘 81%”。建议通知分三级，只有 critical 或连续两次 warn 才推送。

5. **权限失控**：直接给 Agent shell 权限会出事。把可执行动作限制为白名单命令或 MCP 工具，默认 dry-run，需要变更时必须生成一个待审批 diff。

6. **状态不收敛**：模型偶尔会输出不同格式。要求在 prompt 里固定 JSON 输出，例如 `{"task":"disk_usage","status":"ok","detail":"68%"}`，并校验 JSON 合法性再写日志。

## 可复用建议

- 把 `HEARTBEAT.md` 和配套的 `heartbeat.sh` / MCP server 放在一个模板仓库，新项目直接复制。
- 任务分为“检查类”和“动作类”。先只开放检查类，动作类必须经过 dry-run + 人工确认。
- 日志用 JSONL：`{"ts":"...","task":"...","status":"...","detail":"..."}`，便于 `jq` 和告警统计。
- 给 Agent 加熔断：同一任务连续失败 3 次，自动把该任务标记为 `paused`，等待人工介入。
- 心跳文件保持精简，不要塞长 prompt 或历史日志；长上下文用外部工具按需检索。

## 总结

`HEARTBEAT.md` 不是一个神奇提示词，而是一套协议：固定入口、有限状态、可追踪日志、最小权限。落地后，Agent 从“等你提问”变成“按节律自检、异常才找你”。在 OpenClaw 生态里，把心跳文件和 MCP 工具、调度器结合起来，成本不高，但能显著减少“我忘了看”的那类故障。

---

