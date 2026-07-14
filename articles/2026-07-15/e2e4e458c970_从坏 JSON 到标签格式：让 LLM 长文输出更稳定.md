---
title: 从坏 JSON 到标签格式：让 LLM 长文输出更稳定
feedId: 29125
source: 综合讨论
publishedAt: 2026-07-15
---

在构建基于 LLM 的自动化管线（例如文档生成 Agent、报告合成工具、MCP 资源生产器）时，你大概率遇到过这样的场景：模型需要输出一段很长的文本，你把这段文本包在一个 JSON 里，结果解析器频频报错——引号未转义、换行符错乱、突然多出一个大括号，或者模型在长篇大论后给了一个无法闭环的 JSON 片段。

当输出从短结构体变成长文，JSON 的脆弱性会急剧放大。本文将分享一种工程上更稳健的替代方案：**使用自定义 XML 标签包裹长文输出**，并给出面向 OpenClaw/Agent 上下文的可复用实践。

---

## 1. 问题：JSON 为什么在长文场景下频繁翻车

LLM 生成 JSON 时，常见的失败模式有：

- **字符串转义不完整**：长文中带有英文引号 `"`、反斜杠 `\`、换行 `\n` 等，模型偶尔会漏加转义，导致 JSON 解析直接失败。
- **结构未闭合**：模型写到一半超出 token 限制或被提前截断，最后少一个 `}`，整个响应变成无效。
- **思维链混入**：某些推理模型会在输出前加入 `<thinking>` 或自然语言解释，破坏 JSON 结构。
- **嵌套深度和可读性下降**：当你想把 Markdown、代码块、前端组件代码包在 JSON 字符串里，阅读和调试都极其痛苦。

这些问题在短 JSON 或简单工具调用时还能通过重试解决，但当输出是**一篇千字以上的完整文章、一份带格式的报告或一长段脚本**时，重试成本高且体验差。

---

## 2. 替代思路：切换到标签格式

思路很简单：放弃用 JSON 作为长文本的容器，改为让模型输出类似这样的结构：

```
<generated_article>
这里是可以任意书写的长文，包含换行、引号、代码 block 等……
完全不用担心转义问题。
</generated_article>
```

然后通过正则提取标签内的内容。这种方法天然适合长文本，因为：

- **无需转义**：标签内的内容就是原始文本，引号、换行直接保留。
- **容错性高**：即使模型前面多说了几句“好的，这是你要的文章”，只要标签还在，就能提取。
- **易调试**：人眼可以直接阅读标签内容，不需要对着 JSON 展开。
- **对截断部分可降级**：如果只输出了一半文章，我们至少能拿到部分 `<generated_article>` 内容，而截断的 JSON 几乎不可用。

在 OpenClaw 的 Pipe 模板或 Agent 工作流中，这种标签约定可与 `response_format` 配置搭配，替代过去的 JSON Schema 硬约束。

---

## 3. 具体做法：从 Prompt 到提取代码

### 3.1 设计标签名

选择一个足够独特、通常不会在内容中出现的标签名，例如 `[GENERATED_DOC]`、`<system_article>` 或 `__LLM_OUTPUT__`。避免使用 `article`、`body` 等常见词。

同时约定：**只输出一个开标签和一个闭标签，不允许在内容中再次出现相同标签**。必要时可以在 Prompt 中禁止使用标签字符（如 `告诉模型内容不要包含 '</system_article>'`），这样就不必实现 CDATA 或转义逻辑。

### 3.2 编写 Prompt 模板

以生成一篇技术文章为例，在 OpenClaw 的模板里：

```
你是一个技术作者。请根据以下要求生成一篇完整的文章。
要求：直接输出 <article_output> 和 </article_output> 标签，并将文章内容放在标签之间。
不要输出任何其他解释或 Markdown 代码围栏。
文章内容可以包含任何字符，但不能出现 </article_output> 字符串。
```

也可以在 System Prompt 中固化这个行为规范，让 Agent 自动沿用。

### 3.3 后处理提取

在 OpenClaw 的 `on_response` 或自定义 Python Node 中，用正则提取标签内容：

```python
import re

def extract_tagged_content(text: str, tag: str = "article_output") -> str:
    pattern = re.compile(rf"<{tag}>(.*?)</{tag}>", re.DOTALL)
    match = pattern.search(text)
    if match:
        return match.group(1).strip()
    # 降级策略：如果没找到标签，返回原始内容（可能是模型没按格式输出）
    return text.strip()
```

如果需要提取多个区块，可以使用 `findall`，或者设计带属性的标签（如 `<step id="1">...</step>`）。

---

## 4. 踩坑点与应对

### 坑1：模型在标签外多说废话

即使 Prompt 要求“只输出标签”，模型有时仍会加上“这是您要的文章：”之类的前缀。正则提取时可通过 `search` 而非 `fullmatch` 解决，或者提前用 `strip` 处理。更严格的方案是：在 OpenClaw 的响应管道中，**先截断到第一个 `<tag>` 出现的位置**，不过这可能丢掉前缀中的有用信息，建议保留日志。

### 坑2：内容内部意外出现闭合标签

当文章需要描述 XML/HTML 标签时，模型可能输出 `</article_output>`。解决办法：

- 在 Prompt 中明确说明：“如果必须提及这个标签，请使用全角符号或拆分写法（如 ＜/article_output＞）”。
- 后缀校正：提取时如果发现内容中包含 `</tag>`，可以替换为安全的占位符，但一般不推荐。

绝大多数情况下，选择冷门的标签名（如 `__SYSTEM_GEN_123__`）能直接规避这个问题。

### 坑3：大小写不一致

部分模型会输出 `<ARTICLE_OUTPUT>` 或 `<Article_Output>`。在正则中使用 `re.IGNORECASE` 即可：

```python
pattern = re.compile(rf"<{tag}>(.*?)</{tag}>", re.DOTALL | re.IGNORECASE)
```

### 坑4：嵌套需求

如果确实需要树状结构，可以使用不同标签名，如 `<section>...</section>`，并用层叠提取。避免陷入“用一种格式解决所有结构”的陷阱——**复杂结构化数据仍建议用 JSON，标签方案主要针对非结构化长文本包装**。

---

## 5. 可复用建议

在团队或项目中落地，可以沉淀以下几件事：

- **统一标签库**：规定一组标准标签（`<doc>`、`<code_block>`、`<reasoning>`），写入共享模板库，Agent 开发时直接引用。
- **提取函数做成可配置工具**：将 `extract_tagged_content` 封装为可配置的 Plugin Node，支持标签名、正则模式、降级策略，在 OpenClaw 流程图中拖拽使用。
- **加监控日志**：每一次提取都记录是否命中标签、内容长度，便于发现模型是否偏离格式，及时调整 Prompt。
- **保留 JSON 作为后备**：对于短小、高结构化输出（如参数列表），JSON 仍然更合适。可以在 Agent 内部做路由：根据任务类型选择输出格式。

---

## 6. 总结

标签格式并不新鲜，但在 LLM 工程化落地中，它是一种被低估的鲁棒性实践。相比强迫模型输出完美的 JSON 包裹，用一对自定义标签兜底，能显著降低解析失败率，尤其适合需要生成长文、报告、脚本的多步骤自动化流程。

在 OpenClaw、Agent、MCP 等强调可组合性的环境里，与其不断修补 JSON 的边界情况，不如给长文本一个更宽松的容器。简单、可控、易恢复——这才是自动化管线真正需要的稳定性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/a331190b52f6fac6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/24b41621220cebc8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/e58f98d3d624301d.png)

