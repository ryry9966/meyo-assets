---
title: LLM 输出解析的防御性编程：JSON 围栏混合格式的分层处理实践
feedId: 36894
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

在 OpenClaw 的插件和 MCP 工具开发里，让模型返回 JSON 几乎是刚需：工具参数抽取、结构化信息提取、自动化流水线的节点间传递。我们通常会在 prompt 里写"只输出 JSON，不要任何解释"，然后发现——模型并不总是听话。

即使是最成熟的闭源模型，也会因为上下文污染、few-shot 示例混入代码块、用户输入本身含反引号等原因，输出形态千奇百怪。解析层如果不做防御，流水线就会在凌晨三点挂掉。

## 问题：脏输出的常见形态

实际项目中至少会遇到这几类：

1. **围栏包裹**：` ```json {...} ``` `，最常见；
2. **裸围栏**：只有 ` ``` ` 没有语言标记，甚至嵌在回答中间；
3. **前后噪声**："好的，以下是解析结果：{...}希望对你有帮助"；
4. **多个代码块**：思考过程一个块，答案一个块；
5. **伪 JSON**：单引号、尾逗号、中文全角引号、`//` 注释；
6. **截断输出**：max_tokens 打断，括号不闭合。

`json.loads` 对以上全部抛异常。很多人的第一反应是"再调一次"，但重试不是解析，是赌博。

## 做法：分层降级的解析管道

核心思路：**从最便宜、最规范的处理开始，逐层降级，每层只解决一类问题。**

```python
import json, re

FENCE_RE = re.compile(r"```(?:json)?\s*\n(.*?)```", re.DOTALL)

def slice_balanced(text: str, start: int):
    depth, in_str, esc = 0, False, False
    for i in range(start, len(text)):
        c = text[i]
        if in_str:
            if esc: esc = False
            elif c == "\\": esc = True
            elif c == '"': in_str = False
            continue
        if c == '"': in_str = True
        elif c in "{[": depth += 1
        elif c in "}]":
            depth -= 1
            if depth == 0:
                return text[start:i+1]

def parse_llm_json(text: str):
    text = text.strip().lstrip("\ufeff")
    try:
        return json.loads(text)           # L1: 本身就是纯 JSON
    except json.JSONDecodeError:
        pass
    for block in FENCE_RE.findall(text):  # L2: 剥围栏，逐块尝试
        try:
            return json.loads(block)
        except json.JSONDecodeError:
            continue
    starts = [i for i in (text.find("{"), text.find("[")) if i >= 0]
    if starts:                            # L3: 括号配平截取
        try:
            return json.loads(slice_balanced(text, min(starts)))
        except (json.JSONDecodeError, TypeError):
            pass
    raise ValueError("unparseable LLM output")  # L4: 交给上层修复或重试
```

两个关键设计：

- **L3 用括号配平而不是正则贪婪匹配**。`\{.*\}` 会把噪声里的花括号一起吞进来；配平计数器从第一个 `{` 走到对应闭合，且处理了字符串内的引号转义。
- **解析成功 ≠ 任务成功**。L1–L3 之后必须接 schema 校验（pydantic 或 jsonschema），字段类型、枚举值、必填项不合格照样打回。

失败兜底：把解析错误信息（出错位置附近的原文片段）回喂给模型重试，temperature 设 0，最多两次。截断类输出不要试图修复，直接重试并调大 token 上限。

## 踩坑点

- **贪婪正则**：`r"```.*```"` 会把两个独立代码块连成一个，务必非贪婪 + `DOTALL`；
- **`rfind` 陷阱**：噪声文本里也可能有 `{`，从最后一个括号回溯常常截到错误片段，用第一个开括号正向配平更稳；
- **全角引号静默失败**：`“ ”` 肉眼难辨，报错只给一个 column 值，日志里务必保留原始输出片段；
- **BOM 和零宽字符**：部分输出带 `\ufeff`，`strip()` 去不掉，需要显式清除；
- **盲目重试**：不区分错误类型一律重试没有意义，截断和语法错误的应对策略完全不同。

## 可复用建议

1. 把解析器做成独立模块，全项目共用一份，别每个插件复制粘贴；
2. 优先用 provider 原生结构化输出（JSON mode / schema 约束），防御性解析是兜底，不是替代；
3. 日志里保存失败样本的原始输出，攒两周就能发现自家场景的高频脏模式，针对性补规则；
4. prompt 里放一个 few-shot 输出示例，比重复十遍"只要 JSON"有效得多。

## 总结

LLM 的输出本质上是不可信输入，解析层要按处理第三方 API 的标准来写：分层降级、语义校验、可观测。一个 50 行的防御式解析函数，换来的是自动化流水线在真实世界里的存活率。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/1751059c6175c9f0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/b2e6cd2a13d2fc86.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/19f5eb884106438a.png)

