---
title: 给 Agent 文件访问加一道锁：本地目录白名单实战
feedId: 30203
source: 综合讨论
publishedAt: 2026-07-23
---

# 给 Agent 文件访问加一道锁：本地目录白名单实战

## 1. 背景：当 Agent 开始读写你的 `~/.ssh`

给 Agent 配一个工具函数去执行本地 Shell 或读写文件，已经成了很多 OpenClaw 用户打通自动化“最后一公里”的日常操作。无论是用 MCP 暴露一个 `run_command`，还是直接在工作流里调用 Python 脚本操作本地文件系统，本质上都是在给 LLM 开了一把可以碰到真实环境的钥匙。

问题在于，这把钥匙很多时候是**全量**的。Agent 说“我要把配置文件写入 `~/app/config.yaml`”——它确实能写，但理论上它也能一个 `rm -rf ~/*` 把家目录扬了，或者把 `.env` 里的密钥吐到某个公开日志里。

在工程化场景里，我们不能只靠 prompt 约束。提示词是概率性的，文件系统是真实的。需要一个确定性的**目录白名单护栏**：不论 Agent 生成什么路径，脚本只允许在指定的目录集合内操作，其他一律拒绝。

## 2. 问题拆解：不是简单的路径前缀匹配

第一反应是：写个函数，检查目标路径是否以允许的前缀开头。比如白名单是 `["/home/user/project/data", "/tmp/agent_workspace"]`，那么 `path.startswith(allowed)` 就算通过。但实际落地时有几个暗坑：

- **路径规范化**：`/home/user/project/data/../../.ssh/id_rsa` 能绕开前缀检查。
- **符号链接**：白名单目录里可能有一个指向 `/etc` 的软链，操作它就越权了。
- **相对路径**：Agent 可能只传 `../config/system.json`，而脚本当前工作目录并非白名单根。
- **新建文件 vs 现有文件**：即使路径在白名单内，是否允许覆盖已有重要文件？可能需要更细粒度的策略。

一个简单的工程化护栏，至少需要解决前三个问题，让路径穿越攻击失效，同时保持实现足够轻量，不引入全量沙箱（比如 Docker）的复杂度。

## 3. 实现步骤：三层校验的本地白名单检查器

下面给出一个可直接在 Python 工具函数中复用的白名单校验器。假设我们的使用场景是 OpenClaw 通过 MCP 或自定义插件调用一个 `safe_file_write` 或 `safe_execute` 工具。

### 3.1 核心函数

```python
import os
import pathlib
from typing import List

def is_path_allowed(target: str, allow_dirs: List[str]) -> bool:
    """
    返回 target 是否落在 allow_dirs 声明的白名单目录树内。
    规则：
    - 先对 target 做严格的规范化（解析符号链接前先限制边界）
    - 要求 target 解析后的真实路径必须处于某个 allowed 目录下
    - 禁止路径穿越，即使解析后落在白名单内，也拒绝包含 '..' 的原始路径
    """
    # 1. 粗暴拒绝显式的向上穿越
    if '..' in pathlib.PurePosixPath(target).parts:
        return False

    # 2. 将 target 转为绝对路径，但暂不解析符号链接
    raw_abspath = os.path.abspath(target)

    # 3. 对每个白名单目录，同样转为绝对路径
    for allow_dir in allow_dirs:
        abs_allow = os.path.abspath(allow_dir)
        try:
            # 解析符号链接，得到最终的真实路径
            real_allow = os.path.realpath(abs_allow)
            # 注意：这里先解析白名单目录本身的真实路径
            # 再检查 raw_abspath 是否以 real_allow 开头
            if not raw_abspath.startswith(real_allow + os.sep) \
               and raw_abspath != real_allow:
                continue

            # 4. 最终检查：对目标路径也解析真实路径，防止符号链接逃逸
            # 但如果目标路径还不存在，realpath 会报错或返回父目录实路径，需特殊处理
            if os.path.lexists(raw_abspath):
                real_target = os.path.realpath(raw_abspath)
                if real_target.startswith(real_allow + os.sep) or real_target == real_allow:
                    return True
            else:
                # 文件不存在：检查其父目录是否真实在白名单内，避免创建时的条件竞争
                parent_real = os.path.realpath(os.path.dirname(raw_abspath))
                if parent_real.startswith(real_allow + os.sep) or parent_real == real_allow:
                    return True
        except (OSError, ValueError):
            continue
    return False
```

