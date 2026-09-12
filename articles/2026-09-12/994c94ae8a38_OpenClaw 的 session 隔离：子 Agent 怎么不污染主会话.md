---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 37172
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

OpenClaw 里主会话是你和 agent 的唯一长期对话：上下文、记忆、工具状态都在这一条线上滚动。一旦开始做多 agent 编排——用 `sessions_spawn` 拉子 agent 跑并行任务、长任务、探索性任务——session 边界就从"实现细节"升级成了"架构问题"。

## 问题：不隔离会发生什么

子 agent 如果和主会话共享上下文，污染主要来自四处：

1. **上下文膨胀**：子 agent 的中间推理和工具输出全部涌入主会话，几轮下来 compaction 提前触发，精心维护的长期上下文被压掉。
2. **消息交错**：子 agent 的过程性消息直接进主频道，你看到的不再是结果，而是一地过程。
3. **状态互踩**：子 agent 和主 agent 共享 workspace，同时写同一份 notes/state，后写的赢。
4. **失败半径**：子 agent 里一次失控的工具调用，和主会话的可靠性绑在一起。

## 做法：三条边界

**1. session 边界。** spawn 时让子 agent 落在独立 session，不继承主会话历史，只拿到任务描述，跑完只回传最终结果。OpenClaw 默认方向如此，关键是别在 spawn prompt 里手动把主会话上下文整段粘过去。

**2. 回传边界。** 约定回传格式：固定三段（结论 / 变更文件 / 遗留问题）。主 agent 收到后先自己摘要再入上下文，不原文透传——这一步能把几百 token 的过程输出压成一两句。

**3. 文件边界。** 子 agent 需要写文件时，给它独立子目录（如 `workspace/subagents/<task>/`），主会话只消费回传结果，不读中间产物。

一个常用的 spawn prompt 骨架：

```
任务：只做 X，不做其他。
不要假设主会话上下文，所需信息都在本任务描述里。
完成后只回传：结论、改动文件路径、未解决问题。
禁止主动向主频道发消息。
```

## 踩坑点

- **子 agent "好心"汇报**：不显式禁止时，它可能直接 message 到主频道。上面那句"禁止主动发消息"不是废话。
- **僵尸子 agent**：长任务没设上限，主会话一直干等。spawn prompt 里写明"最多 N 分钟 / N 轮工具调用，超时回传部分结果"。
- **共享 state 文件**：多个子 agent 并行写同一个 `state.json` 会互相覆盖，改成每 agent 一份、主会话合并。
- **中间结论当事实**：子 agent 回传的"已完成"要抽查，文件改动类任务用 read 验证一次，成本很低。
- **session 垃圾场**：子 agent session 用完不清理，`sessions_list` 很快没法看。任务结束顺手清，或按 key 前缀批量清。

## 可复用建议

- 职责单一化：一个子 agent 只做一件事，需要编排就多 spawn 几次，别造超级子 agent。
- 把"三边界"（session / 回传 / 文件）固化成自己的 spawn 模板，每次复用。
- 探索性任务（调研、试错、不确定能不能成）一律走子 agent，主会话只收结论，保住长期上下文质量。
- 把验证动作（read 关键文件、跑一次测试）固化为主 agent 的收尾步骤，形成固定回路。

## 总结

session 隔离的本质是控制两个东西：**上下文边界**和**失败半径**。子 agent 的全部意义，就是让它可以放心地"跑脏"——脏活、长活、可能失败的活——而主会话始终保持干净可信。OpenClaw 的隔离机制给了你边界，剩下的纪律（回传格式、权限、清理）要靠模板固化，而不是靠每次临场发挥。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/d0c5c1e666530685.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/3a61a0b69e82ff7c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/f2154230a05dab3f.png)

