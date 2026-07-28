---
title: Windows PowerShell 中文 API 乱码：不是你的 API 有问题，是编码在搞鬼
feedId: 30823
source: 综合讨论
publishedAt: 2026-07-28
---

# Windows PowerShell 中文 API 乱码：不是你的 API 有问题，是编码在搞鬼

## 背景
在 Windows 上做 Agent、MCP 服务或插件开发时，我们经常需要用 PowerShell 脚本调用 HTTP API，接收或发送包含中文的 JSON 数据。一个很常见的反馈是：控制台输出的中文变问号、保存的日志文件全是乱码，甚至后端收到的请求体里中文被转成了奇怪的符号。

多数同学第一反应是检查 API 本身、网络工具或 JSON 结构，但最终定位到的元凶往往是 **PowerShell 的编码继承体系**——尤其是在 Windows PowerShell 5.1 环境下，这个体系叠加了控制台代码页、cmdlet 默认编码、重定向行为以及 JSON 序列化规则等多个环节，任何一个环节没对齐都会把中文打坏。

## 问题复现与根因拆解

### 典型表现
1. 用 `Invoke-RestMethod` 请求接口，返回的 JSON 里中文正常，但直接在控制台 `Write-Host` 或输出到文件后变成 `????` 或方块。
2. 脚本中构造中文 JSON 字符串通过 `Invoke-RestMethod` 发出，后端收到的却是 `\uXXXX` 转义序列，甚至直接是乱码。
3. 脚本自己打印中文没问题，但通过管道传给外部程序（如 `curl.exe`）后，外部程序收到的中文被破坏。
4. 把结果 `Out-File` 保存为日志，用记事本打开正常，用其他工具（如 VS Code）打开却乱码。

这些问题的根源，都可以归结为 **PowerShell 5.1 的以下三个默认行为**：
- **控制台输出编码**：`[Console]::OutputEncoding` 默认跟随系统语言区域的 OEM 代码页（中文系统为 GBK/936），而非 UTF-8。
- **文件输出编码**：`Out-File`、`Set-Content` 在 PS5.1 中默认使用 **UTF-16 LE**（Unicode），而不是 UTF-8。重定向运算符 `>` 和 `>>` 同样继承这个行为。
- **JSON 序列化转义**：`ConvertTo-Json` 在 PS5.1 中默认会将所有非 ASCII 字符（包括中文）转义为 `\uXXXX` 格式，保证 ASCII 兼容，但可读性差，且部分后端如果没正确处理转义可能导致问题。

### 为什么在 VS Code / Windows Terminal 下反而没事？
因为现代终端（尤其是 Windows Terminal）默认要求 UTF-8 输出，即使控制台编码是 GBK，终端也会在渲染层做转码或强制设定 UTF-8，所以你“感觉”正常。但当脚本运行在计划任务、系统服务或者旧的 `conhost` 窗口时，乱码立刻暴露。

在 MCP 场景中尤其危险：MCP 服务端通常通过 `stdio` 与 Host 通信，PowerShell 脚本的输出会直接进入 MCP 协议管线。如果 stdout 编码不是 UTF-8，Host 解析 JSON 时就会直接 Failed to parse。

## 彻底修复的工程化步骤

### 1. 统一进程编码（脚本开头三板斧）
在需要处理中文的 PowerShell 脚本最顶部加入以下代码块，覆盖进程级编码设置：

```powershell
# 强制控制台输出使用 UTF-8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
# 设置向外部程序发送管道数据时的编码
$OutputEncoding = [System.Text.Encoding]::UTF8
# 为当前会话设置默认文件编码（PS5.1 有效）
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

对于 **PowerShell 7+**，上述设置大部分已是默认，但仍建议显式保留 `[Console]::OutputEncoding`，防止宿主环境差异。

### 2. 调用 API 时显式指定 UTF-8 编码
使用 `Invoke-RestMethod` 时，明确通告 Content-Type：

```powershell
$headers = @{ 'Content-Type' = 'application/json; charset=utf-8' }
$body = @{ message = '用户未登录' } | ConvertTo-Json -Compress
$response = Invoke-RestMethod -Uri $apiUrl -Method Post -Headers $headers -Body $body -UseBasicParsing
```

**关键点**：
- `-UseBasicParsing` 防止 PS5.1 调用 IE 引擎解析 HTML，避免性能与编码二次伤害。
- 如果你使用 PS7，可以不加此参数（已默认使用基本解析）。

如果你的 API 返回的 JSON 里中文仍然乱码，可以绕过自动解析，手动解码：

```powershell
$raw = Invoke-WebRequest -Uri $apiUrl -UseBasicParsing
$json = [System.Text.Encoding]::UTF8.GetString($raw.RawContentStream.ToArray())
$obj = $json | ConvertFrom-Json
```

### 3. 文件读写强制使用 UTF-8 without BOM
日志文件、配置文件等应统一使用无 BOM 的 UTF-8，避免混合环境下的 BOM 灾难。

```powershell
# 不推荐：Out-File -Encoding utf8 （PS5.1 会添加 BOM）
# 推荐：
[System.IO.File]::WriteAllText('output.json', $jsonString, [System.Text.UTF8Encoding]::new($false))

