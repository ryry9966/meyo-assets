---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 37255
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

OpenClaw 的主会话是长生命周期上下文：一个持久化的 JSONL session 文件，里面堆着消息、工具调用、compaction 摘要。跑自动化时我们经常需要子 agent 干脏活——批量执行命令、翻网页、反复试错。麻烦在于，如果子 agent 和主会话共用上下文，它的全部中间过程都会灌进主会话。

## 问题

上下文污染通常有三种表现：

- 子任务的工具输出和失败重试挤占主会话窗口，compaction 提前触发，主任务的关键约束反而被压缩掉；
- 子 agent 的中间结论被当成主任务记忆，后续回答张冠李戴；
- 消息层面，子 agent 往同一个 channel 发进度，消息流全是噪音。

## 做法

我们的约定是「会话分层 + 契约通信」，五步：

1. **子 agent 必须用独立 session。** 用 `openclaw agent` 显式指定子任务自己的 session id，session 文件落在该 agent 的 sessions 目录下，生命周期与子任务绑定，不复用主会话的 key。
2. **下发任务契约，而不是上下文。** 给子 agent 的输入只有四样：目标、必要输入、输出格式（建议 JSON）、预算（最大轮数 / token 上限）。不复制主会话历史。
3. **回传结构化摘要。** 子 agent 结束时只把结果对象写回（文件或回调均可），主会话读摘要继续推理，不读过程。
4. **收窄权限。** 子 agent 的工具白名单按任务给，默认禁写长期 memory；exec、浏览器这类高权限工具走独立 workspace。
5. **及时清理。** 子任务结束即归档 session 文件，只保留摘要；失败重试开新 session，不在污染过的上下文里续跑。

## 踩坑点

- **session id 默认继承**：子 agent 拿到主会话的 session key 就直接往里写，等于没隔离。id 必须显式生成。
- **让子 agent 直接向 channel 汇报**：过程信息刷屏，汇报动作应该收回主会话做。
- **共享 memory 目录**：子任务把试错的错误结论写进了长期记忆，主会话后来一直引用错答案。memory 写入要门控。
- **子会话触发 compaction**：任务契约可能被压缩走，子 agent 开始自由发挥。契约放进每轮系统提示，别只放首轮。
- **中断后没人清理**：超时/崩溃留下僵尸 session，甚至残留 cron。编排脚本里务必加 finally 清理。

## 可复用建议

- 把「主会话 = 决策层，子会话 = 执行层」当架构原则，通信只走契约；
- 输出定义 schema，主会话做校验，不合格就重派，别靠子 agent 自觉；
- 预算前置：maxTurns、超时、token 上限写进任务契约；
- 强隔离场景（爬虫、批量脚本）直接用独立 agent + 独立 workspace，比共享 agent 再收权限省心得多；
- 定期跑清理脚本归档子 session，只留摘要索引。

## 总结

session 隔离的本质不是「多开几个文件」，而是控制信息流向：过程留在子会话，结论才回主会话。任务契约化、回传结构化、memory 门控，这三件事做扎实，主会话就能长期保持干净，自动化流程也更好排查。这套做法不依赖特定版本，核心是纪律，而不是配置。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/3b4e62e4ef979c96.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/38971523941a32ba.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/7ff1557bda7b2ce1.png)

