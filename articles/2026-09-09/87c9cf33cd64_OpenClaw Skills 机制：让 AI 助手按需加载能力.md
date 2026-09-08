---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 36690
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

OpenClaw 网关跑起来之后，最容易膨胀的不是代码，是上下文。接了几个 MCP server、配了一堆常驻工具后，系统提示里躺着几十个工具 schema，token 先涨上去，模型的工具选择准确率反而往下掉。

Skills 的思路是把「能力说明书」从常驻上下文里挪出去：每个 Skill 是一个文件夹，核心是一份 SKILL.md（YAML frontmatter + 正文）。网关平时只把 name 和 description 注入上下文，正文只有当模型判断当前任务匹配时才完整加载——典型的渐进式披露（progressive disclosure）。

## 做法

**1. 建目录。** 技能放 `~/.openclaw/skills/<name>/` 或工作区 `skills/` 下，一份 SKILL.md 起步：

```markdown
---
name: gh-issue-digest
description: 抓取并总结 GitHub issue 讨论串；当用户贴出 issue 链接时使用
metadata:
  requires:
    bins: ["gh"]
---

（正文：操作步骤、边界条件、脚本调用约定）
```

**2. 把 description 当路由信号写。** 不写营销文案，写触发条件：「做什么 + 什么时候用」。这是整个机制里最值得花时间的一行。

**3. 正文只放决策路径。** 步骤、判断分支、脚本调用方式；大段参考文档拆成附属文件，正文里写明「需要细节时读取 docs/xxx.md」，让加载也变成按需的。

**4. 校验与启用。** `openclaw skills list` 确认被识别；在 openclaw.json 里按 agent 做 allowlist，别把所有技能喂给所有 agent。改完 SKILL.md 热重载或重启网关。

**5. 实测触发。** 丢一个该触发的请求进去，看日志确认正文真的被加载，而不是模型在凭描述猜。

## 踩坑点

- description 写得太泛或太文艺 → 永不触发，或什么都触发。
- 技能堆到几十个后，描述本身也是上下文开销，路由质量随之下降。定期删掉从不触发的。
- 正文塞满文档 → 每次触发全量进上下文，省了个寂寞。
- `requires` 声明的 CLI 没装，技能会被静默跳过且不易发现——部署时统一校验一遍。
- 一个技能塞多个职责，容易和系统提示词打架。一个技能解决一类事。
- 别在 SKILL.md 里放密钥，它是会被读进上下文的文件。

## 可复用建议

- 按「高频常驻 vs 低频按需」拆分能力：高频的走 MCP/常驻工具，低频的做成 Skill，两者互补而非替代。
- description 用团队统一模板：动作 + 触发场景，降低协作时的理解成本。
- 重复动作固化成脚本放进技能目录，让模型调用脚本而不是自由发挥，结果更稳定。
- 技能当代码维护：进 git、走 review；改 description 要像改接口签名一样谨慎。

## 总结

Skills 不是魔法，本质是把上下文当预算做工程分配：描述常驻、正文按需、细节再按需。真正决定效果的是 description 的质量和文件组织的克制程度。建议从把现有某个低频工作流改造成一个 Skill 开始，观察触发日志，再决定要不要铺开。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/514d67ce92c5f143.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/5f8d746869d608ab.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/a9a57e4186f708c1.png)

