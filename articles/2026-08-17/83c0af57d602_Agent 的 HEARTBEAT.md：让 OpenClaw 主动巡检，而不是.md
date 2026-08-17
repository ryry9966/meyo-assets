---
title: Agent 的 HEARTBEAT.md：让 OpenClaw 主动巡检，而不是等你提问
feedId: 33585
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

OpenClaw 这类 Agent 默认是响应式的：你给指令，它执行，然后回到等待状态。即使挂了一堆 MCP 工具，主动性仍然来自 prompt。很多自动化场景需要相反的形态——定时巡检、发现异常、主动报告，而不是每天手动问一句“今天有什么问题吗”。

如果只是把定时任务散落在 crontab、插件或脚本里，Agent 的上下文、状态和动作边界都会变得不可控。更工程化的做法，是在 agent 工作区放一个 `HEARTBEAT.md`，把“什么时候主动、主动做什么、做到哪一步”外置成可审计的协议。

## 问题

主动行为难做，通常卡在四件事：

1. **状态不连续**：每次唤醒，Agent 都要重新理解“上次做到哪了”。
2. **动作边界模糊**：主动执行一旦越界，可能产生真实副作用。
3. **时间判断不可靠**：让 LLM 自己算 `now` 和 `next_run`，容易出错。
4. **排障困难**：没有统一日志，很难知道它为什么在某个时刻做了某个动作。

## 做法

核心思路是：`HEARTBEAT.md` 不负责复杂调度，只回答三件事——**何时该动、检查什么、允许做什么**。

### 1. 定义心跳文件

在 agent 工作区创建 `HEARTBEAT.md`，写入最小结构：

```markdown
# HEARTBEAT
interval: 30m
state_file: .heartbeat/state.json
log_file: .heartbeat/run.log
dry_run: true

## Check
- 读取 state_file，确认距离上次运行是否超过 interval
- 运行 `git status --porcelain`，若有未提交变更则记录
- 用日志 MCP 查询最近 1h 的 ERROR 数量
- 检查通知插件是否有未读消息

## Actions
- report: 将异常汇总写入 .heartbeat/report.md
- auto: 仅在 dry_run=false 时执行白名单命令，如 `git pull --rebase`
```

### 2. 用外部 cron 触发最小 prompt

不要让 Agent 自己常驻等待。用 cron 或系统调度器，到点执行：

```bash
echo "Now: $(date -Iseconds). Read HEARTBEAT.md and act if due." | openclaw run
```

这样时间判断由外部环境给出，Agent 只负责读文件、比对状态、执行检查。

### 3. 状态外置

把 `last_run`、`last_result`、`retry_count` 写进 `state.json`。每次心跳先读状态，再决定是否真的执行。不要依赖对话上下文保存进度。

### 4. 检查项绑定 MCP 工具

每条 Check 写清楚用哪个工具、参数从哪来。例如：

```markdown
- 日志 MCP：query(level=ERROR, since=state.last_run)
- Git 工具：status(repo_path=./)
```

避免模型自由发挥，减少幻觉。

### 5. 动作分级

坚持三级动作：

- `report`：只写报告，不做修改。
- `suggest`：给出建议，等待确认。
- `auto`：自动执行，但必须单独列白名单，且默认 `dry_run=true`。

### 6. 写审计日志

每次心跳写 `run.log`，至少包含：触发时间、读到的状态、执行了哪些检查、工具调用结果、后续动作。回滚和排障都靠它。

## 踩坑点

- **频率过高吃 token**：30 分钟一次可能仍然太密。按实际价值设 interval，宁可稀疏，不要为跑而跑。
- **循环触发**：Agent 可能自己改了 `next_run` 或触发了会产生新检查的动作。白名单里不要放会修改心跳文件本身的命令。
- **权限失控**：`auto` 白名单一定要限定命令和目录，避免“自主执行”变成删库。
- **状态文件并发**：多个 Agent 或进程同时写 `state.json` 会损坏数据。加锁，或保证同一时间只有一个心跳实例。
- **失败重试风暴**：动作失败后要退避。`retry_count` 超过阈值就暂停，改由 report 汇总。
- **时间判断偏差**：不要让模型自己推算“现在是不是该跑了”。触发时由外部传入精确时间。

## 可复用建议

- **把心跳当兜底，不替代事件驱动**。真正的实时任务用 MCP/webhook 触发，心跳只做周期性巡检。
- **先跑 dry_run 一周**。积累足够日志，确认没有误判和越权，再开 `auto`。
- **用最小 prompt 触发**。触发词越短越明确，Agent 越不容易跑偏。
- **每个 Check 都对应到具体工具**。不要写“检查系统是否正常”这种模糊项。
- **保留审计链**。日志里要有输入摘要、工具调用、结果，方便回滚和复盘。

## 总结

`HEARTBEAT.md` 的本质，是把 Agent 的主动行为从 prompt 里外置成可审计的协议。它不能替代可靠的调度器，但能让 OpenClaw 在无人值守时更可预测、可调试。最划算的起点是：一个 `HEARTBEAT.md`、一个 `state.json`、一条 cron，先跑 dry-run 一周。

---

