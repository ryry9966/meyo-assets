---
title: 不等你开口：给 OpenClaw Agent 做 proactive 的三层工程结构
feedId: 35893
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

多数 AI 助手默认是"你问我答"的被动模式。但 OpenClaw 这类常驻 agent 网关的价值恰恰在于它一直在跑：挂着 channel、装着插件、接着 MCP 工具、还有 heartbeat。如果只在被 @ 时才工作，等于养了个全职员工却只让他接电话。所谓 proactive，就是让 agent 基于外部信号自己发起任务：CI 红了主动报、日程冲突前提方案、每天固定时间给一条摘要。

## 问题

把"你要主动一点"写进 system prompt 是最常见的第一反应，效果通常很差：模型会自己编造触发时机、频繁输出"暂无可汇报事项"的空转轮次、token 成本失控，还会在你没授权时自作主张。本质原因是：模型不擅长决定"什么时候"，但擅长决定"怎么做"。所以 proactive 不是 prompt 问题，而是触发、决策、权限三层叠加的工程问题。

## 做法

我目前在用的结构分三层：

**1. 确定性触发层。** 所有触发来自确定性来源：cron 定时、插件事件（git push、CI 失败、日历变更）、webhook、文件监听。绝不把"何时主动"交给 LLM。heartbeat 保持低频（30 分钟起步），只作兜底巡检，不做主触发。

**2. 廉价决策层。** 每个触发先过一个小模型或规则过滤器，输入是触发上下文摘要，输出三选一：act / ignore / ask。实践中九成触发在这一层被丢弃，只有高信号事件才进主模型。这是成本的关键闸门。

**3. 受限执行层。** proactive 会话单独配置工具白名单：只读操作直接执行；写操作默认 dry-run，先回报计划；高风险动作转 ask。执行完往 channel 推一条结果卡片，必须带一句"为什么做这件事"。

一个最小配置示例：

```yaml
triggers:
  - cron: "0 9 * * 1-5"     # 工作日早报
  - event: ci.failed         # CI 失败即时通知
decision:
  model: small
  output: [act, ignore, ask]
proactive_tools:
  deny: [deploy]
  dry_run: [write_file, post_message]
```

## 踩坑点

- **heartbeat 太密。** 最初设 5 分钟，一天烧的 token 比正常聊天还多，且绝大多数轮次输出"无事发生"。降到 30 分钟并改为事件驱动后，成本降了一个量级。
- **主动消息变成骚扰。** 连着几天收到无信息量的早安摘要后，我把自己的 bot 静音了——这是最讽刺的失败方式。现在低优先级事件进缓冲区，聚合成每日一条 digest，并设 quiet hours。
- **会话污染。** proactive 轮次写进主会话会把上下文越撑越大，日常对话质量肉眼可见地下滑。单独开 session，跑完只回写结果摘要。
- **触发抖动。** CI 自动重试导致同一事件 10 分钟内触发三次。给事件加去重 key 和冷却窗口。
- **静默失败。** proactive 任务没人盯着，挂了也不知道。失败通知本身也要是一条通知。

## 可复用建议

- 一句话原则：**触发要确定，决策要便宜，动作要限权。**
- 按风险分三档响应：静默执行只记日志；执行并通知；先问再做。新场景一律从第三档起步，跑稳了再降档。
- 每周回看触发日志，统计漏报和误报，据此删规则、调阈值。proactive 的精度是维护出来的，不是一次配置出来的。
- 先只做一两个高信号场景（CI 失败、日程冲突、deadline 提醒），跑出信任再扩展。信任是 proactive 的前置条件。

## 总结

proactive 的难点不在模型，在工程：把"什么时候主动"交给确定性触发器，把"要不要行动"交给廉价决策层，把"能做什么"关进权限白名单，再把结果和理由透明地交还给人。这套做完，agent 才真正从问答工具，变成一个在值班的助手。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/4f6c07e4342ab497.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/b5641fb0c1f54179.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/179e259aacb3a5f1.png)

