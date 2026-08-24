---
title: Agent 的 tools.md：把本地环境差异关进笼子
feedId: 34464
source: 综合讨论
publishedAt: 2026-08-24
---

# Agent 的 tools.md：把本地环境差异关进笼子

## 背景
OpenClaw 这类本地 Agent 最常做的事情是：跑一条命令、写个脚本、调个 CLI。但本地环境从来不是一致的——有人用 macOS，有人用 Linux 容器，Python 可能是 3.11 也可能是 3.12，ffmpeg 可能装在 /opt/homebrew/bin，也可能根本没装。直接在 prompt 里写死 `python main.py` 或 `/usr/local/bin/ffmpeg`，换台机器大概率失败。

tools.md 是给 Agent 读的“本地能力清单”：这台机器有哪些可用工具、分别怎么调、前置条件是什么、缺了怎么办。它不替代 MCP，也不替代插件，而是把 Agent 的本地执行边界和环境差异一次性说清楚。

## 问题
没有 tools.md 时，Agent 会暴露几种典型毛病：
- 假设环境一致，写死 macOS 路径，在 Linux 上直接报错；
- 遇到缺依赖就尝试 `pip install` 或 `brew install`，把环境改得面目全非；
- 为了完成目标，打印或提交包含密钥的环境变量；
- 反复试错，一条命令失败后连续瞎猜，浪费 token 也污染 shell 状态。

## 做法/步骤
1. **确定位置**：项目级放 `.openclaw/tools.md`，全局放 `~/.openclaw/tools.md`。项目级优先，全局兜底。让 Agent 在任务规划前先读这个文件。
2. **只写允许清单**：不要罗列 `man` 级别的全量命令，只写当前任务或项目真正需要的工具。未列出的命令默认不要执行。
3. **每个工具固定结构**：
   - 工具名：`ffmpeg`
   - 用途：音视频转码/提取音频
   - 检测命令：`which ffmpeg && ffmpeg -version | head -n 1`
   - 命令模板：`ffmpeg -i $INPUT -vn -acodec copy $OUTPUT`
   - 前置条件：版本 ≥ 6.0，已加入 PATH
   - 失败处理：找不到时改用 `python -m imageio_ffmpeg`；都没有则停止并报告
4. **环境差异用变量表达**：不要写 `/Users/xxx/project`，写 `$PROJECT_ROOT`；不要写 Python 3.11 具体路径，写 `$PYTHON_BIN`，并注明“优先 python3，其次 python”。tools.md 里可以有一段“环境探测结果”，由脚本自动生成。
5. **敏感信息只引用不落盘**：API key 写 `$OPENAI_API_KEY`，并加一句“不要打印、不要写入日志”。如果必须用本地密钥文件，写路径引用 + 权限要求，不写内容。
6. **写失败契约**：明确工具缺失时是 fallback 还是停下。例如：`git` 缺失时允许尝试 `$PROJECT_ROOT/scripts/ensure_git.sh`，其他情况禁止自动安装。

## 踩坑点
- **把 tools.md 写成大字典**：Agent 上下文有限，读不完就会乱翻。控制在一个屏幕内，或按任务拆成多个文件。
- **全局和项目混用**：项目 A 的工具清单出现在项目 B 的任务里。必须让 Agent 知道优先级：项目级覆盖全局，冲突时以项目级为准。
- **写死绝对路径**：这是最常见的翻车点。换机器、换用户、换容器镜像就全废。
- **漏掉前置条件**：比如写了 `python main.py`，却没说需要先 `source .venv/bin/activate`。Agent 会直接用系统 Python 跑，缺包报错。
- **给 Agent 安装权限**：除非明确要求，否则不要允许 `sudo`、`pip install`、`npm i -g`。环境变更应由人完成，Agent 只执行已声明工具。
- **明文密钥**：tools.md 被提交到 git 后，密钥就泄露了。永远用环境变量或 secret 引用。

## 可复用建议
- 用一个生成脚本检测系统环境，输出到 `.openclaw/tools.md` 的“探测结果”段：`python3 --version`, `node -v`, `which ffmpeg`, `echo $OPENAI_API_KEY`（只检测是否非空，不打印值）。
- 在 Agent 的 system prompt 里固定一句：*“执行任何本地命令前，先读取 .openclaw/tools.md；只使用其中声明的工具；未列出的命令默认不执行。”*
- 与 MCP 分工：本地 shell、文件、打包、版本控制走 tools.md；外部服务、数据库、第三方 API 走 MCP。不要在一个文件里混写。
- 每次环境升级、迁移、换机器后，重新运行检测脚本，更新 tools.md，并在 git 提交记录里说明变更。

## 总结
tools.md 不是给 Agent 看的命令大全，而是本地环境契约。它解决两件事：让 Agent 知道“这台机器能怎么干活”，同时限制它“别乱装、别乱改、别乱猜”。一旦把它版本化、模板化，并且只暴露允许清单，你会发现 Agent 在跨机器任务里的失败率明显下降，排障也从“不知道它为什么这么干”变成“看 tools.md 就知道边界在哪”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/b4bbd64170acac71.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/54996845a2c4d9bb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/1acc5ee3bb53ebca.png)

