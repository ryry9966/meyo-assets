---
title: LLM 输出解析的防御性编程：JSON 混合格式的一条处理流水线
feedId: 36255
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

在 OpenClaw 的插件、MCP 工具和子 Agent 通信里，让模型返回 JSON 几乎是标配。但跑得够久就会发现，模型给的 JSON 从来不"纯"：同一段提示词，今天是 ```json 代码块，明天是裸 JSON，后天前面多一句"好的，以下是结果"，偶尔还混进 `<result>`、`<tool_call>` 这类自造标签。解析层如果只写一行 `json.loads(resp)`，上线迟早出事。

## 问题

实际踩到的失败模式大致五类：

1. **包裹层混乱**：fenced block、裸 JSON、XML 风格标签、前后解释文字，四种形态随机出现；
2. **语法瑕疵**：尾逗号、单引号、`True/None`、字符串里未转义的换行；
3. **截断**：`max_tokens` 打满，JSON 只有一半；
4. **多对象**：一次输出里拼了两段 JSON；
5. **静默漂移**：换模型、换供应商、升版本，输出格式跟着变，测试全绿上线就挂。

本质是同一件事：**模型输出是不可信输入**，解析层要做防御，不是信任。

## 做法

我们的做法是一条"逐层降级"的解析流水线，核心骨架：

```python
import json, re
from json_repair import loads as repair_loads

FENCE_RE = re.compile(r"```(?:json)?\s*(.*?)```", re.S)
TAG_RE   = re.compile(r"<\w+>\s*([\[{].*?[\]}])\s*</\w+>", re.S)

def candidates(text: str):
    yield text.strip()
    yield from (m.strip() for m in FENCE_RE.findall(text))
    yield from (m.strip() for m in TAG_RE.findall(text))
    yield from balanced_braces(text)   # 字符串感知的 {} / [] 平衡扫描

def parse_llm_json(text: str):
    for c in candidates(text):
        try:
            return json.loads(c)
        except json.JSONDecodeError:
            continue
    return repair_loads(text)          # 修复层：尾逗号/单引号/截断
```

对应四步：

1. **直接解析**：最干净的情况，一次 `loads` 命中；
2. **候选提取**：按 fenced → 标签 → 花括号扫描的顺序抽候选逐个试。花括号扫描必须感知字符串（跳过引号内的 `{}`），嵌套对象交给平衡扫描兜底；
3. **修复层**：交给 `json_repair` 处理尾逗号、引号、截断。修复永远放在提取失败之后，不放在最前面；
4. **校验与重试**：解析成功 ≠ 数据正确，过一遍 pydantic/jsonschema；失败且预算允许时，把原始输出加具体报错发回模型要求修正，**最多重试一次**。

## 踩坑点

- **贪婪正则**：`{.*}` 会把 prose 里的多段 JSON 一网打尽，返回"看起来对其实错位"的数据，用非贪婪加候选列表代替单一大正则；
- **修复层太激进**：截断的 JSON 可能被"补"成合法但语义错误的对象（数组被截成一项），修复后必须过 schema，不能只看解析成功；
- **重试无上限**：坏 prompt 上没 cap 的重试循环纯烧钱，我们定死 1 次并把失败原文落盘；
- **只修解析器不修 prompt**：先统计失败分布，80% 是同一种包裹格式的话，改提示词或用 structured output 模式比堆正则划算。

## 可复用建议

- 插件体系里做**统一规范化层**：所有模型输出先过同一个 `parse_llm_json`，业务代码不直接碰原始文本；
- 每次解析记录**命中了哪条策略**（direct / fence / tag / brace / repair / retry），这是排障时最有用的字段；
- 有 JSON mode 就用，但**保留降级路径**——各家对 JSON mode 的实现并不等价；
- schema 对未知字段宽容（`extra="allow"`），模型升级时加字段不至于全线崩。

## 总结

模型输出的解析是典型的边界问题：格式不可控，但工程可控。一层提取、一层修复、一层校验、一次有上限的重试，再配上命中率日志，就能把"偶尔解析失败"变成可观测、可收敛的工程问题。别指望模型永远乖，也别让解析器替 prompt 背锅。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/a52a4b4f72bb9b8c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/97eb5df42b72b7b9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/f339e35dae04574a.png)

