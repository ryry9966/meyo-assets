---
title: OpenClaw 工程实践：让 AI 助手从“问一句动一下”到可靠主动触发
feedId: 34968
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

多数 OpenClaw 实例接上 MCP、插件后，工具能力不弱，但触发方式仍然以人开口为主。proactive 能力不是让 AI 替你发布、删库、重启服务，而是把低风险、高频的发现类动作前移：证书快过期、备份没跑完、PR 躺了五天、某个队列开始堆积。

这类任务的价值不在模型多强，而在触发、去重、审批、送达是否可靠。

## 问题

早期尝试 proactive 时踩过几类坑：

- 用 cron 直接拉起完整 agent，每次跑一堆上下文，token 成本高。
- 收到事件后直接交给大模型判断是否处理，噪声多、响应慢。
- 主动任务没有去重，同一事件重复提醒。
- 执行权限开太大，总担心误操作。

后来把 proactive 拆成三段：**事件源、决策器、执行器**，问题才收敛。

## 做法

### 1. 事件源只做轻量探测

不要让定时任务直接跑复杂 agent。优先让 cron 或 systemd timer 调用 MCP tool 做 read-only probe。例如：

- `cert_watch.check(domain)` 返回证书剩余天数；
- GitHub MCP 查询 PR 最近更新时间；
- 文件系统检查备份目录最新文件时间。

只有 probe 结果命中阈值，才进入后续决策。

### 2. 决策先规则，后模型

原始事件先经过规则过滤：阈值、白名单、静默窗口、去重。满足条件后，再让模型做归纳或建议。不是所有事件都需要 LLM，模型只处理“需要解释”的部分。

一个抽象配置示意：

```yaml
proactive_rules:
  - name: cert-expiry
    source: cron
    schedule: "0 9 * * 1"
    probe:
      tool: mcp.cert_watch.check
      args: { domain: "example.com" }
    condition:
      min_days_left: 14
    decision:
      summarize: true
      approval: required
    action:
      type: notify
      channel: ops
    dedupe:
      key: "cert-expiry:example.com"
      ttl: 24h
```

这是逻辑示意，不是完整可用配置。实际可以包成 OpenClaw 的 skill，或由外部调度器调用 CLI/MCP 完成。

### 3. 执行器先收口

主动动作先限定为：发通知、建 issue、打标签、写入队列。危险操作只生成 dry-run 报告；需要审批时，用明确状态机，而不是让模型输出“是否同意”。

## 踩坑点

**定时频率过高，token 成本失控**  
如果每次定时任务都拉大模型看一遍原始信息，成本会很快上涨。正确做法是 probe 廉价、确定性执行，只有异常才进 LLM。

**去重只看“发没发过”**  
证书续期后，域名相同但事件指纹可能变化。建议用事件内容哈希 + 时间窗口，并保存 cursor/状态，避免漏报和重复报。

**权限边界模糊**  
不要让 proactive 任务拥有重启服务、合并 PR、删除资源等写权限。先 read-only，逐步开放。审批流是状态机，不是自然语言确认。

**通知过载**  
同类事件连续到达时应聚合为一个摘要；夜间非紧急事件进入静默窗口，次日合并发送。

**忽略运行幂等**  
cron 可能重复触发，任务可能跑一半被再次拉起。用 run_id 或本地状态锁，保证同一事件在执行中不并发处理。

## 可复用建议

1. 从单一高价值场景切入，比如证书过期、备份失败、PR 超过 N 天未处理。
2. 所有 proactive 任务必须有 dry-run 和日志，至少记录 event_id、fingerprint、run_id、action 结果。
3. 把 observe/decide/act 拆开单独测试，不要每次改完都端到端跑。
4. 事件量小的时候先规则后模型，别让 LLM 当数据库。
5. 主动任务设置静默窗口和优先级，避免凌晨打扰。

## 总结

Proactive 能力最怕“看起来很智能，实际不可控”。如果把触发、去重、审批、送达这四件事工程化，哪怕规则很朴素，也能稳定发挥作用。OpenClaw 的 MCP 和插件能力足够支撑这条链路，缺的通常不是模型能力，而是约束和可观测性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/7b4fdaa37a7db6bd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/f5ada4134de7fda3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/9c42637d74391cf7.png)

