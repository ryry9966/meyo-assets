---
title: 给 Agent 的双手戴上手套：MCP 文件系统白名单实战
feedId: 30679
source: 综合讨论
publishedAt: 2026-07-27
---

## 为什么要给自动化脚本装“护栏”？

Agent 在自动化任务中免不了要读写本地文件：生成报告、导出数据、修改配置文件。一旦把操作系统命令或脚本执行能力交给 Agent，风险就随之而来——恶意提示词可以诱导 Agent 执行 `rm -rf ~/`，或者悄悄读取 `~/.ssh/id_rsa`。

传统的隔离做法是跑 Docker 容器，但在轻量场景（比如本地开发辅助）中过于臃肿。我们需要一种更经济的方案：只让 Agent 触碰我们指定的目录，其他地方一律禁入。

OpenClaw 生态中的 MCP (Model Context Protocol) 提供了现成的 `filesystem` 服务器，它支持 `allowed_directories` 白名单参数，正好能实现这种细粒度的访问控制。

## 问题再现：一个没有护栏的 Agent

假设我们在 OpenClaw 中启用了本地命令执行工具（如终端或 Python REPL），Agent 接到任务：“分析当前文件夹下所有 CSV 并合并”。它很可能直接调用 `os.listdir('/')` 或 `open('/etc/passwd')` ——如果不加限制，这就打开了潘多拉魔盒。

即便 Agent 本意无害，意外错误也会导致严重后果。比如脚本中拼接路径时少了一个点，从 `/project/output` 变成 `/output`，就可能写到系统关键分区。

因此，我们需要从工具层面就锁死可访问的目录范围。

## 做法：用 MCP filesystem 服务器锁死白名单

### 1. 了解 MCP filesystem 服务器

MCP 官方提供了一个 `@modelcontextprotocol/server-filesystem`，它能将文件系统操作暴露为 MCP 工具：`read_file`、`write_file`、`list_directory` 等。最关键的是，初始化时可以传入 `allowed_directories` 参数，该服务器内部会对所有路径做规范化校验，任何超出白名单的访问都会被拒绝。

### 2. 配置 OpenClaw 工具

在 OpenClaw 的 `openclaw.yaml`（或通过 UI 配置）中添加此 MCP 服务器：

```yaml
mcp_servers:
  filesystem:
    command: npx
    args:
      - "@modelcontextprotocol/server-filesystem"
      - "/home/user/projects/data-analysis"
      - "/home/user/projects/shared-scripts"
    env: {}
```

以上指定了两个允许目录。Agent 通过此工具能读写的路径必须位于 `data-analysis` 或 `shared-scripts` 下，其他位置如 `/etc`、`~/.ssh` 甚至其他项目目录都无法触及。

### 3. 替换掉原始的文件操作工具

为了真正实现隔离，你需要在 Agent 配置中**仅**提供上述 MCP 文件系统工具，而移除或禁用任何能绕过它的文件访问途径——比如不要再额外注册一个无限制的 `shell` 工具。如果必须保留 Shell，也要在 Shell 工具前置一层路径检查代理。

## 踩坑与排障

**踩坑1：相对路径陷阱**  
MCP filesystem 服务器会将所有传入路径解析为绝对路径，再检查是否以 `allowed_directories` 中某个条目开头。因此 Agent 传入相对路径 `../../etc/passwd` 会被 resolve 后拒绝。但前提是服务器正确实现了 resolve。务必使用最新版本，并尽量在配置中传入绝对路径。

**踩坑2：符号链接绕过**  
如果你的白名单目录下包含指向外部的符号链接（如 `ln -s /etc ./config`），Agent 可能通过它访问到系统文件。官方服务器默认不跟随符号链接跨越允许目录边界，但仍建议主动删除项目中的符号链接，或者启用服务器的 `read_only` 模式进行防御。

**踩坑3：Windows 路径大小写与盘符**  
在 Windows 下，需要特别注意盘符一致性和大小写。建议始终使用小写盘符且统一使用正斜杠格式，比如 `C:/projects`。实测如果配置时写 `C:\Projects`，但实际路径是 `c:\projects`，可能因为大小写不一致导致校验失败。

## 可复用的工程建议

- **最小权限原则**：只给 Agent 它完成任务绝对需要的目录，不要贪图方便开放整个 home。
- **分层分离**：只读目录（如参考数据）可以用 `read_only` 模式挂载，进一步降低风险。
- **审计日志**：记录每次文件工具调用的参数，方便回溯 Agent 行为，可通过 OpenClaw 的跟踪或自定义日志钩子实现。
- **定期审查白名单**：随着项目迭代，移除不再使用的目录，避免权限蔓延。
- **结合环境隔离**：白名单文件控制可与 Docker 或 Firejail 结合，构成纵深防御。对于高风险操作，仍然推荐完整沙箱。

## 总结

给 Agent 的自动化脚本加本地目录白名单，本质上是在工具层面实施最小权限原则。借助 MCP 的 filesystem 服务器，我们仅需几行配置，就能让 Agent 乖乖待在我们划定的“操作区”内，即便它尝试越权也会被底层安全校验阻拦。

这种做法既不过度加重工程负担，又能在日常开发辅助、数据分析、文档生成等场景中显著降低误操作和恶意利用的风险。下次当你准备给 Agent 开放文件能力时，不妨先问自己一句：“这个目录真的需要全盘可见吗？”

---

