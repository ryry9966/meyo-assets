---
title: OpenClaw Session 隔离实践：让子 Agent 干脏活，不污染主会话
feedId: 36935
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

在 OpenClaw 里，主会话（main session）承担的是"控制面"角色：它和用户长期对话，携带完整上下文、偏好和记忆。子 Agent 则是"数据面"：跑检索、批量处理、代码验证这类脏活。两者如果共写一份 transcript，主会话很快会被无关的工具输出灌满。

## 问题

实际用下来，污染主要有三条路径：

1. **上下文污染**：子 Agent 的中间步骤、原始抓取结果直接进主会话，几千 token 的噪声挤占有效上下文，模型开始"分心"，回复质量肉眼可见地下滑。
2. **状态污染**：子 Agent 在主 workspace 里改文件、动 git、写全局 memory，主会话后续的工具调用踩到意料之外的现场。
3. **行为污染**：主会话 transcript 里留下大量子 Agent 的工具调用记录，模型会被带偏，后续推理时模仿子 Agent 的行为模式。

## 做法

我们的落地步骤：

1. **独立 session 派生**。子 Agent 用新的 session id 启动，不继承主会话历史，只传入一段自包含的任务简报（目标、约束、返回格式）。这是隔离的第一道墙。
2. **返回值契约**。子 Agent 结束时只回一份结构化摘要，限定字段和长度（比如 JSON、不超过 500 token）。原始中间产物落在它自己的 workspace，主会话按需再取。
3. **workspace 隔离**。每个子 Agent 分配独立 scratch 目录，工作目录指向它，不与主 workspace 共享写权限。需要交付的文件由主会话显式校验后合并。
4. **工具范围收敛**。子 Agent 默认只挂只读工具；写操作走沙箱（Docker/容器模式），出不了边界。
5. **session 清理**。子 Agent session 文件单独归档，定期清理，避免 sessions 目录无限膨胀。

## 踩坑点

- 图省事把完整 transcript 传给子 Agent 当"背景"，隔离直接失效，token 翻倍。
- 子 Agent 忠实地把几千行原始输出塞进返回值，主会话照样被灌爆——契约里的长度限制要真的生效。
- 多个并发子 Agent 写同一个文件，出现覆盖和竞争。结论：写操作要么串行，要么一人一份。
- 共享的 memory / 全局配置文件被子 Agent 改写，主会话的"记忆"悄悄变了样。memory 写入权限要单独收。
- 忘了清理，一个月后 sessions 目录几个 GB，排查问题时根本翻不动。

## 可复用建议

- 记住一句话：**主会话是控制平面，子 Agent 是数据平面**，数据平面不直接写控制平面。
- 给每类任务定义固定的返回 schema，摘要先行，细节留文件引用路径。
- 工具白名单按子 Agent 角色配置，而不是全局一套。
- 每周看一眼 sessions 目录的大小和结构，异常膨胀往往先于质量劣化出现。

## 总结

Session 隔离的本质不是"多开几个会话"，而是控制上下文的写入权。让子 Agent 在自己的边界里干活，只把一份干净的结论交回主会话，长对话的稳定性和成本都会明显改善。这套做法我们跑了两三个月，主会话"越聊越笨"的问题基本消失了。欢迎在评论区交流你们的隔离策略。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/6c5408572fff4f4d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/8ab5b0f9d7a24165.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/898df50ee20fdfe1.png)

