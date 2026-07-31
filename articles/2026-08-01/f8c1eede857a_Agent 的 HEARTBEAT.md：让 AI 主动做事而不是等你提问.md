---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 31153
source: 综合讨论
publishedAt: 2026-08-01
---

# Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问

## 1. 背景：Agent 总是“等你开口”

在 OpenClaw、MCP 和各种插件体系里，常见的 Agent 工作模式是“触发-响应”：用户发一条消息或一个事件，Agent 收到后调用工具、推理、执行，然后返回结果。这种模式在对话、按需查询等场景下已经够用，但一旦你希望 Agent 能**持续监控状态、定时巡检、主动推送提醒**，就立刻暴露出一个缺口——Agent 自己不会“心跳”。

你没法让它每天早上九点检查 CI 状态并主动发个摘要，也没法让它每 30 分钟扫一遍日志文件看是否有异常，除非你在外部再搭一套 cron 脚本去触发它。这相当于把自动化的“发动机”又拆回到手工启动，违背了 Agent 自主性的初衷。

## 2. 问题：需要一个轻量自驱心跳机制

理想状态下，Agent 应该像一个值班工程师，即使没人提问，也能定期巡视自己负责的环境。要实现这点，需要一个在 Agent 进程内部就能运行、且不依赖重度外部调度系统的“心跳”机制。不能用重量级工作流引擎，否则额外运维成本会吃掉收益。

一个实用且极简的方案是：在 Agent 的工作目录里放一个 `HEARTBEAT.md` 文件，作为心跳任务清单。Agent 搭载一个基于 MCP 的工具，定期读取该文件，解析其中的任务定义，判断当前时间该做什么，然后主动执行。

## 3. 做法：三步搭起自驱心跳

### 3.1 定义 HEARTBEAT.md 格式

用 YAML front matter 来定义周期性任务，Markdown 正文只做人类阅读的记录。例如：

```markdown
---
tasks:
  - id: morning-report
    schedule: "0 9 * * 1-5"
    action: summarize_ci
    params:
      repo: main-backend
      branch: develop
    ttl_minutes: 10
  - id: log-watch
    schedule: "*/30 * * * *"
    action: scan_logs
    params:
      path: /var/log/app/error.log
      pattern: "FATAL|CRITICAL"
      lookback_lines: 200
    notify_on_match: true
---
# Heartbeat tasks for primary Agent
```

每个任务包含：
- `id`：唯一标识
- `schedule`：cron 表达式，定义执行周期
- `action`：对应 Agent 内部某个工具名称
- `params`：传入工具的参数
- `ttl_minutes`：任务执行超时（可选）
- 其他控制字段（如 `notify_on_match`、`retry` 等）

### 3.2 实现 HEARTBEAT MCP 工具

为 Agent 增加一个 MCP server，暴露一个 `run_heartbeat` 工具，逻辑大致为：

1. 读取 `HEARTBEAT.md` 并解析 YAML front matter。
2. 遍历 tasks，对每个 task 计算其 `schedule` 的下次执行时间，并与当前时间比较；如果在允许的漂移窗口内（例如 ±1 分钟），则认为该任务需要执行。
3. 检查任务在 `HEARTBEAT.md` 中记录的 `last_run` 和 `status`，确保不会重复执行（可通过追加 YAML 字段 `last_run: "2025-03-15T09:00:02Z"` 实现）。
4. 调用对应 `action` 工具，传入 `params`，设置超时。
5. 任务结束后，将执行结果写回 `HEARTBEAT.md`（追加 `last_run`、`last_status`、`last_output_summary` 等），并可选地发送通知。

这里最关键的是**执行状态的持久化直接写在 HEARTBEAT.md 自身**，不引入外部数据库。文件本身既是指令源，也是状态记录本。

### 3.3 让 Agent 定期触发心跳

利用 OpenClaw 的定时器或内部循环，每 60 秒调用一次 `run_heartbeat` 工具。如果 Agent 框架不支持自主定时，可以简单地让 Agent 在处理完任何外部请求后，在下一次空闲时主动调用该工具，或者写一个辅助进程每分钟向 Agent 发送一次 `heartbeat` 伪请求。

## 4. 踩坑点

**文件竞争与并发**。如果多个 Agent 进程或同一进程的多次心跳执行并发读取、写入同一个 `HEARTBEAT.md`，可能造成状态覆盖或任务重复执行。解决方式：使用文件锁（如 `flock`），或在 MCP 工具内部对写操作做排他控制；另一种更轻量的方案是，只在任务执行前后直接通过 `fs.rename` 原子性地替换文件内容，但实现稍复杂。

**cron 处理与时间漂移**。不要自己去实现 cron 解析器，直接用成熟库（如 `cron-parser`）。注意系统时区统一为 UTC，避免因服务器时区变更引起混乱。心跳间隔建议设为调度精度的 1/3~1/2，例如任务最小粒度为 30 分钟，心跳间隔取 5~10 分钟就够。

**任务超时和僵尸任务**。如果一个任务执行时间过长，会阻塞整个心跳循环。必须给每次 `action` 调用设置超时时间，并在超时后记录失败、继续下一个任务。此外，需要定义任务执行失败后的重试策略（可简单约定：忽略，等下一周期再试）。

**HEARTBEAT.md 膨胀**。`last_run` 和 `last_output_summary` 如果每次追加不清理，文件会越来越大。可以只保留最近 N 次执行记录，或按任务只保留最新一条，减少解析开销。

## 5. 可复用建议

- **任务模板规范**：统一 YAML 结构，可以在团队内复用。将 `action` 与 Agent 的工具注册表对齐，保证每个 `action` 都有对应的实现。
- **状态可见性**：`HEARTBEAT.md` 的 Markdown 区域可以自动生成上次执行结果的概要表格，方便人一眼看懂心跳状态。
- **与通知体系结合**：任务定义中增加 `notify_channel`，匹配公司内部的 IM、邮件等通道，让主动执行的结果能被及时消费。
- **逐步放开权限**：初期只让心跳执行只读或低风险操作，验证稳定后再加入自动修复、重启服务等变更类任务。
- **监控心跳自身**：如果 `HEARTBEAT.md` 超过预期时间未被更新，可以视为 Agent 心跳丢失，由外部守护程序发出告警。

## 6. 总结

`HEARTBEAT.md` 是一个极低成本的自驱机制，它利用文件系统作为“内存”，让 Agent 从被动响应者变成主动巡视者。不引入额外存储和调度系统，用几行 YAML 定义周期任务，配合 MCP 工具和定时触发，就能实现值班报告、异常扫描、主动通知等实用自动化。工程实践中注意文件锁、超时和状态清理，就能稳定运行。如果你的 Agent 现在只会等你说“帮我……”，不妨给它一个 `HEARTBEAT.md`，让它学会自己“心跳”。

---

