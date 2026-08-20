---
title: LLM 输出解析的防御性编程：JSON 与标签格式混合处理
feedId: 33924
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

在 OpenClaw、Agent、MCP 工具调用等场景里，我们经常需要把 LLM 的输出当结构化数据消费：工具参数、插件配置、审校结果、任务计划。即使 Prompt 写得再明确，比如“只输出 JSON，不要解释”，真实返回仍会出现：

```text
好的，结果如下：
```json
{
  "status": "ok",
  "items": [{"name": "a", "count": 1}]
}
```

或另一种常见形态：

```text
<result>
{"status": "ok", "items": [{"name": "a", "count": 1}]}
</result>
```

这就是“JSON 与标签格式混合输出”。直接 `json.loads()` 会失败，且失败原因通常不是模型笨，而是我们的解析策略太脆。

## 问题

典型故障点包括：

- 前后有自然语言解释；
- JSON 被 ```json``` 或 ``` 代码块包裹；
- 使用 `<result>`、`<json>` 等 XML 风格标签；
- JSON 内部有尾逗号、注释、单引号；
- 中英文标点混入、全角引号、零宽字符；
- 输出被 token 截断，JSON 残缺；
- 多个代码块、多个 JSON 片段并存。

如果解析器只做一次裸 `json.loads`，这些情况会让整个 Agent 流程中断，或更糟——把错误结果静默传给下一步。

## 做法/步骤

### 1. 先约定输出契约

Prompt 里提供固定地标，比“只输出 JSON”更有效。例如：

```text
请严格按以下格式返回，不要添加任何解释：
<json>
{"status": "...", "items": []}
</json>
```

但不要完全依赖模型遵守。契约只是降低失败率，不能替代防御解析。

### 2. 归一化与候选提取

先做输入清洗，再提取候选 JSON 片段。建议的处理顺序：

```python
import re
import json
import json5

FENCE_RE = re.compile(r"```(?:json)?\s*(.*?)```", re.S)
TAG_RE = re.compile(r"<(?:result|json)>\s*(.*?)</(?:result|json)>", re.S)

def extract_candidates(text: str):
    text = text.replace("\ufeff", "")  # 去 BOM
    # 优先级：代码块/标签优先，避免自然语言中的花括号干扰
    for pattern in (FENCE_RE, TAG_RE):
        m = pattern.search(text)
        if m:
            payload = m.group(1).strip()
            if payload:
                return [payload]
    # 最后才用花括号平衡截取
    start = text.find("{")
    end = text.rfind("}")
    if start != -1 and end > start:
        return [text[start:end + 1]]
    return []
```

要点：不要一上来就找第一个 `{` 和最后一个 `}`，因为自然语言解释里可能也包含花括号。优先使用代码块标识和标签地标。

### 3. 多级解析

候选文本不一定能直接 `json.loads`。建议设计解析链：

```python
def safe_parse(text: str):
    candidates = extract_candidates(text)
    last_err = None
    for cand in candidates:
        for parser in (try_json_loads, try_json5, try_json_repair):
            try:
                return parser(cand)
            except Exception as e:
                last_err = e
    raise ParseError(f"all parsers failed: {last_err}")
```

其中 `json5` 能处理尾逗号、注释、单引号；如果环境受限，可以引入 `json_repair` 作为截断修复兜底。但不要把 `eval` 当解析器——LLM 输出不可信。

### 4. Schema 校验与字段级兜底

结构化解析成功不是终点。必须校验关键字段、做类型转换和默认值填充：

```python
def normalize(data: dict):
    data.setdefault("status", "unknown")
    if isinstance(data.get("items"), list):
        data["items"] = [
            item for item in data["items"]
            if isinstance(item, dict)
        ]
    else:
        data["items"] = []
    # 布尔/数字字符串归一化
    if isinstance(data.get("count"), str):
        data["count"] = int(data["count"]) if data["count"].isdigit() else 0
    return data
```

字段缺失、类型错误时，记录告警并降级，不要直接抛异常。

### 5. 可观测与重试

解析失败时，保留原始输出、归一化后的文本、解析器错误，方便回放排障。对于关键路径，可以让 Agent 把解析错误作为工具返回结果重试一次，而不是静默返回 `None`。

## 踩坑点

1. **只依赖正则抓代码块**：没考虑 ```json``` 前有缩进、语言标识后还有空格或换行。正则要宽松，`re.S` 必须加。
2. **标签与 JSON 嵌套**：`<result>{"a": 1}</result>` 用花括号平衡截取通常没问题，但如果标签内还有解释性花括号，就会截错。因此标签/代码块优先级要高于花括号。
3. **多个 JSON 片段**：模型可能返回“备选方案 A {...} 方案 B {...}”。不要盲目取第一个，要按照契约地标选择；没有契约时，取候选后做 schema 校验，选择第一个能通过校验的。
4. **尾逗号和注释**：`json.loads` 炸，但 `json5` 能修。不要因此就完全放弃 `json.loads`，它可以作为最快、最严格的解析器。
5. **截断 JSON 被“修复”成错误结构**：`json_repair` 可能把缺字段补成合法 JSON，但值不一定对。必须校验必填字段和值域，否则会带着错误参数继续执行。

## 可复用建议

把解析器做成一个独立工具，放在 LLM 输出和业务逻辑之间。推荐接口：

```python
parse_llm_struct(raw_text, schema=None, strict=False)
```

默认宽容解析，`strict=True` 时开启字段级严格校验。业务方拿到的不再是裸字符串，而是带 `parsed`、`raw`、`warnings` 的结构化结果。

另外，在 Prompt 层和解析层同时防御：Prompt 给清晰示例和地标，解析器不信任任何格式承诺。这样即使模型输出漂移，也能有稳定的降级路径。

## 总结

LLM 输出解析不是“解析 JSON”的问题，而是“从不可靠文本中可靠地恢复结构化数据”的问题。防御性编程的核心不是写出一个完美解析器，而是设计一条从地标提取、多级解析、Schema 兜底到日志回放的完整链路。对 OpenClaw/Agent/MCP 用户来说，这套逻辑一次写好，能反复用在工具调用、插件配置和任务结果消费上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/b767ff6baa833b11.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/0babbd388070079b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/b266744b452a75d9.png)

