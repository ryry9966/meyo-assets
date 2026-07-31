---
title: Agent 记忆系统设计：如何让 AI 助手真正记住你的偏好
feedId: 31131
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景

我们在用 OpenClaw 搭建 Agent 时，经常遇到一个尴尬场景：每次新会话都要把“我喜欢用 TypeScript”“我的数据库端口是 5433”“文档风格偏简洁”再说一遍。即使通过 System Prompt 写入固定偏好，一旦需要修改就得改配置重启，不够灵活。而真正的“个人助手”应该能跨会话记住并主动应用你的习惯。

Agent 记忆系统就是解决这个问题的工程模块：在对话中持久化用户偏好，并在后续交互中自动注入上下文。它不是“长期记忆”的 AGI 幻想，而是一套可落地的轻量级设计，核心是把“记住偏好”这件事做成工具，Agent 按需读写。

## 问题拆解

设计记忆系统前，先梳理要解决的子问题：

1. **存什么**：偏好类型可能是键值对（如 `language: TypeScript`）、列表（`avoid_topics: [politics]`）或结构化片段（`project_a: { port: 5433, branch: main }`）。
2. **怎么存**：需要跨会话持久化，且读取频率远高于写入。JSON 文件、SQLite、甚至 Redis 都可以，选择取决于部署环境。
3. **何时召回**：每次会话初始化时预加载全部偏好？还是让 Agent 在需要时主动查询？实际工程里“预加载摘要 + 按需精准查询”更实用。
4. **如何更新**：用户说“其实我更偏好 Golang 了”，Agent 必须能覆盖旧值，且不能靠人类去手动改配置文件。
5. **多用户隔离**：如果 Agent 服务多个人，记忆必须按用户 ID 分片，否则偏好会交叉污染。

## 实现步骤（以 OpenClaw + MCP 为例）

### 1. 构建记忆 MCP Server

用 Python 写一个轻量 MCP Server，暴露两个工具：`remember_preference` 和 `get_preferences`。存储层用 JSON 文件，按 `user_id` 建立独立的 namespace。

工具定义示例：

```python
@app.tool()
async def remember_preference(key: str, value: str, user_id: str) -> str:
    """存储或更新用户偏好。key 为偏好名，value 为偏好值。"""
    prefs = load_prefs(user_id)
    prefs[key] = {"value": value, "updated_at": datetime.now().isoformat()}
    save_prefs(user_id, prefs)
    return f"已记住：{key} = {value}"

@app.tool()
async def get_preferences(user_id: str, query: str = None) -> str:
    """获取用户偏好。query 可选，用于模糊匹配 key。"""
    prefs = load_prefs(user_id)
    if query:
        matched = {k: v for k, v in prefs.items() if query.lower() in k.lower()}
        return json.dumps(matched, ensure_ascii=False)
    return json.dumps(prefs, ensure_ascii=False)
```

### 2. 在 OpenClaw 中注册 MCP Server

在 `openclaw.yaml` 中增加 `mcp_servers` 配置：

```yaml
mcp_servers:
  - name: memory
    command: python
    args: ["-m", "memory_server"]
    env:
      STORAGE_PATH: /data/preferences
```

### 3. 设计 Agent 的行为指令

在 System Prompt 中加入记忆使用协议：

```
你有一个记忆系统，可以通过工具记住和获取用户偏好。
- 当用户明确表达偏好时（如“我喜欢…”，“以后都用…”），
  调用 remember_preference 存储。
- 每次会话开始时，先用 get_preferences 获取当前用户的全部偏好，
  并默默应用到后续回答中。
- 如果用户修改偏好，调用 remember_preference 更新。
- 注意：所有记忆工具调用都要带入 user_id = {current_user_id}。
```

`{current_user_id}` 由平台变量注入，例如从 JWT 或会话 ID 映射。

### 4. 实际运行流

用户 A 在第一次对话说：“我写的代码都用 TypeScript，注释用英文。”

Agent 内部行为：
- 识别到偏好表达 → 调用 `remember_preference(key="language", value="TypeScript", user_id="A")`
- 调用 `remember_preference(key="comment_lang", value="English", user_id="A")`

48 小时后用户 A 问：“帮我写一个登录接口。”
Agent 在会话初始化时调用 `get_preferences(user_id="A")`，拿到 preferences JSON，生成代码时直接用 TypeScript 并用英文注释，无需用户再提示。

## 踩坑记录

1. **工具描述不足导致 Agent 乱用**  
   MCP 工具的 `description` 必须精确，否则 Agent 可能在每次回复前都调用 `get_preferences`，造成大量冗余请求。我们在 description 里明确：“仅在会话开始或用户要求更新记忆时调用”，问题解决。

2. **偏好冲突合并**  
   用户可能说“用 Python 吧，哦不还是 Go”。如果 Agent 把两次都记下来，会变成两条 `language: Python` 和 `language: Go`。方案是工具层面只存最新值，description 注明“相同 key 会覆盖旧值”。

3. **多用户隔离被忽略**  
   初期我们用全局 JSON 文件，测试时两个同事的偏好互相覆盖。改成 `user_id.json` 后解决。建议存储层一开始就按用户分片。

4. **隐私与持久化位置**  
   偏好可能包含个人项目路径、内部系统端口。如果 Agent 部署在公网，存储路径必须限制访问权限，文件内容可考虑对称加密。我们最终把 JSON 放在一个仅容器内可读的 Volume 里。

5. **预加载 vs 实时查询的权衡**  
   如果偏好条目很多（超过 50 条），会话启动时全量读取会造成延迟。我们改为加载最近更新的 10 条 + 按需查询，既保证了高频偏好的命中率，又控制了上下文长度。

## 可复用建议

- **把记忆做成标准 MCP 工具包**：任何支持 MCP 的 Agent 框架（如 OpenClaw、Claude Desktop 等）都能直接挂载，零侵入。
- **存储格式用 JSON，加时间戳**：便于调试，也方便未来加“遗忘策略”。
- **提供模糊查询能力**：Agent 在不确定 key 英文名时可以搜，比如用户说“我关于代码风格的设置”，Agent 能查询包含“style”的 key。
- **渐进式扩展**：初期只用键值对；后期可扩展为语义记忆（向量化存储），通过 embedding 匹配模糊意图，但不要一上来就上向量库，工程复杂度会爆炸。
- **工具职责单一**：读写分离，`remember_preference` 只负责写/改，`get_preferences` 只负责读，避免一个工具既读又写导致 Agent 意图混乱。

## 总结

Agent 记忆系统的核心不是“让 AI 拥有记忆”的玄学，而是把持久化读写偏好包装成 MCP 工具，让 Agent 像调用其他插件一样管理这些信息。工程实现的本质是：**存储用户状态，在正确的时机注入上下文，并提供无摩擦的更新路径**。

如果你已经在用 OpenClaw 搭建自动化工作流，可以花 1 小时实现一个基础版记忆 Server，体验会立刻不一样。别把“记忆”做重，先解决“别让我再说一次”就很实用了。

---

