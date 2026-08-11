---
title: 用 USER.md 给你的 OpenClaw Agent 装上「用户记忆
feedId: 32508
source: 综合讨论
publishedAt: 2026-08-11
---

# 用 USER.md 给你的 OpenClaw Agent 装上「用户记忆」

## 背景：Agent 很能打，但总不认得你

OpenClaw、MCP 和各种插件让 Agent 能联网、能写代码、能操控本地工具，但大多数 Agent 启动时对你的情况一无所知。你的常用技术栈、项目路径、编辑器偏好、甚至时区和语言，都要在 prompt 里反复手打。每开一个新会话，就像换了个新助手，重复交代上下文成了最大的隐性成本。

如果能在对话开始前，就自动注入一份关于“你是谁”的结构化档案，Agent 就能更自然地对齐你的工作流，少问废话，多出干货。这就是 **USER.md** 的初衷：一份 Markdown 格式的个人上下文文件，让 Agent 在每次会话中都先读一遍。

## 问题拆解

- **冗余沟通**：基础信息（比如“我用 pnpm 而不是 npm”）每次都要说明，否则 Agent 可能给出错误的命令。
- **上下文遗漏**：Agent 不了解你的机器环境（操作系统、可用工具链），生成的脚本可能无法直接运行。
- **多身份切换困难**：同时维护开源项目、写公司内网文档、做个人 Side Project，不同场景需要完全不同的上下文，手动切换 prompt 很累。
- **团队协作断层**：在团队中共享 Agent 使用时，每个人偏好不同，无法直接用同一套配置。

## 方案：一份 USER.md，随处注入

### 1. 设计内容模块

USER.md 不是流水账日记，而是给语言模型看的结构化提示。建议至少包含以下区块：

```markdown
# 关于我
- 昵称：alex
- 时区：Asia/Shanghai (UTC+8)
- 常用语言：中文、English
- 角色：全栈开发 / 开源维护者

# 环境
- OS：macOS 14, Apple Silicon
- Shell：zsh + oh-my-zsh
- 编辑器：VS Code (code 命令可用)
- 包管理：全局 pnpm，部分老项目用 yarn

# 项目结构
- 所有代码在 ~/dev 下，按组织分目录
- 当前主要项目：openclaw-ui（前端）、openclaw-core（Rust）
- 习惯的启动方式：`pnpm dev` 或 `make run`

# 偏好与约束
- 代码示例优先给 TypeScript，其次是 Python
- 命令说明中不要用 sudo，除非明确需要
- 解释技术概念时给出类比，但不要过度简化
- 避免生成带有 emoji 的正式代码注释

# 目标与状态
- 短期：2 周内完成插件系统的 MVP
- 今日关注：解决 MCP server 的握手超时问题
```

可以根据需要增加“会议记录格式”“常用 Docker 命令”等定制区块。重点在于：**把那些你每次都要在 prompt 里重复的内容，提前落成文档。**

### 2. 在 OpenClaw 中配置注入

OpenClaw 的灵活性允许通过多种方式将 USER.md 喂给 Agent。以下是我验证过最稳定的做法：

**方式 A：通过 systemPrompt 追加**

在 `openclaw.config.yaml`（或你使用的配置文件）中，利用全局 system prompt 直接加载文件内容：

```yaml
assistant:
  systemPrompt: |
    ${file:./user.md}
    
    以上是你的用户档案。请始终基于这些信息回答问题、生成代码。如果遇到档案未覆盖的情况，先询问再行动。
```

> 注意：如果你的配置引擎不支持 `${file:...}` 语法，可以用一个简单的 Node.js 脚本在启动前解析并展开，或直接复制内容进配置（不推荐，难以维护）。

**方式 B：通过 MCP Resource 动态加载**

如果你的 OpenClaw 实例连接了 MCP Server，可以将 USER.md 注册为一个 resource：

```json
// mcp-server 部分伪代码
{
  "resources": [
    {
      "uri": "file:///home/alex/.openclaw/user.md",
      "name": "user_profile",
      "mimeType": "text/markdown"
    }
  ]
}
```

