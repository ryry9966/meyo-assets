---
title: 牺牲 JSON，拥抱标签：让 LLM 长文本结构化输出更稳的务实方案
feedId: 29497
source: 综合讨论
publishedAt: 2026-07-18
---

## 背景：JSON 输出的隐性成本

在构造 OpenClaw Agent、MCP 工具链或自动化管线时，我们习惯让 LLM 返回结构化 JSON，以便直接喂给下一环节：解析参数、调度动作、提取关键块。对于短小、字段固定的场景（例如 `{"tool":"search","query":"..."}`），JSON 模式配合 constrained decoding 效果不错。但一旦要求输出**长文档、多层嵌套结构、带自然语言的混合内容**，坏 JSON 就开始了：

- LLM 生成长文本时很容易遗漏闭合大括号、方括号，或者多加一个逗号。
- 模型偶发的“解释性前缀”破坏完整结构，比如在 `{` 之前输出一句“好的，这是你要的 JSON：”。
- 字段值内含有未转义的双引号或换行符，导致标准解析器直接报错。
- 在经过多次对话轮次或思维链延长后，不完整/幻觉字段出现频率显著上升。

更麻烦的是，当这些 JSON 被放入自动化链路，你最终会在 Worker 日志里看到 `JSONDecodeError: Expecting ',' delimiter: line 47 column 18`，整条 pipeline 中断，需要额外写大量的修复正则和重试逻辑——而这些逻辑本身就不稳定。

## 问题根因：结构化生成 ≠ 结构化语言

LLM 本质上是一个序列预测器，它在 token 级别预测下一个最可能的 token。JSON 的结构要求精确的括号匹配、严格的语法，这与模型“输出最合理继续”的自然倾向相悖，尤其在生成长度很长、内容复杂时。模型在生成过程中并不“知晓”括号状态，除非借助 constrained sampling（比如 JSON 语法约束），然而即便有 constrained 方案，在处理自由文本字段时又限制了表达的灵活性，外部约束与内部自由文本混合时仍然容易产生冲突。

在实测 OpenClaw 任务分解插件时，我们需要模型输出一个包含 `rationale`、`subtasks`（列表，每个 subtask 又包含 `title`、`steps`、`expected_output`）的结构，且 `steps` 字段可能包含代码块和多行注释。强行要求 JSON 时，出错率高达 15%~20%，导致重试开销过大。

## 替代方案：基于标签的伪结构化输出

思路很简单：放弃严格的 JSON 容器，转而用**自定义标记（标签）作为逻辑分割符**，类似 XML 但不对模型要求 DTD 或严格嵌套。例如：

```
<task_breakdown>
<rationale>
这里包含自由解释，甚至可以有多行和列表。
</rationale>
<subtask>
<title>子任务1</title>
<steps>
- 第一步：xxx
- 第二步：xxx
</steps>
<expected_output>
预期输出的一段长文本，可能包含代码：
```python
print("hello")
```
</expected_output>
</subtask>
<subtask>
...
</subtask>
</task_breakdown>
```

这种格式对 LLM 而言更自然，因为它模拟了人类对“分节标题”的直觉，而且在生成时只需记住当前节的标签名称，闭合标签只是一个简单的 token 序列（如 `</subtask>`），远比 JSON 的 `}]}` 组合更不容易出错。实际测试中，复杂输出任务的格式正确率从 ~82% 提升到了 97% 以上，并且后处理解析非常快。

## 落地步骤与代码模式

### 1. 定义标签集并固定 Prompt 模板

在系统提示中显式给出标签结构和示例，并强调“只使用以下标签，不要添加额外解释”。避免使用可能与内容冲突的通用词，建议加项目前缀或唯一后缀。例如 OpenClaw 场景：

```
你的输出必须严格按以下标签结构组织，不要加任何外部文本：
<openclaw_response>
<reasoning>...</reasoning>
<plan>...</plan>
<actions>
<action name="...">
<params>...</params>
</action>
</actions>
</openclaw_response>
```

### 2. 后处理：用非贪婪正则提取

核心提取函数只需针对顶层标签，或每一级递归提取。注意使用 `re.DOTALL`，并采用非贪婪匹配避免跨标签串扰：

