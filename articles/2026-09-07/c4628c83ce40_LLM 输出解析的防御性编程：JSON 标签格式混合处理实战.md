---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 36405
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

在 OpenClaw 插件和 Agent 工作流里，让模型返回结构化 JSON 是最常见的集成方式：工具调用参数、MCP 资源字段、自动化任务的中间结果，几乎都靠它。但实践下来你会发现一个事实——模型“同意”返回 JSON，和“稳定地”返回 JSON，是两回事。

## 问题

同一段 prompt，在不同模型、不同温度、不同上下文长度下，输出形态至少有这几种：

1. 干净的裸 JSON；
2. ```json 围栏包裹；
3. ``` 无语言标注的围栏；
4. 前后各带一句“以下是结果：/希望对你有帮助”；
5. `<json>...</json>` 这类自定义标签包裹；
6. 混合形态：围栏外面还套着说明文字；
7. 细节污染：全角引号、尾逗号、注释、被 max_tokens 截断。

如果解析器只写了一种 pattern，上线后失败率会随着模型或渠道切换明显波动，日志里反复出现 `JSONDecodeError: Expecting property name enclosed in double quotes`。

## 做法

核心思路：**不假设格式，按优先级逐层提取，最后统一 schema 校验**。

```python
import json, re

FENCE = re.compile(r"```(?:json)?\s*\n?(.*?)```", re.S)
TAG   = re.compile(r"<json>\s*(.*?)\s*</json>", re.S)

def extract_json(text: str) -> dict:
    candidates = FENCE.findall(text) + TAG.findall(text)
    s, e = text.find("{"), text.rfind("}")
    if s != -1 and e > s:
        candidates.append(text[s:e + 1])          # 兜底截取
    for c in candidates:
        for attempt in (c, repair(c)):            # 先原样，再修复
            try:
                return json.loads(attempt)
            except json.JSONDecodeError:
                continue
    raise ValueError(f"unparseable output: {text[:200]}")
```

分四步：

- **Step 1 预处理**：去 BOM、去零宽字符。注意不要全局替换全角引号——字符串值里可能合法地包含中文引号，无脑替换会破坏正确数据。
- **Step 2 多模式提取**：围栏正则、标签正则、首 `{` 到尾 `}` 截取，依次生成候选。
- **Step 3 解析与修复**：先 `json.loads` 原文；失败后做最小修复（去尾逗号等），或直接用 `json_repair` 这类库。
- **Step 4 校验与留痕**：过 schema（pydantic / jsonschema），失败时把**原始输出完整落日志**。

## 踩坑点

- **贪婪匹配陷阱**：一段输出里有两个独立 JSON 时，`{.*}` 截取会把中间说明文字一起吃进去，所以截取之后必须靠 schema 拦住。
- **括号配对扫描**要跳过字符串字面量里的 `{}`，否则 `"tpl": "use {brace}"` 会让配对错位。
- **temperature=0 不等于格式稳定**。换模型、扩上下文都可能改变输出形态，解析器要按“总会失败”来设计。
- **失败重试别裸重试**：把报错信息和上次原始输出拼回 prompt 再试，或直接降级到 function calling。
- **截断场景**：fence 没闭合时围栏正则匹配不到，`find/rfind` 兜底反而能救回一部分——但截断残骸最终靠 schema 拦截。

## 可复用建议

1. 格式约定写两遍：system prompt 里写清楚，代码里当它一定会被违反。
2. 优先用平台的结构化输出能力（tool calling / response format），能消掉大部分问题；但做跨模型、跨渠道插件时，兜底解析器仍然必备。
3. 把解析失败率做成指标，失败样本完整存档。
4. 提取逻辑收敛到一个工具模块，不要散落在各个 handler 里各自造 regex。
5. 用真实失败样本攒一个小回归测试集，改解析器时跑一遍。

## 总结

LLM 输出解析的健壮性不来自更强的 prompt，而来自“假设它一定会变形”的分层兜底：多模式提取 → 原样解析 → 最小修复 → schema 校验 → 原始输出落盘。这套逻辑写一次、处处复用，远比每次线上踩坑再临时补 regex 划算。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/4b7839401e0eab34.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/34f438da9e892ca0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/f5986647bbbae0f8.png)

