---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 36177
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

OpenClaw 里主会话（session）承载你和 Agent 的完整对话：系统提示、历史消息、每次工具调用与返回都在里面。当主 Agent 接到一个重活——批量检索代码、跑长链路自动化、连续处理大输出——如果这些中间过程全量写进主上下文，问题很快就来：token 成本线性上涨、早期指令被稀释、回答质量肉眼可见地下滑。子 Agent（subagent）机制就是为这个场景设计的。

## 问题

实际的"污染"通常来自三条路径：

1. 主 Agent 自己连打几十次工具调用，所有返回都堆进主会话；
2. 子 Agent 的中间过程被流式回传，等于换个地方堆垃圾；
3. 多个子 Agent 误用同一个 sessionId，互相写入对方历史。

最终表现是：上下文膨胀、该记住的东西被挤出窗口、单次对话费用翻倍。

## 做法

OpenClaw 的隔离逻辑一句话就能说清：**子 Agent 拥有独立 session，主会话只接收结果摘要**。

以当前版本为例，流程是：

1. 主 Agent 通过 `sessions_spawn` 派生子 Agent，传入一段**自包含**的任务描述；子 Agent 拿到全新的 sessionId，与主会话零共享上下文。
2. 子 Agent 在自己的 session 里自由调用工具，中间输出只落在它自己的历史文件里。
3. 任务结束后，子 Agent 把最终结论作为 result 返回，主会话只追加这一条摘要；如果开启了 announce，才会向频道广播一条完成通知。
4. 用 `openclaw sessions` 查看会话列表，确认子任务确实落在独立 session；排障时看子 Agent 自己的 session 文件，不要翻主会话历史。

核心约束：task prompt 必须自包含（子 Agent 看不到主会话），返回内容控制为一小段结论。

## 踩坑点

- **以为子 Agent 能读主会话**：读不到。路径、参数、验收标准都要写进任务描述，否则它只能猜。
- **结果摘要写太长**：几百行的"结果"等于把污染搬了个位置。约定只回结论 + 关键数据。
- **消息权限没收紧**：子 Agent 的输出行为要在任务描述里约束清楚，避免过程消息直接推到用户频道。
- **并行子 Agent 抢共享资源**：写同一文件或同一目录会互相覆盖，按任务分目录或加锁。
- **不设超时**：长任务子 Agent 挂着，主会话一直等。配好 timeout 和失败兜底。

## 可复用建议

- **一次任务一个子 Agent**：预计超过十次左右工具调用的活，一律下沉，主 Agent 只做拆解与验收。
- **数据落盘、摘要回传**：大输出写文件，返回值里只给路径和结论，主会话保持轻。
- **把子 Agent 当纯函数设计**：输入自包含、输出结构化、副作用不泄漏回主会话。
- **定期检查 session 体积**：配合 compaction（压缩）或按需重置，别让历史无限增长。

## 总结

session 隔离本质是边界管理：主会话负责"与用户的对话状态"，子 Agent 负责"干脏活"。让过程留在子 session、只让结论回家——上下文干净了，响应质量和成本都会明显改善。这条原则与具体插件无关，做任何 OpenClaw 自动化时都值得先想清楚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/938e82cb95060132.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/47a85a649ee7c8a8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/08ff1268b577c9c5.png)

