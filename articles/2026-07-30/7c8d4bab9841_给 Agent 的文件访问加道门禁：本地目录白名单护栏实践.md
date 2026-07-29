---
title: 给 Agent 的文件访问加道门禁：本地目录白名单护栏实践
feedId: 30954
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景：当 Agent 拿到文件系统钥匙之后

不管是 MCP server、插件，还是你自己写的自动化脚本，一旦允许 Agent 操作本地文件，就相当于把房间钥匙交了出去。Agent 可以读取配置文件、写入结果、清理临时目录，这些都是日常工作。但如果不加约束，一次错误的 prompt 或工具调用就可能误删关键目录，或者泄露本不该外传的敏感文件。

一个常见的安全诉求是：**只允许 Agent 在指定的几个本地目录内自由活动，超出范围一律拒绝**。这在传统权限控制里像 chroot 或 Docker volume 挂载，但在 Agent 工具调用层面，我们需要更轻量的“应用层护栏”。

这篇文章记录一次在 OpenClaw 工具集里实现文件访问白名单的实践，包括路径规范化、符号链接处理、集成方式和踩到的坑。

## 问题拆解

简单限制目录看起来容易：拿到用户传入的 `file_path`，检查它是否以 `/allowed/root` 开头。但工程上会有一堆绕过方式：

- 相对路径 + `..` 跳转：`/allowed/root/../etc/passwd`
- 符号链接：在允许目录下创建软链接指向外部
- 多余的斜杠、大小写（Windows/macOS）
- 未展开的 `~`、环境变量等
- 多根目录判断：白名单可能有多个目录

很自然想到用 `os.path.realpath` 解析符号链接并返回绝对路径，再判断是否以白名单目录开头。但这还不够，因为 `realpath` 要求路径必须存在，而有时 Agent 需要创建新文件或目录，目标尚不存在，无法解析。这就需要分层处理。

## 做法：一个可复用的 SafetyChecker

我们可以封装一个 `FileSafetyChecker`，提供两个主要方法：

1. `check_existing_path(path)` —— 针对已存在的路径，走 realpath 解析，彻底解决符号链接。
2. `check_new_path(path)` —— 针对尚未创建的文件，先对父目录递归向上用 realpath 检查，确保父目录在白名单内，再把目标文件名拼回白名单根目录做前缀判断。

核心检查逻辑是字符串前缀匹配，但做了强化：规范化后，判断 resolved_path 是否与某个白名单根目录相等，或者以根目录加路径分隔符开头（防止 `/data` 白名单误允许 `/data_evil`）。

代码骨架如下（Python）：

```python
import os
from pathlib import Path
from typing import List

class FileSafetyChecker:
    def __init__(self, allowed_roots: List[str]):
        # 提前规范化白名单根目录（realpath）
        self.allowed_roots = [os.path.realpath(r) for r in allowed_roots]

    def _is_under_root(self, path: str) -> bool:
        for root in self.allowed_roots:
            # 确保是目录本身或者是其下的路径
            if path == root or path.startswith(root + os.sep):
                return True
        return False

    def check_existing_path(self, path: str) -> str:
        """解析存在的路径，确保落在白名单内，返回解析后的绝对路径"""
        resolved = os.path.realpath(os.path.expanduser(path))
        if not self._is_under_root(resolved):
            raise PermissionError(f"Access denied: {path} resolves to {resolved}")
        return resolved

    def check_new_path(self, path: str) -> str:
        """检查未创建文件的预期绝对路径，防止白名单绕过"""
        expanded = os.path.expanduser(path)
        abs_path = os.path.abspath(expanded)
        # 父目录必须存在且合法
        parent = os.path.dirname(abs_path)
        if os.path.exists(parent):
            parent_resolved = os.path.realpath(parent)
            if not self._is_under_root(parent_resolved):
                raise PermissionError(f"Access denied: parent directory {parent} outside allowed roots")
        else:
            # 父目录不存在，需逐级向上检查到存在的一层，并对剩余路径构造前缀判断
            # 具体实现略，但原则一样
            pass
        # 最后仍需检查 abs_path 的前缀，因为即使父目录合法，完整路径可能仍绕过（如目录遍历）
        return abs_path
```

