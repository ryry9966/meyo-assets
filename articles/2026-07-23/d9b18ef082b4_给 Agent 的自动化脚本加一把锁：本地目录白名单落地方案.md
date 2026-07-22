---
title: 给 Agent 的自动化脚本加一把锁：本地目录白名单落地方案
feedId: 30129
source: 综合讨论
publishedAt: 2026-07-23
---

# 为什么你的文件访问护栏会在 Agent 场景失效

很多人给自动化脚本做文件访问控制时，习惯沿用传统应用那一套：检查路径是否包含 `../`、禁止绝对路径、过滤敏感关键字。在普通脚本里这套逻辑能跑通，但一旦接入 OpenClaw、MCP 或任意工具链里的 Agent，情况就变了。

Agent 会组合调用多个工具，一次任务里可能经过“读取文件 → 写入临时文件 → 移动文件 → 调用外部命令”的链路。如果只在单个工具入口做校验，中间步骤产生的临时文件或重命名操作很容易绕过最初的限制。更麻烦的是，LLM 生成的路径参数可能经过字符串拼接、环境变量展开或符号链接，仅做简单字符串匹配的护栏几乎形同虚设。

真实的工程需求是：给 Agent 配套的脚本执行器设定一个明确的本地区目录白名单，所有文件读写操作必须落在该目录内，即便中间有临时文件、软链接或路径穿越尝试，都不能逃逸。

# 从路径字符串匹配到目录树约束

第一版的直觉做法是在工具调用入口拦截路径参数，判断是否以白名单目录开头。比如：

```python
ALLOWED_DIR = "/home/user/agent-workspace"

def is_allowed(path: str) -> bool:
    abs_path = os.path.abspath(path)
    return abs_path.startswith(ALLOWED_DIR)
```

这在简单场景下可用，但`os.path.abspath`并不解析符号链接。如果`/home/user/agent-workspace/link`指向`/etc`，那么`/home/user/agent-workspace/link/passwd`会被判定为合法，实际上已经逃逸到系统目录。攻击面不止于此：路径分隔符差异、大小写敏感（取决于文件系统）、Unicode 等价字符都可能让字符串匹配失效。

因此，更可靠的方案是*解析真实路径后再判断*，同时保证所有写操作都在同一个挂载点或文件系统边界内完成。

# 实现步骤：一个可复用的白名单执行器

下面给出一个面向 OpenClaw 插件或自定义 MCP 工具的最小可行实现，Python 版本 3.10+，适用于 Linux/macOS，核心逻辑同样适合 Node.js/Go 实现。

**1. 定义白名单基础目录并解析真实路径**

```python
import os
from pathlib import Path

BASE_DIR = Path("/home/user/agent-sandbox").resolve(strict=True)
```

使用`Path.resolve()`可以解析所有符号链接并返回绝对路径，`strict=True`确保基础目录本身存在，避免配置错误。

**2. 通用的路径安全校验函数**

```python
def sandbox_path(user_path: str, base: Path = BASE_DIR) -> Path:
    # 1. 构造基础目录下的目标路径
    candidate = (base / user_path).resolve()
    # 2. 确保最终路径仍然在基础目录内
    try:
        candidate.relative_to(base)
    except ValueError:
        raise PermissionError(f"Access outside sandbox: {user_path}")
    return candidate
```

这里的关键是`candidate.relative_to(base)`，如果`resolve`后的路径已经跳出基础目录，会抛出`ValueError`，直接拒绝。

**3. 对读/写/执行操作统一套用同一校验入口**

不管是文件读取、写入、重命名还是调用子进程，所有涉及本地路径的工具实现都不能绕过`sandbox_path`。例如，在自定义的 MCP 工具中：

```python
@tool
def read_file(filename: str) -> str:
    safe_path = sandbox_path(filename)
    return safe_path.read_text(encoding="utf-8")
```

别在业务逻辑里直接拼接路径或再用一次`os.path.join`，所有路径的来源只能是`sandbox_path`的返回值。

