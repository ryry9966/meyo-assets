---
title: AI Agent 权限边界：把“该问谁”做成策略，而不是靠 prompt 自觉
feedId: 34454
source: 综合讨论
publishedAt: 2026-08-24
---

在 OpenClaw 里接入 MCP 工具后，权限问题会很快从“能不能调”变成“该不该自动调”。尤其是文件写入、数据库执行、命令运行这类工具，一旦把决策全交给 Agent，等于把爆炸半径交给一个会被长上下文、工具描述和检索结果影响的系统。

问题不是模型听不听话，而是我们很少在工具层定义清楚：什么可以自己干，什么必须问人类。

## 背景：工具变多以后，边界开始模糊

OpenClaw / Agent / MCP 这类组合的常见状态是：

- 一些只读工具被设成自动执行；
- 一些写操作靠系统提示词里写“不要直接删除”；
- 少量危险操作弹确认；
- 很多中间地带没有规则，模型有时候问，有时候直接干。

结果是要么每条都确认，人类变成审批机器人；要么模型一次性跑了高风险操作，事后才看到日志。

## 问题：缺少可执行的权限分级

仅靠 prompt 约束不稳定的原因是：模型会受上下文长度、工具描述、示例和用户措辞影响。今天它在长会话里可能忽略“不要执行 DROP”，明天换个工具描述又会误判。

更合理的是把权限决策外置：不依赖模型自我约束，而是在工具调用链路上加一层策略。

## 做法：把“该问谁”拆成 4 层

### 1. 给每个工具打风险标签

不要只写 `allow` / `deny`。至少区分：

- `read`：只读、可自动；
- `write`：写入/状态变更，测试环境可自动，生产环境需确认；
- `destructive`：删除、覆盖、权限变更、迁移，默认必须确认；
- `exfil`：外发数据、访问外网、读取高敏资源，默认拒绝或严格白名单。

可以在 OpenClaw 的工具描述或注册配置里增加字段：

```yaml
execute_sql:
  risk: destructive
  default: confirm
  dry_run: false
  require_rollback_plan: true

read_file:
  risk: read
  default: auto
  max_bytes: 20000

send_http:
  risk: exfil
  default: deny
  allow_hosts: []
```

这样策略判断不用猜工具名，直接读风险等级。

### 2. 在 MCP 客户端与 server 之间加策略门

如果 MCP server 本身暴露得太粗，就在 Agent 侧包一层 policy gate。不用改 server，只需要在工具调用前检查：

```text
tool call -> risk check -> resource tag check -> dry-run check -> auto / confirm / deny
```

例如：

```text
execute_sql(risk=destructive, target=prod_db)
  -> 无 dry_run
  -> 需要人工确认
```

`write_file` 到 `/tmp` 测试目录可以自动；同样工具写到 `/etc` 或生产配置目录就必须确认。判断依据是“工具 + 目标资源 + 动作影响面”，而不是模型说自己有多少信心。

### 3. 人工确认要带上下文，不要只问“是否执行”

频繁确认会疲劳，疲劳之后人会随手点同意。所以确认卡片必须一次给够信息：

- 将执行什么动作；
- 影响范围：多少条数据、哪个目录、哪个服务；
- 回滚方案：有无快照、备份、上一版本；
- 有效期：批准后 10 分钟内有效；
- 默认拒绝：超时未处理不执行。

例如：

```text
将执行：DELETE FROM logs WHERE created_at < '2025-01-01'
预计影响：约 12 万行
回滚：从每日快照 logs_20250101 恢复
有效期：10 分钟
默认：拒绝
```

这比单纯问“是否执行 SQL？”可靠得多。

### 4. 提权要有范围和时效

不要在会话里说一次“允许写生产库”就永久放开。批准时授予一个临时 scope，例如：

```text
允许 tool=execute_sql
目标=prod_db
动作=DELETE
有效期=10 分钟
```

超出范围、超时、换目标，都需要重新确认。

### 5. 记录决策日志，定期复盘

每次 `auto` / `confirm` / `deny` 都记一条：

```text
tool, risk, target, decision, reason, timestamp
```

每周看两类：

- 误放行：模型自动跑了高风险动作；
- 误拒绝：大量确认其实可以降级为自动。

根据日志调整阈值，而不是靠感觉收紧或放松。

## 踩坑点

- **把 read 当无风险**：读文件、读数据库可能把敏感信息带进上下文，再通过另一个工具外发。只读工单系统不等于不会泄露。
- **确认疲劳**：确认太多之后，人会把“同意”当成机械动作。不要对低风险写入也做确认。
- **单工具安全，组合起来危险**：先读文件，再通过 HTTP 工具发送，单看每个工具可能都不越权。策略要能识别组合风险，至少限制高敏数据的后续去向。
- **忽略 MCP server 本身的系统权限**：一个 MCP server 如果拥有 shell 或宽泛文件权限，工具层再分级也拦不住。应该限制 server 的运行账号、目录和网络出站。
- **只靠模型自我报告置信度**：模型说“置信度 0.9”不代表安全。策略应基于工具风险、资源标签、参数结构，而不是模型的自我评价。

## 可复用建议

- 默认拒绝，最小权限。新接入 MCP 工具先挂 `read-only` 或 `dry-run` 跑几天。
- 把策略写成配置文件并版本化，放在 git 里，不要散落在系统提示词中。
- 对资源打标签，策略引用 `env=prod`、`data_class=internal`，而不是维护一长串工具名。
- 确认只留给不可逆、高影响、低把握的操作。
- 人工批准的内容要包含作用域和有效期，不提供“本次会话永久允许”。

## 总结

AI Agent 的权限边界，本质上不是“信不信任模型”，而是把工具的风险、影响、回滚路径提前标清楚，让人类只在不可逆和低置信时介入。对 OpenClaw 这类快速接入 MCP 的环境来说，这比继续加系统提示词更可靠，也更可维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/1670b7374ac8e5b1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/c68defc7fadec943.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/a3d1288cb39a338d.png)

