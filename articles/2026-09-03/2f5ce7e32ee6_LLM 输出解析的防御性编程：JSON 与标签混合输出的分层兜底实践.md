---
title: LLM 输出解析的防御性编程：JSON 与标签混合输出的分层兜底实践
feedId: 35907
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

在 OpenClaw 的插件和 Agent 流程里，LLM 输出经常被当作结构化数据用：工具调用参数抽取、MCP 资源元信息、自动化流水线的中间结果。提示词里写明“只输出 JSON"，模型大多数时候也确实遵守——但“大多数时候”在自动化场景里远远不够，一次解析失败就可能让整条流水线挂掉。

## 问题

实际跑一段时间后，模型输出至少有这么几种形态混在一起：

1. 纯 JSON（理想情况，占比往往没有想象中高）
2. 包在 markdown 代码块里：` ```json ... ``` `
3. 前后带说明文字："好的，以下是解析结果：{...}"
4. 代码块里再嵌代码块（比如让模型生成包含脚本内容的任务描述）
5. 思考型模型把推理过程和 JSON 混在一段里输出

只写一行 `json.loads(output)`，前四种全部会炸。而异常处理如果只是“失败就原样重试”，三次重试可能拿到三种新的失败形态。

## 做法：分层解析，逐级兜底

核心思路：不指望一次解析成功，把“清洗”和“解析”拆成多级，每级处理一类污染，失败就降级到下一级。

```python
import json, re

def extract_json(text: str):
    # L1: 直接解析
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass

    # L2: 剥掉代码块围栏，逐块尝试
    for block in re.findall(r"```(?:json)?\s*(.+?)```", text, re.DOTALL):
        try:
            return json.loads(block.strip())
        except json.JSONDecodeError:
            continue

    # L3: 定位第一个 { 或 [，括号配对截取
    for i, ch in enumerate(text):
        if ch in "{[":
            return json.loads(balanced_slice(text, i))

    raise ValueError("no parsable JSON found")
```

`balanced_slice` 从起始位置数括号，关键是要跟踪“当前是否在字符串引号内、是否被转义”，跳过字符串内部的括号，找到真正配对的位置再截取。解析成功后还要加一层 schema 校验（pydantic 或手写字段检查），把“解析成功但字段缺失”拦在业务逻辑之前。

## 踩坑点

1. **贪婪正则**：`re.search(r"\{.*\}", text, re.DOTALL)` 会从第一个 `{` 匹配到最后一个 `}`，如果模型尾部顺带写了个示例对象，截出来的就不是你要的。
2. **字符串内嵌围栏**：JSON 字符串值里含 ` ``` ` 时，简单剥围栏会把 JSON 从中间切断。所以围栏剥离之后必须保留括号配对这条退路。
3. **截断输出**：`max_tokens` 给小了，JSON 被截断，括号配对也救不回来。解析前先检查结尾特征，这种情况直接重试并调大预算，不要在解析层死磕。
4. **不可见字符**：BOM、零宽空格、全角引号都会让 `json.loads` 失败，清洗时统一 strip 并把全角标点映射回半角。
5. **只重试不换策略**：解析失败后带上“上次输出格式错误，请只输出 JSON”的错误反馈重试，比原样重发有效得多。

## 可复用建议

- 优先用结构化输出能力（function calling / JSON mode / response_format），从源头消除大部分格式噪声；解析兜底仍然要有，因为不同模型、不同版本行为不一致。
- 原始输出原文落盘。排查格式问题时，真实脏样本比任何推断都有用。
- 解析器做成独立模块加单元测试，把线上遇到的真实脏样本沉淀为测试用例，越跑越稳。
- 解析失败、校验失败、重试耗尽分开打点，才能判断瓶颈在模型还是在解析层。

## 总结

LLM 输出解析的防御性编程，本质是不把“模型偶尔不守规矩”当异常，而是当常态输入的一种。分层兜底 + schema 校验 + 带反馈重试，三层下来，格式问题从“随机炸流水线”变成“日志里一行降级记录”。这套模式不复杂，但值得在写第一个 Agent 插件时就搭好。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/f94d7098684578b4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/7374f577687ad6a8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/9b19d8bad1f4ed67.png)

