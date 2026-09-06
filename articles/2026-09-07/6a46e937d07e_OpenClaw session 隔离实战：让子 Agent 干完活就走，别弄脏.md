---
title: OpenClaw session 隔离实战：让子 Agent 干完活就走，别弄脏主会话
feedId: 36383
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

OpenClaw 的主会话（session）是用户对话、长期记忆和工具调用的默认容器。跑自动化任务时经常需要拆子 agent：并行抓数据、执行长任务、调用一组 MCP 工具。但默认配置下，子 agent 和主会话之间的上下文边界是模糊的，用几次就会体会到“污染”的代价。

## 问题：污染是怎么发生的

实际项目里遇到三种典型污染：

1. **上下文膨胀**：子 agent 的中间推理和工具原始输出全部回灌主会话，几轮下来 token 消耗翻倍，主 agent 开始“忘事”。
2. **记忆污染**：子 agent 在任务中触发了 memory 写入，把主会话的用户偏好文件覆盖成了任务临时结论。
3. **指令串扰**：子 agent 的 system prompt 或任务描述里的关键词被主会话读到，主 agent 后续行为被带偏，比如开始替用户执行不该做的运维动作。

## 做法

核心原则一句话：**子 agent 是函数，不是同事**。只传参数，只收返回值。

以下以自托管 OpenClaw 网关为例（字段名以你本地版本文档为准）：

1. **spawn 时显式开隔离 session**，不复用主会话 id：

```yaml
subagent:
  session: isolated        # 独立 session，不继承主会话历史
  inherit_prompt: false    # 不继承主 system prompt
  tools: [web.fetch, fs.read]   # 最小工具集
  cwd: /tmp/oc-worker-01   # 独立工作目录
```

2. **传任务只传任务**：目标、约束、输出格式写进任务描述，不要指望子 agent“自己看上下文”。
3. **回传走结构化摘要**：约定子 agent 最终只返回一个 JSON（结论 + 关键数据 + 失败原因），中间 transcript 留在隔离 session 里，任务结束即销毁。
4. **MCP 工具按 session 授权**：给子 agent 单独的 MCP server scope，禁止它触碰 memory 写入、session 管理这类主会话级工具。
5. **加超时和资源上限**：子 agent 必须有 timeout，跑飞了由网关回收。

## 踩坑点

- `inherit_prompt` 忘了关，子 agent 带着主会话“人格”干活，返回结果里夹带对用户的回复，回灌后主 agent 逻辑错乱。
- 并行子 agent 写同一个 memory 文件互相覆盖，要么禁写，要么按任务 id 分文件。
- 回传结果没截断：一个网页抓取任务返回 40KB 原始 HTML，“隔离”形同虚设。摘要层务必设长度上限。
- session 池化复用：某些部署下 isolated session 会被回收复用，上个任务的残留指令影响下个任务。确认池化策略，或在任务头加一次性 nonce 让旧内容失效。

## 可复用建议

- 把 spawn 配置做成模板：工具白名单 + 输出 schema + 超时，新任务只填参数。
- 每个子 agent 留独立审计日志，主会话只记“派发了什么、收到了什么”，出问题能回放。
- 定期检查主会话的 token 构成，如果工具输出占比异常，说明隔离有漏洞。

## 总结

session 隔离的本质是控制信息的合法通道：**进只走参数，出只走摘要，权限按最小化配置**。做到这三点，子 agent 并发再多，主会话依然干净、可预测。隔离不是性能优化，是正确性问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/e8813bc23e337873.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/1a2f2504ff0b5c91.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/a092c3214143de71.png)

