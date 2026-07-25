---
title: 给自动化脚本戴上紧箍咒：Agent 本地目录白名单实战
feedId: 30397
source: 综合讨论
publishedAt: 2026-07-25
---

# 给自动化脚本戴上紧箍咒：Agent 本地目录白名单实战

## 背景：一个被忽视的缺口

在构建 AI Agent 或自动化流水线时，我们越来越习惯把“执行脚本”作为一种工具开放给模型。比如让 Agent 调用 Python 脚本处理文件、运行数据分析、整理下载内容。常见做法是通过 MCP Server、插件系统或直接 subprocess 拉起外部脚本，传入参数后回收结果。

这个模式天然存在一个风险缺口：**脚本能访问整个文件系统**。  
如果 Agent 生成的参数包含 `../../.env`、`/etc/passwd`，或者脚本内部有 bug 遍历了上层目录，后果可能不只是任务失败。即使模型本身没有恶意，这种不受限的文件访问也像一个没有护栏的叉车，随时会撞到不该碰的东西。

## 问题拆解

在工程里，真正需要隔离的场景通常不是“完全沙箱化”（那样太重），而是**在既有的脚本执行框架上增加一层白名单目录约束**。目标很明确：

- 只允许脚本读写指定目录（如 `./data`, `./output`）
- 拦截任何试图越界的路径操作
- 开销足够小，不影响现有代码结构
- 能清晰报错，方便排障

典型的约束点是脚本**启动前参数校验**和**脚本内部文件操作拦截**。前者简单但不彻底，后者彻底但需要改造脚本。我们的做法是在脚本库层面加一层可复用的 Python 装饰器/上下文管理器，兼顾轻量和安全。

## 做法与步骤

下面以 Python 自动化脚本为例，假设你的 Agent 会调用类似 `python3 process_file.py --input ./data/source.csv --output ./output/result.csv` 的命令。

### 1. 定义白名单根目录

在脚本入口统一声明允许访问的基础路径列表：

```python
ALLOWED_ROOTS = [
    Path("/app/data").resolve(),
    Path("/app/output").resolve(),
]
```

使用 `resolve()` 尽早消除符号链接和相对路径干扰。

### 2. 路径安全校验函数

核心是一个校验函数，判断任意路径是否处于白名单子树内。注意必须处理 `..` 穿越、符号链接、以及 `Path` 比较时的边界。

```python
import os
from pathlib import Path

def is_path_allowed(target: Path, allowed_roots: list[Path]) -> bool:
    try:
        real_target = target.resolve(strict=False)
    except (OSError, RuntimeError):
        return False
    for root in allowed_roots:
        try:
            real_target.relative_to(root)
            return True
        except ValueError:
            continue
    return False
```

这里 `relative_to` 会判断 `real_target` 是否以 `root` 为前缀。`strict=False` 避免路径不存在时抛异常直接拒绝，后面可以再单独处理“不存在”的情况。

### 3. 装饰器注入文件操作拦截

如果你的脚本会大量使用 `open`、`shutil`、`pandas.read_csv` 等，最省力的方式是包装内置的文件操作，在打开文件前做白名单校验。一个例子：

```python
import builtins
from functools import wraps

original_open = builtins.open

@wraps(original_open)
def safe_open(file, mode='r', *args, **kwargs):
    path = Path(file).resolve()
    if not is_path_allowed(path, ALLOWED_ROOTS):
        raise PermissionError(f"Access denied for path: {file}")
    return original_open(file, mode, *args, **kwargs)

builtins.open = safe_open
```

执行脚本时先导入这段初始化代码（或放在一个 `init_guard.py` 中，用 `-c` 或 `PYTHONSTARTUP` 注入），后续所有 `open` 调用都会被拦截。

### 4. 命令行参数校验

如果脚本通过参数接收路径，在 `argparse` 解析后立即对所有路径类型参数做校验：

```python
for arg_name in ["input", "output"]:
    path = Path(getattr(args, arg_name))
    if not is_path_allowed(path, ALLOWED_ROOTS):
        raise SystemExit(f"Illegal path argument: {arg_name}={path}")
```

### 5. 结合 subprocess 从外部卡点

如果不想改动脚本内部，可以在调用脚本前用启动器包裹。例如编写一个 `run_sandboxed.py`：

