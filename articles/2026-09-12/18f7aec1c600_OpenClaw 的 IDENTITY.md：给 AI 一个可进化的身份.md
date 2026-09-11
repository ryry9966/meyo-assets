---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 37139
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

OpenClaw 的配置哲学是「workspace 即身份」：agent 是谁、怎么说话、记得什么，全靠工作目录里的一组 Markdown 文件，而不是散落在代码里的 system prompt。其中 `SOUL.md`（性格与价值观）和 `USER.md`（对用户的记忆）讨论得比较多，`IDENTITY.md` 反而常被当成装饰——实际上它每次 bootstrap 都会被读入上下文，里面的 Name、Creature、Vibe、Emoji 会直接进入对话表现。

## 问题

我早期把 agent 身份写死在启动脚本里，遇到三个麻烦：

1. **不可迁移**：换机器、换模型，身份就丢了；
2. **不可审计**：改过几次名字、为什么改，完全无迹可查；
3. **不可演进**：想调整语气或自我定位，只能翻脚本改字符串。

身份应该是一个有版本的文件，而不是一段埋在代码里的硬编码。

## 做法

**第一步，手写初版。** 初始化 workspace 后，按默认字段结构填好 IDENTITY.md：Name、Creature、Vibe、Emoji，外加三行以内的 notes。第一版由人写，别让 agent 自己编。

**第二步，划清边界。** IDENTITY.md 只放「稳定的自我描述」；行为风格、价值观进 `SOUL.md`；对用户和事实的记忆进 memory 体系。三者不重叠，这是后面不出乱子的前提。

**第三步，开放自编辑，但立规矩。** 在 `AGENTS.md` 里写清楚：agent 可以提议修改 IDENTITY.md，但每次只改一处，且必须先在 daily note 里记录动机。改名字、改 creature 这类重大变更标记为「需人工确认」。

**第四步，git 化。** workspace 整个进 git，agent 的每次身份修改对应一个 commit，message 写原因。一个月后你就有了一条可回溯的身份变更史，`git log` 就是 changelog。

**第五步，低频复盘。** 用 heartbeat 或定时任务，每月让 agent 回看自己的 IDENTITY.md，提出「是否需要修订、为什么」。人审批后合并。频率别高于月级，身份不是日志。

## 踩坑点

- **身份漂移**：不设约束就放开写权限，几周内 vibe 会变来变去，连名字都不稳定。规则必须显式写进 AGENTS.md。
- **文件膨胀**：notes 区慢慢把 SOUL.md 的职责吸了过来，最后变成大杂烩。我的经验是硬性控制在 30 行以内，超了就拆出去。
- **注入风险**：IDENTITY.md 会被拼进 system prompt，任何能写 workspace 的工具（MCP server、浏览器下载、脚本）都可能污染它。我在 git 上加了 pre-commit 校验，字段缺失或超长直接拒绝提交。
- **多 agent 共仓库**：两个 agent 同时自编辑会冲突。建议一人一目录或一人一仓库，别省这个事。

## 可复用建议

- 把 IDENTITY.md 当「产品里的关于页」：短、稳定、有版本，而不是流水账。
- 身份进化走 PR 流程：agent 开分支提修改，人 review 后 merge。成本很低，审计性很强。
- 新 agent 冷启动时，先用固定身份跑一两周任务，再允许它提第一版修订——先干活，再认识自己。
- 字段最少化：Name、Creature、Emoji、Vibe 加三条 notes 足够，想写的其他内容多半属于 SOUL.md 或 memory。

## 总结

IDENTITY.md 的价值不在于让 agent「看起来有个性」，而在于把「我是谁」从散落的 prompt 片段，收敛成一个可版本化、可审计、可演进的文件。配上 git 和几条克制的编辑规则，身份进化就从玄学变成了一件普通的工程活。它不需要多精致，只需要像代码一样被认真对待。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/b549d078e4a76f0f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/b6b43dfcd9598173.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/befe69d37e107b79.png)

