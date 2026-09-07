---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 36416
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

OpenClaw 的 workspace 里有一组以 Markdown 承载的"记忆文件"：`SOUL.md` 管行为风格与边界，`USER.md` 记录用户偏好，而 `IDENTITY.md` 负责回答最基础的问题——这个 agent 是谁。每次会话启动，这些文件会被注入系统上下文，agent 因此在跨会话、跨设备之间"记得"自己。

机制很小，但很多人要么不写，要么写崩。

## 问题：没有身份文件的三种典型症状

1. **人格漂移**。今天自称"通用助手"，明天换个名字，长期用户能明显感觉到"不是同一个 agent"。
2. **身份散落各处**。名字写在系统提示里，口头禅在插件配置里，头像在别处。改一处漏三处，最后没人知道权威定义在哪。
3. **身份与用户信息互相污染**。把"主人是谁"写进 agent 自己的身份里，多用户或迁移部署时立刻出问题。

## 做法

在 workspace 根目录建 `IDENTITY.md`，核心原则：只回答"它是谁"，不回答"它该怎么做"。

```markdown
# IDENTITY.md
- Name: Claw
- Creature: 一只务实的机械蟹
- Emoji: 🦀
- Description: 工程向助手；先复现再修复，不确定就明说
- Behavior: 详见 SOUL.md
```

几个执行要点：

- **控制在 10–20 行以内**。它会被注入每一次会话，每行都是常驻 token 开销。
- **把身份进化当 git 问题**。改动走 commit，重要转向在 message 里写清原因，随时可回滚、可回溯。
- **与 USER.md 严格分层**。IDENTITY.md 低频变化，USER.md 随用户变化，两者混写是多用户部署的常见事故源。
- **多 agent 部署时一个 workspace 一份**，靠 name + emoji 快速区分。

## 踩坑点

- **写成大而全的规则手册**。有人把全部行为规则塞进 IDENTITY.md，结果与 SOUL.md 大量重复，上下文膨胀，行为反而摇摆。身份文件答"是谁"，规则归 SOUL.md。
- **留空字段占位**。没想好的字段宁可不写，空占位同样会被注入上下文，等于注入噪音。
- **一次改太多**。身份突变会让长期对话里的 agent 前后不一致，小步提交比大版本重写安全得多。
- **只在配新环境时想起来写**。身份文件需要定期 review，否则它会退化成一份过时的初始配置。

## 可复用建议

- 把 IDENTITY.md 当**配置而非文案**：可 diff、可 review、可回滚，纳入版本管理是底线。
- 用三段式模板：**Who**（名字/形象/emoji）→ **Positioning**（一句话定位）→ **Boundary**（一行引用 SOUL.md）。
- **进化节奏与版本对齐**：每月或每个大版本后 review 一次，改动必有 commit。
- 团队协作时，身份变更走和代码一样的 review 流程——身份被改过的 agent，行为确实会变。

## 总结

IDENTITY.md 的价值不在"人设"这个词，而在把身份变成一份可版本化、可演进、可审计的配置。写短、分层、留 git 记录，身份就能随使用慢慢长成该有的样子，而不是停留在第一天的一次性设定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/1f9292e0648a7e89.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/6d8cc288d58bbd9e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/abad15f42ca93bbd.png)

