---
title: 防御性 JSON 解析：从容应对 LLM 文本与标签混合输出
feedId: 31396
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景：为什么你需要一个“防弹”的 JSON 提取器

在以 OpenClaw 搭建的自动化工作流、Agent 与 MCP 工具链中，LLM 的结构化输出几乎是刚需——无论是工具调用参数、结构化任务分解，还是向插件传递配置。理论上我们可以要求模型“只输出 JSON”，但实际运行中，推理引擎可能返回这些形态：

- 纯 JSON 对象：`{"key": "value"}`
- 被 markdown 代码块包裹：````json\n{"key":"value"}\n````
- 前导说明文本 + JSON：`这是分析结果：\n{"score": 0.95}`
- 多个 JSON 块（多步推理场景）：`{"step":1}\n{"step":2}`
- 包含注释、尾随逗号、单引号的无效 JSON
- 流式输出被意外截断的 JSON 片段

如果直接对 `llm_response` 调用 `json.loads()`，大概率会在真实流量中翻车。这类解析错误往往在上下游中传播，导致 Agent 流程中断，且排查成本不低——原始输出可能因隐私或审计原因未被完整保存。

因此，我们需要一个**防御性编程**的 JSON 解析层，它不是“一次性写完就能永远安心”，而是一套逐步试探、容错降级的策略。

---

## 问题拆解：五种常见“不干净”输出形态

在 OpenClaw 的 Agent 循环中，我们观察到的典型案例可归纳为五种：

1. **干净 JSON**：`{"files":["a.txt","b.txt"]}` —— 可直接解析。
2. **Markdown 代码块包裹**：````json\n{"files":["a.txt"]}\n```` —— 模型为可读性加上了语法标记。
3. **前导文本混合**：`根据要求，输出如下：\n{"action":"read","path":"/data"}` —— 说明文字在前。
4. **多个 JSON 对象**：模型在一次回答中输出两个工具调用序列，用换行分隔。
5. **类 JSON 但非法**：包含了单行注释 `// tool config`、尾随逗号 `"key":"val",}`，或键名未使用双引号。

如果在每个工具调用点单独处理这些变体，代码会迅速膨胀且不可维护。更好的做法是封装一个**可复用的解析函数**，并注入到所有依赖 LLM 结构化输出的节点中。

---

## 做法与步骤：构建 `robust_json_extract`

下面给出一个可运行的 Python 实现，已在 OpenClaw 的若干插件中验证。它按优先级顺序尝试不同策略，直到成功或返回未解析的信号。

```python
import re
import json
from typing import Optional, Union, List

def robust_json_extract(text: str, multi: bool = False) -> Union[dict, list, List[dict], None]:
    """防御性提取文本中的 JSON 对象或数组。
    若 multi=True，返回列表；否则返回第一个有效 JSON，不成功即 None。
    """
    if not text or not isinstance(text, str):
        return None

    candidates = []
    # 策略1：提取所有 ```json ... ``` 代码块
    code_blocks = re.findall(r'```(?:json)?\s*\n?(.*?)```', text, re.DOTALL)
    candidates.extend(block.strip() for block in code_blocks if block.strip())

    # 策略2：若未找到代码块，尝试寻找顶级 { } 或 [ ] 区域
    if not candidates:
        # 匹配第一个完整 JSON 对象或数组（简单启发式）
        brace_match = re.search(r'(\[.*\]|\{.*\})', text, re.DOTALL)
        if brace_match:
            candidates.append(brace_match.group(0))

    # 策略3：整个原文作为候选（当策略1/2无收获时）
    if not candidates:
        candidates.append(text.strip())

    parsed = []
    for cand in candidates:
        # 先尝试直接解析
        result = _safe_json_parse(cand)
        if result is not None:
            parsed.append(result)
            if not multi:
                return parsed[0]
        else:
            # 进一阶容错：去除常见错误
            repaired = _repair_json(cand)
            if repaired is not None:
                parsed.append(repaired)
                if not multi:
                    return parsed[0]

    if multi and parsed:
        return parsed
    return None

def _safe_json_parse(s: str) -> Optional[Union[dict, list]]:
    try:
        return json.loads(s)
    except json.JSONDecodeError:
        return None

def _repair_json(s: str) -> Optional[Union[dict, list]]:
    """简单修复常见格式错误，返回解析后的对象，失败返回 None。"""
    # 去掉行注释//（粗暴，但有效）
    s = re.sub(r'//.*', '', s)
    # 去掉块注释 /* ... */
    s = re.sub(r'/\*.*?\*/', '', s, flags=re.DOTALL)
    # 移除尾随逗号（数组和对象中）
    s = re.sub(r',\s*([}\]])', r'\1', s)
    # 替换单引号为双引号（仅限键或字符串值风险很大，此处仅作参考，谨慎使用）
    # 偏保守：只尝试将键的单引号替换为双引号
    s = re.sub(r"'([^']*?)'\s*:", r'"\1":', s)
    return _safe_json_parse(s)
```

