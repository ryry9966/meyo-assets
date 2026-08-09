---
title: 用 USER.md 告别重复交代：让 AI Agent 真正了解你的工程实践
feedId: 32225
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景：AI Agent 的知识盲区

不管是用 OpenClaw 跑自动化任务，还是让 Claude Code 帮你改代码，Agent 对项目规则与代码仓库的理解已经相当不错。但一旦跳出项目边界，它们就“失忆”了：不知道你的工作习惯、常用本地工具路径、偏好的技术栈倾向，甚至不知道你本职是后端还是产品。于是每次新会话你都要再交代一遍：“我用 pnpm 不用 npm”，“我习惯用中文注释”，“我的 SSH 密钥路径是 `~/.ssh/id_ed25519`”……这显然不工程化。

我们需要一种轻量、可复用的机制，让 Agent 在每次会话启动时自动获取**与你个人相关的上下文**，而不必污染项目级别的 `CLAUDE.md` 或 `COPILOT_INSTRUCTIONS`。

## 问题拆解

一个可用的“用户说明书”方案需要解决几个问题：

1. **存什么**：信息要有结构，能覆盖日常交代的高频内容，又不能变成个人日记。
2. **怎么注入**：Agent 如何可靠地读取这个文件？如何在不用手动复制粘贴的前提下，保证每次会话都能取到？
3. **安全边界**：文件里可能包含半敏感信息（组织架构、内部路径），如何防止泄漏或被 Agent 误改？
4. **维护成本**：信息需要长期有效，不能每次改个终端别名就要修文件。

## 做法：从 USER.md 到可执行的上下文层

### 1. 定义个人上下文文件

在用户目录下创建 `~/.agent/user.md`（名字随意），内容按“领域-信息”结构组织，例如：

```markdown
# About me
- Role: 后端/基础设施工程师，偶尔写前端
- Languages: Go, Python，Shell，能读 Rust 但不写
- Work pattern: GMT+8，上午专注不被打断，下午2-4点回复协作消息

## Preferences
- Package manager: pnpm for Node，poetry for Python
- Git: 用 `git c` 代替 `git commit`，`git lg` 看日志
- Linter: 写 Go 时喜欢用 `golangci-lint run` 全部检查

## Local environment
- Workspace root: `~/dev/work`（公司项目），`~/dev/side`（个人）
- SSH key: `~/.ssh/id_ed25519_work`
- Docker: 用 Colima 代替 Docker Desktop，socket 路径 `~/.colima/default/docker.sock`

## Active constraints
- 所有新脚本如果涉及密码/Token，一律用 `pass` 或 1Password CLI，不要明文存盘
- 临时文件放 `/tmp/openclaw-task/`，任务结束必须清理
```

文件本身保持“声明式”，只写事实与偏好，不写推理过程，避免让 Agent 产生过度的前置假设。

### 2. 打通 MCP 读取通路

最稳定的做法是走 MCP 的 `filesystem` 服务。以 OpenClaw 为例（同样适用 Claude Desktop 或任何支持 MCP 的 Agent runtime），配置一个只读的 `user-context` 类型 resource：

```yaml
# agent.yaml（OpenClaw 配置）
context_providers:
  - type: mcp
    server: filesystem
    resources:
      - uri: file:///home/yourname/.agent/user.md
        inject_as: system_message_prefix
        read_only: true
```

如果是 Claude Desktop，`claude_desktop_config.json` 中类似：

```json
"mcpServers": {
  "fs-user": {
    "command": "npx",
    "args": [
      "-y",
      "@anthropic/mcp-server-filesystem",
      "/home/yourname/.agent"
    ]
  }
}
```

然后在会话开头用提示词规则要求 Agent 先读取 `user.md`，或利用 OpenClaw 的 `on_session_start` 钩子自动读取并拼入系统消息顶部。后者的好处是用户连“请先读我的 user.md”都不必说。

### 3. 分层控制：不要把所有信息全塞进去

如果 `user.md` 体积较大（例如包含了详细的工具别名表、私有脚本用法），建议拆分为：

