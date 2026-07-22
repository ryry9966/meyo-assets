---
title: 给 Agent 做减法：给自动化脚本套上本地目录白名单
feedId: 30140
source: 综合讨论
publishedAt: 2026-07-23
---

# 给 Agent 做减法：给自动化脚本套上本地目录白名单

## 为什么需要这个护栏

在 OpenClaw 这类 Agent 框架里，我们经常把文件读写能力通过 MCP Server 或插件丢给模型。典型的配置可能是一个 filesystem 工具，允许 Agent 随意操作 `~` 下的所有文件。这在早期原型阶段看着很方便，但一旦 Agent 流程开始对接真实的本地工程目录、配置文件或日志，风险就肉眼可见地变大：

- 非预期的文件删除或覆盖：Agent 可能误以为某个 JSON 文件可以重写，实际上那是你的系统配置。
- 敏感信息泄露：Agent 读取了 `.env`、密钥文件，顺手把内容贴进了输出摘要里。
- 调试信息污染：Agent 创建了大量临时文件，却不知道应该放到哪个沙箱里。

解决这类问题的工程化做法，不是彻底禁用文件访问能力，而是给能力套上一个**本地目录白名单**，让 Agent 只能在预先设定的安全路径内读写操作。

## 问题场景具体化

以 OpenClaw 常见搭配的 `@modelcontextprotocol/server-filesystem` 为例，它的启动参数允许传入多个 `--allowed-directory`。但如果只是简单配一个比如 `/home/user/projects`，会留下一些工程隐患：

- 白名单目录下的符号链接可能指向外部路径，Agent 顺着就走出去了。
- 相对路径操作取决于 Agent 的当前工作目录，有时会跑到白名单之外。
- 多用户或多项目环境里，目录权限交叉，白名单边界模糊。

下面用一个实际配置改良过程，展示如何把文件访问规则收紧，让 Agent 安心在限定区域里干活。

## 实操步骤：构建一个严格的文件访问护栏

假设我们有一个 OpenClaw 的 Agent 实例，需要用它来整理某个项目 `/data/projects/my-app` 下的 Markdown 文档，并允许把生成的汇总文件写到 `/data/output` 目录。我们不允许 Agent 碰任何系统文件、配置或主目录下的东西。

### 1. 准备白名单目录结构

在宿主机或容器内创建两个干净的目录，并确保没有不必要的符号链接或挂载点：

```bash
mkdir -p /data/agent-safe/project /data/agent-safe/output
```

然后将实际项目需要用到的文件，以只读或最小权限方式复制或挂载到 `/data/agent-safe/project`。用 bind mount 或 rsync 都可以，取决于是否需要实时同步。重点是这个目录内部不能有指向白名单外的符号链接。

强制检查符号链接：

```bash
find /data/agent-safe -type l -exec ls -l {} \;
```

如果存在外指链接，替换为真实文件挂载或直接解除链接。

### 2. 配置 MCP filesystem 工具

在 OpenClaw 的 MCP 配置文件（通常是 `mcp.json` 或 `cline_mcp_settings.json`）中，不要写成：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/data/agent-safe/project",
        "/data/agent-safe/output"
      ]
    }
  }
}
```

这样虽然只暴露了两个目录，但仍然有一个坑：工具启动后的默认工作目录可能继承自 Agent 进程，导致某些不受限的相对路径操作意外生效。因此需要把 server 自身也锁在一个安全工作目录下：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "--allowed-directory",
        "/data/agent-safe/project",
        "--allowed-directory",
        "/data/agent-safe/output"
      ],
      "cwd": "/data/agent-safe"
    }
  }
}
```

同时检查 MCP server 的版本，老版本可能没有正确处理多个 `--allowed-directory` 的严格隔离，需要升级到最新发布。

### 3. 在 OpenClaw 中定义工具访问边界

即便 MCP 工具设了白名单，Agent 的系统提示词里也需要明确声明允许的目录范围，避免模型因为“幻觉”而试图通过其他工具（如命令行执行）绕过界限。在 OpenClaw 的规则或自定义指令中增加：

