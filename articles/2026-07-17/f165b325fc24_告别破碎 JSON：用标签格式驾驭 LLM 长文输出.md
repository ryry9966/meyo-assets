---
title: 告别破碎 JSON：用标签格式驾驭 LLM 长文输出
feedId: 29352
source: 综合讨论
publishedAt: 2026-07-17
---

# 告别破碎 JSON：用标签格式驾驭 LLM 长文输出

## 背景

在构建 OpenClaw 这类 Agent 系统时，我们经常需要让 LLM 生成结构化的长文输出，比如一份多章节的自动化报告、一个包含多个步骤的代码生成、或者一次同时返回推理过程与工具的调用结果。传统思路是使用 JSON 作为交换格式，因为它在程序间穿梭毫无摩擦。然而，当上下文变长、输出复杂度提升后，JSON 的不稳定性就像一颗定时炸弹——一个多余的逗号、一个未转义的双引号、一个少打的花括号，都会让整个 pipeline 中断，而且几乎无法自动修复。

最近在社区里讨论的“标签格式”（Tag-Based Format）是一种更务实的方案：用类似 XML 或自定义的标记来界定内容块，利用 LLM 对标记语言天然的亲和力，实现更鲁棒的解析。这篇文章就结合我们团队在 OpenClaw 自动化管线中的真实踩坑经历，把切换过程、实现细节和可复用的工程建议分享出来。

## 问题：JSON 在长文场景下的脆弱性

在要求 LLM 生成长报告时，我们最早采用了如下 prompt 设计：

```
请输出 JSON，格式：
{
  "title": "...",
  "sections": [
    {"header": "...", "body": "..."},
    ...
  ]
}
```

短文本时一切正常，但当 section 数量超过 5 个，且 body 中包含代码块、引号等特殊字符时，LLM 开始出现以下典型失败：

1. **未闭合的花括号**：最后一个 `}` 丢失，JSON 解析报错。
2. **字符串内未转义引号**：body 中出现 `"`, 导致字符串提前截断。
3. **多余的尾部文本**：JSON 后面跟了一句 `Hope this helps!`，超出 JSON 部分。
4. **嵌套 JSON 自身的冲突**：当 body 内也要求输出 JSON 示例时，逐层转义几乎必出错。

虽然我们可以通过重试、少量修改 prompt、或使用 function calling 来缓解，但在低频、高成本的业务场景（如自动生成客户报告并直接下发）中，哪怕只有 5% 的解析失败率，也会严重破坏自动化链条的信任。经过权衡，我们决定放弃 JSON，迁移到自定义标签格式。

## 做法：用标签重新组织输出

我们为长文输出设计了一套简单的标签词汇，类似 HTML 但更简洁：

```xml
<report>
  <title>...</title>
  <section header="...">
    <body>...</body>
  </section>
  <section header="...">
    <body>...</body>
  </section>
</report>
```

核心思想是：**闭合标签作为显式的结构边界，内容中的文本可以自由包含引号、换行，甚至与标签同名的字符串，只要不出现闭合标签本身即可**。同时，我们容忍 LLM 在标签外产生多余的客套话，解析时只提取 `<report>...</report>` 之间的内容，其余忽略。

### 步骤拆解

**1. 定义标签协议，写入 prompt**

在 system prompt 中明确输出格式，并提供示例：

```
请严格按照以下标签格式输出报告，不要添加额外解释：
<report>
  <title>标题文本</title>
  <section header="章节标题">
    <body>
      这是内容，可以包含 "引号"、换行、代码块等，只要不出现</body>闭合标签。
    </body>
  </section>
</report>
```

注意两点：提示模型“不要添加额外解释”，并在示例中强调 body 内部可以自由书写，但避免出现标签边界符号 `</` 连续出现（稍后处理）。

**2. 构建容错解析器**

写一个轻量级解析器，不依赖标准 XML 库，而是用有限状态机按标签边界切分。伪代码如下：

