---
title: Agent 记忆系统设计：把“记住偏好”做成可维护的三层结构
feedId: 34242
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw/Agent 实践里，经常遇到：用户说“我不用 emoji”“输出不要客套话”，同一会话内有效，一开新会话又恢复原样。把对话全量存下来再塞回 prompt，会让上下文膨胀，Agent 指令遵循变差。记忆系统的目标不是存更多，而是在正确时机给正确信息。

## 问题

常见的错误做法：把 memory 当聊天记录备份；把每一条偏好都写进 system prompt；自动从对话抽取偏好但从不确认；没有过期和冲突处理。结果要么记不住，要么记太多导致混乱。

## 做法/步骤

### 1. 分清层级

建议起步只做三层：

- **profile**：稳定身份与核心偏好，会话启动时注入。
- **preferences**：可变的偏好键值，写入/覆盖/删除。
- **facts**：用户说过的事实性信息，按需检索。

短期会话内工作记忆继续交给上下文，不作长期持久化。

### 2. 定义最小 schema

使用 JSON 文件起步，不必上向量库。每条记忆带几个字段：

```json
{
  "profile": {
    "name": "alex",
    "timezone": "Asia/Shanghai",
    "language": "zh-CN"
  },
  "preferences": [
    {
      "id": "p1",
      "type": "tone",
      "value": "no_emoji",
      "scope": "all",
      "source": "user_correction",
      "confidence": 1.0,
      "updated_at": "2025-01-01T10:00:00Z"
    }
  ],
  "facts": []
}
```

字段不要贪多，够定位一条记忆即可。

### 3. 写入策略

只写入三类：

- 用户显式纠正或确认，例如“以后都别用 emoji”。
- 稳定事实，例如“我在用 macOS”。
- 任务结束后用户确认的结论，例如“这个报告模板可以保留”。

不要每轮自动抽取偏好。一次性的要求容易被误判为永久偏好。可以设置触发词：`记住`、`以后都`、`不要`、`总是`，但仍需在该条记录上标 `confidence`。自动推断的偏好最好 `confidence < 1`，下次会话前再确认。

### 4. 读取策略

会话开始时，只注入 profile 和少量高置信 preference，渲染成短文本，例如：

```text
用户偏好：输出简洁；不使用 emoji；遇到不确定时直接说明。
```

不要把所有 memory 都注入。需要更具体记忆时，通过 MCP 工具 `recall` 做关键词检索。这样避免 context 污染。

### 5. 更新与冲突

同 type 同 scope 的新记录覆盖旧记录，并更新 `updated_at`。支持 `/forget` 命令，把记录 `archived: true` 而不是物理删除，方便回滚。每次写入保留 `source`，排障时能知道是用户纠正还是模型推断。

## 在 OpenClaw 里的集成方式

比较稳的做法是把记忆做成 MCP memory server，暴露三个工具：

- `remember(type, value, scope, source)`
- `recall(query, limit)`
- `forget(id)`

Agent 在判断用户有长期偏好时调用 `remember`；会话启动插件读取 memory.json 生成 profile 注入 system prompt；不确定时调用 `recall`。示例流程：

```text
[写入] 用户：“以后不要用 emoji”
Agent -> memory.remember(type="tone", value="no_emoji", scope="all", source="user_correction")

[读取] 下次会话启动 ->
插件读取 memory.json ->
生成 profile 文本 ->
拼进 system prompt

[冲突] 用户：“这次报告可以用 emoji 标注”
Agent -> 识别为临时指令，不写入长期记忆

[删除] 用户：“/forget no_emoji”
Agent -> 标记 archived，下次不再注入
```

这样记忆系统不会影响单次任务的灵活性。

## 踩坑点

- **偏好过细**：system prompt 里塞满“不要用感叹号、每段不超过三行、优先用表格”，会互相冲突，Agent 反而不知道该听哪条。
- **自动抽取假阳性**：把一次性的“这次用英文”当成永久偏好。必须加显式确认或低 confidence 二次确认。
- **记忆过期**：用户工作流变了，旧偏好仍然生效。定期检查 `updated_at` 并清理。
- **隐私**：偏好文件可能包含个人信息，尽量本地存储，不要直接上传给第三方模型。
- **上下文膨胀**：全量 memory 注入会导致工具调用质量下降，token 成本上升。

## 可复用建议

- 先维护 5-10 条高价值偏好，不要一上来做复杂记忆图谱。
- 记忆读写做成 MCP 工具，而不是把整个文件塞进 prompt。
- 提供 `/memory` 和 `/forget` 命令，让用户可查看、可删除。
- 用 JSON 文件起步，启动加载、更新写回即可；等检索需求变复杂再上 SQLite 或向量库。
- 每条记忆保留来源和更新时间，方便排障。

## 总结

Agent 记忆不是“无限上下文”，而是可维护的小型偏好库。分层、控制写入、显式确认、可删除，比不断堆 prompt 更实际。在 OpenClaw 里，用 MCP 把记忆做成工具，可以保持系统稳定，也能让用户在自动化流程中真正感受到“它记住了我”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/2d747ce16cc7fbcf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/e4682843911d25bd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/41b865f6d1a9a923.png)

