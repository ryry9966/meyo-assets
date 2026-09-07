---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 36541
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

OpenClaw 的 workspace 里有几个约定俗成的 markdown 文件：`AGENTS.md` 管工作方式，`SOUL.md` 管行为倾向，`MEMORY.md` 管记忆，而 `IDENTITY.md` 管的是最基础的一件事——这个 agent 是谁。它会在每次会话组装上下文时被读入，通常只有十几行。

很多人第一次部署时会跳过这个文件，直接堆任务规则。跑一段时间后发现问题慢慢浮出来。

## 问题

没有显式身份文件时，agent 的“人设”完全由模型默认值决定，具体表现为：

1. **跨会话漂移**：周一冷静简洁，周五热情话痨，同一个 agent 像换了个壳；
2. **身份散落**：性格设定混在 system prompt、任务规则和对话记忆里，改一处动全身，且不可审计；
3. **多实例冲突**：两个实例共用 workspace 时互相污染，输出里经常串台词；
4. **演进无据**：想调整人格只能凭感觉改 prompt，没有 diff、没有回滚。

## 做法

**1. 补一个最小文件。** 位置在 workspace 根目录（默认 `~/.openclaw/workspace/`，以你的安装配置为准）：

```markdown
# IDENTITY.md

- **Name:** Aria
- **Creature:** 机械猫头鹰
- **Emoji:** 🦉

## Tone
冷静、简短、先给结论再给依据。

## Boundaries
不做超出授权范围的系统改动；不确定时先问，不猜。
```

**2. 版本化。** workspace 整体进 git，身份变更走 commit，改动可 diff、可回滚。

**3. 严格分层。** IDENTITY.md 只回答“我是谁”；价值观进 SOUL.md，工作流程进 AGENTS.md，用户偏好进 USER.md。别往身份文件里塞任务逻辑。

**4. 定期进化。** 每周或每次大改动后，抽查会话日志：语气跑偏了改 Tone，边界失守了改 Boundaries，每次只动一两行，小步提交。

## 踩坑点

1. **写太长。** 这个文件每次会话都会注入，几百行会吃掉 token 预算，还稀释真正的任务指令。建议控制在 20–40 行。
2. **混入任务规则。** “收到 bug 报告先复现再修”是 AGENTS.md 的事，写进 IDENTITY.md 只会让身份文件越来越像垃圾抽屉。
3. **期待即时生效。** 会话中途改文件，当前上下文里的旧身份不会消失，新会话才稳定生效——具体注入时机以你用的版本实测为准。
4. **多实例共用 workspace。** 每个 agent 独立 workspace，身份文件各自维护，否则互相覆盖是迟早的事。
5. **把它当安全护栏。** IDENTITY.md 影响的是呈现层，真正约束行为要靠 SOUL.md 配合权限与工具白名单，别指望一段自我介绍能挡住越权操作。

## 可复用建议

- **身份当代码管理**：小步提交、PR review、留 changelog，和改 prompt 同等严肃；
- **从模板起步**，哪怕只有名字和一个 emoji，也不要留空文件；
- **建立回归验证**：定期抽十条历史回复，对照 Tone 和 Boundaries 检查符合度，作为下次修订依据；
- **团队场景**给身份变更定 review 流程，避免某人顺手改了整个 assistant 的性格。

## 总结

IDENTITY.md 的价值不在文件本身，而在于它把“AI 是谁”从隐式运行时状态，变成了显式、可版本化、可审计的资产。三十行以内的投入，换来的是跨会话的一致性和一条可持续演进的身份曲线。如果你的 workspace 里还没有这个文件，建议今天就补一个最小版本，跑两周，再根据实际偏差回来调——身份和代码一样，是迭代出来的，不是一次写成的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/223f2db8cda95ed9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/dcedb7fbd823d174.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/8770e9f6dfb93461.png)

