---
title: 让 Agent 主动干活：HEARTBEAT.md 心跳任务层实战
feedId: 34032
source: 综合讨论
publishedAt: 2026-08-21
---

很多 OpenClaw 配置把重点放在“如何回答得更准”和“如何执行一次性指令”，结果 Agent 越来越像客服：你问一句，它动一下。但在自动化场景里，真正有价值的是让 Agent 周期性巡检、主动发现异常、按条件触发动作。

我实践下来的做法是：在 Agent 工作区增加一个 `HEARTBEAT.md`，把“什么时候该做什么事”从 system prompt 里拆出来，变成可维护、可约束的心跳任务层。

## 背景

通常我们会在 `AGENTS.md` 里描述行为准则、工具偏好、目录结构。但 `AGENTS.md` 更适合静态规则，不适合表达“每 30 分钟检查一次服务健康”“当某个目录出现新文件时执行分诊脚本”这类时序行为。

如果把这些内容硬塞进 system prompt，会出现三个问题：

- 每次会话都执行，噪音大；
- 没有执行状态记录，任务可能重复跑；
- Agent 自己分不清“判断”和“动作”，容易做出危险写操作。

所以我把时序任务独立成 `HEARTBEAT.md`。

## HEARTBEAT.md 是什么

它本质上是一个任务清单 + 触发策略文件，放在 Agent 工作区根目录，和 `AGENTS.md` 平级。一个简化示例如下：

```markdown
---
heartbeat: 30m
notify: silent
max_runs: 3
---

# HEARTBEAT

## Trigger
- on-session: once
- interval: 30m
- file-watch: data/inbox/*.json

## Tasks
- [READ] 检查 git status 是否有未提交变更
- [READ] 请求 http://localhost:8080/health
- [CHECK] 若 data/inbox/ 有新文件，运行 ./scripts/triage.sh
- [ACT] 当健康检查连续失败 3 次时，推送告警
```

关键点：`HEARTBEAT.md` 只放触发条件、任务类型和动作目标，不放执行日志。

## 做法与步骤

**第一步：建立 HEARTBEAT.md**

把任务按标签分类：

- `[READ]` 只读检查；
- `[CHECK]` 条件判断；
- `[ACT]` 写操作或外部动作；
- `[NOTIFY]` 通知。

这样 Agent 在执行前可以先判断动作级别，减少误操作。

**第二步：在 AGENTS.md 中引用**

让 Agent 在启动或每次新会话开始时先读取 `HEARTBEAT.md`。可以在 `AGENTS.md` 中写：

> 每次会话开始或定时触发时，先读取 HEARTBEAT.md，按 Trigger 条件执行任务，并将结果写入 HEARTBEAT_LOG.md。

**第三步：接入调度机制**

OpenClaw 可以配合 MCP cron 工具或定时插件触发心跳。如果没有独立调度器，也可以用外部 cron 执行类似：

```bash
openclaw run --heartbeat
```

让外部系统按节奏拉起 Agent，而不是让 Agent 自己常驻等待。

**第四步：输出到独立日志**

执行结果写入 `HEARTBEAT_LOG.md`，并设置截断或轮转。例如只保留最近 50 条记录，避免日志无限膨胀。

**第五步：通知收敛**

默认使用 `notify: silent`，只有 `[ACT]` 和 `[NOTIFY]` 任务命中异常条件时才发送消息。避免每 30 分钟发一次“一切正常”。

## 踩坑点

**1. Agent 修改了 HEARTBEAT.md 本身**

如果 Agent 把执行状态回写到 `HEARTBEAT.md`，下一次心跳就可能把它当成“文件变更”继续触发。解决方法是：日志只写 `HEARTBEAT_LOG.md`，并且不要把 `HEARTBEAT_LOG.md` 纳入触发文件监听范围。

**2. 任务非幂等**

“发现新文件就移动”这类任务，在失败重试时可能重复执行。建议加状态目录或用文件 hash 去重。比如只处理 `data/inbox/` 中不在 `data/processed.txt` 里的文件。

**3. 上下文持续膨胀**

心跳日志如果每次都被读进 system prompt，上下文会越来越大。正确做法是：日志不进入主上下文，需要分析时再让 Agent 读取日志尾部。

**4. 自动执行破坏性操作**

自动跑 `rm -rf` 或 `git push` 风险很高。给 Agent 的 MCP 工具加白名单，心跳任务中默认只允许 `[READ]` 和 `[CHECK]`；`[ACT]` 需要明确条件或二次确认。

**5. 定时和文件触发并发**

同一个任务可能被定时器和文件监听同时触发。可以在 HEARTBEAT.md 里加 `lock: single` 约束，或用外部调度器保证单实例运行。

## 可复用建议

- 小步开始：先只做 `[READ]` 巡检，稳定运行一周后再加 `[ACT]`。
- 给心跳任务单独使用 profile 或 namespace，避免和交互式会话的工具权限混用。
- `HEARTBEAT.md` 纳入版本控制，`HEARTBEAT_LOG.md` 不提交或定期清理。
- 每次心跳只执行最小必要动作，把判断逻辑放在脚本里，不要让 Agent 在心跳期间做复杂推理。
- 将 Agent 的主动行为限制在“检查、判断、通知”三层，真正有破坏性的操作交给人工确认。

## 总结

`HEARTBEAT.md` 不是魔法，它只是把 Agent 的“何时行动”从零散提示词中抽出来，形成一层可读、可审查的规则文件。它让 Agent 从被动问答转向主动巡检，但前提是控制好副作用和可观测性。

如果你的 Agent 现在只是问一句动一下，可以试试加一个只读版心跳任务：每天检查一次服务健康、仓库状态、未处理文件。稳定之后再逐步开放写操作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/745e276765e193b2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/e56d5e0087cf5d4a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/5fbdbda891fe156e.png)

