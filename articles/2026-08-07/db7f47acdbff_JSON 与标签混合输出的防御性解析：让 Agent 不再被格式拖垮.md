---
title: JSON 与标签混合输出的防御性解析：让 Agent 不再被格式拖垮
feedId: 32015
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：JSON 解析里的“小问题”正在吃掉你的自动化时间

在 OpenClaw 的 Agent 流程、MCP 工具调用，甚至日常的 plugin 开发里，我们大量依赖 LLM 输出结构化数据——最常见的就是 JSON。理想中，prompt 里写好“只返回 JSON”，LLM 乖乖吐出 `{"action":"search","query":"..."}`，下游 `json.loads()` 一切完美。

现实嘛，你拿到的东西可能是：

```
好的，我为您找到了结果：
```json
{
  "action": "search",
  "query": "防御性编程"
}
```
```

或者更离谱的，半角全角引号混用、多对象拼接、字段值里嵌套未转义的大括号。于是你的 pipeline 在凌晨三点就因为一个 `JSONDecodeError` 躺平了。

这不是个别人的痛点——任何长期运行、面向多种模型、开放给不同 prompt 的自动化系统，都会撞到这个暗礁。与其在 prompt 里不断打补丁“千万不要加注释”“千万不要用 markdown 代码块”，不如在解析层做好防御。

## 问题本质：LLM 的格式承诺并不可靠

我们面对的不只是“JSON 字符串不符合规范”那么简单，常见变异有：

- **Markdown 代码块包裹**：` ```json ... ``` `，有时还缺语言标识
- **前后文本混排**：开头的“根据您的需求...”和结尾的“希望能帮到您”
- **多 JSON 对象拼接**：`{"a":1} {"b":2}` 或字典列表 `[{...},{...}]`
- **尾逗号、注释、单引号等 JSON5/JSCS 宽松语法** 
- **字段值中的非转义特殊字符**，如内部包含 `}` 导致正则匹配提前终止
- **Unicode 控制字符或 BOM 头**，让解析器一声不吭就报错

对这些情况进行分类，本质上都是“我们想要一个纯 JSON 文本，但得到的是一个可能包含 JSON 的混合文本”。

## 防御性解析三步法

不用推翻现有流程，只需要在接过 LLM 返回的 `content` 后，用一个健壮的解析函数代替裸 `json.loads()`。以下步骤经过多个 OpenClaw 实际场景验证，可组合使用。

### 1. 预处理：去除常见污染
先做轻量清洗，不必全面，但能拦截大部分低级错误：
- 去掉 BOM 头 `\ufeff`
- 移除首尾不可见控制字符：`strip()` 之外，再用 `re.sub(r'[\x00-\x1f\x7f-\x9f]', '', text)`（排除常用空格和换行）
- 处理全角引号和空格（尤其在中文 prompt 下极易引入）：
  ```python
  text = text.replace('\u201c', '"').replace('\u201d', '"')
  text = text.replace('\u2018', "'").replace('\u2019', "'")
  text = text.replace('\u3000', ' ')
  ```

### 2. 分层提取：先找 JSON 再解析
解析失败时，不要直接放弃，而是从外到内尝试萃取：

**a) 直接解析**
先 `json.loads(text)`，成功则返回。这能覆盖已经干净的输出。

**b) 提取 markdown 代码块**
用正则匹配：
```python
pattern = r'```(?:json)?\s*([\s\S]*?)\s*```'
matches = re.findall(pattern, text)
```
如果找到，对每个匹配内容尝试 `json.loads()`。很多时候 LLM 会只给一个代码块，直接解析第一个即可。如果有多个，可以根据业务决定：取最后一个、取最长，或返回列表。

**c) 提取最外层大括号/方括号**
如果代码块提取为空，说明 JSON 裸露在文本中。使用平衡括号提取：
```python
def extract_balanced(text, open_char, close_char):
    stack = 0
    start = text.find(open_char)
    if start == -1: return None
    for i in range(start, len(text)):
        if text[i] == open_char:
            stack += 1
        elif text[i] == close_char:
            stack -= 1
            if stack == 0:
                return text[start:i+1]
    return None
```
先找 `{`，再找 `[`。通常 JSON 对象或数组能覆盖 90% 的情况。

### 3. 宽松解析与 Fallback
对上一步得到的候选字符串，如果 `json.loads` 仍失败，继续降级：
- 使用 `json5.loads()`（需安装 json5 库），容忍尾逗号、注释、单引号
- 如果还不行，尝试自动修复常见错误：将单引号替换为双引号（注意区分字符串内部的单引号，可用状态机简单判断）
- 记录完整原始输出和部分提取片段到日志，供后续排查

如果所有策略都失败，最后的手段是 **LLM 自我纠正**：将错误信息和原始输出发回模型，要求修复格式再解析。这在 OpenClaw 里可以用带工具调用的 Agent 实现回退。

## 踩坑实录

- **嵌套大括号误匹配**：当字段值本身含有 `}` 且没转义时，平衡提取会提前闭合。解决方式是优先信任代码块提取，因为 markdown 代码块内的 JSON 通常语法正确。
- **多对象 JSON Lines**：有时模型返回每行一个 JSON 对象。我们可先 `splitlines()`，逐行尝试解析，再聚合成列表。注意空行和前后空白。
- **大数字精度**：`json.loads` 默认将整数作为 int，如果数字超长会报错或损失精度，可配合 `parse_int=Decimal` 或直接用字符串传递。
- **非 UTF-8 编码**：理论上 API 都应返回 UTF-8，但偶有 BOM 等，`content.encode('utf-8', errors='ignore').decode('utf-8')` 能救命。
- **性能**：防御性解析多了一步正则和多次尝试，但相较于 LLM 推理延迟，这点开销可忽略。

## 可复用建议

把以上逻辑封装成一个 `robust_json_parse(text)` 函数，放在你所有 Agent 工具的出口和 MCP handler 之前。建议：

- 支持配置当前场景优先把 JSON 当成对象还是数组
- 返回一个 `(parsed_obj, error_info)` 元组，上游可以决定是否熔断
- 错误信息要带上 raw 的前 200 字符，方便快速定位 prompt 问题
- 如果使用 OpenClaw 的 plugin，可以做成一个 middleware：所有 LLM 输出自动经过该解析器，按需开关

## 总结

防御性 JSON 解析看似“不优雅”，但它是 Agent 自动化长期稳定的地基。与其相信模型会严谨遵守格式约束，不如用一套分层兜底逻辑去适应真实输出。当你的解析器从“偶尔报警”变成“即使乱来也能吞下”，你才能放心让 Agent 在无人看管时持续作业。

希望这套方法能减少你凌晨收到 `JSONDecodeError` 告警的次数。

---

