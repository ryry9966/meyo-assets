---
title: 防御性解析实践：当 LLM 输出同时带着 JSON 与 Markdown/标签
feedId: 32827
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景

在构建 OpenClaw Agent、MCP 工具或自动化流水线时，我们经常需要从 LLM 的回复中提取结构化的 JSON 数据——无论是工具调用参数、多步推理的中间结果，还是最终向用户展示的格式化信息。理想情况下，我们通过 function calling 或严格 prompt 约束就能拿到干净的 JSON，但现实里总有例外：

- 某些自部署模型不支持原生函数调用，只能靠 prompt 要求它“返回 JSON”。
- 即便支持 function calling，当模型输出被意外中断或处于“思考→行动”混合模式时，仍会出现非纯 JSON 文本。
- 工具调用结果需要人工审计，模型会在 JSON 前后附带解释、Markdown 代码块甚至 XML 标签。

如果解析逻辑只做了 `json.loads()`，那任意一点格式“污染”都会让整个自动化流程中断。防御性编程在这里不是过度设计，而是工程落地的刚需。

## 问题：一个典型故障现场

某次用 Agent 执行多工具编排，模型返回如下内容：

```
我需要先查询订单状态，然后再决定是否退款。
```json
{
  "tool": "query_order",
  "params": {
    "order_id": "ORD-20250321-001",
    "include_detail": true
  }
}
```
请稍等，我获取结果后继续。
```

上游解析代码直接 `json.loads(llm_response)`，结果抛出 `JSONDecodeError`，整个工作流卡死。这就是混合格式带来的最直接伤害：**模型输出的“辅助信息”破坏了 JSON 的纯净性**。

除此之外，还有几种常见变异：

- 直接 `{...}`，但 key 或字符串值中使用了单引号、尾逗号（JSON5 风格）。
- 包裹在 ````xml<tool_call>...</tool_call>```` 里，需要与 XML 解析结合。
- 多段 JSON 并列，比如先输出思考链，再输出工具调用，最后输出结果摘要。

## 做法：构建逐级降级的解析管道

我最终沉淀出一套防御性解析策略，不追求一次性完美解析，而是让解析器像剥洋葱一样逐级尝试，每一步都记录日志，便于回溯。

### 第一步：最严格解析

首先假设模型完全遵守约定，直接尝试解析纯 JSON。

```python
import json

def parse_llm_json(raw: str):
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        pass  # 进入降级
```

### 第二步：提取 Markdown 代码块

用正则匹配 ```` ```json ... ``` ```` 或 ```` ``` ... ``` ```` 中的内容。注意使用非贪婪匹配，并处理代码块内可能存在的其他反引号。

```python
import re

def extract_markdown_json(text: str) -> str | None:
    # 匹配 ```json ... ``` 或 ``` ... ```
    pattern = r'```(?:json)?\s*\n(.*?)\n\s*```'
    matches = re.findall(pattern, text, re.DOTALL)
    if matches:
        # 如果有多个代码块，默认取最长的一个，或根据上下文选择
        return max(matches, key=len).strip()
    return None
```

### 第三步：裸花括号截取

如果没有代码块，就尝试找到第一个 `{` 和最后一个 `}`，截取这一段。这是最“暴力”但很多时候有效的方法。但要小心截断字符串内嵌套的花括号，所以需要做简单的括号计数平衡。

```python
def extract_braced_json(text: str) -> str | None:
    start = text.find('{')
    if start == -1:
        return None
    stack = 0
    for i, ch in enumerate(text[start:], start=start):
        if ch == '{':
            stack += 1
        elif ch == '}':
            stack -= 1
            if stack == 0:
                return text[start:i+1]
    return None
```

### 第四步：宽松解析器修复

拿到候选 JSON 字符串后，不直接 `json.loads`，而是先通过 `json5` 或 `json_repair` 这样的库自动修复尾逗号、单引号、注释等问题，再尝试解析。

```python
import json5  # 或 from json_repair import repair_json

def try_repair_json(candidate: str) -> dict | None:
    try:
        return json5.loads(candidate)
    except Exception:
        try:
            from json_repair import repair_json
            return json.loads(repair_json(candidate))
        except Exception:
            pass
    return None
```

### 第五步：处理标签混合（XML/自定义标签）

当模型输出带有 `<tool_call>...</tool_call>` 或其他自定义标签时，先用简单的 XML 解析或正则提取标签内的文本，再将其扔回 JSON 解析管道。

```python
def extract_tag_content(text: str, tag: str) -> str | None:
    pattern = rf'<{tag}>\s*(.*?)\s*</{tag}>'
    match = re.search(pattern, text, re.DOTALL)
    return match.group(1).strip() if match else None
```

整个管道流程可以封装为：

```
raw_text
  → json.loads 成功？ → 返回
  → 提取 markdown json 代码块 → 成功？ → 返回
  → 提取标签内容（如 tool_call） → 成功？ → 再次进入管道
  → 提取花括号段 → 成功？ → json5/repair → 返回
  → 失败，抛异常或返回默认值（并记录raw_text到日志）
```

## 踩坑点

实施过程中遇到几个容易忽略的细节：

1. **字符串内反引号**  
   如果 JSON 的某个字符串值本身包含三个反引号，Markdown 提取正则会提前闭合，导致截取不完整。需要用更严格的边界检查或直接放弃 Markdown 提取，改用花括号平衡策略兜底。

2. **花括号截取误判**  
   当输出中存在多个 JSON 对象（比如先给一个参数列表，再给一个结果映射），取第一个对象还是最后一个？必须根据业务场景做判断。默认取第一个完整的 JSON 对象比较安全，但需确认是否符合预期。

3. **BOM 与不可见字符**  
   某些自部署模型会输出 BOM 头或零宽字符，导致 `json.loads` 第一字节失败。可以在入口处做 `text.strip('\ufeff')`。

4. **JSON5 修复的副作用**  
   `json5` 会把 `True` 解析成 `True`，但也会把 `null`、`NaN` 等非标准值合法化。如果后续流程依赖严格类型检查，需要额外验证。

5. **日志噪音**  
   每次降级都打 warning 日志，高并发下日志量剧增。建议只在最终失败时记录完整原始文本，中间降级用 debug 级别。

## 可复用建议

- **封装成独立工具函数**：`parse_llm_json(raw_text, tag=None)`，在组织内统一使用，迭代修复一处即可。
- **结合 prompt 约束，但不依赖**：可以在系统提示中明确要求“只输出 JSON，不要 Markdown 包裹”，但仍保留防御性解析。因为模型行为不可 100% 控制。
- **保留原始文本**：解析失败时，把原始响应存入日志或数据库，用于离线分析和 prompt 调优。
- **为 MCP 工具也加一层防护**：如果工具返回的是 LLM 生成的 JSON 结构，同样可以复用该解析器，降低下游调用方的崩溃风险。

## 总结

在 LLM 应用落地过程中，输出格式的不确定性不是 bug，而是特征。防御性解析不是对模型的“不信任”，而是对工程质量的起码尊重。这套多层降级管道经过数个 Agent 项目的打磨，已经帮我们避免了数十次深夜告警。如果你的自动化流程也正在被 JSON 解析错误折磨，不妨试试这种洋葱式的弹性设计——它不会让模型更听话，但会让你的系统更扛揍。

---

