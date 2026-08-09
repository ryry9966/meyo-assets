---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 32229
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景：为什么“格式约定”不可靠

在基于 LLM 的 Agent 或自动化流程中，我们经常需要模型输出结构化数据，比如：

- OpenClaw 插件调用工具时返回的参数
- MCP 服务间的标准 JSON 消息
- 工作流节点间传递的状态快照

通常的做法是在 prompt 里约定输出格式，例如：

> 请只输出 JSON，不要包含额外解释。

实际运行中你会发现：

- 模型会在 JSON 前后加上 markdown 代码块标记（```json ... ```）
- 或是在 JSON 前后添上一些“好的，以下是结果：”
- 甚至将 JSON 包裹在自定义标签里（如 `<tool_call>...</tool_call>`）
- 某些模型还会在 JSON 内部插入注释或尾逗号

如果解析环节是脆弱的 `json.loads(raw_str)`，Agent 流程可能直接崩溃，尤其在自动化链路里，这种崩溃极难排查，因为日志里只有一行 `JSONDecodeError`。

## 问题分析：混合格式的典型来源

造成“纯 JSON 不可得”的根源，不是提示词写得不够严厉，而是：

1. **SFT/RLHF 习惯残留**：模型在训练时见过大量带代码块标记的对话，倾向美化输出。
2. **多轮上下文污染**：前一轮对话中包含了 markdown 结构，模型会延续这种风格。
3. **不同模型行为差异**：GPT-4 偶尔添加代码块；Claude 在长文本后喜欢加解释；开源模型（如 Qwen、DeepSeek）可能使用自定义标签。
4. **内部推理泄漏**：某些 Agent 框架让模型先输出 `<thinking>` 再输出最终 JSON，如果 prompt 没约束好，这些标签会留在最终输出里。

所以问题不是“让模型听话”，而是**假设模型会出错，在解析层做好防御**。

## 工程做法：一个可复用的容错解析器

下面是一个在实践中反复打磨的解析函数，思路是：

- 按优先级尝试多种提取模式
- 每种模式独立、可测试
- 不依赖单一正则或假设

```python
import json
import re

def robust_json_extract(raw: str):
    """从混合了 markdown、标签、自然语言的字符串中提取最外层的 JSON 对象。"""
    if not raw or not raw.strip():
        raise ValueError("Empty input")

    strategies = [
        # 1. 直接解析（最快路径）
        lambda s: json.loads(s.strip()),

        # 2. 去掉 markdown 代码块
        lambda s: json.loads(re.sub(r"```(?:json)?\s*|\s*```", "", s, flags=re.DOTALL).strip()),

        # 3. 提取 ```json ... ``` 代码块中的内容
        lambda s: json.loads(
            re.search(r"```(?:json)?\s*\n?(.*?)\n?```", s, re.DOTALL).group(1).strip()
        ),

        # 4. 提取 <tool_call>...</tool_call> 或类似标签中的内容
        lambda s: json.loads(
            re.search(r"<(?:json|tool_call|output)\s*>\s*\n?(.*?)\n?\s*</(?:json|tool_call|output)\s*>", s, re.DOTALL).group(1).strip()
        ),

        # 5. 提取第一个匹配的 {...}（贪婪匹配，跨行）
        lambda s: json.loads(
            re.search(r"\{.*\}", s, re.DOTALL).group(0)
        ),
    ]

    for idx, strategy in enumerate(strategies):
        try:
            return strategy(raw)
        except (json.JSONDecodeError, AttributeError, ValueError):
            continue

    raise ValueError("All strategies exhausted, unable to extract valid JSON")
```

### 使用示例

```python
out1 = "好的，这是结果：\n```json\n{\"tool\": \"search\"}\n```"
out2 = "<tool_call>\n{\"tool\": \"search\"}\n</tool_call>"
out3 = "{\"tool\": \"search\",} "  # 尾逗号问题由后续进一步清洗

parsed1 = robust_json_extract(out1)  # 成功
parsed2 = robust_json_extract(out2)  # 成功
```

### 尾逗号与注释的补充清洗

上述解析器不处理 JSON 内非法尾逗号或注释。可在策略链之前加入预清洗：

```python
import re

def clean_json_text(s: str) -> str:
    # 移除单行注释 //
    s = re.sub(r"//.*", "", s)
    # 移除多行注释 /* ... */
    s = re.sub(r"/\*.*?\*/", "", s, flags=re.DOTALL)
    # 移除尾逗号（紧邻 ] 或 } 前的逗号）
    s = re.sub(r",\s*([}\]])", r"\1", s)
    return s
```

将 `clean_json_text` 放在 `robust_json_extract` 的第一步，对原始字符串进行预清洗，再交给策略链。

## 踩坑点

1. **嵌套 JSON 或标签**  
   如果模型输出了多个 JSON 块（如 Agent 思考过程 + 最终参数），贪婪 `{.*}` 模式会捕获错误对象。解决方案是业务层面要求模型只输出一个 JSON，或者在提取时明确匹配模式（如 `"tool_call"` 字段的存在性），而不是盲目用第五策略。

2. **性能开销**  
   顺序尝试 5 种解析在大多数脚本中可忽略，但在高频 MCP 调用链路（每秒数十次）中需要缓存解析结果或提前优化。实测 Python 下 5 策略开销约 20-50 微秒，尚可接受。

3. **回退策略太“贪婪”**  
   策略 5 会提取字符串中任何 `{...}`，一旦输出里同时出现两个 JSON 对象，可能拿到错误的一个。建议在策略 5 前加入业务字段校验：例如解析后判断是否包含必需键 `tool`，否则继续下一策略。这需要定制化，不宜放在通用解析器内。

4. **修复后日志丢失原始信息**  
   防御性解析成功后，务必在日志中保留原始 `raw` 字符串（至少一行），否则当解析出“错得不是错误”的 JSON 时，无法复现问题。

## 可复用建议

- **封装为 OpenClaw 工具或 MCP 预处理中间件**  
  写成一个 `parse_json_field` 函数，供所有 action/plugin 调用，避免每个节点重复实现。
- **支持配置白名单标签**  
  不同场景可能使用 `<function_call>`、`<output>` 等标签，解析器应支持传入自定义标签名列表。
- **返回清晰错误信息**  
  当所有策略失败，抛出异常时附上 `raw` 前 200 字符，便于快速定位。
- **单元测试覆盖真实模型输出**  
  收集生产环境真实错误样本，做成 regress tests，确保解析器更新不引入回归。

## 总结

LLM 输出格式的不可靠是一个工程现实，试图通过更精细的 prompt 完全消除是徒劳的。将解析视为防御性编程的一环，用多层提取、预清洗和业务校验构建容错体系，能够显著降低 Agent 流程因格式问题中断的概率。这个思路同样适用于 XML、YAML 等其它结构化格式，核心在于：**不要信任输入，逐层降解，失败不丢上下文**。

---

