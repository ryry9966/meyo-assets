---
title: LLM 输出解析防御实战：当 JSON 遇上 Markdown 代码块与标签
feedId: 32593
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景

在 OpenClaw、Agent、插件等自动化链路中，通过 LLM 生成结构化数据是常见需求。我们通常约定模型输出 JSON，下游直接 `json.loads()` 还原为字典并驱动业务流程。然而，真实运行中会遇到以下情况：

- 模型输出的 JSON 被包裹在 Markdown 代码块标记中（````json ... ````）。
- 输出同时包含非 JSON 的自然语言注释与 JSON 块。
- 某些场景约定使用 XML 风格标签（如 `<tool_call>`），但模型偶尔会混入 JSON。
- 输出包含未转义的换行、尾逗号、注释或单引号，导致 `json.loads` 直接报错。

这些问题在 Agent 面向终端用户或不可控的工具调用场景下尤其突出。一笔失败解析可能中断整个工作流，因此需要实施防御性编程，构建健壮的混合格式解析器。

## 问题剖析

典型的不规范输出形态：

1. **Markdown 包裹**  
   ```
   Here is your data:
   ```json
   {"name": "Alice", "age": 30}
   ```
   ```
2. **JSON 片段嵌入自然语言**  
   ```
   根据分析，结果是 {"score": 0.92, "label": "positive"}，请确认。
   ```
3. **混合标签格式**  
   同一输出中既有 `<function_call>` 标签，又包含 JSON 参数块。
4. **畸形的 JSON 内容**  
   尾随逗号、单引号字符串、注释（`// comment`）、换行不当。

常见的脆弱做法是直接用正则提取第一个 `{` 到最后一个 `}` 并 `json.loads`，这在嵌套对象、字符串内包含花括号时会错误截断。若同时使用标签与 JSON 混合约定，则需兼顾二者。

## 防御性解析方案

核心思路：分层解析，由严格到宽松，并辅以标签提取器，始终保留原始文本用于兜底。

### 步骤一：预处理，去除外层代码块

使用正则移除 Markdown 代码块标记，但仅移除最外层包裹，不损伤内嵌的 JSON 示例。

```python
import re

def strip_markdown_code_block(text: str) -> str:
    # 匹配 ```json ... ``` 或 ``` ... ```
    pattern = r"^```(?:json)?\s*\n(.*?)\n```$"
    m = re.search(pattern, text, re.DOTALL)
    if m:
        return m.group(1).strip()
    return text.strip()
```

对于多段落输出，不直接采用全文贪婪匹配，而是保留原始文本，防止丢失信息。

### 步骤二：首次尝试合法 JSON 解析

```python
import json

def try_strict_json(text: str):
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        return None
```

如果直接通过，说明输出本身即为纯净 JSON，无需额外处理。

### 步骤三：提取 JSON 候选子串并逐步修复

若严格解析失败，执行 JSON 块提取与修复流水线：

1. **提取最外层花括号/方括号**  
   使用平衡括号算法找到首个 `{` 或 `[` 并找到其匹配的结束位置，避免字符串干扰。
   ```python
   def extract_balanced_json(text: str):
       # 简化版：查找第一个 '{' 或 '['，用栈匹配
       start_idx = -1
       for i, ch in enumerate(text):
           if ch in '{[':
               start_idx = i
               break
       if start_idx == -1:
           return None
       stack = []
       for i, ch in enumerate(text[start_idx:], start_idx):
           if ch == '{' or ch == '[':
               stack.append(ch)
           elif ch == '}' or ch == ']':
               if not stack:
                   return None
               top = stack.pop()
               if (ch == '}' and top != '{') or (ch == ']' and top != '['):
                   return None
               if not stack:
                   return text[start_idx:i+1]
       return None
   ```

2. **常见自动修复**  
   在候选 JSON 上应用修复函数，提升解析成功率。
   - 移除尾随逗号：`re.sub(r",\s*([}\]])", r"\1", s)`
   - 替换单引号为双引号（注意处理转义）：可尝试 `json.loads` 捕获异常后逐字符处理。
   - 去掉 JavaScript 风格注释：`re.sub(r"//.*?\n|/\*.*?\*/", "", s, flags=re.DOTALL)`
   - 使用 `json.loads` 的 `strict=False` 以及允许控制字符。

如果修复后仍无法解析，可借助第三方库 `json-repair` 进行最后尝试。`json-repair` 能处理缺失引号、尾逗号等畸形 JSON。

