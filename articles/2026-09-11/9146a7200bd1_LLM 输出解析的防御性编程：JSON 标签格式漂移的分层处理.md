---
title: LLM 输出解析的防御性编程：JSON 标签格式漂移的分层处理
feedId: 37016
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

在 OpenClaw 插件和自动化链路里，让模型把结构化数据包进 `<json>` / `<result>` 标签，再在代码里提取解析，是最常用的轻量结构化方案——不依赖特定模型的 function calling，跨模型可移植。我的几个消息处理插件都是这个模式。

Demo 阶段一切正常，上了自动化之后问题就来了。

## 问题

**格式漂移**。同一段 prompt，模型输出并不稳定：

- 有时输出 ```` ```json ```` 围栏，有时是裸 JSON；
- 有时标签里再套一层围栏；
- 有时前后带解释性文字；
- 偶尔出现尾逗号、全角引号，或因 max_tokens 截断产生半截 JSON。

单次解析失败，整条自动化链路就断。换模型、调高 temperature、输出变长，漂移概率都会明显上升。`temperature=0` 只能降低、不能消除这种不确定性。

## 做法：分层解析，逐层降级

核心思路：**不要假设输出长什么样，按"最规整 → 最脏"的顺序逐层尝试，并记录每层命中路径。**

1. **预处理**：去首尾空白、BOM、零宽字符；
2. **标签提取**：非贪婪正则，容忍大小写和属性（如 `<result lang="zh">`）；
3. **围栏剥离**：提取 ```` ```json ```` 块；标签内嵌围栏同样剥掉；
4. **直接解析 + 边界截取**：候选依次 `json.loads`，全失败则取首个 `{` 到最后一个 `}` 再试；
5. **轻修复**：去尾逗号、替换全角引号；复杂情况直接上 `json_repair` 这类库；
6. **Schema 校验**：JSON 合法不代表字段对，用 pydantic 再验一层；
7. **兜底**：落日志、带纠错提示重试一次，或走降级分支。

核心代码十几行：

```python
import json, re

TAG = re.compile(r"<(json|output|result)\b[^>]*>(.*?)</\1>",
                 re.IGNORECASE | re.DOTALL)
FENCE = re.compile(r"```(?:json)?\s*(.*?)```", re.DOTALL)

def parse_llm_json(text: str):
    text = text.strip().lstrip("\ufeff")
    candidates = [m.group(2).strip() for m in TAG.finditer(text)]
    candidates += [m.group(1).strip() for m in FENCE.finditer(text)]
    candidates.append(text)
    for c in candidates:
        c = re.sub(r"^```(?:json)?\s*|```\s*$", "", c).strip()  # 嵌套围栏兜底
        try:
            return json.loads(c)
        except json.JSONDecodeError:
            continue
    s, e = text.find("{"), text.rfind("}")   # 边界截取
    if s != -1 and e > s:
        frag = re.sub(r",\s*([}\]])", r"\1", text[s:e + 1])  # 去尾逗号
        return json.loads(frag)  # 仍失败则抛异常，交由上层重试/降级
    raise ValueError("unparseable LLM output")
```

## 踩坑点

- **贪婪匹配**：`<json>(.*)</json>` 会把两个独立块吞成一坨，必须非贪婪 + `DOTALL`；
- **嵌套围栏**：模型偶尔输出四反引号包三反引号的块，简单正则会截错边界，兜底里再剥一次围栏值得；
- **截断识别**：边界截取后仍失败，多半是半截 JSON，重试提示里要明确说"输出被截断，请精简"；
- **静默吞异常**：解析失败直接返回 None，坏 case 永远不会被你看见。

## 可复用建议

- 解析逻辑**收敛成一个工具函数**，全项目共用，别在每个 handler 里各写一套；
- 原始输出（脱敏后）全量留日志，每个坏 case 沉淀成测试 fixture 做回归；
- 记录各层命中率（tagged / fenced / raw / repaired），占比数据比直觉更能指导 prompt 怎么改；
- 目标模型支持原生 structured output 时优先用，标签解析只作为兼容层保留。

## 总结

把 LLM 输出当成不可信输入对待，是写 Agent 工具链的基本功。分层降级的解析管线不追求消灭格式漂移，而是保证漂移发生时系统仍能正确工作——这比在 prompt 里反复强调"请严格输出 JSON"可靠得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/ac079caf6165993a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/e75c0e7cad7b066a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/85d6feec9b58a997.png)

