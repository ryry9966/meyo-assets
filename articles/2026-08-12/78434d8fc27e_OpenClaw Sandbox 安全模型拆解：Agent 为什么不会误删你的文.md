---
title: OpenClaw Sandbox 安全模型拆解：Agent 为什么不会误删你的文件
feedId: 32790
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：当 Agent 可以执行 Shell 命令

在 OpenClaw 这类支持 MCP 或插件化工具调用的 Agent 框架里，最常见的需求之一是让模型调用本地命令行、读写文件、执行脚本。这带来了一个绕不开的问题：一段由 LLM 生成的指令，万一包含了 `rm -rf /` 或 `DROP TABLE` 级别的破坏性操作，安全边界在哪里？

OpenClaw 给出的答案是 **Sandbox（沙箱）安全模型**——并非单纯通过提示词约束，而是基于文件系统隔离和权限白名单的工程化手段，让 Agent 在受控环境中操作，即使出现“幻觉”也不会删掉宿主机关键文件。

## 问题：传统“伪隔离”的局限

很多早期 Agent 尝试使用 `os.system` 或 `subprocess` 直接执行命令，仅在 prompt 中告知“不要删文件”。这种方式有两个致命弱点：

1. **LLM 不可信**：复杂任务中，模型可能生成路径遍历、变量拼接错误，甚至被注入恶意提示。
2. **无最小权限**：一旦 agent 进程以当前用户身份运行，所有用户可写的文件都对它敞开，包括 SSH 密钥、项目源码、配置等。

而容器的原生隔离又太重，不符合本地工具链轻量化、直接操作用户代码仓库的诉求。OpenClaw 的 Sandbox 试图在“安全”和“便利”之间找到切面。

## Sandbox 模型的做法：三层边界

从工程实现角度看，OpenClaw 的 sandbox 并非一个完整的虚拟机，而是组合了三个层面的约束：

### 1. 文件系统挂载白名单
Agent 启动时，通过配置项指定允许读写的目录白名单，例如：
```yaml
sandbox:
  allowed_write_paths:
    - ./workspace
    - /tmp/openclaw
  allowed_read_paths:
    - ./workspace
    - /tmp/openclaw
    - ./config
```
底层利用 `chroot`、`mount namespace` 或 Windows 的 `SetFileSecurity`，确保 Agent 进程视角下只能看到白名单中的路径。任何对 `/etc/passwd`、`~/.ssh` 的访问都会被文件系统层直接拒绝，而不是抛给 prompt 判断。

### 2. 操作权限分级
OpenClaw 将文件操作分为只读和读写两类。MCP 工具中的 `read_file`、`list_directory` 自动工作在只读域；而 `write_file`、`delete_file` 必须显式声明在 `allowed_write_paths` 内。更危险的操作（如执行 shell 命令）需要额外开启 `exec` 权限，并可限制允许的命令列表（如只允许 `git`、`python`）。

### 3. 删除操作的“软硬”策略
针对 `delete_file` 和脚本中的 `rm`，sandbox 提供了三种模式：
- **模拟模式**：文件被移动到回收站目录（如 `.openclaw/trash`），不实际从磁盘擦除，可事后恢复。
- **审核模式**：所有删除动作生成一条待确认记录，用户可通过 UI 或 CLI 逐条放行。
- **直接拒绝**：设为只读域时，任何删除系统调用都会返回 `EPERM`。

实践证明，**模拟 + 审核** 组合在生产中误删率几乎为零，即使 Agent 生成了错误的删除指令，也只是移动了文件，用户可以轻松找回。

## 实操步骤：配置一个安全的 Agent 工作区

以本地开发环境为例，我会给一个 Python 项目 Agent 配上最小权限 sandbox：

1. **创建工作区隔离目录**
   ```bash
   mkdir -p ~/openclaw-sandboxes/project-alpha/{workspace,tmp}
   ```

2. **编写 sandbox 配置文件**
   ```yaml
   sandbox:
     root: ~/openclaw-sandboxes/project-alpha
     allowed_write_paths:
       - /workspace
       - /tmp
     allowed_read_paths:
       - /workspace
       - /tmp
       - /config   # 只读配置
     exec:
       enabled: true
       allowed_commands: ["python", "pip", "git"]
     delete_policy: simulated
   ```

3. **启动 Agent 时挂载此 sandbox**
   OpenClaw CLI 会自动将项目目录绑定挂载到 sandbox root 下的 `/workspace`，并将 HOME 环境变量重定向到 `/tmp`，避免污染用户真实 home。

这样 Agent 看到的文件系统类似于：
```
/sandbox-root/
├─ workspace/   # 宿主项目目录
├─ tmp/
└─ config/      # 只读挂载
```
即使它执行 `rm -rf /*`，也只能清空 workspace 和 tmp 内的文件，不会触及系统文件。

## 踩坑与经验

在实际使用中，有两个地方最容易翻车：

- **白名单路径没包含日志目录**：Agent 调用工具链时往往会产生日志、缓存。如果未将 `./logs` 加入 `allowed_write_paths`，调试难度会急剧上升。建议一开始就预留 `/tmp` 作为临时输出区。
- **模拟删除导致磁盘膨胀**：`simulated` 策略会把删除文件移动到回收站，长时间运行后 `.openclaw/trash` 可能很大。可以设置定期清理任务，或切换为 `audited`，在确认后真正删除。

## 可复用建议

1. **永远显式声明白名单**，即使测试环境也不要全局开放。
2. **生产环境采用审核模式**，所有危险操作（删文件、执行不可信脚本）需人工确认。
3. **结合 MCP 权限描述**，在每个工具注册时声明所需的最小路径、权限，OpenClaw 会在调用前自动比对 sandbox 配置，避免遗漏。
4. **定期审计 trash 目录**，发现模式化错误训练提示词或微调模型。

## 总结

OpenClaw 的 sandbox 安全模型背后是一个朴素但牢固的工程原则：**安全不靠 prompt，靠边界**。通过文件系统隔离、权限白名单和细粒度的删除策略，Agent 即使被误导或自身推理出错，也难以对宿主机造成实质性破坏。这种设计让开发者可以更放心地把创造性工作交给 LLM，同时把风险关在笼子里。

对于正在将 Agent 接入生产流水线的团队，sandbox 不应是选配，而应该是默认开启的基石配置。

---

