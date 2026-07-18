---
title: 从坏 JSON 到标签格式：让 LLM 长文输出更稳定
feedId: 29589
source: 综合讨论
publishedAt: 2026-07-19
---

在基于 LLM 的 Agent 或自动化流水线中，我们经常让模型输出结构化数据，再交给下一步解析执行。JSON 几乎是默认选择，但模型一输出长文本，JSON 的脆弱性就暴露无遗——少个引号、多了个逗号、嵌套层级一深，整个解析就崩了。反复重试、修复、换模型，都不如从协议层面换一种更稳定的格式。本文将分享一种实践：用自定义标签分隔内容块，替代 JSON 做长文结构化输出。

---

## 1. 背景：为什么 JSON 在长文输出里不靠谱

在 OpenClaw 做多步骤任务拆解、MCP 工具调用编排、或者让 Agent 输出执行计划与中间结果时，我们要求模型返回结构化的步骤列表，每个步骤可能包含：步骤名、依赖、参数、预期结果和一段较长的自然语言说明。模型处理 JSON 时有几个典型问题：

- **结尾括号缺失**：长文本生成时，模型注意力衰减，经常忘记闭合最外层 `}` 或 `]`，导致整个 payload 不可解析。
- **内部引号未转义**：模型中生成的文本里出现 `"` 或 `'`，没有按要求转义，直接破坏 JSON 结构。
- **嵌套深度导致幻觉**：多层嵌套的 JSON 让模型容易产生“幻觉化”的漏逗号、错逗号。
- **修复成本高**：简单的正则修补不可靠，引入 `jsonrepair` 等库也只能解决一部分情况。重试又浪费 token。

当输出长度超过 1000 tokens，JSON 错误率在笔者的实际测试场景（200 次请求）中接近 15%，对于需要稳定运行的自动化任务完全不可接受。

---

## 2. 转向标签格式：以分隔符替代嵌套

思路很简单：放弃让模型输出合法 JSON，转而要求它输出一种用自定义标签包裹的“伪标记文档”，然后我们用正则或简单解析器提取内容。不需要复杂的语法树，只要求格式规整。

例如，原本期望的 JSON：

```json
{
  "steps": [
    {
      "id": 1,
      "action": "search",
      "params": {"query": "some query"},
      "reasoning": "Because the user asked..."
    }
  ]
}
```

替换为标签格式：

```
<step>
  <id>1</id>
  <action>search</action>
  <params>
    <query>some query</query>
  </params>
  <reasoning>
    Because the user asked...
  </reasoning>
</step>
```

这种写法对模型更友好：自然语言中本身不会出现 `<step>` 这样的标签，省去了引号转义问题；每个块有明确的开始和结束标记，即使某个块内部格式稍有瑕疵，也不影响其他块的提取；而且它支持流式输出时边收边解析——遇到完整的 `<step>...</step>` 就可以作为一条记录处理。

---

## 3. 做法与步骤

### 3.1 定义标签协议

设计一组无歧义的标签名，避开模型训练数据中常见的 HTML/XML 标签（如 `<div>`、`<p>`）。可以使用业务相关前缀，比如 `oac-`（Open Agent Command），例如 `<oac-step>`、`<oac-param>`。或者直接用全大写如 `<STEP>`、`<REASONING>` 降低冲突。

### 3.2 构造系统提示与示例

系统提示中明确输出格式要求，并在 few-shot 示例中展示正确结构。例如：

```
You must output steps using the following format:
<step>
  <id>1</id>
  <action>tool_name</action>
  <params>
    <param_name>value</param_name>
  </params>
  <reasoning>
    Your explanation here. Can be multi-line.
  </reasoning>
</step>
Output exactly one or more step blocks. Do not use Markdown code blocks.
```

强调 **不要使用代码块**，避免 Markdown 包裹导致提取多一层处理。

### 3.3 解析器实现

用正则提取每个 `<step>` 块，再提取子标签内容。Python 示例：

