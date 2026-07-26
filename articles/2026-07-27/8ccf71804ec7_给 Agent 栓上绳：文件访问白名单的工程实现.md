---
title: 给 Agent 栓上绳：文件访问白名单的工程实现
feedId: 30627
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景

在 OpenClaw 这类 Agent 框架中，我们经常让 LLM 驱动的自动化脚本直接操作文件系统：生成代码后写入文件、读取配置、处理文档……如果脚本权限等于进程权限，一次错误的 `rm -rf /` 或者 `os.remove` 就可能把整个项目目录甚至系统文件清空。更隐蔽的风险是信息泄漏：Agent 可能不经意地读走 `~/.ssh` 或环境变量文件。

无论是基于 MCP 的工具调用，还是插件形式的自定义脚本，都迫切需要一种「目录白名单」机制：**只允许在指定目录（及其子目录）内读写，其他路径一律拒绝**。这不能仅仅靠“提示词约束”，必须落到代码层面的强制检查。

## 问题定义

我们的目标是实现一个轻量级、不依赖容器或额外守护进程的文件访问护栏，可以直接嵌入到 Python 自动化任务中。核心需求：

- 接受一个或多个“安全目录”作为白名单（绝对路径）
- 任何文件操作前，将目标路径解析为真实绝对路径
- 判定该路径是否位于任一白名单的子树内
- 若是，放行；否则抛出异常或拒绝操作
- 能够防御符号链接、`..` 回溯等常见绕过手段

## 实现步骤

### 1. 路径规范化是关键

Python 有多个路径处理函数，但安全场景必须用 `os.path.realpath()`。它会：
- 将相对路径转为绝对路径（基于当前工作目录）
- 解析所有 `..` 和 `.`
- 递归解析符号链接，直到得到真正的物理路径

对比：
- `os.path.abspath()` 只做前两步，**不解析 symlink**，可直接绕过。
- `pathlib.Path.resolve()` 默认也解析 symlink，行为与 `realpath` 一致，可以互用。

### 2. 判断是否在白名单子树内

给定一个安全目录 `allowed_dir`（同样是 `realpath` 之后的值），只需检查：

```python
def is_allowed(target_real, allowed_dir):
    # 确保 allowed_dir 以分隔符结尾，避免 /var/app 匹配到 /var/app-v2
    allowed_dir = os.path.join(allowed_dir, '')
    return target_real.startswith(allowed_dir)
```

对多个白名单目录，依次遍历即可。

### 3. 封装安全操作

创建一个 `FileGuard` 类，暴露 `safe_open`、`safe_remove`、`safe_listdir` 等常用接口。每个接口在执行实际 I/O 前，先调用内部校验方法，校验失败抛出 `PermissionError`。

```python
import os
import builtins

class FileGuard:
    def __init__(self, allowed_dirs):
        self._allowed = [os.path.realpath(d) for d in allowed_dirs]

    def _check_path(self, path):
        real = os.path.realpath(path)
        for allowed in self._allowed:
            if real.startswith(os.path.join(allowed, '')):
                return real
        raise PermissionError(f"Access denied: {path} -> {real}")

    def safe_open(self, path, mode='r', *args, **kwargs):
        real = self._check_path(path)
        return builtins.open(real, mode, *args, **kwargs)

    def safe_remove(self, path):
        real = self._check_path(path)
        os.remove(real)
```

注意：这里使用 `builtins.open` 避免递归调用。

### 4. 集成到 Agent 的工具调用

在 MCP 工具或自定义插件中，可以全局维护一个 `FileGuard` 实例，在需要文件操作的函数内部调用其安全方法。例如：

```python
guard = FileGuard(["/app/workspace"])

def write_file(filename, content):
    with guard.safe_open(filename, 'w') as f:
        f.write(content)
```

如果使用 `subprocess` 执行外部命令（如 `mv`、`cp`），也要提前用 `_check_path` 检查参数中的路径，但无法防止命令内部的路径变换，强烈建议让 Agent 只通过自己的文件操作函数工作，不直接暴露 shell。

## 踩坑实录

1. **符号链接绕过**：即使 `allowed_dirs` 是 `/safe`，若 `/safe/link -> /etc`，Agent 写 `/safe/link/passwd` 实际会写入 `/etc/passwd`。`realpath` 会得到 `/etc/passwd`，检查时发现不在 `/safe` 子树内，从而拒绝。因此每次操作前都要 `realpath`，不能提前缓存“允许列表”并跳过解析。

2. **结尾分隔符陷阱**：`startswith` 不处理路径边界。`/var/app` 是 `/var/app-v2` 的前缀。必须给比较基准加上结尾分隔符，或使用 `os.path.commonpath` 比较。更稳妥的方法是：

```python
def is_subdir(child, parent):
    child = os.path.realpath(child)
    parent = os.path.realpath(parent)
    relative = os.path.relpath(child, parent)
    return not (relative == os.pardir or relative.startswith(os.pardir + os.sep))
```

3. **当前工作目录**：Agent 进程的 `cwd` 可能在运行中改变。如果允许相对路径输入，务必意识到 `realpath` 依赖 `cwd`。要么在入口处将相对路径转绝对，要么固定 `cwd` 到安全目录并在检查时显式传入基准。

4. **TOCTOU 竞态**：先在检查时得到 `real`，随后文件系统可能变化（例如文件被删并替换为 symlink）。对于高安全需求，可在打开时使用 `os.open` 加 `O_NOFOLLOW` 标志，禁止跟随 symlink，然后从文件描述符包装成 Python 文件对象。大部分场景的短时间窗口内风险较低，但值得了解。

5. **Windows 兼容**：若需跨平台，注意盘符、大小写不敏感以及分隔符差异。建议统一用 `os.path.normcase` 或 `pathlib` 处理。

## 可复用建议

- **最小化白名单**：只给 Agent 分配一个项目目录，甚至读写目录分开（例如 `output/` 可写，`config/` 只读）。
- **关闭危险的系统调用**：如果可能，直接禁用 Agent 执行 `os.system`、`subprocess` 等，改用受限的函数封装。
- **日志与审计**：在 `_check_path` 中记录每一次文件访问，便于发现异常行为。
- **代码片段即用**：上述 `FileGuard` 类已经可以直接复制使用。将其放在一个独立模块中，所有文件相关工具都经过它，防止疏忽。
- **进阶方案**：当对隔离性要求更高时，可以结合 `chroot`、Linux namespace 或 bubblewrap，在进程级别限制文件系统可见性。目录白名单方案适合轻量、嵌入式场景，部署成本极低。

## 总结

给自动化脚本加上文件访问目录白名单，把“原则”变成了代码里的硬约束。核心是路径解析与前缀判断的组合，用 `realpath` 堵住符号链接和路径回溯的漏洞，再用简洁的封装切掉裸 `open` 的习惯。这个小护具在 OpenClaw / MCP / 插件项目中几乎零成本集成，却能让一次误操作从灾难降格为一次可控的 `PermissionError`。

文件系统的“最小权限”是 Agent 安全的第一条防线，早做起，持续受益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/25f6c3c4a59ae43a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/16a5363d176f81fa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/7a5c84bd9358e66a.png)

