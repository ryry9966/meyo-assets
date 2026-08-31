---
title: LLM 输出解析的防御性编程：JSON 与标签格式混合处理实践
feedId: 35549
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 这类 Agent/MCP/插件自动化场景中，我们大量依赖 LLM 返回结构化数据。最典型的例子是让模型输出 JSON，供工具调用、状态更新或插件间传递。但实际部署后你会发现，模型的输出远没有 prompt 里写得那么“听话”：有时 JSON 外面包着 ````json` 代码块，有时包着 `<output>` 标签，有时前面还多一句解释，有时甚至同时出现多段 JSON。如果解析代码只有一句 `json.loads()`，线上错误率会直接教你做人。

防御性编程的原则很简单：**不要假设模型永远规范，而是假设它一定会不规范**。本文分享的是一套在 OpenClaw 场景下经过验证的混合格式解析策略。

## 问题

典型的不规范输出包括：

- JSON 前后带自然语言：`Here is the result: {"action": "send"}`
- 被 Markdown 代码块包裹：````json\n{"action": "send"}\n````
- 被 XML 风格标签包裹：`<output>{"action": "send"}</output>`
- 多层混合：````json\n<output>{"action": "send"}</output>\n````
- 多个 JSON 片段拼接：`{"a":1}\n{"b":2}`
- JSON 内部字符串包含花括号或引号，导致正则截断

这些问题用简单的正则或 `json.loads` 无法解决。我们需要一套分层的解析流程。

## 做法/步骤

### 1. 提取候选文本

先用非贪婪正则剥离常见包装：

```python
candidates = []
# 去掉 ```json ... ``` 或 ``` ... ```
candidates += re.findall(r"```(?:json)?\s*(.*?)```", text, re.S)
# 去掉 <output> ... </output>、<result> 等标签
candidates += re.findall(r"<(?:output|result|json)\s*>(.*?)</(?:output|result|json)>", text, re.S)
# 原始文本本身也作为一个候选
candidates.append(text)
```

注意这里只是“提取候选”，不做最终解析。代码块内部可能还有标签，所以后续需要递归剥离。

### 2. 用括号匹配定位 JSON 边界

对每个候选文本，找到第一个 `{` 或 `[`，然后用栈做括号匹配，同时考虑字符串内转义：

```python
def find_json_bounds(s, start):
    stack = []
    in_str = False
    escape = False
    for i in range(start, len(s)):
        ch = s[i]
        if in_str:
            if escape:
                escape = False
            elif ch == "\\":
                escape = True
            elif ch == '"':
                in_str = False
        else:
            if ch == '"':
                in_str = True
            elif ch in "{[":
                stack.append(ch)
            elif ch in "}]":
                if not stack:
                    return None
                open_ch = stack.pop()
                if (open_ch == "{" and ch != "}") or (open_ch == "[" and ch != "]"):
                    return None
                if not stack:
                    return i
    return None
```

这个方法比正则可靠得多，能处理字符串里嵌套的 `}`。

### 3. 多策略解析

对每个候选片段，先尝试直接 `json.loads()`。如果失败，按顺序尝试以下兜底策略：

- 剥离外层的 `<output>` 等标签后重试
- 去掉代码块残留的反引号
- 处理非严格 JSON（例如单引号 key、尾逗号），但要非常克制
- 如果存在多个 JSON 片段，用 `json.JSONDecoder().raw_decode` 游标方式逐个解析，取第一个有效结果

```python
def safe_load_json(text):
    for cand in candidates_from_text(text):
        try:
            return json.loads(cand)
        except Exception:
            continue
    # 最后尝试 raw_decode 处理粘连 JSON
    try:
        decoder = json.JSONDecoder()
        obj, _ = decoder.raw_decode(text.lstrip())
        return obj
    except Exception:
        return None
```

### 4. 失败降级与错误恢复

如果所有策略都失败，不要静默吞掉。返回一个结构化错误，包含原始输出摘要和失败原因，让上层 Agent 感知并决定是否重试或走文本兜底。重试时可以在 prompt 中带上“上次输出解析失败，请只返回纯 JSON”的约束，但重试次数要限制。

## 踩坑点

- **不要用 `.*?` 跨括号匹配 JSON**。遇到字符串里的 `}` 会过早截断。
- **代码块和标签可能嵌套**。例如 ````json <output>...</output> ````，需要递归剥离。
- **不要全局替换单引号为双引号**。会破坏字符串内部的单引号。
- **多个 JSON 拼接时**，`raw_decode` 只会解第一个，如果业务需要全部结果，要用循环。
- **模型返回数组而非对象**时，调用方需要做类型检查，避免后续 `.get()` 报错。
- **修复 prompt 重试失败后仍返回错误格式**是常见现象，所以要设置次数上限。

## 可复用建议

- 优先使用模型供应商提供的结构化输出能力（function calling / JSON mode），但保留防御解析作为兜底。
- 在 MCP 工具描述或插件 prompt 中，用一两句示例明确输出格式，成本很低。
- 把解析逻辑封装成独立模块，所有需要 JSON 的插件统一调用，不要各写各的。
- 记录原始输出、解析结果和失败原因到日志，方便分析模型行为变化。
- 失败时返回结构化错误而不是 `None`，让 Agent 有机会自我纠正。

## 总结

防御性解析不是给模型找借口，而是承认 LLM 输出天然带有不确定性。把解析层当作基础设施来建设，用边界匹配代替脆弱正则，用多策略代替单一 `json.loads`，用显式降级代替静默失败。这样，即使模型偶尔“抽风”，你的 OpenClaw 插件和自动化流程依然能稳定运行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/aa22182f854b50bd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/67cd7b7e96f1ad00.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/a6da09b94b363a60.png)

