---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 35935
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

多数人用 Agent 的方式还是问答机：打开会话、提问、得到回答、关掉。OpenClaw 在对话之外给了第二条通道——**心跳**。Agent 每隔一段时间自动醒来，读取工作区里的 `HEARTBEAT.md`，判断有没有该做的事。这让 agent 从"被动应答"变成"定时巡查"。

## 问题

没有心跳机制的常见痛点：

- 周期性任务全靠人记：看一次服务器日志、确认备份、汇总订阅，最后都是自己上；
- 在对话里写"以后每小时提醒我"这类指令不可靠，会话一长 agent 就忘；
- shell、浏览器、MCP server 各管一摊，缺一个统一的触发点。

## 做法

**1. 确认心跳已开启。** 在 `openclaw.json` 里配置：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "target": "telegram:your_chat_id"
      }
    }
  }
}
```

**2. 在工作区建 `HEARTBEAT.md`**（默认 `~/.openclaw/workspace/HEARTBEAT.md`）。这个文件本身就是给 agent 看的提示词，建议用清单体：

```markdown
# HEARTBEAT
- 每次醒来：检查 workspace/inbox/ 是否有未处理文件，
  有则摘要后发到 target，处理完移入 done/
- 每天 21:00 后：跑 df -h，磁盘使用超 85% 才告警
- 每周一：汇总上周 RSS 抓取结果，压到 5 条以内
无事可做时直接回复 HEARTBEAT_OK，不要展开。
```

**3. 工作方式。** 每次醒来，agent 逐条判断。没事做就回 `HEARTBEAT_OK`（不通知、不消耗你的注意力）；有事就直接执行，把结果推到 target 频道。

**4. 分层处理。** 需要不同频率的任务，在条目里写清"何时动手"让 agent 自行判断；重活建议交给系统 cron 加 webhook，心跳只负责轻量巡检。

## 踩坑点

- **条目写成愿望而非条件。**"帮我关注 AI 新闻"没法执行；要写成"抓取 X 列表，超过 3 条新内容才汇总"。
- **文件越长，每次心跳越贵。** `HEARTBEAT.md` 会随提示词注入每次醒来，控制在几十行内，做完的事及时删。
- **心跳跑在主会话里**，任务太多会污染对话上下文。任务重就单独建一个 agent，或给心跳配便宜模型。
- **间隔别贪短。** 30 分钟够用，5 分钟一次纯属烧钱。
- **别放无确认的破坏性操作**（rm、批量发消息）。心跳是无人值守场景，出事没人拦。

## 可复用建议

- **固定模板：每条任务三要素**——检查什么、触发阈值、触发后动作。
- **巡查结果落盘**：让 agent 把每次结论 append 到 `log.md`，出问题可回溯。
- **心跳只做"发现"，复杂动作走 skill/MCP**：发现异常 → 调用工具处理，保持心跳本身轻。
- **善用 Immediate 参数**：控制聊天后是否立即巡检，适合把多条消息攒一起批量处理。

## 总结

HEARTBEAT.md 的价值不在于省几次提问，而在于把 agent 从"工具"变成"在岗的同事"：它知道该定期看什么、什么时候值得打扰你。工程要点就三条——**条件可判、文件精简、动作克制**。先从一两个低风险巡检跑起，稳定后再加。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/16510b6c9013283b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/c94a20d1d2621449.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/1a5d9ff16ae5a196.png)

