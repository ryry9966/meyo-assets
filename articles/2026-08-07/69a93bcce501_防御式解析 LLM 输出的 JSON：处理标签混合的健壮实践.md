---
title: 防御式解析 LLM 输出的 JSON：处理标签混合的健壮实践
feedId: 31993
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

在构建基于 LLM 的 Agent、MCP 工具或自动化流程时，我们几乎都会要求模型返回结构化数据——最常见的就是 JSON。然而模型输出远非干净的 `{"key":"value"}`，它往往包裹着 Markdown 代码块标记、前置的解释性文字，甚至被截断。直接在管道的下一步调用 `json.loads()`，会使整个自动化链路脆弱不堪。

Agent 与自动化场景对稳定性的要求极高：一次解析失败可能导致整个 workflow 终止，或让下游插件吃进错误指令。

## 核心问题

LLM 输出的典型“混合格式”包含：

- ```` ```json ```` 包裹的代码块，周围有说明文字
- 仅 ```` ``` ```` 包裹（缺语言标识）
- 输出以 `{` 开始，但末尾多出一段无关心得
- 模型生成的 JSON 内部字符串未正确转义
- 多对象（如 `{...}{...}` 或 `[{...},{...}]`）
- 尾部多余逗号或单行注释（JSON 标准不允许）
- 输出被 truncation 截断，导致括号不配对

严格解析器会直接崩溃，而我们需要的是一个**容忍度高**的提取器。

## 防御性解析策略与步骤

下面是一种在实践中迭代出来的健壮解析流程，兼顾性能与容错。示例用 Python 3.10+。

### 1. 先尝试直接解析

```python
import json

def defensive_json_parse(text: str):
    # 尝试直接解析
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass
```

不要忽略这个步骤：有些模型输出恰好就是裸 JSON，直接解析最快。

### 2. 提取 Markdown 代码块

大多数情况是 ```` ```json ... ``` ```` 包裹。用正则非贪婪提取：

```python
import re

def extract_code_block(text: str, lang: str = "json"):
    pattern = rf"```{lang}\s*(.*?)```"
    match = re.search(pattern, text, re.DOTALL)
    if match:
        return match.group(1).strip()
    # 尝试没有语言标识的代码块
    match = re.search(r"```\s*(.*?)```", text, re.DOTALL)
    if match:
        return match.group(1).strip()
    return None
```

提取后再次尝试 `json.loads()`。

### 3. 查找第一个 `{` / `[` 到最后一个配对的 `}` / `]`

如果仍失败，说明输出可能被文字包裹但不是规范的代码块。此时定位最外层的括号结构：

```python
def extract_bracket_region(text: str):
    # 寻找第一个 { 或 [
    start_idx = min(
        (text.find("{") if text.find("{") != -1 else float("inf")),
        (text.find("[") if text.find("[") != -1 else float("inf"))
    )
    if start_idx == float("inf"):
        return None
    # 从该位置向后扫描，计数括号，找到配对的结束
    open_brackets = {"{": "}", "[": "]"}
    close_brackets = {"}": "{", "]": "["}
    stack = []
    for i in range(start_idx, len(text)):
        ch = text[i]
        if ch in open_brackets:
            stack.append(ch)
        elif ch in close_brackets:
            if stack and stack[-1] == close_brackets[ch]:
                stack.pop()
                if not stack:
                    return text[start_idx:i+1]
    # 可能未闭合，根据需要尝试修复（见下文）
    return text[start_idx:]  # 返回到末尾，后续可补括号
```

提取后尝试解析。

### 4. 容忍非标准 JSON

有些模型模仿人类书写，在 JSON 中加入注释或尾逗号。可以用预清洗函数：

```python
import re

def sanitize_json_like(text: str):
    # 移除单行注释 //...  (注意不要误删字符串内的 //)
    # 简单处理：假设注释不出现在字符串内
    text = re.sub(r'//.*', '', text)
    # 移除尾随逗号（在 } 或 ] 之前）
    text = re.sub(r',\s*([}\]])', r'\1', text)
    return text
```