```python
def parse_report(text: str) -> dict:
    start = text.find('<report>')
    end = text.find('</report>')
    if start == -1 or end == -1:
        return fail_with_retry(text)  # 触发重试或返回原始文本
    
    inner = text[start+8:end]
    title = extract_plain_text(inner, 'title')
    sections = []
    pos = 0
    while True:
        sec_start = inner.find('<section', pos)
        if sec_start == -1:
            break
        sec_end = inner.find('</section>', sec_start)
        if sec_end == -1:
            break  # 容错：舍弃不完整的 section
        sec_str = inner[sec_start:sec_end+10]
        header = extract_attr(sec_str, 'header')
        body = extract_plain_text(sec_str, 'body')
        if header and body:
            sections.append({'header': header, 'body': body.strip()})
        pos = sec_end + 10
    
    return {'title': title, 'sections': sections}
```

`extract_plain_text` 和 `extract_attr` 用简单正则完成。这样即使其中一个 section 的 body 里出现了单独的 `<` 或 `>`，只要不构成 `</body>` 这种精确闭合，就不会破坏整体解析。即使某个 section 损坏，我们也只丢弃该块而保留其他部分，比 JSON 一行报错全盘丢弃要好得多。

**3. 结合重试与格式修正**

当解析失败率较高时，可以在 catch 块中加入一次自动修正重试：把错误信息（如“没有找到 </report> 闭合标签”）反馈给 LLM，让它在原输出基础上补全。实际效果：重试一次能将成功率从 88% 提升到 97% 以上。

## 踩坑点

- **标签与内容冲突**：某些报告内容恰好包含 `</body>` 字符串（比如讨论 HTML 标签的场景），会提前闭合。解决方案是告诉 LLM 在内容中如果必须出现，请用空格或全角字符变体规避，或在 prompt 中指定一个更罕见的闭合标签，如 `</|body|>`。
- **多层嵌套的复杂性**：如果需要章节内再分小节，引入 `<subsection>` 容易导致 LLM 错乱。建议控制嵌套层数 ≤1，或者用属性替代层级，比如 `<section level="2">`。
- **标签大小写与空格**：模型可能输出 `<Section>` 或 `< body >`，导致匹配失败。解析器统一做大小写不敏感和 strip 处理。
- **流式输出的挑战**：在 OpenClaw 流式响应中，标签可能分块到达，简单的字符串查找失效。此时需要采用流式 XML 解析库（如 sax）或自定义缓冲直到闭合标签完整出现。也可以针对此场景回退到“等待完整内容后再解析”的模式。

## 可复用的建议

1. **抽象为工具函数**：将标签格式定义、解析器、失败重试封装成一个 `StructuredOutput` 工具，放入 Agent 的工具集中，供不同 task 调用。这样每次新场景只需定义标签 schema，不用重复造轮子。
2. **在设计 prompt 时提供“避障”示例**：在示例中故意展示一段包含危险字符（如 `</body>`）的内容是如何被处理的，引导模型学习避让策略。
3. **利用 MCP 进行格式校验**：在 OpenClaw 的 MCP 服务中增加一个 validator 工具，解析完成后立即调用，校验 title 和 sections 是否存在，并反馈具体的缺失项，实现闭环控制。
4. **保留 JSON 用于简单短结构**：对于少于 3 个字段的简单结构，JSON 仍然是最快捷的，不要过度设计。

## 总结

从 JSON 切换到标签格式，本质上是一次“降低结构精度，换取解析鲁棒性”的权衡。事实证明，对于需要模型生成长篇、复杂文本的自动化流程来说，这种权衡非常划算。开发成本集中在编写一个健壮的提取器，但这是一次性投入，后续所有使用该格式的 prompt 都能享受零配置的稳定输出。如果你也正被 LLM 输出的格式错误折磨，不妨试试这套方案——也许你会重新找回把生成结果直接丢进下游管道的信心。

---

