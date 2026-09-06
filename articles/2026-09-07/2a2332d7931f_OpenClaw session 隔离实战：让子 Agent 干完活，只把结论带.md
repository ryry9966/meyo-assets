---
title: OpenClaw session 隔离实战：让子 Agent 干完活，只把结论带回主会话
feedId: 36396
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

OpenClaw 里每个对话都对应一个持久 session：transcript 以 JSONL 形式存在 `~/.openclaw/agents/<agentId>/sessions/` 下，模型每轮都在这个上下文里续写。主会话（比如你日常用的聊天频道绑定的那个 session）是长期资产，里面沉淀着你的偏好、项目约定和多轮上下文。

## 问题

但只要让主 Agent 直接干重活——抓网页、分析上千行日志、批量读代码——工具输出就会原样写进主会话 transcript。后果有三个：

- 上下文窗口被工具输出吃掉，compaction 提前触发，早期的重要约定被压缩丢失；
- 多轮对话变迟钝，每轮都拖着这些噪音；
- 如果频道是共享的，中间产物还构成隐私面。

子 Agent 的 session 隔离就是为这个准备的。

## 做法

**1. 摸清现状。** 用 `openclaw sessions` 列出所有 session，直接 `tail` 主会话的 jsonl，看它现在有多胖。

**2. 派活。** 对主 Agent 说：用 `sessions_spawn` 派子代理去干这件事。关键是任务描述必须自包含——子代理看不到主会话历史，只拿到 spawn 里的那段话。目标、输入路径、允许的工具、超时、返回格式一次写清，比如"只回 10 行内结论 + 关键数据 + 失败原因"。

**3. 回收。** 子代理的所有中间输出只写进它自己的 session 文件，`sessions_spawn` 的工具结果里只有最终总结回到主会话。需要追问时用 `sessions_send` 继续跟那个子会话聊，别把历史拉回来。

**4. 验证。** spawn 前后对比主会话 jsonl 的大小：如果中间输出进了主会话，说明 spawn 没生效，或模型走了直连工具的老路，回去改指令。

## 踩坑点

- **隔离是会话级，不是系统级。** 子代理和主代理共享文件系统、凭证和网络。它删错文件、推错分支，污染的是环境，session 隔离救不了你。危险操作给单独的低权限 agent。
- **总结不是备份。** 主会话只有摘要，子会话可能被清理。重要的中间产物让子代理落盘成文件，别只留在对话里。
- **谨防递归 spawn。** 子代理再派子代理、没人设超时，就会空转烧 token。每个 spawn 都带超时和"必须回传结论"的约束。
- **别手改 jsonl。** session 文件是 append-only 的事实来源，手动编辑容易把索引弄坏；要清理就走命令或删整个文件。

## 可复用建议

把"重 I/O 默认派子代理"变成习惯：抓站、日志分析、批量检索都进子会话，主会话只留决策和多轮交互。给子代理固定一个返回模板（结论 / 数据 / 风险三段），减少主 Agent 二次总结的信息损耗。按职责拆 agent——写代码、跑运维、做检索各一个——session 天然分域。最后，定期备份 `~/.openclaw`，那些 jsonl 是你唯一可信的上下文事实来源。

## 总结

session 隔离的本质是上下文分账：主会话是稀缺资源，子代理是消耗品。让脏活和脏数据死在子会话里，主会话只接收压缩后的结论——这是 OpenClaw 长期对话不崩的基本功。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/1d73b05dfbdf8eee.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/c05f742d1c8c2f6a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/5b13afdac52bedd9.png)

