---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 36302
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

OpenClaw 的 workspace 里躺着几个不起眼的 Markdown 文件：AGENTS.md 管行为、SOUL.md 管性格、MEMORY.md 管记忆。其中最小的一个是 IDENTITY.md，通常只有几行，却决定了 agent “是谁”——名字、形象、emoji、头像。很多人装完就用默认值，或者随手改个名了事。这篇帖记录我们在多实例部署中把它当作正式配置来维护的实践。

## 问题

早期没把它当回事，后来接连遇到三个麻烦：

1. **多实例混淆**。同一台机器跑三个 agent（个人助理、值班机器人、CI 通知），会话记录里全是“我”，回溯日志时分不清哪条是谁说的。
2. **人格漂移**。换底座模型或升级版本后，agent 的自称、语气前后不一致。身份信息散落在系统提示词和对话历史里，没有单一事实来源。
3. **不可审计**。某次 agent 突然用了一段奇怪的自我介绍，排查半天发现是上周一次手工编辑的残留——没有版本记录，无法 diff。

## 做法

**1. 固化文件结构。** 把 IDENTITY.md 收敛成固定字段，示例：

```markdown
# IDENTITY.md
- Name: Nova
- Creature: 夜行机械猫头鹰
- Vibe: 冷静、简洁、先给结论
- Emoji: 🦉
- Avatar: assets/nova.png
```

**2. 纳入版本管理。** workspace 整个目录 git 化，身份变更必须走 commit，格式如 `identity: rename Orion -> Nova`。出问题时 `git log -p IDENTITY.md` 直接定位。

**3. 划清边界。** IDENTITY.md 只回答“是谁”，行为约束写 AGENTS.md，价值观写 SOUL.md。字段超过 15 行，基本说明有东西放错了地方。

**4. 允许有限自主演化。** 我们加了个定时任务，让 agent 每周可以提交一份身份修订建议（PR），附上改动理由，人工 review 后才合并。agent 参与自己的身份设计，但决定权始终在人。

**5. 多实例隔离。** 每个实例独立 workspace、独立 IDENTITY.md，用同一份模板初始化，保证字段一致性，也避免串写。

## 踩坑点

- **改名的残留效应。** 把 Orion 改成 Nova 后，旧会话里 agent 仍自称 Orion——MEMORY.md 里存着旧的自我介绍。处理办法：改身份时顺手 grep 一遍记忆文件，或在下次会话开头让 agent 做一次显式的重新自我介绍。
- **行为规则混进来。** 曾把“回复不超过三句”写进 Vibe 字段，结果文件越写越长，模型权重也混乱。规则归 AGENTS.md，这条边界要守住。
- **Avatar 相对路径失效。** 换了个目录启动 gateway 后头像 404，统一改成 workspace 内相对路径，并在启动时加了一次存在性校验。
- **改得太频繁。** 一个月改三次身份，记忆和身份开始互相矛盾。现在约定：变更间隔不低于两周，且必须带理由。

## 可复用建议

- 把 IDENTITY.md 当**配置**而不是文档：小、稳定、可 diff、可回滚。
- 排查 agent 行为异常时，第一件事先 diff 三个身份相关文件（IDENTITY / SOUL / AGENTS），很多时候根因就是最近一次“顺手编辑”。
- 团队场景给身份文件加 review 流程，比事后在群里问“谁改的名字”便宜得多。
- 模板化初始化，新实例 5 分钟内拥有合规身份。

## 总结

IDENTITY.md 小到容易被忽视，但它是 agent 可观测性的一部分：名字是谁定的、什么时候改的、为什么改，都应该有答案。所谓“可进化”，关键不在让 AI 随便改自己，而在**版本化 + 有限自主 + 人工把关**这三件事上。文件只有几行，工程化的价值却实打实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/6e6e2bf62147732f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/00cc79ab23f36409.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/10b00ed8fc7186a5.png)

