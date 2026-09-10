---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 36942
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

OpenClaw 的 agent 默认能读文件、跑命令、发消息、操作浏览器。但"能力多"和"上下文里塞满说明书"是两回事。如果把所有操作指南全文注入 system prompt，token 开销大，模型注意力也会被稀释。Skills 采用的是渐进式披露（progressive disclosure）：会话开始时，常驻上下文的只有技能名和一句话描述；当模型判断某个技能与当前任务相关，才通过 read 工具把 SKILL.md 正文读进来，再按里面的步骤执行。

## 问题

我早期的做法是把所有 SOP 直接写进 AGENTS.md，prompt 膨胀到几万 token，日常对话里 90% 的指令根本用不上；另一个极端是什么都不写，靠模型自由发挥，行为不稳定。Skills 是中间态：索引常驻，正文按需，成本随实际使用线性增长。

## 做法

1. 在 `~/.openclaw/skills/` 下建目录，放一个 `SKILL.md`。
2. frontmatter 写 `name` 和 `description`。**description 是唯一常驻上下文的部分，本质是"触发条件"**。
3. 正文写操作步骤，可引用同目录的脚本、模板。确定性逻辑放脚本，SKILL.md 只写"什么时候调、怎么判断结果"。
4. 重启 gateway（部分版本支持热加载），用 `/skills` 确认索引里有它。
5. 测试触发：说一句应该命中的话，确认 agent 是先读 SKILL.md 再行动，而不是凭空猜。

```markdown
---
name: invoice-reconcile
description: 当用户要求对账、核对月度账单或导出台账汇总时使用
---
```

## 踩坑点

- **description 写成功能介绍而非触发条件**。"这是一个对账工具"模型不知道何时用；"当用户要求对账时使用"命中率明显更高。
- **技能数量失控**。二三十个之后，光索引就开始占上下文，还会互相误触发。定期 `/skills` 审一遍，禁用不用的。
- **正文太长**。几千字的说明书模型读得勉强，执行容易丢步骤。超过一屏就拆分，或把细节下沉到脚本。
- **和 MCP 混淆**。Skill 是指令不是工具：它教模型如何用已有工具（包括 MCP 工具）完成一件事，不要拿它替代 MCP server。
- **依赖没声明**。脚本依赖某个 CLI 或环境变量时，用 `requires` 元数据声明，让 preflight 检查提前暴露缺失，而不是运行时才炸。

## 可复用建议

- 一个技能只干一件事，description 用一句话写清触发场景。
- 脚本管确定性，正文管判断。边界清晰后，技能可跨 agent 复用。
- 技能就是文件，进 git 管理；团队共享走 ClawHub 或私有仓库同步。
- 排查"技能没触发"按顺序看三件事：索引里有没有它、description 是否匹配用户的说法、agent 有没有 read 权限。

## 总结

Skills 的价值不在于"能力更多"，而在于把能力从固定的上下文成本变成按需取用的文件。写好一句 description、正文保持克制、逻辑下沉到脚本，这套机制就能让 agent 在不膨胀 prompt 的前提下稳定扩展。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/2e6544b0d327077f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/98620a95dbb2d133.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/f6bcb824ebd3334f.png)

