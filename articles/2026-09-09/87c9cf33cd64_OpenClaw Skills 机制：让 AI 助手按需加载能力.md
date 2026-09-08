---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 36669
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

OpenClaw 的 Agent 默认把可用的工具说明、工作流指令都放进上下文。工具少的时候没问题，但一旦接入多个 MCP server、自定义脚本和流程文档，system prompt 就会迅速膨胀——token 成本上升还是小事，真正的麻烦是注意力被稀释：模型面对几十条指令，反而更容易选错工具、漏掉关键约束。

Skills 机制就是针对这个问题设计的：把"能力"拆成独立的技能包，运行时按需加载，而不是一次性全量注入。

## 问题：全量注入 vs 按需加载

早期两种常见做法各有代价：

1. **全写进 system prompt**。简单直接，但一段 2000 token 的说明哪怕本次用不上也照付成本，还会干扰其他指令的权重。
2. **全部封装成 MCP tools**。工具描述同样会进上下文，工具一多，模型的选择准确率明显下降。

理想状态是：平时只知道"有哪些能力、什么时候用"，真正用到时才加载完整说明。Skills 的三层渐进式披露就是这个思路。

## 做法

**1. 技能包结构**

每个 Skill 是一个目录，核心是 `SKILL.md`：

```text
~/.openclaw/skills/
└── weekly-report/
    ├── SKILL.md        # 元信息 + 使用说明
    ├── template.md     # 模板，用到才读
    └── gen_chart.py    # 辅助脚本
```

**2. SKILL.md 写法**

frontmatter 里的 `name` 和 `description` 是常驻上下文的唯一部分，正文按需加载：

```yaml
---
name: weekly-report
description: 生成周报。当用户要求整理本周工作、汇总提交记录或输出周报时使用。
---
```

正文写清执行步骤、模板引用方式和脚本调用方法。

**3. 三层加载逻辑**

- 第一层：启动时只注入所有技能的 name + description，每个几十 token；
- 第二层：模型判断命中某技能后，读取完整 `SKILL.md`；
- 第三层：正文引用的模板、脚本，执行时才加载。

改完文件后重启 gateway（或触发 reload），先用一句"你现在有哪些技能"验证列表，再实际跑一遍确认加载路径。

## 踩坑点

- **description 是路由键**。写得含糊（"辅助办公"）技能永远不会被命中；写得太宽（"处理任何文档"）又会被频繁误触发。建议格式：做什么 + 什么场景触发。
- **正文别贪长**。SKILL.md 本身超过一两千 token，按需加载就失去意义了。细节下沉到引用文件。
- **脚本权限和路径**。脚本要给执行权限；路径建议写成相对 workspace 的形式，绝对路径换个机器就挂。
- **技能重叠**。两个技能触发条件高度相似时，模型会犹豫甚至选错，要么合并，要么在 description 里明确边界。
- **别在 SKILL.md 里放密钥**。正文会进入上下文，敏感信息一律走环境变量。

## 可复用建议

- description 套路：「<动词短语>。当<场景列表>时使用」；
- 一个技能只解决一类任务，宁可拆小也不要做大杂烩；
- 技能目录放进 git，团队共享的就是一个可复现的能力包；
- 复杂流程可以在技能里只写编排逻辑，具体动作仍走 MCP tools——两者是互补，不是替代。

## 总结

Skills 的价值不在于"又多了一种插件格式"，而在于把上下文当成预算来花：元信息常驻、正文按需、资产延迟加载。把 description 当路由、正文当说明书、脚本当工具，大多数"prompt 越写越长"的问题都能收敛下来。建议从一个高频重复的任务开始试，跑通加载链路后再逐步迁移。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/f916392bcc9bd5cb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/d67edcf824310003.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/9fb5674ed8755d23.png)

