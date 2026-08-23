---
title: 把“主动”关进笼子：在 OpenClaw 里实现可回滚的 proactive agent
feedId: 34347
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

现在大多数 AI 助手还是“你问一句，它动一下”。但在巡检、告警处理、证书过期检查、依赖升级、CI 失败排查这类场景里，真正有价值的是：事情发生之后，助手能自己先看一眼，而不是等你把日志喂给它。

但“主动”很容易做歪。给 Agent 挂一个 cron，再给几个写工具，最后常常变成重复告警、误改配置、半夜自动合并分支，还没法追溯是谁让它干的。

这篇帖子聊聊在 OpenClaw + MCP + 自动化插件体系下，怎么把 proactive 能力做成可控制、可回滚、可审计的工程管道。

## 问题

Proactive 的核心难点不是“能不能触发”，而是：

- 什么时候该主动，什么时候该闭嘴？
- 哪些动作可以静默执行，哪些必须人工确认？
- 重复事件如何去重？
- 模型误判上下文时，怎么避免不可逆操作？
- 出了事能不能回滚、能不能查到完整轨迹？

如果这些问题不解决，主动能力越多，运维负担越重。

## 做法：Detect → Decide → Act → Report

以一个具体场景为例：**CI 失败后，Agent 主动分析日志、定位可疑 commit，并创建 issue 或修复分支。**

### 1. 事件源与只读侦察

先不要给 Agent 写权限。用 MCP server 暴露一组只读工具：

- `get_failed_jobs`
- `get_log_tail`
- `get_recent_commits`
- `get_file_blame`

Agent 默认只挂这个 read-only MCP。这样即使模型误判，也只能看，不能动。

### 2. 触发与上下文注入

通过 webhook 或 scheduler 触发 OpenClaw Agent。不要在 prompt 里写“修复 CI”，而是给结构化上下文：

```text
You are an on-call agent. Recent CI failed.
First inspect the failure. Then produce a plan.
Do not modify code unless plan is approved.
Return JSON: {event_id, diagnosis, plan, risk, needs_approval}
```

这里的关键是：**默认动作是 produce a plan，而不是 fix。**

### 3. 分层动作策略

把动作按风险分三级：

- **低风险**：创建 issue、打标签、发通知。可静默执行。
- **中风险**：创建分支、生成补丁、修改文档。必须 dry-run 展示 diff，再执行。
- **高风险**：合并 PR、发布、改生产配置。必须人工 approval。

工具描述里要写清副作用。例如 `create_branch` 只创建 git ref，没有 merge 权限。这样模型在调用时更容易做出正确判断。

### 4. 去重、幂等与预算

每个事件生成一个 fingerprint，例如：

```text
event_type + repo + job_id
```

存入 Redis 并设置 TTL 24h。重复触发时直接跳过，避免 webhook 重试或 cron 重叠导致同一事件被处理多次。

同时设置动作预算：每个 Agent 每天最多执行 5 次主动写操作，超过后自动转人工。

### 5. 审计与回滚

所有动作写审计表：

```text
trigger_time, event_id, plan_json, tool_calls, result
```

对于创建 issue、创建分支等动作，同时记录可逆操作：关闭 issue、删除分支。这样即使做错了，也能快速撤销。

## 踩坑点

1. **读写权限混在一起**  
   一个 MCP server 同时暴露 read 和 write，模型很容易绕过限制。建议 read/write 分离，Agent 默认只读。

2. **重复触发**  
   webhook 重试、cron 重叠都会导致同一事件被处理多次。必须做 fingerprint + 分布式锁，不能靠 sleep 或运气。

3. **时区和时间窗口**  
   定时任务要明确 timezone，不要默认 UTC 然后半夜主动干活没人看。最好设置“允许主动动作的时间窗口”。

4. **Prompt 过度激励**  
   写“发现问题就解决”会让模型做出激进操作。改成“先诊断、再提案、默认 dry-run”，能明显降低误操作概率。

5. **工具副作用描述含混**  
   如果 MCP 工具描述里 `delete` 和 `list` 分不清，模型可能把删除当查询用。每个工具描述必须包含 `sideEffects` 和 `idempotency` 提示。

6. **缺少人工刹车**  
   高风险动作没有 approval 机制，出了事没有中断入口。可以加一个 `request_approval` 工具，或者接入外部审批队列。

## 可复用建议

- **默认 dry-run**：先输出 plan，再执行；写操作默认 no-op。
- **四段式管道**：Detect → Decide → Act → Report，每段都输出结构化结果，方便审计。
- **把“主动”定义为可中断、可回滚、可追踪**，而不是“更快做更多”。
- **灰度路线**：先做只读巡检和通知，稳定后再开放创建 issue，最后才开放分支和补丁。
- **在系统提示里加上一句**：“宁可多问一句，不要做不可逆操作。”

## 总结

Proactive 不等于激进。工程上可行的主动助手，是靠 MCP 分离读写权限、分层授权、事件去重、动作预算和审计回滚共同构成的。

真正好用的 proactive agent，不是“不等你开口就一顿操作”，而是：**你还没开口，它已经看完现场，拿着一份可审查的方案在等你点头。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/f3edd877fcb07d5c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/95a0323b281a837a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/c28ed7183d1275ea.png)

