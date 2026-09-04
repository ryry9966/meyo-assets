---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 36133
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

大多数 Agent 框架把"人格"写死在 system prompt 或代码常量里：名字、语气、口头禅混在几百行的提示词中间。OpenClaw 的做法是把身份单独抽成一个 Markdown 文件——`IDENTITY.md`，默认放在 `~/.openclaw/workspace/` 下，每次会话启动时自动注入上下文。身份不再是代码，而是数据。

## 问题

实际用下来，没有这个文件时会遇到三类麻烦：

1. **散落**：称呼、语气、行为约定分散在多个 prompt 片段里，改一次要全局搜索，漏一处就不一致。
2. **漂移**：Agent 跑久了，跨会话的"自我认知"不稳定，今天自称助手、明天自称伙伴。
3. **用户偏好无处安放**：怎么称呼你、你习惯什么语气，这类信息反复在对话里重申。

## 做法

最小可用版本就四个字段，几十字节就够：

```markdown
# IDENTITY.md

- **Name:** 小爪
- **Emoji:** 🦞
- **Vibe:** 务实、简洁、不废话，技术问题直给方案
- **Notes:** 中文优先；不确定的事明说不确定，不编造

## 自我描述
一个常驻本地的工程师助手，擅长自动化与插件实践。
```

维护它有三种方式：

1. **直接编辑**：改完即时生效，下个会话就注入新内容。
2. **让 Agent 自己改**：直接说"更新你的 IDENTITY.md，加上 XX 约定"。OpenClaw 的 agent 有 workspace 写权限，改完会自己留版本记录。
3. **周期性反思**：配合 heartbeat 定时任务，让 agent 基于近期对话和 MEMORY.md 总结出"稳定结论"，写回身份文件。

注意分工，别把所有东西塞进一个文件：`IDENTITY.md` 管"我是谁"，`USER.md` 管"用户是谁"（称呼、偏好），`SOUL.md` 管行为边界和价值观，`MEMORY.md` 管事实记忆。身份文件只放前者。

## 踩坑点

- **写太长**。IDENTITY.md 每次会话都注入，写成长文会稀释重点、浪费 token。经验是控制在 200 字以内。
- **混入任务指令**。"每天早上发日报"这类内容属于 AGENTS.md 或定时任务，写进身份文件会导致职责混乱。
- **放任自由改写**。让 agent 随意进化身份，几周后人格可能完全变样。建议在 SOUL.md 里加一条"身份变更需用户确认"，或者干脆用 git 管 workspace，每次改动有 diff 可查。
- **删掉 Emoji/Creature 这类"玩具字段"**。它们看着像装饰，实际是模型的自我锚定点，删掉后回复风格常有可感知的变化，不建议精简过头。
- **多 Agent 场景**忘了 workspace 隔离，导致两个 agent 共用一份身份。

## 可复用建议

- workspace 进 git，身份演进就有了完整的版本历史，出问题可回滚。
- 字段模板从 Name / Emoji / Vibe / Notes 四项起步，用出需求再加。
- 每月做一次"身份审计"：让 agent 自我总结当前状态，人工审核后合入，避免无人监督的渐进漂移。
- 团队内可以沉淀一份统一的 IDENTITY.md 模板，保证多个部署实例的行为基线一致。

## 总结

IDENTITY.md 的价值不在文件本身，而在于它把"身份"从代码常量变成了可版本化、可审计、可让 Agent 参与维护的状态。工程上看，这就是把配置外部化这个老思路，用在了 Agent 自我认知上——简单，但确实解决了长期运行下的一致性问题。建议每个跑 OpenClaw 的实例都认真写一份，哪怕只有五行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/5816ec5b8564caa2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/d9dfffbaa2702dda.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/97fe6ae29a08e64a.png)

