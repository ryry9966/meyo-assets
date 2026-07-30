---
title: Agent 文件访问护栏：通过路径包装器实现本地目录白名单
feedId: 31020
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景

在基于大模型的 Agent 工作流里，让模型编写并执行本地脚本已经不再稀奇。OpenClaw 这类框架允许模型通过插件或 MCP 工具调用系统命令，例如用 Python 处理一批 CSV、用 FFmpeg 转码视频，或者调用用户自定义的自动化脚本。能力越强，风险越大：如果脚本被诱导去读取 `/etc/passwd`、`~/.ssh/id_rsa`，或者在 `/tmp` 外写入恶意文件，传统权限控制（如文件所有者和 POSIX 权限位）不一定挡得住——因为 Agent 就是以当前用户身份运行的。

一个现实的案例：我们用 Agent 提供一个“批量图片压缩”服务，允许它执行 ImageMagick 命令，输入文件夹和输出文件夹由用户通过对话给出。某次提示词注入后，Agent 生成了这样的命令：

```bash
convert /etc/shadow -resize 100x100 /var/www/html/out.png
```

好在测试环境只读挂载了 `/etc`，没有造成泄露，但足以说明问题：必须给脚本的执行加一道护栏，只允许它碰指定的目录。

## 问题定义

我们要为一个自动化脚本（或任意被 Agent 调用的可执行文件）加上**本地目录白名单**，使得该脚本**只能读取、写入白名单内的路径**，对其他路径的访问直接拒绝。需求特点：

- 脚本类型多样，可能是 Python、Shell、二进制程序，我们不一定能修改脚本本身。
- 要求轻量，不依赖 Docker、KVM 等重型沙箱，在一台开发机或边缘设备上快速部署。
- 透明：对调用者（Agent 运行时代码）而言，像正常调用子进程一样，只是多了一层校验。

一个朴素的想法：写一个**包装器**，在调用真正的脚本之前，解析命令行参数中出现的路径，检查它们是否落在白名单目录内，如果不通过就拒绝执行。这种方法虽然不是绝对安全的沙箱（无法防止脚本内通过硬编码或环境变量引用非法路径），但对于大多数“参数注入”类的攻击，足够有效且容易落地。

## 实现步骤

### 1. 确定白名单

在白名单列表 `ALLOWED_ROOTS` 中声明一个或多个绝对路径，比如：

```python
ALLOWED_ROOTS = ["/data/agent-workspace", "/tmp/agent-scratch"]
```

所有被脚本访问的路径都必须是其中某个根目录下的子路径。

### 2. 路径校验函数

核心逻辑：使用 `pathlib.Path.resolve()` 将路径标准化，消除 `../` 和符号链接，然后检查其前缀是否属于白名单根目录。注意必须同时检查根目录本身也被标准化，避免绕过。

```python
from pathlib import Path

def is_allowed(path_str: str, allowed_roots: list[Path]) -> bool:
    try:
        real = Path(path_str).resolve(strict=False)
    except (OSError, RuntimeError):
        return False

    for root in allowed_roots:
        # 确保根目录本身也被标准化过
        try:
            if real.is_relative_to(root):
                return True
        except ValueError:
            # Python <3.9 用 str.startswith 加分隔符
            if str(real).startswith(str(root) + "/") or real == root:
                return True
    return False
```

`strict=False` 允许路径暂时不存在（例如输出目录尚未创建），仍然能解析到其应属的目录，避免因为不存在而抛出异常导致绕过。

### 3. 包装器主体

写一个 Python 脚本 `run_allowed.py`，接受的第一个参数是真实脚本的路径（当然也要检查它本身在白名单内或单独允许），后面是该脚本的参数。包装器扫描所有参数，提取看上去像路径的字符串（基于简单的启发式，如包含 `/` 或以 `-` 开头但后接路径的模式），逐一过 `is_allowed`，不通过则拒绝执行并输出明确错误信息。

简化版代码骨架：

