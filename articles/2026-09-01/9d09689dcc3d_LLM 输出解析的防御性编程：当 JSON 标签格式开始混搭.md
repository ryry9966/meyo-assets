---
title: LLM 输出解析的防御性编程：当 JSON 标签格式开始混搭
feedId: 35714
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

在 OpenClaw 的插件和 MCP 工具链里，只要涉及让模型产出结构化数据——路由决策、参数抽取、批量标注——就绕不开解析 JSON。Prompt 里明明写了「只输出 JSON，不要任何解释」，实际拿到的却五花八门。我在写一个批处理插件时被这个问题磨了两天，把踩过的坑整理成这篇。

## 问题：同一种指令，N 种返回

翻完一周的失败日志，模型输出大致落在这几类：

1. ` ```json ... ``` ` 围栏代码块（最常见）
2. 裸 JSON，没有任何包裹
3. 前后带一句「好的，以下是结果：」
4. `<json>...</json>` 之类的自创标签
5. 「近似 JSON」：尾逗号、单引号、`True/None`、全角引号
6. 被 max_tokens 截断的半截 JSON

前四种是格式混搭，第五种是语法瑕疵，第六种根本不是解析问题。用一条正则通吃，必然顾此失彼。

## 做法：分层剥离，而不是硬匹配

核心思路是把解析写成洋葱结构，每层只处理一类形态，命中即返回：

```python
import json, re

FENCE_RE = re.compile(r"```(?:json)?\s*(.*?)```", re.S)
TAG_RE   = re.compile(r"<(json|result)>\s*(.*?)</\1>", re.S | re.I)

def parse_llm_json(text: str):
    raw = text.strip().lstrip("\ufeff")
    candidates = (
        FENCE_RE.findall(raw)          # 1. 围栏块（取全部）
        + [m[1] for m in TAG_RE.findall(raw)]  # 2. 自定义标签
        + [extract_balanced(raw)]      # 3. 括号配平，从噪声里抠对象
        + [raw]                        # 4. 原文兜底
    )
    for c in candidates:
        for s in (c, light_repair(c)):
            try:
                return json.loads(s)
            except json.JSONDecodeError:
                continue
    raise JsonParseError(raw)
```

几个关键点：

- **括号配平扫描**：从第一个 `{` 或 `[` 起，维护深度计数，同时跟踪是否在字符串内、是否有转义。它能从「以下是结果：{...} 希望有帮助」这类文本里准确抠出完整对象，比贪婪正则可靠得多。
- **light_repair 只做三件事**：删尾逗号、全角引号转半角、去注释。更脏的输入（单引号、`True/None`）直接交给 `json_repair` 这类库，别自己堆正则。
- **解析成功后立刻过 schema**（pydantic 或 jsonschema）。JSON 合法 ≠ 数据正确，字段缺失、类型漂移要在这一层拦住。

## 踩坑点

- **贪婪正则跨块吞并**：模型有时一次返回两个代码块，`.*` 会把中间的解释文字一起吃进去。用非贪婪 `.*?` 并取所有匹配。
- **嵌套围栏**：让模型生成含代码示例的内容时，字段值里本身就有 ` ``` `，围栏正则会提前截断。这也是保留括号配平这条兜底路径的原因。
- **截断救不了**：`finish_reason` 是 `length` 时 JSON 断在半截，repair 写得再好也补不回来。先判截断，再谈解析，重试时调大 token 上限。
- **repair 污染数据**：在字符串值里全局替换 `'` 或 `True`，会把用户内容改坏。轻修复只应在「确认原生 loads 失败之后」作为第二遍执行。
- **静默失败最危险**：`except: pass` 返回 None，上游拿去继续跑，bug 会漂到很远的下游。解析失败要抛错并携带原始输出。
- **流式场景别复用终态解析**：增量拿到的是合法前缀，需要单独的增量解析或攒齐再解。

## 可复用建议

1. 解析函数收敛到一个共享 util，全项目统一，不要每个插件各写一套正则。
2. 失败时把 raw 原文完整落日志，这是后续归类的唯一依据。
3. 重试时把上次的错误喂回去：「你上次输出的 JSON 有尾逗号导致解析失败」，实测比原样重试成功率高不少。
4. 能用 provider 原生 structured output 或 tool calling 就优先用，标签解析只做跨模型兜底。
5. Prompt 里给一个精简的输出示例，比反复强调「必须合法 JSON」更有效。

## 总结

防御性解析的前提，是承认模型输出属于「不可信输入」。分层剥离、轻量修复、schema 校验、失败留痕——四件事做齐，插件对输出格式的容忍度会明显上一个台阶。不要试图用一条正则解决所有问题，那恰恰是脆弱的来源。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/0ecc7126148fec5b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/ab2e895468aa2e57.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/dd13de0ed40f27e3.png)

