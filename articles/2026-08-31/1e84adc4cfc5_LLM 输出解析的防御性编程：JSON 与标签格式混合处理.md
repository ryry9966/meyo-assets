---
title: LLM 输出解析的防御性编程：JSON 与标签格式混合处理
feedId: 35453
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 的 Agent、插件和 MCP 链路里，模型输出经常不是干净 JSON。实际拿到的内容可能是：

- ```` ```json ... ``` ```` 围栏代码块
- `<output>{"a":1}</output>`、`<result>...</result>` 这类标签包 JSON
- 一段自然语言说明后直接跟 JSON
- 一个块里同时出现围栏、标签和裸 JSON

如果下游插件或 MCP 工具用 `json.loads(text)` 直接解析，失败率会很高。真正可靠的做法不是“换一个更强的正则”，而是把解析做成防御性流程：先提取候选块，再逐个尝试解析，最后用 schema 校验兜底。

## 问题

直接解析常见的几个坑：

- BOM、零宽空格 `\u200b`、`\u2060` 会让 `json.loads` 失败。
- 模型输出 ```` ```json ... ``` ```` 或 ```` ``` ... ``` ```` 围栏。
- 返回 `<output>...</output>` 这类类 XML 标签。
- 多个候选 JSON 块并存，其中可能只有一个符合目标结构。
- 尾逗号、注释或单引号导致 JSON 解析失败，但盲目修复又可能改变语义。

在 OpenClaw 的 plugin adapter 或 MCP 参数整理阶段，如果这里不做防御，一次异常输出就会中断整条自动化链路。

## 做法与步骤

我一般把解析拆成四步：归一化、候选提取、逐个解析、schema 校验。

### 1. 归一化

先处理不可见字符和 BOM：

```python
def _strip_noise(s):
    return s.strip().lstrip("\ufeff").replace("\u200b", "").replace("\u2060", "")
```

不要忽略这一步，零宽空格在失败日志里看起来像“明明字符串是对的”。

### 2. 候选提取

不要只写一个“万能正则”，建议按三个通道提取：

- 围栏代码块：匹配 ```` ```json ... ``` ````、` ``` ```` 等。
- 类 XML 标签：匹配 `<output>...</output>`、`<result>...</result>` 等，注意用非贪婪匹配。
- 裸 JSON：用栈扫描 `{}` 和 `[]`，处理字符串、转义和嵌套，不要用 `r"\{.*\}"` 这种正则。

一个简化实现如下：

```python
import json
import re

_TAG_RE = re.compile(
    r"<(\w+)[^>]*>\s*([\s\S]*?)\s*</\1>", re.I
)
_FENCE_RE = re.compile(
    r"```(?:json|javascript|python|jsonc)?\s*([\s\S]*?)```",
    re.I,
)

def _strip_noise(s):
    return s.strip().lstrip("\ufeff").replace("\u200b", "").replace("\u2060", "")

def _balanced_json(text):
    start = min(
        [i for i in (text.find("{"), text.find("[")) if i >= 0],
        default=-1,
    )
    if start < 0:
        return None

    pairs = {"{": "}", "[": "]"}
    stack = []
    in_str = False
    esc = False

    for i, ch in enumerate(text[start:], start):
        if in_str:
            if esc:
                esc = False
            elif ch == "\\":
                esc = True
            elif ch == '"':
                in_str = False
            continue

        if ch == '"':
            in_str = True
        elif ch in pairs:
            stack.append(pairs[ch])
        elif ch in ("]", "}"):
            if stack and stack[-1] == ch:
                stack.pop()
                if not stack:
                    return text[start : i + 1]
    return None

def _try_load(s):
    if s is None:
        return None
    s = _strip_noise(s)
    attempts = [
        s,
        re.sub(r",\s*([}\]])", r"\1", s),  # 处理尾逗号
    ]
    for item in attempts:
        try:
            return json.loads(item)
        except Exception:
            continue
    return None

def extract_json_payload(text):
    text = _strip_noise(text)
    candidates = []

    candidates.extend(_FENCE_RE.findall(text))
    candidates.extend(m.group(2) for m in _TAG_RE.finditer(text))

    raw = _balanced_json(text)
    if raw:
        candidates.append(raw)

    for cand in candidates:
        data = _try_load(cand)
        if data is not None:
            return data

    return None
```

这个实现只是基础版，生产环境建议加候选数量上限和 debug 日志。

### 3. 可解析性优先于出现顺序

注意不要遇到第一个候选就返回。比如模型可能先给一段示例 JSON，再给真正的输出。正确做法是：按照候选列表顺序，先找到第一个能通过 `json.loads` 的块。

如果下游有 Pydantic 或 dataclass schema，应该把解析结果再走一次：

```python
for cand in candidates:
    data = _try_load(cand)
    if data is None:
        continue
    try:
        return TargetModel(**data)
    except ValidationError:
        continue
```

这样能把“是 JSON 但不是目标结构”的候选块自然淘汰掉。

### 4. 失败可观测

所有候选都失败时，不要把 `None` 静默传给插件。建议记录原文、候选数量、每个候选的失败原因，并向上层返回可读错误。这样 Agent 可以重试，或者至少我们知道该修 prompt 还是修 parser。

## 踩坑点

- **零宽空格和 BOM**：日志里看起来无辜，实际会让 `json.loads` 直接失败。
- **标签正则贪婪**：如果标签内容里还有同类型标签，`.*` 会吞掉后面内容。务必用非贪婪 `[\s\S]*?`。
- **裸 JSON 用正则提取**：`r"\{.*\}"` 遇到嵌套对象会截断。栈扫描才可靠。
- **不要无脑替换单引号**：JSON 字符串里可能包含撇号，全局替换会破坏内容。
- **多个候选块**：不要假设第一个就是目标，schema 校验比出现顺序更可靠。
- **过度修复**：尾逗号可以顺手处理，但单引号、注释、无引号键不建议乱修；如果确实需要，可以引入 json5 或专门的 repair 库，并补测试。

## 可复用建议

1. 把解析逻辑做成独立模块，放在 LLM 输出进入 OpenClaw 插件、MCP adapter 之前。
2. 建立固定 fixture 测试：纯 JSON、围栏 JSON、标签 JSON、混合文本、多个候选块、带 BOM、尾逗号、嵌套对象。
3. prompt 里可以要求“只输出 JSON，不要 Markdown，不要注释”，但这只能降低概率，不能替代防御性解析。
4. 对第三方模型或不可控输出启用 repair 模式；对本地可控模型使用严格模式，先让失败暴露出来。
5. 失败信息要保留原文摘录和候选数量，不要只打印一句 `JSON decode error`。

## 总结

LLM 输出混合 JSON、代码围栏和标签格式不是偶发 bug，而是需要默认防御的场景。把“提取候选、逐个解析、schema 校验、失败可观测”做成固定链路，比维护一堆 if-else 正则更可靠。对于 OpenClaw 的插件和自动化实践用户来说，这一层解析器值得作为基础设施沉淀下来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/27ae9ae84497510f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/27b665f0a8415fef.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/d50e422c7fd1b692.png)

