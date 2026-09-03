---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 35941
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

OpenClaw 的 workspace 里有几个 Markdown 文件承担着“轻量操作系统”的角色：AGENTS.md 管行为约定，USER.md 记录用户画像，而 IDENTITY.md 是其中最短的一个——通常只有几行：agent 的名字、形象（creature）、emoji 和头像路径。

它的特殊之处不在于内容，而在于机制：这份文件在启动时被注入 system prompt，运行过程中又允许 agent 自己读写。换句话说，OpenClaw 把“我是谁”从写死的系统提示里抽了出来，变成一份 agent 可操作的一等数据。

## 问题

没有身份文件的 agent 通常有两个症状：

1. **跨会话不一致**。每次冷启动的行为风格取决于上下文和运气，用户要反复“调教”。
2. **多实例不可辨识**。几个 agent 同时出现在日志、通知、消息路由里时，分不清谁是谁。

把身份硬编码进 system prompt 能解决一部分问题，但代价是：改动需要改配置并重启，agent 无法根据协作反馈自我调整，也没有任何演化历史可言。

## 做法

一套可落地的步骤：

1. 在 workspace 根目录建 IDENTITY.md（默认在用户目录下的 `.openclaw/workspace/`，以实际安装为准），最小模板：

```markdown
# IDENTITY.md

- Name: Kestrel
- Creature: 隼（效率向、少废话）
- Emoji: 🦅
- Avatar: avatars/kestrel.png

一个负责代码审查与 CI 修复的 agent。语气直接，先给结论再给依据。
```

2. 首次启动后，主动让 agent 读一遍并复述它对自己的理解，确认“文件 → system prompt → agent 认知”这条注入链路真的生效。
3. 允许 agent 在明确节点自改：职责变化、与其他实例撞名、收到明确的协作反馈。同时在行为约定里写清“身份稳定优先，修改需说明理由”。
4. workspace 纳入 git。agent 每次自我修订都会留下 diff，身份演化变成可追溯、可回滚的历史。

## 踩坑点

- **别让 agent 随手重写。** 高频自改会放大人格漂移，和初衷背道而驰。我把它限制进低频的心跳任务里做“季度自查”。
- **身份文件不是行为规范。** 见过 agent 把十几条“如何回复消息”的规则塞进去，文件越写越厚。行为归 AGENTS.md，身份保持几行。
- **避免双源。** 同一份描述既写在 IDENTITY.md 又写在 system prompt，两边一旦不同步，行为就会飘忽。保持单一事实来源。
- **emoji 和头像不是装饰。** 它们是多 agent 场景下消息前缀、通知标题里最快的视觉路由标识，删掉后翻日志明显变慢。
- **团队或对外场景慎用过于拟人化的名字**，先想清楚这个 agent 会出现在谁的会话里。

## 可复用建议

- IDENTITY.md 和 USER.md 分开管：“我是谁”与“你是谁”混写是最常见的早期错误。
- 多实例部署时每个 workspace 独立一份身份文件，绝不共享。
- 定期 `git log -p IDENTITY.md`，这是观察 agent 自我认知变化最便宜的可观测性手段。
- 新 agent 上线流程加四步：人写初版 → agent 复述确认 → 观察一周 → 开放自改。

## 总结

IDENTITY.md 看起来只是“给 AI 起名字”的趣味功能，但工程价值在于：它把身份变成一份 agent 可读写、可版本化、可演化的数据。几行 Markdown，换来跨会话一致性、多实例可辨识性，以及一条完整的自我演化轨迹。建议每个跑起来超过一周的 OpenClaw 实例，都认真维护这份文件。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/b5a53a308993c19b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/83d9926b8afb48ec.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/f23d0c1fde8c3b1a.png)

