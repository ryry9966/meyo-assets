---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 36057
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

OpenClaw 的能力扩展走的是 Skills 这条路：每个 Skill 是一个目录，核心是一个带 frontmatter 的 `SKILL.md`，外加可选的脚本和资源文件。它和 MCP 的分工可以这样理解——MCP 解决“工具怎么接进来”，Skills 解决“指令和流程怎么在合适的时机进入上下文”。

OpenClaw 采用渐进式加载：系统提示词里默认只注入每个 Skill 的 name 和 description 两条元信息，全文只有在模型判断当前任务匹配时才会被读入。这个设计直接决定了写 Skill 的第一原则：**description 不是简介，是触发条件。**

## 问题

实际用下来最常见的三个毛病：

1. **上下文膨胀。** 把所有操作手册都塞进系统提示词，token 消耗大，注意力被稀释，回答反而变差。
2. **该触发的不触发。** 模型不知道某个流程存在，或者 description 写得太含糊，匹配不上任务。
3. **不该触发的乱触发。** description 写得太宽，闲聊也把几十行操作指令拉进来。

## 做法

以一个“发布前检查”的 Skill 为例：

```markdown
---
name: release-check
description: 执行版本发布前检查时使用。触发词：发版、上线前检查、release checklist。
---

# 发布检查流程
1. 运行 `scripts/lint.sh`，确认无告警
2. 核对 CHANGELOG 最新条目与版本号一致
3. 任何一项失败则中止并输出原因
```

步骤：

1. 在 `~/.openclaw/skills/` 下建目录（workspace 级 skills 目录也可以）。
2. 写 `SKILL.md`，frontmatter 必须有 name 和 description，正文只放步骤。
3. 确定性操作写成脚本放在同目录，`SKILL.md` 里只写“何时调用哪个脚本”，不要用自然语言复述脚本逻辑。
4. 用一条明确该触发的话术实测，通过注入日志确认全文确实被加载。
5. 用不上的内置 Skill 直接在配置里关掉，减少元信息占用。

## 踩坑点

- **description 太泛或太窄。** “处理各种发布相关的事情”会被无关对话触发；“需要查询 PR 状态时”又接不到发版任务。正确写法是列出任务特征和触发词。
- **正文超长还全塞 `SKILL.md`。** 拆成多个文件，主文件只留索引和入口，模型按需再读。
- **依赖缺失。** Skill 依赖某个 CLI（比如 jq）但机器上没装，触发即挂。在正文开头写明前置条件，先检测再执行。
- **密钥硬编码。** Skill 目录建议进 git，密钥一律走环境变量。
- **改了不生效。** 确认改的是当前 agent 实际加载的那份——workspace 级会覆盖全局级。

## 可复用建议

- 一个 Skill 只干一件事，克制比大而全好。
- description 用“什么时候用”的句式写，不要写成功能宣传语。
- 能用脚本的不用散文：确定性步骤脚本化，`SKILL.md` 只负责编排和触发。
- 团队内沉淀成 git 仓库，Skill 变更走 PR 评审，和代码同权重。
- 定期清理：三个月没触发过的 Skill，要么合并，要么下线。

## 总结

Skills 的价值不在“多”，而在“准时”——对的能力在对的时机进入上下文。把 description 当触发器写、把确定性逻辑下沉到脚本、把不常用的关掉，这三件事做到位，助手的表现会明显稳定。我们现在把 Skill 当“可评审的运维文档”来管理，比堆一堆没人看的 wiki 有效得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/8416db31d5365384.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/ee6ddd07a88b189a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/cc2c23ed8c4d1d07.png)

