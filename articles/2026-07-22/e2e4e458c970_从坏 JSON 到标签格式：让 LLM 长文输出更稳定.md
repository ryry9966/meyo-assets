---
title: 从坏 JSON 到标签格式：让 LLM 长文输出更稳定
feedId: 30034
source: 综合讨论
publishedAt: 2026-07-22
---

## 背景

在构建基于 OpenClaw 的自动化链或 Agent 时，我们需要让 LLM 输出结构化内容，供下游工具调用或下一步推理使用。JSON 自然成了首选格式——它有成熟的解析库、易于人类阅读、并且几乎每种语言都支持。问题出在：当输出变长、嵌套层级变多，或者模型需要在“思考过程”中穿插结构时，JSON 的脆弱性就暴露出来了。

我最初跟很多人一样，用 `response_format` 强约束模型输出 JSON，但一旦要求模型先输出一段分析再给出工具调用，或者回答内容超过 2k token，JSON 就容易出现残缺。结果就是上游解析器频繁报错，整个流程卡死。

## 问题：JSON 为什么容易“坏”？

在长文输出场景中，我遇到的典型失败模式包括：

1. **截断**：达到 `max_tokens` 时 JSON 缺少闭合括号，整个对象无法解析。
2. **嵌套引号与转义错误**：模型在字符串中嵌入示例代码或对话，引号未正确转义，导致解析器提前结束字符串。
3. **注释与多余文本**：模型有时会在 JSON 前后附加解释性文字（如 "Here is the JSON:"），或者在对象内部添加 `// comment`，标准 JSON 都不允许。
4. **尾逗号与拼写错误**：LLM 偶尔会犯人类写 JSON 时的常见错误，比如数组末尾多余的逗号。

在 Agent 循环里，每一次解析失败都意味着需要重试，浪费算力与时间。即使使用重试逻辑，也不能从根本解决问题，因为模型在长上下文下的输出分布并不可靠。

## 做法：切换到基于标签的格式

一个简单且工程上实用的方案是放弃纯 JSON，改用**自定义标签对**包裹不同区块。这种格式类似于 XML 或 HTML，但规则更宽松，解析更健壮。例如：

```
<analysis>
这个任务需要先查询用户余额，然后决定是否放款。
</analysis>
<tool_call>
<name>query_balance</name>
<params>
<user_id>42</user_id>
</params>
</tool_call>
```

这种格式的优势在于：

- **允许部分恢复**：即使输出被截断，我们仍可提取已完成的片段。例如缺少闭幕标签的 `<tool_call>`，仍可以稍后从重试的输出中补全。
- **容忍注释与多余文本**：标签之外的杂散文字可以忽略或记录为日志。
- **标签名即语义**：解析时直接知道当前块的含义，不需要遍历整个结构。

## 实施步骤

### 1. 设计标签 schema

根据业务需要定义标签集合。常见的是 `<thought>`、`<action>`、`<params>`、`<content>` 等。避免与 markdown（如 `<code>`）或通用 XML 冲突，建议加上项目前缀，比如 `<o-thought>`。

### 2. 编写流式解析器

简单的基于正则的“轻解析器”就能够工作。核心思路：用 `re.finditer` 扫描标签头和闭合标签，建立一个栈，同时收集文本内容。示例 Python 代码：

```python
import re

TAG_RE = re.compile(r'<(/?)(\w[\w.-]*)>')

def parse_tags(text: str):
    stack = []
    pos = 0
    blocks = []
    for m in TAG_RE.finditer(text):
        tag_text = text[pos:m.start()]
        if stack:
            stack[-1]['text'] += tag_text
        is_close = bool(m.group(1))
        tag_name = m.group(2)
        if not is_close:
            # 开始标签
            stack.append({'tag': tag_name, 'text': ''})
        else:
            if stack and stack[-1]['tag'] == tag_name:
                block = stack.pop()
                blocks.append({**block, 'full': f'<{tag_name}>{block["text"]}</{tag_name}>'})
            # 不匹配则忽略，避免非法结构卡死
        pos = m.end()
    # 剩余尾部文本
    tail = text[pos:]
    if stack:
        stack[-1]['text'] += tail
        # 未闭合块，按不完整块处理
        blocks.append({'tag': stack[-1]['tag'], 'text': stack[-1]['text'], 'full': f'<{stack[-1]["tag"]}>{stack[-1]["text"]}'})
    return blocks
```

