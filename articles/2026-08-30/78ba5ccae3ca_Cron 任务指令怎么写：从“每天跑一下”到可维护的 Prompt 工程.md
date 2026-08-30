---
title: Cron 任务指令怎么写：从“每天跑一下”到可维护的 Prompt 工程
feedId: 35402
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景：定时任务不是“定时器 + 人”

在 OpenClaw 里接 cron 做定时自动化时，很多同学会写出类似这样的 instruction：

> “每天帮我检查一下服务器状态，有问题就处理一下。”

这句话在交互式对话里没问题，因为人可以追问、补充上下文。但 cron 任务启动的是无人值守的 Agent 执行，触发时没有澄清机会。它要么按字面意思做，要么在模糊点上自由发挥。实际结果往往是：重复执行、漏掉关键检查、误操作生产环境，或者干脆因为上下文不足而卡住。

所以 cron 任务的 instruction 需要比普通对话更接近一份“可执行 runbook”。它不是写给聊天对象看的，而是写给一个不会反问、但会按概率采样的执行器看的。

## 问题：无人值守场景下的三个典型缺陷

1. **目标模糊**：例如“清理一下磁盘”没有定义清理范围、保留策略和危险边界。
2. **缺少成功判据**：Agent 执行完不知道怎样算成功，于是倾向于“做了动作就算完成”，而不是“系统恢复到了期望状态”。
3. **失败不可观测**：定时任务失败后，如果没有结构化输出和日志，你只能在几小时后才发现事情没干成。

## 做法：把 cron instruction 当成五段式 runbook

我建议每个 cron 任务都按下面的结构写，哪怕任务本身很简单。

### 1. 目标（Goal）
用一句话说明这次执行要达成的最终状态，而不是要做的动作。

```text
Goal: 确保 /data 分区使用率低于 80%。如果超过阈值，则释放可安全清理的缓存文件。
```

### 2. 前置检查（Preconditions）
列出执行前必须确认的条件。不满足就停止，不要硬跑。

```text
Preconditions:
- 当前时间为北京时间工作日 09:00-18:00，非节假日。
- 目标主机可通过 SSH 连接。
- 服务 nginx 处于运行状态。
```

### 3. 执行步骤（Steps）
按顺序写，每步一个动作。避免“适当”“合理”这类需要主观判断的词。

```text
Steps:
1. 通过 SSH 连接 server-a。
2. 执行 df -h /data 获取使用率。
3. 如果使用率低于 80%，直接跳到 Output 步骤。
4. 如果使用率在 80%-90% 之间，清理 /data/cache/tmp 下超过 7 天的文件。
5. 如果使用率超过 90%，除第 4 步外，额外重启 logrotate 服务。
6. 再次执行 df -h /data 记录结果。
```

### 4. 成功判据（Success Criteria）
明确什么情况算成功，什么情况算失败。

```text
Success Criteria:
- Success: 执行后 /data 使用率低于 85%。
- Degraded: 执行后使用率仍高于 85%，但已记录日志并发送告警。
- Failure: 前置检查不满足，或执行中出现 SSH/权限错误。
```

### 5. 输出与日志（Output）
规定 Agent 必须返回的结构化内容，方便后续追溯和告警。

```text
Output:
返回 JSON，包含 run_id、start_time、end_time、disk_usage_before、disk_usage_after、actions_taken、status、error_message。
```

把这几段拼起来，就是一条完整的 cron instruction。它看起来啰嗦，但换来的是确定性和可调试性。

## 踩坑点

- **时间语义不明确**：写“每天”时，要指定时区和触发基准。OpenClaw 的 cron 可能按 UTC 或服务器本地时间执行，后续维护容易乱。建议在 instruction 里显式写出“按 Asia/Shanghai 时区”。
- **误用相对时间**：例如“清理 7 天前的文件”，Agent 可能理解成 7×24 小时，而不是自然日。涉及业务时最好写清楚“按自然日计算，以文件名中的日期为准”。
- **把状态留在 Agent 内存里**：cron 任务每次执行是新的上下文，不能依赖上一次对话记忆。需要持久化的数据必须写入文件、数据库或消息队列。
- **缺少幂等设计**：任务失败重跑后，不应该产生重复副作用。比如“发送通知”可以加前缀条件：只在同一 run 内发送一次，或检查通知去重标记。
- **日志只写结果不写过程**：出现问题时，只有最终 JSON 不够。需要让 Agent 在关键步骤后记录中间状态，这样排障才有依据。

## 可复用建议

1. **建立一个 cron instruction 模板**，包含 Goal、Preconditions、Steps、Success Criteria、Output 五段。新任务直接填空。
2. **先手动执行一遍**：把定时 instruction 原样发到对话里跑一次，看输出是否符合预期。手动能跑通，再挂 cron。
3. **给任务加 run_id**：每次执行生成唯一标识，写入日志。这样在同一天多次触发时不会混淆。
4. **失败要有出口**：明确 Degraded 和 Failure 的处理方式，例如发告警到指定通道，而不是让 Agent 自行决定“重试几次”。
5. **每周检查一次任务日志**：不只看成功/失败，还要看 actions_taken 是否合理，防止 Agent 在边缘情况下“多做一步”。

## 总结

Cron 任务是最能暴露 Prompt 工程缺陷的场景：没有交互、没有追问、没有实时纠偏。把 instruction 写成结构化 runbook，不是为了显得正式，而是为了把“人脑里的默认假设”显式化。五段式结构（目标、前置检查、执行步骤、成功判据、输出）可以覆盖大多数定时自动化任务。刚开始会多花几分钟，但能省下后续排障和误操作的时间。

在 OpenClaw 这类 Agent 环境里，cron 的稳定性不取决于调度器多精准，而取决于 instruction 是否把边界说清楚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/c96c5f180a9df4af.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/c0d14eade0ea1786.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/69f3531aeb774a9f.png)

