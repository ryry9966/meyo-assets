---
title: Agent 文件访问护栏：给自动化脚本挂本地目录白名单
feedId: 29624
source: 综合讨论
publishedAt: 2026-07-19
---

## 问题背景

当 Agent 通过 MCP 或插件获得文件系统访问能力后，最危险的“自动化”不是跑偏任务，而是拿到一把不受限的文件读写钥匙。经典场景：你让一个任务助手整理下载目录，它却扫到 `~/.ssh` 并“好心”归档；或者一个看似无害的链接 `file:///etc/passwd` 被大模型当成普通文件读走。

现实中有不少开发者为了方便，直接把文件系统 MCP 的 `allowed directories` 设为 `/` 或用户家目录，认为“只要 prompt 里不写就不会访问”。这是典型的假设安全。要做的是**在系统层切断可能，而不是靠提示词拦车**。

## 核心做法：通过 MCP filesystem 配置白名单

以 OpenClaw 社区常用的 MCP 文件服务器为例，其核心配置是一个 JSON 对象，关键字段是 `args` 里的 `allowed directories`。一个最小安全配置如下：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/yourname/projects/agent-workspace",
        "/Users/yourname/projects/agent-sandbox"
      ]
    }
  }
}
```

这里只允许访问两个目录，其余路径对 Agent 透明，即便模型生成访问 `/etc` 的代码也会在 MCP 层被拒绝，返回 `Access denied`。

如果你的 OpenClaw 实例跑在 Docker 里，需要注意宿主目录映射。常见做法是把白名单目录通过 volume 挂载进容器，然后 MCP 服务器内部路径使用容器内路径，比如 `/workspace`。但要小心：**不要在容器内把宿主机根目录挂进去再设白名单为 `/workspace`，那样等于门户大开**。严格限制 volume 映射范围是基础。

## 进阶：读写分离与只读目录

很多自动化流程只需要部分目录可写，其余只需读取。MCP filesystem 本身不直接支持读写分离，但可以通过**启动两个 MCP 实例**来实现：一个挂载只读目录（通过修改源码或换用约束更细的包装），另一个负责可写操作。实际上，社区有方案基于 `server-filesystem` 做了一个 `readonly` 参数补丁，可以在 args 中追加 `--readonly` 标记某些路径。你的配置会变成这样：

```json
"args": [
  "-y",
  "@modelcontextprotocol/server-filesystem",
  "/workspace/output",
  "--readonly",
  "/workspace/input"
]
```

如果使用的 MCP 版本不支持，一个土办法是启动两层 Node 包装：外层利用 `fs.access` 做前置检查，再交给原服务器。我们团队实际项目中，对只读数据源（如模型权重、配置模板）映射进容器时直接用 `:ro` 挂载，双重保险。

## 踩坑实录

1. **路径符号链接绕过**  
   即便只允许 `/workspace`，如果该目录下存在指向 `/etc` 的符号链接，默认的 `allowed directories` 检查不会递归校真实路径。在 Linux 下需要确认服务器实现是否使用了 `realpath` 规范化。测试方法：在允许目录里故意建一个软链 `ln -s /etc config`，通过 Agent 尝试读取 `config/passwd`，观察是报错还是返回内容。安全做法：挂载时用 `--no-symlinks` 之类的挂载选项，或在 MCP 前加一层代理解析真实路径后比对白名单。

2. **相对路径与 `..` 穿越**  
   如果你传给 MCP 的路径是相对路径，底层可能基于当前工作目录解析，当 Agent 或运行时改变了 `cwd`，白名单可能失效。配置中务必使用绝对路径，并在测试时尝试 `../../` 类请求。

3. **Windows 盘符与大小写**  
   在 Windows 上，白名单 `C:\workspace` 和 `c:\workspace` 可能被视为不同路径。MCP 文件服务器不同版本处理不一致。建议统一使用小写盘符并做标准化。跨平台项目最好在 CI 里加入路径解析测试。

4. **MCP 服务器进程权限**  
   白名单控制了目录，但没限制进程执行 shell 命令。如果你另外接了终端执行 MCP，Agent 仍可能通过 `rm -rf` 损坏文件。要记住：文件白名单只防这条通道，安全需要多通道统一收敛。

## 可复用建议

- **最小权限起步**：新接入一个文件相关 MCP 时，先只开放一个空目录，确认功能后再逐步添加目录。  
- **审计日志**：在 MCP 服务器前面加一层日志代理，记录每次文件操作的请求路径、操作类型和时间。很多生产事故不是越权，而是“合法的自动删除”量太大。  
- **路径白名单与内容校验结合**：对于输出目录，除了白名单，还可以用文件扫描监控意外覆盖（比如系统配置文件重名）。  
- **把安全配置版本化**：将 MCP 配置和 Docker Compose 一同纳入版本管理，防止“线上临时放开权限忘记回收”。

## 总结

给 Agent 加文件访问护栏，本质是把运行时能力收敛到最小必要集合。目录白名单是最直接有效的一步，但仅靠它不够。结合读写分离、符号链接防护、审计日志和多通道约束，才能让自动化脚本跑得安心。在 OpenClaw 生态里，改造和加固一个 MCP 服务比想象中简单，成本远低于恢复被误删的目录。

下次在 `mcpServers` 里写下 `"/"` 之前，不妨想想：这个自动化，真的需要看到我的整个硬盘吗？

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/08018cd026ed27b5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/b8733d2778d601c5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/bcfdd041870d7734.png)

