---
title: LLM 输出解析的防御性编程：JSON 与标签混合格式的容错提取
feedId: 35556
source: 综合讨论
publishedAt: 2026-08-31
---

在 OpenClaw 的 Agent 流程、MCP 工具参数回传、插件配置生成等场景里，我们经常要求模型输出 JSON。但真实环境中，模型并不会每次都规规矩矩返回裸 JSON：有时包在 ```json 代码块里，有时用 `<json>` 或 `<output>` 标签，有时前面先解释两句，有时甚至把多个候选 JSON 混在同一段文本中。直接 `json.loads()` 会让自动化流水线频繁中断。

这篇文章整理一个我实际在 OpenClaw 工具调用边界使用的防御性解析思路，不依赖单一正则，也不假设模型“听话”。

## 典型问题

假设你让模型返回：

```json
{"tool": "search", "args": {"query": "OpenClaw MCP"}}
```

实际可能收到：

- ```json
  {"tool": "search", "args": {"query": "OpenClaw MCP"}}
  ```
- 解释 + JSON + 收尾客套
- `<json>{"tool": "search"}</json>`
- 代码块里再套标签，或标签里再套代码块
- 多个 JSON 对象，第一个不完整，第二个才有效
- JSON 内字符串包含 `}`、`]`、`"`，导致简单正则截断

如果解析器只处理其中一种，Agent 的稳定性会很差。

## 做法/步骤

我目前采用“候选提取 + 逐个解析 + schema 校验”的三段式方案，核心函数只做纯文本处理，不访问模型，不抛未封装异常。

### 1. 归一化输入

先去除常见噪声：零宽字符、BOM、NBSP、全角花括号等。注意不要过度清洗，否则可能破坏 JSON 字符串内的合法内容。

```python
import json
import re

def normalize(raw: str) -> str:
    text = raw.replace("\u00a0", " ")
    text = re.sub(r"[\u200b-\u200d\ufeff]", "", text)
    text = text.replace("“", '"').replace("”", '"')
    text = text.replace("‘", "'").replace("’", "'")
    return text
```

### 2. 提取候选块

候选来源按优先级排列：

- fenced code block：```json ... ```
- XML-like 标签块：`<json> ... </json>`、`<output> ... </output>`、`<result> ... </result>`
- 平衡容器扫描：从第一个 `{` 或 `[` 开始，用栈扫描到配对结束，忽略字符串内的括号
- 整段文本兜底

```python
def extract_json_candidates(text: str):
    candidates = []

    # 代码块
    for m in re.finditer(r"```(?:json|JSON)?\s*(.*?)```", text, re.S):
        candidates.append(m.group(1))

    # 标签块
    for tag in ("json", "output", "result", "response"):
        for m in re.finditer(rf"<{tag}>\s*(.*?)</{tag}>", text, re.S | re.I):
            candidates.append(m.group(1))

    # 平衡容器
    candidates.extend(balanced_extract_candidates(text))

    # 整段文本
    candidates.append(text.strip())
    return dedupe_preserve_order(candidates)
```

其中 `balanced_extract_candidates` 可以用栈实现：遍历字符，遇到 `{` 或 `[` 入栈，遇到 `}` 或 `]` 出栈，栈空时记录一个完整候选。遇到字符串状态时跳过括号。

### 3. 逐个尝试解析

不要只取第一个候选项。按候选长度和出现顺序排序，逐个 `json.loads`。成功一个就进入下一层校验。关键点是：解析失败不抛出，而是记录错误，继续尝试下一个候选。

```python
def safe_parse(text: str):
    candidates = extract_json_candidates(normalize(text))
    errors = []
    for cand in candidates:
        try:
            return json.loads(cand)
        except Exception as e:
            errors.append(str(e))
            continue
    return None, errors
```

### 4. schema 二次校验

JSON 解析成功不代表字段完整。OpenClaw 工具调用可以配合 Pydantic 或 dataclass 做 model_validate。例如工具名、arguments、content 等字段必须存在，arguments 必须是 dict。缺失字段不靠默认值硬填，关键任务直接进入重试流程。

### 5. 有限修复与失败回退

常见的小问题可以有限修复：尾部逗号、单引号替换为双引号、`NaN`/`Infinity` 替换为 `null`。但这些修复必须放在白名单里，不要引入 `eval`，也不要用模糊的“万能正则”。如果全部失败，返回结构化错误对象，并带上“无效原因 + 前若干字符”，方便日志和重试提示。

## 踩坑点

- **不要用 `re.DOTALL + 贪婪匹配`** 跨块抓 `{.*}`，多个 JSON 对象时会把第一个 `{` 和最后一个 `}` 连起来。
- **标签大小写和属性**：`<JSON>` 或 `<json type="object">` 会让简单标签正则漏掉。建议简单标签只做低优先级候选，复杂情况交给平衡扫描。
- **字符串内部出现结束符**：比如 JSON 的某个 value 里包含 `}`、`]` 或 `</json>`，基于字符串匹配的提取会提前截断。平衡扫描更能保证完整。
- **多个候选块**：模型可能先输出一个解释性片段，再输出真正 JSON。必须逐个尝试，不要直接取第一个。
- **别用 `eval` 或 `exec`**：单引号伪 JSON 虽常见，但一旦执行任意代码风险极高。宁可解析失败进入重试，也不要放宽到动态执行。
- **异常吞噬要有限度**：解析失败需要记录原始片段和错误，但不要把超大原文全部打进主日志，容易刷屏。

## 可复用建议

1. 把 `extract_json_candidates` + `safe_parse` 做成独立纯函数，放在 OpenClaw 工具调用、MCP 返回解析、插件配置加载的统一入口。
2. 解析成功后必须再过 schema 校验，字段类型错误比 JSON 错误更难排。
3. 保留失败样本，形成回归集。每次遇到新格式，往解析器里加候选规则或有限修复项，但不要无限膨胀。
4. 对关键任务，提示词可以要求“只输出一个 JSON 对象，不要使用代码块或标签”，但防御性解析仍然必须有。
5. 重试时把解析错误原因反馈给模型，例如“第 12 行出现尾随逗号，请输出严格 JSON”，比简单说“JSON 无效”更有效。

## 总结

LLM 输出解析的核心不是写好一个正则，而是建立“候选提取 + 逐个尝试 + 结构校验”的防御性流水线。混合 JSON 与标签格式不是偶发问题，而是 Agent、MCP、插件自动化里的常态。与其在四处散落 `json.loads()`，不如把容错逻辑收敛到一个纯函数，统一处理、统一返回、统一记录失败。这样 OpenClaw 的自动化流程才能真正抗住模型输出的不确定性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/f373492cdfe37344.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/463710d5fae146c1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/9786a6f13d3144db.png)

