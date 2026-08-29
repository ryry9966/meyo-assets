---
title: OpenClaw Skills 机制：把 AI 助手的能力从“常驻”改造成“按需加载”
feedId: 35173
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 里接入多个 MCP server、插件和自动化脚本后，系统提示词很容易变成“全家桶”：所有工具描述、使用示例、边界条件都常驻在上下文里。结果是上下文预算被快速消耗，模型在工具选择时噪声变大，甚至出现用错工具的情况。后来我把部分能力拆成 Skills，默认只保留触发声明，命中的技能才注入完整指令，体验明显改善。

## 问题

全量加载主要有四类成本：

1. **上下文膨胀**：每个工具、每个插件的说明都占 token，对话还没开始，预算已经去掉一大块。
2. **工具冲突**：多个能力暴露相似工具名时，模型容易混淆，出现错误调用。
3. **维护困难**：所有能力耦合在主 prompt 里，改一个能力要动全局配置。
4. **安全暴露**：不常用的高危脚本或工具也常驻，增加了误触发和攻击面。

Skills 机制的核心思路是：默认只保留“技能的索引”，不加载完整技能内容；等模型判断需要某个技能时，再把该技能的指令、脚本说明、输出要求注入当前会话。

## 做法/步骤

先确认 OpenClaw 的 skills 目录扫描路径。以我目前环境为例，结构如下：

```text
skills/
  pdf-extract/
    SKILL.md
    scripts/parse_pdf.py
    references/output_format.md
```

`SKILL.md` 是技能入口，示例：

```yaml
---
name: pdf-extract
description: Extract text and tables from PDF files.
when_to_use: When the user asks to read, summarize, or extract content from a PDF.
not_for: Scanned PDF without OCR, image-only documents, editing PDFs.
tools: [file-reader]
---
```

正文只写必要步骤和命令：

```text
1. 确认文件存在。
2. 调用 file-reader 读取指定页。
3. 将输出整理为 Markdown 表格。
```

注意不要在 `SKILL.md` 里贴大段文档或代码；大资源外置到 `references/` 或脚本中，按需二次读取。

触发加载时，可以看日志确认哪个 skill 被注入，避免“我以为加载了但实际没加载”。测试时用最小 prompt 验证，比如“帮我读下这个 PDF”应命中；而“把这张图片扫描成文字”不应命中 `pdf-extract`。

与 MCP 协同的原则是：skill 负责编排和约束，MCP 负责执行。不要在 skill 里重复注册工具，而是引用已有 MCP 工具名，否则容易造成工具定义重复。

## 踩坑点

- **触发词过宽**：`when_to_use` 写成“处理文档”会导致几乎任何文件操作都加载，反而污染上下文。要写具体场景和负向边界。
- **正文过重**：把完整 prompt 模板、大段数据或示例塞进 `SKILL.md`，加载成本高。可以拆到 `references/` 下按需读取。
- **脚本安全**：skill 调用脚本时容易直接拼接参数，存在路径穿越/命令注入风险。建议只允许执行 skill 目录内脚本，并使用绝对路径和参数数组。
- **工具命名冲突**：多个 skill 若暴露同名工具，会互相覆盖。可以用命名空间或声明依赖工具，不自己注册。
- **没有退出条件**：skill 加载后一直占用上下文。可以在正文最后写“完成本次任务后不再使用该技能说明”，减少多轮对话中的残留。

## 可复用建议

- 把 skill 数量当作上下文预算管理，不是越多越好。
- 写 `description` 时同时写“适合什么/不适合什么”，负向边界和正向触发同样重要。
- 每个 skill 至少配一条命中用例和一条不命中用例，方便回归。
- 记录每次加载的 skill、token 增量、耗时，长期看可以淘汰低价值技能。
- 将大段示例外置到 `references/`，只在需要时二次读取。

## 总结

Skills 机制不是要替代插件或 MCP，而是在工具层和模型层之间增加一个“能力调度层”。按需加载的核心不是技术复杂，而是把触发描述写准、把正文保持轻量、把边界和退出条件管住。这样技能多了以后，助手不会变得更笨，反而更稳定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/f3eeca2f03b7e252.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/0157549e9ac9ea88.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/e16afbd72a867302.png)