然后在 prompt 模板中要求 Agent 首先读取该 resource：  
`请先读取 resource: user_profile，再开始回答我的问题。`

**方式 C：作为对话的第一条隐式消息**

某些 Agent 框架支持在每次会话开始前自动发送一条用户消息。可以配置为：

```
[SYSTEM] 用户档案: ... (此处为 user.md 内容)
```

推荐方式 A 或 B，因为它们对 Agent 来说更“自然”，避免占据用户消息的 token 配额。

### 3. 多档案切换

不同工作场景可以使用不同的 `.md` 文件：`user.work.md`、`user.oss.md`，再通过别名或启动参数切换。例如：

```bash
openclaw chat --profile=oss
```

内部实现只是替换 systemPrompt 中的文件路径。也可以根据当前工作目录自动选择档案（检测 `.git/config` 的 remote 或 `package.json`）。

### 4. 维护与版本管理

把 USER.md 放到 dotfiles 仓库中，和 `.zshrc`、`.gitconfig` 一起用 git 管理。每次换了工具链、调整了工作重点，都随手改一行。如果档案超过 800 字，考虑按“稳定信息”和“动态目标”拆成两个文件，动态部分用脚本生成，避免手工维护造成信息过时。

## 踩坑记录

### ❌ 坑 1：档案太长，吃掉上下文
刚开始我把所有能想到的偏好都塞进去，结果 USER.md 膨胀到 2000+ token。Agent 虽然了解我了，但处理长任务时留给实际工作的窗口太小。**对策**：硬性限制在 **500–800 字内**，动态目标部分用简短列表，详细说明放外部文档（让 Agent 需要时再通过 MCP 工具抓取）。

### ❌ 坑 2：敏感信息误暴露
如果在 USER.md 里写了 API Key、内网 IP、密码等，一旦通过 MCP 或配置发到云端 Agent，就会成为安全隐患。**对策**：档案中只用占位符（`API_KEY_BASE64`），实际值通过环境变量注入到 Agent 的运行时，禁止在 Markdown 文件中写秘钥。

### ❌ 坑 3：Agent 无视档案中的指令
偶尔 Agent 会“忽略”档案中的某条约束（如不要使用 sudo）。**对策**：在 systemPrompt 中用 **显式指令** 强调，例如“以上用户档案的优先级高于所有通用规则”，并在档案中使用祈使句：“命令中永远不要包含 sudo”。测试时若发现违反，直接在反馈中修正，或者用更短的句子重复。

### ❌ 坑 4：中文编码与特殊字符
如果你的配置文件是 YAML，而 USER.md 含有中文，可能出现编码解析问题导致 Agent 收到乱码。**对策**：确保文件存储为 UTF-8 无 BOM，YAML 中如果内嵌文件内容，用 `|` 保留换行，并检查 Agent 运行环境的 locale 设置（尤其是运行在容器中时）。

## 可复用建议

1. **提供模板仓库**：团队共享一套 USER.md 模板（如 `user.template.md`），新人 fork 后删除注释即可使用。
2. **用脚本动态注入可变部分**：当前工作目录、今天的待办事项可以用一个 `pre-openclaw.sh` 脚本抓取并拼接到静态档案底部，让 Agent 更“实时”。
3. **结合 MCP 的 Context 管理**：如果使用支持 context 的 MCP 实现，可以把 USER.md 作为默认 context 在连接握手时传递，而不是每次都读文件。
4. **定期做“档案体检”**：每两周花 2 分钟检查一次档案内容是否还符合实际。过时的偏好比没有偏好更危险。

## 总结

USER.md 的本质，是把频繁重复的个人上下文固化成一个可维护、可注入的资源。它不需要花哨的功能，只需要做到三点：**结构清晰、内容克制、注入可靠**。通过 OpenClaw 的 systemPrompt 或 MCP resource 机制，你可以为每一个 Agent 装上符合你身份的“记忆”，让工具真正围绕你的工作方式运转。

少一点解释，多一点产出——一份 500 字的 USER.md，往往能省下一千次反复说明。

---

