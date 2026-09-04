---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 36142
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

用 OpenClaw 跑自动化任务一段时间后，很多人的主会话会慢慢"变重"：子 Agent 的中间输出、工具的原始返回、失败重试的报错，全被追加进主上下文。表现是主 Agent 回答越来越发散，改一个配置要翻很长的历史，甚至开始引用几小时前某个子任务里的过时信息。

这不是模型变笨了，是 session 隔离没做干净。

## 问题：三类典型污染

1. **上下文污染**：子 Agent 的过程性叙述和工具原始返回直接进主会话历史，token 占用持续膨胀。
2. **状态污染**：子 Agent 与主 Agent 共用同一 workspace，互相覆盖文件、抢同一个工作区。
3. **记忆污染**：子 Agent 把阶段性结论写进共享 memory，下次对话主 Agent 引用了早已失效的信息。

## 做法

**第一步：用 spawn，别让主 Agent 亲自干。** 通过 `sessions_spawn` 拉起子任务，子 Agent 拿到独立 session 文件，主会话只接收它的 final message。这是隔离的地基。

**第二步：把任务描述写"窄"。** spawn 的 prompt 里明确三件事：目标、约束（哪些目录可读、哪些工具可用）、输出格式。最后加一句"只返回最终结论，不要输出过程"，回流内容会显著减少。

**第三步：工具最小化。** 子 Agent 默认只给只读工具（读文件、搜索、查询）。写操作按需放开、用完回收。多数查资料类子任务根本不需要写权限。

**第四步：workspace 分离。** 涉及写文件的子任务，指定独立子目录，或直接用 sandbox。主 Agent 的工作区保持只被自己改动。

**第五步：频道任务开 thread 模式。** 挂在 IM 渠道上的子任务设 `thread: true`，回复走独立线程，不会插进主聊天流。

**第六步：设超时。** `run_timeout_seconds` 给合理上限，避免子 Agent 卡死占资源，也避免主 Agent 无限等待。

## 踩坑点

- **复用了旧 session**：spawn 时误传已有 session id，子 Agent 带着历史跑，隔离直接失效。每次用全新 id，或先确认目标 session 为空。
- **子 Agent 反向写主会话**：个别工具的回调目标是主 channel，结果中途插话。spawn 前检查工具的输出路由配置。
- **超时 ≠ 终止**：超时后任务可能还在后台跑，要确认进程真的被回收，否则会和下一个任务抢同一批文件。
- **返回结果太长**：子 Agent 老实交回三千字的"最终报告"，主会话照样被撑大。约定"结论 + 文件路径"格式，细节落盘，主会话只看摘要。

## 可复用建议

1. 沉淀一个 spawn 模板：目标 / 约束 / 工具白名单 / 输出格式 / 超时，五项必填。
2. session 文件按"日期-任务-序号"命名，事后审查 sessions 目录时方便对账。
3. 定期看子 session 的文件大小，异常膨胀往往意味着 prompt 没写窄。
4. 把"只读优先"固化成团队约定：能不给写权限就不给。

## 总结

session 隔离的核心不是某个开关，而是一组约束的组合：独立 session、窄任务描述、最小工具集、分离 workspace、明确超时。子 Agent 应该像临时工——干完活只交一份摘要，工具用完归还，不留垃圾在主人桌上。做到这几点，主会话的长期质量会稳定很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/52ceaa73c8a34fda.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/900139af801175fd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/db5fe46d722573ba.png)

