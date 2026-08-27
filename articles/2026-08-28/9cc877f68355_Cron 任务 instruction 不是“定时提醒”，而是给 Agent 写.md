---
title: Cron 任务 instruction 不是“定时提醒”，而是给 Agent 写操作边界
feedId: 35002
source: 综合讨论
publishedAt: 2026-08-28
---

# 背景：Cron 任务在 Agent 环境里为什么容易跑偏

在 OpenClaw 这类 Agent、MCP、插件自动化环境里，Cron 任务很容易被理解成“定时执行命令”。但如果 Cron 触发的是一个 Agent，实际执行链路会变成：

scheduler 触发 → 组装 instruction → Agent 调 MCP 工具 → 产生文件、请求或副作用。

问题往往不出在“定时”，而出在 instruction 给了 Agent 太多解释空间。

一个典型例子是：任务写“每天早上检查队列并处理异常”。Agent 可能自行决定“处理”包括删除、重命名、重试，甚至在工具权限较宽时把整个目录整理了一遍。另一个典型失败是重复触发：同一时间 Cron 补跑或重试，任务没有幂等键，导致重复写入、重复调用外部 API。

# 问题：把 Cron instruction 当“自然语言备注”

如果 instruction 只有一句目标，Agent 会替你补全未定义的策略：时区、文件范围、失败标准、退出码、工具边界、输出格式。

工程上这不是智能，而是不可控。Cron 任务比交互式任务更脆弱，因为触发时没有人在旁边确认。越聪明的 Agent，越容易在模糊 instruction 下自行发挥。

# 做法：给 Cron instruction 固定结构

建议把每个 Cron 任务的 instruction 写成结构化块，至少包含七部分：

1. **Trigger**：明确 cron 表达式和时区，不要写“每天早上”。  
   示例：`0 9 * * 1-5 (Asia/Shanghai)`

2. **Goal**：一句话说明任务终点，并写清“只读/可写”边界。

3. **Preflight**：列出执行前必须满足的条件，不满足就退出，不让 Agent 自行修复。

4. **Steps**：按顺序列出可验证动作，动词只用具体工具名，如 `list_files`、`read_file`、`write_file`。

5. **Constraints**：工具白名单、禁止操作、数量上限、超时上限。

6. **Idempotency**：定义幂等键和 rerun 策略，避免重复执行产生副作用。

7. **Output**：固定输出 JSON 或 Markdown 结构，方便日志和后续检查。

示例模板如下，可直接改成自己的任务：

```text
[Trigger]
cron: 0 9 * * 1-5 (Asia/Shanghai)

[Goal]
汇总 /data/queue 下近 24h 新增 .json 文件中 status=failed 的记录，
写入 /data/reports/queue-report.md。只读检查，不修改队列文件。

[Preflight]
- /data/queue 存在且可读
- /data/reports 可写
- 如任一失败，退出码 2，不执行后续步骤

[Steps]
1. list_files /data/queue，过滤 mtime > now-24h 且扩展名为 .json
2. 对每个文件 read_file，提取 status=failed 记录
3. 汇总为 JSON：{processed, failed, items: [{file, error_code}]}
4. write_file 写入 /data/reports/queue-report.md
5. 输出一行 JSON：{"status":"ok","processed":N,"failed":M}

[Constraints]
- 只允许 list_files / read_file / write_file
- 禁止删除、移动、重命名队列文件
- 单次最多处理 200 个文件，超出时只处理最近 200 个并标记 truncated=true

[Idempotency]
- 幂等键：报告文件名中的日期。若当天 report 已存在且 status=ok，则跳过。
- 仅当 rerun=force 时覆盖。

[Output]
- 终端输出单行 JSON，不输出多余解释
- 文件固定标题：# Queue Report
```

这个模板的重点不是“写得多”，而是把 Agent 的决策空间压缩到具体执行步骤内。

# 踩坑点

1. **时区不一致**：服务器用 UTC，任务写“每天 9 点”，本地根本看不到对应报告。所有 Cron instruction 必须显式写时区。

2. **使用相对时间词**：`最近`、`过期`、`异常` 都不该出现。写成 `mtime > now-24h`、`status=failed`、`error_code in (E1,E2)`。

3. **没有退出码**：失败和成功都返回 0，后续任务无法判断。应定义：`0=成功，2=preflight 失败，3=部分失败，4=工具权限不足`。

4. **工具权限过宽**：Cron 任务只给最小工具集，不要复用日常 Agent 的全量 toolset。

5. **重复执行**：Cron 重试、补跑、手动触发叠加，没有幂等键会重复发请求。用日期、ID、源文件 hash 做幂等键。

6. **输出不可观测**：Agent 输出一段自然语言总结，无法 grep。固定输出单行 JSON，字段结构化。

7. **没有 dry-run**：新任务直接上线，首次跑就可能写坏数据。强制先 `dry-run=true`，只输出“将要执行的动作”。

# 可复用建议

- 把 Cron 任务当“传感器”而不是“动作器”。默认只读 + 报告，写操作单独拆任务，并加人工确认或二次 guard。
- instruction 里每个步骤都能用“是/否”验证，不能有“适当处理”“必要时清理”等开放词。
- 给每个 Cron 任务固定一个状态文件或状态表，记录 `last_run_at`、`last_status`、`idempotency_key`。
- 新任务先跑 3 次 dry-run，观察 Agent 是否稳定输出相同动作列表。
- 错误处理写清楚：什么情况重试、什么情况跳过、什么情况报警，不要让 Agent 自己决定“重试 3 次”。
- 日志和产品分开：终端输出一行 JSON，文件报告固定路径，方便 MCP 或插件链路采集。

# 总结

Cron 任务的 instruction 不应该是一句自然语言，而应该是 Agent 的“操作边界文件”。把时区、动作、工具、幂等、输出、失败码写清楚，Cron 才能从“定时触发一个不可控 Agent”变成“定时执行一个可验收的自动化单元”。

在 OpenClaw、Agent 与 MCP 链路里，这比继续研究“更聪明的 prompt”更优先。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/92c64ae74e1c6007.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/8657eec0bbf507a7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/f7a1daed525e5f30.png)

