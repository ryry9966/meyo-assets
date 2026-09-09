---
title: LLM 输出解析的防御性编程：混合 JSON 标签格式的策略链处理
feedId: 36757
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

在 OpenClaw 的 agent 工作流里，“让模型输出 JSON”几乎绕不开：工具调用的结构化参数、多 agent 消息传递、MCP 工具返回结果的二次加工，都依赖下游 `json.loads` 成功。常见做法是在提示词里约定格式——“放进 `<json>` 标签”或“用 ```json 代码块输出”。

问题在于，这份“格式合同”只在大部分时候成立。

## 问题

本地跑通、上线后偶发解析失败，是这类管线的典型症状。收集一段时间的失败样本后，常见变种有：

- 代码块标记不统一：` ```json `、` ```JSON `、` ```jsonc `，甚至裸 ` ``` `；
- 约定了 `<json>` 标签，模型有时外面又套一层代码块，或干脆只用代码块；
- JSON 前后带散文：“以下是分析结果：{...}”；
- 字符串值里本身含 ` ``` ` 或 `{}`，朴素正则和括号计数会被带偏；
- 输出被 max_tokens 截断，JSON 只剩前半截；
- 尾随逗号、单引号、`True`/`None` 这类“长得像但不是”的输出。

每条单看都是小概率，叠加起来在生产环境就是每天几次的 parse error。

## 做法

核心思路：把解析做成一条**降级策略链**，而不是单次尝试。

**第 0 层：直接解析。** 先对原文 `json.loads`，最快路径命中就不折腾。

**第 1 层：模式提取。** 按优先级收集候选片段——自定义标签、各种语言标记的代码块。关键点：所有候选**独立尝试解析，取第一个成功的**，而不是取第一个匹配的。

**第 2 层：括号配平扫描。** 用一个感知字符串边界的状态机：从首个 `{`/`[` 起计数，遇未转义引号进入字符串态，字符串内括号不计数，栈空即截取。专兜“JSON 混在散文里”的情况。

```python
import json, re

PATTERNS = [r"<json>(.*?)</json>",
            r"```(?:json|JSON|jsonc)?\s*\n(.*?)```"]

def scan_balanced(text: str) -> list[str]:
    out, stack, in_str, esc, start = [], [], False, False, -1
    match = {"{": "}", "[": "]"}
    for i, ch in enumerate(text):
        if in_str:
            if esc: esc = False
            elif ch == "\\": esc = True
            elif ch == '"': in_str = False
        elif ch == '"': in_str = True
        elif ch in match:
            if not stack: start = i
            stack.append(match[ch])
        elif stack and ch == stack[-1]:
            stack.pop()
            if not stack: out.append(text[start:i + 1])
    return out

def parse_llm_json(text: str):
    text = text.replace("\r\n", "\n").strip()
    try:    return json.loads(text)          # 第 0 层
    except json.JSONDecodeError: pass
    cands = [c for p in PATTERNS
             for c in re.findall(p, text, re.S)]   # 第 1 层
    cands += scan_balanced(text)                    # 第 2 层
    for c in cands:
        try:    return json.loads(c.strip())
        except json.JSONDecodeError: continue
    raise ValueError("unparseable")   # 第 3 层：截断判定 / 修复 / 重试 / 落日志
```

**第 3 层：失败处置。** 配平扫描后栈非空或引号未闭合，基本可判定截断——不要硬修，直接重试（提高 max_tokens，并明确要求“只输出裸 JSON”）。非截断失败可过一遍 json-repair 类工具，但只作最后手段。无论走哪条路，原始输出和错误位置都落日志。

## 踩坑点

- `r"```json\n(.*?)```"` 配 `re.S` 看似够用，但字符串值里含 ` ``` ` 时会截在半截，所以“每个候选独立 try parse”必须做；
- 括号计数不感知字符串，碰到值里的 `"}"` 就配平错误，且这种错误往往“看起来成功”，比直接报错更危险；
- 全文替换单引号为双引号会破坏字符串值里本就合法的内容。修复要么感知字符串边界，要么交给现成库；
- `\r\n` 会让 fence 匹配静默失败，入口先统一换行；
- 重试提示词只说“重新输出”，模型可能在错误格式上再包一层，要明确“去掉所有代码块标记”。

## 可复用建议

1. 解析器独立成 util 模块，线上失败样本脱敏后存成 fixture 跑回归单测——失败样本是最有价值的测试集；
2. 策略链返回 `(obj, strategy)`，统计各层命中率，命中率突变往往对应上游模型或提示词变更，是免费的监控信号；
3. 解析成功 ≠ 数据正确，后面接 schema 校验，类型错误单独计数——它们比 parse error 更隐蔽；
4. 优先级永远是：API 层结构化输出 / JSON mode > 提示词约束 + 防御解析。解析器是保底，不是借口。

## 总结

对 LLM 输出不要做格式信任，要做格式降级。解析是 agent 管线里最便宜、也最易被忽视的故障点。一条四层策略链加一份持续维护的失败样本回归集，能把“偶发 parse error”变成可观测、可归因、可修复的普通工程问题——这是我们在 OpenClaw 插件管线里跑了几个月后的真实体感。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/0acc01a545871c91.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/c2a2228daaecc0e0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/a0e65c5b9578f6e7.png)

