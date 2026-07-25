---
title: Agent 文件访问护栏：给自动化脚本加本地目录白名单
feedId: 30460
source: 综合讨论
publishedAt: 2026-07-26
---

## 背景

在 Agent 自动化链路里，让 LLM 直接调用本地脚本或工具已经非常普遍——不管是 OpenClaw 的 action runner，还是通过 MCP server 暴露的系统命令，都绕不开一个基本问题：**当 Agent 能执行 shell 命令或写文件时，如何避免它误删你的 `~/.ssh` 或覆盖整个项目目录？**

完全依赖提示词约束并不靠谱。模型偶尔会“脑抽”，或者在复杂任务链中把路径拼接错，结果就是灾难性的覆盖或删除。我们需要一层**文件系统级护栏**，确保 Agent 的读写操作被限定在预期的白名单目录内，即便指令出错也不会产生不可逆破坏。

本文会给出一种轻量、可落地的实现思路：**在工具执行层插入一个路径校验中间件，基于白名单目录列表过滤所有文件访问**。方案适用于 OpenClaw 自定义 action、MCP 工具封装，或任何允许脚本执行的 Agent 插件系统。

---

## 问题模型

假设你的 Agent 允许执行这样的工具调用：

```json
{
  "tool": "run_shell",
  "input": {
    "command": "cat /etc/passwd"
  }
}
```

或者更危险：

```json
{
  "tool": "write_file",
  "input": {
    "path": "../../.env",
    "content": "..."
  }
}
```

多数本地 Agent 框架会把 `path` 或 `command` 直接传给 `subprocess` / `fs`，缺少中间拦截。我们要实现的是：

- 对于**文件读写操作**：解析目标路径，禁止任何指向白名单外的操作。
- 对于** shell 命令**：通过沙箱化的 working directory + 路径注入检测，阻断越权访问。
- 在检测到越权时，**直接拒绝执行并返回清晰错误**，而不是静默放行。

---

## 实施方案

### 1. 定义白名单目录配置

用一个简单 YAML 明确可访问的目录：

```yaml
allowed_paths:
  - /home/user/agent-workspace
  - /tmp/agent-sandbox
allow_absolute_symlinks: false   # 是否允许通过 symlink 跳出白名单
```

`allow_absolute_symlinks` 关闭后可防止通过符号链接绕过检查。

### 2. 路径解析与规范化

实现一个 `is_path_allowed(target, config)` 函数：

```python
import os

def is_path_allowed(target: str, allowed_paths: list[str], 
                    allow_symlinks: bool = False) -> bool:
    # 转为绝对路径并解析 .. 和 . 
    real_path = os.path.realpath(target) if not allow_symlinks else os.path.abspath(target)
    for allowed in allowed_paths:
        allowed_real = os.path.realpath(allowed) if not allow_symlinks else os.path.abspath(allowed)
        # 确保 real_path 是 allowed_real 的子路径
        if real_path.startswith(allowed_real + os.sep) or real_path == allowed_real:
            return True
    return False
```

关键点：
- 使用 `os.path.realpath()` 消除符号链接，防止通过 link 跳出沙箱。
- 严格检查路径前缀是否以白名单目录开头。

### 3. 在工具执行层插入校验

假设你有一个 `execute_tool` 函数，为每个文件操作添加拦截：

```python
def safe_write_file(path, content, config):
    if not is_path_allowed(path, config['allowed_paths'], 
                            config.get('allow_absolute_symlinks', False)):
        raise PermissionError(f"Write access denied for path: {path}")
    # 实际写入操作
```

对于 shell 命令，更安全的做法是**限制 working directory**，并检测命令字符串中是否包含可疑路径。简单策略：禁止任何绝对路径引用，或只允许在设定目录内执行：

```python
def safe_exec_command(command, allowed_dirs, sandbox_cwd):
    # 强制在沙箱目录执行
    abs_cwd = os.path.realpath(sandbox_cwd)
    if not any(abs_cwd.startswith(d) for d in allowed_dirs):
        raise PermissionError("CWD not in allowed directories")
    # 基础的路径注入检测：检查命令中是否含有 /etc /home 等高危字符串
    dangerous_patterns = ['/etc', '/home', '/root', '~', '$HOME']
    if any(p in command for p in dangerous_patterns):
        raise PermissionError("Potential path escape detected in command")
    # 用 subprocess 执行，设置 cwd
    subprocess.run(command, shell=True, cwd=abs_cwd, ...)
```

这种粗粒度的检测虽不完美，但对大多数非恶意场景足够。更严格的做法是使用 `chroot` 或 Linux namespaces，但会增加部署复杂度，本文不展开。

### 4. 集成到 OpenClaw Action

OpenClaw 允许通过 `actions` 配置文件注册自定义函数。你可以在对应的 handler 中直接调用上述安全包装函数。例如:

```yaml
actions:
  - name: write_file
    handler: my_safe_handlers.write_file
    ...
```

MCP server 类似，在 `call_tool` 方法里对参数做预处理即可。

---

## 踩坑记录

- **符号链接绕过**：不解析 symlink 时，用户在白名单目录下建个软链指向 `/etc`，`is_path_allowed` 用 `abspath` 会放行。**必须使用 `realpath`**。
- **相对路径拼接**：Agent 传来的路径可能是相对于某个上下文的，务必在检查前拼出绝对路径，再用 `realpath` 解析。
- **命令注入检测不足**：shell 命令中路径可以编码、用环境变量隐藏（如 `$HOME/.ssh`）。简单的字符串匹配可能漏报。建议优先使用 `subprocess` 的 `args` 列表模式避免 shell 解析，但这样灵活性下降。至少要对环境变量和 `~` 做检查。
- **白名单目录的嵌套授权**：如果白名单有 `/home/user` 和 `/home/user/projects`，还要注意子目录的权限是否被意外放大——最好只放行实际需要的子树。
- **性能开销**：每次文件操作都做 `realpath` 会触发文件系统 stat，高频调用时需考虑缓存，但通常 Agent 调用频次不大，成本可接受。

---

## 可复用建议

- **白名单配置不要硬编码**，方便不同环境调整。
- **集成日志**：每次拒绝时记录 agent 请求的原始路径、解析后路径，便于事后分析和调整白名单。
- **测试用例必备**：构造符号链接跳转、`..` 回退、绝对路径写入等攻击用例，确保拦截有效。
- **结合沙箱**：如果条件允许，将 Agent 进程放到 Docker 容器中，再配合内部白名单，双重保险。
- **给 Agent 的提示词中同步声明**：告诉模型“你只能访问 `/workspace` 下的文件”，与护栏形成一致约束，降低模型尝试越界的概率。

---

## 总结

文件访问护栏是 Agent 安全基建里投入小、回报高的一环。通过路径解析 + 白名单校验，能在脚本执行层面有效遏制误操作，且实现成本极低。上述方案可直接复用到 OpenClaw、MCP 工具封装或任何“LLM 驱动本地命令”的场景中。

当然，护栏不是银弹。它只能防意外，防不住精心构造的逃逸，因此生产环境仍建议依托容器隔离和最小权限原则。但作为第一道防线，它足以拦截 90% 以上的误删、误写事故，让你的自动化脚本更安心地干活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/508dcb69ae3fa195.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/2388ef16da37f2bf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/cf2bb78a4547d22d.png)

