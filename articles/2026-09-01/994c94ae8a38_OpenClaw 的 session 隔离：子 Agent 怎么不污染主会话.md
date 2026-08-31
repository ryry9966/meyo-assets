---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 35591
source: 综合讨论
publishedAt: 2026-09-01
---

# OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话

## 背景

在 OpenClaw 里做多 Agent 编排时，一个很常见的模式是：主 Agent 收到任务后，拆解并派发给若干子 Agent 执行，然后汇总结果。这个模式本身不复杂，但如果不做 session 隔离，子 Agent 的上下文会直接回灌进主会话——对话历史膨胀、系统提示词被稀释、中间推理过程混入最终决策流，导致主 Agent 后续行为跑偏。

这不是 bug，是架构上欠了一笔债。

## 问题：污染具体指什么

子 Agent 污染主会话通常表现为三种形态：

1. **上下文膨胀**：子 Agent 的完整推理链、工具调用日志、错误重试记录全部写回主会话历史，几轮下来 token 消耗翻倍。
2. **状态串味**：子 Agent 内部的临时变量、中间结论、甚至未完成的 thought 片段残留在主会话中，主 Agent 后续推理时把这些噪声当成有效上下文。
3. **指令漂移**：子 Agent 带着自己的一套 system prompt 或行为约束进场，执行完后这些约束没有被剥离，反而渗透进主会话的语义空间。

核心矛盾在于：**子 Agent 需要足够的上下文才能干活，但主 Agent 只需要它的结论，不需要它的过程。**

## 做法：四步隔离

### 1. 独立 session 创建子 Agent

给每个子 Agent 分配独立的 session ID，禁止复用主会话的 session：

```python
sub_session = openclaw.create_session(
    parent_session_id=main_session.id,
    scope="subagent",
    ttl=600  # 子会话独立过期
)
sub_agent = SubAgent(
    session=sub_session,
    system_prompt=SUB_TASK_PROMPT,  # 子 Agent 自己的提示词，不继承主提示词
)
```

关键点：`parent_session_id` 只用于追踪血缘关系，不用于共享上下文。

### 2. 显式裁剪输入上下文

不要直接把主会话的完整历史传给子 Agent。只抽取与子任务相关的必要信息：

```python
sub_input = extract_relevant_context(
    main_session,
    keys=["user_requirement", "target_file", "constraints"],
    max_tokens=2000
)
result = await sub_agent.run(sub_input)
```

这要求主 Agent 在派发任务时，明确子任务的输入边界，而不是"你自己看着办"。

### 3. 结构化回收结果

子 Agent 执行完毕后，只把结构化结果写回主会话，过程日志全部留在子会话内：

```python
# 子 Agent 内部：完整过程记录在 sub_session
sub_session.messages.append(intermediate_steps)

# 主会话只接收结构化摘要
main_session.inject(
    role="tool_result",
    content={
        "sub_task_id": task.id,
        "status": "completed",
        "summary": result.summary,     # 100-200 字的结论
        "artifacts": result.artifacts, # 产出物引用
        "errors": result.errors[:3],   # 只保留关键错误
    }
)
```

### 4. 子会话生命周期收口

子任务完成后，子 session 要么立即销毁，要么标记为只读归档。不要让子 session 继续挂在主会话下面累积状态：

```python
await sub_session.close(reason="task_completed", archive=True)
```

## 踩坑点

**坑一：图省事直接传主会话引用。** 子 Agent 拿到主会话的 session 对象后，一个不留神就往里写东西。最隐蔽的情况是子 Agent 调用工具时，工具回调把日志写到了全局上下文。解决：子 Agent 的工具回调必须绑定子会话的 logger。

**坑二：裁剪过度导致子 Agent 瞎猜。** 如果只给子 Agent 一句"处理一下这个文件"，它缺上下文就会自己编造假设。裁剪的关键不是"给得少"，而是"给得准"。至少要包含：任务目标、输入数据引用、约束条件、预期输出格式。

**坑三：结果回收格式不统一。** 有的子 Agent 返回纯文本，有的返回 JSON，有的返回文件路径。主会话面对五花八门的返回格式，后续处理很痛苦。建议定义一个统一的 `SubTaskResult` schema，所有子 Agent 必须遵守。

**坑四：忽略了并发子 Agent 的写冲突。** 多个子 Agent 同时执行时，如果都尝试往主会话注入结果，可能出现顺序错乱。解决：主会话侧设置一个结果收集器，子 Agent 只往收集器写，由主 Agent 在收口阶段统一合并。

## 可复用建议

1. **默认隔离，显式共享**。子 Agent 的 session 默认完全独立，需要共享什么必须显式声明。把"默认继承"反过来，能避免 80% 的污染问题。
2. **结果 schema 先行**。在写任何子 Agent 逻辑之前，先定义 `SubTaskResult` 的结构。这个 schema 是子 Agent 和主 Agent 之间的契约。
3. **子会话要有 TTL 和归档机制**。不要信任"任务完成后手动关闭"这个动作，靠 TTL 兜底。
4. **主会话里只保留"决策相关信息"**。问自己一个问题：这条信息如果从主会话里删掉，主 Agent 的下一步决策会变差吗？不会就删。
5. **调试时开启会话血缘追踪**。在日志里记录每个 session 的 parent/child 关系，排障时能快速定位是哪个子 Agent 污染了主会话。

## 总结

Session 隔离的本质是**信息边界管理**：子 Agent 需要充分的执行上下文，主 Agent 需要干净的决策上下文。这两者天然冲突，所以不能靠"自觉"来维护，要靠架构约束。四个步骤——独立 session、裁剪输入、结构化回收、生命周期收口——落地成本不高，但对多 Agent 编排的稳定性提升是实打实的。

当你的 OpenClaw 编排里出现了三个以上子 Agent 时，这套东西就不是可选项了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/91c3023beca4260b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/564e5c0c1e476192.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/d3ce57709378cbc7.png)

