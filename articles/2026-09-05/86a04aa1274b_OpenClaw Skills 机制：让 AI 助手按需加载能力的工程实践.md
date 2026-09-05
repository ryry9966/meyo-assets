---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 36192
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景：上下文膨胀是所有 agent 的宿命

跑 OpenClaw 一段时间后，多数人会撞上同一个拐点：最初只接两三个 MCP 工具，上下文很干净；等工具、流程、个人偏好越攒越多，system prompt 越来越胖，模型开始"看花眼"——该走既定流程的自由发挥，该调浏览器的去翻了 shell。问题的本质是：**所有能力全量常驻上下文，既烧 token，也稀释注意力。**

OpenClaw 的 Skills 机制就是冲着这个问题去的：能力不常驻，按需加载。思路是渐进式披露——平时上下文里只有每个 skill 的名字和一句 description；当任务命中某个 skill 时，对应的 SKILL.md 正文才会被注入。

## Skill 长什么样

一个 skill 就是一个文件夹：

```
workspace/skills/
└── weekly-report/
    ├── SKILL.md
    └── scripts/render.py
```

SKILL.md 分两部分：YAML frontmatter（name、description 等元数据）和正文（具体执行指令）。加载是两级的：

- **第一级**：name + description 常驻上下文，充当"能力索引"，通常几十 token；
- **第二级**：正文命中才加载，写细一点、长一点都不心疼。

## 实操步骤

1. **建 skill**：在工作区 skills 目录下建文件夹，写 SKILL.md。description 是检索的唯一依据，要写成"什么场景该用我"，而不是"我是什么"。
2. **正文写执行路径**：步骤化、可验证；能脚本化的逻辑放 `scripts/`，正文引用脚本而不是复述逻辑。
3. **验证识别**：跑 `openclaw skills list` / `skills check`，确认 skill 被识别、依赖满足。
4. **测试触发边界**：问一句应该命中的话，看日志确认正文被注入；再问一句不该命中的，确认没误触发。

## Skill 还是 MCP？

这两个机制经常被混淆，我的划分标准：

- **MCP 工具**：需要结构化参数、精确调用、结构化返回的动作（查 API、操作数据库）；
- **Skill**：稳定的操作知识——流程、约定、环境细节、踩坑经验。

Skill 本质是"教模型怎么用好已有的工具"，不是替代工具调用。把需要精确参数的能力硬塞进 skill，是常见的方向性错误。

## 踩坑点

- **description 写太泛**（如"处理各种任务"）→ 永不触发或乱触发。写具体场景和关键词。
- **SKILL.md 写成万言书** → 一旦命中就把省下的 token 全吐回去。正文控制篇幅，细节进脚本和附件。
- **frontmatter 格式错**（缩进、字段名）→ skill 静默失效。每次改完必跑一次 `skills list` 验证。
- **skill 互相引用** → 形成加载链，命中一个拖一串。尽量让每个 skill 自包含。
- **环境依赖没写清** → 模型不知道要先装什么、路径在哪，执行时现场翻车。前置条件和检查命令写进正文开头。

## 可复用建议

- skills 目录进 git，像管代码一样 review 和回滚；
- 定期看触发日志，删掉从未命中的 skill——索引越长，检索越差；
- 团队共性流程做成共享 skill，比往 memory 里塞长文稳定得多；
- 新 skill 先小范围验证触发边界，再放量。

## 总结

Skills 的价值不在于"能力多"，而在于"上下文干净"。它把 agent 能力管理从一次性堆料变成一个工程问题：**description 是索引质量，SKILL.md 是实现质量，触发日志是监控指标。** 三件事做扎实，能力越多，agent 反而越稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/1205f06e93159aa6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/1e07adde2dcdf88f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/2d118d9c7174dfc4.png)

