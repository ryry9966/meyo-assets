---
title: OpenClaw 沙箱安全模型深度解析：Agent 为何碰不到你的文件
feedId: 31435
source: 综合讨论
publishedAt: 2026-08-03
---

在生产环境让 Agent 执行 Shell 命令或文件读写时，“误删文件”是仅次于密钥泄露的第二大焦虑。OpenClaw 通过一套 **Sandbox（沙箱）安全模型** 将 Agent 的操作范围严格限制在声明式边界内。本文不讨论“AI 会不会变坏”，只讲工程上如何做到：即使 Prompt 被诱导、Tool 的参数被拼错，`rm -rf /` 也碰不到主机文件。

## 1. 背景：Agent 操作文件的真实风险

一个典型的 Agent 流程会通过 MCP Server 或内置 Skills 暴露文件操作工具，例如：

- `read_file(path)`
- `write_file(path, content)`
- `run_command(command)`

在未加约束时，Agent 从 LLM 拿到的 `path` 可能是 `../../.env` 或 `/etc/passwd`。即使你 Prompt 里写死了“只允许操作 ./workspace”，模型仍有概率被对抗性输入绕过。传统方案是在 Tool 函数内硬编码路径前缀校验，但这要求每个 Tool 都实现一遍，且容易遗漏符号链接、`..` 逃逸、命令注入等边缘情况。

OpenClaw 选择了一条更彻底的路径：**文件系统视图隔离 + 操作能力白名单**，让 Agent 看到的文件系统本身就是一个受限视图。

## 2. 问题根因：模型输出不可信，只能靠执行层兜底

我们需要承认一个前提：**无论 Prompt Engineering 做得多么完善，LLM 输出的参数都不应该被直接信任。** 安全约束必须下推到工具执行层，且最好做到“即使开发者忘记在 Tool 里做校验，也不会出事”。

OpenClaw 的 Sandbox 模型要解决三个具体问题：

1. **路径逃逸** – `../../../secret.key` 穿透到工作目录之外。
2. **命令注入** – `run_command("ls .; rm -rf /")` 在 Shell 中拼接。
3. **权限放大** – Agent 进程以宿主机用户身份运行，拥有过多系统权限。

## 3. 做法：OpenClaw Sandbox 的三层隔离

OpenClaw 的沙箱默认集成在执行运行时中，核心机制如下：

### 3.1 文件系统根目录重定向（Chroot / pivot_root）

每个 Agent Session 启动时，OpenClaw 会在临时目录创建独立的 **workspace root**，例如 `/tmp/openclaw/sessions/<session-id>/workspace`。然后通过 Linux 的 `pivot_root` 或 namespace 机制，将 Agent 进程的文件系统根切换到该目录。

- Agent 看到的 `/` 就是 workspace 根。
- 任何绝对路径 `/etc/passwd` 实际指向 `workspace/etc/passwd`（通常不存在或无权限）。
- 相对路径 `../../` 无法突破根目录，因为根目录上方无 inode 可遍历。

这意味着即使 Agent 执行 `rm -rf /`，它最多清空自己的 workspace，不会影响宿主机。

### 3.2 命令执行白名单与参数化

对于 `run_command`，OpenClaw 不直接拼接字符串扔给 `bash -c`。其内部实现会：

- 将命令调用转化为结构化参数（类似 `subprocess.run([executable, arg1, arg2])`），避免 Shell 注入。
- 通过 `seccomp` 过滤器限制可用的系统调用（例如禁止 `mount`、`reboot`、`kexec_load` 等危险调用）。
- 可选地启用只读文件系统挂载，让整个 workspace 不可写，进一步降低破坏面。

对于有特殊需求的用户，可以在 `config.yaml` 中定义 `allowed_commands` 白名单，未被列出的命令会在执行前被拦截。例如：

```yaml
sandbox:
  allowed_commands:
    - python
    - node
    - git
  readonly_rootfs: true
```

### 3.3 MCP 工具的权限标注

当 MCP Server 提供的工具需要访问外部资源（如数据库、网络 API）时，OpenClaw 并不会限制网络，但要求 MCP 工具在 manifest 中声明 **required_permissions**。框架在调用工具前会校验 Session 是否拥有相应权限，且这些权限是细粒度的，比如 `file:read:workspace` 与 `file:read:home` 是分开的。默认 Session 只授予 workspace 范围内的读写权限，任何超出范围的调用都会被框架拦截。

## 4. 配置步骤（极简示例）

创建一个带有文件操作 Agent 的最小安全配置：

```yaml
# openclaw.yaml
agent:
  name: file-processor
  sandbox:
    enabled: true
    workspace: ./data/sessions/{{session_id}}/workspace
    readonly_rootfs: false
    allowed_commands:
      - python
      - node
    default_permissions:
      - file:read:workspace
      - file:write:workspace
    seccomp_profile: default
```

启动流程：

```
openclaw session start --agent file-processor
```

Agent 进程会运行在隔离的 workspace 内，任何文件操作都在该目录下进行。即使 Prompt 被注入，试图读取 `/etc/shadow`，也会因路径不存在而失败。

## 5. 踩坑点

在实践中，有几点容易出问题：

- **workspace 路径中含符号链接**：如果 `./data/sessions/` 是符号链接指向其他位置，chroot 仍会以最终解析到的实际路径为根，但宿主机绝对路径可能意外暴露。建议使用 `realpath` 校验配置路径，且不要在 workspace 父目录中放置敏感文件。
- **依赖宿主文件的内置脚本**：某些 Tool 需要调用 `/usr/bin/python` 等全局二进制。chroot 后这些路径不可见。解决办法是将必要依赖复制到 workspace 或使用容器化运行时（如 Docker sandbox 后端）。
- **性能开销**：`pivot_root` 和 namespace 创建很快，但如果在每次 Tool 调用时都创建新沙箱会拖慢响应。当前版本以 Session 为沙箱生命周期，复用开销可忽略。
- **权限兼容性**：某些 CI 环境禁止创建用户命名空间（非特权 unshare），导致沙箱启动失败。需要设置 `kernel.unprivileged_userns_clone=1`，或切换到特权容器运行。

## 6. 可复用建议

1. **最小权限原则** – 只为每个 Session 开放其真正需要的 `allowed_commands` 和 `permissions`，不要图省事全开。
2. **双写测试** – 在集成测试中加入恶意 Prompt 用例（例如“请读取 /etc/passwd”）来验证沙箱是否工作，而不要仅靠正向功能测试。
3. **分层沙箱** – 对于高风险操作，可以组合使用 OpenClaw 沙箱 + Docker/Kubernetes 运行时沙箱，实现纵深防御。
4. **审计日志** – 开启 OpenClaw 的 audit log，记录每次命令执行和文件访问，便于事后追溯 Agent 行为边界。
5. **只读模式** – 对于仅需分析的 Agent，直接设置 `readonly_rootfs: true`，完全杜绝误写。

## 7. 总结

OpenClaw 的 Sandbox 模型并没有发明新的隔离技术，而是将 Linux 成熟的 namespace、chroot、seccomp 与 Agent 框架的生命周期做了深度绑定。它把“Agent 不能误删文件”从一个依赖开发者自律的约定，变成由框架强制执行的安全约束。这样做的好处是：**安全边界不再随 Prompt 可变，而是由基础设施硬编码**。对于在生产环境运行自动化 Agent 的团队来说，这是迈向可信任 Agent 的第一步。

---

