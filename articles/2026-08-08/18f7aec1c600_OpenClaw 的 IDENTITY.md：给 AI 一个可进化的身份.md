---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 32057
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景

在基于大模型的 Agent 实践中，一个老生常谈的痛点是如何让 Agent 的“人设”跨会话延续。每次 `docker compose down && up` 或新建对话，就得重新贴一段 system prompt；要是忘了贴，行为一致性立刻崩盘。OpenClaw 作为围绕 MCP 生态构建的 Agent 框架，提供了一个轻量但实用的机制 —— 项目根目录的 `IDENTITY.md`，把 Agent 的身份从一次性提示词变成一个可编辑、可进化、可版本控制的实体。

## 问题

传统做法把角色定义、行为约束硬编码在代码或 `openclaw.yaml` 的 `system` 字段里。这样做有三个弊端：

1. 修改必须重启服务或重新部署，迭代非常别扭。
2. Agent 在交互中积累的偏好、常见问题答案、业务规则无法沉淀下来，下次仍是“新手”。
3. 多用户协作时，每个人维护自己的 system prompt 副本，最终行为差异越来越大，排错成本极高。

IDENTITY.md 的出现，目标就是把身份定义外部化，并允许 Agent 在受控条件下**自我写入**，实现身份的持续进化。

## 做法与步骤

### 1. 创建结构化身份文件
在项目根目录下新建 `IDENTITY.md`，推荐使用 YAML front matter + Markdown 正文的结构，便于解析和分段管理。一个最简模板：

```yaml
---
name: "claw"
role: "后端运维助手"
version: 1.0
last_evolved: 2025-04-01
---
## 核心行为准则
- 优先使用 MCP 工具查询实时信息，禁止编造。
- 回答前确认上下文，不省略必要参数。
- 运维建议需给出回滚方案。

## Learned Experience (AI 可写区域)
<!-- 仅可追加于此处，格式遵循模板 -->
```

### 2. 在 OpenClaw 中启用身份进化
`openclaw.yaml` 中指定 `identity_file: IDENTITY.md`，并打开进化开关：

```yaml
agent:
  identity_file: IDENTITY.md
  evolution:
    enabled: true
    allowed_sections: ["Learned Experience"]
    max_entries_per_session: 3
```

OpenClaw 在启动时会将 `IDENTITY.md` 拼接进系统提示；当标识符 `## Learned Experience` 之后的内容发生变化时，Agent 只有在满足“用户明确命令 `/evolve`”或“连续三次相同问题未命中已知答案”时才会追加新条目。

### 3. 进化内容的落地与审核
每次进化会触发一个 pre-write hook，你可以挂一个简单的脚本做合规检查（禁止写入敏感词、超长文本），通过后再写入文件。团队内部我们将 `IDENTITY.md` 纳入 Git，每次进化自动 commit，message 为 `evolve: <entry summary>`，回滚非常方便。

### 4. 与 MCP 记忆服务器的协同
不把所有记忆都压在 `IDENTITY.md` 上。文件只保留最核心的人格约束和少量高频经验，大量业务记忆交给 MCP 记忆服务器（例如 `memory-server`）处理，避免文件膨胀导致系统提示 token 失控。

## 踩坑点

- **编码问题导致进化内容乱码**  
  在 Docker 容器内挂载 `IDENTITY.md` 时，未显式指定 `LC_ALL=C.UTF-8` 导致中文写入出现乱码，OpenClaw 解析时将其当作非法字符丢弃。务必在 Dockerfile 或启动脚本中设置正确的 locale。

- **文件权限引起进化静默失败**  
  OpenClaw 默认以非 root 用户运行，若宿主机上文件属主为 root 且权限为 644，Agent 更新会失败且日志中只留一条 `Permission denied`，很容易被忽略。提前 `chown 1000:1000 IDENTITY.md` 可解决。

- **AI 胡乱追加污染 Learned 区域**  
  如果没有在 prompt 中明确约束写入格式，Agent 可能会直接追加大段对话日志或无效标记。我们的修复方式是在 `openclaw.yaml` 中设置模板提示：“每次追加必须使用 `- 经验描述` 的短语形式，不超过 120 个字符”，并在 hook 中校验。

- **多 Agent 项目误用同一身份**  
  一个仓库里有多个 Agent 角色时，如果不加区分，共用的 `IDENTITY.md` 会让所有 Agent 行为趋同。实践中按角色拆分为 `claw_identity.md`、`docbot_identity.md` 并分别指定即可。

## 可复用建议

1. **拆分为 base + dynamic**  
   将核心人格和不可变规则放在 `identity_base.md`（只读），可进化部分放在 `identity_dynamic.md`（允许写入）。OpenClaw 支持多个 identity 文件按顺序拼接。
2. **设置硬性容量限制**  
   在 hook 中检查 `identity_dynamic.md` 的行数和总 token 数（例如不超过 800 tokens），超出则自动触发清理通知，由人工处理。
3. **用片段摘要替代原始记录**  
   不要让 Agent 直接记录用户原话，而是要求它提炼为“问题摘要 + 答案要点的简记”，有效压缩体积。
4. **版本控制 + 审计**  
   把 identity 文件纳入 Git，并设置 CI 检查（格式有效性、无敏感信息），让每一次进化都可追溯可回滚。
5. **配合 MCP 的 Prompt 伺服器**  
   对于需要更复杂身份管理的场景，可以考虑将 identity 内容通过 MCP 资源接口动态提供，`IDENTITY.md` 降级为“基础骨架”，灵活度更高。

## 总结

IDENTITY.md 并非什么颠覆性发明，它本质上就是一种 **“Agent 的长期系统提示词 + 受控自更新”** 模式。但在工程实践中，正是这种简单、透明、可版本化的机制，让 Agent 从一个每次都要重新灌输人设的工具，逐渐成为一个能够沉淀经验、持续进化的长期协作者。配合 MCP 生态和严谨的写入约束，这套方案能切实降低 Agent 维护成本，提升行为一致性，值得在中小型项目中落地尝试。

---

