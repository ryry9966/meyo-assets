---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 36407
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景：上下文是稀缺资源

跑 Agent 时间长了都会遇到同一个问题：能力越加越多，提示词越来越长。早期我的做法很粗暴——把所有工具说明、操作规范、API 约定全塞进 system prompt。三五个工具时还行，等堆到几十项，上下文直接膨胀，模型的注意力被无关内容稀释，该遵守的规范反而执行不到位。

OpenClaw 的 Skills 机制就是针对这个问题的解法，核心思路是**渐进式披露（progressive disclosure）**：不把能力全量注入，而是按需加载。

## 机制拆解：三层加载结构

Skills 的加载分三层：

1. **启动时**：只把每个 skill 的 `name` 和 `description`（元数据）注入上下文，单个 skill 只占几十个 token；
2. **触发时**：模型判断当前任务与某个 skill 相关，才去读完整的 `SKILL.md` 正文；
3. **深入时**：正文里引用的 `references/*.md` 细节文档、`scripts/` 下的脚本，只在真正需要时才被读取或执行。

也就是说，你装了三十个 skill，常驻上下文的可能只有一两千 token。

## 做法：写一个能被正确触发的 Skill

以我最近写的「周报生成」skill 为例：

1. 在 `~/.openclaw/skills/` 下建目录，比如 `weekly-report/`；
2. 写 `SKILL.md`，frontmatter 里 `name` 用短横线小写，`description` 最关键——它是模型路由的依据；
3. 正文按固定结构写：**何时使用、前置条件、操作步骤、约束与边界**；
4. 详细格式规范拆到 `references/format.md`，可复用的统计脚本放 `scripts/count.py`；
5. 重启会话后验证加载链路。

description 我最终写成："Use when the user asks to generate a weekly report, summarize git commits of the current week, or mentions 周报."——明确触发条件，而不是"用于生成周报"这种模糊描述。

## 踩坑点

- **description 写得像目录名**：模型永远不会触发。它是给模型看的路由信号，不是给人看的简介；
- **把所有内容塞进 SKILL.md 正文**：等于换了个地方污染上下文，渐进式披露就失效了，细节务必外置；
- **以为脚本会自动执行**：skill 里的脚本仍是模型通过 exec/bash 工具调用的，路径建议写绝对路径，或在正文里写明工作目录；
- **与内置 skill 重名**：会出现覆盖或触发混乱，命名前加自己的前缀更稳妥；
- **改完不重启**：测试跑的永远是旧版本，白调半天。

## 可复用建议

- description 一律从模型视角写触发条件，句式用 "Use when..."；
- `SKILL.md` 正文控制在几百行内，展开内容一律走 `references/`；
- skills 目录纳入 git 管理，改动可回溯、可共享给团队；
- 排查触发问题时，让模型复述它实际读了哪些文件，快速定位断在哪一层。

## 总结

Skills 机制本质上是上下文的经济学：把 token 花在当前任务真正需要的地方。它不炫技，就是「元数据常驻、正文按需、细节再按需」的三层结构。写好一个 skill 的门槛不在格式，而在你能不能站在模型的视角，把触发条件说清楚。建议从替换自己 system prompt 里最臃肿的一段开始，收益立竿见影。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/84a755f29b71aadf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/31115b1228a063ec.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/d605a8c3049a4179.png)

