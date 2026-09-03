---
title: OpenClaw session 隔离实践：子 Agent 如何不污染主会话
feedId: 35946
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

OpenClaw 的 Gateway 为每个会话维护独立上下文：主会话（私聊里的 `main`，或群组按 channel 生成的 session key）长期存活，历史以 JSONL 落在 `~/.openclaw/agents/<agentId>/sessions/` 下。主会话里堆的是你的指令、偏好和决策历史，它的干净程度直接决定 agent 的长期表现。

子 Agent 是另一回事。通过 `sessions_spawn` 派生的运行会拿到独立的 session key（形如 `agent:<父会话>:sub:<label>`，不同版本前缀略有差异），有自己的 transcript 和上下文窗口，跑完后只有最终结果作为 tool result 回到主会话。

## 问题

机制上天然隔离，但实践中污染主会话的路子很多：

- 调研、批量文件处理这类重活直接在主会话干，几万 token 中间输出撑爆上下文，触发 compaction，早期关键指令被压掉；
- 子 Agent 通过 `sessions_send` 反向往主会话发消息，中间过程绕一圈灌回来；
- 子 Agent 产物没有落盘习惯，只能把大段内容贴回对话。

结果都一样：主会话越长越钝，重置又丢状态。

## 做法

1. **重活一律 spawn。** 任务描述里写清输出契约：“只返回结论、关键数据和产物路径，不复述过程。”
2. **收窄工具面。** spawn 时限制子 agent 的工具列表，至少去掉 `sessions_send`，杜绝反向写主会话。
3. **中间产物落盘。** 约定子 agent 把明细写进 workspace 的 `scratch/<任务名>/`，主会话只拿路径和摘要；需要细节时用 `sessions_history` 按需回查，不默认加载。
4. **定期检查。** 用 `/sessions` 或 `openclaw sessions` 看会话列表，确认子会话确实分离、没有异常膨胀。
5. **彻底隔离的场景**，直接为该类任务单独配一个 agentId——独立 workspace 加独立 session 存储，物理隔离。

## 踩坑点

- **子 Agent 反向发言**：没砍掉 `sessions_send`，子 agent“好心”把完整调研发回主会话，等于没隔离。
- **后台任务没人收尸**：spawn 一堆后台子会话不清理，session 文件膨胀，Gateway 重启后的恢复明显变慢。
- **compaction 后重复派发**：主会话被压缩后忘了自己派过什么，同一个任务反复 spawn。约定派发前先查 `scratch/`。
- **并发写冲突**：多个子 agent 写同一个文件。任务粒度拆开，目录隔离。
- **模型继承**：子 agent 默认继承主会话模型，跑批量机械活前记得降级，否则账单很难看。

## 可复用建议

把子 Agent 当函数调用设计：输入明确、输出收敛、副作用可控。主会话只保留“决策上下文”，执行细节交给子会话和文件系统。团队内可以固化一个 spawn 模板：任务目标、输出格式、产物目录、禁用工具四项必填。

## 总结

OpenClaw 的隔离机制本身够用，问题多半出在使用姿势：spawn 而不是主会话硬扛、砍掉子 agent 的回写通道、产物落盘、定期看 sessions。隔离的本质不是“少用上下文”，而是让每一层只保留它该记住的东西。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/fb3b52e95f4e9369.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/1ef3d1771ef9698b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/fb9cc2a6319b8d8e.png)

