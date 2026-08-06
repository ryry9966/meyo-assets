---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 31929
source: 综合讨论
publishedAt: 2026-08-07
---

# OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份

## 背景：当 Agent 活得比你预期的长

用 OpenClaw 跑过 7x24 小时 Agent 的人都会遇到同一个问题：它的行为第一天还很乖，一周后就开始“自由发挥”。不是因为模型变差，而是**上下文窗口里已经找不到最初的设定**。长期运行的 Agent，尤其是接了 MCP server、可以调外部工具的，最容易出现**身份漂移**——它会逐渐忘掉自己是运维助手，开始以“通用助手”的方式回答琐碎问题，甚至绕过你定好的安全约束。

OpenClaw 提供了一套轻量但有效的解决方案：用 `IDENTITY.md` 作为 Agent 的**可进化身份文件**。它不只存初始 prompt，更是一个被你的工作流反复读写、随着实践修正的记忆锚点。

## 问题：静态 prompt 的三大硬伤

1. **版本腐化**：Prompt 写在代码里或环境变量中，改一次要重启部署，于是大家选择不改，行为越来越僵；
2. **知识无法沉淀**：Agent 执行任务时学到“这种日志格式代表 OOM”“这个 MCP tool 的 timeout 设到 15s 才稳”，这些经验没法自动回到身份定义中；
3. **多实例不一致**：同一份 Agent 部署在 dev/staging/prod，却用着同一段硬编码指令，没有任何环境感知。

## 做法：把 `IDENTITY.md` 变成活的配置层

### 1. 初始模板：三明治结构

别把所有东西塞进一个字段。让 Agent 自己也能读懂的结构最好：

```markdown
# IDENTITY
role: Site Reliability Engineer
constraints:
  - 只操作 k8s 资源和日志查询
  - 禁止执行 destructive 命令
  - 所有操作前输出 reasoning
goals:
  - 诊断 5xx 错误
  - 生成可读事故报告

# KNOWLEDGE
- data/runbooks/*.md 是团队沉淀的处置手册
- 日志中 "slow_sql" 关键词代表需要 DBA 介入

# TOOLS (MCP)
prometheus-mcp: 查询 PromQL, 默认步长 5m
k8s-readonly: 仅 get/describe/logs
slack-mcp: 发送消息到 #incidents
```

`IDENTITY.md` 不仅是 prompt，更是**元能力清单**。OpenClaw 在启动时会解析它，把 `IDENTITY` 部分注入系统消息，`TOOLS` 部分决定哪些 MCP server 实际被暴露给 Agent。

### 2. 让身份进化：从日志回写

只靠人工改 `IDENTITY.md` 不够。我们做了个小而实用的机制：Agent 的每轮结论都附录一段 `delta_identity`（可选），格式固定：

```json
{
  "add_knowledge": "elasticsearch-prod 的 timeout 至少 30s",
  "deprecate": "之前的 nginx 错误码映射已更新"
}
```

一个 20 行的 `sync_identity.py` 扫描最近 24 小时的任务日志，提取 `delta_identity`，如果三条以上日志都提到了同一个知识点，就自动 append 到 `KNOWLEDGE` 区域，并加上时间戳。这样 Agent 自己用出来的经验会“长”回身份里。

### 3. 版本化与环境感知

将 `IDENTITY.md` 纳入 Git。不同分支对应不同环境，dev 分支允许更高风险操作，prod 分支收紧权限。部署流水线在渲染 Agent 配置时，用 `envsubst` 替换 `$ENVIRONMENT` 变量，把当前环境的事实塞进去。多实例一致性靠**由同一份 Git tag 驱动**，而不是靠 Ops 手改。

## 踩坑点

### 1. 自动回写会制造“知识膨胀”

不设上限的话，`KNOWLEDGE` 段会在两周内变成一锅粥。我们定了两个硬规则：

- **冷却计数**：同一知识点必须被 3 条以上不相关的任务日志提及，才进入待写入列表；
- **最大行数**：`KNOWLEDGE` 超过 50 行时，触发人工 review 或自动摘要（用模型把多条知识合并成一句话）。

### 2. MCP tool 名称冲突

如果两个 MCP server 提供同名 tool（例如都有 `list_containers`），OpenClaw 会按加载顺序覆盖。**不要在 `IDENTITY.md` 里只写 server 名，必须映射出实际使用的 tool 名和调用建议**，否则 Agent 不知道该选哪个。我们习惯用下面这种分栏式写法：

```
prometheus-mcp: 
  - instant_query(query, time)
  - ...
k8s-readonly:
  - get_pod_logs(namespace, pod, container)
```

这也会被注入到工具选择提示中，减少困惑。

### 3. 身份与“动态记忆”混淆

`IDENTITY.md` 应该只放**稳定的身份、约束与核心知识**。会话级的临时记忆（比如“用户刚才说了要优化那个接口”）请用 OpenClaw 的 short-term memory 后端（Redis/PostgreSQL snippet），不要写入 identity 文件，否则重启后 Agent 会以为那是永恒真理。

## 可复用建议

- **周期审查**：每两周做一次 identity review，清理过时的约束（比如早已不存在的内部端口号），否则 Agent 会因“死规则”而反复给自己设限。
- **最小权限 + 身份继承**：如果你有几个不同的 Agent（运维、安全、数据），可以用一个 base identity 文件，只 override 差异部分。我们通过 OpenClaw 的 `extend_identity` 指令实现，避免了全量粘贴。
- **可观测性**：在 OpenClaw 的回调里面埋一个 hook，每当 Agent 偏离身份约束输出时，记录并告警。我们抓到最隐蔽的一次是：Agent 自作主张把 Slack 通知改成了 “info” 级别，因为它的 `KNOWLEDGE` 里出现了一次“用户曾抱怨太多报警”，自动写入导致约束弱化。

## 总结

`IDENTITY.md` 在 OpenClaw 里扮演的角色，很像 Kubernetes 的 `Deployment` spec——看起来只是一段静态 YAML，但实际上连接了控制器、自动伸缩与健康检查。把 Agent 的身份当成代码来管理，让它基于真实运行反馈持续小步演进，这才是长期跑稳的工程前提。与其反复手修 prompt，不如建好这套“身份 CI”，让 Agent 能带着你的工程约束慢慢长大。

---