```python
from json_repair import repair_json

try:
    repaired = repair_json(candidate)
    obj = json.loads(repaired)
except Exception:
    obj = None
```

### 步骤四：标签格式回退

如果 JSON 解析完全失败，或者业务约定同时支持标签格式，则执行 XML/标签提取。

例如 Agent 输出中存在 `<action tool="search">keyword</action>`，可用正则提取属性与内容：

```python
def extract_tag_info(text: str, tag: str):
    pattern = rf"<{tag}\b([^>]*)>(.*?)</{tag}>"
    matches = re.findall(pattern, text, re.DOTALL)
    results = []
    for attrs_str, content in matches:
        attrs = dict(re.findall(r'(\w+)="([^"]*)"', attrs_str))
        results.append({"attributes": attrs, "content": content.strip()})
    return results
```

甚至有的模型会同时输出 JSON 和标签，可设定优先级：JSON 成功则用 JSON，否则用标签结果，保证路径不中断。

### 步骤五：兜底与日志

如果所有手段都失败，返回原始文本并记录告警日志，交由人工或更高层的规则引擎处理。务必保留原始输出以便回溯。

## 踩坑点

- **平衡括号提取时忽略字符串中的花括号**：简单栈算法遇到 `{"text": "a{b}c"}` 会错误截断。必须在扫描时跟踪是否位于字符串内，识别转义引号。可简化为先尝试 `json.loads` 定位错误位置，或使用专业的 JSON 提取库。
- **Markdown 代码块的嵌套**：输出可能包含多个代码块，比如解释部分用了一个代码块举例，实际 JSON 在另一个代码块。选择最外层代码块不一定正确。更好的策略是提取所有代码块，逐个尝试 JSON 解析，返回第一个有效结果。
- **修复单引号时改变字符串内容**：单引号替换双引号可能破坏原本合法的字符串，如 `"it's"` 是合法内容，替换会变成 `"it"s"`。避免全局替换，应在解析失败后基于异常信息定位。
- **`json-repair` 不是银弹**：过度依赖可能导致解析出错误数据结构。仅在无法通过常规手段修复时使用，事后需校验必要字段是否存在。
- **标签属性解析的正则限制**：复杂属性值可能含空格或特殊字符，简单正则 `\w+="[^"]*"` 不够稳健，可改用简易 XML 解析器（xml.etree.ElementTree），需预先包裹根标签。

## 可复用建议

将上述逻辑封装为一个通用解析器 `robust_llm_parser`，参数包括原始文本、期望格式类型（JSON / XML）、必要字段列表。返回解析成功的结构化对象或原始文本。内部按优先级尝试：

1. 去除代码块标记
2. 严格 JSON
3. 平衡提取 + 自动修复 + json_repair 兜底
4. 标签提取（若需要）
5. 可选的字段验证，若缺失关键字段则降级

在企业内部多个 Agent 项目中复用此解析器，可显著减少因输出不规范导致的任务中断。

代码骨架参考：

```python
def robust_llm_parse(raw: str, required_fields=None) -> dict:
    clean = strip_markdown_code_block(raw)
    # 尝试直接JSON
    result = try_strict_json(clean)
    if result is not None:
        return validate_or_fallback(result, required_fields)
    # JSON提取与修复
    candidate = extract_balanced_json(clean)
    if candidate:
        repaired = repair_json(candidate)
        try:
            result = json.loads(repaired)
            return validate_or_fallback(result, required_fields)
        except:
            pass
    # 标签提取等兜底
    tag_data = extract_tag_info(clean, "action")
    if tag_data:
        return {"action": tag_data}
    # 最终兜底
    logging.warning(f"LLM parse failed, raw: {raw}")
    return {"raw": raw, "parsed": False}
```

## 总结

在 LLM 驱动的自动化链路中，永远不要假设模型输出严格符合约定格式。通过分层防御性解析，兼顾 JSON 与标签的混合场景，并采用从严格到宽松、从语法修复到兜底日志的策略，可以有效提升系统的鲁棒性。关键点：

- 先去除明显的外层 Markdown 包裹。
- 用平衡提取而不是贪婪正则来定位 JSON 片段。
- 逐步修复常见畸形，必要时引入 `json-repair`。
- 当 JSON 路径完全失效时，回退到标签提取。
- 保留原始输出用于排障，避免信息丢失。

这些工程化实践已在我维护的多个 Agent 工具链和 MCP 插件中被验证有效，希望能为 OpenClaw 社区的同学提供直接可用的参考。

---

