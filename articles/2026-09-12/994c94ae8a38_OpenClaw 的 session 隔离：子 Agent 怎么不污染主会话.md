---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 37167
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

OpenClaw 的主会话是用户交互的默认上下文：聊天记录、系统提示、工具调用结果都堆在里面。当我们通过插件或 MCP 工具拉起子 agent 去跑长任务——批量检索、代码审查、数据清洗——如果子 agent 的中间过程全部回灌主会话，几轮下来上下文就会被工具输出撑爆。

## 问题

我们实际踩到的症状有三个：

1. 主会话 token 占用随任务数线性上涨，几十轮后触发压缩，早期指令被挤掉，模型开始"忘事"；
2. 子 agent 的海量中间输出混进对话历史，主模型注意力被无关内容稀释，出现指令漂移；
3. 回放日志时分不清"哪句话是子 agent 产生的"，排障成本很高。

根因是子 agent 和主会话共享了上下文边界。

## 做法

OpenClaw 当前的 session 模型是：每个 agent 实例绑定独立 session，各自维护上下文，父子之间只通过显式结果通道通信。落地分四步（字段名以你所用版本文档为准）：

1. **spawn 时声明隔离参数**，指定独立 session 与独立工作目录：

```json
{
  "spawn": "research-agent",
  "session": { "isolate": true },
  "workdir": "/tmp/openclaw/run-42",
  "timeout": 600
}
```

2. **收敛工具白名单**。子 agent 只授予完成任务所需的最小工具集，避免它顺手调用主会话级工具产生副作用。
3. **约定结果契约**。子 agent 结束时只回传结构化摘要：结论、关键数据、产物路径。原始日志落盘到自己的 workdir，不进主会话。
4. **主会话侧只留一行回执**，例如"子任务 #42 完成，摘要见附件"，需要细节时再按 run 目录按需读取。

## 踩坑点

- **全量回传 stdout**：子 agent 把命令输出原样塞进结果，一次吃掉几千 token。务必在包装层做截断和摘要。
- **共享工作目录**：两个子 agent 写同一个临时文件互相覆盖，我们排障花了一小时。每个 run 必须独立目录。
- **结果里带本机绝对路径**：换环境即失效。回传时统一转成相对 run 目录的路径。
- **不设 timeout**：子 agent 卡死，session 挂着占资源。超时后要显式清理，不能指望自动回收。
- **子 agent 的角色设定泄漏**：回传摘要里如果带上了它自己的系统提示片段，主模型会被带偏。摘要走固定模板，只填数据字段。

## 可复用建议

- 把"结果契约"固化成团队共享的 JSON schema，所有子 agent 复用同一套字段；
- 定期用 `sessions list` 审计残留 session，配一个定时清理任务；
- 需要人工复核的长任务，让子 agent 把完整轨迹写进 workdir 日志文件，主会话只保留索引，兼顾可追溯与上下文干净。

## 总结

session 隔离的核心不是"把子 agent 关起来"，而是收紧通信面：独立上下文、最小工具集、结构化回传。做到这三点，主会话就能长期保持干净，子 agent 跑得再重，也不会拖垮主对话的质量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/39f893cc0b2272be.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/07d062254e13cab7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/4a02a3346a7c9048.png)

