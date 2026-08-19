---
title: 给 Agent 装上心跳：用 HEARTBEAT.md 从“问答机器人”变成“值班工程师”
feedId: 33839
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景

很多人已经把 Agent 接进了日常开发、运维和自动化流程里。工具链不差：MCP 能读文件、发通知、查监控、调用接口；OpenClaw 也能在会话里完成多步操作。但大多数 Agent 仍然停留在“你问一句，它动一下”的状态。

真正有价值的 Agent，不应该是问答机器人，而应该像一个值班工程师：按固定节奏巡检系统，发现异常时主动报告，必要时给出处理建议，而不是等你打开聊天窗口。

## 问题在哪

想让 Agent 主动做事，常见的做法有两种：

1. **开一个长期会话**，靠 prompt 让它每隔一段时间检查一次。问题是上下文会不断膨胀，时间一长模型就开始丢信息、重复执行或忘记约束。
2. **用 cron 直接触发脚本**。但脚本只能处理确定逻辑，缺少 Agent 的判断能力；而临时在 cron 里塞一段长 prompt，又很难维护和调试。

缺少的是一层轻量的“持久化任务清单”：让每次被定时唤醒的 Agent，都能快速恢复上下文，知道自己该做什么、做到什么程度、上次做到哪里。

`HEARTBEAT.md` 就是干这件事的。

## 做法：把值班表写进文件

核心思路是：在项目或工作区根目录放一个 `HEARTBEAT.md`，作为 Agent 每次被定时唤醒后的“心跳入口”。

### 1. 定义一个最小 HEARTBEAT.md

不用很复杂，先保证可执行：

```markdown
# HEARTBEAT.md

interval: 30m
cooldown: 10m
mode: check          # check | suggest | act
max_steps: 5

tasks:
  - id: disk-check
    every: run
    action: 检查 /data 磁盘使用率
    notify_if: "> 80%"
    evidence: true

  - id: stale-pr-review
    every: workday 09:30
    action: 查看未合并 PR，超过 24h 未回复的评论提醒 owner
    notify_if: "found"
```

这里 `mode` 很重要。一开始只用 `check`，只读检查、只报告；稳定后再放开到 `suggest` 或 `act`。

### 2. 用外部触发器定时唤醒 Agent

不要依赖模型自己在会话里“等半小时再执行”，这不稳定。用系统 cron、MCP scheduler 或你自己的调度脚本，每 30 分钟触发一次 Agent 会话。

例如系统 cron 可以这样：

```bash
*/30 * * * * ~/home-ops/run_heartbeat.sh
```

`run_heartbeat.sh` 里调用你的 Agent 入口，传入固定指令：

```text
Read HEARTBEAT.md in the workspace.
Execute tasks that match the current time and cooldown rules.
Do not add new tasks.
Write results to heartbeat.log and state.json.
Stop after max_steps.
```

这样每次心跳会话都是全新的、上下文干净的，但任务来源和约束来自文件，不会丢。

### 3. 固定执行流程

在 HEARTBEAT.md 里明确告诉 Agent 执行顺序：

1. 读取当前时间和 `HEARTBEAT.md`
2. 检查冷却时间，跳过未到期的任务
3. 按任务定义执行，优先只读检查
4. 记录 `evidence`，例如实际磁盘使用率、PR 链接
5. 更新 `state.json` 和 `last_run`
6. 达到 `max_steps` 就停止

这样每次运行行为可预测，也方便排查。

### 4. 日志与状态分离

不要把完整历史都堆在 HEARTBEAT.md 里。任务定义保持稳定，运行状态放 `state.json`，日志放 `heartbeat.log`。日志按条追加，定期轮转，只保留最近 N 条。

## 踩坑点

**死循环与自我新增任务**。Agent 有时会“顺手”把新任务写回 HEARTBEAT.md，下次又触发更多动作。必须约束：只允许写日志和状态区域，不允许修改任务清单本身。

**上下文膨胀**。如果每次心跳都读取完整日志，很快会撑爆上下文。解决方法是日志轮转，或者在 prompt 里只让 Agent 读取最近 10 条日志。

**定时不准确**。不要给模型一个模糊的“每半小时检查一次”，它可能不理解或忘记。最好由外部 cron 触发，并在 prompt 中给出明确的当前时间。

**通知骚扰**。没有冷却时间，Agent 会在同一个问题上反复提醒。任务级别和全局都要设置 cooldown。

**权限过大**。一开始就允许 Agent 执行删除、重启、发公告等操作，很容易翻车。先做只读检查，人工确认后再逐步放开。

## 可复用建议

- 从 2～3 个只读巡检任务开始，跑满一周再考虑自动修复。
- `HEARTBEAT.md` 纳入版本控制，`heartbeat.log` 和 `state.json` 可以 gitignore。
- 每个任务都要求输出 `evidence`，没证据的“已检查”不要信。
- 工具权限尽量收窄：只读文件、查监控、发 webhook 就够了，不要一上来暴露 shell。
- 采用“检查—建议—执行”三级权限，而不是直接全自动。

## 总结

`HEARTBEAT.md` 不是神奇提示词，而是一份给 Agent 的“值班表”。它解决的核心问题是：让定时唤醒的 Agent 有稳定上下文、明确任务和可回滚记录，从而从被动问答变成按节奏主动工作。

工程化落地时，控制权限、设置冷却、记录日志，比写出一个花哨的 prompt 更重要。先让它学会巡检查异常，再慢慢把动作放权给它。

---

