---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 35926
source: 综合讨论
publishedAt: 2026-09-03
---

# OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份

## 背景

OpenClaw 的 workspace 本质上是一组 Markdown 文件：`AGENTS.md` 管操作规范，`SOUL.md` 管性格与价值观，`MEMORY.md` 管长期记忆。其中 `IDENTITY.md` 是最容易被当成“装饰”跳过的一个——它只回答几个很短的问题：我叫什么、我是什么形象、我的 emoji、我的头像。

但用下来会发现，它其实是 agent 自我一致性的锚点。

## 问题

不认真维护身份文件时，常见这些现象：

- agent 的自称随模型默认值漂移，跨会话、跨设备不统一；
- 想改名字或形象，只能翻系统提示词和配置，改一次重启一次；
- 身份相关内容散落在各处提示词里，没有任何历史，改坏了没法回滚；
- 长期记忆里引用的还是旧名字，越用越乱。

身份不是“好玩的名字”。它影响 agent 怎么自称、怎么在消息里被辨识、怎么在记忆和对话中被引用。

## 做法

1. **找到或新建 `IDENTITY.md`**。默认字段就五个：Name / Creature / Vibe / Emoji / Avatar。
2. **填具体值，不要留占位符**。默认模板里是 `choose your own`，不替换的话 agent 会把占位符当真。
3. **配头像**：把图片放进 workspace（如 `avatars/xxx.png`），Avatar 字段写相对路径，支持的聊天渠道会同步使用。
4. **纳入 git**：整个 workspace 建议 `git init`，身份变更走 diff，看得见、可回滚。
5. **加一个进化机制**：每月或每个里程碑后，让 agent 自己读一遍 `IDENTITY.md`，提出修订建议（“这一个月的协作里，哪些特质值得写进 vibe”）。你 review diff 后合入。身份就从“安装那天定死的东西”变成有 commit 历史的活文件。

## 踩坑点

- **和 SOUL.md 分工要清楚**：IDENTITY 管“是什么、叫什么”，SOUL 管“信什么、怎么说话”。把说话风格塞进 IDENTITY，两边都会乱。
- **注入范围**：IDENTITY.md 这类引导文件默认在主会话注入，群组会话行为以你所用版本的文档为准。群里发现它“不认识自己”，先查注入配置，别急着改文件。
- **字段要短**。写几百字的小作文会稀释 system prompt，且 agent 未必遵守，一行一个字段足够。
- **改完要开新会话**，老会话上下文里还是旧身份，别误判为没生效。
- **emoji 会出现在消息前缀**，选一个在通知列表里辨识度高的，太冷门的图标在手机上看不清。

## 可复用建议

- 五行字段起步，先跑起来再迭代，别一上来设计复杂的“人格系统”；
- 身份变更哪怕一个人用，也走 diff → review → commit，回滚成本低；
- 与 `USER.md` 成对维护：一个写“我是谁”，一个写“你是谁”，都保持极短；
- 敏感信息和长篇价值观不要放这里，那是 SOUL.md / AGENTS.md 的职责。

## 总结

IDENTITY.md 的价值不在趣味，而在工程：把 agent 的自我呈现从散落的提示词，收敛成一个可版本化、可 review、可回滚的文件。配合 git 和定期修订，身份可以随实际使用一起进化，而不是停在第一天。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/f81d49a3bfb412c0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/ac21fbfaa492d62b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/ff34d66fb739f786.png)