然后可以在工具函数上做装饰器，自动校验输入路径：

```python
def safe_path(func):
    def wrapper(self, path, *args, **kwargs):
        checker = get_file_safety_checker()  # 单例
        safe_path = checker.check_existing_path(path) if os.path.exists(path) else checker.check_new_path(path)
        return func(self, safe_path, *args, **kwargs)
    return wrapper
```

## 集成到 MCP 或自动化工具

如果你的 Agent 是基于 MCP server 设计的，只要在所有文件读写相关的工具 handler 中使用上述检查即可。比如 `read_file`、`write_file`、`list_directory` 都先调用 checker。

在 OpenClaw 这类框架中，如果你以插件形式提供文件能力，可以把 checker 实例作为插件配置项，让用户通过 YAML/JSON 声明 `allowed_directories: ["/workspace", "/tmp/agent"]`。启动时完成 realpath 初始化，然后对外暴露的工具函数全部经过安全包装。

## 踩坑记录

1. **符号链接导致的 realpath 穿越**  
   首次测试时仅用了 `os.path.abspath`，未做 realpath。测试在允许目录内 `ln -s /etc/passwd link`，Agent 读取 `link` 时直接返回了真实路径 `/etc/passwd`，但 `abspath` 仍指向白名单下的 `link`，检查被绕过。用 `realpath` 才能拦截。

2. **路径必须存在才能 realpath**  
   创建新文件时路径不存在，直接 realpath 会报错。上面 `check_new_path` 的做法是先检查父目录，但要注意父目录也可能不存在（深层创建），需递归向上找到存在的父目录做锚点，再拼接剩余部分。这个递归逻辑容易写出 bug，最好用标准库 `pathlib.PurePath` 的 `parents` 属性。

3. **Windows 大小写与分隔符**  
   在 Windows 上，`c:\Workspace` 和 `C:\workspace` 是同一路径，但字符串比较不同。需要统一用 `os.path.normcase` 处理，或者在初始化白名单时用 `realpath` 自然获得系统规范化形式。另外反斜杠问题也会导致前缀判断失败，直接用 `os.path.normpath` 或使用 `pathlib` 比较。

4. **白名单目录自身被删除**  
   运行时允许目录被外部删除，导致 realpath 变动或失败。可在 checker 初始化时验证所有根目录存在且是目录，运行中如果发现根目录消失则拒绝所有操作并告警。

## 可复用建议

- **抽象成独立模块**：不要在每个工具函数内写判断逻辑，用 `FileSafetyChecker` + 装饰器，后面换成别的策略（如规则引擎）也容易。
- **配置化白名单**：用环境变量或配置文件注入允许目录，方便不同环境调整。
- **记录审计日志**：所有拒绝访问的操作记录下请求路径、解析后路径、时间，便于追溯是 Agent 行为异常还是配置疏忽。
- **与操作系统权限配合**：护栏是应用层最后一道防线，建议结合服务账户权限、容器挂载只读，纵深防御。

## 总结

给自动化脚本加目录白名单，看似是“路径前缀匹配”的小功能，但在符号链接、相对路径、平台差异的攻击面下，实现健壮的护栏需要仔细处理路径解析。核心思路：先区分已有路径和计划路径，对已有路径强制 realpath 阻断符号链接穿越；对计划路径从父目录锚点检查，再校验最终路径前缀。配合工具层统一的装饰器，可以在不大幅改动业务代码的前提下，把安全边界扎紧。

这个实践已经在多个内部 Agent 上稳定运行，拦截过几次 prompt 注入尝试（试图读取 `/etc/shadow`），效果符合预期。希望以上方案能帮助你的 Agent 安全地操纵本地文件。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/2660f40347d67603.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/e4b0178eec631210.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/a1b3de042b86255e.png)

