---
title: LLM 输出解析的防御性编程：一套处理混合 JSON 标签格式的实用方案
feedId: 36068
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

在 OpenClaw 插件和 MCP 工具链里，"让模型输出 JSON" 几乎是绕不开的一步：解析用户指令、填充工具参数、生成结构化执行计划。理想情况是一次 `json.loads` 完事，但实践中模型输出格式会漂移，解析代码写得越乐观，线上故障来得越快。

## 问题

同一个 prompt，在不同模型、不同温度、不同上下文长度下，输出可能是：

- ` ```json ... ``` ` 代码块包裹
- 裸 JSON
- `<json>...</json>` 或自定义标签包裹
- JSON 前后带解释性文字
- 格式瑕疵：尾逗号、单引号、`//` 注释、未转义换行
- 最阴险的：**字符串值里本身包含 ``` 或嵌套 JSON**

大多数解析代码只处理了第一种，剩下全是长尾故障。

## 做法：分层解析，先剥壳再修复

核心思路：不要指望一步解析成功，拆成三层流水线。

**第一层：剥壳**，按优先级提取候选片段——代码围栏块、自定义标签、括号配对扫描出的第一个平衡 `{...}`，最后全文兜底。

**第二层：宽松解析**，标准库失败后先试 json5 类容错解析；仍失败再做最小修复（去尾逗号、单引号转双引号），修复动作必须打日志，不能静默。

**第三层：schema 校验 + 受控重试**，结构对了但字段缺失时，把具体校验错误回喂模型，重试上限 2 次，超限走降级路径（默认值/人工队列），不要无限烧 token。

骨架代码：

```python
def extract_json(text: str):
    candidates = []
    for m in re.finditer(r"```(?:json)?\s*(.*?)```", text, re.S):
        candidates.append(m.group(1))
    for m in re.finditer(r"<json>(.*?)</json>", text, re.S):
        candidates.append(m.group(1))
    candidates.append(text)
    for c in candidates:
        try:
            return json.loads(c)
        except json.JSONDecodeError:
            continue
    return repair_and_load(text)  # 最小修复 + 日志
```

## 踩坑点

1. **贪婪正则**：`r"```(.*)```"` 会把两个独立代码块之间的普通文本全吞进去，必须非贪婪。
2. **嵌套围栏**：字符串值里含 ``` 时，先按围栏切；剥出内容校验失败再退回括号配对，别把提取顺序写死。
3. **修复别太激进**：`'id': 123` 修成 `"id": 123` 没问题，但对整段做引号替换可能破坏值内的引号语义。修复器必须有单元测试。
4. **拼接输出**：模型偶尔输出两个 JSON 对象连写，`json.loads` 报 "Extra data"。取第一个平衡对象，比整体重试便宜得多。
5. **温度问题**：高温下格式漂移明显。工具调用类任务建议 temperature 0~0.3，这比事后修复更省。

## 可复用建议

- 解析器做成独立模块；线上每出现一个失败样本，就存进测试集做回归，样本库就是最真实的测试用例来源。
- Prompt 端约束（few-shot + 明确"只输出 JSON"）和解析端兜底都要做：前者降低失败率，后者兜住长尾，缺一不可。
- 有结构化输出 / JSON mode 就优先用，但不要 100% 信任——各模型支持程度不一，MCP 场景下尤其要保留兜底层。
- 降级路径要提前设计好：默认值、标记待处理、告警通知，绝不让脏数据静默流入下游。

## 总结

LLM 输出解析的稳健性，不靠 prompt 一次写对，而靠"剥壳 → 修复 → 校验 → 受控重试 → 降级"这条流水线。把每个失败样本变成测试用例，解析层才会越跑越稳。这套模式不挑模型，换底座也能直接复用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/21366a697e54cbad.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/25caa53dec89e571.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/d0f3530080513cbc.png)

