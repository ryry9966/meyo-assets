---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 31530
source: 综合讨论
publishedAt: 2026-08-04
---

## 背景

把 Agent 接到自己的机器上跑自动化，第一反应通常不是"它能帮我做什么"，而是"它会不会把我家拆了"。尤其是当 Agent 拿到 shell 权限、文件读写权限之后，这种担忧是合理的——LLM 不是形式化验证工具，它生成命令时有可能出错、产生幻觉，甚至被 prompt injection 诱导执行危险操作。

OpenClaw 的 sandbox 机制解决的就是这个问题。它不是"尽量别让 Agent 干坏事"，而是从内核层面限制 Agent 能碰到的资源范围——即使模型真的生成了 `rm -rf /`，在 sandbox 里也只是一条被拒绝的系统调用。

## 问题

很多人在本地跑 OpenClaw 时，第一版配置往往是这样的：

```yaml
agent:
  sandbox:
    enabled: false
```

理由是"我在自己机器上跑，要什么沙箱"。这个想法非常危险。OpenClaw 的 Agent 不只是执行你手动敲的命令——它会在工具调用链中自主决策。一次 MCP 工具返回的异常内容、一个被污染的网页读取结果，都可能导致模型在下一次工具调用中尝试破坏性操作。

更隐蔽的风险是**路径逃逸**。如果你只是简单地把 Agent 的工作目录限制到一个文件夹，但没做系统调用级别的拦截，Agent 完全可以通过 `../../` 或者符号链接绕过目录限制。

## 做法：正确的 sandbox 配置

OpenClaw 的 sandbox 默认使用 **Landlock LSM**（Linux 安全模块），从内核层面约束进程的文件系统访问权限。核心配置如下：

```yaml
agent:
  sandbox:
    enabled: true
    filesystem:
      read: ["/usr/share", "/etc/ssl"]
      write: ["/tmp/workspace", "/tmp/cache"]
      execute: ["/usr/bin/python3", "/usr/bin/node"]
    network:
      allow: ["api.openai.com", "api.anthropic.com"]
```

关键点是：**给 Agent 的写权限尽量少**。大多数 Agent 任务只需要一个工作目录和临时目录的写权限，读权限也只给必要的系统路径。工具链（python、node）按需放行。

对于 MCP 插件，每个插件应该单独配置权限域：

```yaml
mcp_servers:
  fs-tool:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem"]
    sandbox:
      write: ["/tmp/workspace/project-a"]
      read: ["/tmp/workspace/shared-data"]
```

这里的设计思路是：**即使 MCP 服务端被注入恶意指令，它的权限边界也是明确的**——只能动 `/tmp/workspace/project-a` 这一个目录。

## 踩坑点

**坑 1：Landlock 需要较新的内核。** 如果内核版本低于 5.13，Landlock 不可用。OpenClaw 会回退到 seccomp 模式，但隔离强度会弱一些。排查方式：`uname -r` 确认内核版本，`dmesg | grep landlock` 确认 LSM 已加载。

**坑 2：不要把 Docker 当成 sandbox 的替代品。** 我之前在 Docker 里跑 OpenClaw，然后直接把宿主机的 `/home/user` 挂载进去。这等于完全绕过了 sandbox——Docker 的 volume mount 是宿主机权限直通。如果要用 Docker，挂载时同样要遵循最小权限原则。

**坑 3：符号链接攻击。** 如果 Agent 能在工作目录里创建符号链接指向外部路径，而 sandbox 没有处理符号链接解析，读操作可能逃逸。OpenClaw 新版本默认用 `openat2` 的 `RESOLVE_BENEATH` 标志处理路径解析，但如果自己实现 MCP 工具，要注意这一点。

**坑 4：网络白名单别开太宽。** 很多人的 sandbox 配置文件里网络策略是 `allow: ["*"]`，等于没有网络隔离。至少应该限制为 Agent 实际需要访问的 API 域名。

## 可复用建议

1. **配置模板化**：把 sandbox 配置抽成 profile，按任务类型复用。比如 `code-agent.yaml`、`research-agent.yaml`、`personal-assistant.yaml`，每个 profile 有独立的权限边界。

2. **审计日志是必需品**：开启 `audit_log: true`，记录 Agent 每次被拒绝的系统调用。这不只是排查问题用的——它能告诉你 Agent 实际想做什么，帮助你收紧或放宽权限。

```yaml
agent:
  sandbox:
    audit_log: "/var/log/openclaw-sandbox/audit.log"
```

3. **用 dry-run 模式验证权限**：OpenClaw 支持 `--dry-run` 启动参数。在此模式下，Agent 的所有工具调用只记录不执行。我建议你在调整 sandbox 配置后，先跑一轮 dry-run，确认没有误伤正常操作。

4. **配套测试用例**：维护一个小的 pytest 集合，模拟常见危险操作（删除系统文件、读 `/etc/shadow`、外连非白名单域名），验证 sandbox 配置确实拦截了它们。

## 总结

OpenClaw 的 sandbox 不是"安全插件"，而是安全模型的核心组成。它的设计哲学可以概括为三点：

- **默认拒绝**：没有显式授权的操作，一律拒绝。
- **最小权限**：按任务需要分配读、写、执行、网络权限，而不是"全给再收紧"。
- **隔离即审计**：每一次被拒绝的操作都是重要信号，帮你理解 Agent 的行为边界。

配置好 sandbox 之后，你实际上获得了一个更有用的 Agent——因为你知道它"做不到什么"，所以你可以更放心地让它去"尝试什么"。安全不是限制能力，而是让能力可以被信任。

---

