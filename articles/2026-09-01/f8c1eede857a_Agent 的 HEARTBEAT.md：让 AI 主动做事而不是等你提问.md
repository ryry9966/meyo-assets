---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 35623
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

在 OpenClaw 的 agent 工作流里，大部分场景仍然是“你问一句，它动一下”。这种方式适合探索性任务，但放到持续维护类的场景里就很别扭：比如每天检查备份是否成功、依赖库有没有安全更新、某个目录下是否出现异常文件、服务健康状态是否正常。这类事如果只靠用户想起来再问，漏掉是迟早的事。

有人可能会说：直接用 cron 跑脚本不就行了？确实可以，但脚本只能做机械判断，遇到“看起来不太对但需要人工确认”的情况就无能为力。Agent 的优势在于能理解上下文、能读日志、能调用 MCP 工具、能用自然语言汇报。问题只是：怎么让 agent 在没有人提问的时候，主动去做这些事。

## 问题

直接对 agent 说“每天帮我检查一下备份”，效果通常很差。原因是：

- 指令太模糊，agent 每次执行的标准不一样；
- 没有固定的检查清单，容易漏项；
- 执行过程没有被记录，出了问题没法回溯；
- 主动行为的触发方式不明确，agent 不会自己“醒来”。

我们需要一个轻量、可版本化、人类可读的“主动任务契约”，让 agent 知道什么时间该做什么、做到什么程度算正常、异常时怎么汇报。在 OpenClaw 的 agent 实践中，我们把它叫做 `HEARTBEAT.md`。

## 做法

### 1. 创建 HEARTBEAT.md

在工作区根目录放一个 `HEARTBEAT.md`，内容用自然语言写清楚四件事：**频率、检查项、判定标准、输出方式**。

示例结构：

```markdown
# Heartbeat 任务

## 频率
每 6 小时执行一次。

## 检查项
1. 运行 `backup/check_status.sh`，确认最近一次备份退出码为 0。
2. 检查 `data/incoming/` 目录下是否有超过 1 小时未处理的文件。
3. 调用 MCP 工具 `security:list_deps`，确认没有高危 CVE。

## 判定标准
- 备份失败：退出码非 0。
- 积压异常：存在超过 1 小时的文件。
- 安全异常：存在 high 或 critical 级别的 CVE。

## 输出
异常时发送消息到 `#ops-alerts`，正常时只追加一行到 `heartbeat.log`。
```

这份文件就是 agent 的“值班说明”。它不需要很复杂，复杂了反而难维护。

### 2. 配置定时触发

在 OpenClaw 中，可以通过外部 cron 调用 CLI 向指定会话发送一条固定消息来实现定时唤醒，例如：

```bash
0 */6 * * * openclaw send --session ops --message "请阅读 ./HEARTBEAT.md 并执行本轮的 heartbeat 任务"
```

如果你的 OpenClaw 环境接入了 scheduler 类型的 MCP server，也可以把触发逻辑交给 MCP 管理。关键是触发消息要固定、简短，把执行细节都交给 `HEARTBEAT.md`，而不是每次都重写提示词。

### 3. 让 agent 执行并记录

Agent 收到触发消息后，会读取 `HEARTBEAT.md`，按检查项逐个执行。执行过程中建议把关键步骤追加写入 `heartbeat.log`，方便之后审计。正常情况只写一行摘要，异常时才主动通知。

```bash
echo "$(date -Iseconds) heartbeat finished: 3 checks passed" >> heartbeat.log
```

### 4. 处理异常

异常时 agent 会根据 `HEARTBEAT.md` 里的“输出”规则主动汇报。汇报内容应包括：检查项名称、实际结果、判定依据、建议动作。不要只发一句“备份失败了”，要把上下文带出来。

## 踩坑点

实际跑了一段时间，有几个坑比较典型：

- **任务太重**：heartbeat 应该是轻量检查，重活交给专门的流程。如果每次 heartbeat 都要拉全量数据、跑复杂分析，执行时间会失控，也容易超时。
- **不幂等**：定时触发可能因为网络超时、agent 重试等原因导致同一轮 heartbeat 被重复执行。检查操作不能有副作用，比如不能直接删除文件、不能重复发送通知。需要加锁或者用“最后执行时间”做去重。
- **agent 自己改 HEARTBEAT.md**：如果 agent 有写文件权限，它可能会在调试过程中修改 `HEARTBEAT.md`，导致行为漂移。建议在触发提示词里明确禁止修改该文件，或者用文件系统权限限制。
- **并发触发**：多个 cron 任务叠加时，可能多个 agent 同时跑同一个 heartbeat。可以用一个简单的 lock 文件来避免并发。
- **敏感信息泄漏**：heartbeat 可能访问数据库、API 凭证等。不要把这些信息写在 `HEARTBEAT.md` 里，使用环境变量或 secrets 管理。

## 可复用建议

- **保持短小**：`HEARTBEAT.md` 不要超过一屏，检查项控制在 3～6 个。
- **版本控制**：把 `HEARTBEAT.md` 纳入 git 管理，修改有记录，回滚方便。
- **先 dry-run**：新增 heartbeat 任务时，先手动触发一次，确认检查逻辑和输出格式符合预期，再接入定时。
- **日志分离**：heartbeat 执行日志和 agent 的对话日志分开存，避免混在一起难查。
- **按场景拆分**：如果检查项太多，可以拆成多个文件，比如 `heartbeat-infra.md`、`heartbeat-security.md`，按不同频率触发。

## 总结

`HEARTBEAT.md` 不是个复杂的技术方案，它更像一种工程约定：把“想让 agent 持续关注的事”写下来，让 agent 有节奏地执行。在 OpenClaw 的生态里，配合定时触发和 MCP 工具，这份文件能把 agent 从被动问答工具变成主动值班员。改动成本低，收益却很实在——很多运维类的遗漏，就是从“没人想起来看”开始的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/2288eaa34c33cef4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/2ab91531c86a5a98.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/1431735545e63dc5.png)

