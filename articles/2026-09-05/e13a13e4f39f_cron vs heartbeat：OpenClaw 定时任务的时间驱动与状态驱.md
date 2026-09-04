---
title: cron vs heartbeat：OpenClaw 定时任务的时间驱动与状态驱动之争
feedId: 36098
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

OpenClaw 里有两条让 agent "自己动起来"的路径：**cron** 和 **heartbeat**。不少新用户的配置里两者都有，但分工是模糊的——同一件事有时 cron 发一次、heartbeat 又发一次，有时该准时的事偏偏迟到。

先说清楚两者的本质：

- **cron**：精确的调度器。到点执行一条预定义的 prompt，时间到就跑，不触发不花钱。
- **heartbeat**：agent 的周期性脉搏。每次到点醒来，扫一眼工作区里的 `HEARTBEAT.md`，自己判断有没有值得做的事，没事就沉默。

一句话概括：**cron 是时间驱动，heartbeat 是状态驱动**。

## 问题

混用或错用导致的典型故障：

1. 用 heartbeat 做"每天 8 点发日报"，结果时间漂移，9 点半才到；
2. `HEARTBEAT.md` 越写越长，每次 tick 都全量评估，token 消耗肉眼可见地上涨；
3. cron 任务静默失败，连续三天没发报告才发现，且无法追溯原因。

## 做法

### 确定性的活交给 cron

固定时间、固定交付物的任务全部走 cron（以聊天命令为例，具体参数名以你版本的文档为准）：

```
/cron add "工作日晨报" \
  --expr "0 8 * * 1-5" \
  --tz Asia/Shanghai \
  --message "汇总昨天的提交和待办，发到 #daily"
```

要点：显式指定时区（容器默认 UTC 是最常见的时间错位来源）；输出路由到固定 session，别和交互会话混在一起。

### "看一眼"的事交给 heartbeat

`HEARTBEAT.md` 只写**条件**，不写任务：

```markdown
- 若收件箱有未处理的高优消息 → 摘要并通知
- 若磁盘剩余空间 < 20% → 告警
- 其他情况：保持沉默
```

间隔从 30 分钟起步，别贪快。最后一条"保持沉默"很关键——显式告诉 agent 没事别吭声，能显著降低误触发。

### 推荐的组合模式

cron 干活，heartbeat 监工。比如 8 点的日报由 cron 负责，heartbeat 里加一条："若今天 8 点的晨报未发出 → 补发并说明原因"。这样确定性和可靠性都有了。

## 踩坑点

1. **heartbeat 不是闹钟**。它是 best-effort 脉冲，agent 繁忙或会话排队都会让 tick 推迟。任何"必须在 X 点发生"的需求都进 cron。
2. **HEARTBEAT.md 是成本放大器**。每一条都会进入每次 tick 的上下文，超过 5 条后单次开销明显上涨，建议控制在 3~5 条。
3. **cron 失败是静默的**。跑没跑、发到哪，要么看日志，要么用 heartbeat 兜底检查。
4. **长任务别挂 heartbeat**。下一次 tick 会与它重叠或被跳过，重活交给 cron 或独立会话。
5. **同一条业务别同时出现在两处**，通知会互相淹没，排查时也分不清是谁触发的。

## 可复用建议

- 选型口诀：知道"**什么时候做**"用 cron；只知道"**什么条件下做**"用 heartbeat。
- heartbeat 条目用条件句式（"若…则…"），不要用任务句式（"每天做…"）。
- 上线后跑一周，统计每次 heartbeat 的平均 token 消耗，超预算就精简条目或拉长间隔。
- 把"cron 出报告 + heartbeat 做校验"当成默认架构，而不是二选一。

## 总结

两者不是竞争关系，而是互补：cron 提供确定性的时间承诺，heartbeat 提供低成本的状态感知。选错的代价是时间漂移和 token 浪费，选对的标志是——cron 管日程，heartbeat 管异常，互不越界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/58fe49e1cda91131.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/fce214dff197b8e5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/7f82d0a54c087970.png)

