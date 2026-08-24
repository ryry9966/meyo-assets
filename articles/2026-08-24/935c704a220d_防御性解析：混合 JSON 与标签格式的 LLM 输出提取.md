---
title: 防御性解析：混合 JSON 与标签格式的 LLM 输出提取
feedId: 34503
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw、Agent、MCP 工具链路里，经常需要把 LLM 输出当作结构化数据使用，例如工具调用参数、插件返回结果、自动化节点数据。常见做法是在提示词中要求模型返回 JSON，但实际输出经常会混合自然语言解释、Markdown 代码块、XML 标签，甚至多个 JSON 片段。

如果解析器只做 `json.loads`，一旦遇到格式漂移，整个流程就会中断。更麻烦的是，这类问题往往不是稳定复现，而是偶发且依赖模型状态。

## 问题

典型的混合输出长这样：

```
<thought>需要查询设备状态</thought>
```json
{"tool":"get_device","args":{"device_id":"A1"}}
```

或者：

```
Here is the result:
<output>{"status":"ok","list":[1,2,3]}</output>
```

又或者：

```
```json
{"status":"ok",}
```
```

这些场景的共同点是：模型没有严格遵循“只输出 JSON”的约定。防御性解析的核心不是“解析一切”，而是分层提取、容错解析、失败可观测、可重试。

## 做法/步骤

### 1. 候选区提取

先不要直接解析，按优先级提取候选区：

- XML 标签区：`<json>...</json>`、`<output>...</output>`、`<result>...</result>`
- Markdown 围栏：` ```json ... ``` `
- 平衡括号对象：从第一个 `{` 或 `[` 扫描到闭合结束

示例：

```python
import re
import json

TAGS = ("json", "output", "result")

def extract_candidates(text: str):
    cands = []

    # 标签区
    for tag in TAGS:
        for m in re.finditer(rf"<{tag}[^>]*>(.*?)</{tag}>", text, re.S | re.I):
            cands.append(m.group(1).strip())

    # Markdown 围栏
    for m in re.finditer(r"```[ \t]*(?:json)?[ \t]*(.*?)```", text, re.S | re.I):
        cands.append(m.group(1).strip())

    # 平衡括号提取
    for start_char in ["{", "["]:
        start = text.find(start_char)
        if start == -1:
            continue
        stack = []
        for i, ch in enumerate(text[start:], start):
            if ch in "{[":
                stack.append(ch)
            elif ch in "}]":
                if stack:
                    stack.pop()
                if not stack:
                    cands.append(text[start:i+1])
                    break

    return cands
```

然后逐个尝试 `json.loads`。若某个候选解析成功且字段校验通过，就直接返回。

### 2. 容错解析

对失败候选做有限修复：

- 去尾逗号：`re.sub(r",\s*([}\]])", r"\1", s)`
- 替换单引号为双引号
- 去除单行注释：`re.sub(r"//.*", "", s)`

不建议做无限修复。如果修复后仍失败，进入重试，而不是继续猜。

### 3. Schema 校验

解析成功不等于可用。建议用 Pydantic 或 dataclass 做字段级校验：

```python
from pydantic import BaseModel

class DeviceResult(BaseModel):
    tool: str
    args: dict
    status: str = "unknown"
```

关键字段缺失时视为解析失败，走重试流程。

### 4. 失败重试

把原始输出截断到 800–1200 字符，附上解析错误原因，回传模型，要求只输出 JSON，并允许使用 ` ```json ` 围栏。最多重试 2 次。重试提示要简洁，避免上下文无限膨胀。

## 踩坑点

- 标签名漂移：模型可能返回 `<response>`、`<answer>`、`<final>`，标签列表要可配置，不要写死。
- 多个 JSON 对象：不要盲目取第一个，应选择通过 schema 且包含关键字段的那个候选。
- 平衡括号扫描必须处理字符串内部的 `{}`。简单正则 `{.*}` 容易截断错误。
- 尾逗号、注释、单引号：json5 或修复函数能救一部分，但不要默认接受未定义类型。
- 静默降级危险：如果用正则兜底提取字段，需要记录 `parse_fallback=True`，并把原始输出写入日志，否则后期排障困难。
- 重试上下文膨胀：不要每次重试都携带完整历史输出，只回传失败片段与错误信息。

## 可复用建议

- 封装 `parse_llm_json(text, schema=None)`，返回 `(parsed, meta)`，其中 `meta` 记录候选数、使用的解析器、错误类型。
- 提示词中建议模型使用 `<output>` 包裹 JSON，但解析器不要依赖这个约定。
- 建立混合格式样本库，覆盖 10–15 条真实坏样本，放入 CI 做回归测试。
- MCP 工具返回解析失败时，返回结构化错误给 Agent，而不是让 Python 堆栈直接抛到用户侧。

## 总结

LLM 输出解析是一个典型的“脏输入”问题。防御性编程不是追求解析所有格式，而是分层提取、容错解析、失败可观测、必要时重试。对 OpenClaw、Agent、MCP 自动化流程来说，稳定的解析器比提示词里写“务必输出 JSON”更可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/aac37ddf77e16b03.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/67681d83c78a29a6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/30ca5963355fc858.png)

