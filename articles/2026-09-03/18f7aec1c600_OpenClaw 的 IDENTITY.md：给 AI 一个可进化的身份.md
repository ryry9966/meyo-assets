---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 35949
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

OpenClaw 的 workspace 里有一组 Markdown 文件共同决定 agent 的行为：`SOUL.md` 管行为准则，`MEMORY.md` 管长期记忆，而 `IDENTITY.md` 管的是最底层的一件事——**它是谁**。文件很短，通常只有十几行：name、creature、vibe、emoji、avatar，每次会话启动时被拼进 system prompt。

听起来很简单，但在长期跑一个跨渠道（Telegram / WhatsApp / WebChat）的 agent 时，这个小文件的工程价值比我最初预期的大得多。

## 问题

早期我把人设直接写死在配置里的一段长 prompt 中，遇到了三个典型问题：

1. **不可版本化**。prompt 改一版就丢一版，出问题时无法回滚，也说不清"上周它为什么表现不一样"。
2. **职责混杂**。"它是谁"、"它怎么做事"、"它经历过什么"三件事搅在一起，改一处容易牵连另一处。
3. **渠道割裂**。同一个 agent 在不同渠道表现不一致，因为各处各写了一份人设。

## 做法

我的现状：把身份收敛到 workspace 的 `IDENTITY.md`，并用 git 管理。

1. **定位 workspace**（默认 `~/.openclaw/workspace`，可在配置中自定义），创建或编辑 `IDENTITY.md`。
2. **只写身份字段**：name / creature / vibe / emoji / avatar。vibe 用两三个短句，其余一行。
3. **划清三个文件的边界**：
   - `IDENTITY.md`：它是谁（慢变量）
   - `SOUL.md`：它怎么做事（中速变量）
   - `MEMORY.md`：它经历过什么（快变量）
4. **git 提交**，commit message 写清"为什么改"，而不是"改了什么"。
5. **按批次迭代**：跑一段时间的会话后回看日志，语气不对就改 vibe 里的一两行，保持小步 diff。

多个 agent 并存时，每个 workspace 独立一份 `IDENTITY.md`，互不干扰。

## 踩坑点

- **写成小说**。我第一版写了四十多行性格描写，token 占用不小，还和 `SOUL.md` 的措辞互相矛盾，行为变得不可预测。现在控制在 10 行以内。
- **改太勤**。一周改三次人设，老会话的记忆里还是旧人格，使用者会觉得"这东西性格分裂"。身份要像人一样慢慢变。
- **字段冲突**。IDENTITY 和 SOUL 都写了语气要求，两处不一致时模型行为漂移。原则：身份只管"是谁"，不碰"怎么做事"。
- **别放事实**。项目信息、日程这类事实属于 memory，放进 identity 会让慢文件被频繁改动，违背它的定位。
- **编辑时机**。会话中途改 `IDENTITY.md` 不一定立即生效，验证改动时记得开新会话。

## 可复用建议

- 把 `IDENTITY.md` 当配置代码对待：短、可 diff、改动走 commit。
- 团队场景做一个模板：固定字段 + 可选自由行，避免每人写法不同。
- 用 git history 做"人格回滚"，效果比想象中好。
- 每月和 memory 清理一起做一次身份回顾，形成固定节奏。
- 保持身份与渠道配置解耦，换平台不用重写人设。

## 总结

`IDENTITY.md` 不是一个炫技功能，它是 agent 里变化最慢、也最该被认真对待的那个文件。所谓"可进化的身份"，核心不是让它随便变，而是把"是谁 / 怎么做事 / 经历了什么"拆开，让身份这个小文件在版本控制下缓慢、可追溯地演化。文件很小，值得 git。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/f14a7b898fb9de5a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/bc681c1c26c4fa57.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/b223423e6f6f858a.png)

