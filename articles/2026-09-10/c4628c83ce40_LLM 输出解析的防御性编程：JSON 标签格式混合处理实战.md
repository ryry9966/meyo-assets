---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 36890
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

在 OpenClaw 插件和 MCP 工具开发里，让模型返回结构化 JSON 是常态：工具参数、路由决策、字段抽取全依赖它。但模型输出天然不可控——同一条 prompt，今天给纯 JSON，明天裹一层 ```` ```json ```` 围栏，后天在 JSON 前后各加一句客套话。解析层如果不做防御，上游一次输出抖动就能让整条自动化链路挂掉。

## 问题

线上真实输出大致有这几种形态：

1. 纯 JSON（理想情况，占比没那么高）
2. markdown 围栏包裹，语言标注五花八门：`json` / `JSON` / `jsonc`，甚至不写
3. JSON 前后混着自然语言："好的，以下是结果：`{...}` 如需调整请告诉我"
4. JSON 字符串值里嵌套代码块，内部还有反引号和花括号
5. 轻微语法瑕疵：尾逗号、智能引号（`""''`）、全角标点
6. 被 `max_tokens` 截断的不完整 JSON

`json.loads()` 只认第 1 种；简单正则剥围栏会在第 4 种翻车。更麻烦的是这类故障开发期很难发现——测试时模型往往很"乖"。

## 做法：四层解析管线

我们收敛为一个统一的 `parse_llm_json()`，内部分四层依次降级：

- **L1 直接解析**：`json.loads` 原文，命中即返回，这是 90% 以上的快路径
- **L2 剥围栏**：正则提取围栏块，优先取标注了 json 的那块，而不是第一块
- **L3 括号配平提取**：从文本找第一个 `{` 或 `[`，做字符串感知的花括号计数（进入字符串后忽略 `{}`，处理转义），截取配平片段，专治"前后带废话"
- **L4 修复重试**：仍失败则依次尝试去尾逗号、智能引号转直引号，全部失败抛出带原文的结构化错误

核心代码骨架：

```python
import json, re

FENCE_RE = re.compile(r"```(?:json|jsonc|JSON)?\s*\n(.*?)```", re.S)

def parse_llm_json(text: str):
    try:
        return json.loads(text)          # L1
    except json.JSONDecodeError:
        pass
    m = FENCE_RE.search(text)            # L2
    for c in ([m.group(1)] if m else []) + [text]:
        snippet = extract_balanced(c)    # L3
        if snippet:
            for s in (snippet, repair(snippet)):   # L4
                try:
                    return json.loads(s)
                except json.JSONDecodeError:
                    continue
    raise ParseError(raw=text)

def extract_balanced(s: str):
    starts = [i for i in (s.find("{"), s.find("[")) if i != -1]
    if not starts:
        return None
    depth, in_str, esc = 0, False, False
    for j in range(min(starts), len(s)):
        ch = s[j]
        if in_str:
            if esc: esc = False
            elif ch == "\\": esc = True
            elif ch == '"': in_str = False
            continue
        if ch == '"': in_str = True
        elif ch in "{[": depth += 1
        elif ch in "}]":
            depth -= 1
            if depth == 0:
                return s[min(starts):j+1]
    return None   # 未配平，多半是截断

def repair(s: str) -> str:
    s = re.sub(r",\s*([}\]])", r"\1", s)     # 尾逗号
    return s.replace("\u201c", '"').replace("\u201d", '"')
```

## 踩坑点

- **截断要单独判断**：`extract_balanced` 返回 None 通常意味着 `max_tokens` 截断，此时修复无意义，应检查 `finish_reason` 并考虑缩小 schema 或分步输出。
- **修复要有度**：全角转半角对中文内容字段是破坏性的。只动结构性位置，内容字段的脏数据交给 schema 校验兜底，别在解析层"自作聪明"。
- **别静默吞错误**：失败时把原始输出落日志，统计解析失败率——这个指标能反过来暴露 prompt 设计问题。
- **多个 JSON 块**：模型有时给两块代码（一个示例一个结果），"标注了 json 的块"优先级应高于"第一块"。
- **别迷信 json mode**：`response_format` 可用时优先用，但插件要兼容不支持的模型，解析层防御不能省。

## 可复用建议

- **prompt 层**：给明确的 JSON Schema 或 few-shot，加一句"只输出 JSON，不要任何其他内容"，能砍掉大半脏输出
- **代码层**：`parse_llm_json` 做成独立模块全插件共用，不要每个插件各写一套正则
- **校验层**：解析成功后用 pydantic 按 schema 校验字段类型——JSON 合法不代表字段对
- **数据层**：收集解析失败的原始输出，定期回放成回归测试集

## 总结

LLM 输出解析的防御性编程，本质是承认"模型是概率性的"，然后在确定性的代码层把不确定性收敛掉。这套四层管线不过几十行代码，却决定了一个 OpenClaw 插件是"演示能用"还是"挂在网上跑一周不用管"。建议直接抄走 `parse_llm_json`，再按自己场景补充修复规则。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/fbcc68c6e3fc650b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/3b319d90d2ed8a99.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/bade991f923813e0.png)