```python
# run_sandboxed.py
import sys, subprocess
from pathlib import Path

ALLOWED_ROOTS = [Path("/app/data").resolve(), Path("/app/output").resolve()]

def check(args):
    for a in args[1:]:
        if Path(a).is_absolute() or a.startswith(".."):
            p = Path(a).resolve()
            if not is_path_allowed(p, ALLOWED_ROOTS):
                raise PermissionError(f"Forbidden path: {a}")

if __name__ == "__main__":
    check(sys.argv[1:])
    subprocess.run([sys.executable] + sys.argv[1:], check=True)
```

Agent 调用时改为 `python3 run_sandboxed.py process_file.py --input ...`。这可以在不进脚本代码的情况下提供一层硬拦截。

## 踩坑点

### 1. `Path.resolve()` 时机与不存在的路径

如果路径指向尚不存在的文件（常见于 `--output`），`resolve(strict=False)` 仍然有效，但相对路径需要拼接当前工作目录。务必在**确定工作目录**后再调用 `resolve`，否则不同执行目录可能导致白名单匹配失败。建议脚本入口固定 `os.chdir("/app")`。

### 2. 符号链接绕过

`resolve()` 可以化解多数符号链接，但如果攻击者能在白名单目录内创建指向外部的符号链接，然后通过 `../` 间接访问，仍然可能绕过。**必须确保白名单目录内部不可写**，或者用 `os.path.realpath` 双重检查。更严格的做法是在 `is_path_allowed` 中对 `real_target` 的每一级目录检测是否包含符号链接，但这会显著增加复杂度，除非你的场景安全等级很高，否则采用“限制写入权限”是更务实的解法。

### 3. 库函数的隐式文件访问

`pandas.read_csv` 底层会调用 `open`，所以替换内置 `open` 能覆盖大部分场景。但有些 C 扩展模块直接使用系统调用而不是 Python 的 `open`（例如某些图像处理库），拦截不到。如果依赖此类库，单纯的白名单装饰器不够，需要配合操作系统级限制如 `chroot` 或 `bind mount`。我们的经验是：在 Agent 调用层先做好参数卡控，再对可信脚本放宽至装饰器级，对高危脚本直接上容器。

### 4. 相对路径与工作目录一致性

如果脚本内部执行了 `os.chdir` 后又进行文件操作，`resolve()` 的结果可能变得不可预测。最简单的方法是**在启动脚本处设置工作目录并禁止脚本内修改**（或至少在处理文件前重置）。也可以通过 `os.getcwd()` 结合路径拼装来动态校验，但维护成本会上升。

## 可复用建议

1. **抽离为独立 security 模块**：把 `ALLOWED_ROOTS` 配置、`safe_open`、校验函数放在一个 `agent_guard.py`，所有自动化脚本第一行 `import agent_guard`，即可激活白名单。模块内设置 `__all__` 并避免污染全局，便于增删。

2. **使用环境变量传递白名单**：在容器化部署时通过 `ALLOWED_ROOTS` 环境变量注入路径列表，启动器解析后生效，避免硬编码。

3. **统一错误信息**：拦截时抛出 `PermissionError` 并带上完整路径和进程信息，方便在 Agent 的日志里快速定位是哪个工具调用出了问题。可以对接你自己的告警系统。

4. **组合多层防御**：  
   - 第一层：Agent 平台/MCP Server 参数白名单校验（拒绝明显越界参数）  
   - 第二层：启动器脚本拦截（subprocess wrapper）  
   - 第三层：脚本内 open 拦截（装饰器）  
   根据团队能力和脚本来自行裁剪。

5. **测试用例陪伴**：针对白名单校验函数写几条边界测试，包括 `../../etc/passwd`、符号链接指向外部、不存在的文件路径等，作为安全基线的回归用例。

## 总结

给自动化脚本加本地目录白名单并不是为了防住国家级黑客，而是为了防止 Agent 的“无心之失”演变成删库跑路。在工程实践中，这层护栏做在**脚本执行的入口和内置 open 处**就足够拦住 90% 的风险，代价很低。如果未来需要更强的隔离，可以平滑升级到容器或虚拟文件系统，但这套路径级别的管控依然可以作为纵深防御的最内层。

在 OpenClaw 这类工具生态里，当你开始把越来越多的“能力”以脚本形式交给 Agent 时，请记得同时把对应的安全边界画好。毕竟，在自动化这件事上，踩油门之前最好先看看刹车灵不灵。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/3d2fae379dcc756d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/ab9dab9313d233de.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/fcea23a76b14efd7.png)