如果项目允许，直接引入 `json5` 库，它原生支持注释和尾逗号。这比自己实现更安全。

### 5. 修复截断的括号

如果输出因 token 限制被截断，且我们已获得大部分 JSON，可尝试补充缺失的闭合括号：

```python
def fix_truncated_json(text: str):
    # 补充缺失的 } 或 ]
    open_counts = {"{": 0, "[": 0}
    close_counts = {"}": 0, "]": 0}
    for ch in text:
        if ch in open_counts: open_counts[ch] += 1
        elif ch in close_counts: close_counts[ch] += 1
    missing_braces = open_counts["{"] - close_counts["}"]
    missing_brackets = open_counts["["] - close_counts["]"]
    return text + "}" * missing_braces + "]" * missing_brackets
```

> 谨慎使用：这会使畸形的 JSON 看似合法，但可能导致下游逻辑错误。建议额外打日志并触发 Agent 重试。

### 完整的防御性解析器骨架

```python
def robust_json_parse(text: str):
    log = {}
    # 直接解析
    try:
        return json.loads(text)
    except Exception:
        pass

    # 提取代码块
    extracted = extract_code_block(text)
    if extracted:
        try:
            return json.loads(extracted)
        except Exception:
            pass

    # 括号区域提取
    region = extract_bracket_region(text)
    if region:
        cleaned = sanitize_json_like(region)
        try:
            return json.loads(cleaned)
        except Exception:
            pass
        # 尝试修复截断
        fixed = fix_truncated_json(cleaned)
        try:
            result = json.loads(fixed)
            log["warning"] = "truncated JSON repaired"
            return result
        except Exception as e:
            log["error"] = str(e)

    # 全部失败则抛异常或返回默认值
    raise ValueError(f"JSON extraction failed. Log: {log}")
```

## 踩坑点

- **JSON 内部字符串含 Markdown 代码块分隔符**：极少数情况下，模型输出内容中的字符串包含 ` ``` `，此时正则提取代码块会提前截断。可以通过先统计反引号出现的位置，或要求模型避开特殊分隔符来缓解。
- **多对象输出**：有时模型返回 `{...}{...}` 是连续的，但标准解析器只认第一个。此时应根据业务决定是只取第一个，还是用 `json.JSONDecoder` 的 `raw_decode` 逐个解析。
- **`json5` 库的副作用**：它会把 `Infinity`、`NaN` 等转为 Python 的 `float('inf')`，这在某些下游序列化时会再次出错，需要额外转换。
- **性能开销**：防御性解析链会多次尝试 `json.loads` 和正则，但相比 LLM 调用的延迟，这些开销可忽略。建议只在失败时逐步尝试，避免在正常路径上拖慢。

## 可复用建议

1. **统一封装**：在项目内构建一个 `llm_json_parse` 工具，所有从 LLM 出口获取结构化的地方都调用它，不要散落各处的特判。
2. **结合重试逻辑**：解析失败时，记录原始输出并让 Agent 重新生成，通常第二次结果更规范。不要仅依赖修复。
3. **结构化日志**：记录原始字符串长度、是否命中代码块提取、是否修复等，便于持续优化 prompt 或调整解析阈值。
4. **在 prompt 中约定分隔符**：例如“请将 JSON 放在两个 `%%%JSON_START%%%` 和 `%%%JSON_END%%%` 之间”，比代码块更不容易出现在内容里。
5. **防御性设计不是万能药**：严重畸形的输出（如结构错误）无法靠自动修复恢复合法语义，此时应安全降级，避免静默错误。

## 总结

让 Agent 稳定运行的关键往往不在模型本身，而在那些不起眼的“胶水”部分。防御性 JSON 解析就是典型例子——用一小段可靠的中介代码，吸收 LLM 输出格式的波动。上述策略经历了多个自动化任务的打磨，从简单的代码块提取到括号修复，形成了一条由快到慢、由通用到激进的 pipeline。将其封装为团队的基础设施，可以让所有人专注于业务逻辑，而不是与模型的格式博弈。

在工程上，对输出的宽容往往比强制模型遵循规范更实际。

---

