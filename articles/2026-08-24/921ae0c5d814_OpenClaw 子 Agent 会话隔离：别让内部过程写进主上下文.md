---
title: OpenClaw 子 Agent 会话隔离：别让内部过程写进主上下文
feedId: 34420
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

OpenClaw 里，主 Agent 经常会派生子 Agent 去处理长任务：读代码库、批量跑 MCP 查询、执行文件整理。子 Agent 的一个常见问题是：如果它直接复用父会话的消息存储或工具状态，所有内部推理、工具输出、重试日志都会回流到主会话。

这会导致三类问题：

1. **上下文膨胀**：几十条中间步骤写进主消息流，模型注意力被稀释，后续决策质量下降。
2. **状态串扰**：子 Agent 的工具调用和父会话共享同一组 MCP 连接或临时状态，可能出现互相覆盖。
3. **审计困难**：用户在主会话看到大量执行细节，很难分辨哪些是决策，哪些是过程。

所以需要做 session 隔离。这里说的不是“把子 Agent 消息折叠起来”，而是让子 Agent 拥有独立的 session，只把受控结果返回给父会话。

## 做法/步骤

### 1. 显式创建独立 session

在配置里不要只隐藏消息，要真正使用独立的 session id 或 conversation store。以当前 OpenClaw 的 spawn/subagent 工具为例，可以这样表示：

```yaml
subagent:
  session:
    isolate: true
    inherit: [system_prompt, user_profile, task_brief]
    max_context_tokens: 8000
```

这里的关键是：子 Agent 写入的是独立的 event stream，不是父 session 的分支。不同版本的字段名可能有差异，但核心是 `isolate` 或类似的 `new_session` 开关。

### 2. 控制上下文继承

父会话历史不要全量复制给子 Agent。子 Agent 只需要：

- 任务目标
- 关键约束和用户偏好
- 必要的文件路径或外部资源引用
- 当前项目的基本背景

可以准备一个 brief 模板，父 Agent 在派发任务时只填写这个模板。不要直接把整个主会话消息数组传下去，否则隔离就失去了一半意义。

### 3. 工具和 MCP 隔离

子 Agent 默认不要共享父 session 的 MCP 连接。对工具做白名单，并且只读优先。例如：

```yaml
mcp:
  scope: subagent
  servers:
    - name: filesystem
      permission: read_only
    - name: github
      permission: none
```

如果子 Agent 需要写操作，比较稳妥的做法是：子 Agent 只返回“建议操作”，由父会话确认后再执行。这样即使子 Agent 出错或产生幻觉，也不会直接污染外部系统。

### 4. 结构化返回，而不是自然语言总结

子 Agent 结束时只返回一个结构化结果，父 session 只追加一条消息。例如：

```json
{
  "summary": "完成 src/ 目录下 12 个文件的类型检查，发现 3 个错误",
  "artifacts": ["s3://agent-logs/report-20250511.md"],
  "exit_code": 0,
  "changed_files": []
}
```

中间日志写到子 session 自己的 log 或对象存储，不要回传。父会话只关心：做了什么、结果如何、有没有副作用。

### 5. 清理与配额

子 session 需要设置生命周期：

- 最大执行步数或工具调用次数
- 返回内容的最大 token 数
- 空闲 TTL，防止孤儿 session 积累
- 结束后关闭临时 MCP 连接、清理临时文件

## 踩坑点

### 假隔离

只把子 Agent 消息在 UI 上折叠，但底层仍然写入同一个 conversation store。这样 token 消耗、检索召回、向量索引照样被污染。判断标准很简单：子 Agent 运行后，父 session 的底层消息数是否显著增加。

### 上下文继承太薄

只传一句“去处理这个任务”，子 Agent 不知道用户偏好、代码风格、项目约束，会反复追问或产生不符合预期的结果。需要把 system prompt 和关键用户约束复制过去。

### 副作用穿透

session 隔离挡不住外部副作用。如果子 Agent 拿到了文件写权限、数据库写权限或消息发送权限，即使 session 独立，仍然可能改坏外部系统。权限分级是必须的。

### 回传结果过大

子 Agent 把完整中间日志或大段代码作为返回值回传，主上下文又一次被撑爆。需要对返回内容做 token 限制，或者只回传 artifact 路径和摘要。

### 孤儿 session 积累

任务超时、异常退出或用户手动中断时，清理逻辑可能没执行。长期运行后会积累大量空闲 session。需要 TTL 和定期回收脚本。

## 可复用建议

- **区分决策上下文和执行上下文**：主会话保留决策、约束和最终结果；子会话保留过程日志和中间产物。
- **结构化返回 + Schema 校验**：不要依赖自然语言总结，用 JSON 返回，父 Agent 侧做一次轻量校验。
- **MCP 工具分级**：至少分 `read_only` / `suggest` / `write` 三级，子 Agent 默认只给 `read_only`，需要写操作时升级到 `suggest`。
- **日志外置**：子 Agent 过程日志写入独立文件或对象存储，主会话只放引用和摘要。
- **做资源监控**：记录每个子 session 的输入/输出 token、工具调用次数、墙钟时间。发现异常时能快速定位是哪个子任务在消耗主会话资源。

## 总结

OpenClaw 的子 Agent session 隔离，不是简单地隐藏聊天记录，而是把执行过程从主上下文里剥离出去，只通过受控接口回传结果。核心是四个点：独立 session id、最小上下文继承、工具白名单与只读优先、结构化返回。做好这些之后，主会话会更稳定，长任务也不容易把上下文拖垮。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/70a38e8ff6b1cdfc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/5812df3867a061a8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/9c7b40759fdbf607.png)

