---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 35248
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景：能力越多，上下文越重

在 OpenClaw 里接 MCP、插件或脚本能力时，很容易陷入一个循环：为了让模型“更聪明”，不断往系统提示里塞工具说明、SOP、示例。结果是常驻上下文越来越大，真正留给业务数据的空间变少，模型也开始选错工具。

Skills 机制要解决的不是“怎样增加更多能力”，而是“怎样让能力在需要时再出现”。它最核心的一点是：**把能力的索引和能力的执行内容拆开**。常驻上下文中只保留轻量描述，真正详细的步骤、约束、脚本引用在命中后才加载。

## 问题：全量加载的典型症状

如果只是把一堆 skill 文件拼进 system prompt，通常会遇到：

- Token 占用高：工具说明和 SOP 挤占业务上下文。
- 误调用率上升：模型看到太多相似工具，容易选错或混用。
- 冷启动变慢：每次会话都要解析大量能力定义。
- 调试困难：出问题时分不清是模型理解错了，还是 skill 本身写得模糊。

这些问题的本质不是能力太多，而是“知道有什么能力”和“加载能力细节”没有分离。

## 做法：两阶段加载

OpenClaw Skills 可以按“索引层 + 激活层”来落地。

### 1. 每个 Skill 是一个目录

建议结构如下：

```text
skills/
  pdf-extract/
    manifest.yaml
    SKILL.md
    scripts/extract.py
```

`manifest.yaml` 只放轻量元数据：

```yaml
name: pdf-extract
description: Extract text and tables from PDF files.
triggers:
  keywords: ["pdf", "extract", "table"]
  tool_need: ["pdf_parser", "file_reader"]
tools: ["read_file", "run_python"]
entrypoint: SKILL.md
ttl: 1
priority: 5
```

### 2. 启动时只建索引

OpenClaw 启动或扫描 skills 目录时，只把 `name`、`description`、`triggers`、`tools`、`priority` 放进可检索索引。不要直接用 `SKILL.md` 里的大段步骤作为常驻提示。

### 3. 命中后再激活

当用户输入、任务规划或工具调用失败信号出现时，根据关键词、工具需求或语义匹配。命中后加载 `SKILL.md` 的内容进入当前任务上下文，并标记作用域。

### 4. 任务结束回收

任务完成或 `ttl` 到期后卸载 skill 内容，只保留结果摘要。如果有多轮任务，把必要状态写到工作目录，而不是一直留在上下文里。

## 踩坑点

### 1. 触发条件写得太宽

不要把 `triggers` 写成 `["*"]`。这等于又把按需加载退化成了全量加载。触发条件要写边界，例如：

```yaml
triggers:
  keywords: ["pdf table", "extract pdf table"]
```

而不是只写 `"pdf"`。

### 2. 描述没写清“什么时候不要用”

`description` 不只是给模型看，也是给匹配器看。要写清边界，比如：“Only for extracting tables from PDF files, not for reading plain text files.” 这会显著降低误加载概率。

### 3. 依赖没有声明

Skill 激活后才发现缺少 Python 库、ffmpeg 或某个系统工具，会直接执行失败。建议在 manifest 里显式声明依赖，并在激活前做一次快速检查。

### 4. 卸载后状态丢失

如果 skill 在运行中产生临时文件或变量，任务结束后就消失，下次又得重来。解决方式是把产物写到任务工作目录，而不是依赖对话记忆。

### 5. 多个 Skill 抢同一个触发词

比如 `pdf-extract` 和 `pdf-ocr` 都监听 `pdf`。可以给 manifest 增加 `priority` 和互斥组，避免同时加载后互相干扰。

### 6. 没有日志，排障全靠猜

至少记录：命中了哪个 skill、命中得分、加载了哪些工具、卸载时间。这样遇到“为什么没触发”或“为什么多加载了”时，能快速定位。

## 可复用建议

- **把 Skill 当压缩的 SOP**：一个 skill 只解决一个可验证任务，步骤控制在 7 条以内。
- **元数据是调试接口**：description 和 triggers 要写得像接口文档，而不是泛泛的说明。
- **提供显式调试入口**：增加类似 `skills list --verbose` 的命令，查看当前索引、匹配得分和激活状态。
- **统计收益**：对比开启按需加载前后的 token 占用、工具选择准确率和任务完成率。
- **与 MCP 分层使用**：MCP 提供底层工具，Skill 提供调用策略、约束和流程。不要在 Skill 里重复实现 MCP 已有的能力。

## 总结

OpenClaw Skills 的按需加载，不是简单地把“用时再读文件”当成优化，而是用两阶段结构管理能力生命周期：常驻上下文只保留能力目录，真正执行时才装载详细内容。这样既能让 Agent 保持轻量，又能在需要时获得足够强的扩展能力。落地时最重要的三件事是：**触发条件要窄、依赖要显式、卸载要干净**。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/9892482dc1a5f911.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/a4d01b17057a540e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/34dbf481eb340ce9.png)

