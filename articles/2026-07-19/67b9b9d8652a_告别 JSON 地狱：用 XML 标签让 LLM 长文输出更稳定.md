---
title: 告别 JSON 地狱：用 XML 标签让 LLM 长文输出更稳定
feedId: 29662
source: 综合讨论
publishedAt: 2026-07-19
---

## 为什么长文格式化输出总在坏 JSON 上翻车？

在 OpenClaw 构建的 Agent 工作流里，我们经常需要 LLM 生成带结构的“长文”——比如一份分章节的调研报告、带多段摘要的新闻简报、甚至是一整篇技术文档。很自然地，团队第一反应是让模型输出 JSON：

```json
{
  "title": "...",
  "sections": [
    {
      "heading": "...",
      "content": "这是一大段包含换行、双引号、"特殊字符"的文本..."
    }
  ]
}
```

但实际跑下来，坏 JSON 几乎成为高频故障源。问题集中在：

- 模型在长文生成过程中被截断，缺少闭合括号或大括号。
- `content` 内部存在未转义的双引号、反斜杠或 Unicode 控制字符，直接破坏语法。
- 嵌套复杂时，LLM 容易多打或少打 `]}`，尤其是当输出恰好到达 token 限制。
- 发生错误后，整个响应体直接报废，很难部分恢复。

在自动化 pipeline 里，这意味着一个本来能用的结果因为解析失败被整包丢弃，调用链中断，重试成本很高。如果这类任务只是偶尔触发，还能靠重试扛过去；一旦变成业务流里的长链路环节，稳定性的问题就会急剧放大。

## 换个思路：用 XML 标签替代 JSON 描述结构

排查了数十次坏 JSON 后，我们发现一个现象：**在 LLM 的 token 流中，多字符的结构定界符（如 JSON 括号）比单词级的标签更容易“断裂”**。而 XML/HTML 风格的标签（`<section>...</section>`）天然具备以下优势：

- 结构信息由开始标签和结束标签共同锚定，即使中间文本再长，闭合关系也有独立标记。
- 标签名本身是明确语义的，错误时修复难度远低于 JSON 的括号匹配。
- LLM 在训练语料中见过大量类 HTML/XML 的文档，对 `<...>` 的生成更稳定，不容易忘记闭合。

因此我们在一批 OpenClaw 的长文生成插件中，把输出格式从 JSON 切换为自定义标签格式，例如：

```xml
<article>
  <title>报告标题</title>
  <section>
    <heading>第一部分</heading>
    <content>这里是完整的正文内容，可以自由包含引号、换行、特殊符号，不再需要 JSON 转义。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/f988043c7cc50f4e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/45f88ebf6d04340b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/ebf4a6b14d3ebc05.png)

