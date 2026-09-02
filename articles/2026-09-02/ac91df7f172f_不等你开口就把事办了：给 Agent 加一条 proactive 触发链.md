---
title: 不等你开口就把事办了：给 Agent 加一条 proactive 触发链
feedId: 35780
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

大多数 AI 助手默认是"应答机"：你发消息，它回消息。但工程里很多事有明确触发条件——CI 挂了、证书快到期、日报该汇总了。这些事不该等你"想起来去问"，而应该在开口之前就有一条链路在跑。OpenClaw 的 Agent + MCP 组合天然适合做这件事：触发靠外部事件，判断靠模型，执行靠 MCP 工具。这篇帖记录我给团队搭的 proactive 链路的做法和踩坑。

## 问题

直接往 system prompt 里写"你要主动一点"没用。真实要解决的是三件事：

1. 模型不会自己醒来，缺触发源；
2. 事件一多，LLM 调用和推送会失控；
3. 主动执行的写操作出一次事故，信任就没了。

## 做法

按五层搭：

1. **触发层**：Cron（定时扫描）+ Webhook（CI、监控告警）汇入统一事件队列，每个事件带 `source`、`dedup_key`、`payload`。
2. **预过滤层**：纯代码，不进模型——静默时段（非工作时间只累积不推送）、去重（同 dedup_key 30 分钟冷却）、阈值（告警级别低于 P3 直接丢弃）。这一层砍掉了约八成无效触发。
3. **决策节点**：LLM 只做一件事，结合事件与 MCP 工具拉到的上下文（git、日历、监控），输出结构化结果 `{action: ignore|digest|notify|execute, confidence, reason}`，用 JSON schema 约束，temperature 0。
4. **分级执行**：ignore 写日志即止；digest 进每日汇总；notify 单条推送并附 reason；execute 只允许白名单内的只读或低风险动作，写操作一律降级为"提案"，等用户确认。
5. **反馈回路**：用户对推送的处理（忽略/点开/确认）写回统计，每两周调一次阈值和 allowlist。

## 踩坑点

- **触发风暴**：监控抖动时一分钟进来几十条 webhook，dedup_key 没带实例 ID，结果合并成一条谁都看不懂的通知。键要设计到"同类事件、不同对象能分开"。
- **时区坑**：静默时段判断用了容器的 UTC 时间，凌晨三点被推送吵醒。必须显式配置 IANA 时区。
- **上下文过期**：事件触发到模型决策隔了几十秒，问题已被同事处理。执行前要重拉一次状态校验，state 变了就降级为 ignore。
- **策略写进 prompt**：起初把"什么该推"全写在 prompt 里，改一次阈值要动 prompt 还要回归测试。后来全部挪到配置文件，prompt 只保留判断逻辑。

## 可复用建议

- 第一个 proactive 场景选"只产草稿不执行"的（日报、摘要、提醒），先攒信任再放开执行能力。
- 推送预算化：每个触发源每天最多 N 条，超了进 digest。宁可漏一条，不可烦一轮。
- 决策日志落盘：action、confidence、reason、当时拉了哪些上下文。排查"它为什么没推/为什么推了"全靠这份日志。
- execute 白名单初始为空，每加一个动作都要配对应的回滚方式。

## 总结

Proactive 的难点不在模型多聪明，而在工程闸门够不够扎实：触发收敛、决策可解释、执行有边界。上线一个月，我们推送打开率从三成升到八成以上，说明分级 + 冷却 + 白名单这套组合是有效的。建议从一条 cron 加一个 digest 场景起步，别一上来就做全自动执行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/2d785a1cc6ae784b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/c4ae02e744e2b9e4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/858840aeb4c3f1a1.png)

