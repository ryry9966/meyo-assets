---
title: OpenClaw 沙盒安全模型深度拆解：为什么你的 Agent 不会误删文件？
feedId: 32629
source: 综合讨论
publishedAt: 2026-08-11
---

最近在社区看到不少关于“Agent 误删宿主文件”的讨论，甚至有人真的因为一次错误的工具调用丢了项目代码。大家开始关心一个问题：**跑在本地的 OpenClaw，会不会也犯这种错？**

答案是不会，前提是你没有主动关掉它的安全机制。我想通过这篇文章，把 OpenClaw 的 sandbox 模型拆开说清楚，顺便给出几个我在生产环境里踩过的坑和加固建议。

---

## 背景：失控的 Agent 有多危险

大部分 Agent 框架本质上是让 LLM 调用本地函数，比如读写文件、执行 Shell 命令。如果这些调用没有边界，只要模型“幻觉”一次，就可能导致真实的系统损坏。传统做法要么完全相信模型输出（玩火），要么只让 Agent 操作一个 Docker 容器（重且不方便本地调试）。

OpenClaw 走了第三条路：**在 Agent 运行时与操作系统之间插入一层轻量沙盒**，把危险操作拦截在用户态，而不需要拉起一个完整的容器。

---

## 问题：普通 Agent 为什么能删掉你的文件？

一个典型的 Python Agent 工具可能是这样的：

```python
def delete_file(path: str):
    os.remove(path)
```

LLM 生成什么路径，它就去删什么。你没法保证它不会传一个 `../../../../etc/passwd` 回来。

哪怕你做了 prompt 约束、加了“请确认”的步骤，黑盒模型依然可能绕过。所以真正可靠的方案是 **以系统级机制做强制性访问控制**，这就是 OpenClaw sandbox 的出发点。

---

## OpenClaw 的 sandbox 安全模型

核心思想是 **“默认拒绝 + 虚拟文件系统映射”**。它不依赖 Docker，而是在 Agent 进程启动时做三件事：

1. **挂载点隔离**：利用操作系统提供的文件系统命名空间（Linux 上是 `unshare` + bind mount，macOS 用沙盒扩展），把 Agent 能“看到”的文件树限制在一个白名单目录内，通常是你的项目 workspace。
2. **文件操作代理**：Agent 不直接调用 `os.remove`，而是通过 OpenClaw 内置的 `file_system` 工具发出请求。这个工具内部会做路径规范化，拒绝任何试图穿越 workspace 边界的操作（比如 `..` 逃逸、符号链接指向外部）。
3. **权限分级**：除了文件，Shell 工具也会被限制。默认配置下，Agent 只能运行一组预设的安全命令（`ls`, `cat`, `grep` 等），危险命令（`rm -rf /`, `chmod`, `curl pipe bash`）直接被拦截或转为 dry-run。

用一句话总结：**Agent 以为自己在操作一台完整的 Linux，实际上它活在 OpenClaw 为它定制的“鱼缸”里。**

---

## 做法/步骤：开启并验证沙盒

OpenClaw 的 sandbox 默认是开启的，但你最好显式确认一下配置。

1. 在 `openclaw.config.yaml` 中检查或设置：

   ```yaml
   sandbox:
     enabled: true
     mode: strict          # strict | permissive
     workspace: "./project"  # 必须是一个相对或绝对路径，不能是 "/"
     allowed_commands: ["ls", "cat", "head", "tail", "wc", "find", "grep"]
   ```

2. 启动时加 `--sandbox strict` 可强制覆盖。如果是开发调试需要临时放权，可以改成 `permissive`，会在危险操作前弹一个交互式确认，但**永远不要在生产环境用 permissive**。

3. 验证：写一个故意使坏的 prompt，让 Agent 尝试读 `/etc/shadow` 或删除上级目录。观察日志，你会看到类似这样的拦截记录：

   ```
   [Sandbox] BLOCKED: read /etc/shadow (outside workspace)
   [Sandbox] BLOCKED: delete ../../src (path traversal detected)
   ```

如果你看到了这些，说明沙盒已经在工作了。

---

## 踩坑点：看起来安全，实际可能漏成筛子

以下是我在实际部署中遇到过的问题，每一个都可能让沙盒形同虚设。

### 1. workspace 设成 "/" 或用户主目录
有的同学图方便，直接把 workspace 指向 `~` 或 `/home/user`，心想“反正我能访问的文件 Agent 都能操作，方便”。这等于直接废掉了沙盒，因为 workspace 内包含所有项目、配置文件、SSH 密钥……**必须把 workspace 严格限制到单个项目根目录**，比如 `./my-agent-project`，不要包含任何上级敏感数据。

### 2. 第三方 MCP 插件绕过沙盒
OpenClaw 支持通过 MCP 协议加载外部工具。如果你加载了一个没有经过审计的插件，它可能在内部直接调用 `eval()` 或 `os.system()`，完全不走 OpenClaw 的文件代理。目前的缓解手段只有两个：① 只用官方签名过的插件；② 把整个 OpenClaw 进程放进 Docker 做外层隔离。后者虽然重，但在运行不受信 MCP 时是必须的。

### 3. Windows 上的权限坑
Windows 的沙盒依赖 Job Objects 和完整性级别控制。如果你以管理员权限运行 OpenClaw，Agent 进程可能会继承高权限，导致部分限制失效。**在 Windows 上一定要用普通用户运行，并确保工作目录不在系统盘敏感位置。**

### 4. 符号链接逃逸
即使 workspace 设置正确，如果项目内部有一个符号链接指向 `/`，Agent 跟随链接后也可能看到外部文件。沙盒的路径规范化逻辑会尝试解析真实路径并检查是否在 workspace 内，但如果你的文件系统比较复杂，仍建议在启动前用 `find -L $workspace -type l` 检查并删除可疑链接。

---

## 可复用建议：把安全做成习惯

- **最小权限 workspace**：每个 Agent 项目一个独立目录，绝不交叉。
- **双层隔离**：如果你需要运行不可信代码或插件，始终在 Docker/Podman 中再套一层 OpenClaw，哪怕沙盒漏了，外面还有一道墙。
- **只读挂载关键资源**：如果 Agent 需要读取一些全局数据（如 API key 文件），用操作系统的只读 bind mount 挂到 workspace 内，而不是让 Agent 直接访问原始路径。
- **审计日志**：永远开启 `sandbox.audit_log`，定期检查有没有被拦截的操作。拦截次数异常增多，往往是 prompt 被注入或模型行为异常的早期信号。
- **CI 红队测试**：在 CI 中加入一条“红队测试用例”，故意让 Agent 执行越权操作，确保构建失败。Sandbox 配置改变时，这个用例能第一时间报警。

---

## 总结

OpenClaw 的沙盒不是什么黑魔法，而是一层 **工程上足够实用、又不引入过高复杂度的安全护栏**。它让一个运行在本地的 Agent 能够安全地操作文件，而不会因为一次错误的推理毁了你的工作成果。但安全感的另一半来自使用者本身：如果你不正确地配置 workspace，或者盲目信任来路不明的插件，再好的沙盒也白搭。

把 sandbox 当作最后一道防线，而不是唯一防线。保持“Agent 会犯错”的预期，你的自动化系统才能真正稳健地跑起来。

---

