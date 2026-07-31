---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 31071
source: 综合讨论
publishedAt: 2026-07-31
---

## 背景

在 OpenClaw 这类 Agent 框架里，`tools.md` 是定义工具能力的主入口：列出可用脚本、API 地址、密钥引用、本地依赖等。当你把一个 Agent 从笔记本迁移到 CI 服务器，或者交给另一个开发者运行时，最常见的故障不是逻辑错误，而是环境差异——路径不对、Python 版本不同、依赖缺失、鉴权 Key 找不到。

这些“明明在我机器上能跑”的问题，根源往往在于 `tools.md` 对环境做了隐式假设。本文不讨论如何写工具元数据，只聚焦一个痛点：**如何让 tools.md 即便在不同操作系统、不同账号、不同安装路径下都能稳定交付正确的配置**。

## 问题拆解

一个典型的“裸写” tools.md 长这样：

```yaml
tools:
  - name: data_fetch
    command: python /home/alice/projects/fetch_data.py --token ${API_TOKEN}
    requirements:
      - python>=3.9
      - httpx
```

问题很明显：

- 硬编码了 `/home/alice/projects/...`，换个用户直接报 FileNotFound。
- `API_TOKEN` 期望从环境中读取，但没有说明默认行为或缺失时的回退。
- `requirements` 只是声明，Agent 启动时并不会真正检查 python 版本，或者检查了但不提示下一步动作。

在日常工程中，还会出现：
- 同一台机器上多个 Agent 共用 global 配置，导致工具冲突；
- Mac/Linux 路径斜杠、换行符差异导致命令解析失败；
- 本地调试需要 mock 服务，线上需要真实服务，但 tools.md 只能写死一个地址。

## 实践方案：三层分离与环境遮蔽

核心思路是**把 tools.md 当成逻辑清单，而不是环境快照**。硬件环境、密钥、本地依赖等差异用三层结构隔离：

**第一层：tools.md 作为逻辑声明**  
只定义“这个工具要做什么”，不写死“在哪里”和“怎么连”。使用相对路径和命名约定，让执行器去解析。

```yaml
tools:
  - name: data_fetch
    command: python ${TOOL_DIR}/fetch_data.py --token ${API_TOKEN}
    env:
      - API_TOKEN
    prefer:
      - local_env_override.sh
```

这里 `${TOOL_DIR}` 由 Agent 启动脚本动态注入，而不是写死绝对路径。`prefer` 指向一个可选的本地覆盖脚本，用于设置环境变量或 alias。

**第二层：local_env_override.sh（不入库）**  
这个脚本由开发者本地维护，放在 `.gitignore` 中。内容例如：

```bash
export TOOL_DIR="/home/alice/projects/tools"
export API_TOKEN="local_test_token_123"
alias python=python3.11
```

Agent 启动时会 source 这个脚本（如果存在），从而实现环境遮蔽。线上部署时，CI 系统通过 secrets 注入同名变量，无需覆盖文件。

**第三层：启动脚本的兜底逻辑**  
Agent 的入口脚本（比如 `run_agent.sh`）负责按顺序加载：

1. 检查 `local_env_override.sh` 是否存在，存在则 source。
2. 将 `tools.md` 中声明需要注入的变量与实际环境对比，缺的报错，给出可读提示。
3. 用 `command -v python` 确认解释器存在，否则提示安装或指向当前可用的解释器。

这种做法避免了过度依赖 Docker 的“隔离一切”思路——开发时你仍然可以用本地最快的 Python 和缓存，同时保持 `tools.md` 的移植性。

## 踩坑记录

1. **变量展开时机混淆**  
   如果在 `tools.md` 里用了 `${VAR}`，但实际是 Agent 的某段 Python 代码去 `os.system` 执行的，那这些变量必须在 Python 进程环境中可见。如果你是在 shell 里 export，启动 agent 时没继承，就会拿到空值。解法是统一在启动脚本中 export，或用 Python 的 `shlex` 预先替换。

2. **Windows 路径陷阱**  
   即使你用 Python，当 command 是 shell 字符串时，反斜杠会被转义。`path\to\script.py` 会变成 `path	o\script.py`。可以使用正斜杠或始终用 `pathlib` 拼接，在 tools.md 里直接写相对路径，让 Python 代码在运行时解析绝对路径。

3. **requirements 检查“假阳性”**  
   很多 Agent 框架只是打印 `requirements` 但不执行检查。你写了 `python>=3.9`，实际线上容器里是 3.8，它不报错，直接运行工具时才抛出语法错误。建议在启动脚本里显式校验关键依赖，失败时给出可操作的错误信息，例如：`python3 -c 'import sys; assert sys.version_info >= (3,9)' || echo "请升级 Python >=3.9"`。

4. **多 Agent 竞态**  
   如果两个 Agent 共用同一个 `local_env_override.sh`，其中一个修改了全局 `PATH` 或 `PYTHONPATH`，可能导致另一个工具行为异常。给每个 Agent 独立的 override 文件，或让 Agent 在启动时 copy 一份干净的初始环境并只做私有修改。

## 可复用建议

- **模板化 tools.md**：提供带占位符的 `tools.md.tmpl`，在 CI 或本地 generate 阶段根据平台填充变量。这样原始声明依然纯净。
- **版本控制策略**：`tools.md` 入仓库，`local_env_override.sh` 不入仓库。提供一个 `local_env_override.sh.example` 作为脚手架文件，注释中写明每个变量的含义和可选值。
- **自检命令**：给每个 Agent 加一个 `--check` 参数，只运行环境检查部分，不在生产环节报路径错误。
- **跨平台路径**：所有路径尽量使用 `python -c 'import pathlib; print(pathlib.Path.cwd())'` 类动态获取，避免写死 `/home` 或 `C:\Users`。

## 总结

`tools.md` 应该是一份“运行期望书”，而不是“环境快照”。一旦你开始把绝对路径、本机权限或临时密钥写进去，复用成本就会成倍增加。三层分离（逻辑声明 + 本地覆盖 + 启动兜底）可以让 Agent 在笔记本、CI、同事的机器上都以相同姿势生效，同时保留本地开发的便捷性。最终的目标是：你只关心工具做了什么，而不必关心它在哪台机器上运行。

---