# 或者使用 Add-Content 配合编码设定
$data | Out-File -FilePath 'log.txt' -Encoding utf8NoBOM -Append   # PS7 支持
# PS5.1 等旧版则需要借助 .NET 方法
```

如果必须使用 `>` 重定向，可以在脚本开头修改 `$PSDefaultParameterValues` 并配合 `Out-File` 的代理，不过更推荐直接养成使用 .NET 文件 API 的习惯。

### 4. 控制 JSON 序列化的中文转义
如果你需要保持 JSON 中中文可读（例如调试或后端要求原始中文），在 PS5.1 中会很头疼，因为 `ConvertTo-Json` 没有控制转义的参数。目前实用的几种绕行方案：

- **升级到 PowerShell 7**：使用 `ConvertTo-Json -EscapeHandling EscapeNonAscii` 可以明确关闭转义。
- **使用 Newtonsoft.Json 第三方库**：通过 `Install-Package Newtonsoft.Json` 在脚本中加载 dll，直接调用其序列化方法。
- **手动构造 JSON 字符串**：仅在简单场景下可用，不推荐复杂对象。

PS7 中正确做法：

```powershell
$body = @{ message = '用户未加载' } | ConvertTo-Json -Compress -EscapeHandling EscapeHtml
# 若要保留中文： -EscapeHandling Default（默认不转义非 ASCII，前提是输出目标编码为 UTF-8）
```

## 踩坑清单和对应解法

| 坑 | 现象 | 最稳解法 |
|----|------|----------|
| PS5.1 脚本文件自身中文乱码 | 脚本中的硬编码中文字符串变成乱码 | 将 `.ps1` 文件保存为 **UTF-8 with BOM**，否则 PS5.1 会用系统 ANSI 代码页解析脚本 |
| `Invoke-RestMethod` 返回中文乱码 | 控制台输出正确，但赋值给变量后字符串已损坏 | 检查 `$OutputEncoding` 和响应 Content-Type 头；用 `Invoke-WebRequest` 获取原始字节流手动解码 |
| `ConvertTo-Json` 把中文变成乱码 | 序列化后的 JSON 里中文变成 `\uXXXX` | 接受转义（无功能影响，但不可读），或升级至 PS7，或引入 Newtonsoft.Json |
| `Out-File` 保存的日志在其他工具打开乱码 | UTF-16 LE 文件被当 UTF-8 读取 | 统一使用 `[System.IO.File]::WriteAllText` + UTF-8 no BOM |
| MCP service 打印 JSON 导致 Host 连接失败 | stdout 输出带 BOM 或 GBK 编码的 JSON 无法解析 | 启动脚本最前设置 `[Console]::OutputEncoding = [Text.Encoding]::UTF8`，并用 `Write-Output` 替代 `Write-Host`，避免额外换行/颜色码 |

## 可复用建议与总结

1. **脚本开头标准化**：将编码三板斧封装成模块或直接复制到所有会处理中文的 `.ps1` 顶部。
2. **强制 UTF-8 everywhere**：无论输入、输出、文件、管道，都显式指定 UTF-8，不要依赖默认值。
3. **尽早迁移到 PowerShell 7**：PS7 在编码方面的默认值更符合现代开发习惯，且 JSON 转义可控，与跨平台 Agent/MCP 工具链契合度更高。
4. **测试环境一致性**：用 `chcp` 检查当前控制台代码页，用计划任务或服务账户执行一次，确保“非交互式”环境下也无乱码。
5. **使用基本解析**：`Invoke-RestMethod/Invoke-WebRequest` 添加 `-UseBasicParsing`，避免 IE 引擎干扰，也提升速度。

归根结底，这不是 API 或网络的问题，而是 Windows PowerShell 为了兼容老旧控制台程序留下的技术债。把编码控制权掌握在自己手里，才能真正让 Agent 与自动化流程稳定、正确地处理中文数据。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/c808291a210bf7f2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/7b93053dea886e01.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/97d796c55077a7fc.png)

