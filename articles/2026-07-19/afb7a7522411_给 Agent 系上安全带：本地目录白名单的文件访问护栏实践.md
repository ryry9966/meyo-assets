---
title: 给 Agent 系上安全带：本地目录白名单的文件访问护栏实践
feedId: 29656
source: 综合讨论
publishedAt: 2026-07-19
---

## 背景：当 Agent 开始触碰你的文件系统

在 OpenClaw 这类可扩展 Agent 框架中，工具函数让大模型获得了与本地环境交互的能力——读写配置、处理临时文件、生成报告。但能力带来风险：一段未加限制的提示指令就可能触发删除整个项目的操作，或读取 `~/.ssh` 等敏感路径。传统的沙箱方案（如 Docker）太重，而简单的权限控制又容易在设计时被忽略。最低权限原则告诉我们：Agent 只应访问它完成当前任务所必需的目录，其余一律拒绝。

## 问题：为什么“能跑就行”很危险

自动化脚本或 Agent 插件在本地执行时，默认继承了运行用户的权限。假设你写了一个“清理过期缓存”的工具，但路径拼接出现了 `../` 穿越，就可能删除非缓存目录；又或者插件被间接注入恶意指令，试图读取 `/etc/passwd`。没有护栏，调试时或许侥幸无事，但生产环境或共享脚本的一次失误就是事故。

工程上的正确解法不是彻底禁止文件访问，而是为文件操作套上一个白名单：只允许在明确指定的目录树内进行读写，其他路径一律拒绝，并且进行路径规范化防御。

## 做法：一个可落地的本地目录白名单

我们基于 Python 实现一个轻量级白名单检查器，可以直接集成到 OpenClaw 的工具函数中。核心思路如下。

### 1. 白名单配置

使用环境变量或配置文件定义允许访问的目录集合，如：
```ini
AGENT_ALLOWED_DIRS=/app/data,/app/output,/tmp/agent-workspace
```
程序启动时解析为绝对路径列表，存入内存。

### 2. 路径规范化与检查

这是最关键的一步。攻击者可能通过符号链接、`..`、多余的斜杠、大小写（Windows）来绕过白名单。检查流程必须：

- 解析相对路径时，强制以**白名单内某个根目录**为基准，或直接拒绝相对路径；
- 调用 `os.path.realpath()` 消除符号链接和 `..`；
- 在 Windows 上统一为长路径并忽略大小写（通过 `os.path.normcase`）；
- 最后判断规范化后的路径是否以任一白名单路径开头（注意末尾加分隔符防止前缀误判，例如白名单 `/app` 不能匹配 `/app2`）。

示例函数：
```python
import os

def is_path_allowed(path: str, allowed_dirs: list[str]) -> bool:
    # 如果传入相对路径，直接拒绝，避免解析歧义
    if not os.path.isabs(path):
        return False
    real_path = os.path.realpath(path)
    for allowed in allowed_dirs:
        # 规范化白名单路径
        allowed_real = os.path.realpath(allowed)
        # 必须确保 allowed_real 以路径分隔符结尾，防止 /app 匹配 /application
        if os.path.commonpath([real_path, allowed_real]) == allowed_real:
            return True
    return False
```
注意 `commonpath` 的用法只检测公共前缀是否正好等于白名单路径，这是比较可靠的方法。

### 3. 集成到工具调用

在 OpenClaw 中，工具函数通常以装饰器注册。我们可以写一个文件访问包装器，对需要读写文件的工具自动做白名单校验。例如：

```python
def with_file_guard(func):
    def wrapper(path, *args, **kwargs):
        if not is_path_allowed(path, ALLOWED_DIRS):
            raise PermissionError(f"Path {path} is not allowed")
        return func(path, *args, **kwargs)
    return wrapper

@register_tool
@with_file_guard
def read_file_content(path: str) -> str:
    with open(path, 'r') as f:
        return f.read()
```

这样，任何通过该工具的文件访问都会经过白名单拦截。

## 踩坑记录

1. **相对路径陷阱**  
   曾有工具接收相对路径并自行调用 `os.path.join`，结果拼接出 `../../etc`。解决方案是：**要求所有路径参数必须是绝对路径**，在检查之前就拒绝相对路径，让调用方（通常是 LLM 生成的代码）负责提供绝对路径，这样攻击面大大缩小。

2. **符号链接绕过**  
   即使传入的路径在白名单内，但如果它是一个指向白名单之外的符号链接，实际读写会越界。`realpath` 可解决，但要注意平台差异（Windows 的快捷方式不是符号链接，需单独处理）。

3. **前缀误判**  
   用 `startswith` 简单检查 `/app` 会匹配 `/app-secret`。改用 `commonpath` 比较后，这个问题消失，但必须保证两个路径都被 `realpath` 规范化。

4. **并发和 TOCTOU**  
   在检查与文件操作之间存在时间窗口，文件系统可能被外部修改（如替换为符号链接）。对于高安全场景，可结合临时文件描述符检查或使用 `openat` 系列系统调用，但对 Agent 日常使用来说，当前实现已足够。记录日志和审计才是性价比更高的选择。

## 可复用建议

将上述检查逻辑封装成一个独立模块 `filesystem_guard`，在项目中统一引用。建议：

- 支持通过环境变量 `ALLOWED_DIRS` 动态配置，方便容器化部署；
- 所有拒绝操作记录 warning 日志并输出原路径与规范化后的路径，便于事后分析；
- 对 OpenClaw 插件开发者，把白名单检查作为工具基类的一部分，或提供装饰器，降低接入成本；
- 考虑与 OpenClaw 的指令级沙箱（如限制工具集）配合，形成纵深防护。

## 总结

目录白名单不是银弹，但它是 Agent 文件访问护栏中最基础也最有效的一环。几十行代码就能杜绝绝大部分路径穿越和误操作风险。对于跑在个人开发机或服务器上的自动化脚本，这个方案轻量、透明、易于调试，非常适合工程实践。下次你的 Agent 试图触碰文件系统前，先问它一句：“你的安全带系好了吗？”

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/cf25e68413083405.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/0dbc5820fa185d35.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/64980101fca3fef3.png)

