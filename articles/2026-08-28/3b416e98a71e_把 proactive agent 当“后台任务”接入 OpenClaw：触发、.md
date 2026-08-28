---
title: 把 proactive agent 当“后台任务”接入 OpenClaw：触发、授权、执行三段式
feedId: 35068
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw 上做自动化，很多同学一开始都是被动式：你发消息，Agent 回你，或者调用 MCP 工具完成一件事。但真正有用的场景往往是主动式：定时汇总未读消息、监控页面变更后生成简报、webhook 收到 CI 失败后主动通知、MCP 资源出现异常时做一次诊断。

proactive 能力看起来很自然，但接进去之后才发现：Agent 不等你开口就把事情办了，同时也可能不等你确认就把事情办砸了。

## 问题

直接让大模型在后台“自己看情况办事”，通常会遇到四类问题：

1. 误触发：webhook 重试、cron 重复、MCP 事件风暴，导致同一件事被处理多次。
2. 越权：模型为了完成任务，调用未被允许的工具，或者把草稿直接发到生产群。
3. 不可观测：你不知道它为什么动了、动过什么、哪些结果被写回记忆。
4. 上下文污染：每次 proactive 结果都塞进长期记忆，时间一长主对话上下文全是过期的主动任务摘要。

所以 proactive 不是给 Agent 加一个“自动模式”开关，而是要把触发、决策、执行拆成可审计的管道。

## 做法/步骤

我在 OpenClaw 里用了一段比较克制的接入方式，步骤如下：

### 1. 先收敛触发源

只保留三类触发：定时 cron、外部 webhook、MCP server 事件。不要一上来就允许 Agent 自己决定什么时候跑。每个触发源都带 metadata：`trigger_type`、`trigger_id`、`timestamp`、`raw_payload_hash`。

### 2. 所有触发先进 pending 队列

增加一个 gate 层，先做去重和静默判断。例如：

- 同一 `trigger_id + raw_payload_hash` 在 5 分钟内只处理一次。
- 用户在静默窗口内不执行非紧急动作。
- 如果当前已有同类型任务在执行，则合并或跳过。

### 3. 让 Agent 只做 plan，不直接执行副作用

把执行层放在确定性脚本或受限 MCP 工具后面。Agent 输出一个结构化的 action plan，例如运行哪个 skill/plugin、参数是什么、目标资源是什么。真正执行前，根据风险等级选择 `auto`、`dry-run`、人工确认。危险动作（发消息、写文件、提交代码、删除资源）默认人工确认或 dry-run。

### 4. 用 action id 做全程追踪

每次执行生成 `action_id`，记录触发原因、决策结果、执行状态、回滚方式。这样即使半夜自动跑了一个任务，第二天你也能看出它为什么这么做。

### 5. proactive 结果只回写摘要，不塞全文

执行完成后，只把 `action_id`、结果摘要、失败原因写回 memory/state，避免上下文膨胀。原始日志放到外部文件或日志系统。

## 踩坑点

- **webhook 去重没做签名校验**：外部重试会导致重复执行。一定要校验签名或 token，再用 `payload_hash` 去重。
- **把 LLM 放在事件循环里**：一个事件触发一次完整推理，事件一多 token 成本和延迟都不可控。可以先用规则/脚本做初筛，再决定是否进入 LLM。
- **自动模式开得太大**：第一次接入就让 Agent 自动发消息、自动改配置，基本都会出事。先只读，再写草稿，最后才开自动执行。
- **cron 时区不明确**：容器里 UTC 和本地时间混用，导致“早上 9 点”的任务在半夜跑。统一用 IANA 时区，触发时间带 timezone。
- **被长期记忆里的 proactive 结果带偏**：历史摘要太多会让主对话上下文越来越长，甚至影响后续任务判断。给 proactive 记忆单独分区，并设置过期时间。

## 可复用建议

- 把流程固化为：`trigger → gate → plan → approve/auto → execute → observe → dedupe/fingerprint`。
- 用 OpenClaw 的 skill/plugin 封装一组 proactive tasks，例如 `proactive:summarize_inbox`、`proactive:ci_alert`、`proactive:monitor_mcp`，配置和逻辑分离。
- 每个触发源都支持 `dry_run` 和 `enabled` 开关，首次接入强制 dry-run 若干次或人工 approve 一次。
- 给执行结果建一个简单的观测页或日志查询，至少能看到 `action_id`、`trigger_type`、时间、状态、耗时、是否回滚。
- 从低风险开始：只读、通知到草稿箱、写临时文件，再逐步放开到生产动作。

## 总结

proactive 能力最吸引人的地方，是它让 Agent 从“问一句动一下”变成“在后台把事情推进”。但工程上真正有用的 proactive，不是让模型更主动，而是让触发、授权、执行变成一条清晰、可暂停、可回滚的流水线。把 proactive 当后台任务而不是魔法功能，接进 OpenClaw 之后才会真正省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/8defc7e2a0fca4f7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/0ab93d9d76229255.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/1a672fbfd8647d90.png)

