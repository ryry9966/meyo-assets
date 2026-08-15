---
title: LLM 输出解析的防御性编程：JSON 与标签混用的工程化处理
feedId: 33323
source: 综合讨论
publishedAt: 2026-08-16
---

在 OpenClaw、Agent、MCP 插件或自动化流程里，让模型返回结构化 JSON 是常见需求。但真实输出往往不是干净的 `{"foo": "bar"}`，而是混合了 Markdown 代码块、`<result>` 标签、前后解释文字，甚至多个候选 JSON。直接 `json.loads()` 会让一个本来能用的结果把整条链路打挂。

本文记录一套可落地的防御性解析思路，适合接入了本地模型、代理模型或输出不稳定模型的场景。

## 背景

很多自动化实践会在 prompt 里写“请只输出 JSON，不要包含其他内容”，但模型仍可能输出：

```text
好的，结果如下：
<result>
{
  "status": "ok",
  "data": [1, 2, 3]
}
</result>
希望有帮助。
```

或者：

````text
```json
{"status": "ok", "data": [1, 2, 3]}
```
````

更麻烦的是，模型还可能输出多个 JSON 片段、在 JSON 里混入注释、尾逗号或单引号。解析器如果只处理理想格式，线上一定会遇到失败重试、流程中断、日志里只留下“Invalid JSON”的问题。

## 问题

核心问题不是“模型不听话”，而是我们把解析假设得太严格。工程上需要默认模型输出是不可信的，把解析设计成一条从脏数据中提取有效结构的管线，而不是单点解析。

## 做法 / 步骤

### 1. 建立解析管线，不做单点 `json.loads`

我一般按以下顺序处理：

```text
原始文本
  -> 预处理：去掉代码块、提取标签内容
  -> 候选提取：扫描所有平衡花括号片段
  -> 修复与解析：JSON5 / json_repair
  -> Schema 验证
  -> 降级返回
```

每一步都可以独立测试，避免把逻辑堆在一个正则里。

### 2. 预处理：先剥离 Markdown fence 和常见标签

优先处理代码块：

```python
import re

def strip_code_fence(text: str) -> str:
    # 去掉 ```json ... ``` 或 ``` ... ```
    text = re.sub(r"```(?:json)?\s*([\s\S]*?)```", r"\1", text, flags=re.IGNORECASE)
    return text
```

再提取标签内容。常见标签有 `<result>`、`<json>`、`<output>`、`<response>`，大小写和空白可能不固定：

```python
def extract_tag(text: str, tags=("result", "json", "output", "response")) -> str | None:
    for tag in tags:
        m = re.search(
            rf"<{tag}\b[^>]*>([\s\S]*?)</{tag}>",
            text,
            flags=re.IGNORECASE,
        )
        if m:
            return m.group(1).strip()
    return None
```

这里用 `[\s\S]*?` 而不是 `.`，是为了跨行匹配。非贪婪避免吞掉后面的内容。

### 3. 候选提取：不要只取第一个大括号

预处理后如果还不是纯 JSON，就扫描所有平衡花括号片段：

```python
def extract_json_candidates(text: str) -> list[str]:
    candidates = []
    start = text.find("{")
    while start != -1:
        depth = 0
        in_string = False
        escape = False
        for i in range(start, len(text)):
            ch = text[i]
            if in_string:
                if escape:
                    escape = False
                elif ch == "\\":
                    escape = True
                elif ch == '"':
                    in_string = False
            else:
                if ch == '"':
                    in_string = True
                elif ch == "{":
                    depth += 1
                elif ch == "}":
                    depth -= 1
                    if depth == 0:
                        candidates.append(text[start:i + 1])
                        start = text.find("{", i + 1)
                        break
        else:
            break
    return candidates
```

这段代码能处理字符串里嵌套的 `{}`，避免正则贪婪犯的错。

### 4. 修复与解析：优先 JSON5，再上修复库

很多模型会输出尾逗号、注释、单引号，这些都是 JSON5 的合法语法。先试标准 JSON，再试 JSON5，最后试 `json_repair`：

```python
import json
import json5

def parse_candidate(candidate: str):
    for parser in (json.loads, json5.loads):
        try:
            return parser(candidate)
        except Exception:
            continue
    try:
        from json_repair import repair_json
        return json.loads(repair_json(candidate))
    except Exception:
        return None
```

注意：`json_repair` 可能“修错”，尤其是当候选片段本身不是 JSON 时。所以一定要先做候选过滤，比如长度太短、不包含关键字段的候选直接跳过。

### 5. Schema 验证与降级

如果使用了 Pydantic 或 JSON Schema，解析后做一次校验。校验不过时，不要立刻抛异常，而是把原始输出作为字符串塞进一个 `raw` 字段，让上层决定是否重试、降级或转人工。

## 踩坑点

- **标签内再套代码块**：模型可能输出 `<result>\n```json\n{...}\n```\n</result>`。所以先提取标签，再剥离代码块，顺序不能反。
- **多个候选 JSON**：不要默认第一个就是对的。可以根据是否包含目标字段、长度、解析后是否通过 schema 来评分，选择最合适的一个。
- **正则跨行与贪婪**：标签提取务必用非贪婪和 `[\s\S]`，否则跨行内容会丢。
- **修复库会改变语义**：`json_repair` 有时会把不完整字符串截断，或者补错括号。候选评估和日志记录很重要。
- **全角引号和 Unicode 空白**：模型可能输出中文引号 `“ ”` 或全角空格。可在预处理阶段做一次归一化，但不要过度替换，避免误伤字符串内容。
- **日志隐私**：解析失败时不要直接把完整原始输出打到错误日志，可能包含敏感信息。记录长度和片段即可。

## 可复用建议

封装一个独立模块，例如：

```python
@dataclass
class ParseResult:
    ok: bool
    data: Any
    method: str  # "strict_json" / "tag_extract" / "fence_strip" / "candidate_repair" / "fallback"
    raw_preview: str
```

每次解析都记录用了哪种方法，线上观察分布，能快速判断模型输出质量。

另外保留原始输出原文，方便回归测试。构造混合样本作为单元测试，比如“前有解释+代码块+标签内多个 JSON+尾逗号”的组合，能显著减少线上意外。

## 总结

LLM 输出解析的防御性编程，核心是放弃“模型会输出完美 JSON”的假设。用预处理、候选提取、修复解析、Schema 验证和降级返回组成管线，把不确定性显式暴露给上层。这样即使模型偶尔输出脏格式，流程也不会轻易中断，排查时也有据可查。

---

