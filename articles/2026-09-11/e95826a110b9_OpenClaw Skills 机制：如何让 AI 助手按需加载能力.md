---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 36960
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

OpenClaw 的 agent 跑久了都会遇到同一个矛盾：能力越多，system prompt 越长。早期做法是把所有工具说明、工作流、领域知识全塞进系统提示或 `AGENTS.md`，结果上下文里大部分内容在具体会话中根本用不上——token 开销固定，模型注意力还被无关指令稀释。

Skills 是对这个问题比较工程化的回答，核心思路是**渐进式披露**：平时只驻留一行元数据，任务匹配时才加载完整说明。

## 问题拆开看

1. **上下文膨胀**：十几个常驻说明文档，每个会话都带着全量开销；
2. **触发混乱**：模型分不清哪段指令在什么场景下生效；
3. **复用困难**：同一套流程在多个 workspace 复制粘贴，改一处漏三处。

## Skills 怎么工作

加载分三层：

- **第一层**：启动时只读取每个 skill 的 `name` + `description`，几十 token；
- **第二层**：任务匹配某 skill 的 description 时，才读入 `SKILL.md` 正文；
- **第三层**：正文里引用的 `references/*.md`、`scripts/*`，用到才加载或执行。

### 实操步骤

1. 建目录 `~/.openclaw/skills/weekly-report/`（或 workspace 级 `.openclaw/skills/`）；
2. 写 `SKILL.md`，frontmatter 里的 description 是触发关键，要写成"什么时候用我"：

```yaml
---
name: weekly-report
description: 当用户要求生成周报、汇总本周提交与待办时使用。不适用于日报或会议纪要。
---
```

3. 正文只写流程性指令：输入、步骤、输出格式、边界情况。长参考资料拆到 `references/`，可自动化的步骤写成 `scripts/*.sh` 并 `chmod +x`；
4. 重启 gateway 或热加载后验证：问一句"你现在有哪些技能"，或查 debug 日志确认注入列表。

### 与 MCP 的关系

Skill 注入的是"怎么做的知识"，实际执行仍依赖 MCP 工具或本机命令。典型分工：MCP 提供 API 调用能力，Skill 描述调用时序和容错策略。两者是互补，不是替代。

## 踩坑点

- **description 写成名词短语**（"周报工具"）→ 永远不触发。必须写成触发条件句式；
- **description 太宽泛**（"帮助你完成任何任务"）→ 常驻加载，等于白做；
- **SKILL.md 几百行不分层** → 触发即爆上下文，细节一律下沉到 `references/`；
- **脚本路径写死绝对路径、没给执行权限** → 换台机器就挂，用相对路径；
- **全局级与 workspace 级同名冲突**，注意加载优先级，调试时先确认实际生效的是哪份；
- **误以为 skill 能新增执行能力**——不能。工具靠 MCP/插件，skill 只是"说明书"。

## 可复用建议

- 一个 skill 只做一件事，宁可拆小再组合；
- description 按"触发场景 + 覆盖范围 + 排除项"三段写；
- 脚本保持确定性（固定输入输出），把判断留给模型；
- skills 目录进 git，团队维护一个共享仓库，新人克隆即用；
- 定期审计日志：从未触发的 skill，大概率是 description 写偏了。

## 总结

Skills 的本质不是给 agent 新增能力，而是**重新组织上下文**：让模型在正确的时刻读到正确的说明书。我们实践下来，常驻 prompt 砍掉约七成，长任务的稳定性反而更好——干扰信息少了，模型执行既定流程的准确率自然上去。建议从一两个高频流程开始试点，跑顺了再逐步铺开。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/93e95a585d4ca321.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/5efd2b633ffaebc1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/c256071067a89490.png)

