---
title: Proactive Agent 落地笔记：让助手在告警前先动手
feedId: 35444
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

大多数 Agent 实践仍然停留在“你问一句、它答一句”的被动模式。用户发起指令，模型调用工具，返回结果。这种模式在开发运维场景里效率不高：很多问题都有明确先兆。比如磁盘使用率超过 85%、CI 连续两次失败、依赖库发布安全公告。等用户发现并开口时，问题往往已经扩大。

更实用的形态是 proactive：助手按固定频率或事件触发巡检，发现异常后先做初步诊断，再推送给用户确认。它不等你开口，但也不越权执行危险操作。

本文基于 OpenClaw 的 Agent 编排能力，结合 MCP 工具接口，记录一次 CI 失败场景下的 proactive 落地过程。

## 问题

主动做事最大的难点不是“能不能调用工具”，而是边界与可靠性：

- 误报会快速消耗信任；
- 写操作一旦自动执行，可能引入比原问题更大的故障；
- 日志体积大，容易撑爆上下文窗口；
- 通知过多会变成另一种噪音。

所以 proactive 不能是“让模型自由发挥”，而是一条结构化、分层、可回滚的检测-决策-执行链路。

## 做法/步骤

以一个 GitHub Actions 持续失败监控为例。仓库 main 分支的 CI 如果连续失败，助手需要主动收集失败日志、分析最近 commit diff、生成摘要并建议下一步动作。

### 1. 事件源接入

写一个 MCP server，暴露两个工具：`get_failed_runs` 读取失败记录，`get_job_logs` 按 job ID 拉取日志。OpenClaw 通过 MCP 协议注册这些工具。触发方式可以选择轮询或 webhook，首选 webhook 推送到本地队列，避免频繁轮询导致 API 限流。

### 2. 配置 Proactive Policy

在 OpenClaw 里可以定义结构化策略，而不是把控制逻辑写死在提示词里。示例如下：

```yaml
proactive_policies:
  - name: ci_failure_watch
    trigger:
      source: github_actions
      condition: "consecutive_failures >= 2 and branch == 'main'"
      cooldown: 30m
    actions:
      - type: read
        tool: get_failed_job_logs
        params:
          max_lines: 50
      - type: propose
        template: "ci_issue_summary"
    auto_approve: false
```

策略触发后，Agent 只做两件事：拉取关键日志、按模板生成摘要。不会主动创建 issue 或回滚。

### 3. 分层动作

把 proactive 动作分成两层：

- **Level 1 只读**：查日志、分析 diff、生成诊断摘要、发送通知。
- **Level 2 写操作**：创建 issue、标记 commit、触发回滚、清理资源。

Level 2 默认不自动执行，必须经过人工确认。确认信息进入 audit log，记录谁在什么时间批准了什么动作。

### 4. 反馈闭环

每次 proactive 任务生成一个 `run_id`，记录触发条件、模型决策、工具调用、最终结果。用户回复“确认回滚”后，Agent 执行白名单内的写操作，并继续监控后续状态。

## 踩坑点

**API 限流**：GitHub Actions 接口对未认证请求限流较严。轮询间隔不要低于 5 分钟，或者改成 webhook 推送。本地做一份运行状态缓存，减少重复请求。

**上下文窗口被日志撑爆**：失败日志动辄几 MB，直接塞给模型极易超 token。第一轮只取每个 step 的时间和最后 50 行，必要时用 `grep` 过滤错误关键字，而不是全文导入。

**通知风暴**：偶发失败或基础设施抖动会反复触发 proactive。加冷却时间和事件指纹去重。例如对 `workflow + commit_sha` 做 hash，相同指纹 24 小时内不重复处理。

**写操作翻车**：即使模型判断“应该回滚”，也不要让它直接执行。默认 `auto_approve: false`，写操作统一走审批。审批记录写入 audit log，方便事后追溯。

**模型把相关当因果**：一次失败和上一次失败可能无关，但 Agent 容易脑补出关联。输入里必须明确给时间线、commit diff、环境变量，减少自由发挥空间。

## 可复用建议

1. 从低频可回滚的场景开始，例如创建 issue、发送通知、打标签，而不是直接重启服务。
2. 把 proactive 链路拆成 `detect -> investigate -> propose -> act`，每层用不同工具完成，方便单独测试和替换。
3. 所有主动动作都记录为事件，包含触发原因、工具参数、模型摘要、审批状态。
4. 设置全局 kill switch：一条命令或开关即可暂停所有 proactive 策略，故障时不扩大影响。
5. 用 MCP 统一工具接口，避免把外部 API 逻辑写死在 Agent 内部。

## 总结

Proactive 能力的核心不是“模型自己想做”，而是你把事件源、策略、工具、审批链都搭好后，模型只在决策环节发挥作用。它更适合做“巡检员 + 初诊医生”，而不是“无人值守的操作员”。

守住只读优先、写操作确认、冷却去重这三条底线，这类助手在 CI 监控、安全公告、资源巡检等场景中会非常实用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/3aa3d31308e31709.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/1ff58006638a21bc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/79b85116e21ad1cc.png)

