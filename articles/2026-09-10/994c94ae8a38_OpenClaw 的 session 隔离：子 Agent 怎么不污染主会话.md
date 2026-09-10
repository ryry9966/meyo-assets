---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 36857
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

OpenClaw 的典型玩法是：主 agent 挂着对话上下文和长期 memory，业务任务再派生一批子 agent——有的是插件的定时任务，有的是通过 MCP 拉起的一次性执行体。子 agent 跑得越多，一个老问题就越明显：主会话的上下文“越来越脏”。

## 问题

我们实际踩到的污染有三类：

1. **上下文膨胀**：子 agent 的中间输出（工具结果、长日志）被回写进主会话历史，几轮任务下来 token 占用翻倍，主 agent 的注意力被无关内容稀释，回答质量肉眼可见地下滑。
2. **状态串写**：子 agent 和主会话共享同一份 memory，一个抓取任务把主会话里的用户偏好字段覆盖了，主 agent 之后的行为就“变了个人”。
3. **事件回灌**：子 agent 内部的事件循环消息回流到主 transcript，重放时主 agent 把子 agent 的中间自言自语当成了对话历史。

根因是同一个：子 agent 与主会话之间的数据边界是隐式的，进出全靠默认行为。

## 做法

我们的隔离方案分四步：

**1. 显式隔离 spawn。** 派生子 agent 时不复用主 session id，而是生成独立 session：子 agent 拿到空白上下文 + 任务描述，主会话历史对它不可见。

**2. 结果回传走 schema。** 子 agent 结束时只回传结构化摘要，字段约定为 `status / result / error / token_used`，主会话只追加这条摘要。全量 transcript 留在子 session，按需落盘归档，不进主上下文。

**3. 共享状态显式化。** 需要跨 agent 共享的数据放 workspace 的键值存储或文件，读写在代码里显式完成。主 memory 对子 agent 只读，或干脆不给权限。

**4. 权限与预算收敛。** 子 agent 只挂载任务必需的 MCP 工具；同时设轮数上限、token 上限和超时，超限直接终止并返回失败摘要，避免失控的子 agent 反复回写。

## 踩坑点

- 最早忘了关子 agent 的流式回传，每条中间消息都进了主历史，隔离形同虚设。上线前要专门确认 `stream_to_parent` 类开关是关的。
- 摘要回传一开始没做 schema 校验，某个子 agent 直接把原始日志塞进 `result` 字段——污染只是换了个形式回来。后来加了校验 + 长度截断。
- 并发子 agent 共用同一个临时目录，两个任务互相覆盖中间文件。改成每个子 session 独立 scratch 目录。
- 重试逻辑会把失败子 agent 的“半成品输出”也回灌，改成只有成功才回传，失败只回错误码和原因。

## 可复用建议

- 把“主会话只追加结构化摘要”写成团队规范，进 code review checklist。
- 隔离分三档：**fully isolated / read-only shared / shared scratch**，按任务性质选档，不要一刀切。
- 监控每个子 agent 的 token 消耗和回传体积，异常膨胀就是隔离失效的第一个信号。

## 总结

session 隔离的本质不是某个技术开关，而是边界的显式化：子 agent 和主会话之间进什么、出什么、共享什么，都应该写成声明式的契约。把 spawn 隔离、回传 schema、权限收敛这三件事钉死之后，主会话的干净程度是可以长期稳定保持的——这也是 OpenClaw 编排规模化跑起来的前提。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/c6f400326ec76cc5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/9e8e4f1356eece3a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/d0e2c9598ba4a7ad.png)

