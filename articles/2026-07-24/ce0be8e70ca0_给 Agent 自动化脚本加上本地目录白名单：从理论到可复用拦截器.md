---
title: 给 Agent 自动化脚本加上本地目录白名单：从理论到可复用拦截器
feedId: 30299
source: 综合讨论
publishedAt: 2026-07-24
---

## 一、背景：Agent 的自由与风险

在 OpenClaw、MCP 以及各类插件化 Agent 实践中，我们时常把文件读写、命令执行这类能力交给自动化脚本。一个典型的场景是：通过 MCP Server 暴露一个工具，让 Agent 可以读取用户文档、写日志或生成报告到本地文件系统。问题在于，绝大多数脚本在实现时默认获得了整个文件系统的访问权限 —— 一个路径参数 `../../../.ssh/id_rsa` 就可能让攻击者或失控的 Agent 读取到本不该接触的敏感文件。

Agent 的安全边界不应依赖“它很乖”的假设，而要通过工程化手段把权限圈定在最小必要范围内。给本地文件访问加上目录白名单，是最低成本、最高收益的安保措施之一。本文面向那些在 Python/Node.js 中编写工具、MCP 插件或 OpenClaw 技能的开发者，介绍一个务实、可复用的实现方案。

## 二、问题定义

我们期望的效果很简单：Agent 通过某个工具写入或读取文件时，只允许它访问我们指定的白名单目录（比如 `/home/user/agent_workspace`）及其子目录。任何试图跳转到白名单外的路径，比如通过符号链接、相对路径回溯、`..` 遍历等，都应该被拦截并抛出异常。

这个需求看似容易满足，但实际操作中如果不处理路径规范化与符号链接，很容易留下绕过的缝隙。下面给出一个 Python 实现，MCP 工具可以直接内嵌或在中间层使用。

## 三、实现步骤

### 1. 定义白名单与核心检查函数

```python
from pathlib import Path
from typing import Union, List

class FileAccessError(Exception):
    """文件访问被拦截"""
    pass

class PathBlocker:
    def __init__(self, allowed_dirs: List[Union[str, Path]]):
        # 预解析所有白名单目录，确保使用真实路径 (resolve)
        self.allowed_dirs = [Path(d).resolve() for d in allowed_dirs]

    def validate(self, path: Union[str, Path]) -> Path:
        target = Path(path).expanduser().resolve()
        # 检查 target 是否在任一个 allowed_dir 内（包括子目录）
        if not any(
            target == allowed or allowed in target.parents
            for allowed in self.allowed_dirs
        ):
            raise FileAccessError(f"Access denied: {target} is not in allowed directories")
        return target
```

关键点：
- 使用 `resolve()` 消除所有的相对路径、`..` 以及符号链接，防止引用绕过。
- `target.parents` 包含所有父目录，可以确保 `/a/b/c/file` 在 `/a/b` 之内。
- 需要提前解析允许目录的真实路径，避免白名单本身是符号链接导致的不一致。

### 2. 安全封装文件操作

在需要暴露给 Agent 的工具函数中，用 `PathBlocker` 做入口检查：

```python
blocker = PathBlocker(allowed_dirs=["/home/user/agent_workspace"])

def agent_read_file(path: str) -> str:
    safe_path = blocker.validate(path)
    return safe_path.read_text(encoding="utf-8")
```

如果是写入操作，最好还要限制不能通过符号链接写回敏感区，上面的 `validate` 已经防范了。

如果工具需要支持多个目录白名单（比如一个读入目录，一个输出目录），分开初始化两个 `PathBlocker` 实例即可。

### 3. 集成到 MCP 服务器

在 MCP 工具的 handler 中直接调用验证逻辑：

```python
@server.tool()
async def read_local_file(path: str) -> str:
    safe_path = blocker.validate(path)
    # 业务代码...
```

如果多个工具都需要文件访问控制，可以将 `blocker` 实例封装成依赖注入或上下文变量，避免重复代码。

## 四、踩坑点实录

**1. 符号链接的真实路径**  
必须 `resolve()` 而不是 `absolute()`。`resolve()` 会递归解析所有符号链接并返回真实路径。如果只用 `absolute()`（仅拼接当前工作目录），攻击者可以创建一个指向 `/etc/passwd` 的符号链接放在白名单目录下，然后让 Agent 读取该链接，`absolute()` 解析出的路径仍在白名单内，从而绕过检查。

**2. 白名单目录本身是符号链接**  
`allowed_dir` 也要 `resolve()`，否则真实文件可能位于白名单之外，但被符号链接引入后，我们对 `target` 的检查会误判。例如 `workspace -> /mnt/external/data`，如果我们只检查 `target` 是否在 `/home/user/workspace` 内（未解析），而 `target` 的真实路径是 `/mnt/external/data/secret`，则被错误放行。预先 `resolve` 白名单目录可以保持一致。

**3. 相对路径的起始点**  
调用 `Path(path).resolve()` 的默认基准是当前工作目录。如果 Agent 修改了 `os.getcwd()`，可能导致意外的路径解析。最简单的办法是要求工具接口传绝对路径，或在 validate 里先转绝对路径再 resolve。

**4. 路径穿越攻击变体**  
测试用例应覆盖：`../../etc/passwd`、`/home/user/agent_workspace/../../../etc/passwd`、符号链接到根目录等。用 pytest 参数化验证，确保抛出 `FileAccessError`。

## 五、可复用建议

- **抽离成独立模块**：将 `PathBlocker` 做成一个独立的包或在项目 `utils/security.py` 中引用，任何需要文件访问的 MCP 工具、OpenClaw 技能都可复用。
- **配置驱动白名单**：通过环境变量 `SAFE_DIRS` 传入，逗号分隔，避免硬编码。
- **审计日志**：在拦截时记录尝试访问的原始路径、用户标识和时间，方便事后追溯。
- **Node.js 版本**：使用 `fs.realpathSync` + `path.resolve` 进行相似检查，核心逻辑可移植。
- **测试优先**：写一个简单的文件系统迷宫（包含符号链接、.. 回溯）放在测试夹具中，确保白名单逻辑真正稳固。

## 六、总结

给本地文件访问加上目录白名单并不复杂，却能把 Agent 的文件能力关进一个安全的沙箱。核心在于对路径的真实解析和一致的验证逻辑。在自动化流程和插件开放调用中，这种“最小权限”实现可以防止绝大部分无意或恶意的文件泄漏。推荐每个暴露给外部调用（无论是大模型、MCP 客户端还是插件系统）的文件接口，都封装一层这样的拦截器。

工程化的安全感，往往就建立在这样一行行小而确定的校验上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/c500275405d91cb4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/3be26811d02a129e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/ed3f4dc6ff53f095.png)

