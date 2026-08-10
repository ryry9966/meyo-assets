---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 32464
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景
在构建个人助手或自动化 Agent 时，大量精力会被投放在工具集成、提示词工程与工作流设计上，但有一个环节始终被低估：**Agent 到底知不知道你是谁**。  
如果没有上下文，通用模型只能给出泛泛的建议；它不了解你的代码风格、正在进行的项目、工作时段，甚至你偏好的 shell。很多实践者会把这类信息硬编码在系统提示词里，但提示词一长就容易失控，也难以在多会话、多 Agent 间复用。  
在 OpenClaw 这类 Agent 框架中，可以把个人上下文提取为一个独立可维护的文件——`USER.md`，再通过 MCP 或内置机制注入。这能让你的 Agent 变得真正“认识你”。

## 问题
- 如何用**单一真相源**维护个人偏好，避免在多处提示词里重复粘贴？
- 如何在每次会话启动时自动加载，而不用手动复制？
- 如何兼顾隐私、Token 上限、更新频率？

一个经过工程化处理的 `USER.md`，配合 MCP 动态读取，就可以低成本解决这些问题。

## 做法与步骤
### 1. 定义 USER.md 模板
采用 YAML front matter + 自由文本的结构，强制保持轻量。  
例：
```markdown
---
lang: zh-CN
preferred_editor: vscode
shell: zsh
timezone: Asia/Shanghai
work_hours: 10:00-19:00
projects:
  - name: openclaw-demo
    repo: ~/Workspace/openclaw-demo
    stack: [python, fastapi, sqlite]
rules:
  - 任何代码必须添加类型注解
  - 优先使用 async/await
constraints:
  - 不要使用未经验证的第三方库
  - 不要生成超过200行的单文件代码
---
# 工作摘要
正在为内部平台开发 MCP 插件，需要兼容 OpenClaw 的 Agent 接口。
...
```
严格限制正文字数（建议≤500字），把 USER.md 当作“个人 schema”而非日记。

### 2. 通过 MCP 暴露读取工具
如果你的 Agent 已经支持 MCP（Model Context Protocol），可以搭建一个最小的 MCP server，提供 `read_user_profile` 工具。也可以直接复用已有的文件系统 MCP server，开放对 `~/.openclaw/USER.md` 的读权限。  
例子（使用 mcp-filesystem）：
```json
{
    "mcpServers": {
        "user-profile": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/yourname/.openclaw"]
        }
    }
}
```
在 Agent 配置中将工具调用结果注入系统提示，例如：
```
<user_profile>
{{ call_tool('user-profile', 'read_file', { 'path': 'USER.md' }) }}
</user_profile>
```

### 3. 插入提示词并设定刷新策略
- 在每次对话初始化时自动读取 `USER.md`，避免缓存过时。
- 可以将内容以 markdown 分隔块的形式固定放在系统提示的靠前位置，用标签 `<user_profile>` 包裹，让模型更容易视其为高优先级上下文。
- 如需在会话中动态修改上下文（比如切换项目），可以再暴露一个 `switch_project` 工具，更新 `USER.md` 并触发 Agent 重读。

### 4. 验证
向 Agent 提问：“我目前在用哪个项目？它的技术栈是什么？”或者“请按我的规则重构以下代码”。如果 Agent 能正确引用 `USER.md` 中的字段，说明注入成功。

## 踩坑点
1. **Token 超限**  
   人为地总想把所有习惯都写进去，导致每次注入的 Token 过多。解决办法：限定 YAML 字段数目，正文只能是摘要。超过阈值的详细说明可以拆成链接或通过工具按需读取。
2. **隐私泄露**  
   `USER.md` 可能被误提交到公开仓库或传至云端。绝对不能包含密码、Token、内网地址。敏感变量应使用占位符或环境变量引用，Agent 侧仅解析变量名。
3. **解析失败导致上下文混乱**  
   有的框架会尝试自动提取结构化信息，如果 YAML 格式错误或存在特殊字符，可能导致整个系统提示被截断。建议在 MCP server 侧做好错误处理，返回空字符串时 Agent 仍能正常工作。
4. **多层级冲突**  
   项目目录下的 `USER.md` 可能与全局 `~/.openclaw/USER.md` 并存。最好约定优先级：项目级覆盖全局，但全局只包含通用偏好。加载时合并两个文件，避免遗漏。

## 可复用建议
- **最小化字段**：只放 Agent 真正需要知道的差异点，如编程语言、框架偏好、命名规范，而不是完整的履历。
- **结构化优先**：YAML 字段便于程序检索，在未来可能需要由 Agent 自主决策时，可以编写工具查询特定 key，如 `get_preference('editor')`。
- **结合 MCP 动态上下文**：不只是静态文件，还可以让 MCP server 同时返回当前 Git 分支、活动终端目录等实时信息，与 `USER.md` 一起组成 `<user_context>` 块。
- **版本控制与忽略**：建议保存在私有 dotfiles 仓库中。如与项目耦合的配置，使用 `.user.local.md` 并加入 `.gitignore`。
- **测试恢复机制**：当 MCP server 不可用时，Agent 仍应能回退到默认 Prompt，而不是直接报错。

## 总结
`USER.md` 不是炫技，而是一个低成本的工程杠杆：它把对“人”的描述从庞杂的提示词中剥离出来，形成可维护、可共享、可程序化读取的单一真相源。在 OpenClaw 的 Agent 工作流里，结合 MCP 可以做到开箱即用的个性化注入。  
记得保持轻量、克制字段、为隐私设防，你就能让 Agent 真正懂你——并且让这种理解随着你的工作节奏一起演进。

---

