---
title: OpenClaw 子 Agent 会话污染实录：三种隔离方案与工程化经验
feedId: 31138
source: 综合讨论
publishedAt: 2026-08-01
---

## 01 背景：一条消息让主会话崩了

在 OpenClaw 里搭建一个“需求澄清 + 代码生成”流水线：主 Agent 接收用户模糊需求，先调一个「需求拆解」子 Agent 补充细节，再调「代码生成」子 Agent 输出实现。一切正常，直到某次长对话里，子 Agent 返回了一堆带格式的历史对话片段——原来它把整段主会话上下文全吞进去了。

结果很干脆：主会话消息区被严重污染，token 消耗暴涨，后续对话逻辑混乱。更麻烦的是，这种污染是累进式的，一旦发生就很难回溯。

翻看 OpenClaw 的 Agent 调度栈发现：默认子 Agent 被调用时，继承的是主 Agent 的 **完整 session 上下文**。这在短平快任务里没问题，可一旦涉及多轮复杂交互，就像是把整个聊天记录当作函数入参传给一个纯函数——既不必要，又危险。

## 02 问题拆解：污染到底从哪里渗入？

根因并不复杂，但表现多样：

- **上下文溢出**：子 Agent 执行需要参考的信息其实很少，但它“看到”了主会话里大量无关历史，导致幻觉和无关输出。
- **内存型状态残留**：主 Agent 依赖的 memory 插件（如 vector store 或 conversation buffer）会被子 Agent 的无用消息稀释，干扰后续检索。
- **工具调用副作用**：子 Agent 通过 MCP 服务读写外部资源时，若使用同一个会话凭证，很容易把临时文件、环境变量写错位置。
- **递归污染**：子 Agent 可能再调子 Agent，若全部共享一个 session_id，错误会逐层放大。

本质上，OpenClaw 的 **session 是驱动式上下文**（driven context），一旦没有显式隔离，它会像水流一样渗进每一个工具节点。

## 03 做法：三种隔离方案与关键步骤

以下基于 OpenClaw 0.7+、支持 MCP 与自定义插件。所有方案都在生产环境跑过，你可以按复杂度递进选择。

### 方案一：独立 session_id + 最小上下文

**步骤**：
1. 在插件或工具函数里，不再直接 `agent.call(sub_agent)`，而是为子 Agent 创建**全新 session**：
   ```python
   sub_session = openclaw.create_session(
       agent_name="demand_splitter",
       user_id=current_user_id,
       inherit_context=False  # 关键点
   )
   ```
2. 只注入必要信息：
   ```python
   sub_session.add_message("user", f"请拆分需求：{requirement}")
   ```
3. 运行后只回传 result 摘要，不合并消息历史：
   ```python
   result = await sub_session.run()
   return result.summary
   ```
4. 子 session 结束后立刻调用 `sub_session.close()`，释放资源并防止残留在全局 store 里。

**踩坑点**：
- `inherit_context=False` 在早期版本不完全生效，需检查框架内 `MemoryManager` 是否仍携带主 session 的上下文片段。解决方案是显式清空子 session 的 memory 插槽。
- 如果子 Agent 需要访问用户偏好设置，不能直接读主 session 的 meta，必须通过参数显式传入。
- 并发调用时，需注意 `create_session` 的 user_id 锁，避免误触速率限制。

### 方案二：基于 MCP 的工具级隔离

如果你把子 Agent 包装成 MCP 工具，隔离可做到更彻底。

**步骤**：
1. 将子 Agent 注册为一个无状态 MCP server：每次调用新建一个临时执行环境。
2. 工具函数内使用 `openclaw.mcp.ToolSession` 代替全局 session：
   ```python
   async def call_sub_agent(input: str) -> str:
       async with ToolSession(agent_def="code_gen", ttl=120) as ts:
           return await ts.ask(input)
   ```
3. `ToolSession` 自动管理上下文注入与清理，不触及主 Agent 的对话树。
4. 输出格式统一为 JSON，避免 unstructured 文本混入主会话。

**踩坑点**：
- MCP 工具的 `ToolSession` 依赖底层 MCP hub 的资源调度，出现过子 session 未及时回收导致 `session_pool` 打满的情况。设置 `ttl` 并加主动超时检查。
- 注意工具调用的 credential 传递，不要复用主 Agent 的 API key 导致权限泄露。

### 方案三：context stack 显式控制（高阶）

适合需要保留部分共享上下文，但精细控制窗口的场景。

**步骤**：
1. 使用 OpenClaw 的 `ContextStack` API 在调用子 Agent 前推入新的隔离层：
   ```python
   ctx.push_layer(Layer(memory_policy="isolated", max_tokens=800))
   await ctx.call_sub_agent(sub_agent, input)
   result = ctx.pop_layer()
   ```
2. 在 layer 内配置记忆策略：`isolated`、`shared_readonly` 或 `diff`。
3. 子 Agent 只看到父层透传的“入口消息 + layer 内新消息”。

**踩坑点**：
- `pop_layer` 必须与 `push_layer` 配对，异常路径下容易造成栈残留。建议用 `try/finally` 或 `async with ContextStack.scope(...)`。
- 多层嵌套时，token 限制是累加计算的，需在每层做边界检查。

## 04 可复用建议

归纳三条能直接带走的经验：

1. **封装隔离调用器**  
   写一个 `isolated_invoke(agent_def, input, options)` 工具函数，统一处理 session 创建、参数注入、结果截断和资源回收。团队内部使用可避免每个人重复踩坑。

2. **建立子 Agent 输出契约**  
   规定子 Agent 只能返回结构化摘要（如 `{"type":"split_result","items":[...],"rationale":"..."}`），并在主 Agent 侧做严格的 schema 校验。不规范的输出直接丢弃并记录。

3. **做“污染检测”桩**  
   在主会话内加入轻量检测：若发现最近 N 条消息中包含子 Agent 的内部标记（如特殊 symbol `__SUB_TRACE__`），立即触发告警并回滚上下文。这个低成本的防御机制能拦截 90% 以上的意外污染。

## 05 总结

子 Agent 会话污染不是 OpenClaw 的 bug，而是 Agent 架构下必然会碰到的一个上下文治理问题。从简单的 session 分叉，到 MCP 工具无状态化，再到 context stack 的细粒度控制，工程上需要根据任务的副作用容忍度和性能要求做权衡。

核心原则是：**不要让子 Agent 看见它不该看见的，也别让它留下的东西混进主线程。** 每次调用子 Agent 前问自己一句——“它真的需要知道我们聊了这么久吗？” 答案往往是否定的。

---