```python
import re

def parse_steps(text: str):
    # 提取所有 step 块，DOTALL 使 . 匹配换行
    pattern = re.compile(r"<step>(.*?)</step>", re.DOTALL)
    steps = []
    for match in pattern.finditer(text):
        block = match.group(1)
        step = {
            "id": extract_tag(block, "id"),
            "action": extract_tag(block, "action"),
            "params": extract_params(block),
            "reasoning": extract_tag(block, "reasoning").strip()
        }
        steps.append(step)
    return steps

def extract_tag(block, tag):
    m = re.search(f"<{tag}>(.*?)</{tag}>", block, re.DOTALL)
    return m.group(1) if m else ""

def extract_params(block):
    # 简单做法：提取 <params> 块，再从中提取所有子标签
    params_block = extract_tag(block, "params")
    param_pattern = re.compile(r"<(\w+)>(.*?)</\1>", re.DOTALL)
    return {m.group(1): m.group(2) for m in param_pattern.finditer(params_block)}
```

这种提取对缺漏容忍度很高，即使某个 `<step>` 缺少 `<reasoning>`，也不会影响整体解析。如果块内部有杂散字符，正则也能自动忽略。

### 3.4 针对流式输出的处理

在做 Agent 的流式消费时，可以维护一个缓冲区，一旦检测到 `</step>` 就把前面的完整块裁出处理。避免等待整个响应结束再解析，提升实时性。

---

## 4. 踩坑点与注意事项

### 4.1 标签冲突

模型偶尔会在 reasoning 文本中输出类似 `<step>` 的字符串（比如解释格式时）。可通过两个手段缓解：

- 使用罕见标签（如 `<OAC_STEP>`），并在提示中明确禁止在内容中输出该标签。
- 解析时只识别独立成行的标签，或在提取前先对内容做一次清理。

### 4.2 嵌套缺失

像 `<params>` 内部还有子标签，模型可能忘记关闭某个参数标签。解析时应当容忍未闭合标签，例如正则匹配只取已闭合的，或者使用逐行扫描，遇到下一个 `<step>` 就强制截断。

### 4.3 内容中的 `<` 和 `>`

如果 reasoning 中包含比较运算符（如 `a < b`），可能干扰解析。可要求模型对这类符号使用 Unicode 全角代替（`＜`）或 HTML 实体（`&lt;`），并在后处理中转回。实践中这类情况出现频率不高，但需要做好防御。

### 4.4 与 JSON Schema 强约束工具的对比

有些框架支持通过 JSON Schema 约束模型输出（如 vLLM guided decoding），虽然能降低 JSON 错误，但会显著增加延迟，且并非所有部署环境都支持。标签格式是一种无依赖、低成本的替代，兼容任何 OpenAI 兼容接口。

---

## 5. 可复用建议

- **先试 JSON，错了再降级**：可以在解析失败时用标签格式做 fallback，在提示中提供备选输出格式，并用解析器自动识别两种格式。
- **单一标签文件结构**：对于需要返回多个不同类型实体的场景，采用一个大标签包裹整体输出，内部用不同子标签区分，如 `<plan>`、`<summary>`、`<actions>`。
- **用校验而非修复**：写一个简单的格式校验规则，例如必须包含至少两个 `<step>` 块、id 必须是数字等，不合格则触发重试，但重试的提示中重复强调格式要求。
- **元信息前置**：将固定字段放在块的开头，可变自然语言放在后面，方便解析器快速定位。

---

## 6. 总结

JSON 在长文本结构化输出中的脆弱性源于它对字符级别精确性的高要求，而这恰恰是当前 LLM 的弱项。自定义标签格式牺牲了一些“通用性”，换来了极高的解析鲁棒性，尤其适合 Agent 开发中需要稳定编排多步骤任务、或 MCP 工具链中输出复杂指令的场景。它实现简单、不依赖额外库、对流式友好，是一种务实的工程权衡。

在 OpenClaw 生态里，我们经常需要让 LLM 生成可被后端稳定消费的指令。标签格式不仅减少了“坏 JSON”引发的故障，也让 Prompt 的编写更加自然——模型只需模仿示例中的块结构，不必记复杂的转义规则。如果你的 Agent 也在和长 JSON 斗智斗勇，不妨试试这种回归简单的方案。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/b3642044c8c4473e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/a7281d282f085bdc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/6ab1f9dc42448b1b.png)