这是一个不会抛异常的宽松解析器，它把每个闭合的标签块收集为字典，未闭合的也尽量保留，以供下游决策（如重试）。

### 3. 集成到 Agent 循环

- 在 prompt 中明确告知模型输出格式，给出一至两个完整示例。
- 使用 `stop_sequences` 而非强 structure，让模型自由生成，但通过提示设计引导标签使用。
- 获得原始输出后，用解析器提取块。如果必要块缺失，则进入重试逻辑，但此时只需要补全缺失部分，而不是重新生成整个响应，节省 token。

### 4. 错误处理与重试

当 `<tool_call>` 未被正确关闭时，我采取“追加补全”策略：向模型发送一条简短消息让模型仅补足闭合标签部分，然后合并解析。由于模型擅长续写，这种局部修复的成功率远高于重新生成完整 JSON。

## 踩坑点

1. **标签冲突**  
   如果用户在对话中帖了 XML 或 HTML 代码，可能会干扰解析。解决方案一是使用带前缀的标签名；二是在标签内容内使用类似 CDATA 的机制，例如利用 markdown 的 code fences 包裹，然后在解析前将 code fence 内容临时替换为占位符。

2. **嵌套标签**  
   若 schema 要求嵌套（如 `<params>` 内部又有条件块），解析器需要正确处理嵌套栈。这需要更严格的正则或使用简单的递归下降解析。避免深层嵌套可以从设计上简化——尽量扁平化，比如用 `<param key="user_id">42</param>` 代替多层嵌套。

3. **流式解析与不完整输出**  
   在 SSE 流式场景下，需要标记“块是否闭合”。上例解析器已考虑了未闭合情况。当收到流式 chunk 时，可以累积到一定长度后尝试解析，保留碎片用于下一轮拼接。

4. **转义**  
   严格来说，用户可能输入 `</tag>` 文本。接受这一点的“优美”方案是仿照 XML 使用 `&lt;` 和 `&gt;` 并要求模型转义，但 LLM 对此遵从度通常不高。一个务实做法是假设内容中不会出现标签字符串，并在文档中建议用户避免使用类似格式；如果必须包含，则使用 Base64 编码或将其放在 code block 中。

## 可复用建议

- **模式与库**：可以将上述解析逻辑封装为一个轻量级 Python 库，提供 `parse_tags(text, schema_validation=False)` 和 `block_to_dict(block)` 等助手。schema 验证仅用于生产关键路径。
- **结合重试策略**：不完整块触发指数退避重试，最多两次，之后上报人工。
- **日志与监控**：记录每次解析的块数量与未闭合块数，用于观察模型行为变化。
- **渐进迁移**：不一定全盘替换 JSON。可以在 prompt 中优先输出标签格式的分析段落，然后关键数据仍用 JSON，作为过渡。

## 总结

从纯 JSON 到标签格式，并非要放弃结构化输出，而是选择一种更贴合 LLM 输出特质的方式。长文本场景下，标签格式天然具备“部分可用”和“容错解析”两种特质，极大地降低了因格式错误导致的中断成本。在 OpenClaw 这类自动化 Agent 中，它提升了流程的健壮性，让模型能用更自然的“内心独白”进行推理，同时仍能把数据块可靠地传递到后续节点。

如果你也在为 JSON 解析频繁报错而头疼，不妨试着引入自定义标签，先从工具调用区块开始替换，观察效果。工程上的“完美格式”远不如一个鲁棒的容错方案来得实在。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/01b6cc716ba0b5dc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/79dae93159ff8e51.png)

