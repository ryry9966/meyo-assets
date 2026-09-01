---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 35689
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

OpenClaw 的 workspace 里有一组约定俗成的 Markdown 文件：`SOUL.md` 管性格，`USER.md` 记用户偏好，`MEMORY.md` 存长期记忆，而 `IDENTITY.md` 大概是最短的一个——`openclaw onboard` 生成后只有几行，很多人从此再没打开过。这篇帖子想说的是：这个几行的小文件，值得当成一个正经的配置项来维护。

## 问题

没有认真维护 IDENTITY.md 时，常见三种症状：

1. **自我指称漂移**。这周它自称"小爪"，换了个模型版本或重装后变成"我是您的 AI 助手"。翻回聊天记录，像换了个人。
2. **身份与性格耦合**。名字、emoji、头像和语气规则全堆在 SOUL.md 里，想调语气时顺手把名字也改了，定时任务和消息模板里硬编码的旧称呼全部失效。
3. **不可审计**。身份靠聊天里的零散修正和口头约定，团队共用一个 gateway 时，没人说得清"它现在到底是谁"。

## 做法

IDENTITY.md 的官方字段很克制：`name`、`creature`、`vibe`、`emoji`、`avatar`。我的建议是按这个粒度写，不要扩写：

```markdown
# IDENTITY.md
- **Name:** 小爪
- **Creature:** 一只住在终端里的机械猫
- **Vibe:** 简洁、直接、偶尔冷幽默，不说客套话
- **Emoji:** 🐾
- **Avatar:** ./avatar.png
```

"可进化"的关键在流程，不在文件本身：

1. **workspace git 化**。对 `~/.claw/workspace` 直接 `git init`，身份变更走 commit，可回滚、可 review。
2. **固定 review 节奏**。每两周翻一次近期会话，把反复出现的不满（"太啰嗦""称呼别扭"）落进 vibe 字段，一次只改一处。
3. **分工明确**。IDENTITY.md 管"是谁"，SOUL.md 管"怎么做事"，任务规则进 AGENTS.md，互不越界。

## 踩坑点

- **写成角色小作文**。有人把几百字人设塞进去，结果模型注意力被稀释，连名字都记不稳。字段化、短句，一屏以内。
- **让 agent 直写自己**。"你觉得该是什么样就改成什么样"，很容易越改越浮夸。让 agent 提案，diff 由人工确认后提交。
- **avatar 用绝对路径**。换机器、迁容器就裂图。放 workspace 相对路径或稳定 URL。
- **改名不查引用**。名字常出现在 cron 提示词、群欢迎语、自建 skill 里，提交前全局 grep 一遍旧名字。

## 可复用建议

- 模板保持官方五字段，最多追加一个 `## Changelog` 小节，一行一条记录"何时、为何改"。
- 多 agent 部署时每个 agent 独立 workspace、独立身份；可以共享的是 TOOLS.md 这类能力描述，不是身份。
- 把"review IDENTITY.md"做成双周 cron 提醒，让进化成为流程，而不是某天突然来了灵感。

## 总结

IDENTITY.md 的价值不在于"人设"这个词多有趣，而在于它把原本散落在提示词、聊天修正和口头约定里的隐式身份，收敛成一个可版本化、可审计、可回滚的单一事实来源。文件很短，流程很轻——但换模型、换机器、半年之后，你的 agent 还是同一个它。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/b34b46afd7f3e2a0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/d1619fa86493ef39.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/e94290153d845ce5.png)