**4. 阻断符号链接逃逸**

即便用了`resolve`，也要提防 Agent 先在白名单内创建符号链接指向外部，然后通过该链接读写。`resolve`会跟随链接并解析到外部真实路径，然后`relative_to`会直接拒绝，所以这个风险已经被第二步覆盖。需要额外注意的是*创建符号链接*这个操作本身：如果你的工具允许 Agent 自由创建符号链接，它可能构造出复杂的内部分布式链路，导致后续解析开销或逻辑漏洞。建议直接禁用创建符号链接的功能，或者在创建时进行严格目的路径校验。

**5. 配合临时文件与子进程**

当 Agent 需要调用外部命令（例如 `ffmpeg`、`pandoc`）并传递文件参数时，确保传递的路径也是经过`sandbox_path`处理后的安全路径。如果命令本身允许`--output`等参数指定输出目录，同样要校验该目录在白名单内。

另外，注意`/tmp`这类共享临时目录不在基础目录内，如果 Agent 写入`/tmp`，则违反隔离原则。正确做法是在基础目录内创建`tmp`子目录，并将`TMPDIR`环境变量指向它，让子进程继承。

# 踩坑记录

- **`os.path.realpath` 与 `Path.resolve` 差异**：`realpath`要求路径必须已经存在才能解析符号链接，如果 Agent 将要创建新文件，目标路径尚不存在，`realpath`无法解析不存在的最后一级。因此需要用`Path.resolve`在不存在的路径上也能正确解析目录部分的符号链接，或者先解析父目录再拼接文件名。
- **跨文件系统移动文件**：`shutil.move`在不同文件系统间会退化为复制加删除，这个操作涉及用户在源和目标上的权限。如果白名单目录和外部某些临时目录在不同挂载点，Agent 可能会用移动操作把外部文件移入白名单。解决方法是禁止`move`跨越文件系统，或者只允许在白名单内部移动。
- **Windows 兼容性**：如果你的 Agent 运行在 Windows 上，`Path.resolve`行为略有不同，且需要考虑盘符、长路径、大小写不敏感等因素。建议在 Windows 上同样做归一化处理并显式转换为小写进行比较。
- **竞争条件**：校验路径和白名单归属之间，如果 Agent 可以并行操作，可能出现 TOCTOU（time-of-check time-of-use）问题。在本地单用户场景通常风险较低，但如果多 Agent 实例共享同一个文件系统，需要用文件描述符操作（`openat`等）来规避。

# 可复用建议

- **把护栏写成一个独立的、无副作用的纯函数**，方便在多个工具模块间共用和单独测试。
- **对所有文件操作封装一层，而不是让开发者记住每个地方都要调校验**。例如提供一个`SandboxedFS`类，暴露`read_text`、`write_text`、`open`等方法，内部自动完成路径校验。
- **将白名单基础目录做成可配置项**，从环境变量或配置文件中读取，方便部署时调整。
- **日志记录所有被拦截的路径请求**，方便发现 Agent 意外行为或护栏本身配置过严。
- **与 OpenClaw 插件机制结合时，把文件访问能力收敛到一个独立的工具中**，而不是分散在多个工具里，降低审计成本。

# 总结

本地目录白名单是最基础但容易被低估的安全措施。在 Agent 工具链里，仅靠字符串匹配会留下大量缝隙，正确做法是解析真实路径、严格检查路径归属、禁用符号链接创建，并收敛所有文件操作到统一的安全入口。这套方案轻量、无外部依赖，可以嵌入到自定义 MCP 工具或 OpenClaw 插件的几行代码中，把文件逃逸的风险从“可能”降到“不可能”。

文件系统护栏不是银弹，还需要配合权限隔离、资源限制和审计日志形成纵深防御。但如果连这层基础校验都没有，再智能的 Agent 也可能因为一次意外的路径穿越把工作目录外的数据搅乱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/d28487c334be1c88.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/9f5765127e5f15a1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/4451366ded6bf879.png)

