---
title: Agent 记忆系统设计：怎么让 AI 助手真正记住你的偏好
feedId: 34148
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 里接 Agent 做自动化时，最容易遇到的不是工具调用失败，而是“它总记不住我上回说过的话”。例如你明确过“代码注释用中文”“部署统一走 Docker Compose”“输出不要加 emoji”，结果新开会话后仍然需要重复纠正。

通常做法是把这些偏好写进 system prompt。短期有效，但问题很快出现：偏好越积越多，上下文被静态描述占满；旧偏好不会自动过期；多条偏好冲突时没有裁决机制；Agent 也无法在任务中更新记忆。于是记忆系统退化成了“更长的 prompt”，而不是一个可维护的外部状态。

## 问题

把记忆等同于“塞进上下文”，核心缺陷有两个：

1. **没有持久化层**：会话结束，状态丢失。system prompt 只是每次启动时读一遍，不具备跨会话、跨设备的读写能力。
2. **没有写入与检索机制**：Agent 只能被动接受，不能根据用户行为提炼偏好；上下文被所有信息无差别占用，真正需要的偏好反而被稀释。

更合理的方式，是把记忆作为独立系统：分类型存储、按需检索、允许更新和删除。

## 做法/步骤

### 1. 先做小模型，不要一上来就做全量记忆

个人助手场景建议只保留三类长期信息：

- `preference`：用户偏好，如“输出用中文”“默认 pnpm”
- `fact`：稳定事实，如“项目部署在 NAS 的 /opt/app”
- `procedure`：可复用流程，如“发布前先跑 test 再 build”

会话内的暂存信息不要进长期库，它属于 working memory，应留在当前上下文。

### 2. 用外部存储 + MCP 暴露记忆工具

在 OpenClaw 中挂一个 memory MCP server，用 SQLite 或 JSONL 存储。MCP 工具只做四件事：`remember`、`recall`、`list`、`forget`。配置大致如下：

```json
{
  "mcpServers": {
    "memory": {
      "command": "node",
      "args": ["/path/to/memory-mcp/dist/index.js"],
      "env": {
        "MEMORY_DB": "~/.openclaw/memory.db"
      }
    }
  }
}
```

### 3. schema 尽量小但必须可追溯

推荐字段：

```sql
CREATE TABLE memories (
  id TEXT PRIMARY KEY,
  namespace TEXT NOT NULL,
  type TEXT NOT NULL,
  key TEXT NOT NULL,
  content TEXT NOT NULL,
  source TEXT NOT NULL,
  confidence REAL DEFAULT 1.0,
  updated_at TEXT NOT NULL
);
```

其中 `namespace` 用于区分用户/项目/Agent，避免串味；`source` 区分显式声明还是行为推断；`confidence` 用于冲突处理。

### 4. 写入策略：显式优先，推断只做候选

用户明确说“以后都用 pnpm”时，直接写入并覆盖同 `key` 旧记录。行为推断如“连续三次选择某个模型”，只生成低置信度候选，不要自动覆盖显式偏好。这样用户改主意后，记忆不会变得顽固。

### 5. 检索策略：先精确，后模糊

召回时先按 `namespace + key` 精确匹配；没有结果再走关键词或向量召回 topK。给 Agent 的信号要短，只返回 `key: content (confidence)`，不要把完整事件流水账塞回去。

在 OpenClaw 的新 session 启动时，从 memory MCP 拉取该 namespace 下高置信度偏好，拼成一段简洁 block 注入 system prompt。临时对话、推理过程不要写回。

## 踩坑点

- **自动记录一切**：把每次对话摘要都存成记忆，会导致召回噪音，Agent 会“记住”很多无关内容。长期库只放提炼后的偏好、事实和流程。
- **没有冲突处理**：用户已经改了偏好，但旧记忆仍在，导致行为矛盾。显式更新应直接覆盖，推断记录要保留来源和时间，必要时提示确认。
- **向量库过度复杂**：个人助手先上 SQLite/JSONL + 简单关键词索引，等记忆量上来再引入向量检索。否则维护成本会吃掉收益。
- **多项目串味**：缺少 namespace 隔离，A 项目的部署偏好会污染 B 项目。每个 Agent 或项目应有独立命名空间。
- **工具描述太宽泛**：MCP 工具的 description 不清晰，Agent 调用不稳定。参数要少，返回结构固定，最好给 remember 加一个必填的 `type` 字段。

## 可复用建议

- 只存跨会话复用的信息，临时消息不进库。
- 显式声明 > 行为推断；覆盖 > 追加。
- 给用户提供 list / forget / update 命令，方便纠错。
- memory MCP 可设为默认只读，写入需要用户确认，减少误写。
- 记录 recall 来源和命中路径，排障时能知道为什么某条记忆被召回。
- 定期清理低置信度、长时间未命中的记录，避免记忆库膨胀。

## 总结

可用的 Agent 记忆系统不是更长的 prompt，而是一套有写入、检索、冲突处理和隔离机制的外部记忆。先做偏好类，再逐步扩展事实和流程，会比一步到位做全量记忆更稳。对于 OpenClaw 场景，通过 MCP 把记忆能力工具化，并且把写入门槛控制在“显式优先”，是短期内容易落地、长期可维护的做法。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b32e01c6be168087.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/7af84b0d4e1a2f64.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/716a480174a53777.png)