- `user-profile.md`：基础身份和偏好（总是注入）
- `user-tools.md`：工具路径与别名详细列表（按需读取）
- `user-security.md`：敏感策略，仅在安全任务中读取

通过 MCP 的不同 resource 路径区分，Agent 可以在需要时用 `read_resource` 动态获取。这样可以避免每条消息都带上几千字上下文，浪费 Token 也容易让模型分心。

### 4. 用简单校验防呆

我在 `~/.agent/` 下放了一个 `validate.sh`，内容大致是：

```bash
#!/bin/bash
# 检查 user.md 是否包含不应该出现的字符串
grep -E '(password|secret|token):\s*[^\s]+' ~/.agent/user.md && echo "WARNING: possible secret detected"
```

然后在 git 提交钩子里跑一次，同时将 `~/.agent/` 加入 `.gitignore` 确保不会误推到远程。如果你用 dotfiles 管理 `~/.agent/`，可以单独开私有仓库，只同步到受信设备。

## 踩坑记录

1. **Agent 误修改 user.md**
   即使设置了 `read_only: true`，某些 MCP server 实现仍可能漏写保护。建议直接在操作系统层设置文件只读：`chmod 444 ~/.agent/user.md`，或用 `chattr +i`（Linux）锁定。别忘记你需要修改时先解锁。

2. **路径膨胀**
   刚开始很容易把 `user.md` 写成“我常用工具一览”，包含几十行 alias。Agent 看到大量别名后会过度自信地去调用，比如以为你有一个 `deploy` 命令就直接执行。实际上很多别名只在交互 shell 里有效，Agent 的非交互环境没加载 `.bashrc`。解决方式是明确标注“以下别名仅在交互 shell 可用，Agent 请用完整命令”，并在测试后逐步加入。

3. **多机多环境不同步**
   如果你在公司台式机和家用笔记本上各有一份 `user.md`，很容易产生分歧。建议根据机器 hostname 在文件开头用一段条件信息，或者使用 `user-$(hostname).md` 按设备区分。OpenClaw 可以在配置里用环境变量替换文件名。

4. **隐私边界**
   即使文件不存云端，Agent 运行时可能将内容拼入请求发送至 API（比如 Claude 或 GPT 的远端服务）。如果你的 `user.md` 里有内部项目代号、组织架构等半敏感信息，务必意识到这些文字会被离开你机器。我的原则是：只放 **Agent 履行任务必需** 的信息，绝不为了方便多写一行业务细节。

## 可复用建议

- **模板先行**：先建立一个最小可用 `user.md` 骨架，包含角色、时区、包管理器偏好、本地路径前缀。运行一周后，根据哪些信息仍被重复询问，再增量补入。
- **与项目规则解耦**：不要复制项目级别的规则到 `user.md`，那里放个人偏好。项目规则仍放在项目根目录的 `CLAUDE.md` 或 `OPENCLAW.md` 中，避免耦合。
- **定期审计**：每月用一句提示检查：“请读取我的 user.md，告诉我哪些信息可能已经过时或相互矛盾。” Agent 会有启发性的反馈，但采信前仍需自己判断。
- **搭配快捷命令**：如果 Agent 不支持自动注入，可以给自己写一个别名 `agent:me`，执行时读取 `user.md` 后粘贴到对话开头，弥补平台限制。

## 总结

Agent 越来越能干活了，但它们对你的了解仍停留在每次会话的方寸之间。`USER.md` 不是革命性技术，只是把“重复交代”这层人机摩擦用工程化的方式消解掉。配上 MCP 的只读资源和分层信息设计，它就能从随手笔记升级为无声的系统提示。在 OpenClaw 这类高度可定制的 Agent 框架里，你甚至可以将它作为 `context_layer` 的第一个组件，彻底告别“我是谁”的重复对话。

如果动手试试，建议从极简版本开始，让它在真实任务中证明自己的价值，再逐渐长成你专属的 Agent 自我介绍书。

---

