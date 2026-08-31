---
title: LLM 输出解析的防御性编程：JSON 与 Markdown 标签混合处理
feedId: 35493
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 里做 Agent、MCP server、插件或者自动化流程时，我们经常让 LLM 返回结构化数据，比如：

```json
{"tool": "search", "args": {"query": "openclaw"}}
```

下游代码直接拿这个对象做工具调用、参数透传或者状态更新。但真实模型输出往往并不干净。你可能已经遇到过：

```text
Here is the result:
```json
{"status": "ok"}
```
Hope this helps!
```

甚至有时模型会返回两个 JSON 块、带尾随逗号、带注释、或者把 `JSON` 写成大写 `JSON`，再或者开头带 BOM、不可见字符。如果你直接 `json.loads()`，大概率会在生产环境里突然崩掉，而且崩得莫名其妙。

这就是为什么我们需要一个防御性的解析层：不假设模型会稳定输出标准 JSON，而是主动处理各种“脏输出”。

## 问题：模型输出到底有多乱

在实际接入 OpenClaw 工作流时，LLM 输出常见以下几种形态：

1. **纯 JSON**：`{"status": "ok"}`
2. **Markdown 围栏**：
   ````text
   ```json
   {"status": "ok"}
   ```
   ````
3. **围栏外带解释文本**：
   ```text
   Sure, here is the JSON:
   ```json
   {"tool": "search"}
   ```
   Let me know if you need anything else.
   ```
4. **多个 JSON 对象**：比如先输出一个方案说明 JSON，再输出一个执行计划 JSON
5. **非标准 JSON**：尾随逗号、注释、单引号、多余文本等

这些形态混合出现时，单靠 prompt 里的“只输出 JSON，不要 Markdown 围栏”约束并不够。模型仍可能因为上下文、示例或者习惯而违反约束。因此，我们在解析入口处必须做防御。

## 做法/步骤

下面是一个可落地的防御性解析流程。

### 1. 预处理

先做基础清理：

- `strip()` 去掉首尾空白
- 去掉开头可能的 BOM（`\ufeff`）
- 去除不可见字符和零宽字符

```python
text = raw_output.strip().lstrip("\ufeff")
```

### 2. 优先提取 Markdown 代码块

如果文本中存在 ``` 围栏，我们优先从围栏里取内容，因为围栏内的部分通常才是模型认为的“代码体”。

```python
fence_pattern = re.compile(r"```[a-zA-Z]*\s*(.*?)```", re.DOTALL)
fenced_blocks = fence_pattern.findall(text)
```

这里有几个细节：

- 语言标识可能是 `json`、`JSON`、`jsonc` 甚至空，所以用 `[a-zA-Z]*`
- 必须用非贪婪 `.*?`，否则多个代码块会被一次吃掉

如果命中多个块，可以筛出包含 `{` 或 `[` 的块，并优先选**最后一个**，因为很多模型会先给说明代码，最后给真正的结果。

### 3. 如果没有围栏，直接从混合文本中提取 JSON 片段

找不到围栏时，不要尝试用 `text[:text.rfind("}")+1]` 这种切片，因为字符串里面可能包含 `}`。更好的方式是定位第一个 `{` 或 `[`，然后用 `json.JSONDecoder().raw_decode()` 按 JSON 语法解析前缀。

```python
start_candidates = [i for i in (text.find("{"), text.find("[")) if i != -1]
if not start_candidates:
    raise ValueError("No JSON object found")
start = min(start_candidates)

decoder = json.JSONDecoder()
obj, end = decoder.raw_decode(text[start:])
```

`raw_decode` 会从给定位置开始解析一个完整的 JSON 对象，并返回结束位置。这样即使前后有多余文本，也能准确拿到第一个 JSON 结构。

### 4. 容错处理

如果标准 `json` 解析失败，而模型输出带有尾随逗号或注释，可以降级使用 `json5` 或 `demjson3` 做宽松解析。但要小心：宽松解析可能把本应报错的错误结构也“放过”，因此最好只在标准解析失败后使用。

### 5. 结合 schema 校验

解析完成后，不要直接信任结果。配合 Pydantic 或 JSON Schema 校验字段类型和必需字段。如果校验失败，可以触发一次重试，或者把原始输出打进日志，方便定位模型到底输出了什么。

一个精简版 parser 可以写成：

```python
import json
import re
from typing import Any

def extract_json(text: str) -> Any:
    text = text.strip().lstrip("\ufeff")
    # 提取 fenced code blocks
    fences = re.findall(r"```[a-zA-Z]*\s*(.*?)```", text, re.DOTALL)
    if fences:
        json_fences = [f.strip() for f in fences if "{" in f or "[" in f]
        if json_fences:
            text = json_fences[-1]  # 通常最后一个才是目标
    # 找起始位置
    starts = [i for i in (text.find("{"), text.find("[")) if i != -1]
    if not starts:
        raise ValueError("No JSON object found")
    decoder = json.JSONDecoder()
    obj, _ = decoder.raw_decode(text[min(starts):])
    return obj
```

## 踩坑点

- **贪婪正则**：用 `.*` 提取代码块会跨多个 ```，导致内容串掉。一定用 `.*?` 加 `DOTALL`。
- **字符串内包含 `{}`**：不要用简单的 `rfind("}")` 切片。`raw_decode` 是更安全的选择。
- **多个 JSON 对象**：`raw_decode` 只解析第一个。如果业务上需要最后一个或多个，要明确语义，或者收集所有候选再逐个校验。
- **尾随逗号和注释**：标准 `json.loads` 直接报错。可以先用标准解析，失败后再用 `json5`，但注意引入额外依赖。
- **语言标识大小写**：模型可能输出 ```JSON 大写，甚至 ```jsonc。正则应放宽为 `[a-zA-Z]*`，不要写死 `json`。
- **BOM 和零宽字符**：这些会直接导致 `json.loads` 失败，预处理要包含它们。

## 可复用建议

1. **独立模块**：把这个解析逻辑封装成 `llm_json_parser.py`，所有需要结构化输出的入口都统一走它。
2. **建坏样本测试集**：把围栏、解释文本、多 JSON、尾随逗号、BOM、大写 ```JSON 等做成测试用例，在 CI 里跑，避免以后改挂。
3. **先标准后宽松**：不要一上来就用 `json5`，否则可能掩盖真正的格式错误。标准解析失败后再降级。
4. **解析后必校验**：Pydantic/schema 校验应该成为固定步骤，失败就重试或记录原始输出。
5. **Prompt 仍然要约束**：防御性解析是最后防线，但 prompt 里明确要求“只输出 JSON，不加 Markdown 围栏”能减少大部分脏输出。
6. **保留原始输出**：日志中记录未解析的原始字符串，方便线上排障时回溯模型行为。

## 总结

不要假设 LLM 会稳定输出标准 JSON。实际模型输出会混入 Markdown 标签、解释文本、多个对象甚至非法 JSON。防御性解析不是复杂算法，而是把一批常见的异常形态固化成一个小而清晰的模块。做好预处理、代码块提取、`raw_decode` 容错和 schema 校验，你的 OpenClaw Agent、MCP 工具和插件就能少很多“突然 json.loads 挂了”的线上事故。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/bfd393251b0939f9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/3830beb1ba94b6c3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/2e35cf4e1aee9cdc.png)

