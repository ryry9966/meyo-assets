---
title: OpenClaw 的 IDENTITY.md：给 Agent 一份可进化的身份
feedId: 35733
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

用 OpenClaw 跑一个长期在线的个人 Agent 之后，你会发现“它是谁”不只是装饰问题。名字、物种设定、说话风格会影响它在群聊里的自我介绍和整体观感，也会影响你自己的使用心态——一个顶着默认名和默认头像的助手，和一只你自己命名、性格明确的“工程同事”，日常交互体验完全是两回事。

OpenClaw 把这件事做成了一个显式文件：工作区里的 `IDENTITY.md`。

## 问题

默认模板只有几个空字段（Name / Creature / Emoji / Avatar / Vibe），很多人装完从没打开过。由此带来几个实际的工程问题：

1. **身份散落**：你在 AGENTS.md、SOUL.md、系统提示里各写一点“怎么称呼自己”，换模型或重装后就丢了。
2. **不可版本化**：人设改过十版，哪版好用？没有 diff，只有记忆。
3. **跨会话不一致**：上一轮还自称 A，新会话又回到默认，群聊里观感很差。
4. **漂移不可审**：如果让 Agent 直接改自己的人设，没有审查机制，几周后你就不认识它了。

## 做法

`IDENTITY.md` 的定位是“我是谁”，只放事实级字段。建议保持最小结构：

```markdown
- Name: 小螺
- Creature: 机器美西螈（六角恐龙）
- Emoji: 🦎
- Avatar: avatars/luo.png
- Vibe: 话少、动作快，工程师同事而不是客服
```

步骤：

1. 编辑工作区里的 `IDENTITY.md`（默认在 `~/.openclaw/workspace/`，多 Agent 时每个 Agent 有独立 workspace，别共用）。
2. **划清边界**：身份字段进 IDENTITY.md，价值观和行为准则进 SOUL.md，操作规范进 AGENTS.md。三者内容重叠是大多数“性格诡异”的根源。
3. 把 workspace 纳入 git。改身份就是一次 commit，可 diff、可回滚。想让 Agent 参与进化，让它产出 diff 或提 PR，你来合并——而不是给它直接写文件的权限。
4. **生效验证**：新开会话，问它“你是谁、什么风格”，再丢一个群聊场景看自我介绍。注意：正在跑的会话不会热加载身份文件。
5. 用一周，记录别扭的点（太话痨、emoji 太多之类），小步改，一次只动一个字段。

## 踩坑点

- **写太长**。这份文件每轮都会注入上下文，五百字的人设是持续的 token 税。事实字段几行就够。
- **Vibe 写成行为规范**。“回复不超过三句”这类规则属于 SOUL.md/AGENTS.md，放错位置会导致两个文件互相打架。
- **改完不验证**。旧会话仍是旧身份，你以为没生效又改一版，结果叠加出矛盾设定。
- **直接给写权限**。没有 git diff 兜底的自我修改，人设漂移只是时间问题。
- **头像/emoji 有平台差异**。部分渠道同步头像会静默失败，别把这两项当核心功能依赖。

## 可复用建议

- 把“身份即代码”当原则：小 commit、有说明、可回滚。
- 命名的成本值得花：一个好名字比你想象中更影响你对它输出的容忍度。
- 每月做一次“身份回顾”：问它自己是谁，对照文件，不一致就查加载链路。
- 多 Agent 场景用不同的 Creature/Emoji 做视觉区分，显著降低切错窗口的概率。

## 总结

`IDENTITY.md` 是个小文件，价值不在内容多少，而在于 OpenClaw 把“身份”从散落的提示词，变成了一份可版本化、可审查、可进化的资产。几行字段加一个 git 仓库，你的 Agent 就有了稳定的自我认知，以及一条清晰的成长轨迹。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/bd08aa7ddee39284.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/214f81f563649db6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/9f8d1ba4675f7bdf.png)

