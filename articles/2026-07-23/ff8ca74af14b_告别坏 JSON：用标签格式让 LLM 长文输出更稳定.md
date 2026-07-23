---
title: 告别坏 JSON：用标签格式让 LLM 长文输出更稳定
feedId: 30176
source: 综合讨论
publishedAt: 2026-07-23
---

# 告别坏 JSON：用标签格式让 LLM 长文输出更稳定

## 背景：结构化输出的两难

在 OpenClaw 的自动化工作流里，让 LLM 输出结构化数据几乎是刚需。从 Agent 的思考链提取、MCP 插件的工具调用参数，到多步任务的中间结果传递，我们都希望模型“好好说话、规规矩矩交数据”。

最常见的做法是让模型输出 JSON。示例给得漂漂亮亮，单测跑得很完美，可一旦放进真实的长文生成场景（摘要 > 800 字、多段落翻译、日志分析报告），解析就开始崩。**根本原因不是模型能力不够，而是 JSON 本身的字符敏感性对长文本极不友好**：

- 未转义的引号、换行符、反斜杠，在长文本中几乎不可避免；
- 输出被 token 限制截断时，JSON 缺少闭合括号，整个 payload 不可解析；
- 模型偶尔在字符串值里多写一个 `{` 或 `[`，直接毁掉整个结构。

于是你会看到写得很稳健的 OpenClaw pipeline，就因为在某个节点的输出里少了一个 `"`，后续处理全部报错，自动化被迫中断。我们需要一种对长文本容错性更强的结构化约定。

## 问题：JSON 的脆弱性在长文中被放大

用一个典型场景来说：我们让 Agent 阅读一篇长文档，输出 `{"summary": "…", "keywords": ["…"]}`。summary 里包含原文中的直接引语，于是产出变成：

```json
{
  "summary": "作者说："这种设计从根本上解决了问题。"",
  "keywords": ["architecture", "design"]
}
```

两个中文双引号就直接让 JSON 非法。即使 prompt 里要求用 `\"` 转义，长文场景下模型很难 100% 遵守。另一个更隐蔽的问题是，OpenAI-compatible 的 `response_format` 强制 JSON 模式时，不支持指定 key 内任意长文本，也不支持 markdown 换行，这极大限制了可读性。

最终我们发现：在很多任务里，我们只是需要可靠地**从一大段自然语言中捞出几个信息块**，而不是一定要严谨的嵌套容器。**这时候，自定义标签格式是更好的选择。**

## 做法：用标签格式替代 JSON，让 LLM「闭合即正确」

### 1. 设计简单的标签约定

直接让模型输出如下结构：

```
<summary>
这里是可以自由换行的长文本，包含任意标点，即使有 < 或 > 也没关系。
</summary>

<keywords>
keyword1, keyword2, keyword3
</keywords>
```

不再需要转义引号，不用操心逗号结尾是否多余，也不存在深度嵌套。对模型而言，这更像是在写带有 XML/HTML 标签的文本，生成过程很自然。

### 2. 在 prompt 中明确规则并给出示例

在 OpenClaw 的节点 prompt 中，我习惯用这样的指令：

> 你的回复必须严格包含以下标签，每个标签单独占一行开始与结束。标签内的文本可以包含任意字符，不需要转义。
> 示例：
> `<summary>文本内容</summary>`
> `<keywords>词1, 词2</keywords>`

### 3. 解析端的容错设计

关键步骤在解析。我会用非贪婪正则逐段提取，**而不是要求整个文档符合 XML 规范**。Python 示例：

```python
import re

def extract_tag(text: str, tag: str) -> str:
    pattern = rf"<{tag}>(.*?)</{tag}>"
    match = re.search(pattern, text, re.DOTALL)
    return match.group(1).strip() if match else None
```

对于可能被截断的标签，比如模型输出到一半就达到 max_tokens，可以增加一个“未闭合回退”：

```python
def extract_tag_lenient(text: str, tag: str) -> str:
    # 先尝试完整闭合
    full = extract_tag(text, tag)
    if full:
        return full
    # 未闭合时，提取 <tag> 后所有内容，直到文本末尾
    pattern = rf"<{tag}>(.*?)$"
    match = re.search(pattern, text, re.DOTALL)
    return match.group(1).strip() if match else None
```

这样即使标签没闭合，我们也至少能得到 summary 的前大半部分，不会让整个 workflow 卡死。

在 OpenClaw 的 MCP 插件中，我通常会为这类提取写一个通用的 `tag_extractor` 工具函数，供多个 prompt 复用，解析失败时记录原始内容并返回 fallback 值，保障下游不崩溃。

## 踩坑点

1. **标签名冲突**：如果任务本身会输出 HTML 或 XML，不要使用通用标签名。可以用带前缀的标签，如 `<o-summary>`，降低误匹配概率。
2. **标签内的“假标签”**：当文本内容包含 `<summary>` 这类字样时，简单的非贪婪匹配会提前截断。可以在 prompt 里要求模型不要输出与标签同名的文本，或使用 CDATA 片段（但在中文场景实用性一般）。更务实的办法是提取时优先匹配最外层，或采用计数方式，但复杂度上升。我目前的经验是，**在真正长文摘要中，出现同名标签的概率极低，简单正则足够**。
3. **多块重复标签**：如果允许多个 block，比如多段摘要，可以要求用 `<item>` 包裹，然后 `re.findall` 提取。但多数场景下，每个标签只出现一次最简单可靠。
4. **与 JSON mode 共存**：OpenClaw 中如果某些节点必须用 JSON mode，标签抽取可以放在它之后的处理节点进行，先拿到原始文本再解析。不要奢望在一次调用里同时享受两者的优势。

## 可复用建议

- **建立标签字典**：为你的项目统一一组标签命名规范，如 `<analysis>`、`<commands>`、`<final_answer>`，便于多个 Agent 复用解析器。
- **始终保留原始输出**：在提取前把完整 LLM 输出记入日志。当解析出问题时，可以人工回溯，快速定位是生成异常还是正则 bug。
- **提供 fallback 链路**：如果标签提取失败，回退到使用全文作为 summary，或用简单的首句截断。自动化系统最怕的是停止运行，而不是结果不够精确。
- **在 prompt 末尾重申格式**：把输出格式要求放在 system prompt 末尾，对长上下文模型更有效。

## 总结

标签格式不是银弹，它放弃了 JSON 的强类型和嵌套能力，换来了 **“长文本输出解析几乎零失败”** 的工程收益。尤其在 OpenClaw 这类需要串联多个 LLM 节点的自动化平台中，一处解析失败就意味着整条 pipeline 的中断，稳定性远比数据结构严谨更重要。

如果你也被坏 JSON 折磨过，不妨试试标签格式。只要设计得当，大部分情况下它带来的可靠性提升，远超过那一点点结构表达上的牺牲。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/51b34438f324aeda.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/156e774dc8380490.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/09c5440503e625c6.png)

