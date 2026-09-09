---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 36782
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

OpenClaw 的主会话承载着用户与主 Agent 之间的完整上下文：聊天记录、工具调用、文件引用，全都在一条历史里。子 Agent 的价值在于把"脏活"——批量检索、长文档解析、多轮试错的重试循环——挪到独立 session 里跑。隔离做得好，主会话只收结果；做不好，子 Agent 的中间过程会一路灌回主上下文，表现为 token 消耗暴涨、主 Agent 决策被噪音带偏、回复质量肉眼可见地下降。

## 问题：污染是怎么发生的

实际用下来，污染基本来自三类：

1. 子任务复用了主 session，所有中间 tool call 直接写进主会话历史；
2. 子 Agent 返回的不是结论，而是整段执行日志或完整 transcript；
3. 共享同一个 workspace/memory，子 Agent 顺手改了 MEMORY.md 或写了同名文件，反过来影响主 Agent 后续判断。

## 做法

以多 agent 配置（`~/.openclaw/openclaw.json`，具体字段名以当前版本文档为准）为例：

1. **按职责拆 agent**：给子任务单独建 `researcher`、`crawler` 之类的 agent，每个 agent 有自己的 workspace，session 文件天然分开存在 `~/.openclaw/agents/<id>/sessions/` 下；
2. **用 spawn 类工具发起子任务**：主 Agent 指定 agentId 和任务描述，子 Agent 全程在自己的 session 里跑，中间 tool call 不会进主会话；
3. **约定"只回结果"**：要求子 Agent 的最终回复是结构化摘要——结论、关键证据、产物路径，parent 只接收这条 final message；
4. **收窄工具面**：子 Agent 只保留完成任务必需的工具，不需要的写能力直接不给；
5. **产物落盘而非回传**：结果写到 `workspace/artifacts/<task-id>/`，主 Agent 需要时按路径读取，而不是把全文塞进上下文。

验证方法也简单：跑一次子任务后查看主会话记录，应该只多出一两条 final message，而不是几十条 tool call。建议每个新流程第一次上线都做这个检查。

## 踩坑点

- **spawn 忘了指定独立 session/agent**，默认回落主会话，这是最常见的污染源；
- **final reply 没限长**，几千字的"总结"回来照样撑大上下文，建议在提示词里硬性约束字数和结构；
- **多个子 Agent 并行写同一 memory/文件**，出现互相覆盖，应按任务 ID 隔离目录，写操作收敛到单独的"汇总"步骤；
- **长任务超时后被反复 spawn**，主 Agent 只拿到报错却没有重试预算，失败循环同样是一种污染——污染的是成本，重试次数要设上限。

## 可复用建议

- 把"隔离跑批 → 摘要回传 → 产物落盘"固化成模板提示词，新流程直接套；
- 用 `task-<date>-<slug>` 命名约定管理子任务产物，方便事后清理和审计；
- 定期 review 主会话长度，一旦发现 tool call 异常增多，优先怀疑 spawn 隔离没生效。

## 总结

Session 隔离的本质是**控制信息回流**：子 Agent 可以在自己的会话里随便折腾，但回到主会话的只有一份受控的、结构化的结果。把这个约定写进 agent 配置和提示词模板，而不是每次靠人肉提醒，主会话的上下文质量和运行成本都会稳定很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/8c4f517d7c3104b1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/f6cdb09af5d2f867.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/692c45bce32d8103.png)

