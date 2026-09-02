---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 35842
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

OpenClaw 里每个 agent 的对话都挂在一条 session 上：主会话保存完整的消息流和工具调用记录，模型每轮看到的就是这条 transcript 加上 workspace 里的文件。当你用 `sessions_spawn` 拉起一个子 agent 去干脏活——批量抓网页、跑长任务、整理文件——它会拿到自己独立的 session id 和上下文。理论上隔离是天然的，但实际用下来，"污染"还是会从别的路径渗回主会话。

## 问题：污染有三条路径

1. **上下文污染**：子 agent 的工具输出如果整段塞回主会话的 tool result，几千 token 的抓取结果会直接把主 transcript 撑胖，触发提前 compaction，主 agent 的"长期记忆"被挤掉。
2. **文件系统污染**：子 agent 和主 agent 默认共享 workspace。它顺手改了记忆文件或工作文件，主会话下一轮就读到了，而且你很难意识到变化来自哪。
3. **会话串号**：复用 session key，或者让子 agent 通过 `sessions_send` 直接往主会话发消息，等于自己把墙拆了。

## 做法

1. **只回传结论，不回传过程**。spawn 时在提示词里明确要求子 agent 返回结构化摘要（固定几个字段），完整过程留在它自己的 session 里，需要回查时用 `sessions_history`。
2. **限制工具面**。给子 agent 最小工具集：只读工具 + 必要的写工具，不给它 `sessions_send`，从根上防串号。
3. **工作目录隔离**。批量任务放进独立的 scratch 子目录，任务结束后由主 agent 决定哪些文件"转正"。
4. **限制深度和并发**。子 agent 默认不再继续 spawn，深度封顶一层；并发控制在 2~3 个，避免 token 和速率双爆。
5. **回收**。后台任务用完即关，定期用 `sessions_list` 扫一眼有没有僵尸 session 占预算。

## 踩坑点

- 最坑的一次：子 agent 把抓取的原始内容顺手写进了记忆文件，之后主会话每轮都带着一堆无关摘要。排查了半天以为是上下文问题，diff 了 workspace 才发现是文件污染。
- 子 agent 返回"成功"但返回值是空的，主 agent 拿着空结果继续推理，越走越偏。返回值格式必须在 spawn 提示词里写死，缺字段就要求重试。
- 忘了后台任务有生命周期，几个长任务挂着不收，后台队列被占满，新任务排不进去。

## 可复用建议

- 把子 agent 当函数调用：入参明确、返回值有 schema、副作用隔离、用完即走。
- 主会话只保留"决策上下文"，"劳动过程"全部下沉到子 session。
- 出问题先分清是上下文脏了还是文件脏了：`sessions_history` 看前者，diff workspace 看后者，别一上来就重置会话。

## 总结

session 隔离在 OpenClaw 里是默认行为，但真正的隔离边界有三条：transcript、文件、会话键。只把摘要带回主会话、锁住子 agent 的工具面和工作区、及时回收任务，主会话就能长期保持干净，compaction 也不会被脏数据提前触发。隔离不是配置出来的，是约束出来的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/91c0bc386742a9fe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/1e19f1bd86184cb9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/fd7f59a042934858.png)

