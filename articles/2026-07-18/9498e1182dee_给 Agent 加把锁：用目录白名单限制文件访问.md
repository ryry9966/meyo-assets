---
title: 给 Agent 加把锁：用目录白名单限制文件访问
feedId: 29458
source: 综合讨论
publishedAt: 2026-07-18
---

# 给 Agent 加把锁：用目录白名单限制文件访问

## 背景：当 Agent 拥有文件系统能力

在 OpenClaw 这类框架中，Agent 可以通过调用本地工具完成复杂任务，例如读写文件、整理目录、生成代码并本地运行。这种能力极大提升了自动化效率，但同时也带来了文件安全风险。一个没有约束的自动化脚本，可能因为提示词偏差、意外输出或逻辑错误，把敏感文件当作临时文件清理掉，或者在错误路径下生成大量垃圾文件。

典型的防护思路是使用沙箱或容器隔离，但在桌面自动化、个人助手等场景中，完全隔离并不现实——Agent 常常需要访问用户的文档、项目、配置目录。于是，我们需要一种更轻量的方案：**目录白名单护栏**。

## 问题拆解

假设我们给 Agent 暴露了一个 `write_file(path, content)` 或 `run_shell(cmd)` 工具。如果不加限制，它就可以写入 `~/.ssh/authorized_keys`、修改 `/etc/hosts`，或者删除 `~/Documents` 下面的重要资料。即便 Agent 本意是好的，一条 `rm -rf $TMP_DIR/*` 而 `$TMP_DIR` 意外为空的情况，就可能导致灾难。

真正棘手的是，Agent 可能使用相对路径、符号链接、路径穿越（`../`）等方式绕过简单的字符串匹配。简单地检查路径是否以 `/safe/dir` 开头并不够，需要解析真是路径后再校验。

## 做法：给文件操作加上透明护栏

我们的目标是在工具实现层，对 **所有文件写入/删除/执行操作** 施加白名单校验，只允许在指定目录集合内生效。下面以 OpenClaw 中自定义 Python 工具为例（MCP 服务端或插件也适用相同原则）。

### 第一步：定义白名单与解析函数

```python
import os
from pathlib import Path

ALLOWED_DIRS = [
    os.path.expanduser("~/projects/automation"),
    "/tmp/agent_workspace"
]

def is_path_allowed(target_path: str) -> bool:
    # 解析真实绝对路径，消除符号链接和 ../
    real_path = os.path.realpath(os.path.abspath(target_path))
    for d in ALLOWED_DIRS:
        real_dir = os.path.realpath(d)
        # 确保 real_path 在 real_dir 子树内
        if real_path == real_dir or real_path.startswith(real_dir + os.sep):
            return True
    return False
```

要点：必须使用 `os.path.realpath()` 解析，否则符号链接可轻松绕过。不要忘记追加路径分隔符，避免 `/safe/dir_bad` 被误判。

### 第二步：封装安全的文件操作函数

在工具函数入口处调用校验：

```python
def safe_write_file(path: str, content: str) -> str:
    if not is_path_allowed(path):
        raise PermissionError(f"Access denied: {path}")
    os.makedirs(os.path.dirname(os.path.abspath(path)), exist_ok=True)
    with open(path, 'w') as f:
        f.write(content)
    return f"Written to {path}"
```

对于 `delete_file`、`rename`、`copy` 等操作也需要同样处理。如果是暴露 `run_shell` 给 Agent，直接拦截命令几乎不可能，更好的做法是为 shell 命令提供一个受限的临时工作目录，并通过环境变量传递允许目录列表，由内部脚本再校验，但这已超出本文范围。建议尽量不直接开放 Shell，而是封装细粒度工具。

### 第三步：集成到 OpenClaw 工具注册

在 OpenClaw 的工具定义中，调用上述安全函数：

```python
from openclaw.tools import tool

@tool(
    name="write_note",
    description="将笔记内容写入 markdown 文件",
    parameters={
        "type": "object",
        "properties": {
            "filename": {"type": "string"},
            "content": {"type": "string"}
        }
    }
)
def write_note(filename: str, content: str):
    return safe_write_file(filename, content)
```

这样，Agent 对文件系统的每一次写入都会经过白名单闸门。

## 踩坑点记录

1. **符号链接攻击**  
   如果仅使用 `os.path.abspath` 而未使用 `realpath`，那么指向 `/etc/passwd` 的符号链接放在允许目录内就能绕过检查。务必同时调用 `realpath`。

2. **创建新文件时的目录检查**  
   `is_path_allowed` 检查的是最终路径，但如果允许创建文件，父目录也必须被允许。可以在写文件前额外调用 `os.makedirs`，但要确保父目录同样在白名单内，或在创建目录时也加校验。

3. **跨平台分隔符问题**  
   Windows 下使用 `os.sep` 和路径比较需谨慎。`Path` 对象提供 `is_relative_to()`（Python 3.9+）可以简化子路径判断。在 Windows 上特别注意盘符和长路径。

4. **规范化与实际存在的竞争**  
   `realpath` 要求路径已经存在才能解析符号链接。对于还不存在的文件路径，可先对存在的最近父目录调用 `realpath`，再拼接剩余部分，然后再次检查。例如：
   ```python
   parent = os.path.dirname(os.path.abspath(path))
   while not os.path.exists(parent):
       parent = os.path.dirname(parent)
   real_parent = os.path.realpath(parent)
   resolved = os.path.join(real_parent, os.path.relpath(path, start=parent))
   ```

5. **白名单目录的粒度控制**  
   过于宽泛的白名单（如允许整个 `~/Documents`）会让防护形同虚设。建议根据实际任务使用项目级临时目录或专门的工作区目录。

## 可复用建议

- **多层防御**：目录白名单只是第一层。对特别敏感的操作，加上写入大小限制、文件类型限制（禁止覆盖 `.ssh`, `.env`）和操作日志记录。
- **集中封装**：将所有文件操作通过统一的 `FileGuard` 类暴露，避免各处散落校验代码。
- **测试用例**：用 `pytest` 编写测试，覆盖符号链接绕过、路径穿越、不存在的文件、权限错误等场景，确保护栏健壮。
- **向用户透明**：在工具描述中明确告知 Agent 的访问范围，有助于调试，也让用户清楚 Agent 的边界。
- **配合审计日志**：记录每一次被拒绝的访问请求，帮助用户发现 Agent 的不当行为趋势或提示词问题。

## 总结

给自动化脚本加上本地目录白名单，是平衡功能与安全的务实手段。核心在于**强制解析真实路径**，并确保所有文件写入路径都经过同一闸门。虽然实现不复杂，但细节处理决定是否真的“安全”。在 OpenClaw 或类似 Agent 框架中，将这种护栏内建为基础设施，可以大幅降低误操作风险，让自动化更加可信。

实际工程中，这个方案可以和只读文件系统快照、定期备份配合使用，形成更完整的文件安全体系。希望这个思路能为你的 OpenClaw 工作流加上一把可靠的锁。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/0c9d5dee435910ed.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/3e50399cc60b3d2f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/34eec26fb1085cda.png)

