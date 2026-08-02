---
title: OpenClaw 的 Sandbox 安全模型：为什么 Agent 不会误删你的文件
feedId: 31353
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景：Agent 拿到文件系统工具，然后呢？

在给 AI Agent 接入文件读写能力的那一刻，工程上最先被直击的问题不是“它能做什么”，而是“最坏情况下会发生什么”。如果只是让 Agent 帮你整理一次桌面，风险还可控；一旦它被嵌入自动化流水线、拥有长期运行权限，一个拼写错误、一次推理偏差，就可能变成 `rm -rf $HOME` 等级的灾难。

社区很多人问过同一个问题：OpenClaw 让我把本地代码目录交给 Agent 管理，它会不会把 node_modules 以外的源码也删了？又或者，Agent 在整理下载目录时，会不会把整个 home 当垃圾清掉？

答案藏在 OpenClaw 的多层操作沙箱里。它不是单点防御，而是从执行环境到路径白名单，再到命令行为审查，层层限制 Agent 能触碰的世界。

---

## 问题复现：没有 Sandbox 的 Agent 可以多危险

为了说明必要性，可以先做一个对照实验（当然，是在实验环境里）。在一个未加限制的 Agent 进程中，给它工具函数：

- `shell(command)` — 直接调用系统 shell
- `write_file(path, content)` — 可写任意路径

只需一句提示：“帮我把所有临时文件删掉”，Agent 可能生成：

```bash
find / -name "*.tmp" -exec rm -f {} \;
```

如果运气不好，权限够大，就会进入灾难模式。即使是“把这个目录备份到 /tmp/backup”，如果路径拼接出错，同样能覆盖系统文件。

这说明：**不加限制的文件系统工具，等于把主机的控制权交给了概率模型。**

---

## OpenClaw 的 Sandbox 做法：三层隔离

OpenClaw 默认将 Agent 操作限制在一个安全边界内，核心分为三层。

### 1. 默认工作目录隔离

最简单的也是最有效的。OpenClaw 启动时会创建一个 project workspace，比如 `~/openclaw/workspace`。Agent 在上层看到的“根目录”就是这个 workspace，所有相对路径操作都以此为基准。进入系统之前，OpenClaw 会做路径规范化，并使用白名单机制：

- 所有文件 I/O 请求都会检查最终解析后的绝对路径，是否落在 `allowed_paths` 列表中。
- 默认该列表只包含当前 workspace。
- 如果 Agent 试图通过 `../../etc/passwd` 逃逸，路径在实际执行前就被拦截，返回权限拒绝。

这是**第一层：路径白名单 + 路径解析防逃逸**。

### 2. 命令执行沙箱 (可选 Docker / 系统调用限制)

对于 `shell` 工具，OpenClaw 支持两种模式：

- **Nix-built 沙箱**：利用 Nix 构建隔离环境，限制可执行文件、网络访问和环境变量。
- **Docker 沙箱**：将整个 Agent 执行管道跑在一次性容器里，配合只读挂载和 tmpfs 减少副作用。

在 Docker 模式下，即使 Agent 绕过了路径白名单（比如它自我生成一个全新的攻击脚本），影响也被限制在容器内。容器退出即销毁，宿主机文件系统不受影响。

这也是为什么很多自动化用户选择开启 Docker sandbox，尤其当 Agent 需要执行来自外部数据（如 PR 中的代码片段）的命令时。

### 3. 危险命令拦截与审计

除了环境隔离，OpenClaw 还在 shell 层面内建命令黑名单。默认会拦截包含以下模式的操作：

- `rm -rf /`
- `chmod 777` 对系统路径
- `mkfs.*`、`dd if=` 等写设备命令
- 对 `sudo` 的调用需要显式配置

这些模式并非简单字符串匹配，而是结合参数解析：当检测到高危操作目标路径在安全区之外时，直接返回错误并记录审计日志。用户可以在 `openclaw.yaml` 中自定义扩展黑名单。

