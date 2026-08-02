---
title: Prompt 工程实战：Cron 任务应该怎么写 instruction
feedId: 31392
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景

在 OpenClaw、Agent、MCP 这类自动化系统里，很多“定时触发”最终都要落到 cron 上。  
不管你是用插件内置的 cron 调度器，还是让 agent 自己维护一份 crontab，一件尴尬的事总在发生：**instruction 写得随意，任务就跑得随意**。

常见现象：让 agent “每天早上帮我汇总一下重要邮件”，它顺手生成 `0 0 * * *`，实际却是 UTC 零点，根本不是你想要的北京时间。或者任务权重不清，两条 cron 撞在一起时，agent 直接放弃执行。再或者，一次失败后没有补偿，第二天才被发现少跑了一天的数据。

问题核心在于：cron 任务 instruction 被当成了“一句话需求”，而不是一份可验证的工程配置。

## 问题拆解

从工程视角看，一个可靠的 cron 任务 instruction 至少要回答清楚以下 4 个维度：

1. **时间规范** — 表达式、时区、是否允许重叠、容忍延迟范围  
2. **任务上下文** — 输入源、输出位置、超时、失败定义  
3. **可观测性** — 执行日志写到哪里、成功/失败的判据  
4. **幂等与补偿** — 重复执行是否安全、遗漏后怎么补救

自然语言的模糊性会破坏这 4 点。比如“每天上午发一条日报”，agent 可能把“上午”解析成 8 点到 12 点之间的某个随机时间；但你要的其实是严格 `0 9 * * *`。  
所以 instruction 必须**消除歧义，走向结构化**。

## 实战步骤

### 1. 直接给定 Cron 表达式，禁止 agent 自行推测时间

不要写“每隔两小时执行一次”，而是把表达式、时区、解释全部写死：

```text
你必须使用以下 cron 表达式定时触发任务：
表达式：0 */2 * * *
时区：Asia/Shanghai
含义：从 00:00 开始，每隔 2 小时执行一次（北京时间）
若当前系统 cron 不支持秒字段，严格使用 5 字段 POSIX 标准。
```

如果你的场景是让 agent 生成 cron，要求在输出中同时给出表达式和时区，并提供 3 次 dry-run 校验时间点。

### 2. 用结构化字段定义任务上下文

在 instruction 中显式列出执行单元，参考 OpenClaw 的插件配置风格：

```yaml
task:
  id: "daily_report_0800"
  trigger: "0 8 * * *"
  timezone: "Asia/Shanghai"
  description: "每日 08:00 生成前一日运营简报"
  input:
    source: "database::metrics_daily"
    range: "yesterday"
  output:
    type: "markdown"
    dest: "notion://daily-report"
    filename: "report_{{date-1}}.md"
  timeout_sec: 300
  retry:
    max_attempts: 3
    backoff: "exponential"
    on_final_failure: "log_and_alert"
  overlap: "skip"   # 上一轮未完成时跳过本轮，防止积压
```

这种写法让 agent 不猜测任何执行细节。你甚至可以把它作为**系统 prompt 的追加片段**，直接喂给 OpenClaw 的调度模块。

### 3. 防重叠、防超时、防静默失败

踩坑最多的三个点：

- **任务重叠**：一个 cron 周期没跑完下次又触发，导致资源竞争或重复计算。  
  在 instruction 里明确策略：`skip`（跳过）、`queue`（排队但可能积压）、`terminate_previous`（终止上一轮）。  
- **无超时限制**：网络波动或上游慢查询会让进程挂起数小时。  
  给 agent 设定绝对超时，并在超时后当作失败处理，走重试逻辑。  
- **静默失败**：没有显式要求输出“成功标志”，agent 可能打印一句 `done.` 就退出，但你无法判断数据是否完整。  
  要求在日志末尾输出确定性摘要，例如 `“SUCCESS: 处理 183 行，写入 report_2025-03-16.md，耗时 12s”`。

### 4. 先 dry-run，再上线

在正式挂 cron 前，强制 agent 输出接下来 5 次的预计执行时间，再和你预期对比：

```text
Run a dry-run: generate the next 5 execution times based on the above cron expression 
and timezone. Print them in ISO-8601 with timezone offset.
```

这会暴露时区偏差、夏令时跳变等问题。

## 踩坑实录

- **时区没写死，agent 用 UTC**  
  解决办法：所有 cron instruction 第一行就声明 `默认时区 = Asia/Shanghai`，并让 agent 在每次输出时间时都带上时区后缀。

- **夏令时导致同一时间不存在或重复**  
  如果你的 cron 落在凌晨 2:30，部分地区的夏令时会让这天不存在 2:30，或出现两次。  
  明确指令：若目标时间不存在，推迟到最近的有效时间（如 `2:30 -> 3:00`）；若重复，只执行第一次。

- **cron 字段数不一致**  
  有些工具支持 6 字段（含秒），多数 Linux 是 5 字段。在 instruction 里指定 `standard POSIX format with 5 fields`，并拒绝任何非标准扩展。

- **遗漏补偿机制缺失**  
  如果服务重启或暂停期间错过了几次 cron 触发，instruction 没让 agent 补跑，造成数据断层。  
  加上 `missed execution policy: catch-up only for last missed window` 或 `skip all missed`，随业务场景而定。

## 可复用的建议

1. **建立 cron instruction 模板**  
   在团队内部形成固定的 yaml/json 模板，直接给 agent 解析，不用每次都自然语言重新描述。

2. **要求 agent 输出自检清单**  
   每次接到 cron 指令后，agent 先输出一份 summary（时区、表达式解释、重叠策略、重试规则），和你确认后再挂载。

3. **用 MCP 工具做 cron 审核**  
   如果你的 OpenClaw 环境通过 MCP 暴露了系统时间、crontab 校验工具，可以在 instruction 里强制 agent 调用它们验证表达式合法性、时区正确性。

4. **分离“做什么”和“何时做”**  
   把任务执行逻辑写成独立的 skill/prompt，cron instruction 只关心触发条件，并引用这个 skill。这样修改执行逻辑不影响调度，反之亦然。

## 总结

给 agent 写 cron 任务 instruction，本质是写一份**确定性的工程配置**，而不是随口一句需求。  
你需要像定义 API 一样定义：时区、表达式、上下文、超时、重试、幂等保证。  
当 instruction 足够结构化，agent 的行为就从“猜”变成“执行”——这才是自动化该有的样子。

---

