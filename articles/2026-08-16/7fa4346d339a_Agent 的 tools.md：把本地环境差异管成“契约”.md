---
title: Agent 的 tools.md：把本地环境差异管成“契约”
feedId: 33352
source: 综合讨论
publishedAt: 2026-08-16
---

# Agent 的 tools.md：把本地环境差异管成“契约”

## 背景：工具调用为什么总在换机器后翻车

Agent 在本地执行工具时，最常遇到的不是模型不够聪明，而是环境差异。同一个自动化流程，在你的机器上能跑通，换到同事电脑或 CI 环境就失败，常见原因包括：

- `ffmpeg` 在 macOS 上存在，但 Linux 服务器没装；
- Windows 上只有 `python`，没有 `python3`；
- 某个 CLI 工具版本不同，参数行为发生变化；
- 认证信息存在于你的 `~/.config` 下，对方没有；
- 路径写死成 `/Users/me/bin/youtube-dl`，换机器直接失效。

这些差异如果散落在提示词、脚本、MCP 配置或代码里，排查成本很高。我们需要一个统一入口，把“工具依赖什么环境、如何验证、差异如何覆盖”说清楚。这个入口就是 `tools.md`。

## 问题：不要用提示词硬编码环境

很多人第一反应是在 system prompt 里写“如果 ffmpeg 不存在就提示用户安装”“优先使用 `/usr/local/bin/ffmpeg`”。这种方式有两个硬伤：

1. **提示词不是配置**：它无法执行检查，只能靠模型自己尝试，试错成本高；
2. **不可审计**：环境变化时，你很难快速知道哪些工具受影响。

更合理的做法是：让 `tools.md` 成为机器可读的环境契约，由脚本解析并生成可用性报告，再注入 agent 上下文。

## 做法：用 `tools.md` 声明工具环境契约

### 1. 定义工具条目结构

每个工具一个条目，至少包含：

- 工具名
- 用途
- 验证命令
- 环境变量覆盖
- 默认路径/fallback
- 跨平台差异说明
- 是否必需

示例如下：

```markdown
## ffmpeg
- purpose: 音视频处理
- check: ffmpeg -version | head -n 1
- env: FFMPEG_PATH
- fallback: /usr/local/bin/ffmpeg
- required: false
- notes: macOS 通常自带；Linux 可能需要 apt install

## python
- purpose: 执行辅助脚本
- check: python3 --version
- env: PYTHON_BIN
- fallback: python
- required: true
- notes: Windows 下可能是 py 或 python
```

### 2. 解析与检查

在 agent 启动或工具注册阶段，用一个轻量脚本读取 `tools.md`，逐条执行 `check` 命令，生成类似报告：

```text
[ok] ffmpeg 6.1.1
[missing] yt-dlp (env: YT_DLP_PATH not set)
[version mismatch] node: need >=20, got 18.17.0
```

这份报告可以作为 agent 上下文的一部分，也可以用于动态禁用不可用工具，避免模型调用必然失败的工具。

### 3. 环境变量优先级

约定统一优先级：**显式环境变量 > 默认路径 > PATH 搜索**。例如：

```bash
FFMPEG_PATH=/opt/ffmpeg/bin/ffmpeg
```

脚本先检查 `$FFMPEG_PATH`，存在则使用；否则检查 `fallback`；最后依赖 PATH。这样本地差异只通过环境变量覆盖，不需要改 `tools.md`。

### 4. 与 MCP 配置分离

`tools.md` 只管“本地环境是否满足工具运行”，MCP 配置管“如何连接外部服务”。不要把认证 token、服务器地址写进 `tools.md`，敏感信息继续用环境变量或 secret 管理。

## 踩坑点

1. **把 `tools.md` 写给人看，没有固定格式**  
   如果字段命名随意，脚本无法稳定解析。建议固定字段名，或使用 YAML front matter，例如：
   ```yaml
   ---
   ffmpeg:
     check: ffmpeg -version
     env: FFMPEG_PATH
     fallback: /usr/local/bin/ffmpeg
     required: false
   ---
   ```

2. **绝对路径进入版本库**  
   `/Users/alice/...` 这类路径会直接破坏可移植性。应优先使用环境变量和 PATH，绝对路径只作为 fallback 并且明确标注。

3. **验证命令输出不稳定**  
   有些工具版本输出格式不同，导致误判。建议验证命令只检查存在性或版本号前缀，用 `head -n 1` 截断，避免完整输出。

4. **忽略 Windows 差异**  
   `python3` 在 Windows 常见发行版可能是 `py`，命令分隔符、路径格式也不同。`tools.md` 里应写明 Windows fallback，检查脚本需要兼容 PowerShell 和 CMD。

5. **只写文档，不接入流程**  
   如果 agent 启动时不读取 `tools.md`，它只是一个静态说明文件，起不到管理作用。必须有一小段解析和检查逻辑。

## 可复用建议

- **一个工具一条目**：保持原子性，避免多个工具混在一起造成依赖关系不明。
- **区分 required/optional**：必需工具缺失时直接中止或明确提示，可选工具缺失时降级。
- **把 `tools.md` 纳入版本控制**：变更走 PR，环境变化有记录。
- **提供 `openclaw tools check` 命令**：用户可以手动运行，快速排障。
- **使用环境变量覆盖，不要编辑文件**：本地差异通过 shell profile 或 `.env` 注入，而不是改仓库文件。
- **工具条目写清 purpose**：不仅为脚本服务，也方便模型理解何时该用这个工具。

## 总结

`tools.md` 不是万能药，但它能把“工具依赖什么环境”从提示词和代码中剥离出来，变成可检查、可审计、可迁移的轻量契约。对于 OpenClaw 这类需要频繁调用本地工具的 Agent，这比反复在 prompt 里打补丁可靠得多。下次你的 Agent 在同事机器上跑不起来，先别急着调模型，看看 `tools.md` 有没有把环境差异说清楚。

---

