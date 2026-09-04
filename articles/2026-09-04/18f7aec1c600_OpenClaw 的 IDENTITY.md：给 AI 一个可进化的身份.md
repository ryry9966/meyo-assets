---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 36020
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

OpenClaw 里一个 agent 的行为，不只由模型决定。workspace 目录下的几个 Markdown 文件——`SOUL.md`、`IDENTITY.md`、`USER.md`、`AGENTS.md`——会在会话组装时拼进系统提示。其中 `IDENTITY.md` 通常最短，只有几行：名字、物种/形态、emoji、头像。很多人初始化完就再没看过它，但它其实是维持行为一致性成本最低的一个杠杆。

## 问题

没有认真维护身份文件时，常见的症状：

- **多实例混跑**：个人助手和项目机器人共用一个 workspace，自我介绍混乱，语气互相污染；
- **人格漂移**：长会话跑偏，每次重启都像换了个"人"；
- **身份散落**：描述分散在网关配置、提示词模板、聊天记录里，改一次要翻三个地方，无法 diff，无法回滚。

## 做法

1. **定位文件**：默认 workspace 在 `~/.openclaw/workspace/`，没有 `IDENTITY.md` 就新建一个。
2. **最小模板**，克制为主：

```markdown
# IDENTITY.md
- Name: 变量猫
- Creature: 一只住在终端里的机械猫
- Vibe: 话少，直接，先给结论
- Emoji: 🐱
- Avatar: assets/avatar.png
```

3. **让改动生效**：身份在会话启动时注入，改完要开新会话才能看到；正在跑的旧会话不会热更新。
4. **验证**：直接问 agent"你是谁、你不会做什么"，或查看系统提示的拼装结果，确认字段没被覆盖。
5. **版本化**：整个 workspace 纳入 git。每次改身份单独一个 commit，写清动机（如"语气太啰嗦，收敛"），回滚就是一条 `git revert`。

## 踩坑点

- **把任务指令塞进 IDENTITY.md**。它只该回答"我是谁"；"怎么做事"放 `AGENTS.md`，"底线"放 `SOUL.md`。职责混了之后排查行为问题会非常痛苦。
- **文件越写越长**。它每个会话都占上下文，超过三十行左右收益递减，还会稀释重点。
- **与 SOUL.md 写出矛盾描述**，模型会在两种人格之间摇摆。
- **改名/换头像但不清理旧会话**，观感上就是"人格闪断"。
- **多个 agent 复用同一 workspace 却期待不同身份**。身份文件按 workspace 注入，要么分目录，要么走多 workspace 配置。

## 可复用建议

- 把身份当代码管理：走 PR 评审、记 changelog，改身份和改配置同级对待。
- 团队场景维护一个 identity 模板仓库，新 agent 起步复制一份再微调。
- 每次只改一个变量（比如先只动语气描述），观察一两天再叠加，否则无法归因。
- 定期（比如每两周）让 agent 自述一次身份，把回答 diff 一下，漂移能早发现。

## 总结

`IDENTITY.md` 的价值不在于"写得多漂亮"，而在于它是一个**可 diff、可回滚、可评审的单点**。几行文字，换来的是跨会话的稳定人格，以及一套能随使用不断微调的身份演进流程。如果你还没做，建议今天就把 workspace 里这个文件纳入版本管理。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/2f1360a564d75643.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/e7ca4502dae3fae7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/39f285222cd5bf0a.png)

