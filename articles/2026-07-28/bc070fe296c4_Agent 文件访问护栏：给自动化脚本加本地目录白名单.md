---
title: Agent 文件访问护栏：给自动化脚本加本地目录白名单
feedId: 30710
source: 综合讨论
publishedAt: 2026-07-28
---

## 背景

在 OpenClaw 这类 Agent 框架里，让大模型调用本地工具（尤其是文件系统操作）是高频需求。无论是读写配置文件、生成报告、整理下载目录，都绕不开对磁盘的访问。一旦放开文件操作权限，风险就来了：Agent 可能读取 SSH 私钥、覆盖系统配置、遍历用户目录，甚至被恶意 prompt 利用。给自动化脚本加上文件访问护栏，远比事后补救划算。

本文只聚焦一个动作：**用本地目录白名单约束 Agent 能触碰的路径，任何越界一律拒绝。** 面向已经或准备在 OpenClaw/MCP/插件体系下跑自动化的工程同学，提供可落地的实现与排坑经验。

## 问题拆解

“只允许访问 `/home/user/project` 目录”这句话看起来简单，但路径可以变着花样绕过去：

- **符号链接**：`/tmp/evil -> /etc`，然后工具访问 `/tmp/evil/passwd`。  
- **路径遍历**：`../` 拼接，例如 `project/../../etc/shadow`。  
- **相对路径**：工具接收相对路径 `data/../../.ssh/id_rsa`，解析后可能离开白名单。  
- **大小写 / 路径分隔符**：Windows 下 `PROJECT\..\Windows` 规则不同，Linux 下没这问题但符号链接更普遍。  

一个可靠的护栏必须做到：拿到任何路径字符串后，**先解析到真实绝对路径，再检查是否以白名单任一目录为前缀**。

## 做法 / 步骤

下面给一个 Python 实现，直接可用在 OpenClaw 的 Tool 函数或 MCP 工具脚本中。

### 1. 白名单配置

```python
import os

# 白名单可以来自环境变量或配置文件
ALLOWED_DIRS = [
    os.path.realpath("/home/user/project/data"),
    os.path.realpath("/home/user/project/output"),
]
```

使用 `realpath` 一次性地把白名单也解析掉，避免每次检查重复计算。

### 2. 核心校验函数

```python
def is_within_whitelist(path: str) -> bool:
    try:
        real_path = os.path.realpath(path)
    except (OSError, ValueError):
        # 路径非法或无权限，直接拒绝
        return False

    for allowed in ALLOWED_DIRS:
        # 确保 allowed 以分隔符结尾，防止前缀误判，如 /var/app 匹配 /var/app2
        if real_path == allowed or real_path.startswith(allowed + os.sep):
            return True
    return False
```

这里 `allowed + os.sep` 的方式避免 `/var/app` 错误放行 `/var/app2_bad`；同时用 `==` 单独处理目录本身。

### 3. 在工具函数中挂载检查

常见的做法是写一个装饰器，拦截文件路径参数：

```python
from functools import wraps

def file_guard(*path_args):
    """装饰器：对指定位置参数做路径白名单校验"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for idx in path_args:
                path = args[idx]
                if not is_within_whitelist(path):
                    raise PermissionError(f"Access denied: {path}")
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

使用示例（模拟 OpenClaw Tool）：

```python
@file_guard(0)  # 第一个参数 filepath
def read_file(filepath: str) -> str:
    with open(filepath, 'r') as f:
        return f.read()

@file_guard(0, 1)  # 源路径和目标路径
def copy_file(src: str, dst: str):
    ...
```

### 4. 接入 OpenClaw / MCP

如果你在用 OpenClaw 的插件机制，可以直接把上述装饰函数注册为 Tool。MCP 的 `server` 端写法也一样，在 `call_tool` 处理函数里做路径校验，不依赖外部 sandbox。

## 踩坑点

1. **符号链接陷阱**  
   `os.path.realpath` 会递归解析所有符号链接，得到最终实体路径。白名单里的目录最好也提前 `realpath`，避免白名单本身是符号链接而校验失败。但注意：如果白名单目录内有指向外部的符号链接，Agent 依然可以通过该链接跳出去——**解决办法是禁止创建符号链接或在文件操作前检查每一段路径的链接指向**。对于内部工具，通常可以接受白名单自身不含外部链接即可。

2. **路径拼接时忘记规范化**  
   比如用户传入 `base_dir + user_input`，若 `user_input` 含 `../`，拼接后路径可能越权。预防：先 `os.path.realpath(os.path.join(base_dir, user_input))` 再检查。

3. **Windows 盘符与大小写**  
   `os.path.realpath` 在 Windows 上会统一大小写，并通过 `\\?\` 前缀处理长路径，但对比时仍需注意盘符和分隔符。使用 `startswith` 前统一用 `os.path.normcase` 或直接依赖 `realpath` 的结果即可，因为 `realpath` 已做归一化。

4. **错误处理**  
   遇到 `PermissionError` 或 `FileNotFoundError` 时，上面的 `is_within_whitelist` 返回 `False`，防止通过抛异常来探测文件存在性。这对信息泄露防护很重要。

5. **临时文件目录**  
   有时 Agent 需要写到 `/tmp`，但 `/tmp` 是全局共享的。可以单独把 `/tmp/agent-xxx` 动态创建并加入白名单，用完清理。

## 可复用建议

- **配置化**：白名单目录列表从配置文件或环境变量读取，不同项目切换成本低。  
- **日志审计**：记录所有被拒绝的路径访问，便于发现误拦或攻击尝试。  
- **单测必备**：为 `is_within_whitelist` 写一套测试用例，覆盖符号链接、`../`、大小写、UNIX socket 等边界。  
- **结合容器**：如果 Agent 运行在 Docker 内，可叠加 `read-only` 根文件系统和 `tmpfs` 进一步减小攻击面，护栏作为纵深防御的一层。  
- **工具清单梳理**：不只是文件读写，解压、移动、重命名、第三方库调用等任何可能产生文件操作的入口都要过护栏，建议统一用内部封装好的 I/O 模块。

## 总结

文件访问护栏不是一个新概念，但在 Agent 自动化的链条里经常被忽略。用不到 50 行代码就能在工具层加一层目录白名单校验，有效避免 prompt 注入导致的数据泄漏。核心思路就两步：解析真实路径、前缀匹配。细碎的坑在于符号链接、路径拼接和异常处理。工程落地后，Agent 的自由度没有明显下降，但安全性提升了一大截——这正是自动化演进中最该早做的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/2d953d8c9d185688.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/209e2220193ebcd8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/b3bdfff213812b6e.png)

