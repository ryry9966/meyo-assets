---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 31276
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景：tools.md 不只是工具列表

在 OpenClaw 和基于 MCP 的 Agent 工程里，`tools.md` 常常被当作一个简单的工具声明文件，用来告诉 Agent 可以调用哪些外部能力——比如读写文件、执行 Shell、查询数据库。但真正落地时你会发现，**同一个 Agent 在开发者笔记本上跑得好好的，到了同事机器或者线上容器里就出现“找不到工具”“环境变量缺失”“路径错误”**。根源在于：`tools.md` 里悄悄耦合了大量本机特有的配置，却没有处理好环境差异。

这并不是 OpenClaw 的缺陷，而是所有本地 Agent 工程的共性问题。Agent 不是纯云端的函数调用，它需要触碰真实的本地运行时：Node 版本、Python 虚拟环境、系统 CLI、API Key、文件路径……这些都不属于 Agent 逻辑本身，却决定了工具能不能用、怎么用。因此，**tools.md 应该是声明式的“工具合约”，而非硬编码的机器快照**。

---

## 问题：隐式依赖与无声失败

常见的问题模式：

1. **环境变量内嵌**：直接把 `OPENAI_API_KEY=sk-xxx` 写在 tools.md 里，导致无法提交版本库，或换了环境忘记替换。
2. **路径硬编码**：`python: /usr/bin/python3` 在 macOS 上能用，到 Ubuntu 就变成 `/usr/bin/python3.11`。
3. **工具缺失无感知**：Agent 执行时才发现 `ffmpeg` 没装，浪费 token 和时间。
4. **跨平台陷阱**：Windows 的 `\` 与 Unix 的 `/` 路径分隔符，shell 命令的差异（如 `copy` vs `cp`），会让一个本应跨平台的 Agent 配置变得脆弱。
5. **环境间行为不一致**：开发环境用 Docker，本地调试用 venv，但 tools.md 没有区分，导致预检失效。

这些问题积累下来，成员会花大量时间在“为什么你那里跑得通我这里不行”的排查上，Agent 的可靠性大打折扣。

---

## 做法：把 tools.md 变成环境适配层

### 1. 声明工具契约，分离运行时配置

不要在 tools.md 里写任何具体路径或秘钥，而是用环境变量占位符。例如：

```yaml
tools:
  - name: "image_convert"
    command: "${IMAGEMAGICK_BIN:-convert}"
    env:
      MAGICK_HOME: "${IMAGEMAGICK_HOME}"
    fallback: "echo 'Please install ImageMagick' && exit 1"
  - name: "run_python"
    command: "${PYTHON_BIN:-python3}"
    working_dir: "${PROJECT_ROOT}"
    pre_check: "${PYTHON_BIN:-python3} -c 'import sys; assert sys.version_info >= (3,10)'"
```

这样做的好处：
- 在不同机器上通过 `.env` 或环境注入提供实际值，不会污染配置。
- 提供了 `fallback` 和 `pre_check`，让 Agent 在工具加载阶段就能发现问题，而不是执行到一半崩溃。

### 2. 环境分层：一个 tools.md 驱动多套环境

推荐方案：`tools.md` 作为基础模板，通过变量切换环境上下文。

```bash
# .env.local
IMAGEMAGICK_BIN=magick
PYTHON_BIN=./venv/bin/python
PROJECT_ROOT=./workspace
```

```bash
# .env.prod
IMAGEMAGICK_BIN=convert
PYTHON_BIN=/usr/bin/python3.11
PROJECT_ROOT=/app/data
```

然后在启动 OpenClaw 时指定环境文件：`openclaw run --env .env.prod`。tools.md 里的每个工具都会按照当前环境解析实际命令。

如果你需要针对不同平台（Windows/Linux/macOS）做适配，可以在 tools.md 里写简单的条件块（如果框架支持），或者在外部做一次工具集的预处理脚本。

### 3. 组件化工具集，避免单文件膨胀

不要把所有工具塞进一个巨大的 tools.md。可以拆分成：
- `tools/base.md`：核心通用工具（文件、搜索、脚本执行）
- `tools/ai.md`：AI 相关工具（需要 GPU、特定模型）
- `tools/media.md`：图像、音频处理（依赖系统库）

OpenClaw 项目里通过 `includes` 或合并机制加载这些文件，方便团队按需启用，也更容易排查某个工具的环境依赖。

### 4. 自动化可用性校验（Pre-flight Check）

写一个独立的 `validate_tools.sh`（或 Python 脚本），读取 tools.md 并执行每个工具的 `pre_check` 命令。这个脚本可以放到 Git Hook 或 CI 里，团队成员拉代码后运行一次就能立刻知道本机缺少什么。

伪代码：
```python
for tool in load_tools("tools.md"):
    if "pre_check" in tool:
        rc = os.system(tool["pre_check"])
        if rc != 0:
            print(f"[FAIL] {tool['name']}: pre_check failed")
            print(f"  Suggestion: {tool.get('fallback')}")
```

这样可以彻底消灭“运行时才发现环境不对”的痛苦。

---

## 踩坑点

1. **环境变量未导出导致 Agent 拿不到值**：注意进程间的传参方式，确保你用的是 `export` 或在 systemd 里设置 `EnvironmentFile`，而不是仅在 shell 会话里设变量。
2. **fallback 太安静了**：有些团队习惯把 fallback 写成 `echo "error"`，Agent 调用时可能根本注意不到错误信息。fallback 应给出明确的退出码和可被 Agent 理解的错误协议（如 JSON 格式）。
3. **Windows 下的命令解释器**：如果 `command` 是原生 shell 命令，务必指明 `shell: true` 或使用 `cmd /c` 来封装，否则在 Linux 容器里用 `/bin/sh -c` 可能不兼容。
4. **路径中带空格**：没有加引号时会断裂，建议在 `command` 前面统一使用数组格式，如 `["${PYTHON_BIN}", "script.py"]`，避免字符串拼接。
5. **工具版本漂移**：pre_check 应检查主版本号，例如 `ffmpeg -version | grep -q "4\."`，确保不因为系统升级引入不兼容变化。

---

## 可复用建议

- **把 tools.md 纳入版本控制**，但绝不含任何秘钥或绝对路径。
- **随仓库提供 `tools.env.example`**，列出该 Agent 需要的全部环境变量，并注明哪些是必填、哪些有默认值。
- **在 CI 中使用矩阵测试不同环境组合**（不同 Python 版本、不同操作系统），确保 tools.md 的抽象层真的跨平台。
- **如果用到 MCP 服务器**，将 MCP 的 `command` 和 `args` 也按照同样原则暴露为可配置项，在 tools.md 中引用，而不是写死在 `claw.conf` 里。
- **给每个工具加一个 `description` 和 `usage_example`**，不仅方便人类阅读，Agent 本身也能利用这些元信息更准确地决定何时调用。

---

## 总结

`tools.md` 是 Agent 与本地世界之间的桥梁。一个工程化的 `tools.md` 不应该是“能跑就行”的快照，而是一份清晰的合约：**定义工具需要什么、环境必须提供什么、异常时如何降级**。当你把它从配置文件的角色提升为“环境适配层”，Agent 的可迁移性和稳定性会得到质的改善。在 OpenClaw 这类框架中，先花半小时搭好分层、校验和 fallback 机制，远胜过后期在无数个“环境问题”上报错中来回救火。

---

