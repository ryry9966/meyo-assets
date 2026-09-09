---
title: 防御性解析 LLM 输出：当  标签、代码围栏和裸 JSON 混在一起
feedId: 36836
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

在 OpenClaw 的插件和 Agent 工作流里，让模型输出结构化数据是最常见的需求之一。典型做法是在 prompt 里约定「用 `<json></json>` 标签包裹结果」，插件端用正则提取后 `json.loads`。这套流程演示时很顺，跑长了就会遇到现实：换模型版本、调温度、上下文变长之后，输出格式开始漂移——有时是代码围栏，有时是裸 JSON，有时标签带属性、没闭合，前后还挂着解释文字。解析代码只按一种格式写，迟早在生产环境翻车。

## 问题

实际遇到过的混合输出大概长这样：

````
好的，结果如下：
<json lang="zh">
{"cmd": "deploy", "note": "先跑 ```bash\nnpm test\n``` 再上线"
````

一段输出里同时出现：说明文字、带属性的标签、未闭合标签、值内嵌套围栏。任何单一正则都会在某种形态上失效。

## 做法：分层提取 + 校验 + 有界重试

核心思路是把模型输出当作**不可信输入**，解析收敛到一个入口，按命中率从高到低分层尝试：

```python
import re, json

TAG_RE   = re.compile(r"<json\b[^>]*>(.*?)</json>", re.I | re.S)
FENCE_RE = re.compile(r"(`{3,})[ \t]*(?:json)?[^\n]*\n(.*?)\1", re.S)

def extract_json(text: str):
    dec, cands = json.JSONDecoder(), []
    cands += [m.group(1) for m in TAG_RE.finditer(text)]
    cands += [m.group(2) for m in FENCE_RE.finditer(text)]
    for c in cands:                              # 标签 → 围栏
        try:
            return dec.raw_decode(c.strip().lstrip("\ufeff"))[0]
        except json.JSONDecodeError:
            continue
    for i, ch in enumerate(text):                # 裸 JSON 兜底
        if ch in "{[":
            try:
                return dec.raw_decode(text, i)[0]
            except json.JSONDecodeError:
                continue
    raise ValueError("no parsable json")
```

关键点：

1. **标签匹配用 `<json\b[^>]*>`**，兼容带属性、大小写、跨行的写法。
2. **围栏用捕获组 + 反向引用 `\1`**。模型有时用四个反引号包裹内容里含三反引号的片段，非贪婪的固定三反引号会提前截断；按「开几个关几个」配对能少踩一个坑。
3. **裸 JSON 兜底用 `raw_decode(text, i)`**，从每个 `{` / `[` 位置尝试消费一个完整值，能容忍前后杂质。它可能捞到正文里的示例片段，所以放最后，且必须过 schema。
4. **提取成功 ≠ 数据可用**。过一遍 pydantic / jsonschema 校验，字段缺失、类型不对直接拒绝，别让脏数据流进下游工具调用。
5. **失败后做一次有界纠错**：把解析错误连同原输出回传模型重试，最多两次。原始输出落盘，作为失败语料。

## 踩坑点

- **贪婪匹配**：`.*` 不加 `?` 会把两个块之间的内容吞进来，表现是「偶尔解析出莫名其妙的合并对象」。
- **未闭合标签**：正则匹配不到就走围栏和兜底路径即可，不要为了「挽救」去补全半截 JSON——补出来的大概率不是模型想表达的。
- **流式输出**：闭合标记没到就解析必然失败。先缓冲，见到闭合标记或流结束再解析。
- **修复要克制**：尾逗号、全角引号这类修复只在直接解析失败后做，且修复完必须重新校验。无条件全局替换引号会破坏字符串值本身。
- **多块输出**：策略要定死——按顺序取第一个过校验的块，而不是「随便取一个能用的」，否则同一 prompt 两次运行可能拿到不同结果。

## 可复用建议

- **平台能力优先**：能用原生 structured output / function calling / JSON mode 就别用标签约定，标签方案是兼容层，不是首选。
- **失败语料即测试用例**：每次线上解析失败，脱敏后存进 fixtures，解析器的测试随真实故障增长，回归有底。
- **监控解析成功率**：按 prompt 版本统计。成功率突降通常意味着模型或 prompt 漂移了，比用户报障早得多。
- **解析器单独成模块、带版本号**：不要每个插件各写一份正则。

## 总结

LLM 的输出格式约定是概率性的，解析就要按最坏情况写。分层提取解决「格式漂移」，schema 校验解决「格式对但数据错」，有界重试和失败语料让系统随时间变强。这三层成本都很低，缺了任何一层，都会在某个深夜还你一条线上告警。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/9df204f29250a29a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/603351155e18be33.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/f5e8d6a626cc8abb.png)

