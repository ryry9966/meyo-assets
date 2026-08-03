---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 31474
source: 综合讨论
publishedAt: 2026-08-03
---

# OpenClaw Skills 机制：如何让 AI 助手按需加载能力

## 背景：Agent 的能力不是越多越好

用过 OpenClaw 的人应该都有体会：插件装多了，Agent 反而变"笨"了——上下文被工具定义撑爆、指令互相干扰、模型频繁选错工具。这不是模型的问题，而是我们给 Agent 喂了太多它当前任务用不到的东西。

OpenClaw 的 Skills 机制就是为了解决这个问题的：把能力拆分成独立单元，让 Agent 在需要时才加载对应 skill，而不是启动时全量塞入。这篇文章讲清楚它的工作原理、落地步骤和几个容易踩的坑。

## 问题：全量加载的代价

以我自己的配置为例，装过十几个 MCP 插件后，每次请求光工具定义就要消耗几千 token，响应变慢，且偶尔会因为 tool 描述相似导致调用错乱。更隐蔽的问题是——当 Agent 面对一个模糊任务时，工具越多，选择越犹豫，行为越不稳定。

核心矛盾：AI 助手的能力需要可扩展，但上下文窗口和决策质量是有限的。

## Skills 机制：怎么做

### 目录结构

OpenClaw 的 skill 本质是一个目录，包含：

- `SKILL.md`：manifest + 触发条件 + 使用说明
- `scripts/`：可执行脚本或代码
- `assets/`：静态资源

一个最小示例：

```
skills/
  pdf-summary/
    SKILL.md
    scripts/summarize.py
  calendar/
    SKILL.md
    scripts/schedule_check.py
```

### SKILL.md 是关键

这个文件决定了 skill 能否被正确触发。我的模板：

```markdown
---
name: pdf-summary
description: 用于提取 PDF 文本、生成摘要，当用户提到 PDF 文件时使用
---

# PDF 摘要 Skill

仅当用户输入涉及 PDF 文件（文件名含 .pdf 或提及"PDF"）时加载。
实现：

1. 运行 `python scripts/summarize.py <pdf_path>`
2. 收集输出，返回摘要
```

重点不在格式，而在 `description` 的写法——它决定了 Agent 何时"想起"这个 skill。

### 触发逻辑

OpenClaw 会在每次请求时扫描技能的 description，由模型判断是否需要加载。实操中触发依赖三层信息：

1. 用户 intent 与 description 的语义匹配度
2. 当前任务涉及的实体（文件名、URL、关键词）
3. 对话上下文中已有的 skill 调用记录

## 踩坑点

**坑一：description 写得太泛 → 误触发**

一开始我给 git 操作 skill 写的是 "handle git commands"。结果 Agent 在写代码时也频繁加载它，白白浪费 context。改成 "only when user explicitly requests git operations (commit, push, rebase, etc.)" 之后情况改善很多。

**坑二：把 skill 当作 MCP 的替代品**

MCP 和应用层 skill 定位不同：MCP 面向外部系统访问（数据库、浏览器），skill 偏向任务编排和本地逻辑。不要全部改成 skill——需要高频实时连接的系统用 MCP 更稳。

**坑三：skill 互相之间没有隔离**

两个 skill 都定义了一个叫 `utils.py` 的公共模块，加载顺序不同会导致行为不一致。建议 skill 目录内使用相对导入，或者统一约定公共库放 `skills/_shared/`。

**坑四：调试困难**

技能不触发时，先查日志里有没有 "skill loaded" 记录。如果没有，大概率是 description 和用户输入语义差距太大。先在 CLI 里用自己的话描述任务，再改 description，迭代三个来回基本能收敛。

## 可复用建议

1. **每个 skill 只做一件事**，并显式写清"不做什么"——负向约束对模型很有效。
2. **description 可包含关键词列表**，比如 pdf 就写 pdf, PDF, .pdf, 论文, 合同，能提升匹配准确率。
3. **优先用 OpenClaw 内置的 CLI 测试 skill 效果**，看每次请求后 loading 的模块列表，确认这轮加载是否合理。
4. **把常用 skill 分好优先级**：非常高的放远程 MCP 做实时同步，普通频率的做本地 skill，一次性的任务用临时 prompt 即可，不必做成 skill。
5. **给 skill 设退出条件**：比如 "after finishing, unload and return to default behavior"，避免一个 skill 污染后续对话。

## 总结

OpenClaw Skills 机制的核心思路是"能力与上下文解耦"——用描述性元数据让模型自己决定何时加载什么。这套机制并不复杂，但对 description 的质量要求很高。花半小时理清你常用任务的触发词，把现有 skill 描述重写一轮，效果立竿见影：上下文占用降低，工具选择失误明显减少。

技术价值往往不在某个特性多新奇，而在合理使用时让系统变得更简单可靠。

---
**聊点实用的**：你目前给 skill 的 description 是什么风格？有没有误触发或漏触发的情况？评论区可以交流一下各自的写法。

---

