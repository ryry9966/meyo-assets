---
title: Agent “越权”读写文件？给自动化脚本加一个目录白名单护栏
feedId: 28958
source: 综合讨论
publishedAt: 2026-07-14
---

## 背景：当 Agent 拥有了文件系统能力

在 OpenClaw / MCP 这类 agent 工程中，越来越多的工具被接入：文件读写、命令行执行、数据库操作…… 尤其是文件系统访问，几乎成了高阶自动化的标配。常见场景包括：

- Agent 帮你整理下载目录、重命名批量文件
- 自动汇总日志文件并生成报告
- 通过 MCP 的 `filesystem` 服务器让 LLM 直接操作工作区

这些能力在提高效率的同时，也引入了最直接的安全风险：**Agent 可能读到你本不应暴露的文件（如 `~/.ssh`、凭据文件），或者误写、删除关键目录下的内容**。即使 prompt 里写了“只允许在某个目录操作”，但 Agent 的工具调用并不理解什么是“禁区”，它看到的只是一个函数签名和一段参数。

## 问题：为什么“相信 prompt”是不够的

实际工程中，有两类典型风险：

1. **意外越界**：Agent 从“整理当前项目”开始，却因为链接或相对路径意外跑到了上层目录，删掉了 `../node_modules` 甚至还跑去了 `/etc`。
2. **提示注入导致的恶意行为**：外部内容（网页、邮件）中嵌入了指令，诱使 Agent 读取 `/etc/passwd` 并回传。

在工具层不加限制，就等于把文件系统的全部权限交给了 LLM 的“推理”。护栏必须做在代码层面，而不是仅在提示词里声明。

一个低成本、高收益的方案，就是 **本地目录白名单**：所有文件 IO 操作强制检查目标路径是否在预设的允许目录内，否则直接拒绝并记录。

## 做法：在工具层嵌入路径检查

我们以 Python 实现为例，给出一个可直接复用的轻量级方案。假设 Agent 的工具函数是 `read_file`、`write_file`、`list_directory` 这类常见封装。

### 1. 定义白名单与路径安全检查函数

```python
from pathlib import Path
import os

ALLOWED_ROOTS = [
    Path("/home/user/agent-workspace"),
    Path("/tmp/agent-sandbox")
]

def is_safe_path(user_path: str | Path, resolve: bool = True) -> bool:
    target = Path(user_path).expanduser()
    if resolve:
        # 解析符号链接和 .. 等相对路径，防止绕过
        try:
            target = target.resolve(strict=False)
        except Exception:
            return False
    else:
        target = target.absolute()
    
    # 任一白名单根目录是目标路径的前缀即放行
    return any(
        root.resolve() in target.parents or root.resolve() == target
        for root in ALLOWED_ROOTS
    )
```

这里注意：

- 使用 `resolve()` 移除 `..` 和符号链接，避免路径遍历绕过。
- `strict=False` 允许文件尚不存在（如准备写入），但仍要检查父目录。
- 检查时同时判断目标本身及其所有父目录是否命中白名单根目录，以适应 `list_directory` 深度遍历的场景。

### 2. 封装工具函数（以 `read_file` 为例）

```python
def safe_read_file(path: str) -> str:
    if not is_safe_path(path):
        raise PermissionError(f"Access denied: {path}")
    return Path(path).read_text(encoding="utf-8")
```

对于 `write_file`，同样在写入前调用 `is_safe_path`。对于 `list_directory`，则确保在白名单目录内列出，并过滤返回的相对路径。

### 3. 接入 MCP 服务器

如果你使用的是官方 `@modelcontextprotocol/server-filesystem`，它本身支持启动时指定允许的目录，例如：

```
npx -y @modelcontextprotocol/server-filesystem /home/user/allowed-dir
```

但很多自建 MCP 服务器或 OpenClaw 插件并没有这一层，这时可将上述检查函数作为 MCP 工具的中间件或装饰器，拦截所有文件路径参数并校验。

```python
def path_guard(func):
    def wrapper(path: str, *args, **kwargs):
        if not is_safe_path(path):
            raise PermissionError(f"Blocked by path guard: {path}")
        return func(path, *args, **kwargs)
    return wrapper
```

这样一来，无论 Agent 模型产生怎样的调用参数，都会被工具层截断。

## 踩坑点

即使加了白名单检查，依然有几个容易忽视的陷阱：

1. **符号链接与 mount 点**  
   `resolve()` 会跟随符号链接，但如果你允许了 `/workspace`，而该处挂载了一个外部分区，Agent 仍可能触及其他数据。最好检查挂载点，或者让白名单目录为独立卷。

2. **写入后的二次执行风险**  
   Agent 被允许写入 `/workspace/scripts`，然后配合命令执行工具运行刚写入的脚本。这时白名单限制不了子进程行为。**必须同时限制命令执行的允许路径和工作目录**，或者将执行隔离到容器中。

3. **部分工具绕过检查**  
   假如 Agent 还能调用 `os.system("cp /etc/passwd workspace/")`，那么文件访问护栏完全失效。只有所有可能产生文件操作的工具都套上安全带，护栏才有意义。梳理全部工具列表，逐一评估风险。

4. **相对路径与工作目录混淆**  
   如果 Agent 修改了当前工作目录，而后使用相对路径 `./config.yaml`，可能指向了白名单之外。检查函数应始终基于绝对路径，必要时显式记录并监控工具调用的当前工作目录。

5. **日志泄露路径信息**  
   在拒绝访问时抛出异常，导致日志中携带着攻击者试探的路径字符串。这可以接受，但要避免在错误消息中暴露实际存在但禁止访问的文件的差异，防止信息泄露。

## 可复用建议

- **配置化白名单**：将 `ALLOWED_ROOTS` 放在配置文件或环境变量中，不同任务启动不同沙箱。
- **抽象成工具装饰器或中间件**：所有文件类工具共用一个 `path_guard`，避免遗漏。
- **写测试用例**：对 `../`、符号链接、绝对路径和空格等做白盒测试，确保解析逻辑健壮。
- **结合审计日志**：每次拒绝都记录时间、调用参数和 Agent 上下文，方便回溯是 prompt 缺陷还是恶意试探。
- **最小权限原则**：即使白名单目录内，也限制 Agent 不可执行、不可修改白名单目录的权限位（如去掉写权限之外的执行位），进一步缩小风险面。

## 总结

给 Agent 的自动化脚本加一个本地目录白名单，属于典型的 **低成本防御性工程**。它不解决所有安全问题，但能把最常见的“误操作”和“低级越权”挡在门外。核心思路很朴素：**不要相信模型输出的路径参数，工具层面必须对任何传入路径做检查**。

对于 OpenClaw 社区的实践者，如果你的 Agent 开始长期运行、处理用户文件，这个护栏值得第一时间落地。结合容器或虚拟机级别的隔离，才能构建出真正可信的 Agent 运行环境。

希望这篇文章能帮你把手头的自动化脚本再用得稳一点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/e011cf37c855263c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/8222211090ba79de.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/cd7827bc59cafa56.png)

