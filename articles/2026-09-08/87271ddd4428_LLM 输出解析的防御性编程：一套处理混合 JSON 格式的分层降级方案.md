---
title: LLM 输出解析的防御性编程：一套处理混合 JSON 格式的分层降级方案
feedId: 36548
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

在 OpenClaw 里写插件、接 MCP 工具，绕不开一件事：把 LLM 的文本输出转成结构化数据——工具调用参数、子任务分发、结果回填，全靠它。理论上 structured output / function calling 能约束格式，但现实中总有兜不住的场景：模型不支持约束解码、请求走第三方网关、或者你自己拼 prompt 要求"只输出 JSON"。这时候，解析层就是你系统的第一道防线，得按处理不可信输入的标准来写。

## 问题：你以为的 JSON，和模型实际给的 JSON

一个足够典型的输出长这样：

````
<think>用户想要搜索……我应该返回 action 为 search</think>
好的，以下是执行计划：
```json
{
  "action": "search",
  "query": "OpenClaw 部署"，
}
```
希望对你有帮助！
````

问题全齐了：思维链标签混杂、说明文字包裹、围栏语言不统一（`json` / `JSON` / `jsonc` / 无标注）、尾逗号、全角逗号引号混入、字符串值里含花括号、流式场景下还可能截断。任何一环直接 `json.loads()` 都会炸，而且炸得毫无上下文。

## 做法：分层降级，便宜的先试

核心思路不是写一个"聪明的正则"，而是搭一条流水线：每层只解决一类问题，失败就进下一层。

**1. 预清洗。** 用非贪婪正则删掉 `<think>...</think>` 块，去掉 BOM 和零宽字符。注意必须配对删除，别只删开标签。

**2. 围栏提取。** 优先抓 ```` ```json ````，其次任意围栏。用 `findall` 收集**全部**候选块，而不是取第一个——模型偶尔会先输出一个示例再输出真实结果，取第一个就取错了。

**3. 括号配对兜底。** 没有围栏时，扫描原文找第一个 `{`，做括号配对，配对时跳过字符串内部（处理转义），取每个平衡片段作为候选。

**4. 宽松解析。** 标准解析失败后，走 json-repair / json5 这类宽松解析器，处理尾逗号等顽疾。

**5. Schema 校验。** 解析成功 ≠ 数据可用。用 pydantic / jsonschema 验证字段类型和枚举；失败就把原输出和报错回喂给模型重试一次，要求只输出修正后的 JSON。

**6. 兜底。** 全部失败就记录原始输出、返回结构化错误，让上层决定重试或人工介入。**绝不静默返回 None。**

```python
import json, re

THINK = re.compile(r"<think>.*?</think>", re.S)
FENCE = re.compile(r"```(?:json|jsonc|JSON)?\s*\n(.*?)```", re.S)

def extract_candidates(text: str) -> list[str]:
    text = THINK.sub("", text).strip().lstrip("\ufeff")
    out = [m.strip() for m in FENCE.findall(text)]
    depth = start = 0
    in_str = esc = False
    for i, ch in enumerate(text):
        if in_str:
            esc = (ch == "\\" and not esc)
            if ch == '"' and not esc:
                in_str = False
            continue
        if ch == '"':
            in_str = True
        elif ch == "{":
            if depth == 0:
                start = i
            depth += 1
        elif ch == "}":
            depth -= 1
            if depth == 0:
                out.append(text[start:i + 1])
    return out

def parse_llm_json(text: str, repair=None):
    for cand in extract_candidates(text):
        try:
            return json.loads(cand)
        except json.JSONDecodeError:
            pass
        if repair:  # json-repair / json5 等宽松兜底
            try:
                return repair(cand)
            except Exception:
                pass
    raise ValueError("no parsable json in output")
```

## 踩坑点

- **贪婪正则跨围栏**：`.*` 会把多个围栏拼成一坨。用非贪婪加 `findall`，全量候选逐个试。
- **括号配对不跳字符串**：遇到 `{"tpl": "}"}` 会在字符串里的花括号处提前截断。配对循环里必须维护 `in_str` 状态。
- **字符替换要克制**：盲目"单引号转双引号"会改坏含撇号的合法值（如 `don't`）。先只处理尾逗号、全角逗号这类结构性问题，正文字符尽量不动。
- **流式别硬修**：半截 JSON 用增量方式判断"是否还可能闭合"，未闭合就继续等或发起续写，不要强行补括号——补出来的往往是语义错误的数据，比解析失败更危险。
- **重试要有上限**：带报错回喂的重试最多 1~2 次。模型经常在同一个坑里反复跌，无限重试只烧 token。
- **别各写一份**：多个插件各维护一套提取逻辑是灾难。收敛成一个共享解析模块，日志同时记原始输出和清洗后输出，排障效率完全是两个量级。

## 可复用建议

1. **能用 structured output 就用**，解析层是给约束不可用时的保险，不是日常主路径。
2. **解析函数保持纯函数**：输入字符串，输出 `(data | error)`，不碰全局状态。从自己的线上日志里捞 20 个真实坏样本，做成回归用例集。
3. **校验靠 schema**，不要散落 `isinstance`；回喂给模型的错误信息只给第一条，附上期望结构的最小示例。
4. **监控各层命中率**：围栏命中、括号兜底、宽松修复各占多少。宽松修复占比持续升高，说明该改 prompt 或换支持约束解码的模型了，而不是继续加固解析器。

## 总结

LLM 输出解析本质上是处理一个概率性的不可靠数据源。防御性编程的价值不在于某个正则写得多妙，而在于分层降级、明确的失败边界、以及可观测性。预清洗、围栏提取、括号配对、宽松修复、schema 校验、带反馈重试——这套组合拳能接住绝大多数线上脏输出；接不住的那些，也要让它们死得清楚、查得方便。这比"偶尔解析成功"重要得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/7018bd8b6dc701bc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/a4043bbadc777b89.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/55cf466a27688c37.png)

