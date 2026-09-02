---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 35766
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

OpenClaw 里每个 agent 都跑在带 key 的 session 里，主会话承载用户对话和工具调用的完整上下文。随着自动化任务变多，它会越来越胖：读文件、浏览器快照、长日志，全都进了 transcript。OpenClaw 有 compaction 兜底，但压缩是被动手段——污染先发生，压缩才补救。

## 问题：主会话被"顺手"污染

典型场景：在主会话里让模型"盘点 docs 下所有失效链接"，它直接开始逐个读文件、抓页面，几十次工具调用的原始输出全部堆在主上下文里。后果有三层：

1. 上下文膨胀，模型开始"忘记"更早的约定；
2. 无关细节（错误栈、HTML 片段）持续干扰后续判断；
3. 提前触发 compaction，摘要时丢掉你不想丢的东西。

根因很简单：把"执行过程"和"对话主线"放进了同一个上下文。

## 做法：让子 Agent 带着自己的 session 去干脏活

`sessions_spawn` 就是为这个准备的，步骤如下：

1. **主会话不直接执行重任务**，让主 agent 调 `sessions_spawn`，把任务写成一份自包含的 task prompt：目标、范围、输出格式，并明确要求"只返回结论/摘要"。
2. 子 agent 拿到独立的 session key，在自己的上下文里跑完所有工具调用；中间输出、错误重试、文件内容都留在它自己的 session 里。
3. 子 agent 结束后，主会话的 tool result 只拿到一份最终摘要。主 transcript 新增的只有一条 spawn 调用加一段结论。
4. 长任务用 background 模式异步跑，主会话继续响应用户；用 `sessions_status` 轮询进度，必要时用 `sessions_history` 做事后审计。
5. 子 session 用完即弃（配好超时和清理），别让它长成第二个"主会话"。

系统提示层面建议加一条纪律：凡是会产生大量中间输出的任务——批量读文件、网页抓取、日志分析——一律 spawn，不在主会话直接执行。

## 踩坑点

- **task prompt 不自包含**：子 agent 看不到主会话历史，路径、约束、格式必须写全，否则它会反复追问或自作主张。
- **忘了限定输出**：子 agent 返回一份五千字"完整报告"，主会话照样被撑爆。要求它"N 字以内、只给结论和异常项"。
- **并发 spawn 太多**：十几个子 agent 同时跑，API 限流和 token 成本一起上头。控制并发，能串行的串行。
- **状态没落盘**：子 agent 的结果只存在于摘要里，下游要用就写文件、发消息或写状态库，别指望回头翻子 session。
- **定时任务也走主会话**：cron 触发的重活同样该 spawn，否则污染只是换了个时间点发生。

## 可复用建议

- 把"读多写少、输出大、一次性"的任务归为 spawn 候选，写进团队模板。
- 给子 agent 统一摘要模板：结论 / 异常 / 产物路径，三段式，主会话好消费。
- 定期用 `sessions_list` 巡检，看看哪些 session 异常长寿、该清了。

## 总结

session 隔离的本质，是把"过程的噪音"关进子上下文，只让结论回到主线。做法不复杂：重任务 spawn、task 自包含、输出限幅、结果落盘。主会话干净了，模型的长程表现和你的账单都会感谢你。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/48bea825fc9b0a4f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/efbd61a26d489cbb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/14aeef0a23b554bc.png)

