---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 32848
source: 综合讨论
publishedAt: 2026-08-13
---

在 OpenClaw 这类本地 Agent 实践中，最常翻车的不是模型能力，而是环境差异：同一套任务，在 macOS 上能跑，换到 Linux 就找不到命令；明明装了 ffmpeg，但 Agent 调用了错误版本；或者直接把 API key 写进工具说明。tools.md 的价值，就是给 Agent 一份可检查的“本地环境契约”。

## 背景

Agent 执行任务时会自主选择命令、路径和工具。如果缺少明确的 tools.md，它会基于训练数据猜测，导致：

- 使用未安装或错误版本的工具
- 不同机器上行为不一致
- 环境变量、路径、shell 差异引发隐性失败
- 越权操作或污染全局环境

tools.md 不是给用户看的帮助文档，而是给 Agent 的运行时契约。它和 MCP 的差异在于：MCP 定义工具协议，tools.md 定义本地执行边界。

## 问题

很多团队把 tools.md 写成“工具名称列表”，或者只写“可以用 python、node”。这基本没用。真正的 tools.md 要回答：什么工具可用、什么版本、在哪、怎么确认、边界在哪、失败怎么办。

## 做法/步骤

### 1. 定位与分层

tools.md 是“能力清单 + 环境契约”，不是脚本，也不是 README 的复制。

建议分层管理：

- `tools.default.md`：项目级默认约定，提交版本库
- `tools.local.md`：本地覆盖，不提交版本库
- 临时覆盖：任务 prompt 中声明

### 2. 字段模板

每个工具至少包含这些字段：

```markdown
## 工具：ffmpeg
- 状态：可用
- 版本：6.1.1
- 路径：/opt/homebrew/bin/ffmpeg
- 检查命令：ffmpeg -version | head -n1
- 环境变量：无
- 允许操作：转码、截帧、合并音视频
- 禁止操作：写入系统目录、覆盖源文件
- 失败回退：若无 ffmpeg，提示用户安装或改用 imageio[ffmpeg]
```

环境变量只写名称，不写值。这样既能告知 Agent 需要哪些变量，又不会泄露敏感信息。

### 3. 强制 preflight 检查

在 system prompt 或项目规则中声明：

> 任务开始前必须读取 tools.md，并对将要使用的工具执行检查命令。若检查失败，不要继续，先报告差异。

这样能避免 Agent 拿着一个不存在的工具硬编命令。

### 4. 动态生成 tools.local.md

写一个脚本，自动检测本机工具并生成本地覆盖文件，避免手写漂移：

```bash
for cmd in python node git ffmpeg jq; do
  if command -v $cmd >/dev/null 2>&1; then
    echo "- $cmd: $(command -v $cmd) ($($cmd --version 2>/dev/null | head -n1))"
  else
    echo "- $cmd: NOT_FOUND"
  fi
done
```

## 踩坑点

**1. 把 secrets 写进 tools.md**

tools.md 可能被提交、被日志引用、被 Agent 全文读取，绝不能包含 API key、token、密码。环境变量只写名称，值由系统注入。

**2. 版本号写死但环境未锁定**

例如写 `python 3.12`，但本地实际是 3.10，Agent 依然会按 3.12 语法执行。应写“检测到的版本”并让 Agent 以检测结果为准。

**3. 使用绝对路径导致不可移植**

本地 `/Users/alice/bin/yq` 换机器就失效。优先使用 `command -v` 或环境变量，并允许 fallback。

**4. 只写工具不写禁用项**

应明确“禁止使用系统包管理器全局安装”“禁止修改 `.bashrc`”“禁止删除用户目录”。边界比能力更重要。

**5. 文档与真实环境漂移**

tools.md 写“可用”，实际已卸载。应定期运行校验脚本，或让 Agent 在每次任务开始前做 preflight，用检查命令确认实际状态。

## 可复用建议

- 保持 tools.md 短小，只写差异。通用工具不必逐项罗列，重点写“特殊路径、版本差异、禁用项”。
- 用 `tools.local.md` 放机器级差异，不提交版本库。
- 给 Agent 提供“不可用工具”清单和替代方案，例如“没有 jq 时用 python -c”。
- 结合 MCP/插件：如果通过 MCP 暴露工具，tools.md 只需说明 MCP 服务名和参数边界，不必重复实现细节。
- 定期用 CI 或本地脚本检查 tools.md 中声明的工具是否仍存在，版本是否匹配。

## 总结

tools.md 的目标不是写更多文档，而是把环境差异收敛到一个 Agent 会读取、会检查、会遵守的契约里。好的 tools.md 让 Agent 少猜、少错、少越界。它不解决所有问题，但能让“本地能跑，换台机器就崩”的概率明显下降。

把 tools.md 当成基础设施的一部分：可检测、可验证、可回退，而不是静态说明。这样本地 Agent 才能真正稳定地复用你的环境，而不是每次都在猜你的环境。

---