**使用示例（OpenClaw 工具调用节点）**：

```python
def tool_dispatcher(raw_llm_output: str):
    parsed = robust_json_extract(raw_llm_output)
    if parsed is None:
        raise JSONExtractionError("无法从 LLM 输出提取有效 JSON", raw=raw_llm_output)
    # 后续 dispatch
```

若需要收集所有工具调用（多步）：

```python
calls = robust_json_extract(raw, multi=True)
for call in calls:
    process(call)
```

---

## 踩坑点：不能只信正则，也不能只信 JSON 库

在实践中，这套实现可以覆盖大部分场景，但仍有几个易踩的坑需要注意：

- **嵌套代码块**：模型可能输出 ````markdown\n```json\n{"key":"```\""}\n````，正则可能截断。解决方案是使用更严格的语言标识匹配，或者基于缩进判断代码块边界。本项目只提取最外层 ``` ，多数情形足够。
- **多维数组/对象内的换行**：正则 `\[.*\]` 在 DOTALL 模式下可能跨越多行，但贪婪匹配可能选到过大范围。可以改为非贪婪匹配 `\{.*?\}`，但可能遗漏嵌套。我们采用“先取代码块，再直接解析”的方式避免复杂正则。
- **不可见字符 / BOM**：`"{"key": "val"}"` 可能开头有 `\ufeff`。`json.loads` 会报错。可以通过 `s.lstrip('\ufeff')` 预处理。
- **流式输出截断**：Agent 在处理流式输出时，JSON 可能不完整。此时应放弃修复，直接请求模型重试返回完整 JSON，或使用增量解析器（例如 ijson）。不要尝试补齐缺失的引号和括号，容易引入错误语义。
- **键值注释干扰**：在 JSON 值内出现 `//` 文本（如 URL）会被 `_repair_json` 中粗暴的 `//.*` 破坏。在生产环境中，可以限制注释清理的范围：仅在键值对最外层进行处理，或使用 AST 解析。若容忍一定的漏网之鱼，可以简单跳过。

---

## 可复用建议：融入 Agent 的容错链路

基于上述经验，提炼出几条可立刻用于 OpenClaw 项目的建议：

1. **统一入口**：将所有 LLM 文本至 JSON 的转换收敛到同一工具函数（例如 `extract_json`），避免散落各处的正则。
2. **记录原始输出**：解析失败时，在日志或异常中保留原始文本的摘要（注意合规）。这对回溯模型行为至关重要。
3. **设计重试机制**：解析失败后，不要直接放弃流程。向模型发送类似 `你的回复无法解析为有效 JSON，请只返回 JSON 对象。` 的提示，并限制重试次数。
4. **提供 MCP 工具封装**：若你的架构基于 MCP，可将 `robust_json_extract` 封装为一个轻量工具，供多个 Agent 节点复用，并附带健康检查。
5. **监控解析失败率**：在可观测系统中埋点，统计解析失败占比及分布，便于觉察模型退化或 prompt 修改引入的新格式。

---

## 总结

在 LLM 驱动的自动化系统中，输入确定性很低，但下游流程又极度依赖结构。投入两小时写好一个防御性 JSON 解析器，能省去日后无数个定位“为什么突然不 work 了”的深夜。上面给出的 `robust_json_extract` 并非银弹，但它像一道缓冲阀，让你在应对模型输出的千变万化时多一份底气。工程上，下一步还可以考虑结合 JSON Schema 验证，确保提取出的内容符合业务约束——这些都可以在同一个解析层内顺滑扩展。

攻击面永远存在，防御先做一步。

---

