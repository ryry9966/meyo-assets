---
title: Agent 脚本防误删：用目录白名单给文件访问加护栏
feedId: 29598
source: 综合讨论
publishedAt: 2026-07-19
---

## 背景

Agent、MCP 工具、自动化插件越来越多地获得本地文件读写能力。一个典型的场景是：你让 Agent 帮你整理某个项目目录下的 Markdown 笔记，它却把你家目录下的 `.ssh` 一并扫描甚至改动了——因为 prompt 里“整理”两个字的边界太模糊，而模型在执行时会倾向于扩展搜索范围来提升“助人性”。

在没有护栏的情况下，一次大模型的自由发挥 + `os.remove()` 就可能酿成灾难。常见的保护手段（如 Docker 沙箱、虚拟机）太重，不适合轻量的 OpenClaw 插件或本地脚本。更务实的做法是：**为每个脚本实例设置一个明确的本地目录白名单，所有文件 I/O 都强制限制在该范围内**。

本文给出一个轻量级、可复用的实现方案，适用于 Python 生态的 Agent 工具 / MCP 服务器 / 自动化脚本。

## 问题定义

我们需要一个 **路径护栏**，具备以下能力：

1. 允许读写的目录由配置显式声明（白名单）。
2. 任何传入的文件路径，无论绝对、相对、包含符号链接、`..` 穿越，最终解析后的真实路径必须在白名单子目录之内。
3. 性能开销极小，不引入额外的 I/O（仅在必要时做一次 `realpath`）。
4. 集成成本低，能够以装饰器或上下文管理器的形式包裹现有函数。

看似简单，实际踩坑点主要来自路径规范化：相对路径基于当前工作目录，符号链接可能跳转到白名单之外，Windows 盘符与大小写等。如果护栏本身逻辑有漏洞，等于没加。

## 实现步骤

### 1. 目录白名单配置

建议用环境变量或 JSON 配置文件，避免硬编码：

```python
import json
import os

def load_whitelist():
    path = os.environ.get("AGENT_WHITELIST_CONFIG", "whitelist.json")
    with open(path) as f:
        data = json.load(f)
    return [os.path.abspath(os.path.expanduser(d)) for d in data["allowed_directories"]]
```

`whitelist.json` 示例：

```json
{
  "allowed_directories": ["~/notes", "./project_data"]
}
```

### 2. 路径安全检查函数

核心逻辑：先规范化路径，再检查每个白名单目录是否为其实前缀。

```python
import os

def is_path_allowed(user_path: str, allowed_dirs: list[str]) -> bool:
    # 展开 ~，转为绝对路径，并解析符号链接
    abs_path = os.path.realpath(os.path.abspath(os.path.expanduser(user_path)))

    for allowed in allowed_dirs:
        # 确保 allowed 也是规范化路径
        allowed_real = os.path.realpath(allowed)
        # 公共前缀比较时尾部分隔符要一致，防止 /data/app 匹配到 /data/app2
        # 简单做法：确保 allowed_real 以分隔符结尾再比较
        if abs_path == allowed_real:
            return True
        # 检查是否为子路径
        if abs_path.startswith(allowed_real + os.sep):
            return True
    return False
```

### 3. 装饰器保护关键操作

可以用装饰器拦截文件路径参数，假设函数第一个位置参数是文件路径：

```python
import functools

def path_guard(allowed_dirs):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(file_path: str, *args, **kwargs):
            if not is_path_allowed(file_path, allowed_dirs):
                raise PermissionError(f"Access denied: {file_path} is outside allowed directories.")
            return func(file_path, *args, **kwargs)
        return wrapper
    return decorator

# 使用
allowed = load_whitelist()

@path_guard(allowed)
def read_file_safe(path):
    with open(path) as f:
        return f.read()
```

对于 `shutil.move`、`os.remove` 等支持多路径操作的函数，需要扩展装饰器或显式调用检查函数。工程上更推荐将检查函数作为断言插入每个文件操作的入口，而非依赖装饰器，因为参数位置可能不固定。

### 4. MCP 工具集成

如果你的工具是一个 MCP 服务器（例如 `mcp` Python 包），可以在每个 `@server.tool()` 函数内部开头统一调用：

```python
@server.tool()
def move_file(source: str, dest: str) -> str:
    if not (is_path_allowed(source, allowed) and is_path_allowed(dest, allowed)):
        return "Error: path not allowed."
    shutil.move(source, dest)
    return "ok"
```

这样护栏代码与业务逻辑分离，审计也方便。

## 踩坑点

### 符号链接穿透

`os.path.realpath()` 能够解析所有符号链接，这是必需的。但如果你的白名单目录本身包含符号链接（例如 `~/notes` 实际是一个软链到 `/mnt/external`），需要将 `allowed_real` 也 `realpath` 化，否则比较可能失败。

### 当前工作目录变化

Agent 脚本可能在运行过程中改变 `os.getcwd()`。因此不要在模块导入时就一次性求出所有允许目录的绝对路径，最好在每次检查时实时计算 `os.path.realpath(os.path.abspath(...))`。为了性能，可以用 `lru_cache` 缓存规范化后的允许目录（假设目录不会在运行时变动）。

### Windows 兼容

Windows 上盘符大小写、路径分隔符、长路径前缀 `\\?\` 会让前缀匹配失效。可以用 `os.path.normcase()` 做大小写不敏感比较，并统一用 `os.sep`。如果你的工具只运行在 Linux 环境，可以忽略。

### “创建新文件”场景

护栏不仅要检查读路径，也要检查写路径。当 Agent 写入一个新文件时，父目录必须在白名单内。因此写操作时要额外检查父目录：`os.path.dirname(realpath)` 是否被允许。本文提供的 `is_path_allowed` 可以直接用于写入目标路径，因为即使文件尚不存在，`os.path.realpath()` 在最坏情况下会抛出 `FileNotFoundError`。解决方案：先规范化路径的目录部分，目录必须存在且被允许，再将目录路径传入 `is_path_allowed`。更稳健的做法是分开检查：对现有文件直接用 `realpath`，对未创建的文件只检查其父目录且父目录在 `realpath` 后属于白名单。

## 可复用建议

- **封装成独立模块**：将 `is_path_allowed`、`load_whitelist` 放入 `agent_guard.py`，所有脚本统一引用，避免散落各处。
- **日志审计**：在拒绝访问时，记录完整路径、操作类型、时间戳，便于事后排查 Agent 的越界意图。
- **测试优先**：编写 pytest 覆盖常见边界：`..` 穿越、软链逃脱、绝对路径挂载点、相对路径基于不同 cwd。用 `tmp_path` 创建临时目录结构来模拟白名单。
- **与权限结合**：操作系统层面还可以使用专用的低权限用户运行 Agent 脚本，白名单作为第二层防护，纵深防御。
- **对于 MCP 客户端**：如果使用的是支持 tool 级别的权限声明机制，可以利用其内置的 `security` 字段加上 `allowed_paths` 原语，但目前多数框架尚未统一，所以自实现仍是主流。

## 总结

给自动化脚本加本地目录白名单，不是一个高深的技术点，却是 Agent 安全实践中投入产出比最高的措施。几十行代码的护栏，可以防止“删库跑路”式的 prompt 误读。工程上需要特别注意路径规范化、符号链接、动态工作目录等细节，否则护栏将形同虚设。把护栏做成模块化，配好日志和测试，再与系统权限组合，就能在保持 Agent 灵活性的同时有效控制文件访问边界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/a478c3c1261b107d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/cceb99cda9cd3501.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/2fb812d0049bc299.png)

