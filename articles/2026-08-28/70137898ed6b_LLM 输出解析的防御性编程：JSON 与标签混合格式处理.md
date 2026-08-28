---
title: LLM 输出解析的防御性编程：JSON 与标签混合格式处理
feedId: 35077
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw、Agent、MCP 插件以及自动化流程里，LLM 经常被当作“结构化数据源”，下游直接消费 JSON。虽然 prompt 中反复强调“只输出 JSON”，但实际返回常常是混合格式：

- ```json 代码块前后带解释
- 自定义标签包裹：`<result>{...}</result>`
- JSON 与标签混在同一个输出里
- 一个响应里出现多个 JSON 块，其中一个是示例，另一个才是真实结果

如果只做一次 `JSON.parse`/`json.loads`，线上会频繁断流。本文给出一套防御性解析顺序，用来处理 JSON 与标签格式混合。

## 问题

核心矛盾：模型输出是自然语言生成结果，格式约束只是“概率性遵守”。纯正则匹配 JSON 容易截断，纯标签匹配又会被代码块干扰。工程上需要把 LLM 输出当作不可信输入处理，而不是把它当协议。

## 做法/步骤

推荐一条小解析流水线，逐步降级：

1. **提取 fenced code block**  
   优先匹配 ` ```json ... ``` `，大小写和空白放宽。若有多个代码块，不直接选第一个，先收集为候选。

2. **尝试完整解析**  
   对 strip 后的原始文本，如果以 `{` 或 `[` 开头，直接 `json.loads`。这能覆盖表面干净的输出。

3. **提取标签内容**  
   匹配 `<tag>...</tag>` 形式的自定义标签，如 `output`、`result`、`json`。注意不要用过于贪婪或非贪婪的嵌套正则，先定位闭合标签，再交给下一步。

4. **平衡括号扫描**  
   从文本中第一个 `{` 或 `[` 开始，维护括号栈，并跟踪字符串内和转义状态，找到完整 JSON 段。这一步能救回“文本前导 + JSON + 文本尾随”。

5. **候选验证与排序**  
   把所有候选交给 JSON 解析器和 schema 校验。按“schema 错误最少、关键字段覆盖率最高”排序，而不是按出现顺序。

6. **失败修复与重试**  
   只修复明确问题：尾部逗号、单引号、代码块残留。修不动或候选都不是 JSON 时，带着错误信息重试一次，不无限重试。

一个简单骨架如下：

```python
def extract_json(text):
    candidates = []
    candidates += extract_fenced(text)
    if is_json(text.strip()):
        candidates.append(text.strip())
    candidates += extract_tagged(text)
    candidates += extract_balanced(text)
    return best_candidate(candidates, schema=MySchema)
```

## 踩坑点

- **非贪婪标签正则早停**：`<output>{"a":"<b>"}</output>` 这种内容里含标签形状的文本，会被切错。先找闭合标签位置，再做平衡扫描。
- **字符串里的括号**：字符串内 `}` 或 `]` 会让简单括号计数误判。扫描器必须处理双引号、转义和单引号。
- **多个候选选错**：示例 JSON 可能排在真实结果前。不要按出现顺序选第一个，必须结合字段完整度和 schema 错误率。
- **不是 JSON 却硬修**：输出可能是 markdown 表格或 YAML，强行修复会把污染数据当成功。候选人不可解析时应快速失败并重试。
- **代码块变体**：```JSON、``` json、代码块内还有注释，都可能导致 fenced 提取失败。匹配放宽，后续仍用平衡扫描兜底。
- **解析失败无限重试**：只重试一次，超过就返回可读错误，保留原始输出。

## 可复用建议

- 将提取器做成独立模块，返回 `parsed`、`source`、`candidate_path`、`score`，不要静默 fallback。
- prompt 仍然要求“只输出 JSON，不要代码块或标签”，但系统设计不依赖 prompt 承诺。
- 关键流程用 pydantic 或 JSON schema 做最终校验，解析成功只是第一步。
- 日志里保留原始 LLM 输出、候选数量、选中路径，便于发现模型输出漂移。
- 对超长输出先截断到安全长度，防止正则复杂度失控。
- MCP 插件或 Agent 工具对解析失败返回结构化错误，不把 Python 堆栈直接抛给调用方。

## 总结

防御性解析的关键不是写出万能正则，而是把 LLM 输出当作不可信输入。用“代码块提取 → 完整解析 → 标签提取 → 平衡扫描 → 候选验证 → 失败重试”这一串降级流程，能明显降低 JSON 与标签混合格式带来的断流。最终保留原始证据和解析路径，比单纯解析成功更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/334508791c2e9e87.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/84e24d3a934db4a2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/193a887551788ca1.png)