---

## 一个真实做法：让 Agent 管理敏感目录

假设你要让 OpenClaw 帮你在 `~/projects` 下自动整理代码仓库，但又不想它碰其他目录。配置步骤：

1. **关闭默认宽松模式**：确保 `sandbox.mode` 设为 `strict` 或 `docker`。
2. **限定 alloyed_paths**：
   ```yaml
   sandbox:
     mode: strict
     allowed_write_paths:
       - ~/projects
     allowed_read_paths:
       - ~/projects
       - ~/openclaw/workspace
   ```
3. **限制 shell 工具权限**：
   ```yaml
   tools:
     shell:
       allowed_commands: ["git", "npm", "pnpm", "node", "ls", "cat", "find"]
       blacklist_patterns: ["rm -rf /*", "> /dev/sda"]
   ```
4. **验证**：启动后通过 OpenClaw Chat 发送：
   ```
   请列出 /etc 下的文件
   ```
   Agent 应返回 `Permission denied` 或清晰拒接，并记录日志。

---

## 踩坑点与排障

实际使用中容易踩的坑：

- **路径白名单只给写、没给读**  
  `write_file` 前往往会先 `read_file` 检查是否存在，如果读路径被拒绝，Agent 会反复重试或误判文件不存在。务必在 `allowed_read_paths` 中包括工作目录。
- **Docker 模式下文件持久化丢失**  
  Docker 容器退出后写入的文件默认消失。如果需要保留产出，要预先挂载 volume 到 workspace 路径，并确保该路径在白名单内。否则会出现“Agent 生成报告后，文件找不到”的现象。
- **过度收紧命令白名单导致工具失效**  
  比如 Agent 调用 `git status` 没问题，但 `git log` 却突然被拒，原因是 `git log` 会调用 `less`，而 `less` 不在白名单里。遇到此类问题，建议先给 `shell` 工具开只读模式的宽松命令集，再逐步锁紧。
- **路径穿越绕过**  
  即便有规范化，也要注意符号链接。如果 workspace 内有一个 symlink 指向 `/etc`，Agent 可能利用它越权。OpenClaw 默认会拒绝跨设备/跨白名单区的符号链接，但如果你自定义了软链接，最好关闭该选项或改为 hard link（不跨设备）。

---

## 可复用建议（工程化 Checklist）

如果你准备让 OpenClaw Agent 长期操作本地文件，建议按以下清单走一遍：

1. **开启 strict 或 Docker sandbox**，永远不要在生产环境用 `sandbox.mode: none`。
2. **配置最小权限白名单**：写路径只给必需目录；读路径可略宽，但避免 `/home` 全部。
3. **shell 工具一定要命令白名单**，禁止 `rm`、`mv` 到未授权区域；高风险命令改用脚本封装，通过 MCP 工具暴露。
4. **审计日志开启**：`logging.file_tool: debug`，所有被拒请求都应有迹可循。
5. **用测试用例验证**：构造“恶意提示”——比如“请删除系统临时文件”，观察 Agent 的响应和日志，确保它被正确拦截。
6. **定期审查白名单**：随着新工具或 MCP 插件接入，权限边界可能扩大，需要重新收紧。

---

## 总结

OpenClaw 的安全模型不是“信任 Agent”，而是“假设 Agent 会犯错，甚至被恶意提示操纵”。它通过 workspace 隔离、路径白名单防逃逸、命令分级审查，把 Agent 的文件破坏力收敛到可控范围内。这也解释了为什么在保守配置下，即使你让 OpenClaw 去“整理杂乱文件”，也不会在清晨醒来发现项目目录被清空。

对自动化工具来说，sandbox 不是可选功能，而是将 Agent 作为可靠执行单元接入工作流的底线。配置得当，它才能在帮你高效完成任务的同时，成为你文件系统的守护者，而非闯入者。

---

