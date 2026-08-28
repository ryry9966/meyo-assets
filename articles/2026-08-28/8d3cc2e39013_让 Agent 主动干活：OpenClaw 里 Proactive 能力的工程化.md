---
title: 让 Agent 主动干活：OpenClaw 里 Proactive 能力的工程化落地
feedId: 35030
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

大部分 AI 助手仍然是“问一句、答一句”的被动模式。你可以让它查日志、写脚本、总结 issue，但前提是你得先开口。真正有价值的场景往往是反过来的：服务挂了、数据异常、新 issue 来了，Agent 应该自己发现、自己判断、自己处理，然后把结果告诉你。

这种“不等你开口就把事情办了”的能力，在工程上叫 proactive。它并不需要多强的模型能力，关键在触发源、决策循环、工具执行和权限控制。

## 问题

在 OpenClaw 这类 Agent 框架里，模型和工具调用已经很成熟了，但要让 Agent 主动起来，通常会遇到几个工程问题：

- 触发源怎么接？是定时轮询、webhook 还是文件系统事件？
- 怎么避免频繁打扰？每次检查都发消息会变成噪声。
- 怎么保证幂等？同一个事件被处理多次怎么办？
- 权限怎么控制？全自动执行写操作很容易翻车。

这些问题模型解决不了，只能在 Agent 周围搭一层工程外壳。

## 做法

以一个真实场景为例：监控 GitHub 仓库的新 issue，自动打标签并通知负责人。

### 1. 拆成“触发条件 + 执行策略 + 动作”

先不要把整个流程写成一个巨大 prompt。拆开：

- **触发条件**：每 5 分钟检查一次，或者通过 GitHub webhook 实时触发。
- **执行策略**：让 Agent 判断新 issue 是否属于已知类型、是否需要人工介入。
- **动作**：调用 GitHub MCP 打标签、写评论、发通知到 IM。

### 2. 用 MCP 暴露工具

把 GitHub、数据库、文件系统等能力封装成 MCP server。这样 Agent 不用关心 API 细节，只需要决定“调用哪个工具、传什么参数”。

例如在 OpenClaw 里接入 GitHub MCP 后，Agent 可以执行：

```
list_new_issues(since=last_run_time)
add_label(issue_id, "bug")
send_notification(channel, message)
```

这里的 `last_run_time` 需要从外部状态存储读取，而不是让模型记在上下文里。

### 3. 定时触发 + 决策循环

在 OpenClaw 中可以使用定时任务（或外部 cron）触发一个“检查型 Agent”。这个 Agent 的 system prompt 可以写成：

> 你是一个 issue 分诊助手。请先调用 list_new_issues 获取上次检查后新增的 issue。如果 issue 标题包含“崩溃”或“core dump”，打上 bug 标签并通知；如果包含“建议”，打上 enhancement 标签；其他情况只记录，不通知。如果发现 5 个以上新 issue，立即发出高优先级通知。

这样 Agent 在每次触发时只做一件事：读取新数据、判断、执行动作。上下文很短，token 消耗可控。

### 4. 事件驱动替代纯轮询

定时轮询适合低频、非紧急任务。但对时效性要求高的场景，建议用 webhook。在 OpenClaw 外部起一个轻量 HTTP 服务，接收 GitHub / 监控系统 / 自定义应用的事件，验证签名后调用 OpenClaw 的 Agent 接口。

这样做的好处是事件来了才触发，不用空跑。但需要自己处理 webhook 的鉴权、重试和去重。

### 5. 先通知后执行（human-in-the-loop）

对于写操作（合并 PR、删除文件、重启服务），建议先让 Agent 发一条消息：

> 检测到服务 A 内存使用率连续 5 分钟超过 90%，建议重启。是否执行？[确认] [忽略]

用户确认后，Agent 再调用工具执行。低风险操作（如打标签、添加评论）可以直接自动执行，但必须写审计日志。

## 踩坑点

**触发太频繁导致上下文膨胀**

如果每 1 分钟跑一次，Agent 的上下文会很快被历史消息塞满，变慢且费钱。解决方法是每次触发都使用独立、精简的上下文，不保留历史；需要记住的状态写进 SQLite 或 Redis。

**没有幂等控制**

同一个 webhook 可能重试多次，导致 Agent 重复处理。一定要在外部状态存储中记录已处理的事件 ID（如 issue ID、告警 ID），处理前先查重。

**权限过大**

别把管理员权限直接给 Agent。只授予最小需要的工具，并把写操作拆成独立 MCP 工具，方便做权限隔离。例如打标签用 `add_label`，而不是一个万能 `github_api`。

**安全风险**

外部 webhook 一定要做签名验证（GitHub 的 `X-Hub-Signature-256`），否则任何人都能伪造事件，让 Agent 执行恶意操作。

**通知噪声**

不要每次触发都发消息。让 Agent 只在“需要人类注意”时通知。可以在 prompt 里明确：“如果一切正常，不要发送任何通知，只记录日志。”

## 可复用建议

- 先做“建议模式”，跑一周看误报率，再逐步放开自动执行。
- 状态存储用 SQLite 就够，记录 `event_id, processed_at, result, next_action`，方便排查和幂等。
- 把 proactive 逻辑封装成插件或 MCP 工具，比如 `proactive_issue_triage`，在多个 Agent 里复用。
- 给 proactive Agent 加心跳和失败告警。如果 Agent 超过 N 分钟没有执行记录，说明触发链路断了。
- 审计日志必须包含：触发原因、Agent 的决策依据、执行了哪些工具、最终结果。否则出了问题没法追责。

## 总结

Proactive 能力不是模型升级，而是工程改造。用“定时/事件触发 + 精简上下文 + MCP 工具 + 状态存储 + 权限控制”这套组合，可以把一个被动的问答助手变成真正能帮你盯事情、干活的协作者。

但克制很重要。低风险任务先自动，高风险任务先建议，等验证稳定后再放开。Agent 主动干活的前提是，你给它划好了边界、装好了刹车。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/fcc7dab3bc105a73.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/b13b5c72d1eb3f87.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/1ac42b81b2c5df75.png)

