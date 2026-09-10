---
title: OpenClaw 会话隔离实践：让子 Agent 干完活只带结论回家
feedId: 36891
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

OpenClaw 的主会话是你和 agent 的长期记忆载体：对话历史、工具调用结果、文件摘录，全堆在同一个 context 里。跑得越久越容易撞上下文上限，触发 compaction，早期关键决策被压缩掉。子 Agent（`sessions_spawn`）就是为解决这个问题的：把重活、脏活放进独立 session，跑完只回传一段结论。但实践中我发现，隔离并不是开箱即用的，配不好照样污染。

## 问题：污染从哪来

实际用下来，“污染”主要有三种形态：

1. **上下文污染**——子任务全程回传，几 KB 的日志直接灌进主会话，一两次就把 compaction 逼出来；
2. **状态污染**——多个子 Agent 并发写同一个 session 或同一个文件，结果互相覆盖；
3. **记忆错觉**——反过来，你以为子 Agent“记住”的上下文，主会话其实根本不知道。

## 做法

1. **重活下沉。** 凡是会产生大输出的任务（批量读文件、浏览器抓页、跑长脚本），交给 `sessions_spawn` 起子 Agent，不要在主会话里直接干。
2. **约束回传格式。** spawn 的 taskPrompt 末尾固定加一句：“只返回：结论、关键数据、产物文件路径，总长不超过 300 token。”主会话看到的就是一段紧凑摘要，而不是整个 transcript。
3. **用文件做慢速通道。** 子 Agent 把详细过程写进 workspace 下的固定目录（比如 `workspace/subagent-out/<task-id>/`），主会话只拿路径，需要时再按需读取。
4. **凭证不进 prompt。** 子 Agent 需要的 token 走环境变量注入，避免敏感串在回传摘要里漏回主会话。
5. **各自清理。** 子 Agent 的 session 落在 `~/.openclaw/agents/<agentId>/sessions/` 下，和主会话是两套文件。`/new`、`/compact` 只作用于当前 session，想彻底清理要单独删对应的 session 文件。

## 踩坑点

- **默认回传 ≠ 隔离。** 如果把子 Agent 的返回设成完整对话记录，隔离等于没做，主会话照样膨胀。
- **子 Agent 的“记忆”不回流。** 它过程中确认的事实，主会话并不知道，必须在结果里显式写出。别指望“刚才子 Agent 已经知道了”。
- **并发竞态。** 并行 spawn 多个子 Agent 时，别让它们写同一个输出文件，用任务 ID 做目录隔离。
- **多渠道串会话。** Telegram、网页端如果 dmScope 配得粗，不同来源可能落到同一个 session key 上，症状很像子 Agent 污染，其实是路由问题。排查时先用 `openclaw sessions` 看 key 归属，再下结论。

## 可复用建议

- 把“结论 + 数据 + 产物路径”的回传模板固化成 spawn 的标准 prompt 片段；
- 每个子任务一个输出子目录，命名带日期和任务 ID，方便事后审计；
- 定期（比如每周）检查 sessions 目录体积，顺手清理已完结任务的子 session；
- 主会话只保留“决策上下文”，执行细节一律外置到文件。

## 总结

会话隔离的本质是**上下文的所有权**：子 Agent 的过程归子 Agent，只有结论才值得进入主会话。把回传接口收窄、把产物外置、把凭证隔离，主会话就能长期保持干净，compaction 频率会明显下降。OpenClaw 的机制本身够用，难的是形成团队约定并坚持执行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/63d08a436abb52a0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/f80b4a5a86dcd5a2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/dba7f2f8706f87a9.png)

