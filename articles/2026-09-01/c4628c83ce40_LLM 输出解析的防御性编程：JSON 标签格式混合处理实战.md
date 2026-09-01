---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 35694
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

在 OpenClaw 里做 Agent 和插件时，我们大量依赖模型的结构化输出：MCP 工具参数、插件间通信、自动化流水线的中间结果，几乎都靠 JSON 串起来。Prompt 里写得再清楚——"只输出 JSON，不要任何解释"——线上跑久了你会发现，模型对格式的"承诺"本质上是概率性的。

## 问题

同一个任务，换个模型、调一下温度、甚至只是多跑几轮，输出格式就会漂移：

- 直接裸 JSON；
- 包在 ```json 围栏里；
- 套了 `<json>` / `<result>` / `<tool_call>` 之类的标签；
- JSON 前后多一句"好的，以下是结果："；
- 再加上尾逗号、全角引号、字符串里没转义的换行。

如果解析层只写了一句 `json.loads(resp)`，任何一种漂移都会断掉整条链路——而且往往断在半夜的定时任务里。

## 做法

核心思路：把模型输出当不可信输入，解析走一条固定的降级链，而不是单点解析。

```python
import json, re

_FAIL = object()

def extract_json(text: str):
    def parse(s):
        try:
            return json.loads(s)
        except Exception:
            return _FAIL

    # 1) 裸 JSON
    r = parse(text)
    if r is not _FAIL:
        return r

    # 2) markdown 围栏，逐块尝试
    for m in re.finditer(r"```(?:json)?\s*(.*?)```", text, re.S):
        r = parse(m.group(1))
        if r is not _FAIL:
            return r

    # 3) 自定义标签，逐个尝试
    for m in re.finditer(r"<([a-zA-Z_]\w*)>(.*?)</\1>", text, re.S):
        r = parse(m.group(2))
        if r is not _FAIL:
            return r

    # 4) 首尾花括号截取 + 轻度修复
    s, e = text.find("{"), text.rfind("}")
    if 0 <= s < e:
        cand = text[s:e + 1]
        cand = re.sub(r",\s*([}\]])", r"\1", cand)          # 尾逗号
        cand = cand.replace("“", '"').replace("”", '"')     # 全角引号
        r = parse(cand)
        if r is not _FAIL:
            return r

    raise ValueError("unparseable output")
```

解析成功只是第一步，后面必须接 schema 校验（pydantic 或 jsonschema）。校验失败同样进降级链：把具体解析错误和原始输出回传给模型重试，而不是原 prompt 原样重发。

## 踩坑点

1. **贪婪正则跨块吞内容**。`.*` 配 `re.S` 会从第一个围栏吃到最后一个，输出里有两段代码块时直接解析失败。用非贪婪加 `finditer` 逐块尝试。
2. **字符串里的花括号**。JSON 值里含 `"{...}"`（模板、代码片段）时，逐字符数花括号会算错边界。"首个 `{` 到最后一个 `}`"能扛住大多数场景，但要清楚它的局限。
3. **静默修复掩盖问题**。修复动作必须落日志：raw output、命中了降级链第几层、改了什么。否则坏 case 会被"看起来正常"的结果掩盖，排查时两眼一抹黑。
4. **重试不是复读**。把"上次输出多了围栏/有尾逗号"这类具体错误喂回去，重试成功率远高于原样重发。
5. **温度与格式漂移正相关**。结构化任务把 temperature 降下来；模型如果支持原生 structured output 或 tool call，优先用，文本解析只做兜底。

## 可复用建议

- 解析逻辑收敛到一个公共模块，全项目只此一份，别在每个插件里各写各的。
- 降级链顺序固定：裸解析 → 围栏 → 标签 → 括号截取 → 修复 → 重试，每层命中都记日志。
- 收集线上真实坏输出，沉淀成回归语料，parser 每次改动跑一遍，比任何 code review 都可靠。
- Prompt 里给一个精确的 few-shot 示例，比十句"请严格输出 JSON"都管用——但仍然别把它当保险。

## 总结

LLM 的格式输出是概率问题，解析层要按确定性工程来做。一条固定降级链、一份留痕日志、一层 schema 校验、一套坏样本回归，这四件事做完，自动化链路的"半夜断链"会少掉绝大部分。防御性解析不酷，但它决定你的 Agent 能不能稳定跑过第一周。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/755a0919f05ed985.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/54ca8b6a50054425.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/ee4a36f323173ff4.png)

