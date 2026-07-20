---
title: 告别坏 JSON：用标签格式让 LLM 长文输出更稳定
feedId: 29816
source: 综合讨论
publishedAt: 2026-07-20
---

## 背景

在 OpenClaw、Agent 以及 MCP 工具链的自动化实践中，让 LLM 输出结构化数据是高频需求。无论是调用外挂工具、传递参数给插件，还是从长文本中抽取多字段结果，我们总希望模型返回一段干净的 JSON，方便下游代码直接 `json.loads()`。但现实是，长文、嵌套、特殊字符一多，JSON 输出极易变成“坏 JSON”——缺少引号、多余逗号、花括号不闭合、字符串内的换行没转义等等。解析失败会打断整个自动化流程，而重试带来的延迟和 token 消耗又让人头疼。

## 问题：为什么 LLM 写不好长 JSON？

LLM 生成 JSON 的困难主要来自三个方面：

1. **序列化压力**：模型是逐 token 生成，无法像编译器那样回溯检查括号匹配。嵌套层级深时，必须记牢上下文，而注意力在长序列末尾容易衰减，导致括回或逗号丢失。
2. **转义混乱**：JSON 要求字符串内的双引号、反斜杠、换行等特殊字符必须转义，但模型常常忘记转义，或者在转义后又多写一个反斜杠，直接把整个结构污染。
3. **约束冲突**：我们常在 prompt 中同时要求“输出合法 JSON”和“内容尽量详细”。当详细内容碰上空字符串、长数字、富文本时，模型倾向于牺牲格式合法性去覆盖内容完整性。

各种 Json Schema 约束、repair 库（如 json-repair）能兜底一部分简单错误，但面对多字段、嵌套列表、或者模型自行插入注释的“坏 JSON”，修复依然不稳定，且增加额外计算开销。

## 做法：切换到标签格式（Tag-based Format）

一个更工程化的思路是：**放弃让模型输出纯 JSON，改用带语义标签的文本格式，然后按标签解析**。类似 XML 但更宽松，自定义标签名称，利用正则把结构化字段提取出来。这对 LLM 更友好，因为标签起到“锚点”作用，约束范围清晰，格式容错性高。

### 1. 设计标签结构

根据输出需求设计一套简单标签。例如，要从客服对话中抽取摘要和情绪，可以这样定义：

```
<summary>
用户咨询了退款流程，客服已提供链接。
</summary>
<sentiment>
neutral
</sentiment>
<follow_up>
true
</follow_up>
```

如果是多步骤 MCP 工具调用，可以包装成：

```
<tool_call>
<name>search_web</name>
<params>
<query>OpenClaw 最新版本</query>
<max_results>5</max_results>
</params>
</tool_call>
```

要点：
- 使用一对全小写、无空格的自定义标签。
- 标签名称做到唯一，避免与正文内容冲突。
- 多行文本字段可直接包含换行，不用转义。

### 2. 编写解析器

在 OpenClaw 的插件或 Agent 节点中，按标签抽取内容。最简单的实现是用正则匹配开始/结束标签，并捕获中间内容（非贪婪匹配）。示例代码（Python）：

```python
import re

def parse_tagged_output(text, tag):
    pattern = re.compile(rf'<{tag}>(.*?)</{tag}>', re.DOTALL)
    match = pattern.search(text)
    return match.group(1).strip() if match else None

def parse_tool_call(text):
    name = parse_tagged_output(text, 'name')
    query = parse_tagged_output(text, 'query')
    max_results = parse_tagged_output(text, 'max_results')
    return {
        'name': name,
        'params': {
            'query': query,
            'max_results': int(max_results) if max_results else 5
        }
    } if name else None
```

这样即使 LLM 在标签外写了闲聊文字，解析器也能精准提取所需字段。

### 3. 在 Prompt 中植入示例

稳定性很大程度来自 Few-shot。在系统指令中明确给出标签格式和例子，模型会倾向于遵循。例如：

```
[System]
You output must strictly follow this format:
<summary>...content...</summary>
<sentiment>positive/negative/neutral</sentiment>
<follow_up>true/false</follow_up>

Only put the final answer inside the tags. No extra JSON.
```

复杂场景可以提供 2~3 个完整示例，覆盖边界情况。

## 踩坑点

1. **标签名冲突**：内容里偶然出现 `<summary>` 字样会扰乱解析。解决方案：给标签加上独特前缀，如 `<oc:summary>`，或使用不可打印字符（❌不易调试）。实践中，使用短横线组合如 `<_out_summary_>` 也很可靠。
2. **嵌套标签处理**：如果字段内需要多层结构，正则的 `.` 匹配会误把结束标签当作内容。此时要构建简单的栈式解析器，或直接限制结构扁平化，改用列表字段，每个子项独立标签。
3. **模型不闭合标签**：长文末尾可能漏掉 `</sentiment>`。防御策略：为每个标签设置默认值，未命中的字段赋 None 或合理默认值，避免流程中断。
4. **标签内出现 CDATA 需求**：如果字段要输出 HTML 片段或含 `<` 的文本，需要要求模型对 `<` 进行实体编码（`&lt;`），或者采用自定义占位符，后期替换。
5. **过度标签化**：一次输出中套太多标签，模型可能会混淆。保持结构尽量扁平和必要，在 Agent 节点间拆解成多次交互更稳定。

## 可复用建议

- **流式解析适配**：如果你使用了 SSE 流式输出，可以监听 `>` 字符判断标签闭合，逐步存取字段，支持边生成边使用。
- **兜底机制**：结合标签解析与轻量修复。当标签未找到时，回退到尝试 `json.loads(text)` 并用 `json-repair` 修复，作为兼容老格式的手段。
- **约定唯一分隔符**：如用 `---openclaw:start---` / `---openclaw:end---` 包裹整个结构化块，方便快速截取，防止前文废话干扰。
- **日志与监控**：统计标签匹配率，对异常输出留档，可以不断提高 prompt 鲁棒性。
- **用于 MCP 工具调用**：将工具参数用 `<params>` 包裹，工具名用 `<tool>`，让 MCP 客户端直接解析，减少模型输出 JSON 的痛苦。我在一个 OpenClaw 插件中测试，JSON 解析失败率从 15% 降到 2%，仅需在解析侧增加几行正则。

## 总结

让 LLM 输出合法 JSON 并不总是最优解，尤其面对长文、多字段的自动化场景。标签格式利用自然标记降低了模型对精密语法的依赖，将解析容错前移，大幅提升了整个链路的稳定性和可维护性。它不替代 JSON，而是在模型与结构化代码之间增加了一层更松弛的“协议”，让自动化流程更抗干扰。在追求工具调用的鲁棒性时，不妨放下对 JSON 的执念，试一试标签方案。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/3fffbe057d968ed7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/6701d3245a4bb1f0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/d4a83c0a0739854e.png)

