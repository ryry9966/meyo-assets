---
title: LLM 输出解析的防御性编程：JSON 与标签格式混合处理
feedId: 33847
source: 综合讨论
publishedAt: 2026-08-19
---

# LLM 输出解析的防御性编程：JSON 与标签格式混合处理

## 背景

在 OpenClaw 的 Agent、MCP、插件和自动化流程里，我们经常让 LLM 直接输出 JSON 作为工具调用参数、状态机结果或插件配置。理想情况下，模型返回干净的 JSON；实际链路里，模型输出经常被 Markdown 代码块、解释文字、XML 标签甚至多个 JSON 块包裹。比如：

````text
好的，结果如下：
```json
{
  "action": "search",
  "query": "OpenClaw MCP",
}
```
````

这种输出直接 `json.loads` 会崩溃。如果解析代码不够防御，一次异常就会中断整条 Agent 链路。更麻烦的是，这类问题不是稳定复现，往往是偶发、跟 prompt 或模型版本有关。

## 问题

典型的坏样本包括：

- ```json``` 代码块包裹，有时语言标记写 ```JSON``` 或没有语言标记
- JSON 前后有模型解释文字
- JSON 被 XML 标签包住，如 `<result>{...}</result>`
- 字段里出现尾逗号、单引号、注释
- 一个回复里出现多个 JSON 块，只有其中一个符合目标 schema
- 字符串内部未转义换行，导致正则提取截断
- 字段类型漂移，例如 `"count": "3"` 而不是 `3`

如果解析逻辑只写了 `response.json()` 或 `json.loads(output)`，这些情况都会变成线上故障。

## 做法 / 步骤

我通常在一个 `parse_llm_json` 工具函数里做分层处理，而不是在业务代码里散落 `json.loads`。

### 1. 归一化输入

先做基础清洗：去掉 BOM、首尾空白、不可见字符。不要直接找 ```json``` 标签，因为语言标记可能缺失。

```python
import re
import json
import json5

def parse_llm_json(raw: str, required_fields: set[str] | None = None):
    text = raw.lstrip("\ufeff").strip()
    candidates: list[str] = []

    # 1. 提取 XML 标签包裹的 JSON
    for m in re.finditer(r"<([a-zA-Z_][\w-]*)>\s*(\{.*?\}|\[.*?\])\s*</\1>", text, re.S):
        candidates.append(m.group(2))

    # 2. 提取 Markdown 代码块
    for m in re.finditer(r"```(?:json|JSON)?\s*(\{.*?\}|\[.*?\])\s*```", text, re.S):
        candidates.append(m.group(1))

    # 3. 兜底：找第一个看起来像 JSON 对象/数组的片段
    for m in re.finditer(r"(\{.*\}|\[.*\])", text, re.S):
        candidates.append(m.group(1))
```

这里我把 XML 标签、代码块、裸 JSON 三种候选都提取出来，后面再按 schema 选择，而不是只取第一个。

### 2. 分级解析

对每个候选依次尝试解析，优先用宽容解析器：

```python
    parsed = None
    errors = []
    for cand in candidates:
        for parser in (json.loads, json5.loads):  # json5 容忍尾逗号/注释/单引号
            try:
                parsed = parser(cand)
                break
            except Exception as e:
                errors.append(str(e))
        if parsed is not None:
            break
```

标准 `json.loads` 对尾逗号和注释会直接抛异常，`json5` 可以容忍多数非标准写法。不要用 `eval`，哪怕它看起来能解析，安全风险不值得。

### 3. Schema 校验与类型强制

解析成功不代表数据结构正确。用 `required_fields` 做轻量校验，并对常见类型做强制转换：

```python
    if parsed is None:
        return {"ok": False, "errors": errors, "raw": raw}

    if isinstance(parsed, dict) and required_fields:
        missing = required_fields - parsed.keys()
        if missing:
            return {"ok": False, "errors": [f"missing fields: {missing}"], "raw": raw}

    # 常见类型漂移处理
    if isinstance(parsed, dict):
        for k in ("count", "limit", "temperature"):
            if k in parsed and isinstance(parsed[k], str):
                try:
                    parsed[k] = float(parsed[k]) if "." in parsed[k] else int(parsed[k])
                except ValueError:
                    pass

    return {"ok": True, "data": parsed, "raw": raw}
```

如果候选有多个，且都解析成功，优先选择包含 `required_fields` 的那个；否则选第一个非空对象。

### 4. 失败重试与日志

当所有候选都解析失败时，不要把原始输出直接丢弃。把解析错误作为反馈发给模型，做一次有限重试。通常最多重试 1～2 次，避免 token 消耗失控。

```python
def call_llm_with_retry(prompt, schema_fields, max_retries=2):
    raw = llm_call(prompt)
    result = parse_llm_json(raw, required_fields=schema_fields)
    for _ in range(max_retries):
        if result["ok"]:
            return result["data"]
        raw = llm_call(
            f"Your previous JSON output failed to parse.\n"
            f"Errors: {result['errors']}\n"
            f"Please return only valid JSON with fields: {schema_fields}\n\n"
            f"Previous output:\n{raw}"
        )
        result = parse_llm_json(raw, required_fields=schema_fields)
    return result  # 最终失败也保留 raw 和 errors，不要静默回退
```

日志里保留 `raw`、候选列表和解析错误，方便回溯偶发问题。

## 踩坑点

1. **不要只依赖 ```json``` 前缀**。模型会写 ```JSON```、```json5``` 或完全不带语言标记。用多种正则会稳得多。
2. **正则提取裸 JSON 时要注意字符串内容**。`\{.*\}` 这种贪婪正则可能跨多个对象，也可能在字符串内截断。优先用代码块和标签提取，裸 JSON 只作为兜底。
3. **多 JSON 块不要只取第一个**。有的模型会先给一个错误示例，再给正确结果。按目标字段或结构选候选更可靠。
4. **不要无限重试**。解析失败重试一次通常能修好大半，两次是上限。超过两次模型往往会重复同样错误，消耗额度。
5. **注意非标准 JSON 值**。Python `json.loads` 会接受 `NaN`、`Infinity`，这些值可能在序列化到其它语言时出问题。需要时在解析后对浮点字段做 `math.isfinite` 检查。
6. **标签提取不要贪婪跨块**。`<result>...</result>` 的正则要写非贪婪 `.*?`，否则多个标签会连成一片。

## 可复用建议

- 把解析逻辑封装成独立函数，所有需要 LLM 返回 JSON 的调用都走它，不要各写各的。
- 准备一个坏样本单元测试集：尾逗号、单引号、代码块、标签包裹、多 JSON、缺失字段、类型漂移。每次改解析逻辑都跑一遍。
- 在 `system prompt` 里要求模型“只返回 JSON，不要代码块或解释”，但不要只靠 prompt。防御性解析是兜底，prompt 只是减少触发概率。
- 对 MCP 工具返回的结构化结果，如果解析失败，把原始输出以可读方式展示给用户，而不是返回一个模糊的“内部错误”。
- 如果下游对字段类型严格，建议接 Pydantic 做二次校验；轻量场景用 `required_fields` 即可。

## 总结

LLM 输出解析的防御性编程，核心不是把解析写得多聪明，而是把“期望干净 JSON”变成“接受混合格式、分级提取、宽容解析、按 schema 选择、失败可追溯”。在 OpenClaw 的 Agent 和 MCP 链路里，这种小工具能显著减少因偶发格式问题导致的流程中断。工程上做到：先归一再提取，先标准后宽容，先校验后使用，失败留现场。

---

