---
title: 让 Agent 从“问答机”变成“值班员”：OpenClaw 下 proactive 能力的最小落地
feedId: 34813
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

很多 OpenClaw 实践里，Agent 已经能通过 MCP 查数据、调工具、写文件，但触发方式仍然是“用户说一句，Agent 动一下”。这种被动模式在演示里够用，在真实工作流里不够：故障发生、指标异常、上游数据更新、定时巡检这些场景，没人会一直守在对话框前发指令。

于是需要 proactive 能力：根据时间、事件、状态变化主动触发 Agent 行动。听起来很自然，但落地时它更像一个工程问题，而不是模型能力问题。

## 问题

proactive 最大的难点不是“能不能自动跑”，而是“怎么跑得不添乱”。一旦 Agent 可以自己醒来做事，权限、频率、上下文、可观测性、误触发都会变成实实在在的风险。很容易做成一个半夜疯狂调用 API、或者把测试环境当生产环境操作的“自动捣乱器”。

对 OpenClaw/MCP 用户来说，核心不是让 Agent 更聪明，而是给它一套受控的自主机制。

## 做法 / 步骤

**1. 先建立 signal 层，而不是直接写定时任务**

把 proactive 触发源统一成 signal，例如：

```text
{type: "schedule" | "webhook" | "threshold" | "file_change", source, payload, ts}
```

定时任务、GitHub webhook、Prometheus 告警、文件系统监听都转换成这个结构。统一结构后，后续策略、审计、去重都更容易做。

**2. 策略 gate 前置**

收到 signal 后先不执行工具，先过 policy：是否在静默窗口？同一 signal 是否在冷却期内？是否超过每日主动执行上限？是否需要人工确认？

可以用简单规则引擎或 OpenClaw 的 policy 插件维护。不要让 Agent 自己“看着办”。

**3. 执行链路走 MCP，且默认只读**

把工具分成 `read_only`、`write`、`dangerous` 三类。proactive 任务默认只允许 `read_only` 工具；写操作必须显式声明 `requires_approval` 或先 dry-run。

MCP server 端也可以做权限裁剪，不要只靠 prompt 里的“请谨慎”来约束。

**4. 回执和通知合并**

proactive 执行完成后不要每条都推。把结果写入本地 SQLite 或日志，按小时或按天生成摘要，只对高优先级异常即时通知。否则用户很快会把通知关掉。

**5. 记录 decision log**

每次触发、是否执行、执行结果、被拒绝原因都落库。没有 decision log 的 proactive 系统出了问题几乎无法排查。

一个最小示例：用 cron 每 30 分钟跑一次巡检，检查 Redis 内存使用率。signal 进来后，policy 判断：夜间且只是超过 70% 未达 85%，只记录不通知；超过 85%，调用只读工具拉取最近 10 分钟慢查询和客户端连接数，生成摘要推送。如果需要清理过期 key，则进入 approval 队列，用户回复“执行”后才调用写工具。

## 踩坑点

- **时区与重复执行**：cron 时间戳要统一 UTC 存储，展示时转换。加 idempotency key，例如周期标识 + signal hash，防止同一次事件被处理两次。
- **上下文过期**：不能把上次对话的 context 缓存直接用于主动任务。每次 proactive 执行前重新获取当前状态，否则容易用几小时前的错误信息做判断。
- **循环触发**：Agent 调用写操作后产生新事件，又被自己的 webhook 监听捕获，形成死循环。可以在 signal 里带 actor 信息，policy 对 `actor=agent` 的事件默认只记录不行动，或加最小触发间隔。
- **权限失控**：最容易出问题的是把“能自动执行”等同于“完全自动”。建议先只读、后白名单、再灰度开放写操作。不要让 proactive Agent 直接持有删除、发布、转账类工具。

## 可复用建议

- 从“巡检摘要”开始，不要一上来做自动修复。让 Agent 每天定时跑只读检查，整理异常发你，跑一周看看误报率。
- 将 proactive 能力封装成状态机：`idle -> evaluating -> approved -> executing -> done/failed`，每个状态可观测。
- 给 MCP 工具标注 approval 策略，而不是把确认逻辑写在 prompt 里。
- 为所有主动操作保留 dry-run 输出，先看“如果执行会怎样”，再决定是否放开。
- 设置全局熔断：主动任务连续失败 N 次，或单日调用次数超过阈值，自动停用 proactive 并通知。

## 总结

proactive 能力的关键不是让 Agent 更“聪明”，而是让它具备受控的自主性。它更像一个值班员：能自己看表、自己巡检、自己报告，但遇到危险操作知道停下来等人确认。

对 OpenClaw/MCP 实践来说，先把 signal、policy、approval、audit 这四件事做扎实，再谈自动执行，会少踩很多坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/6d2288bd46c58201.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/3d5b8e060dc4c8bd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/fafa98a6431209b5.png)

