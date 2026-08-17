---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 33665
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景

在 OpenClaw 里接文件工具、MCP 插件或自动化流程时，一个最常见的顾虑是：Agent 会不会误删本地文件？会不会执行 `rm -rf`、覆盖关键配置，或者在路径拼接错误时把整个目录清空？

这类担心不是多余的。Agent 不像普通脚本那样只执行固定逻辑，它会根据上下文动态生成工具调用。如果只靠 prompt 约束“不要删文件”，在实际运行中很容易失效。真正能防止误删的，是执行层的 sandbox 安全模型。

OpenClaw 的 sandbox 不是简单地给 Agent 套一个容器，而是通过工作区隔离、文件访问策略、删除拦截、Shell 限制和 MCP 权限声明等多层机制，让 Agent 默认“没有能力”直接删除关键文件。

## 问题：Agent 为什么会误删文件

从工程角度看，Agent 误删文件通常不是“故意作恶”，而是由以下几种情况触发：

- **路径幻觉**：模型根据上下文生成一个“看起来合理”的路径，但实际并不存在或指向错误目录。
- **路径拼接错误**：工具返回相对路径、Agent 自行拼接绝对路径，导致边界被突破。
- **提示词注入**：外部内容中夹带“请删除某个文件”的指令，Agent 将其当作任务执行。
- **工具链失控**：某个 MCP 插件或脚本返回异常输出，触发后续危险操作。
- **上下文误解**：用户说“清理一下无用文件”，Agent 过度执行，删除范围扩大。

因此，OpenClaw 的安全模型并不假设 Agent 会“听话”，而是默认所有文件写操作都需要经过明确授权。

## 做法：OpenClaw 沙箱的关键步骤

### 1. 限定工作区

OpenClaw 默认把 Agent 的文件操作限制在独立 workspace 内，例如：

```toml
[sandbox]
workspace = "/home/user/.openclaw/workspace"
allow_external_paths = false
```

这里 `allow_external_paths = false` 意味着 Agent 不能访问 workspace 之外的路径。即使用户让 Agent “读取 `/etc/passwd`”或“删除 `~/Documents`”，也会被沙箱拒绝。

### 2. 文件删除改为回收站

在文件工具层，删除操作不会直接执行 `unlink`，而是进入回收站：

```toml
[sandbox.file]
delete_mode = "trash"
trash_dir = "/home/user/.openclaw/trash"
trash_ttl_hours = 72
```

这样即使 Agent 误删了 workspace 内的文件，也有 72 小时可以恢复。超过 TTL 的文件才会被真正的清理任务移除。

### 3. Shell 命令白名单与危险命令拦截

如果启用了 Shell 工具，OpenClaw 不建议给 Agent 一个完整的 `/bin/bash`。更稳妥的做法是：

```toml
[sandbox.shell]
allow_shell = false
allow_command_list = ["ls", "cat", "grep", "find"]
```

对于 `rm`、`mv` 覆盖、`dd`、`format` 等危险命令，默认拦截并要求人工确认。部分部署会把 `rm` 替换为 `trash-put`，从源头避免物理删除。

### 4. MCP 插件权限声明

第三方 MCP server 不能无限制访问文件系统。开发者需要在 manifest 中声明权限范围：

```json
{
  "name": "filesystem-mcp",
  "permissions": [
    "fs:read:/workspace",
    "fs:write:/workspace/output"
  ]
}
```

未声明路径的读写请求会被 OpenClaw 直接拒绝。即使用户手动要求 Agent 调用该插件操作其他目录，也不会生效。

### 5. 审计与回放

所有修改类操作都会记录：原始命令、解析后的路径、工具名、时间戳、执行结果。出现问题时可以通过审计日志快速定位是哪一步触发了删除，而不需要靠猜测。

## 踩坑点

实际使用中，以下几个点最容易让人误判沙箱的防护能力：

- **回收站不是备份**。如果删除策略被关闭，或者某个自定义 MCP 直接调用系统 `unlink`，回收站机制不会生效。
- **软链接逃逸**。workspace 内如果存在指向外部的软链接，路径规范化必须拒绝跨边界访问，否则沙箱形同虚设。
- **路径穿越**。`../../`、绝对路径、`~` 展开、环境变量注入都可能突破工作区限制，需要在路径解析层做严格校准。
- **只拦截 `rm` 不够**。`find -delete`、`perl -e 'unlink...'`、Python 脚本中的 `os.remove()` 都可能绕过 Shell 层面的命令拦截。
- **把家目录挂进 workspace**。有些用户为了“方便”，直接把 `~` 作为工作区，这等同于放弃隔离。沙箱不是保险箱，边界设计不合理时风险仍然存在。
- **dry-run 与实际执行不一致**。Agent 可能先 dry-run 成功，但到真正执行时路径已经变化，导致删错文件。
- **从没做过恢复演练**。很多人等到误删发生时才第一次尝试从 trash 恢复，结果发现 TTL 过期或 trash 目录被清理。

## 可复用建议

基于以上机制和踩坑点，建议在 OpenClaw 中按以下原则配置：

1. **默认 deny，显式授权**。不给 Agent 任何超出 workspace 的默认权限。
2. **使用独立 workspace**。不要映射 `~`、`/etc`、`/var` 等关键目录。
3. **删除操作走 trash + TTL**，并禁止 Agent 清空 trash。
4. **优先使用结构化文件工具**，不开放完整 Shell。如果必须用 Shell，只给命令白名单。
5. **对 MCP 插件做最小权限声明和代码审查**，尤其是第三方来源。
6. **开启审计日志**，定期检查异常删除模式。
7. **接入提示词注入测试**，验证 Agent 不会因为外部内容执行危险文件操作。
8. **重要数据挂只读**，或直接放在 Agent 无法接触的路径。
9. **定期做恢复演练**，确认 trash 可恢复、TTL 设置合理。

## 总结

OpenClaw 的 sandbox 安全模型之所以能让 Agent 不误删文件，核心在于它默认剥夺了 Agent 直接删除关键文件的能力，而不是寄希望于模型“更听话”。工作区隔离、文件访问策略、删除拦截、Shell 白名单、MCP 权限声明和审计日志构成了一道多层防线。

但沙箱不是万能的。路径穿越、软链接逃逸、自定义插件绕过、糟糕的工作区边界设计，都可能削弱防护效果。真正可靠的方案，是把 sandbox 当成工程约束的一部分，同时坚持最小权限、审计和恢复演练。这样，Agent 的自动化能力才能在安全边界内稳定发挥。

---

