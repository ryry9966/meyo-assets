---
title: OpenClaw 子 Agent 会话隔离实战：如何不让推理链污染主上下文
feedId: 31808
source: 综合讨论
publishedAt: 2026-08-06
---

# OpenClaw 子 Agent 会话隔离实战：如何不让推理链污染主上下文

## 背景：多 Agent 协作中的上下文污染

在 OpenClaw 里，当主 Agent 通过 MCP 工具或子任务代理调用另一个 Agent（子 Agent）时，最容易被忽略却伤害最大的问题之一，就是**子 Agent 的内部推理过程被原封不动地写进了主会话历史**。

举个例子：用户让主 Agent “根据最新财报写一份风险摘要”。主 Agent 调用一个专门做财务分析的子 Agent，子 Agent 经过 8 轮思考、搜索、计算，最后才产出了 200 字的摘要。然而默认情况下，主 Agent 的对话记录里不仅包含了这 200 字结果，还包含了子 Agent 那 8 轮完整的 `thought`、`tool_call`、`observation` 消息。这些消息像噪声一样挤占上下文窗口、抬高 token 开销，更严重的是——它们会干扰主 Agent 后续的推理，让模型“迷失”在子任务的细枝末节里。这就是典型的**会话污染**。

## 问题：主上下文为什么会膨胀？

OpenClaw 的 Runner 在解析 MCP 工具调用时，如果把子 Agent 当成一个普通工具，那么子 Agent 执行期间产生的所有消息都会被 append 到当前 session 的 `messages` 列表中。这导致三个直接后果：

- **上下文膨胀**：原本 4k token 的对话，因为一次子任务就涨到 15k，成本线性增加。
- **推理串味**：主 Agent 看到的最后几条消息是子 Agent 的“嗯，我还需要查一下表，再用计算器核验……”，这些碎片会弱化主 Agent 对全局任务的把握。
- **越狱风险**：如果子 Agent 有 system prompt 或特殊指令，不小心泄漏到主上下文，可能绕过安全策略。

理想的形态是：**子 Agent 只向主 Agent 返回一个干净的结果（或有限的摘要），中间推理过程完全隔离。**

## 做法 / 步骤：在 OpenClaw 中实现 session 隔离

OpenClaw 的 session 隔离并不是一个默认打开的功能，但实现起来非常工程化，核心思路是**为子 Agent 创建独立 session，只回传最终结果**。

### 方式一：使用内置子 Agent 工具的 `isolated` 参数

如果你的子 Agent 是以 `AgentTool` 形式注册的 MCP 工具，且 OpenClaw 版本 ≥ 0.9（代码层面已支持 session 隔离），可以直接配置：

```python
from openclaw.tools import AgentTool

subagent_tool = AgentTool(
    agent="finance_analyst",
    description="分析财务数据并生成摘要",
    isolated=True,          # 关键：开启 session 隔离
    max_turns=10,           # 限制内部步数
    return_summary=True     # 只返回最后一条消息的浓缩结果
)
```

`isolated=True` 会让 Runner 在调用该工具时：
1. 基于当前消息新建一个子 session（`session_id` 自动派生）。
2. 子 Agent 在该 session 内运行，其所有消息不写入父 session。
3. 工具返回值只包含子 Agent 的最后一条 `assistant` 消息的 `content`。

如果你的项目用的是原生 MCP 工具而不是 `AgentTool`，那需要自己封装。

### 方式二：自定义 MCP 工具，手动启动新 Runner 并截断返回

适用场景：需要对返回内容做更多控制，比如提取结构化摘要、只返回图片 markdown 而不返回分析过程。

示例代码逻辑：

```python
from openclaw.runner import Runner

def isolated_finance_query(prompt: str, api_key: str) -> str:
    # 创建独立 session
    child_session = Runner.create_session(
        system_message="你是一个专业的财务分析师，只输出最终摘要。",
        isolated=True  # 不挂载到当前全局上下文
    )
    # 运行子 Agent，并限制最大轮次
    final_state = Runner.run(
        session=child_session,
        user_message=prompt,
        max_turns=8,
        model="gpt-4-turbo",
        api_key=api_key
    )
    # 仅提取最终消息
    if final_state.messages:
        return final_state.messages[-1].content
    return ""
```

然后将这个函数包装成 MCP 工具暴露给主 Agent。主 Agent 只会看到一个返回字符串，完全看不到内部蛛网式的 tool call。

### 关键配置项一览

- **session 隔离**：使用 `create_session(isolated=True)` 或工具定义时指定 `isolated=True`。
- **max_turns**：严格控制子 Agent 的推理步数，防止跑飞。
- **return_summary**：如果子 Agent 输出过长，可以打开此选项让框架自动生成一句话摘要返回。
- **不同模型**：子 Agent 可指定比主 Agent 小得多的模型（如 Main: gpt-4, Sub: gpt-3.5-turbo），既省钱又减负。

## 踩坑点：实践中容易翻车的地方

1. **session_id 冲突导致消息串混**  
   自定义子 Agent 时，如果忘记生成新的 session_id（或使用全局默认 session），子 Agent 的消息仍然会写进主会话。**务必使用 `Runner.create_session()` 生成全新 session**，不要复用。

2. **异常消息泄漏**  
   子 Agent 抛异常时，Runner 默认会将 traceback 作为工具返回内容塞给主 Agent。这会导致主会话里出现无法理解的错误堆栈。解决办法：在包装函数里用 `try/except` 捕获异常，返回一个干净的“分析失败，请稍后重试”。

3. **时间顺序混乱**  
   由于子 Agent 独立运行，它产生的消息没有时间戳对齐主会话，如果后续要回溯调试，很难还原完整时序。建议在配置中开启 `logger` 将子 session 的全量消息输出到单独的日志文件，既不影响主上下文又可复盘。

4. **非文本内容的传递**  
   若子 Agent 访问了图像或文件，返回的可能是本地路径或二进制引用。直接以字符串形式返回给主 Agent 会丢失信息。正确做法：将文件先上传到共享存储，返回给主 Agent 一个带 markdown 图片链接的字符串。

## 可复用建议：封装成通用子任务工具工厂

实际项目里，你肯定不止一个子 Agent。推荐写一个工厂函数，统一管理 session 隔离、错误处理和返回格式：

```python
def create_isolated_tool(agent_system_prompt: str, max_turns=6):
    def tool(query: str) -> str:
        session = Runner.create_session(
            system_message=agent_system_prompt,
            isolated=True
        )
        try:
            state = Runner.run(session, query, max_turns=max_turns)
            return state.messages[-1].content
        except Exception:
            return "[子Agent执行失败，已跳过此步骤]"
    return tool
```

然后注册为 MCP 工具时只要一行代码。对于需要返回图片、表格的子 Agent，还可以传入后处理函数，实现“只返回第一个图表，文字丢弃”这样的高级过滤。

另外，测试隔离是否生效的小技巧：**在主 Agent 的对话中添加一句 “如果你看到了子 Agent 的思考过程，就说‘污染了’”作为探针**。如果主 Agent 从未说出这句话，说明隔离成功。

## 总结

OpenClaw 的 session 隔离不是魔法，而是对多 Agent 架构最基本的敬畏。它用很低的成本（一个参数 + 少量封装）解决了上下文污染、成本激增和推理干扰三大问题。当你开始把子 Agent 当成真正的“黑盒工人”——只关心交付物，不关心内部流程——整个系统的健壮性会明显上一个台阶。记住：**主 Agent 的上下文是稀缺资源，别让任何一种中间推理擅自闯入。**

---

