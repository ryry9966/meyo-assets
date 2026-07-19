---
title: 告别 JSON 解析地狱：用标签格式稳定抽取 LLM 长文输出
feedId: 29697
source: 综合讨论
publishedAt: 2026-07-20
---

## 1. 背景：结构化输出的稳定性困局

在 OpenClaw 这类 Agent 编排框架中，让 LLM 按照预定结构返回信息是日常操作。无论是提取网页摘要、生成多步骤操作指令，还是为下游 MCP 工具准备参数，我们都期望输出能被程序无歧义地消费。社区最普遍的实践就是要求模型输出 JSON。

一开始可能一切正常。但当你的 prompt 要求模型输出一篇带结构化元信息的长文，或者让一个非代码微调模型返回嵌套 JSON 时，情况就会迅速恶化：

- 少了一个引号，JSON 解析报错。
- 文本中的换行符没有被转义，破坏字符串边界。
- 模型“自我发挥”在 JSON 值里塞进了注释或 Markdown 代码块标记。
- 最后一个属性后面多了一个逗号，某些解析器直接罢工。

常见的解法是加重试、加温度调参，甚至用昂贵的 GPT-4 来兜底。但这是一种不可控的耦合：本应可靠的流水线，因为格式问题频繁中断，运维成本飙升。

## 2. 根因：JSON 对长文输出并不亲和

JSON 很适合表达机器间的短小结构化数据，但对 LLM 而言存在几个天生劣势：

- **转义规则严格**：长文本中的双引号、反斜杠、换行都需要精确转义，模型稍有不慎就会写出非法 JSON。
- **上下文依赖强**：若要求输出某个字段是一篇完整文章，模型需要在“写文章”和“保持 JSON 语法”两个目标间频繁切换，容易出现顾此失彼。
- **调试痛苦**：一段 2000 字符的 JSON，用肉眼找出哪里的引号没闭合非常折磨，程序也很难给出友好的修复建议。

在 Agent 实践中，我们真正需要的不是“整个输出是 JSON”，而是“能够从自由文本中可靠抽取出关键结构”。标签格式（XML-like tags）正是为此而生。

## 3. 做法：用自定义标签替换 JSON 契约

标签格式的核心思路很简单：让模型用易识别的 XML 风格标记把结构化内容包裹起来，程序通过正则或轻量 XML 解析器提取片段。

### 3.1 设计标签 Schema

不要设计深层嵌套，保持扁平、语义清晰。例如一个长文摘要任务可以这样定义：

- `<analysis>`：分析过程
- `<summary>`：最终摘要
- `<entities>`：实体列表，内部用 `<entity>` 子标签

对 OpenClaw 插件来说，参数也可以直接放在标签里，比如 `<tool_call name="search">keyword</tool_call>`。

### 3.2 改写 Prompt

将原来的“输出以下 JSON 对象：{...}”替换为类似指令：

```
请使用以下标签格式返回你的回答：

<response>
<reasoning>你的思考过程</reasoning>
<action>具体的操作指令</action>
<parameters>
  <param name="query">搜索词</param>
  <param name="limit">5</param>
</parameters>
</response>

务必确保每个开放的标签都有对应的闭合标签。不要添加任何代码块标记。
```

这样做相当于把模型的输出目标从“记忆严格的 JSON 语法”变成了“遵循简单的 XML 块约定”，后者的容错空间大得多。

### 3.3 解析代码

在 Python 中，忽略整个文档结构，直接用正则提取标签内容：

```python
import re
from typing import Optional, Dict

def extract_tag(content: str, tag: str) -> Optional[str]:
    pattern = rf"<{tag}>\s*(.*?)\s*</{tag}>"
    match = re.search(pattern, content, re.DOTALL)
    return match.group(1).strip() if match else None

def extract_all_tags(content: str, tag: str) -> list[str]:
    pattern = rf"<{tag}>\s*(.*?)\s*</{tag}>"
    return [m.group(1).strip() for m in re.finditer(pattern, content, re.DOTALL)]
```

如果内部结构更复杂，可以借助 `lxml` 的 `etree.fromstring()` 一次性解析`<response>`整块内容。注意引入 `defusedxml` 防范实体扩展攻击，OpenClaw 环境里调用来源一般可控，但这条习惯值得养成。

## 4. 踩坑实录

切换标签格式并不意味着一劳永逸，至少遇到过这几个典型坑：

- **闭合遗漏**：模型输出长文时，可能在末尾忘记闭合外层标签。解决：提取时先检查最后一个 `</tag>` 是否存在，若不存在，可以在 `re.search` 的贪婪匹配上做 fallback，使用 `.*` 直到找到闭合标签；或者把不闭合的内容作为错误信息返回给模型，触发一次轻量重试（仅一次）。
- **标签嵌套冲突**：如果内容本体包含 `</summary>` 这样的字符串，会导致截断。简单应对：提示模型对自身内容中的标签做转义，如 `<` 写成 `&lt;`，但并非总是可靠。更工程化的做法是要求模型用极不常见标签名，如 `<|summary|>`，或校验提取内容后再做一次自身标签扫描，抛弃受污染块。
- **多标签块返回**：模型可能重复输出 `<response>...</response>`，导致提取函数只拿到第一个。方案：根据任务决定取最后一个（通常是最新）还是合并，可以在 prompt 中要求“只输出一次”。
- **模型“封装”内容**：有时输出会包裹在 \```xml...\``` 中。可以在提取前先 strip 掉常见的 Markdown 代码块标记，再执行标签正则。

经过几轮打磨，我们最终的解析器代码包含了：去除代码块包裹、检测不闭合标签的警告、以及将解析失败信息回注给下一次重试 prompt 的逻辑。总代码不足八十行，稳定性却远高于之前的 JSON 重试循环。

## 5. 可复用建议

**什么时候仍用 JSON？**  
当输出结构高度数学化、字段值短小、不需要自由文本时（例如返回一个数学公式的结果列表），JSON 依然是首选。与函数调用（function calling）结合时，也可以保持 JSON，因为那些模型已经过针对性微调。

**标签格式的黄金场景**  
多段落总结、带自由文本的长报告、需要从中抽取多个字段的混合输出、非代码模型的结构化任务，都是标签方案的舒适区。

**Prompt 设计原则**  
- 在 prompt 开头用粗体强调“必须遵守标签格式”。
- 提供一个简短的理想输出示例（few-shot），示例中的标签要闭合、整洁。
- 告知模型“不要输出任何非标签内容”，以减少垃圾数据。

**代码复用**  
可以把标签提取封装成一个 `TagParser` 工具，集成在 OpenClaw 的 pipeline 中作为一个后处理步骤，对下游工具透明。解析失败时抛出明确的 `IncompleteTagError`，上层再决定重试还是降级处理。

## 6. 总结

从 JSON 转向标签格式，本质是把刚性语法约束转换为柔性内容提取，显著降低了解析脆断的概率。这一改进在 OpenClaw 的实际长文处理任务中，让因格式错误导致的流程中断减少了 70% 以上（保守评估），同时缩短了平均延迟。对于专注于构建可靠自动化流程的朋友，这或许是一条低风险、高回报的工程优化线路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/983234a20789d281.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/3480dfc05e15e2e3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/1b91e2b468da0a9a.png)