```python
import re

def extract_tag(content: str, tag: str) -> str | None:
    pattern = rf"<{tag}>(.*?)</{tag}>"
    match = re.search(pattern, content, re.DOTALL)
    return match.group(1).strip() if match else None
```

对于重复出现的同级标签（如多个 `<subtask>`），可以先用 `re.finditer` 逐块取出，再对每个块提取子标签。始终以 `<tag>...</tag>` 为原子单元，避免一次性正则横跨多个块。

### 3. 备用策略与容错

即使在标签方案下，仍可能遇到边界踩坑。我们在 OpenClaw 的 MCP 代理里实施了三级 fallback：

- **严格解析**：按完整标签提取，若成功则使用。
- **宽松解析**：只要求存在开始标签 `<tag>`，以该标签后内容直到下一个标签或文本结束作为内容，并忽略缺失的闭合标签。对某些场景（如 `reasoning`）损失部分尾部文本是可接受的。
- **原始全文**：以上均失败，则将整个输出作为原始文本传给容错处理逻辑（如降级成人工标记）。

### 4. 温度与长度惩罚调参

高温度（>0.8）会明显增加标签遗漏概率。对于结构要求严格的输出，建议温度设置在 0.2~0.5，并适当加大 `frequency_penalty` 避免同一 token 重复（比如一连串 `<`）。如果使用 vLLM 或本地模型，可开启 logprobs 监控闭合标签概率，低于阈值时触发重新采样。

## 踩坑实录：那些不算 bug 但恶心的细节

- **标签嵌套干扰**：当标签内内容本身包含类似标签的字符串（例如用户输入中包含 `<final_answer>`），会导致提前匹配。对策：要求模型对内容中出现的冲突符号进行转义，或使用更复杂的定界符如 `<|TAG_NAME|>...<|/TAG_NAME|>`，降低自然文本中出现的概率。
- **大小写与空格**：`<Subtask>` vs `<subtask>`，导致正则漏摘。统一在 prompt 中规定大小写，并在提取前先做一次 `lower()` 标准化（若标签唯一性允许）。
- **多轮对话累加**：当长对话中包含多个独立标签块时，建议每次都只解析最后一条 assistant 消息，或使用唯一 session 标签包裹单次输出，避免历史消息污染。
- **模型“补充说明”强迫症**：即使 prompt 明确禁止额外文本，某些模型仍会在最后添加“这是分解结果”。处理方式：在提取时对顶层标签做 `re.search` 而不是 `re.match`，忽略标签外的垃圾文本；或者直接裁剪到第一个 `<root_tag>` 和最后一个 `</root_tag>`。

## 可复用建议清单

1. **标签命名**：采用 `项目前缀_用途` 或唯一的符号包裹，如 `[[[ACTION]]]`，显著降低正文冲突概率。
2. **少样本示例**：在 prompt 中至少提供 2 个完整上下文示例，示范所有标签的正确使用，尤其嵌套场景。
3. **提取顺序**：永远从最外层标签开始提取，再递归进入内层，不要用一条大正则横跨整个文档。
4. **验证最小合约**：解析后快速检查必要字段是否存在，而不是只检查格式。例如，`actions` 标签下必须至少有一个 `action`，否则触发 fallback。
5. **记录坏样本**：将解析失败的原始输出持久化，每隔一段周期分析，以迭代改进 prompt 设计。

## 总结

标签格式不是银弹，它牺牲了严格的 schema 校验和跨语言通用性，但换来了极高的鲁棒性和 LLM 原生输出的亲和度。在 OpenClaw/Agent/MCP 这类以“自然语言驱动的过程控制”为主的工程场景里，**降级接受一种“看起来结构够用”的表示，比追求理论完美但频频中断的 JSON 管道更务实**。我们大量使用标签方案来包裹规划、推理、多步骤动作，再结合轻量的正则解析器，整套管道几乎不再因格式问题崩溃。当你的 JSON 解析异常开始吃掉开发时间，不妨认真考虑这个“后退一步”的方案。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/d0d64e860621d754.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/e9d4f5733cdd6643.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/e06bcfc6106f5a77.png)