```python
import sys, subprocess

ALLOWED_ROOTS = [Path("/data/agent-workspace").resolve(),
                 Path("/tmp/agent-scratch").resolve()]
TARGET_SCRIPT = Path(sys.argv[1]).resolve()
if not is_allowed(str(TARGET_SCRIPT), ALLOWED_ROOTS):
    print(f"Refused to execute: {TARGET_SCRIPT}", file=sys.stderr)
    sys.exit(1)

args = sys.argv[2:]
for arg in args:
    # 简单路径启发性检测
    if arg.startswith("/") or arg.startswith("./") or arg.startswith("../") or "/" in arg:
        if not is_allowed(arg, ALLOWED_ROOTS):
            print(f"Refused path argument: {arg}", file=sys.stderr)
            sys.exit(1)

os.execv(str(TARGET_SCRIPT), [str(TARGET_SCRIPT)] + args)
```

使用 `os.execv` 替换当前进程，避免占用多余 PID，同时确保真实脚本的运行时行为与直接调用一致。若脚本需要 stdin / stdout 重定向，可以通过标准流继承。

### 4. 集成到 Agent 工作流

在 OpenClaw 的 MCP 工具或动作定义中，将原本直接执行的命令替换为通过包装器调用：

```
# 原始动作
convert {input} {output}

# 替换后
python3 /opt/agent-guard/run_allowed.py /usr/bin/convert {input} {output}
```

白名单配置写在包装器脚本内的常量或外部 JSON 文件，由运维管理，Agent 无法修改。

## 踩坑记录

### 符号链接与挂载点

- **坑**：`/tmp/agent-scratch` 如果是某个挂载点的符号链接，`resolve()` 会将其展开成真实路径，但根列表里存的是原始路径。如果 `resolve()` 后的根与预存根不一致，`is_relative_to` 会失败。
- **解**：在初始化 `ALLOWED_ROOTS` 时就对每个目录调用 `.resolve()`，确保比较双方处于同一次符号链接解析的维度。

### 相对路径与工作目录

- 如果脚本内部使用相对路径（如 `open("config.ini")`），包装器无法截获，但这已超出“命令行参数注入”的防护范围。可通过 `chdir` 到白名单内的一个安全目录来缓解，但注意这会影响脚本行为。建议在文档中限定：包装器只防护命令行传入的路径，脚本自身的硬编码路径需通过单独的代码审查保障。

### TOCTOU 竞态

- 在检查通过后、`exec` 执行前，文件系统的布局可能被并发修改（例如恶意程序替换符号链接）。在生产环境，如果并发用户不可信，可考虑使用 Landlock 或 Seccomp 进一步限制，但会引入更多复杂度。对于单用户 Agent 场景，这个风险可以接受。

### 动态构造的路径参数

- 有些程序通过多个参数拼接路径，比如 `--dir=/data/agent-workspace --file=..%2F..%2Fetc%2Fpasswd`。包装器只能检测到完整字符串，难以拆解。建议限制只接受单一绝对路径参数，或在 Agent 层面进行输入消毒；包装器可附加一个“只允许绝对路径”的策略。

## 可复用建议

- **作为通用中间层**：将包装器做成一个可配置的小工具，发布到内部工具库，任何面向 Agent 的命令行调用都先经过它。提供 YAML 配置指定允许的根目录、允许的脚本清单，甚至可以配置黑名单路径（如 `/etc`、`/proc`）直接拒绝，即使它们在白名单内（例如白名单是 `/`，当然不建议但可兜底）。
- **与 MCP 集成**：如果使用的是 MCP 工具，可以在工具实现中调用此包装器逻辑，而非另起一个子进程，从而减少开销，也更容易统一日志与审计。
- **日志告警**：拒绝访问时，将尝试路径、时间戳、调用者信息打入日志，配合监控，可以及时发现 Agent 异常行为。
- **渐进增强**：先上包装器，后续结合 Linux 的 `Landlock` 或 `bubblewrap` 做内核级限制，形成纵深防御。特别是在容器内，可以挂载只读的根文件系统，然后只将白名单目录以读写方式 bind mount 给子进程。

## 总结

给自动化脚本加本地目录白名单，不必追求绝对安全的沙箱，用一行轻量包装器就能解决大部分命令行注入风险。核心思路是标准化路径后做前缀匹配，拦截已知的参数传入，同时对符号链接、相对路径等常见绕过手段提前化解。在 OpenClaw Agent 的实际使用中，这层护栏既降低了出事故的概率，也为后续更严格的安全措施留出了接入点。用一个不到 100 行的 Python 文件，就能让模型驱动的自动化脚本在受控的边界内运行。

---

