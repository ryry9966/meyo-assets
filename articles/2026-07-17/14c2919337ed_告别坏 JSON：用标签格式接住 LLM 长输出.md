---
title: 告别坏 JSON：用标签格式接住 LLM 长输出
feedId: 29408
source: 综合讨论
publishedAt: 2026-07-17
---

## 背景：你有多少时间在修 JSON？

在 OpenClaw 的自动化流水线里，我们需要大模型产出结构化内容——多步任务计划、工具调用参数、摘要、分类标签等。社区习以为常的做法是要求模型输出 JSON，然后 `json.loads()` 拿结果。

但模型从不是可靠的 JSON 打印机，尤其是在长文输出、流式接收时：

- 嵌套数组少一个 `]`
- 字符串里多出一个未转义的 `"`
- 末尾标点被截断，JSON 整体不合法
- 模型在 JSON 外附加解释性文字

我们在一个 MCP 工具串联的 Agent 里统计过，自有提示词下，长 JSON 输出的一次解析成功率只有约 70%。这意味着 30% 的运行会因为一行代码抛异常而卡住，后续重试又消耗 token 和时间。自动修复库（比如 `json_repair`）能缓解，但碰到严重截断或上下文错误仍然无解。

## 问题：JSON 的全局一致性在流式场景天然脆弱

JSON 要求整段文本必须遵守一对全局的 `{}` 或 `[]`，流式输出时，我们无法判断一个不完整的片段是否最终能合法闭合。一旦网络中断或模型提前停止生成，我们已经收到的部分几乎只能丢弃。

标签格式（类 XML，但不必完全遵守 XML 规范）提供了另一种思路：**结构由成对的开始/结束标记决定，天然允许局部解析。** 哪怕只拿到部分输出，也可以从容提取已经闭合的标签内容。例如：

```
<step>分析用户需求</step>
<step>查询天气 API</step>
```

即使后面 `</response>` 尚未生成，我们已经可以提取出两个完整步骤，而不会被一个缺失的 `]` 拖垮所有逻辑。

## 做法：设计、提示、解析三步走

### 1. 定义标签协议
先约定一个简单的根标签和内部字段标签。标签名最好有命名空间前缀以避免与正文冲突。

```
<o:response>
  <o:intent>查询天气</o:intent>
  <o:actions>
    <o:action>调用 get_weather</o:action>
    <o:action>格式化回复</o:action>
  </o:actions>
</o:response>
```

对于可能包含特殊字符的文本字段，可以要求模型将内容中的 `<` 替换为 `&lt;`（不过多数小模型做不好，更可行的策略是选用罕见的前缀，如 `ocl:`, `x:`, `action:` 加上冒号）。

### 2. 修改提示词，并给出冷硬约束
在系统提示中加入格式指令，要求模型 **必须且只能** 输出一个有效的标签段，不附加任何解释。提供一个 few-shot 示例，明确展示开始/结束标签的位置。

```markdown
你的输出必须严格包裹在 `<o:response></o:response>` 标签内。
使用子标签表达字段，不要输出任何额外文字。
示例：
<o:response>
<o:summary>用户想查询本地天气</o:summary>
</o:response>
```

如果使用的是 OpenAI/Claude 等支持结构化输出（structured output）的模型，可以保留 JSON 模式；但在工具调用链中，自定义标签对非 OpenAI 端点、开源模型更友好，不受 function calling 约束。

### 3. 实现带缓冲的解析器
不依赖完整 XML 解析（因为输出可能不合法），而是使用正则非贪婪匹配提取闭合标签。

简易 Python 方案：

```python
import re
from typing import Dict, Optional

def extract_tag(content: str, tag: str) -> Optional[str]:
    pattern = rf"<{tag}>(.*?)</{tag}>"
    m = re.search(pattern, content, re.DOTALL)
    return m.group(1) if m else None

def parse_response(text: str) -> Dict:
    root = extract_tag(text, "o:response")
    if not root:
        raise ValueError("未找到根标签")
    return {
        "summary": extract_tag(root, "o:summary"),
        "actions": extract_tag(root, "o:actions"),
    }
```

对于流式场景，在缓冲区中不断追加 `token`，每次检查是否出现闭合的 `</o:response>`。一旦闭合，截取完整段交给 `parse_response()`。若超时仍未闭合，可提取已有的子标签作为部分结果，并记录警告。

## 踩坑点

- **标签名侵入正文**  
  如果模型生成的回答或摘要里恰好出现了 `<o:summary>`，正则会错误捕获。规避方法：使用足够生僻的前缀，并在提示中说明“如果需要输出标签符号本身，请使用 `(左尖括号)` 等替代”。

- **模型在标签外写“小作文”**  
  许多模型的“安全意识”让它们忍不住在输出前加上“好的，我会按格式输出：”。这会破坏解析。通过提示语严厉要求“除了标签内容，不要输出任何文字”，并在解析端用正则查找第一个 `<o:response>` 开始的位置。

- **嵌套标签与换行**  
  模型倾向在标签间插入换行、多余空格。`re.DOTALL` 可以跨行匹配，但若字段内容本身也是多行且包含换行，正则没问题。如果使用了命名空间前缀里的冒号，务必在正则中转义，或直接使用字符串匹配。

- **流式缓冲区膨胀**  
  一次对话可能吐出上千 token，如果永久等待闭合标签，内存会持续增长。设置最大缓冲区长度（如 4096 字符），或设定超时时直接提取已有闭合标签。

## 可复用建议

在 OpenClaw 的插件管道里，封装一个 `TagParser` 工具类：

```python
class TagParser:
    def __init__(self, root_tag, required_tags, optional_tags):
        ...
    def feed(self, chunk: str) -> Dict | None:
        # 返回已闭合的 root 标签对应字典，未闭合返回 None
    def partial_result(self) -> Dict:
        # 尽力提取已闭合的子标签
```

与 Prompt 模板结合，定义一次“格式契约”，后续所有 Agent 节点都复用这个解析器。结合 Pydantic 做字段类型校验，形成“文本→字典→数据模型”的可靠转换。

当模型厂商升级、模型行为漂移时，只需优化提示词中的格式说明和标签名，解析器代码基本不用改。

## 总结

JSON 是机器间交换数据的优秀格式，但在 LLM 的流式长文输出中，它的“全局一致性”成了软肋。引入自定义标签格式，本质上是把一个大括号的压力，拆解成多个可独立匹配的标记任务。对于 OpenClaw 这类需要把 LLM 输出塞进工具链的实践者来说，牺牲一点点 JSON 的“正统”，换来解析的稳定和流式下的容错，是一项收益明显的工程决策。

下次再看到 `JSONDecodeError` 刷屏时，不妨试试让模型吐出 `<step>...</step>`。

---

