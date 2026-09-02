---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理实战
feedId: 35781
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

在 OpenClaw 的插件与 MCP 工具链里，大量环节依赖“让模型返回 JSON”：工具参数抽取、意图路由、结构化摘要。但实践中模型很少给你干净的 JSON——最常见的是 markdown 代码围栏包着 JSON，前后还各带一句解释；偶尔混入注释、尾逗号、全角引号。直接 `json.loads()` 的解析器在这些场景下就是定时炸弹。

## 问题

我们的一条自动化流水线曾因为解析层只支持裸 JSON，在模型开始加 ```json 围栏后，失败率从 1% 跳到 18%。更麻烦的是失败是概率性的：同一 prompt，九成调用正常，一成带围栏或前导文本，导致问题拖了两周才被定位。

## 做法：分层降级解析

核心思路不是“写一个更强的正则”，而是按成本从低到高分层尝试，并记录每层命中情况：

```python
def parse_llm_json(raw: str):
    # L0: 直接解析
    try:
        return json.loads(raw)
    except JSONDecodeError:
        pass

    # L1: 提取 ```json / ``` 围栏内容
    for m in re.findall(r"```(?:json|jsonc)?\s*\n(.*?)```", raw, re.S):
        try:
            return json.loads(m)
        except JSONDecodeError:
            continue

    # L2: 括号配平扫描，取第一个平衡的 {...} 或 [...]
    #     扫描时需感知字符串字面量内的括号
    candidate = extract_balanced(raw)
    if candidate:
        return json.loads(fix_common_issues(candidate))
```

L2 有两个关键点：

1. 括号配平必须感知字符串状态——遇到 `"` 切换 in_string 标志，正确处理 `\"` 转义，否则 `{"text": "a}b"}` 会被截断。
2. `fix_common_issues` 只做保守修复：删尾逗号、替换全角冒号/逗号/引号、去掉行注释（同样跳过字符串内部）。修复前备份原文，修复失败时把原文完整打进日志。

L2 仍失败时，再上 json-repair 这类修复库；最后兜底是把解析错误信息拼回 prompt 做一次结构化重试。超过一次就放弃并告警，不要无限重试。

## 踩坑点

- 贪婪正则 `\{.*\}` 会把响应里多个 JSON 对象吞成一个，必须用配平扫描。
- 模型偶尔返回 JSONL（多行对象），“响应里只有一个对象”的假设会挂；先按行切分再逐个尝试。
- 修复函数删注释时，`https://` 里的 `//` 被误删过——注释清理同样要感知字符串状态，或只处理围栏外的内容。
- `temperature=0` 不是保证，格式漂移仍会发生。结构化输出 / 函数调用参数是更好的第一选择，但对接部分小模型时不可用，所以解析兜底层必须常驻。

## 可复用建议

- 解析器做成公共工具，返回值带上“命中层级”元数据，便于统计哪一层在兜底、兜底率是否在涨。
- 把线上失败样本脱敏后收进测试夹具，每次改动解析器跑一遍回归。
- prompt 侧同时收紧：“只输出 JSON，不要围栏，不要解释”。提示词和解析层双管齐下，能消掉大半脏输出，但不要指望任何一边单独解决问题。

## 总结

LLM 输出解析的健壮性不靠单一技巧，而是“分层降级 + 保守修复 + 可观测 + 有界重试”的组合。把解析层当成独立的容错组件来维护，而不是散落在各业务代码里的裸 `json.loads`——这是这次实践中最值得带走的一条。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/5c2c2d024ba291ea.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/4e7a6dd6e3533038.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/f1b4664d869c0b56.png)

