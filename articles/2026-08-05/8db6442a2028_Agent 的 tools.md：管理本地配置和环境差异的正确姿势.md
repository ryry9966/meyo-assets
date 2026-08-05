---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 31767
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景：Agent 工具链的“环境混沌”

当你给 Agent 写完一个本地文件搜索工具、一个调用 `ffmpeg` 的音视频处理器，或者一个依赖 `pandoc` 的转换脚本，兴冲冲推到同事机器上，十有八九会碰到：

- `ffmpeg: command not found`
- `ModuleNotFoundError: No module named 'magic'`
- 路径用 `~` 展开在不同 shell 下行为不一致
- macOS 上 `grep` 是 BSD 版，缺少 GNU 的某个参数
- 环境变量有，但名字拼写差一个下划线

面对这些本地配置和环境差异，常见的补救手段是“在 `README` 里加一句提醒”，或者用 `if/else` 兜底，再不然靠团队成员凭经验修修补补。随着工具数量增长，这种隐式知识很快变成熵增的来源。Agent 变成了只能在自己机器上跑顺的半成品。

**问题核心**：Agent 的本地工具普遍缺少一份 **可被机器读取、版本管理、唯一事实来源的环境声明**。

## 引入 `tools.md`：一份声明式工具环境定义

在 OpenClaw 生态里，我们给每个自定义工具（或插件）同级目录下放一个 `tools.md`，它不是给人看的文档，而是一份 **结构化的元信息文件**，让 Agent 启动时就知道这个工具需要什么，本机少了什么，如何适配。

典型的 `tools.md` 长这样：

```markdown
---
id: local-video-transcode
name: video_transcode
description: Transcode video using ffmpeg
runtime:
  os: [darwin, linux]
  dependencies:
    ffmpeg:
      min_version: "4.4"
      detection:
        - type: which
          command: ffmpeg
        - type: path
          path: "/usr/local/bin/ffmpeg"
        - type: fallback
          docker: "jrottenberg/ffmpeg:4.4-ubuntu"
env:
  OUTPUT_DIR:
    description: Directory for transcoded files
    default: ./output
  FFMPEG_THREADS:
    description: Number of CPU threads
    default: "auto"
config:
  timeout: 300
  strict_version: false
```

核心字段：

- `runtime`：明确操作系统、二进制依赖、最低版本、检测策略、回退方案。
- `env`：声明会用到的环境变量，有默认值，作为工具的自文档。
- `config`：工具行为参数，可被全局配置覆盖。

这份文件随代码一起进入版本控制，成为工具与 Agent 之间的“契约”。

## 从声明到生效：Agent 启动时的处理流程

Agent 的 loader 模块在扫描工具目录时，会解析 `tools.md`，形成 `ToolManifest` 对象，关键步骤：

1. **环境探测**  
   按 `detection` 列表顺序尝试，直到找到可用二进制。对 `which` 类型，执行 `shutil.which("ffmpeg")`；对 `path` 类型直接判断 `os.access`；都失败则尝试 fallback（如拉起 Docker 容器、使用 pip 包、报错）。

2. **环境变量注入**  
   从 `os.environ` 中读取对应变量，若不存在则使用 `default`。所有值注入工具进程或 worker 上下文。**敏感变量（如 API key）只声明名字，不写默认值**，实际值从外部的 `.env` 或 Secret Manager 注入。

3. **版本校验（可关闭）**  
   获取 `ffmpeg -version` 并解析版本号，判断是否满足 `min_version`。若未满足且 `strict_version` 为 false，给出警告但不崩溃；为 true 则拒绝加载。

4. **兜底策略执行**  
   fallback 为 Docker 时，Agent 自动生成对应的 `docker run` 命令，挂载必要目录，工具函数仍然以统一接口调用，对上层透明。

这样一来，工具的“运行环境差异”被集中在一份声明里处理，而不是散落在代码的 `try/except` 或注释中。

## 踩坑记录

### 1. 检测策略顺序引发的误判

如果先写 `path` 再写 `which`，而路径 `/usr/local/bin/ffmpeg` 存在但损坏，Agent 会认为依赖已满足，实际调用时崩溃。正确顺序：**优先使用 PATH 探测，再 fallback 到固定路径**，并对路径做可执行性检查。

### 2. 环境变量泄露与覆盖混乱

某个工具在 `tools.md` 里写了 `default: "my_prod_secret"`，差点把密钥暴露进仓库。**原则：tools.md 里绝不出现真实密钥，只定义变量名和用途**。另外，当项目级 `.env` 和系统环境变量同时存在时，加载顺序要文档化，避免覆盖导致调试困难。

### 3. Windows 兼容性被忽略

`tools.md` 中 `detection` 仅写了 Unix 路径，Windows 下 Agent 直接报错。改进：增加 `os` 条件分支，为 Windows 准备独立的 `detection` 列表，比如用 `where ffmpeg` 或 Chocolatey 安装路径。

### 4. tools.md 更新滞后

需求变了（比如改用 `ffmpeg 5.x`），代码改了，但 `tools.md` 还写着 4.4。同事的 CI 环境因此提示版本不匹配。**把 tools.md 的更新纳入 PR checklist**，或在 CI 里加一步读取 tools.md 并校验实际环境。

## 可复用建议

- **模板化**：为常用依赖（Python、Node、Docker 工具）制作 `tools.md` 样板，避免每次都从头写。
- **集成 MCP 工具描述**：如果你在用 MCP 协议暴露工具给 Agent，可以在 Server 端复用同一份 `tools.md` 中的 `env` 和依赖信息，减少重复。
- **CI 早发现**：在 CI 中运行 `agent toolkit check`（自定义脚本）扫描所有工具，对每份 `tools.md` 用当前 Docker 镜像做环境验证，不通过则阻断合并。
- **生成文档**：写个小脚本从 `tools.md` 生成面向用户的安装指南，保证文档始终与配置同步。
- **运行时自省**：Agent 可以暴露一个 `/tools/local` 端点，返回当前各工具的可用状态、版本、缺失依赖，加速远程协作排障。

## 总结

`tools.md` 不是一个银弹，但它在 Agent 的本地工具管理中扎下了一个锚点：**把“在我机器上能跑”变成一份可执行的环境契约**。它降低了因为环境差异产生的调试成本，也让工具开发者能更专注于功能本身，而不是无穷无尽的跨平台适配。

在一个由众多松散组件构成的 Agent 系统中，显式地管理环境差异，远比事后补丁更高效。

---