### 3.2 集成到工具调用

在工具函数入口处直接调用：

```python
ALLOWED_DIRS = ["/var/agent_workspace", "/tmp/sandbox"]

def write_file(path: str, content: str):
    if not is_path_allowed(path, ALLOWED_DIRS):
        raise PermissionError(f"Access denied: {path}")
    os.makedirs(os.path.dirname(os.path.abspath(path)), exist_ok=True)
    with open(path, 'w') as f:
        f.write(content)
```

同理，任何 `read_file`、`list_dir`、`shell_exec` 中涉及本地路径的参数，都要经过这个校验。对于 Shell 命令，更建议只用参数化的形式，避免直接拼接路径字符串。

## 4. 踩坑实录

- **符号链接检查的顺序**：如果先对目标路径调用 `realpath`，而路径尚不存在，就会抛出异常或返回不真实的结果。所以用了“先检查原始绝对路径前缀 → 再在文件存在时做 `realpath` 兜底”的两阶段策略。
- **`os.path.abspath` 依赖当前工作目录**：如果你的 Agent 工具进程可能在不同工作目录下运行，建议在入口处显式设置 `os.chdir("/")` 或使用 `pathlib.Path(target).resolve()` 提供更可控的行为。
- **`PurePosixPath` 在 Windows 上的兼容性**：如果你的环境是 Windows，`'..'` 检查需要同时适配 `'..'` 和路径分隔符。使用 `pathlib.PurePath(target).parts` 检查更具跨平台性。
- **竞争条件**：创建文件时，检查父目录的真实路径和实际写入之间，存在 TOCTOU 风险。高安全场景下，应该先 `open(path, O_CREAT | O_EXCL)` 再由文件描述符确认路径，或者采用绑定挂载、容器等方案。但在日常自动化脚本的护栏中，上述实现已能阻断绝大多数无意或恶意的路径穿越。
- **不要把白名单目录设为 `/` 或 `/home`**：这是自己拆护栏。白名单应该越精准越好，比如一个项目专用的 `project/agent_outputs/` 目录。

## 5. 可复用建议

- **白名单配置化**：将 `ALLOWED_DIRS` 抽取为环境变量或配置文件，与 Agent 运行的上下文绑定，避免硬编码。
- **与 OpenClaw 的权限声明结合**：如果你使用 OpenClaw 的插件体系或 MCP 模块，把文件写入工具的 manifest 中声明 `requires: ["fs:write"]`，并在描述里清晰告知 Agent 只能访问特定目录。但不要只依赖声明做安全，实际执行层必须做校验。
- **日志与审计**：每次被白名单拦截的路径请求，都应记录原始路径和被拒绝原因，便于排错和发现 Agent 的非预期行为。
- **进阶：分层权限**：可以考虑将“只读目录”与“可写目录”分开配置，提供更细粒度的控制。例如允许 Agent 读取 `/usr/share/docs`，但只允许写入 `/tmp/output`。
- **测试套路**：编写单元测试覆盖以下场景：
  - 路径穿越：`../etc/passwd`
  - 绝对路径直接指向白名单外：`/etc/passwd`
  - 利用软链接跳板：在白名单内创建指向 `/etc` 的软链，然后操作 `symlink/file`
  - 不存在的文件写入
  - 白名单目录本身的真实路径中有符号链接（例如 `/var` 是一个软链接）

## 6. 总结

给 Agent 加文件访问护栏，不是为了拒绝自动化，而是为了让自动化在**可预测的边界内**运行。一个小而硬的目录白名单，可以防止 prompt 扰动造成的路径穿越、抑制插件行为扩散，也能在多人协作的脚本环境中守住底线。

上面给出的实现足够轻量，几行代码就能嵌入现有的工具函数，同时覆盖了符号链接、不存在文件等真实场景。在工程实践中，建议把它作为 Agent 工具函数的第一道门禁，配合最小权限原则和日志审计，让文件系统访问不再是心悬一线的操作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/223bda2056277bbc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/7137c31537c522ae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/48a4ca9aeb1fb1e9.png)

