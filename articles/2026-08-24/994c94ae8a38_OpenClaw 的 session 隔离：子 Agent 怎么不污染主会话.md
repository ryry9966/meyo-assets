---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 34513
source: 综合讨论
publishedAt: 2026-08-24
---

# OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话

## 背景

在 OpenClaw 里做自动化和多 Agent 协作时，子 Agent 很容易变成“上下文污染源”。比如你让子 Agent 去抓一批网页、生成一段代码，或者通过 MCP 调外部工具，默认情况下它会沿用当前 session。于是子 Agent 的中间推理、重试堆栈、工具输出、报错信息，甚至敏感变量，都会混进主会话。主 Agent 开始说一些跟当前任务无关的话，token 消耗暴涨，更麻烦的是，有时候子 Agent 的输出会被主 Agent 误当成新指令执行。

这个问题不是 OpenClaw 独有的，但 OpenClaw 的 session 和工具继承机制决定了它特别需要显式隔离。

## 问题拆解

我一般把“污染”分成三类：

1. **上下文污染**：对话历史被无关步骤占满，主 Agent 找不到关键信息。
2. **指令污染**：子 Agent 返回的文本被主 Agent 当成用户输入继续操作。
3. **副作用污染**：子 Agent 通过 MCP 工具写文件、改配置，影响主任务。

只解决第一类是不够的。很多同学做了 session 隔离，结果又把子 Agent 的完整输出手动塞回主会话，等于白隔离。

## 做法 / 步骤

我现在的做法是三层隔离，不一定每层都用，但前两层基本必须做。

### 第一层：session 级隔离

创建子 Agent 时不要沿用当前 session，而是新建独立 session。配置示例：

```yaml
# openclaw.yaml
subagent:
  isolation: strict
  session:
    ttl: 300
    namespace: child-{task_id}
  return:
    mode: summary
    max_chars: 1800
  tools:
    inherit: false
    allow: [read_file, web_search]
```

代码调用大概是这样：

```python
child_session = openclaw.session.create(
    namespace=f"child-{task_id}",
    isolated=True,
    ttl=300,
)

try:
    result = openclaw.agent.run(
        task=task,
        session=child_session,
        return_mode="summary",
        max_tokens=4000,
    )
finally:
    openclaw.session.close(child_session.id)
```

`isolated=True` 保证子 Agent 的对话历史、缓存、系统提示不会进入主 session。`ttl` 和 `finally` 是为了防止子 Agent 异常退出后 session 残留，下个任务读到上一个子任务的上下文。

### 第二层：返回值级隔离

不要让子 Agent 返回完整日志。即使 session 隔离了，如果你把 `result.full_output` 塞回主会话，等于手动把污染引回来。

我建议只回传结构化摘要：

```json
{
  "ok": true,
  "summary": "抓取到 3 个页面，主要变化在价格字段",
  "artifacts": ["obj://child-12/result.json"],
  "errors": []
}
```

字段控制在 3-5 个，摘要长度限制在 1500 字符内。关键文件写进独立对象存储或子工作区，主会话只拿引用，不拿内容。

### 第三层：工具权限隔离

子 Agent 通过 MCP 调用外部工具时，默认继承主会话工具集是很危险的。要配置 `inherit: false`，只给子 Agent 必需的只读工具。如果必须写文件，给子 Agent 单独的工作目录，例如 `/workspace/child/{task_id}`，不要直接写主工作区。

## 踩坑点

1. **只隔离上下文，不隔离副作用**。有次子 Agent 用同一个 `write_file` 工具写了一个临时配置文件，结果把主任务配置覆盖了。后来统一给子 Agent 挂载独立目录，才解决。

2. **`summary` 模式不是万能**。摘要模型可能把关键数字或异常信息截断，主会话看着“正常”，实际子任务已经失败。建议 summary 里强制包含 `ok`、`error_count`、`artifact_ids` 字段。

3. **session 复用残留**。如果子 Agent 异常退出，没有执行清理，下个任务可能读到上一个子 Agent 的系统提示。设置 TTL 和唯一 namespace，并在 `finally` 里关闭 session。

4. **环境变量继承**。子 Agent 默认继承主进程的 API key 和 secrets，日志里可能泄露。用最小权限注入，只传必需变量。

## 可复用建议

- 默认 `inherit: false`，需要什么工具显式加白名单。
- 回传摘要而不是原文，字段越少越好。
- 给每个子 Agent 设置独立 namespace 和 TTL。
- 给子 Agent 设置 `max_tokens` 和 `timeout`，避免失控。
- 关键失败要在主会话可见，不要用“静默失败”的返回结构。

## 总结

OpenClaw 的 session 隔离不只是一个参数开关。它需要把上下文边界、返回值结构、工具权限和清理策略一起设计。只做 session 隔离容易被“看似隔离，实则回流”坑到。把子 Agent 当成不可信的外部服务，只给它完成工作所需的最小上下文和权限，主会话才会稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/d450e29f2ac01876.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/a6a14712ef536f9d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/30f68c4efccc1d01.png)

