---
title: OpenClaw 子 Agent 如何彻底隔离主会话：session 工厂与上下文边界实践
feedId: 32748
source: 综合讨论
publishedAt: 2026-08-12
---

# OpenClaw 子 Agent 如何彻底隔离主会话：session 工厂与上下文边界实践

## 背景：多 Agent 协作中的隐藏雷区

在 OpenClaw 中，当你拆解复杂任务给多个子 Agent 分别执行时，很容易遇到一个“非功能性问题”：**子 Agent 的思考过程、工具调用记录、中间变量，悄无声息地混入了主 Agent 的 session 存储，导致主会话上下文膨胀、状态污染，甚至逻辑错乱**。

这在以下几种场景尤其致命：

- 同一个主 Agent 里启动多个子 Agent 并行处理，最后汇总时出现了不该有的“幽灵记忆”；
- 子 Agent 内部使用了与主 Agent 同名的变量（如 `summary`、`result_list`），覆盖了主会话的中间数据；
- 工具调用结果中的敏感信息本该只存在于子任务生命周期内，结果却留在了主 session 的 history 中，被后续的工具调用或 LLM 推理意外引用。

本质上，这是一个**上下文边界（context boundary）与状态生命周期管理**的问题。OpenClaw 的 session 机制天然支持隔离，但默认工厂可能复用同一个 `session_id`，导致跨 Agent 的 memory 和 history 共享。要想彻底隔离，需要我们显式控制 session 的创建与注入策略。

## 问题梳理：污染是如何发生的

### 1. 默认 session 复用
OpenClaw Agent 在初始化时，如果不指定 session 工厂，很多实践会用同一个 `session_id` 或者干脆不传，导致所有子 Agent 的对话历史、工具返回值、变量存储都写入同一个 session 后端（如 `MemoryStore` 或 Redis）。主 Agent 再次调用 LLM 时，整个 history 都会带上前一个子任务的痕迹。

### 2. 变量命名空间未隔离
即使你用前缀区分，也难保复杂业务里不会冲突。更糟的是，在 function calling 流程中，LLM 有时会“看到”不该出现的变量值，从而做出错误决策。

### 3. session 的生命周期耦合
子 Agent 执行完毕后，你期望释放掉它的上下文，但因为 session 共享，你不敢轻易 `clear`，否则连主 Agent 的有效信息一起被清空。这导致内存/存储占用越来越高。

## 做法步骤：构建隔离的子 Agent session 工厂

以下做法基于 OpenClaw 的 `AgentConfig` 与 session 管理能力，无需改动框架源码，只需在定义子 Agent 时主动注入隔离策略。

### Step 1：定义隔离的 session 工厂

创建一个能生成独立 `session_id` 且使用独立存储后端的工厂函数：

```python
import uuid
from openclaw import SessionConfig, MemoryStore

def isolated_session_factory(base_store: MemoryStore):
    """返回一个新的 session 配置，使用唯一的 session_id 和独立的 memory 空间"""
    sid = f"sub_{uuid.uuid4().hex[:8]}"
    # 核心：使用独立的 MemoryStore 实例，避免共享主 session 的变量
    sub_store = MemoryStore()
    return SessionConfig(
        session_id=sid,
        store=sub_store,
        # 可选：限定此 session 的最大轮数，防止 sub Agent 对话过长
        max_rounds=10,
        # 可选：关闭 session 的自动持久化，避免写入公共存储
        auto_persist=False,
    )
```

### Step 2：在子 Agent 创建时注入该工厂

```python
from openclaw import Agent, AgentConfig

def create_sub_agent(task_description: str, parent_main_agent):
    """基于主 Agent 的环境，但使用隔离 session 创建子 Agent"""
    sub_session = isolated_session_factory(parent_main_agent.session.store)
    config = AgentConfig(
        role=task_description,
        llm_config=parent_main_agent.config.llm_config,  # 复用 LLM 配置
        # 工具列表需要显式指定，不宜直接复用主 Agent 的全部工具
        tools=[web_search, file_reader],
        session=sub_session,
        # 关键：标记为子 Agent，便于上层控制器管理生命周期
        metadata={"type": "sub", "parent_session": parent_main_agent.session.id},
    )
    return Agent(config=config)
```

### Step 3：主 Agent 调用子 Agent 并只取结果

主 Agent 侧应当只关心子 Agent 的最终产出，而不是整个会话历史。因此可以这样封装：

```python
async def run_sub_agent_isolated(main_agent, task_prompt):
    sub = create_sub_agent(task_prompt, main_agent)
    # 仅将必要信息作为 user 消息传给子 Agent，不要共享主 session 上下文
    response = await sub.run(task_prompt)
    # 关键：只提取最后一条消息内容作为返回值，不保留子 Agent 记忆
    final_output = response.messages[-1].content if response.messages else ""
    # 可选：手动清理子 session 的 store
    sub.session.store.clear()
    return final_output
```

## 踩坑点与避坑指南

1. **工具权限泄露**  
   如果子 Agent 复用了主 Agent 的工具列表，而某些工具具备对主 session 的写入能力（例如往主 store 塞变量），隔离就会失效。务必为子 Agent 构建一个受控的工具集，必要时通过 MCP 服务远程调用，进一步切分边界。

2. **LLM 记忆残留**  
   即使 session 隔离了，子 Agent 的 LLM 调用仍可能在无状态模式下遗留部分上下文在客户端。建议为子 Agent 设置较低的 `max_rounds`，强制在预设轮次内结束，避免长对话产生意外缓存。

3. **调试困难**  
   隔离后，子 Agent 的内部链式思考、工具报错不再出现在主日志流中。推荐在子 Agent 的 session 配置中开启独立的日志记录，或使用异步 hook 将关键事件发送到专门的事件总线。

4. **不要过度创建 store 对象**  
   `MemoryStore` 虽然轻量，但并发过多时会占用大量内存。如果需要持久化，可以共享同一个 Redis 实例，但用 `session_id` 作为 key 前缀天然隔离数据，并通过 TTL 自动清理。

## 可复用建议

- **工厂模式沉淀**  
  把 `isolated_session_factory` 和 `create_sub_agent` 抽象成团队公共组件，所有子 Agent 创建统一入口，避免在业务代码里手动拼接 session 配置。

- **设计清晰的上下文协议**  
  规定主 Agent 与子 Agent 之间的数据传递格式：主 Agent 只传 `task_context`（结构化的 JSON），子 Agent 返回 `result_payload`，不依赖任何共享内存变量。这便于以后替换为 MCP 工具或远程 Agent。

- **生命周期钩子**  
  在子 Agent 的 `on_finish` 回调中自动执行 `store.clear()`，防止开发人员遗忘导致内存泄露。OpenClaw 的 Agent 生命周期事件支持这种扩展。

- **为子 Agent 单独配置 MCP 服务**  
  如果 MCP 工具涉及敏感资源，可以为子 Agent 启动一个临时 MCP session，使用一次性令牌，任务结束即销毁。

## 总结

OpenClaw 的 session 隔离看似是“加一个 session_id”的小改动，但在工程实践中牵扯到存储、工具权限、上下文生命周期等一系列设计细节。通过**工厂模式显式控制 session 创建 + 上下文传递协议 + 自动清理机制**，可以做到子 Agent 完全不污染主会话，让多 Agent 协作体系的熵值真正可控。

下一步可以考虑：结合 OpenClaw 的 `SessionMiddleware` 实现更自动化的隔离策略，让框架为每个子任务自动 fork 出一个隔离 session 并回收。在此之前，手工控制依然是最可靠的方式。

---

