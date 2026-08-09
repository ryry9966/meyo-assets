---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 32294
source: 综合讨论
publishedAt: 2026-08-10
---

## 为什么需要防御性解析

在 OpenClaw、Agent、MCP 这类自动化流程中，LLM 通常被要求输出结构化数据，例如 JSON。理想情况下，返回的内容应该是纯净的 `{"key":"value"}`，可以直接送入 `json.loads()`。但生产环境中，模型的输出远没有那么规矩：

- 前后附着解释性文本：“好的，以下是结果：”
- 包裹在 Markdown 代码块中：`` ```json ...``` ``
- 混入 XML 标签：`` <response>{"tool":"search"}</response> ``
- 甚至多段 JSON 夹杂在一起

如果解析层只做一次 `json.loads()`，大量合法但不规整的输出会导致流程硬中断。面向自动化实践，必须建立一套防御性解析策略，让系统尽可能从“脏”输出中提取出可用的结构化载荷。

## 常见“脏”JSON 形态与解析思路

实践中观测到的典型模式包括：

1. **纯 JSON，无干扰**  
   直接反序列化即可，这是最理想的情形。

2. **Markdown 代码块包裹**  
   ```` ```json\n{...}\n``` ```` 或 ```` ```\n{...}\n``` ````。  
   策略：用正则提取代码块内的内容。注意要兼容 `json`、`javascript`、无语言标识等情况。

3. **前置/后置说明文本**  
   “以下是搜索结果：`{"items":[]}`。你可以继续提问。”  
   策略：定位第一个 `{` 和最后一个 `}` 之间的内容，尝试解析。

4. **标签混合**  
   `<response>{"result":"ok"}</response>` 或 `<tool_call>{"name":"search","arguments":{...}}</tool_call>`  
   策略：先剥离已知标签，再按前述逻辑提取 JSON。

5. **多层嵌套/转义**  
   LLM 输出的 JSON 内部可能包含它自己生成的代码块，例如 `{"code":"```python\nprint(1)\n```"}`。如果简单使用正则提取代码块，可能错误截断。此时需要更智能地平衡检测或先按完整 JSON 解析。

## 实现步骤

以 Python 为例，封装一个 `robust_json_extract` 函数，按照优先级依次尝试：

```python
import re
import json

def robust_json_extract(text: str):
    # 1. 直接解析尝试
    try:
        return json.loads(text.strip())
    except json.JSONDecodeError:
        pass

    # 2. 去除常见 XML 包裹标签
    cleaned = re.sub(r'</?(\w+)(\s+[^>]*)?>', '', text)

    # 3. 提取 Markdown 代码块（兼容 json/javascript 或无标识）
    code_block_pattern = re.compile(r'```(?:json|javascript|js)?\s*\n?(.*?)\n?```', re.DOTALL)
    matches = code_block_pattern.findall(cleaned)
    for match in matches:
        try:
            return json.loads(match.strip())
        except json.JSONDecodeError:
            continue

    # 4. 尝试提取最外层的 JSON 对象
    # 找第一个 '{' 和最后一个 '}' 之间的内容
    start = cleaned.find('{')
    end = cleaned.rfind('}')
    if start != -1 and end != -1 and start < end:
        json_candidate = cleaned[start:end+1]
        try:
            return json.loads(json_candidate)
        except json.JSONDecodeError:
            pass

    # 5. 如果仍未成功，尝试提取第一个 JSON 数组
    start = cleaned.find('[')
    end = cleaned.rfind(']')
    if start != -1 and end != -1 and start < end:
        json_candidate = cleaned[start:end+1]
        try:
            return json.loads(json_candidate)
        except json.JSONDecodeError:
            pass

    raise ValueError("无法从 LLM 输出中提取有效 JSON")
```

这段代码不做任何内容修复（如补逗号），只围绕格式做清理。理由：修复语义错误会让行为变得不可预测，而且错误内容应当被上层捕获后反馈给模型重试。

## 踩坑点

### 1. 流式输出下的切割问题
如果你使用 SSE 流式接收 LLM 输出，代码块标记 ` ``` ` 可能在 chunk 边界被拆分。例如：

> chunk1: ` ```json\n{"na`
> chunk2: `me":"test"}\n` ````

在实时解析场景中，需要将 chunk 缓存拼接到足够的长度，或使用状态机检测代码块开始与结束，否则正则提取会失效。建议对延迟不敏感的流程先完整收集再解析；必须流式解析时，维护一个缓冲区和当前代码块边界状态。

### 2. 正则 `.*?` 非贪婪匹配的边界问题
模式 ```` ```(?:json)?\s*([\s\S]*?)\s*``` ```` 在遇到 JSON 内嵌的 ` ``` ` 时可能提前截断。例如：

```
```json
{"code":"```python\nprint(1)\n```"}
```
```

内部的三反引号会导致匹配过早结束。**缓解方法**：优先直接尝试解析整段文本，只在直接解析失败时才用正则提取，并接受可能误截的场景；或使用更复杂的嵌套检测（从第一个 ` ``` ` 开始计数，平衡寻找结束标记）。

### 3. 多重 JSON 并存
有时 LLM 会输出多个候选 JSON，常见于模拟工具调用列表。当前实现只提取第一个对象/数组。如果你的场景需要提取全部，可修改为 `re.finditer` 遍历所有候选，而非仅返回第一个。注意，这可能引入虚假匹配，需要验证每个候选是否合法。

### 4. 转义字符不一致
偶见 `\"` 变成 `\\"` 或直接输出单引号 JSON。标准库 `json.loads` 不接受单引号，如果业务允许，可尝试使用 `ast.literal_eval` 或 `json5` 库做宽泛解析，但同样会降低刚性，应谨慎评估。

## 可复用的工程建议

- **封装为独立函数并添加日志**  
  每次尝试失败时记录尝试策略，方便后期分析模型行为。例如：“使用代码块提取失败，尝试首尾截取”。
- **二次防御：解析失败回退**  
  如果最终解析失败，不要直接抛出异常让流程崩溃。可以将错误信息（包括原始输出片段）通过你框架的错误处理回调发送回 LLM，让模型重新生成（例如 `"Your last output was not valid JSON. Please retry with only the JSON object."`）。这在 Agent 循环中非常有效。
- **提示词层面减少脏输出**  
  尽管不能百分之百依赖，但在系统提示中加入：“请 **只** 输出 JSON，不要使用 Markdown 代码块，不要添加任何解释文字。” 可以显著降低混合格式的比例。
- **为插件/MCP 工具提供标准解析器**  
  OpenClaw 插件作者可以将此解析函数作为内置工具提供给用户，避免每个动作步骤都重复造轮子。

## 总结

LLM 输出解析的防御性编程不是过度设计，而是自动化系统保持稳健运行的必要实践。正视 LLM 输出的多变特性，通过分策略尝试提取、容忍常见干扰，能大幅降低解析失败导致的流程中断。核心思路是：先清洁后解析，由严到宽，捕获失败时利用反馈机制自修复。将这套逻辑固化为通用函数，可显著提升 OpenClaw、Agent 以及 MCP 生态中各类工具调用的可靠性。

---

