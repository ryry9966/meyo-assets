---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 36389
source: 综合讨论
publishedAt: 2026-09-07
---

# 背景

在 OpenClaw 的插件和自动化流程里，LLM 的结构化输出几乎无处不在：工具调用参数、任务路由决策、MCP 工具的参数填充。我们通常会在提示词里写明“只输出 JSON，不要任何解释”，但实际跑起来会发现，模型并不总是听话。

# 问题

一段时间收集下来的“脏输出”大概这几类：

- JSON 被 ```json 代码围栏包着；
- 前面一句“好的，以下是处理结果”，后面再跟一段解释；
- 被 `<result>...</result>` 这类 XML 风格标签包裹；
- 尾逗号、单引号键、中文全角引号；
- 输出超过 max_tokens 被截断，JSON 只剩一半。

任何一种出现，`json.loads` 直接抛异常，整条自动化链路就断了。更麻烦的是日志里往往只剩一行 JSONDecodeError，没法回溯。

# 做法：分层防御

我们的思路是不信任任何单一环节，按层兜底：

1. **提示词层**：明确格式要求，给一个 few-shot 示例。有用，但不能当唯一防线。
2. **预处理层**：剥掉代码围栏和已知标签；然后做括号平衡扫描——从第一个 `{` 开始，跟踪字符串内的转义状态，找到与之配对的 `}`，截取子串。这比“找第一个 { 和最后一个 }”可靠得多。
3. **解析层**：先标准解析；失败后做受限修复——去尾逗号、替换全角引号，重试一次。
4. **校验层**：用 pydantic 或 jsonschema 做结构校验，字段缺失或类型不对同样算失败。
5. **兜底层**：带着解析错误信息重试一次（让模型自我修复），仍失败则降级到默认行为或告警，原始输出必须落盘。

核心截取逻辑大致是：

```python
def extract_json(text: str) -> str:
    start = text.find('{')
    if start < 0:
        raise ValueError('no json object')
    depth, in_str, esc = 0, False, False
    for i, ch in enumerate(text[start:], start):
        if in_str:
            if esc: esc = False
            elif ch == '\\': esc = True
            elif ch == '"': in_str = False
        else:
            if ch == '"': in_str = True
            elif ch == '{': depth += 1
            elif ch == '}':
                depth -= 1
                if depth == 0:
                    return text[start:i + 1]
    raise ValueError('unbalanced json')
```

# 踩坑点

- **别用“第一个 { 到最后一个 }”的正则截取**。字符串值里出现 `{` 或嵌套对象时极易截错位，必须做括号平衡扫描并处理转义。
- **剥围栏的贪婪匹配会误伤** JSON 里本身包含 ``` 的内容（比如让模型生成含示例代码的字段），优先按行匹配行首围栏。
- **全局替换智能引号有风险**：字符串值里本来就该有中文引号，提前替换会破坏语义。修复只应在标准解析失败后局部进行。
- **别因为模型支持 function calling 就省掉防御**。多模型路由、本地小模型、长上下文场景都会漏。
- **截断也是一种失败**：进修复流程前先看 finish_reason 和输出长度，被截断的 JSON 修复不出来。
- **日志要存原始输出**，附带模型名和参数，否则排查全靠猜。

# 可复用建议

- 把解析做成独立模块，提供 strict / lenient 两档接口，所有插件共用，不要各写一套。
- 失败样本落盘积累，定期回看。实践中很多脏输出改一句提示词就能消掉大半，但前提是你有数据。
- 这套思路同样适用于 MCP 工具参数校验和插件间的消息解析，不只是对话出口。

# 总结

LLM 输出解析的本质，是在和一台不确定的生成器做接口。防御性解析的目标不是消灭脏输出，而是保证每一种脏形态都有明确的处理路径、以及可观测的失败记录。解析层做厚一层，自动化链路的稳定性就多一分。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/5d1890eecbb35ea8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/75704c8db423e51e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/dba7580d4e7f8fa1.png)

