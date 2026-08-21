---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 34007
source: 综合讨论
publishedAt: 2026-08-21
---

很多 OpenClaw 实践里，Agent 都活得像一个“应答机器”：你问一句，它动一下；你不开口，它就永远待着。但在自动化、巡检、插件维护这些场景里，我们更希望 Agent 能自己判断“现在该不该干活”，而不是等人类发指令。

问题是：把“什么时候主动做、做什么、做到什么程度”全部写进 prompt，会变成一坨难以维护的上下文。直接让 Agent 自由发挥，又容易出现误操作、重复报告、动作范围漂移。

我在 OpenClaw 工作区用了一个比较笨但有效的办法：放一个 `HEARTBEAT.md`，把它当作 Agent 的心跳配置。它不是知识库，也不是日志，而是一份“何时醒来、醒来做什么、做完怎么记录”的工程文件。

## 背景：被动 Agent 的局限

很多自动化方案依赖外部 cron 触发固定脚本。但如果触发后只是让 Agent“看看有没有需要处理的”，Agent 很容易给出空洞回复；如果把所有要求都塞进 system prompt，又会造成 token 膨胀，且每次微调动作都要改一大段提示词。

MCP 工具本身能提供能力，但 MCP 不会告诉 Agent 什么时候该调、调完该不该自动执行。这个“判断层”需要单独一个地方承载。

## 做法：把心跳配置独立成文件

`HEARTBEAT.md` 只保留最小信息，例如：

```markdown
agent: ops-agent
cadence: every 4 hours
triggers:
  - uncommitted_changes > 3
  - overdue_todos > 0
  - health_check_fail
actions:
  - name: repo_status
    tool: mcp.git.status
    mode: read_only
  - name: health_check
    tool: mcp.service.health
    mode: report
  - name: auto_commit_notes
    tool: mcp.git.commit
    mode: confirm
    matcher: "*.md"
log: HEARTBEAT.log
```

这里的核心是动作分级：

- `read_only`：只看不改，结果写进心跳日志。
- `report`：发现问题后生成结构化摘要，需要人类查看。
- `confirm`：生成执行计划，进入待确认队列，不直接动手。
- `auto`：只允许做幂等、可回滚、低风险的操作。

默认全部动作最多到 `report` 或 `confirm`，不要一上来给 `auto`。

## 步骤

1. 在 OpenClaw 工作区建 `HEARTBEAT.md`，先只写 3 个触发器和 3 个动作，不要贪多。
2. 把具体能力封装成 MCP 工具。心跳文件只引用工具名，不写完整实现。工具描述里写清楚入参、出参、副作用。
3. 配置外部调度。如果 OpenClaw 没有原生周期任务，就用系统 crontab 或持续集成定时触发 CLI 入口，让 Agent 执行固定命令：读取 `HEARTBEAT.md`、检查触发条件、按动作分级执行、追加日志。
4. 每次心跳输出一条结构化记录到 `HEARTBEAT.log`，包含时间、触发条件、动作名、结果、是否需要人类介入。
5. 每周人工看一眼日志，删除过期触发器和无效动作。

## 踩坑点

**别让 Agent 把 HEARTBEAT.md 当日志写。** 配置文件一旦被 Agent 自己修改，很容易漂移。日志单独放 `HEARTBEAT.log`，心跳文件保持只读或版本锁定。

**别把心跳文件全量灌进 prompt。** 触发器和动作变多后，全量加载会很费 token。心跳文件应保持短小，动作细节放工具描述里，让 Agent 按需取用。

**自动动作要防误触发。** 即使标记为 `auto`，也要加上幂等键、去重条件或最大执行次数。比如“发现同一类异常连续 3 次才执行”，避免凌晨频繁操作。

**日志会污染上下文。** 如果 Agent 每次心跳都读回完整日志，几周后上下文窗口就爆了。日志通常只追加，不回读；需要回顾时，只读最近 N 条摘要。

**重复报告很烦人。** 同一个 trigger 如果没有去重，每 4 小时都会提醒一次。可以给每条日志加一个 `trigger_key`，同一 trigger 在未处理前只报告一次，状态变了再重新报告。

**动作范围会自己扩大。** Agent 有时会“顺手”做一些心跳文件里没有的动作。心跳文件需要版本控制，修改必须走 review；运行入口也最好固定命令，不让 Agent 自己改调度配置。

## 可复用建议

- 把心跳内容拆成三层：触发器、动作、记录。三层分开维护，比一段长 prompt 更清晰。
- 新动作先跑 dry-run，观察 3 到 5 次心跳的误报率，再考虑放开权限。
- 允许 Agent 输出“nothing to do”。空跑也是有效心跳，不要逼它每次都要找出点事。
- 自动动作优先选择幂等操作，比如生成报告、清理过期缓存、打标签；涉及数据修改默认走 `confirm`。
- 心跳频率别太高。先从一个保守频率开始，比如每天 2 次，稳定后再调密。

## 总结

`HEARTBEAT.md` 不是为了把 Agent 变得更聪明，而是把“主动做事”从一个黑盒行为，变成一个可评审、可回滚、可观测的工程配置。它让 Agent 的主动性有了边界：什么时候醒、醒了看什么、看完能做什么、做完怎么记。

对 OpenClaw 社区来说，这个文件成本很低，但能把很多“等用户提问”的被动 Agent，往前推一步：它们开始有节奏地干活，而不是一直等下一句话。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/9f7b9eeb44d79cd0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/f7cf7af58c27e653.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/6ccd6062c25ea948.png)

