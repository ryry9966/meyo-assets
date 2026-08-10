---
title: LLM 输出解析的防御性编程：JSON 与标签格式混合处理
feedId: 32424
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景

在构建基于 LLM 的自动化流程时（无论是 MCP 工具调用、Agent 任务编排，还是插件化的提示链），我们经常让模型返回结构化的 JSON 数据。理想情况下，一个明确的 prompt 就能让模型输出纯净的 JSON。但工程实践中，模型的输出经常被包裹在 Markdown 代码块、XML 标签、甚至冗长的解释文本中。直接使用 `json.loads` 常常因为格式不纯而抛错，导致整个自动化链路中断。

当模型输出类似以下内容时，解析就会失败：

```
好的，这是您需要的结果：
```json
{"action": "search", "query": "OpenClaw agent"}
```
希望这对您有帮助。
```

因此，需要对 LLM 的 JSON 输出进行**防御性解析**，使其在面对多种混合格式时仍能稳定工作。

## 问题描述

在非严格 JSON 模式下，模型可能返回以下格式之一的“JSON”：

- 纯 JSON，但前后有空白、BOM 头或注释（`//` 行）
- Markdown 代码块：```` ```json ... ``` ````
- 自定义 XML 标签：`<json>...</json>` 或 `<output>...</output>`
- 首尾嵌有自然语言引导语或结束语，如“以下是结果：{...}，请继续。”
- 多个 JSON 对象连缀（不利于自动解析，但有时会出现）
- JSON 字符串内含有未转义的换行或引号，导致提取后的内容仍然非法

若每次都将输出直接丢给 JSON 解析器，流程将频繁因 `JSONDecodeError` 而失败。

## 防御性解析的工程做法

核心思路是**分层尝试提取合法 JSON 文本的暴力/启发式方法**，直到解析成功或所有策略都失败。我们用 Python 实现一个 `robust_json_parse` 函数，其处理流程如下（可复现的代码见文末）：

1. **预处理与直接解析**  
   去除首尾空白，若以 `{` 或 `[` 开头，直接尝试 `json.loads`。有时模型确实听话，直接解析即可。

2. **提取 Markdown 代码块**  
   用正则匹配 ` ```json ... ``` ` 或 ` ``` ... ``` ` 块，提取内部字符串后解析。有时模型中还存在多语言块（如 ` ```python `），需要只取 `json` 或无标注的块。注意代码块可能嵌套，最好用非贪婪匹配并取第一个。

3. **提取 XML 标签内容**  
   如果输出中包含 `<json>` 标签，提取标签间的文本。需要考虑可能的实体转义（如 `&quot;`），但实践中 LLM 很少输�� XML 转义，可先忽略。扩展时可支持任意自定义标签。

4. **首尾大括号定位提取**  
   如果上述都失败，寻找第一个 `{` 和最后一个 `}`，裁剪出子串解析。对数组同理，寻找第一个 `[` 和最后一个 `]`。这是一种有损但有效的兜底策略，但需要注意嵌套情况和字符串中可能的大括号干扰。为降低风险，可以使用括号平衡计数来找到最外层匹配的大括号。

5. **轻量修复**  
   若提取的字符串仍然无法解析，可以尝试一些低风险的自动修复：
   - 移除 JavaScript 风格的尾随逗号（如 `"key": value,}` 中的多余逗号）
   - 尝试用 `json5` 库解析（支持注释、单引号、尾随逗号等宽松格式），但需要额外依赖，视项目而定

6. **异常反馈与降级**  
   如果所有策略都失败，记录原始输出并抛出自定义异常或返回 `None`，由上层业务决定重试、回退模型或求助人工。日志中应保留截取过程信息，便于事后优化 prompt。

## 踩坑记录

- **嵌套大括号的误判**：直接截取第一个 `{` 和最后一个 `}` 可能得到跨多个对象的文本。应优先用括号计数器找到最外层的正确配对。
- **数组中的大括号**：采用同样的平衡计数法处理数组，而不是简单取首尾符号。
- **字符串内的代码块标记**：若 JSON 的某个字符串值中含有 `` ```json `` 文本，正则可能误提取。因此优先尝试直接解析，再考虑 regex 提取，且限定非贪婪匹配。
- **性能陷阱**：不要试图穷举所有可能的修复，尤其是递归修复。三层内无法解决，就交给错误处理流程。
- **BOM 与零宽字符**：有些模型输出可能带有 BOM 头，导致解码问题。可在预处理阶段自动移除。

## 可复用代码示例

```python
import json
import re

def robust_json_parse(text: str):
    text = text.strip().lstrip('\ufeff')  # 去除 BOM
    # 1. 直接解析
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass

    # 2. 提取 ```json 代码块
    patterns = [
        r'```json\s*(.*?)\s*```',
        r'```\s*(.*?)\s*```',
        r'<json>\s*(.*?)\s*</json>',
    ]
    for pat in patterns:
        m = re.search(pat, text, re.DOTALL)
        if m:
            try:
                return json.loads(m.group(1))
            except json.JSONDecodeError:
                continue

    # 3. 首尾大括号提取（带平衡计数）
    for left, right in [('{', '}'), ('[', ']')]:
        start = text.find(left)
        if start == -1:
            continue
        # 从 start 开始找匹配的右括号
        depth = 0
        end = start
        for i in range(start, len(text)):
            if text[i] == left:
                depth += 1
            elif text[i] == right:
                depth -= 1
                if depth == 0:
                    end = i
                    break
        if depth == 0:
            try:
                return json.loads(text[start:end+1])
            except json.JSONDecodeError:
                pass

    # 4. 可选：用 json5 库做最后一击
    try:
        import json5
        return json5.loads(text)
    except (ImportError, ValueError):
        pass

    raise ValueError("无法从输出中提取合法 JSON")
```

## 可复用建议

- **优先使用结构化输出**：如 OpenAI 的 `function_call` / `response_format` 模式、Anthropic 的工具调用等，能从源头大幅减少格式混乱，但防御性解析仍然是兜底的工程实践。
- **把解析器做成 Agent 的中间件**：在调用模型后、进入业务逻辑前，统一过一遍 `robust_json_parse`，让后续代码总是拿到干净的 dict/list。
- **记录原始输出和解析路径**：为每种策略的成功/失败打点，便于持续优化 prompt 和调节解析器阈值。
- **不要过度“修复”**：让解析器保持简单、可预测。复杂的模糊解析会掩盖 prompt 问题，增加维护成本。

## 总结

LLM 输出的不稳定性是工程化的天然挑战，而“JSON 与标签格式混合输出”是最常见的坑。通过分层的防御性解析，用直接解析、代码块提取、XML 标签提取和大括号匹配逐步降级，可以在绝大部分情况下稳定获得结构化数据。这个思路不仅适用于 JSON，也可扩展到 YAML、CSV 等其他格式。当你的 Agent 或 MCP 插件不再因为一个多余的 ```` ```json ```` 而停滞，你能更专注于业务逻辑本身。

---

