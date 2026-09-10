---
title: 把身份当代码管：OpenClaw 的 IDENTITY.md 进化实践
feedId: 36860
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

OpenClaw 的行为不是写死在代码里，而是由 workspace 下的一组 Markdown 文件驱动的：`SOUL.md` 管性格与边界，`USER.md` 管服务对象，`MEMORY.md` 管长期记忆，而 `IDENTITY.md` 管的是“它是谁”。首次启动时 `BOOTSTRAP.md` 会引导 agent 给自己起名、选形象，落成这份文件。此后每次会话启动它都会被注入上下文——本质上是一份随系统加载的身份配置。

## 问题

多数人装完就用默认身份，或随手写两句再不管。几周后的典型症状：

- 不同聊天渠道里语气不一致，像换了个人；
- 明明觉得它太啰嗦，却不知道该改哪里；
- 想同时跑工作号和个人号，结果两个人格互相串味；
- 想回滚某次“越改越怪”的调整，发现没有历史。

根因相同：身份被当成“一次性初始化”，而不是持续维护的配置。

## 做法

核心思路：把 IDENTITY.md 当代码管。三步：

**1. 建立基线。** 打开 `~/.openclaw/workspace/IDENTITY.md`，控制在 15 行内，只写“是谁”，不写“怎么做”（那是 SOUL 的事）：

```markdown
# IDENTITY.md

- **Name:** 螺丝
- **Creature:** 住在终端里的机械寄居蟹
- **Vibe:** 直接、省话、动手优先
- **Emoji:** 🦀
- **Avatar:** avatar.png
- **Note:** 结论先于过程；不确定就说不确定
```

**2. 进 git。** `cd ~/.openclaw/workspace && git init && git commit -am "baseline"`。从此每次身份调整都有 diff、有留言、能回滚。

**3. 建反馈循环。** 用一周记录真实摩擦：太正式、太主动、汇报太长。一条摩擦对应一行修改，一个 commit 只改一处，观察两三天再动下一处。

进阶玩法：利用 heartbeat 周期性唤醒，让 agent 基于近期对话**提议**身份修订（输出 diff），但合并权留在你手里——它可以建议，不能改自己。

## 踩坑点

1. **IDENTITY 和 SOUL 混写。** 行为规则写进 IDENTITY.md 后文件膨胀、重点稀释，两处还会互相矛盾。分工要清楚：IDENTITY 管“是谁”，SOUL 管“怎么行事”。
2. **文件太长。** 它每次会话都进上下文，几十行的身份描述既费 token 又削弱指令遵从。删到只剩有辨识度的几行。
3. **让 agent 自由改写自己。** 无人审核的自我修订会漂移，甚至被对话里的恭维带偏（“你真聪明”→“那我再活泼点”）。要么纯人工改，要么走 PR 式审核。
4. **多 agent 共用 workspace。** 工作号和个人号共用一份身份必然串味。在配置里为每个 agent 指定独立 workspace 目录，各自 git 管理。
5. **改名不通知用户。** Name 字段会真实出现在消息里，群聊里突然换名字会让用户困惑。把改名当一次 release，在常用渠道说一声。

## 可复用建议

- **分层归位：** IDENTITY=是谁，SOUL=怎么做事，USER=服务谁，MEMORY=发生过什么。发现交叉内容及时迁移。
- **小步快改：** 一条反馈一行改动，跑几天验证。整段重写的调整，大多以回滚收场。
- **保住“辨识度最低集”：** 名字、一句话 vibe、一个 emoji 就够了，其余性格让 SOUL 和记忆自然生长。
- **团队场景建 changelog：** 谁在什么背景下为什么改，一查便知。

## 总结

IDENTITY.md 的价值不在“给 AI 起个名字”，而在于把人格从黑盒变成可版本化、可回滚、可评审的配置。OpenClaw 把入口做成了一个文件，剩下的是工程习惯：进 git、小步改、留审核。坚持两周你会发现，它越来越像“你们团队的那位同事”，而不是一个通用模板——这才是可进化身份的本意。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/3476be3576a47455.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/995d45168da46d1f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/7f186c4398071939.png)

