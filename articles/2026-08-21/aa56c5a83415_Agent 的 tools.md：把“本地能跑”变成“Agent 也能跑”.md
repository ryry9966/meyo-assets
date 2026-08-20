---
title: Agent 的 tools.md：把“本地能跑”变成“Agent 也能跑”
feedId: 33956
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

很多 Agent 工具在本地手动执行没问题，但交给 OpenClaw、MCP 或插件调用后，经常因为 Python 版本、路径写法、环境变量缺失、平台差异而失败。问题通常不在工具逻辑本身，而在于 Agent 看到的工具说明里，只写了“能做什么”，没写“在什么环境里怎么跑”。

`tools.md` 的价值就在这里：它不是功能说明书，而是给 Agent 看的**环境契约**。把本地配置和差异显式写出来，Agent 才能少做无意义的试错。

## 常见问题

- Agent 默认按 Linux 习惯生成命令，在 Windows 上写反斜杠、不处理空格路径
- MCP server 进程里的环境变量，和你的本地 shell 环境变量不是一套
- 工具依赖的二进制路径写死在文档里，换一台机器就失效
- `tools.md` 里混入真实 API key 或本地绝对路径，既危险又不可迁移
- 没有验证命令，Agent 只能直接跑，失败后再猜原因

这些问题的共同点是：**环境信息是隐性的，而 Agent 只能根据显性文本行动**。

## 做法：给 tools.md 加一层环境说明

不建议推翻现有 tools.md，而是在每个工具或技能段落里补充固定字段。一个最小可用的结构如下：

````markdown
## tool: compress_image

- **功能**：压缩图片，输出到指定目录
- **运行时**：Python 3.11+
- **依赖**：Pillow >= 10.0
- **环境变量**：
  - `IMAGE_MAGICK_BIN`：可选，默认使用 `magick`
  - `OUTPUT_DIR`：必须由调用方提供绝对路径
- **路径规则**：
  - 输入/输出目录使用绝对路径
  - 路径包含空格时必须加引号
  - 优先使用正斜杠 `/`，避免反斜杠转义
- **平台差异**：
  - macOS/Linux：命令为 `magick` 或 `convert`
  - Windows：命令为 `magick.exe`，路径形如 `C:/tools/magick.exe`
- **验证命令**：
  ```bash
  python -c "import PIL; print(PIL.__version__)"
  ```
- **失败处理**：
  - 不要擅自安装系统包
  - 先报告缺失项和当前环境，再等待用户确认
````

这个字段组合不复杂，但能覆盖大多数本地环境差异。关键是：**验证命令必须幂等、只读、无副作用**。例如 `ffmpeg -version`、`python -c "import xxx"` 是安全的；不要写会下载、删除、修改配置的命令。

## 踩坑点

1. **硬编码路径**  
   不要写 `/home/me/project` 或 `C:\Users\张三\data`。使用 `$WORKSPACE`、`~` 或 `$HOME` 占位符，并说明由谁展开。如果必须写本机路径，放到 `tools.local.md` 并加入 `.gitignore`。

2. **环境变量不透明**  
   MCP server 启动时注入的 `env`，和你在终端里 `export` 的变量不是一回事。`tools.md` 里要明确区分：哪些变量由 MCP 配置提供，哪些需要 Agent 从用户 shell 环境读取。

3. **Windows 路径问题**  
   空格、中文用户名、反斜杠转义是最常见的三类问题。文档里直接给一个 Windows 示例，比如 `"C:/Program Files/ImageMagick/magick.exe"`，比写“注意路径”更有效。

4. **验证命令带副作用**  
   不要写 `pip install -r requirements.txt` 作为验证。验证是确认环境是否满足，不是修复环境。修复动作应该由用户决定。

5. **敏感信息泄漏**  
   API key、token 只写占位符，例如 `$OPENAI_API_KEY`，并注明从环境变量读取。不要把真实密钥写进 tools.md，尤其是会被 Agent 读取并可能被记录日志的文件。

## 可复用建议

- **分层管理**：  
  `tools.md` 放通用契约，`tools.local.md` 放本机路径和差异，后者加入 `.gitignore`。这样团队共享时不会互相覆盖本地配置。

- **给每个工具加 verify**：  
  让 Agent 先跑验证命令，通过后再执行实际任务。这比直接失败后重试更高效。

- **在 MCP 配置里统一注入环境变量**：  
  例如在 MCP server 配置里写 `env` 字段，而不是在 prompt 里拼接密钥。tools.md 只引用变量名。

- **给 Agent 系统提示加一条规则**：  
  > 工具依赖信息以 `tools.md` 为准；缺失时停止并报告，不要猜测路径或版本。

- **记录版本输出**：  
  验证命令的输出版本号，能帮助快速定位“本地能跑、Agent 不能跑”的差异。例如 `python --version` 和 `which python` 的结果。

## 总结

`tools.md` 管理本地配置和环境差异的核心，不是写更多说明，而是把隐性前提显式化。一个合格的 tools.md 至少应该让 Agent 知道三件事：**当前工具需要什么环境、如何验证环境是否满足、不满足时应该做什么**。做到这三点，Agent 在真实机器上的执行成功率会明显提升，调试成本也会下降。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/cac23fc379729608.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/b47c9bd7978f4aed.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/b62872f8c45a9de9.png)

