---
title: 别赌格式：LLM 输出 JSON 的分层防御性解析实践
feedId: 36905
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

在 OpenClaw 的插件和 MCP 工具链里，让模型输出结构化 JSON 几乎绕不开：工具调用参数、工作流节点间传数据、插件配置生成，全都依赖解析成功。Prompt 里写一句"只输出 JSON，不要任何其他内容"，演示时百试百灵，上了生产就开始出现各种"格式个性"。

## 问题

我把近两个月插件日志里的解析失败样本翻了一遍，原因基本收敛为三类：

1. **围栏包裹**：JSON 被 ` ```json ... ``` ` 包住，有时连 `json` 前缀都没有，只有裸围栏。
2. **标签混用**：同一会话里，第一轮用 `<json>` 包，第二轮换成 `<output>`，偶尔还有 `<result>`、`<answer>`，甚至自创标签。
3. **前后污染**：JSON 前面带一句"以下是解析结果"，后面跟一段总结；或者被 max_tokens 截成半截。

只按一种格式写的解析器，会在这三类面前轮番翻车。

## 做法：分层解析管道

思路很简单：不赌格式，按成本从低到高依次尝试，任何一层成功即返回。核心代码约 50 行：

```python
import json, re

TAGS = ("json", "output", "result", "answer")

def extract_balanced(text: str):
    """提取第一个括号配平的块，跳过字符串字面量内的括号"""
    for open_ch, close_ch in (("{", "}"), ("[", "]")):
        start = text.find(open_ch)
        if start == -1:
            continue
        depth, in_str, esc = 0, False, False
        for i in range(start, len(text)):
            c = text[i]
            if in_str:
                if esc: esc = False
                elif c == "\\": esc = True
                elif c == '"': in_str = False
            elif c == '"':
                in_str = True
            elif c == open_ch:
                depth += 1
            elif c == close_ch:
                depth -= 1
                if depth == 0:
                    return text[start:i + 1]
    return None

def parse_llm_json(text: str):
    # L0：直接解析，最快路径
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass
    # L1：剥代码围栏
    m = re.search(r"```(?:json)?\s*(.+?)\s*```", text, re.S)
    if m:
        try:
            return json.loads(m.group(1))
        except json.JSONDecodeError:
            pass
    # L2：自定义标签，列表可配置
    for tag in TAGS:
        m = re.search(rf"<\s*{tag}\s*>(.+?)<\s*/\s*{tag}\s*>", text, re.S | re.I)
        if m:
            try:
                return json.loads(m.group(1))
            except json.JSONDecodeError:
                pass
    # L3：括号配对扫描，兜底标签缺失和前后废话
    block = extract_balanced(text)
    if block:
        try:
            return json.loads(block)
        except json.JSONDecodeError:
            pass
    return None
```

L3 是关键：它不关心任何标签，只找第一个 `{` 或 `[`，深度计数直到配平。字符串内的花括号靠 `in_str` / `esc` 两个状态位跳过。拿到候选块后统一交给 pydantic 做字段校验，不合格进重试队列（附完整原始输出，上限 2 次）。

## 踩坑点

- **贪婪正则**：`r"```json(.*)```"` 会把多个代码块全吃进去，务必用非贪婪 `.*?` 加 `re.S`。
- **括号计数不看字符串**：我自己踩过最深的坑。配平算法必须带字符串状态机，否则遇到 `"desc": "输出 {xxx}"` 这种值必然截断错位。
- **标签大小写**：`<JSON>` 和 `<json>` 都出现过，正则记得加 `re.I`。
- **截断的 JSON**：L3 只能提取配平块，纯截断文本提不出完整结构，别指望在解析层修复，交给重试更干净。
- **LLM 二次修复慎用**：让模型修 JSON 时温度给 0，且必须附完整原始输出，否则它会在截断处"编"出一个看似合理的字段。

## 可复用建议

1. 解析器做成独立中间件，所有插件共用一份，别各自为政。
2. 返回值附元数据：命中了哪一层、原始文本长度，方便排查和统计。
3. 监控各层命中率。L0 命中率突然下降，大概率是上游模型或版本变了，这是很好的早期信号。
4. 日志落原始输出前过一遍脱敏，别把用户上下文原样存盘。
5. Prompt 里的格式约束照写，但把它当"降低 L1/L2 命中率的手段"，而不是"可以省掉解析防御的依据"。

## 总结

LLM 的输出本质上是不可信的外部输入源。防御性解析的价值不在于让解析器"更聪明"，而在于把不确定性收敛成一条明确的管道：逐层尝试、统一校验、失败可重试、全程可观测。这套管道上线后，我们插件的解析失败率降了一个数量级，剩下的失败基本是真实截断，走重试即可解决。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/0968dba0bdf11c8c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/94ebbf3bf1945cbd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/5b8e0311a52a126d.png)

