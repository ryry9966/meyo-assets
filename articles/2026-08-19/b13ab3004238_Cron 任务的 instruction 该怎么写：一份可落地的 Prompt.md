---
title: Cron 任务的 instruction 该怎么写：一份可落地的 Prompt 检查单
feedId: 33846
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景

在 OpenClaw 这类 Agent 自动化环境里，Cron 任务和对话式运行是两种完全不同的模式。对话时可以随时追问、修正、确认；Cron 触发时往往没有人在电脑前。定时任务一旦跑飞，通常不是模型能力不够，而是 instruction 写得像“给人看的备注”，而不是“给程序看的约束”。

我在早期踩过一个典型坑：写了一句“每天早上同步一下 GitHub issues 到 Notion”。第二天打开 Notion，发现多了十几个重复页面，时间还晚了 8 小时。原因不是模型笨，而是两个关键信息缺失：没有指定时区；没有规定“已存在就更新”。

调度器只负责按时唤醒 Agent，任务里所有判断条件都要靠 prompt 明确。下面是一套我目前在用的写法。

## 问题

Cron instruction 最容易在四类地方出问题：

1. 时间与触发环境不明确。
2. 只写目标，不写输出和验收标准。
3. 没有错误处理策略，重试无上限。
4. 忽略幂等性，重复执行产生副作用。

这些问题在交互式对话里可以被人类现场兜底，但在 Cron 任务里会直接变成事故。

## 做法 / 步骤

我会把一个 Cron instruction 拆成七块：角色、触发环境、输入源、输出目标、验收标准、错误处理、副作用约束。

下面是一个可直接修改使用的模板：

```text
role: 定时同步助手，只做 GitHub issue 到 Notion 的单向同步。

trigger_context: 当前时间由系统传入，时区固定为 Asia/Shanghai。
input: 从 ./cache/issues.json 读取；如果文件不存在，先调用
       github_mcp.list_issues(repo="org/repo", since=24h)。

output: 追加写入 ./logs/sync.jsonl，每行 JSON 包含：
        run_id, timestamp, status, issue_id, notion_page_id。

rules:
1. 只处理最近 24 小时更新的 issue。
2. 以 issue_id 作为唯一键：Notion 中已有对应页面时必须更新，禁止新建。
3. GitHub API 限流时等待 120 秒重试一次，仍失败则 status=failed 并停止。
4. 禁止删除、归档或修改与本次 issue 无关的 Notion 页面。

success criteria: 所有目标 issue 均已同步且日志 status=ok；
如果出现 failed，必须在 summary 中列出失败 issue_id 和原因。
```

这段 instruction 没有要求模型“聪明”，而是把判断条件固定下来。尤其是“唯一键”和“禁止删除”这两句，能挡住大部分重复执行产生的副作用。

## 踩坑点

**时间表达模糊**  
不要写“早上”或“晚上”，要写清 IANA 时区，例如 `Asia/Shanghai`，并说明“当前时间由系统传入”。不要让模型根据训练记忆猜时区。

**只写目标不写输出**  
定时任务必须留痕。固定 JSONL 输出比自然语言总结好排查得多。没有日志，失败后只能靠猜。

**把重试交给 Agent 但不设上限**  
无上限重试可能打光 API 额度。写清“重试一次，失败停止”，比“如果失败就重试”安全得多。

**上下文过长**  
Cron prompt 不要塞一长串背景说明。历史状态尽量放文件或通过 MCP 工具按需读取。prompt 越长，越容易被截断或注意力稀释。

**忽略幂等**  
凡是会创建资源的任务，都要写唯一键和“存在即更新 / 跳过”。否则每次触发都可能在业务系统里留下重复数据。

**敏感信息硬编码**  
密钥、token 不要写进 Cron instruction。放环境变量或插件配置里，通过工具调用注入。

## 可复用建议

- 把 Cron prompt 模板化，每新增一个定时任务只改 `input/output` 和 `rules`。
- 用 `run_id` 关联每次执行。系统不提供时，可以约定用 `timestamp + task_name` 生成。
- 先手动跑一遍同款 instruction，确认工具调用和输出格式符合预期，再挂到 Cron。
- 把 schedule 和 task 分离：schedule 只负责触发和传参，task 专注业务规则。
- 对关键任务先加 `dry-run` 模式，只输出将要执行的动作，不真正写操作。

## 总结

Cron 任务的 instruction 不是“告诉模型做什么”，而是“在无人值守时替你做决策”。写得越具体、越可验证、越少留给模型自由发挥的空间，跑飞概率越低。稳定的定时 Agent，往往不靠更长的 prompt，而靠更明确的边界。

---

