---
title: OpenClaw 实战：子 Agent 不污染主会话的 Session 隔离方案
feedId: 31786
source: 综合讨论
publishedAt: 2026-08-06
---

### 背景

在基于 OpenClaw 构建多 Agent 自动化流水线时，我经常遇到一个隐蔽却致命的问题：明明主 Agent 的推理逻辑是对的，但只要调用过一两次子 Agent，主会话就逐渐“跑偏”，开始把子 Agent 内部的调试信息、工具调用回执当成对话历史，甚至对用户自言自咎。排查后发现，根源在于子 Agent 执行过程中的所有中间消息被直接写入了主会话的 `Session` 对象，造成了上下文污染。

在 OpenClaw 的早期版本中，`Session` 是所有消息的共享容器，子 Agent 通过 `Agent.run()` 派生出的交互会直接追加到同一个消息列表。对于需要频繁调用子 Agent 的产线（如路由客服、多步数据分析），这种设计会让主 Agent 的上下文急速膨胀，同时引入噪声，严重影响稳定性和可解释性。

### 问题透析

典型的污染路径如下：

1. 主 Agent 识别出需要读取数据库，调用 `fetch_report` 工具。
2. 工具内部启动子 Agent，子 Agent 先生成 SQL、再根据执行结果进行二次追问、纠错，这个过程产生了五六条消息（包括带 `tool_calls` 的内容）。
3. 工具只返回最终的表格字段，但这些中间推理全部留在了主会话的同一个 `Session` 中。
4. 主 Agent 在下一步规划时，看到了那些子 Agent 的中间消息，误认为用户曾提出过那些细节追问，开始编造不存在的前置条件。

从工程角度看，这不是模型幻觉，而是会话边界失守。

### 做法：利用 `new_child` 显式隔离

OpenClaw 从 `1.3.0` 起提供了 `Session.new_child(isolated=True)` 方法，专门用于派生上下文隔离的子会话。核心思路是：子 Agent 在自己的会话中完成全部推演，主会话只接收必要的最终结果。

**步骤拆解**

```python
from openclaw import Session, Agent

# 主会话
main_session = Session()

# 注册为主 Agent 的工具
def isolated_lookup(natural_query: str) -> str:
    # 1. 创建完全隔离的子会话，不继承历史，只携带必要的上下文摘要
    child = main_session.new_child(isolated=True)
    # 可选：若子 Agent 需要最近一轮用户消息，可传递摘要
    child.add_message("system", "你是一个信息检索助手，只输出最终答案，不要解释过程。")

    # 2. 绑定独立的 Agent，该 Agent 的工具集不会影响主会话的工具状态
    child_agent = Agent(session=child, tools=retrieval_tools)

    # 3. 执行子任务，获取最终回复对象
    response = child_agent.run(natural_query)

    # 4. 仅提取纯文本结果，丢弃中间步骤
    final_text = response.final_output.strip()

    # 5. 将结果返回主会话（只会作为工具返回值呈现给主 Agent）
    return final_text

main_agent = Agent(session=main_session)
main_agent.register_tool("lookup", isolated_lookup)
```

执行后，主会话的历史里只会出现 `lookup` 工具的调用记录和返回值，子 Agent 的思考过程完全不可见。

**进阶用法**

- **携带有限上下文**：如果子 Agent 必须看到最近几轮对话，可使用 `new_child(history_window=3)` 仅复制最近三条消息，而非整个历史。
- **使用上下文管理器**：`with main_session.isolated_child() as child: ...` 可以自动在退出时销毁子会话，防止残留引用。
- **结合 MCP 工具隔离**：为每个子 Agent 实例化独立的 `Toolset` 对象，避免工具实例内部缓存成为新的污染源。

### 踩坑记录

1. **忘记设置 `isolated=True`**  
   若直接用 `Session()` 并赋值 `main_session=child`，子会话仍然是主会话的一片内存，所有消息双向可见。这块在文档里默认值并不明显，我至少被坑过两次。

2. **`final_output` 未被正确清洗**  
   子 Agent 可能在最终回复中附带 `<thinking>` 标签或 Markdown 代码块标记，直接返回会让主 Agent 误解为需要继续对话。务必在返回前做一次正则清洗，只保留纯答案。

3. **并发场景下的 session 竞态**  
   当多个子 Agent 同时运行且共享某些全局工具注册表时，即使会话隔离，部分工具的内部状态（例如共用的数据库连接池）仍可能交叉影响。我在高并发压测中见过 `Session` 对象非线程安全导致的偶发串行化异常。解决方案是使用 `threading.local()` 为每个子 Agent 创建独立连接或封装成无状态函数。

4. **子 Agent 异常退出后会话未释放**  
   如果子 Agent 内部抛出异常而未显式销毁 `child`，可能造成内存泄漏或日志里出现悬空 session id。一律使用 `try/finally` 或上下文管理器清理。

### 可复用建议

- **封装装饰器**：将隔离逻辑收口成一个 `@isolated_task` 装饰器，内部处理会话创建、清洗、异常回收，团队其他人直接使用，不易出错。
- **压测验证**：每次发布前用 100 次连续子 Agent 调用的场景检查主会话历史长度，确保每次调用只增加一条工具消息，而不是成倍膨胀。
- **记录子会话 ID**：在日志里输出 `child.session_id`，方便事后审计哪一次调用造成了污染。
- **摘要策略**：若子 Agent 必须了解主会话进展，建议用轻量摘要模型生成上下文摘要，而不是直接把原始历史塞进去，成本和噪声双降。

### 总结

Session 隔离是 OpenClaw 从原型转向生产化必须越过的一道坎。通过 `new_child(isolated=True)` 将子 Agent 的推理放进黑盒，让主会话保持清爽，既控制了 token 开销，又避免了上下文污染导致的隐性逻辑漂移。这个模式在长流程、工具密集、需要多级调用的 Agent 应用中已成为我的标准做法。

---

