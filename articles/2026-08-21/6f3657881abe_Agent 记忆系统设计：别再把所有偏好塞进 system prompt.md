---
title: Agent 记忆系统设计：别再把所有偏好塞进 system prompt
feedId: 34071
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在给 OpenClaw 接自动化、MCP 工具或自定义插件时，常见问题：Agent 每次都要重新问“你希望用什么语言”“代码风格是什么”。一些开发者直接把偏好写死在 system prompt 或 Markdown 里，上下文越来越长，行为却越来越不稳定。记忆问题不是“能不能记住”，而是如何分层、检索、更新和隔离。

## 问题

我见过最典型的失败模式：用户说了“以后都用中文”，Agent 把它写进一个 unstructured memory.txt；下个会话因为没读到，又改回英文。或者系统把几十条偏好全量注入 system prompt，token 成本高，Agent 还容易忽略。更麻烦的是多工作区/多用户场景，偏好互相污染，甚至把 token 写进记忆文件。

这些问题的根因是：没有把“记忆”当作一个可维护的工程模块，只把它当作 prompt 的一部分。

## 做法/步骤

**1. 先分层，不要一开始就建向量库。**  
短期记忆：当前会话、任务上下文，用完即弃。长期 Profile：用户偏好、环境信息、默认工具。程序性记忆：怎么执行某类任务的步骤。对多数 OpenClaw 自动化，只需要 Session Memory 和 Profile Memory 两层。

**2. 设计结构化 Profile。**  
用 JSON 或 SQLite 存字段，别存自然语言段落。示例：

```json
{
  "user": "default",
  "preferences": {
    "language": "zh-CN",
    "response_style": "concise",
    "code": {"lang": "python", "typing": true}
  },
  "updated_at": "2025-01-01T00:00:00Z",
  "source": "explicit"
}
```

字段要少而明确。显式偏好（用户明确说）和推断偏好（Agent 猜测）要分开，推断偏好带 `confidence`，避免污染。

**3. 存储位置。**  
单用户小规模用 JSON 文件，放工作区 `.openclaw/memory/profile.json`，纳入版本控制。多用户/多 Agent 用 SQLite，避免并发写坏。需要语义检索时才加向量库，且只做补充检索，不替代结构化字段。

**4. 读写接口。**  
不要把记忆文件暴露给 Agent 直接编辑。通过工具或 MCP server 暴露 `memory_get` / `memory_set` / `memory_delete`。这样每次写操作都可以校验 schema、记录来源和时间戳。OpenClaw 可以接一个 MCP memory server，底层换成 SQLite 或 JSON 都可以，工具接口不变。

**5. 注入策略。**  
不要全量注入。每次任务开始时，按需加载 profile 摘要，比如最多 10 条，按更新时间或置信度排序。摘要规则固定：只注入相关字段，敏感字段做脱敏。

**6. 更新与冲突。**  
写操作要留痕：`source`、`updated_at`、`confidence`。显式偏好覆盖推断偏好；同级别冲突时保留最新显式，或询问用户。可以给每条偏好加 `TTL` 或定期 review，防止“以后都用中文”这种短期指令永久化。

## 踩坑点

- **全量注入 system prompt**：token 膨胀且 Agent 不一定遵循长列表。改为摘要注入。
- **把一次性要求当长期偏好**：用户说“这次用中文”不应写入 profile，只有“以后都用中文”才写。需要区分任务级和偏好级。
- **JSON 文件并发写坏**：如果多个进程或 Agent 同时写，用 SQLite 或文件锁。别假设单进程。
- **敏感信息进记忆**：token、密钥、私钥不要写普通记忆文件，放 secret store，记忆里只存引用。
- **没有 schema 校验**：记忆文件很快变成垃圾场。用 JSON Schema 或 Pydantic 在工具层校验。
- **多工作区污染**：偏好要带 namespace（workspace/user/project），避免 A 项目的偏好影响 B 项目。

## 可复用建议

- 先做结构化 profile，再考虑向量检索。90% 的场景结构化字段够用。
- 把记忆写操作收口到工具/MCP，保留变更日志。
- 给 profile 做回放测试：新会话启动后，检查 Agent 是否能按偏好执行。
- 命名空间和 TTL 是成本最低、收益最高的两个机制。
- 在 OpenClaw 中，用 MCP 插件封装记忆读写，可以让不同 Agent 复用同一套记忆后端，而不必改 prompt。

## 总结

Agent 记忆系统的目标不是给更大的上下文，而是把长期状态管理起来：分层、结构化、工具化写入、按需检索、显式优先。先把这几件事做对，比上向量库更实际。一个可维护的 memory 模块，应该让 Agent 在新会话里少问重复问题，同时不把上下文变成不可控的偏好清单。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/ef6cfb289d323412.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/8bf9821347857de3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/ca25b74159c4f247.png)

