---
title: 给 Agent 脚本加一把锁：文件访问白名单的工程化落地
feedId: 30994
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景

在搭建基于 MCP 的本地 Agent 或自动化工具链时，我们经常需要让脚本访问文件系统——读配置、写日志、拉取数据，甚至通过 Tool 定义让大模型直接操作文件。这类实践在 OpenClaw 生态中很常见：一个 Python MCP Server 注册了 `read_file`、`write_file` 方法，Agent 在对话中就能调用它们。

问题随之而来：如果没有任何限制，一次 prompt 注入、模型幻觉或者参数拼接过宽，就可能读走 `~/.ssh/id_rsa`，覆盖系统配置，或者在 `/tmp` 下写入恶意文件。这不是“理论上可能”，在给客户部署的自动化流程中，我亲眼见过一个调试脚本因传入了 `../../.env` 把生产环境密钥打到了日志里。

所以，我们需要一个最小、可控的 **文件访问护栏**——让 Agent 只能访问我们指定的目录，其余路径一律拒绝。本文讨论如何用几十行 Python 实现一个工程上可用的本地目录白名单，以及实际落地时踩过的坑。

## 问题拆解

目标很明确：提供一个校验函数 `is_safe_path(path, allowed_roots)`，传入任意用户输入路径，判断其是否真正落在管理员指定的目录树下。听起来简单，但真正“安全”需要处理以下情况：

1. 相对路径如 `../etc/passwd` 跳出白名单根目录；
2. 符号链接将 `/tmp/link` 指向 `/etc/shadow`；
3. 文件系统挂载点或 bind mount 导致的隐形越界；
4. 路径字符串大小写问题（Linux 的 ext4 大小写敏感，macOS 默认不敏感）；
5. Windows 下盘符、UNC 路径、短文件名等特殊情况。

在工程实践中，我们需要权衡实现复杂度与安全级别，通常以 **“阻止无意的越界访问”** 为底线，对符号链接等风险点做显式处理。

## 实现步骤

以下以 Python 3.10+ 为例，给出一个可直接复用的核心函数：

```python
import os
from pathlib import Path

def is_safe_path(candidate: str, allowed_roots: list[str]) -> bool:
    """
    检查候选路径是否安全落在 allowed_roots 中的某个目录下。
    使用真实路径解析，阻止符号链接逃脱。
    """
    if not candidate.strip():
        return False

    # 将 allowed_roots 转为真实路径的 Path 对象
    safe_roots = [Path(root).resolve() for root in allowed_roots]
    target = Path(candidate)

    # 如果路径存在且是符号链接，解析真实位置
    if target.is_symlink():
        target = target.resolve()
    else:
        # 即使不存在，也展开 ".." 等，避免路径穿越
        target = target.resolve(strict=False)

    # 对不存在的路径，resolve 会拼接 CWD，需验证解析后的绝对路径
    # 确保目标恰好位于某个根目录下
    for safe_root in safe_roots:
        try:
            target.relative_to(safe_root)
            return True
        except ValueError:
            continue

    return False
```

在实际 MCP 工具中集成：

```python
ALLOWED_ROOTS = ["/home/user/sandbox", "/data/app-logs"]

def safe_read_file(path: str) -> str:
    if not is_safe_path(path, ALLOWED_ROOTS):
        raise PermissionError(f"Access to {path} is not allowed.")
    return Path(path).read_text()
```

## 踩坑记录

### 1. 符号链接旁的“幽灵路径”

有一次测试时，`/tmp/safe` 被允许，里面有个链接 `link` 指向 `/etc`。如果用户传入 `/tmp/safe/link/passwd`，我们代码只判断 `/tmp/safe/link` 是不是符号链接，但它的子路径 `passwd` 是在已经跳转后的 `/etc` 里。更稳妥的做法是在遍历父目录阶段就截断，或者检查每一个路径组件。但性能开销太大。我们最终选择调用 `Path.resolve()` 后再验证，并在文档中明确：**白名单目录内不应存在指向外部的符号链接，否则风险自负。**

### 2. `resolve(strict=False)` 的危险诱惑

早期实现用了 `strict=False` 去解析可能还不存在的路径（如即将创建的新文件）。这在 Python 3.6 之前版本行为不一致，可能返回部分解析的结果，留下了绕过可能。从 3.8 起逐渐稳定，但最安全的方式是先解析父目录：`parent.resolve(strict=True)` 检查父目录安全，再拼接文件名，避免对不存在路径的依赖。可以在性能允许时使用该方法。

### 3. macOS 大小写陷阱

在 Mac 上，`/tmp/App/` 和 `/tmp/app/` 默认指向同一文件夹，但 `Path.relative_to()` 按字符串比较会失败，因为一个是大写 A，一个是小写 a。此时需要 `Path.resolve()` 返回实际路径，它会转为文件系统真实的大小写形式。所以上面的 `resolve()` 步骤不可或缺。

### 4. 临时目录爆破

`/tmp` 通常 777 权限，如果不小心将白名单扩大到 `/tmp`，可能被用来写入大量文件耗尽磁盘。建议白名单使用独立挂载的分区或子卷，配合磁盘配额；或者至少在 `/tmp` 下创建专用前缀目录并在 Agent 启动时创建，不允许访问 `/tmp` 根。

## 可复用建议

1. **先做路径黑名单而非仅白名单**  
   有些运维脚本动态需要访问大量预定义目录，白名单维护困难。可以补一层“关键目录黑名单”如 `/etc`、`/root`、`~/.ssh`，再结合白名单。但在安全关键场景，白名单是首选。

2. **与容器 / chroot 配合**  
   最坚固的防线应当由 OS 提供。我的推荐方案是：Agent 进程运行在 `chroot` 或 Docker 容器内，将白名单目录 bind mount 进去，这样即使 Python 侧失误，系统调用也会被内核阻止。然后 Python 层再做一次路径校验作为深度防御。

3. **MCP 层面的 control plane**  
   如果你用 OpenClaw 等客户端搭建 Agent，可以将文件访问白名单声明为一个 **capability policy**，由客户端侧在调用工具前过滤，这样即使 Server 端疏漏，也不会把危险路径传过去。客户端检查路径是否在白名单内，比 Server 自己检查多一层保险。

4. **审计日志不打印完整路径**  
   错误日志中打印被拒绝的路径可能泄露用户目录结构。生产环境应该只记录“Access denied”，或记录归一化后的路径 hash。

## 总结

给 Agent 加上文件访问白名单不是“高精尖”技术，而是一个容易忽视但出问题代价极大的工程细节。几十行 Python 能挡住绝大多数无意的越界，但要应对故意的绕过，需要理解文件系统的解析特性，并将 OS 级别的隔离作为基石。建议每一位给 Agent 开放文件写入能力的朋友，在第一个 Tool 上线前，先把这道护栏搭好。代码可以很简单，但工程态度不能马虎。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/f6d90bd860346c6d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/577e9dbfdf06bca8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/0dae16f49360f716.png)

