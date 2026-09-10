---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 36969
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

在 OpenClaw 插件和 MCP 工具链里，LLM 的输出往往不是终点，而是下一段流程的输入：解析成 JSON 后路由工具调用、更新状态、触发自动化。一旦解析挂掉，整条链路就断在中间。实践中最大的问题不是模型“不会输出 JSON”，而是**同一个 prompt 下，它的输出格式会在多次调用之间漂移**。

## 问题：一份 prompt，五种输出

真实日志里常见的变体：

- 包裹在 ` ```json ` 代码围栏里
- 纯 JSON，但前后粘着一句“好的，以下是结果”
- 推理模型下，前置一整段思考文本
- 尾逗号、单引号、中文智能引号
- 长输出被 `max_tokens` 截断，只剩半截
- JSON 值里嵌套 markdown 围栏，简单正则直接匹配错块

任何只针对单一格式的解析器，上线后迟早会炸。

## 做法：分层解析管线

核心思路：不要指望一条规则覆盖所有情况，而是像漏斗一样逐层降级，每层只处理自己认识的格式。

```python
import json, re

def parse_llm_json(raw: str):
    # 第 0 层：剥离思考标签
    text = re.sub(r"<think>.*?</think>", "", raw, flags=re.S).strip()

    # 第 1 层：代码围栏，非贪婪，逐块尝试
    for block in re.findall(r"```(?:json)?\s*(.*?)```", text, flags=re.S):
        try:
            return json.loads(block)
        except json.JSONDecodeError:
            continue

    # 第 2 层：无围栏时定位首个 { 或 [，按括号配对截取
    snippet = match_brackets(text)  # 需跳过字符串字面量内的括号
    if snippet:
        try:
            return json.loads(snippet)
        except json.JSONDecodeError:
            pass

    # 第 3 层：宽松修复——尾逗号、智能引号
    cleaned = re.sub(r",\s*([}\]])", r"\1", text)
    cleaned = cleaned.replace("\u201c", '"').replace("\u201d", '"')
    try:
        return json.loads(cleaned)
    except json.JSONDecodeError:
        save_raw_for_review(raw)   # 失败也要留痕
        raise
```

三个要点：每层失败落到下一层而非直接抛错；括号配对必须跳过字符串内部的 `{`；最终失败时把原始输出完整落盘，供离线分析。

## 踩坑点

1. **贪婪正则**：提取围栏时把 `.*?` 手滑写成 `.*`，两个代码块被合并成一个。
2. **括号计数不跳过字符串**：JSON 值里的 `{` 会提前“闭合”，截取残缺。
3. **`json.loads` 默认接受 `NaN`/`Infinity`**，严格场景要用 `parse_constant` 拦截。
4. **截断的半截 JSON 别静默丢弃**——多数情况下带着“只输出 JSON，不要解释”重试一次就能救回。
5. **解析成功 ≠ 数据正确**：解析后必须过 schema 校验（pydantic/jsonschema），字段缺失照样要走重试分支。

## 可复用建议

- **双侧防御**：prompt 侧给 few-shot 固定格式，解析侧做兜底，缺一不可。
- 把线上真实失败样本沉淀成测试语料，每次改动解析器跑一遍回归。
- 给解析器加版本号，失败日志带上版本，方便评估改动收益。
- 记录“命中了第几层”：某层命中率突然上升，通常意味着上游 prompt 或模型行为变了。

## 总结

LLM 输出解析的本质不是写出“完美的正则”，而是承认格式不可控，用**分层降级 + 原始留痕**把不确定性圈在可控范围内。解析器是管线里最不起眼、但故障率最高的一环，值得像核心业务代码一样给它配测试。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/a4a6e01cf675a4e6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/892e7b22903d56bf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/529d80d8c5e75482.png)

