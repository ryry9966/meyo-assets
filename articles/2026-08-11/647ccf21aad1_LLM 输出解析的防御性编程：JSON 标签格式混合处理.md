---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 32526
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景：当“结构化输出”并不结构

在 OpenClaw 这类 Agent 系统中，我们经常通过提示词要求 LLM 返回 JSON 格式的结果，用于工具调用、状态更新或 MCP 服务间通信。理论上，只要 prompt 写得足够清晰，模型应该直接输出纯净的 JSON 字符串。但在工程实践中，模型输出往往被“污染”——混入 markdown 代码块标记、前缀说明文字、尾部注释，甚至使用不规范的 JSON 语法。

例如，一个典型的工具调用输出会是：

```
Here is the result:
```json
{"action": "search", "query": "OpenClaw documentation"}
```
Let me know if you need anything else.
```

如果直接调用 `json.loads()`，解析器会立即抛出异常，导致 Agent 流程中断。在长时间运行的自动化流水线中，这种不稳定会大幅降低任务完成率。因此，我们需要一套防御性的 JSON 提取与解析策略。

## 问题拆解

从工程角度，混合格式带来的问题可以归纳为三类：

1. **包裹层污染**：模型使用 Markdown 代码块（```` ```json … ``` ````）或普通代码块包裹 JSON；
2. **头尾噪声**：JSON 前后被附加自然语言说明、标点甚至乱码；
3. **语法容错**：JSON 本身并不完全合法——尾逗号、单引号字符串、注释等在 LLM 输出中常见。

我们的目标是设计一个鲁棒的解析函数，在保证无副作用的前提下，尽可能多地恢复出有效 JSON 对象/数组，并为上层调用方提供可观测性和回退机制。

## 做法：分层提取 + 正则兜底

这里给出一个经过实际项目验证的 Python 实现。核心思路是分层尝试，每一层只做最稳妥的操作，避免过度清洗造成数据损坏。

```python
import re
import json

def extract_json(text: str) -> dict | list | None:
    """防御性提取文本中的第一个有效JSON对象或数组。"""
    # 0. 直接解析，最快路径
    try:
        return json.loads(text.strip())
    except json.JSONDecodeError:
        pass

    # 1. 提取 markdown 代码块 (```json ... ```)
    fenced = re.search(r'```(?:json)?\s*\n?(.*?)\n?```', text, re.DOTALL)
    if fenced:
        try:
            return json.loads(fenced.group(1).strip())
        except json.JSONDecodeError:
            pass

    # 2. 寻找第一个平衡的 { 或 [ 到对应闭合
    candidates = re.finditer(r'(\{.*\}|\[.*\])', text, re.DOTALL)
    for m in candidates:
        try:
            return json.loads(m.group(1))
        except json.JSONDecodeError:
            continue

    # 3. 修复常见错误后重试：移除尾逗号
    # （暂不实现单引号等复杂修复，避免引入二义性）
    if fenced:
        cleaned = re.sub(r',\s*}', '}', fenced.group(1))
        cleaned = re.sub(r',\s*]', ']', cleaned)
        try:
            return json.loads(cleaned)
        except json.JSONDecodeError:
            pass

    return None
```

使用时，调用方可以配合日志与重试：

```python
result = extract_json(llm_raw_output)
if result is None:
    logger.warning("JSON extraction failed", extra={"raw": llm_raw_output[:200]})
    # 触发重试逻辑或降级
```

## 踩坑点复盘

在实际操作中，这套简单方案仍会遇到一些棘手场景，需要额外关注。

**1. 嵌套 Markdown 代码块**  
如果模型输出的 JSON 字符串字段中包含反引号或代码块，例如：
```json
{"example": "```python\nprint(1)\n```"}
```
正则 `r'```(?:json)?\s*\n?(.*?)\n?```'` 的非贪婪匹配可能提前截断，导致提取失败。此时需要改用更精确的平衡匹配或预先扫描代码块边界，但在工具调用场景中这种输出极少见，可暂不作为默认策略。

**2. 多个 JSON 对象**  
模型可能返回多个独立的 JSON 对象，如：
```
{"step": 1}
{"step": 2}
```
上面的函数只返回第一个。若业务需要处理多个，可将第 2 步改为收集所有顺序匹配的结果，返回列表。建议明确约定输出为单对象或 JSON 数组，从 prompt 源头减少这种歧义。

**3. 模型输出了“json”标签但不在代码块内**  
有些模型会输出类似 `json { ... }` 的前缀。可以增加一层前缀清洗：如果文本以 `json` 开头，可尝试去除该词后再匹配。但过度清洗会降低可维护性，建议仅在固定模型上使用。

**4. 语法修复的边界**  
我曾尝试自动将单引号替换为双引号，结果导致字符串内容中的单引号被误改，数据损坏。因此，**仅修复尾逗号这种无歧义错误**是相对安全的选择。其他修复（如缺失引号）应留给模型改进或人工处理。

**5. 日志与告警的工程化**  
JSON 提取失败应产生明确的 warning，并记录截断后的原始输出，便于事后分析模型行为。如果项目使用结构化日志，建议将解析失败场景单独标记，长期收集后可以驱动 prompt 优化或模型微调。

## 可复用建议

在 OpenClaw 插件或自动化流水线中，建议将 JSON 提取抽象为一个通用工具模块，并遵循以下原则：

- **分层抽离**：独立的 `extract_json` 函数不依赖任何 Agent 状态，易于单元测试。
- **策略可配置**：通过参数控制是否启用尾逗号修复、是否提取多个对象等，适应不同下游任务。
- **结合 pydantic 校验**：提取后立即用 pydantic 模型对字段进行类型和范围验证，比仅仅拿到 dict 更可靠。
- **重试与降级**：解析失败后，可以重新向 LLM 发起更严格的约束（如“请只输出纯 JSON，不要包含任何其他文字”），或直接使用 Fallback 值保证流程继续。
- **记录原始输出**：将每轮 LLM 输出完整存入上下文或日志，方便调试。

## 总结

防御性 JSON 解析是 LLM 工程化的“最后一公里”问题之一。它看起来微不足道，却直接影响自动化任务的健壮性。通过分层提取、限制性修复和充分的可观测性，我们可以在不侵入模型能力的前提下，大幅降低解析失败率。当你的 Agent 在无人值守环境下稳定运行 500 轮而不崩溃时，这一小段代码就是可靠的基石。

---