```markdown
- 你只能通过 filesystem 工具访问 /data/agent-safe 下的文件。
- 任何创建、修改、删除操作仅允许在 /data/agent-safe/output 下执行，/data/agent-safe/project 为只读区域。
- 如果任务需要访问超出此范围的文件，请直接拒绝并说明原因。
```

如果 OpenClaw 支持工具层策略（如 tool policy），把 filesystem 工具的调用参数纳入正则校验，只允许路径以 `/data/agent-safe/` 开头，这可以作为第二道防线。

### 4. 验证与测试

启动 Agent 后，用一系列试探性 prompt 测试护栏：

- “读取 /etc/passwd 的内容” → 工具应返回错误或拒绝访问。
- “在 /data/agent-safe/output 下创建一个 test.txt，写入当前时间” → 成功。
- “删除 /data/agent-safe/project/readme.md” → 应被拒绝（如果设了只读策略）或工具受限。
- 尝试使用符号链接绕过：在项目目录内创建一个链接 `ln -s /etc/hosts ./shortcut`，再用 Agent 读取 `project/shortcut`。如果服务器或内核层面没有额外保护，这一步可能成功，因此前置的符号链接清理很重要。

在 Linux 环境中，还可以结合 AppArmor 或 seccomp 为 MCP server 进程框定文件系统边界，作为内核级的加固。

## 踩坑点

1. **白名单路径的尾斜杠差异**  
   有用户报告，传入 `--allowed-directory /data/agent-safe/project/` 和 `/data/agent-safe/project` 在部分 filesystem MCP server 版本中行为不一致，导致路径匹配失败。统一使用无尾斜杠的绝对路径，并查看 server 日志确认被正确解析。

2. **环境变量和 home 目录展开**  
   配置里写 `~` 或 `$HOME` 可能不会被 MCP server 正确展开，特别是在非交互式启动环境里。永远使用完整绝对路径。

3. **MCP server 的错误返回过于模糊**  
   当 Agent 尝试访问白名单外路径时，工具可能只返回 “Permission denied” 或 “No such file”，Agent 容易误读成文件不存在而继续试错，甚至转而用其他方式去读取。可以在系统提示里明确“访问受限时的正确行为是停止并报告”。

4. **白名单内的文件仍可能被模型误操作**  
   白名单只是路径约束，不防止 Agent 把正确的文件写得面目全非。如果输出目录里原有重要文件，应配合版本控制或定期备份，而不要仅仅依赖护栏。

## 可复用建议

- **最小化白名单目录数量**：不要一次性给整个 home 目录，而是拆分成明确的独立目录，并为每个目录赋予独立的读写属性（如读/写隔离）。
- **容器化运行时天然隔层**：把 OpenClaw 和 MCP server 放在容器里，用 volume 映射只暴露白名单目录，这样多了一层挂载隔离。
- **保留操作日志**：让 MCP server 的 stderr 输出到日志系统，审查所有文件访问请求，便于事后审计。
- **定期自检**：写一个自动化测试脚本，模拟 Agent 调用 filesystem 工具尝试越权，确保配置未经意外修改而失效。
- **结合版本控制**：对于输出目录的内容，如果可能，用 `git init` 并配置自动提交，让 Agent 的修改可追溯、可回滚。

## 总结

Agent 的文件访问护栏看似是一个简单的目录白名单配置，实际落地时却需要处理符号链接、相对路径、工具自身行为不一致等多重细节。真正可靠的方案不是单个配置项，而是“提示词约束 + 工具参数白名单 + 运行环境隔离”的三层保护。在工程化流程中，这个护栏不仅保护宿主机安全，也限制了 Agent 自身的破坏范围，让独立自动化脚本更可控、更可审计。

在 OpenClaw 社区里，这类实践还是“刚需但容易被忽视”的环节。希望这次的配置套路和踩坑记录，能帮你在给 Agent 开文件权限时多一分底气，少一点事后擦屁股的时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/8a4b071c79cd55b7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/0698f7206617de2b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/8e0c9d1f163c47b7.png)

