---
title: OpenClaw 实践：让子 Agent 独立会话，告别主线程污染
feedId: 32100
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景

在 OpenClaw 这类多 Agent 编排框架中，主 Agent 调用子 Agent 是极为常见的模式：主 Agent 负责解析用户意图、协调工具与子流程；子 Agent 则专注执行特定任务，例如代码生成、数据检索或文档总结。  

默认情况下，子 Agent 往往会直接复用主 Agent 的会话上下文（Conversation / Memory）。这带来一个潜默化的工程问题：子 Agent 内部的中间推理、工具调用过程、调试信息，会全部写入主会话历史，**污染用户的对话视图**。常见现象包括：

- 用户看到不该暴露的中间步骤（如子 Agent 的 CoT 推理链）
- 主会话上下文迅速膨胀，带来不必要的 token 消耗
- 子 Agent 的局部记忆干扰主 Agent 的后续决策

对于生产级 Agent 应用，这些问题不仅影响体验，还埋下调试和维护的深坑。Session 隔离是最直接的解法，本文给出具体做法、踩坑记录与可复用的工程封装。

## 问题复现

假设你使用 OpenClaw（Python SDK）定义一个主 Agent，并通过 `create_sub_agent` 或类似方法挂接一个代码执行子 Agent：

```python
from openclaw import Agent, Conversation, Memory

# 主会话
main_memory = Memory()
main_conv = Conversation(memory=main_memory)
main_agent = Agent(conversation=main_conv, name="assistant")

# 子 Agent 默认继承主会话
sub_agent = main_agent.create_sub_agent("code_interpreter")
# 执行子任务
sub_agent.run("Generate a Fibonacci function and test it")
```

执行后，`main_conv` 中会包含子 Agent 的完整交互过程，而不仅仅是最终的代码输出。这意味着用户会看到：

```
[assistant] 我需要用 Python 写一个斐波那契函数，然后用 pytest 测试……  
[code_interpreter] 先定义函数…  
[code_interpreter] 调用函数出现错误，需要修复…  
[code_interpreter] 最终代码和测试结果如下：…
```

主 Agent 的对话流被污染，后续会话管理变得复杂。

## 做法与步骤

根本思路是：**为每次子 Agent 调用创建独立的短活会话，只将最终回复摘要或结果注入主会话**。在 OpenClaw 中，可以通过手动注入新会话对象实现：

1. **隔离子 Agent 的 Conversation**：每次调用前，新建一个独立的 `Conversation`，并挂接到子 Agent。
2. **限制子 Agent 的上下文**：仅通过系统提示或 `run` 的 `context` 参数，将完成任务所必需的信息传入，而不是整个主会话历史。
3. **仅捕捉最终输出**：子 Agent 运行完毕后，获取 `response.content`（或结构化结果），将其作为主 Agent 的一条消息追加到主会话中，其余中间信息丢弃。

### 最小可用示例

```python
from openclaw import Conversation, Memory

def run_isolated_sub_agent(sub_agent, task, essential_context: str):
    # 为子 Agent 创建独立内存与会话
    sub_memory = Memory()
    sub_conv = Conversation(memory=sub_memory)
    # 注入必要上下文作为 system 消息
    sub_conv.add_system_message(essential_context)
    # 挂载临时会话
    sub_agent.conversation = sub_conv
    # 执行
    response = sub_agent.run(task)
    # 只将最终回复注入主会话
    return response.content

# 在主 Agent 逻辑中使用
final_output = run_isolated_sub_agent(
    sub_agent,
    task="write a Fibonacci function and test it",
    essential_context="You are a Python coding specialist. Only return the final code and test result."
)
main_conv.add_assistant_message(final_output)
```

### 进阶：复用隔离会话节省 overhead

频繁新建 `Conversation` 会带来 token 起步开销（系统消息重复注入）。可以维护一个“纯净”模板会话 + `Memory`，每次调用时克隆：

```python
import copy

base_conv = Conversation(memory=Memory())
base_conv.add_system_message("You are a Python coding specialist. Only return final result.")

def isolated_runner(sub_agent, task):
    sub_agent.conversation = copy.deepcopy(base_conv)
    return sub_agent.run(task).content
```

注意 `deepcopy` 需保证 `Conversation` 实现可拷贝，若框架不支持，则自己封装一个 `reset_and_setup` 方法：清空消息列表，重新注入 system message，再执行。

## 踩坑点

- **子 Agent 需要共享部分主会话历史**  
  全量隔离可能导致子 Agent 丢失必要上下文（如已定义的变量名称、前置任务结果）。此时不能简单地一隔了之，而应手动提取“上下文摘要”，作为 `essential_context` 参数注入。可以写一个轻量工具函数，从主会话中抽取最近 N 轮关键消息，摘要后传入。

- **工具调用结果仍可能泄漏**  
  如果子 Agent 在隔离会话中使用了工具（如文件操作），而工具的日志或回调默认也写入主会话的观测通道，则依然可能污染。需要确认子 Agent 执行时的 `observer` 或 `listener` 是否也需隔离。在 OpenClaw 中，可将 `sub_agent.observer = None` 或指定一个日志收集器，仅将关键结果向上传递。

- **异步并发时 session 对象竞态**  
  若有多个子 Agent 调用并发执行，且你对同一个 `sub_agent` 实例复用并修改其 `conversation`，会引发竞态条件。最佳实践：每个并发子任务创建一个独立的 `sub_agent` 实例，或使用线程局部存储 (`threading.local`) 隔离会话。

- **token 预算的记账**  
  隔离会话仍需消耗 token，且这些 token 不体现在主会话中，容易导致成本意外增加。应监控子 Agent 调用频次与长度，对需要长时间推理的子 Agent 单独限流。

## 可复用建议

将上述模式封装为 `SessionIsolatedAgentRunner`，支持：

- 模板会话的注入与重置
- 必要上下文的自动抽取（可配置抽取深度、摘要方式）
- 最终输出的后处理与注入主会话
- 并发安全（每个任务使用独立的 `sub_agent` 副本）

使用示例：

```python
runner = SessionIsolatedAgentRunner(
    sub_agent_factory=lambda: main_agent.create_sub_agent("coder"),
    system_prompt="You are a coding specialist. Only return final code and result.",
)
final = runner.run("write a fibonacci function", context_hint="no extra libraries")
main_conv.add_assistant_message(final)
```

这种封装可有效避免主会话污染，同时保持工程上的整洁与可测试性。

## 总结

Session 隔离不是 OpenClaw 的官方 API 能力，而是通过合理运用 Conversation / Memory 的自由度实现的一种工程模式。关键点在于：

- 子 Agent 的中间过程不应对用户可见
- 通过独立会话 + 最小上下文注入，实现调用边界清晰
- 注意工具回调、并发安全和 token 成本

这一模式对于所有使用 OpenClaw 构建多 Agent 应用的团队都具备普适参考价值。在后续迭代中，如果框架本身能够原生支持“沙盒式子调用”，将大幅降低此类问题的工程负担。

---

