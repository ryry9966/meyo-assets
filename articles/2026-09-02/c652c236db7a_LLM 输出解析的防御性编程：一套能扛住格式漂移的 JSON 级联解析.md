---
title: LLM 输出解析的防御性编程：一套能扛住格式漂移的 JSON 级联解析
feedId: 35817
source: 综合讨论
publishedAt: 2026-09-02
---

# 背景

在 OpenClaw 的 agent 和 MCP 工具链里，让模型输出结构化 JSON 是家常便饭：工具调用参数、插件间消息、自动化流水线的中间结果。即使平台提供了 structured output / function calling，实际工程中仍大量依赖"提示词约定 + 文本解析"：兼容不同后端模型、本地小模型、或者模型把 JSON 混在解释文字里输出的场景。

# 问题

最朴素的写法是 `json.loads(response)`，然后祈祷。现实是同一份提示词，换个模型、换个温度、甚至同一个模型升级一版，输出格式就会漂移：

- JSON 包在代码围栏里，标签有时是 ` ```json `、` ```JSON `、` ```jsonc `，有时干脆不写；
- 外面裹着 `<json>...</json>`、`<tool_call>...</tool_call>` 之类的自造标签，偶尔还漏闭合；
- 前后带一句"以下是解析结果："之类的客套话；
- 尾逗号、中文引号、`True/False/None` 这类 Python 字面量。

任何一个变体都能让流水线在半夜挂掉。防御性编程的思路很直接：把模型输出当不可信输入，解析层做成级联，而不是单点猜测。

# 做法：解析级联（parse cascade）

核心原则是"从最便宜到最贵的尝试依次排开"，任何一层成功即返回：

1. 直接 `json.loads` 全文；
2. 提取代码围栏内容再试；
3. 正则提取自造标签内容再试；
4. 括号配对扫描，取第一个平衡的 `{...}`；
5. 全部失败后做最小修复，整体重试一轮。

骨架二十来行：

```python
import json, re

FENCE_RE = re.compile(r"```(?:json|JSON|jsonc)?\s*(.*?)```", re.S)
TAG_RE   = re.compile(r"<(?:json|result|tool_call|answer)>(.*?)</(?:json|result|tool_call|answer)>", re.S | re.I)

def balanced_snippet(text):
    start = text.find("{")
    if start < 0:
        return None
    depth, in_str, esc = 0, False, False
    for i in range(start, len(text)):
        ch = text[i]
        if in_str:
            if esc: esc = False
            elif ch == "\\": esc = True
            elif ch == '"': in_str = False
        elif ch == '"': in_str = True
        elif ch == "{": depth += 1
        elif ch == "}":
            depth -= 1
            if depth == 0:
                return text[start:i + 1]
    return None

def repair(s):
    s = re.sub(r",\s*([}\]])", r"\1", s)  # 尾逗号
    s = s.replace("\u201c", '"').replace("\u201d", '"')
    s = re.sub(r"\bTrue\b", "true", s)
    s = re.sub(r"\bFalse\b", "false", s)
    s = re.sub(r"\bNone\b", "null", s)
    return s

def robust_parse(text):
    candidates = [text, *FENCE_RE.findall(text), *TAG_RE.findall(text)]
    if (snip := balanced_snippet(text)):
        candidates.append(snip)
    for c in candidates:
        c = c.strip()
        for attempt in (c, repair(c)):
            try:
                return json.loads(attempt)
            except json.JSONDecodeError:
                continue
    raise ValueError(f"unparseable output: {text[:200]}")
```

解析成功不等于数据正确。拿到 dict 后必须过 pydantic / jsonschema 校验；校验失败时把报错路径喂回模型重试一次，比原样重发有效得多。

# 踩坑点

1. **括号配对必须感知字符串**。朴素的 depth 计数遇到字符串里的 `{}` 或转义引号就数错，上面 `in_str/esc` 状态机不能省。
2. **修复要克制**。`replace("'", '"')` 一把梭会破坏字符串值里的单引号，甚至静默改数据。只做无歧义的几项，且仅在首轮全失败后启用，修完必须过 schema 兜底。
3. **findall 可能拿到多块**。模型偶尔输出两个 JSON 对象。建议"全文 → 围栏 → 标签 → 括号片段"逐个试，取第一个**能通过校验**的，而不是第一个能 parse 的——能 parse 但字段不对的应该让位。
4. **失败必须留原始文本**。解析失败时把完整原始输出落日志，否则没法离线复现、没法补测试用例。
5. **提示词里写"只输出 JSON"作用有限**。few-shot 示例加低温度比命令式措辞稳定，但级联仍然要有，因为漂移不会归零。

# 可复用建议

- 级联封装成一个纯函数：输入字符串，输出 dict 或抛异常，不要各插件各写各的。
- 解析后一律过 schema 校验；重试时带反馈："上次第 3 个字段类型不符，请修正"。
- 用线上真实坏输出维护 fixture 目录，进 CI 跑回归。这份"脏语料"比想象出来的边界用例值钱得多。
- 优先级排序：原生 structured output > function calling > 标签约定 + 级联 > 自由文本硬解析。级联是兜底，不是首选。
- 修复函数保持幂等，修复前后各打一行 debug 日志，方便审计。

# 总结

LLM 输出解析是 agent 稳定性里性价比最高的一层防御：二十来行代码加一个 fixture 目录，就能把"半夜挂掉"变成日志里一行 fallback 记录。原则只有两条——把模型输出当不可信输入，把解析做成有序级联而非单点猜测。格式漂移消灭不了，但可以让它不再伤害流水线。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/a248278b0d389506.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/f2e8e60bbc71437d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/9b683b73e8777db7.png)

