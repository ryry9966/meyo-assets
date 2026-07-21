---
title: Agent 文件访问护栏：给自动化脚本加本地目录白名单
feedId: 29896
source: 综合讨论
publishedAt: 2026-07-21
---

## 为什么需要目录白名单

当 Agent 通过工具或插件获得文件读写能力后，一个常见的隐患是：脚本在无约束的情况下运行，可能意外操作到配置文件、密钥、系统目录或用户隐私数据。更糟的是，通过 MCP 服务器或自定义插件暴露给 LLM 的文件接口如果没有收敛，一次错误的生成就可能引发难以追踪的破坏。

工程化实践里，解决这个问题的方法不是“信任 prompt 约束”，而是**在文件访问层面构建目录白名单**——即脚本运行环境只能读写预先声明的一组本地目录，越界操作直接拒绝。这既是一道防呆护栏，也降低了审计成本。

本文以 Python 自动化脚本为例，说明如何实现一个轻量、可复用的文件访问白名单，并归纳踩坑点和可复用建议。思路同样适用于 Node.js 或其他语言。

## 问题拆解

要给文件操作加白名单，核心是要拦截所有路径输入，经过规范化后判断是否落在允许的目录集合内。典型的调用链可能是：

1. LLM 生成一个文件读写指令
2. 工具函数接收文件路径（可能是相对路径、包含 `..` 的路径、符号链接）
3. 归一化为绝对路径
4. 检查该路径是否以白名单目录为前缀
5. 如果不在白名单内，抛出异常、记录日志并拒绝执行

关键点在于**路径规范化和边界判断必须可靠**，否则白名单形同虚设。

## 实现一个路径沙箱

以下是一个基于 Python 的轻量实现，适用于 Agent 插件或自动化脚本中任何涉及文件 IO 的地方。

### 1. 定义白名单目录

```python
import os
import pathlib
from typing import List, Union

class PathSandbox:
    def __init__(self, allowed_dirs: List[Union[str, pathlib.Path]]):
        # 统一转换为绝对路径并规整，保留一份列表
        self._allowed = []
        for d in allowed_dirs:
            abs_dir = os.path.realpath(os.path.abspath(d))
            self._allowed.append(abs_dir)

    def check_access(self, target_path: Union[str, pathlib.Path]) -> str:
        """
        校验路径是否在白名单内，是则返回绝对路径，否则抛出异常。
        """
        # 解析绝对路径并消解符号链接
        abs_target = os.path.realpath(os.path.abspath(target_path))

        # 检查目标路径是否以任一允许目录开头
        for allowed in self._allowed:
            # 统一加路径分隔符，避免 /project/app 误匹配 /project/app2
            if abs_target == allowed or abs_target.startswith(allowed + os.sep):
                return abs_target

        raise PermissionError(
            f"Access denied: '{abs_target}' is not in allowed directories."
        )
```

使用时，只需在脚本入口处实例化 PathSandbox 并显式校验所有传入路径：

```python
sandbox = PathSandbox(allowed_dirs=["/workspace/my_project/data", "/workspace/my_project/output"])

def safe_read_file(filename: str) -> str:
    abs_path = sandbox.check_access(filename)
    with open(abs_path, 'r') as f:
        return f.read()
```

任何调用 `safe_read_file` 的代码都会先经过白名单过滤，即使 LLM 构造了一个越界路径也会被立即拦截。

## 踩坑点

### 1. 符号链接穿透

如果允许的目录 `/workspace/project` 下存在一个指向 `/etc` 的符号链接，而用户请求 `project/secret_link/passwd`，直接用 `os.path.abspath` 再字符串前缀判断就会绕过白名单。**必须使用 `os.path.realpath` 将符号链接解析为目标路径的真实位置**，然后再进行前缀匹配。这也是上面实现中始终调用 `realpath` 的原因。

### 2. 未尾加分隔符导致的子串匹配误伤

`abs_target.startswith('/var/app')` 对 `/var/app2/config` 会返回 True。标准做法是在匹配时给白名单目录追加 `os.sep`，同时对完全相等的情况额外处理，如代码所示。

### 3. 路径不存在的文件

Agent 可能尝试写入“新文件”，此时目标路径不存在，`realpath` 将只解析已存在的目录部分，末尾文件分量保持不变。但 `realpath` 对于不存在的路径会返回其规范化后的“预期”路径，因此上述逻辑依然有效。不过如果要求白名单目录必须真实存在，可以在初始化时检查 `allowed_dirs` 是否均为有效目录，防止配置错误。

### 4. 日志泄露路径

出于审计目的，拒绝访问时通常会记录被拒绝的绝对路径。注意日志存储的权限，避免敏感路径再度泄露。建议只记录相对路径或路径哈希，或将日志访问也纳入文件系统白名单。

### 5. 大小写敏感与 Windows 兼容

如果在 Windows 环境下，`os.path.realpath` 会处理大小写问题，但前缀检查时使用 `startswith` 需注意盘符和 `os.sep`。可以统一使用 `pathlib` 的 `Path` 对象进行比较，更安全且跨平台。示例代码可改用 `pathlib.Path.resolve()` 并在比较时使用 `path_obj.is_relative_to(allowed_path)`（Python 3.9+），这能自动处理分隔符和相对判断，但需注意 `is_relative_to` 的陷阱：它也是纯前缀比较，最好还是用 `realpath` 标准化后再对比。

## 可复用建议

1. **封装成独立模块**：将 PathSandbox 抽离为通用库，不仅供读写文件使用，也可以集成到其他系统操作如子进程调用、日志输出中。所有的文件系统交互统一走沙箱入口。
2. **集成到 MCP 服务器**：如果通过 MCP 向 LLM 暴露文件工具，在工具定义函数内部强制调用沙箱校验，这样即便多个模型共享同一服务器，也能保持访问隔离。可以根据不同工具声明不同的白名单。
3. **环境变量配置白名单**：将允许目录列表通过环境变量注入，而不是硬编码，以便在部署时动态调整。
4. **禁止取消限制**：在沙箱初始化后，禁止运行时更改白名单，防止意外 override。
5. **配合最小权限原则**：生产环境中 Agent 运行的用户也应该只是普通用户，仅对工作目录有读写权限，沙箱是第二道防线。

## 总结

Agent 文件操作的白名单护栏，本质是将“什么可以访问”显式化，用代码保证而非提示词约束。实现难度不高，但在工程细节上容易留下符号链接、相对路径、子串匹配等安全缺口。通过路径规范化加前缀检查的方案，配合一些经验性补丁，可以显著降低脚本失控的风险。

在实际落地中，把这一层做成可复用的基础设施，并在 MCP 工具和自动化插件中统一调用，比零散地在每个工具里重复实现更可靠、也更易维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/f4df5fdd1a5bcb8d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/4ad4eae77f98653b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/57c2b18d6f348b41.png)

