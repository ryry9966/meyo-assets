---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 35139
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw、Agent、MCP、插件和自动化链路里，很多地方会要求模型返回结构化数据，最常见的是 JSON。比如 MCP 工具调用的参数、插件配置、工作流节点的数据交换，往往要求模型输出：

```json
{"tool": "search", "args": {"q": "openclaw"}}
```

但真实模型输出不会总是这么干净。它们可能加代码块、中文说明、注释、自定义标签，甚至把 JSON 包裹在 XML 标签里。如果我们直接在工具侧 `json.loads`，链路很容易因为一条解析异常整体中断。

防御性编程的目标不是“把模型的输出洗得完美”，而是让解析层在可预期范围内兼容脏格式，同时保留原始错误信息，方便后续回归和调模型。

## 问题：脏输出长什么样

常见的混合格式大致有这几类：

1. 带代码块：

```
```json
{"tool":"search","args":{"q":"x"}}
```
```

2. 前后夹杂自然语言：

```
好的，以下是结果：
{"tool":"search","args":{"q":"x"}}
```

3. 标签包裹：

```
<json>
{"tool":"search","args":{"q":"x"}}
</json>
```

4. 带注释或尾随内容：

```json
{"tool":"search","args":{"q":"x"}} // 注意这里
```

5. 多个 JSON 片段，其中只有第一个是目标结构。

这些情况直接 `json.loads` 会全部失败。如果用粗糙正则，又容易把嵌套花括号、字符串里的特殊符号误判。

## 做法：分层解析，而不是一层正则

我建议把解析收敛到一个入口函数，比如 `parse_llm_json`，所有需要结构化输出的地方都走这里。

处理顺序可以这样设计：

### 1. 基础清洗

先处理 BOM、零宽字符、全角标点、中文引号和弯引号。模型有时会把 `"` 转成 `“”`，或者把逗号写成全角逗号，这会让标准 JSON 直接失败。

```python
def clean_text(text: str) -> str:
    text = text.replace("\ufeff", "")
    text = text.replace("“", '"').replace("”", '"')
    text = text.replace("‘", "'").replace("’", "'")
    # 按需补充全角逗号、冒号等替换
    return text
```

### 2. 剥离 fenced code block

不要用 `re.search(r'```.*?```', text, re.S)` 这种全局匹配，因为它可能跨多个代码块误删。更稳妥的做法是逐行处理，只剥离行首出现的 fence 标记，并且保留内部内容。

常见 fence 形式：

- ```` ```json ````
- ```` ``` ````
- `~~~json`

### 3. 剥离外层的轻量标签

有些模型会用 `<json> ... </json>`、`<output> ... </output>`、`<result> ... </result>` 包裹。注意不要用 `re.sub(r'<.*?>', '', text, flags=re.S)`，因为 JSON 字符串里可能包含 `<` 或类似内容。可以先做大小写不敏感的标签名匹配，优先剥离最外层的常见标签。

### 4. 边界扫描提取 JSON

核心是用 `JSONDecoder.raw_decode` 从候选位置尝试解析，而不是简单地截取第一个 `{` 到最后一个 `}`。

```python
import json
from json import JSONDecodeError

def extract_json(text: str):
    decoder = json.JSONDecoder()
    candidates = []
    for i, ch in enumerate(text):
        if ch in "{[":
            candidates.append(i)

    for start in candidates:
        try:
            obj, _ = decoder.raw_decode(text[start:])
            return obj
        except JSONDecodeError:
            continue

    # 兜底兼容注释、尾逗号、单引号
    try:
        import json5
        return json5.loads(text)
    except Exception:
        raise ValueError("no valid json found")
```

`raw_decode` 会从给定位置开始解析，只消费合法的 JSON 前缀，能避免“最后一个 `}` 之后还有自然语言”的情况。它返回 `(obj, end)`，如果成功，我们只关心 `obj`。

### 5. Schema 校验

解析成功不等于数据可用。比如工具参数需要 `{"tool": str, "args": dict}`，解析出来却是数组或字段类型错误。建议在解析后用 Pydantic、jsonschema 或自定义校验再兜底一次。

```python
from pydantic import BaseModel

class ToolCall(BaseModel):
    tool: str
    args: dict
```

只有通过校验的数据才进入后续 MCP 调用或插件逻辑，否则记录清洗中间态并返回结构化错误。

## 踩坑点

- **不要用单条正则匹配首尾花括号**。嵌套对象、数组或字符串里的 `}` 会直接破坏结果。
- **剥 fence 要判断行首**。JSON 字符串中间出现反引号时，不能误删。
- **标签剥离避免非贪婪通配**。`<json>` 标签如果带属性、换行或大小写变化，简单正则容易漏掉。
- **先标准 JSON 再 json5**。json5 能容忍注释和尾逗号，但可能在数字、布尔值上产生非预期类型。优先用标准解析保住数据一致性。
- **不要过度宽容**。如果解析层什么都接受，后面 schema 校验的压力会变大，错误也更难追溯。明确接受哪些格式，拒绝哪些格式。
- **错误日志不要明文打印完整输出**。模型输出可能包含敏感 token。记录输出长度、错误类型、关键位置，必要时记录哈希，方便排查即可。

## 可复用建议

在 OpenClaw 项目里，可以把 `clean_text -> strip_fence -> strip_tags -> extract_json -> schema_validate` 串成一个小的 `LLMJSONParser`，放到统一 utils 里。MCP server、插件调用、Agent 工具参数处理都复用它。

解析失败时，最好返回结构化错误，而不是抛一个裸异常让上游抓瞎：

```python
@dataclass
class ParseError:
    reason: str
    cleaned_text: str
    original_len: int
```

这样上层可以决定重试、降级，还是把错误信息反馈给模型。

## 总结

LLM 结构化输出不能只靠 prompt 约束。即使模型大部分时候会返回 JSON，只要有一次混入标签、注释或代码块，下游自动化链路就可能中断。防御性解析是稳定性最后一道门：用分层清洗、边界扫描、`raw_decode` 提取和 schema 校验，能显著降低 OpenClaw/Agent/MCP 体系里的解析崩溃率。

关键不是写一个“能清洗一切”的正则，而是把解析流程做得可预期、可追溯、可复用。这样既不会被脏输出打穿，也不会因为过度宽容引入更难排查的类型问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/2b89d877b73437fc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/906034c2a419bc00.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/bc143d922b405e4c.png)

