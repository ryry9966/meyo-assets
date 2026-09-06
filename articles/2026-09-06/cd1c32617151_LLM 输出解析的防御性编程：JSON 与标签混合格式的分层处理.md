---
title: LLM 输出解析的防御性编程：JSON 与标签混合格式的分层处理
feedId: 36359
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

在 OpenClaw 的 skill / 插件开发里，让模型输出一段 JSON 再交给下游工具，是最常见的集成方式。理想很美好，现实是：同一份 prompt，今天给你纯 JSON，明天包一层 ```json 围栏，后天在 `<result>` 标签里塞 JSON、外面还附一段中文说明。接本地小模型（比如 Ollama 跑的 7B/14B）时，格式漂移几乎必然发生。解析层不设防，整条自动化链路的稳定性就取决于模型的"心情"。

## 问题

裸 `json.loads()` 的失败形态大致五类：

1. **污染**：围栏、前后缀解说文字；
2. **标签包裹**：`<result>...</result>`、`<json>`，且标签名会漂移；
3. **截断**：`max_tokens` 打满，JSON 只写了一半；
4. **伪 JSON**：单引号、尾逗号、全角引号；
5. **多对象**：模型先写示例再写答案。

多数人的第一反应是失败就重试。重试两三次仍挂，链路断掉，而且原始输出没留存，无法复盘。

## 做法：四层解析管线

**第一层，Prompt 立契约。** 给精确 schema 示例，明确"只输出一个 JSON，无围栏无解释"；能用结构化输出 / 工具调用就优先用，MCP 工具参数本身就是约束。但本地模型不保证配合，解析层仍是必备兜底。

**第二层，分层抽取**，按成本从低到高依次尝试，而不是堆一个巨型正则：

```python
import json, re

def balanced(text):                      # 括号配平扫描，取第一个完整对象
    for o, c in (("{", "}"), ("[", "]")):
        s = text.find(o)
        if s < 0: continue
        depth, in_str, esc = 0, False, False
        for i, ch in enumerate(text[s:], s):
            if in_str:
                if esc: esc = False
                elif ch == "\\": esc = True
                elif ch == '"': in_str = False
            elif ch == '"': in_str = True
            elif ch == o: depth += 1
            elif ch == c:
                depth -= 1
                if depth == 0: return text[s:i + 1]

def candidates(text):
    yield text.strip()
    yield from (m.group(2) for m in
                re.finditer(r"<(\w+)>(.*?)</\1>", text, re.S | re.I))
    yield from re.findall(r"```(?:json)?\s*(.*?)```", text, re.S)
    if (b := balanced(text)): yield b

def parse_llm_json(text):
    for c in candidates(text):
        try: return json.loads(c)
        except json.JSONDecodeError: continue
    raise ValueError(f"unparseable: {text[:200]}")
```

**第三层，Schema 校验。** parse 成功 ≠ 数据可用。用 pydantic 校验字段、类型、枚举，不合规按失败处理。

**第四层，兜底与留痕。** 仍失败时，把解析错误连同原文喂回去重试一至两次（设上限）；无论成败，原始输出先落盘。

## 踩坑点

- **贪婪正则 `\{.*\}` 是重灾区**：会从第一个 `{` 吃到最后一个 `}`，模型先给示例再给答案时必然抓错。配平扫描取"第一个完整对象"稳得多，注意跳过字符串内部的括号和转义。
- **截断不是坏 JSON，是数据缺失**。repair 库硬补个空壳对象，下游拿空值继续跑，比直接报错危险。先看 `finish_reason` / 长度，截断走重生成，别走修复。
- **中文场景高发全角引号、全角冒号**。直接全局替换可能破坏字符串内容，只对 parse 失败的候选做，且失败要留日志。
- **标签名会漂移**：你要求 `<result>`，它回 `<Result>`、`<output>`。匹配做小写化，标签只当候选之一，别当唯一依赖。
- **流式场景**记得缓冲到完整 JSON 再解析，别对增量片段做抽取。

## 可复用建议

- 抽取器做成**单一工具模块**，所有 skill 共用一个入口，埋点、升级只改一处。
- 每次失败样本都收进"脏输出语料库"，进 CI 做回归——解析器迭代最有价值的资产就是历史故障样本。
- **Schema 优先**：先定 pydantic 模型，再从模型生成 prompt 中的示例，保证两边一致。
- 原始输出永远先落盘再解析，解析器以后改进了还能离线重放。

## 总结

输出解析本质上是在和概率系统签合同。别假设模型守约，把容错做成分层管线，把校验和日志做成默认行为。parser 的健壮性，决定整条 agent 链路的可用性下限。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/864bd71360c037b0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/4f5affd3af1452c0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/1126fcb64d87212f.png)

