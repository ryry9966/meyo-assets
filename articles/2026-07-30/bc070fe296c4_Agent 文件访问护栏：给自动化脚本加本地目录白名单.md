---
title: Agent 文件访问护栏：给自动化脚本加本地目录白名单
feedId: 30953
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景
在 Agent 自动化链路中，我们经常需要让大模型调用本地脚本来完成具体任务：图片压缩、日志归档、临时文件清理、数据导出等。这些脚本通常运行在与 Agent 同机的环境中，通过 MCP 工具或自定义插件暴露给模型。一旦 Prompt 注入、模型幻觉或用户恶意构造参数，脚本就可能越权访问 `/etc/passwd`、`~/.ssh` 或其他敏感路径。对于直接操作文件系统的工具，最小权限原则不能只靠 prompt 约束，必须在执行层硬控——**本地目录白名单**是成本最低、见效最快的护栏。

## 问题定义
假设你为 OpenClaw 编写了一个压缩脚本 `archive_logs.py`，接收参数 `--source-dir /var/log/app`。如果模型传入了 `--source-dir /`，而脚本内部只是简单地执行 `tar -czf ... $SOURCE_DIR`，后果不可控。我们需要在脚本内部强制保证：**所有文件操作只能发生在预定义的工作目录下**。这个约束不依赖调用方自觉，而应由执行环境（脚本自身）做最终裁决。

## 做法与步骤
### 1. 约定工作区根目录
通过环境变量或配置文件指定允许访问的根目录，例如：
```bash
export WORKSPACE_DIR=/data/agent-workspace
```
脚本只被允许在该目录内读写。

### 2. 实现路径规范化与白名单校验
核心逻辑：将用户传入的路径转换为绝对真实路径，再判断是否以 `WORKSPACE_DIR` 开头，且必须紧接路径分隔符（或完全相等），防止前缀欺骗，例如 `/data/agent-workspace` 与 `/data/agent-workspace-evil`。

**Python 版本（推荐）**
```python
import os
import sys

def safe_resolve(user_path: str, workspace: str) -> str:
    # 解析所有符号链接，得到绝对路径
    real_path = os.path.realpath(user_path)
    real_workspace = os.path.realpath(workspace)
    # 使用 commonpath 做最安全的前缀判断
    if os.path.commonpath([real_workspace, real_path]) != real_workspace:
        raise PermissionError(f"Access denied: {real_path} is outside {real_workspace}")
    return real_path
```

在脚本入口处检查所有文件类参数：
```python
workspace = os.environ.get("WORKSPACE_DIR", "/data/agent-workspace")
source_dir = safe_resolve(args.source_dir, workspace)
# 后续安全使用 source_dir...
```

**Shell 版本（简化但需谨慎）**
```bash
#!/bin/bash
WORKSPACE="${WORKSPACE_DIR:-/data/agent-workspace}"
target=$(readlink -f "$1")
workspace_real=$(readlink -f "$WORKSPACE")
if [[ "$target" != "$workspace_real"* ]] || [[ "${target#$workspace_real}" =~ ^[^/] ]]; then
  echo "Denied: $target outside $workspace_real" >&2
  exit 1
fi
```
注意条件判断要处理“前缀但非目录边界”的情况。

### 3. 集成到 Agent 工具描述
给模型的 tool definition 中明确说明：
```
功能：压缩指定目录下的文件
限制：只能操作 /data/agent-workspace 内的路径，传入外部路径将返回权限错误。
```
工具描述本身就是第一层提示护栏，而脚本内的是硬护栏。两层配合让行为更可预测。

### 4. 增加审计日志
所有违规尝试都应记录到 stderr 或专用日志文件，并包含时间戳、目标路径、调用链（可通过环境变量注入 `AGENT_TASK_ID`），便于事后追溯。

## 踩坑点
- **符号链接陷阱**：即使传入的是白名单内的路径，该路径下可能有指向外部的符号链接。`os.path.realpath` 会跟踪到底，可能导致合法路径解析后落在工作区外。需要根据业务决定：是拒绝穿越链接（可额外检查每个分段是否为符号链接），还是允许但追加说明。通常建议拒绝，并提示用户移除链接。
- **相对路径与 `..`**：绝对不要直接拼接相对路径。一律先 `realpath`/`readlink -f` 再校验。
- **竞态条件**：检查通过后到实际文件操作前，路径可能被替换（TOCTOU）。对高安全场景，应使用基于文件描述符的操作（如 `openat`），但复杂度大增。大部分运维级 Agent 场景中，先 resolve 后操作已足够，同时保证工作目录权限仅为 agent 专用用户可写。
- **Windows 兼容**：`os.path.realpath` 在 Windows 下也有效，但盘符处理和路径分隔符需要额外注意，最好统一用 `pathlib.Path.resolve()` 并比较 `pathlib.Path` 对象。
- **环境变量未设置**：脚本应预设一个严格默认值（如当前目录的子目录），避免因遗漏设置导致全盘可访问。

## 可复用建议
1. **封装成通用函数库**：将校验逻辑抽成 `guard.py` 或 `guard.sh`，被所有本地工具脚本 source/import。每次新增工具只需调用 `check_path`，统一维护。
2. **限制脚本能力**：白名单之外，还可用操作系统级能力（如 `systemd` 的 `ProtectSystem=strict`、`ReadOnlyPaths`、Docker 的 `--read-only` 和 volume 映射）作为纵深防御。但脚本层白名单是最细粒度、最独立的一环。
3. **拥抱现有方案**：如果工具是通过 MCP server 暴露的，可以直接使用 `mcp-server-filesystem` 的 `allowed_directories` 配置，它已经实现了安全的路径检查。本文的方法适用于需要定制逻辑或非 MCP 工具链的场景。
4. **测试用例**：针对常见绕过手法编写单元测试：包含 `../` 的路径、绝对路径、符号链接、拼接欺骗（`/workspace` vs `/workspace2`），确保校验逻辑健壮。

## 总结
为 Agent 调用的本地脚本加上目录白名单，是把“信任模型”转变为“验证模型”的关键一步。它不需要重量级安全框架，只需要几十行代码，就能杜绝绝大部分越权文件访问。在每一次脚本开发中，都把安全边界放在首位——先 resolve，再 commonpath，最后操作。这个习惯将让你的 Agent 生产环境多一道可靠的护城河。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/3294e04c874e5813.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/f9488fc09b6bab4b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/b511a51ff0a4edac.png)

