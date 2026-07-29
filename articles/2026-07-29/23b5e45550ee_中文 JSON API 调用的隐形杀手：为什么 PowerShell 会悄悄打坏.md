---
title: 中文 JSON API 调用的隐形杀手：为什么 PowerShell 会悄悄打坏你的 UTF-8 数据
feedId: 30928
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景：当 Agent 工具链撞上 Windows 的中文编码问题

在 OpenClaw 社区里，越来越多人在用 MCP (Model Context Protocol) 插件、Agent 工具脚本调用各种 JSON API。这些调用可能来自 Python、Node.js 写的 MCP 服务，也可能直接使用系统自带的 PowerShell 脚本来做快速集成。Windows 用户经常会发现一个诡异的现象：同样的 URL、同样的参数，用浏览器或 `curl` 能正常返回中文，用 PowerShell 的 `Invoke-RestMethod` 或 `curl.exe` 代理后处理时，中文就变成了乱码，甚至导致 JSON 解析失败。

这不是网络问题，不是 API 服务端问题，而是 PowerShell 继承了 Windows 控制台历史悠久的编码行为，在管道、重定向和变量赋值时把 UTF-8 字节流错误地当成系统默认代码页（通常是 GBK/CP936）解析了。

问题一旦发生，后续的 Agent 工作流失效：JSON 解析抛出异常，提取字段为空，Agent 认为 API 调用失败而进入错误恢复或重试循环。对于依赖 MCP 工具来获取实时数据的应用，这是致命故障。

## 问题根源：两个编码层级的冲突

现代 Web API 几乎都返回 `Content-Type: application/json; charset=utf-8`，JSON 标准本身就是 UTF-8。但当你用 PowerShell 发起一次 API 调用并捕获输出时，会经过以下环节：

1. **HTTP 响应**：服务器返回 UTF-8 字节流，这是正确的二进制数据。
2. **PowerShell 的 Web cmdlet 内部编码处理**：`Invoke-RestMethod` 和 `Invoke-WebRequest` 根据响应的 charset 或自身的默认编码将字节流转成 .NET 字符串。在较新的 PowerShell (6+) 中，默认 UTF-8 支持较好；但在 Windows PowerShell 5.1 中，处理可能不可靠，尤其当响应头缺失 charset 时可能回退到 ISO-8859-1。
3. **输出到控制台或变量**：PowerShell 把 .NET 字符串输出到控制台或赋值给变量时，如果有重定向或 `Out-File`，会使用当前控制台的编码。对于 Windows PowerShell 5.1 默认控制台，输出编码通常跟随系统区域设置（中文 Windows 为 GBK）。
4. **管道与外部程序交互**：如果脚本调用外部程序（如 `curl.exe`），PowerShell 在捕获标准输出时，会将其视为基于 `[Console]::OutputEncoding` 的字节流，并转换为字符串。默认为系统 OEM 代码页（GBK），导致 UTF-8 输出的 `curl` 结果被错误解码成乱码。

关键点：即使 API 返回的是完美 UTF-8，当 PowerShell 把字符串向文件写入或跨进程传递时，不显式指定 UTF-8 编码就随时可能被转换为 GBK，从而“打坏”中文。

## 实战重现：一个典型的 MCP 工具脚本

假设你为某个 Agent 写了一个 MCP 工具，功能是查询天气 API。你用 PowerShell 编写 `Get-Weather.ps1`：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/weather?city=北京" -Method Get
$city = $response.city
$temp = $response.temperature
Write-Output $city
```

在 PowerShell 窗口直接运行，输出可能正常，因为控制台渲染时还能正确显示 Unicode。但如果你将输出重定向到文件或由父进程（例如 Python 子进程）捕获：

```powershell
.\Get-Weather.ps1 > result.txt
```

打开 `result.txt` 可能发现“北京”变成了“鍖椾含”。同样，当 Node.js 的 MCP 管理进程使用 `child_process` 执行该脚本并读取 stdout 时，接收到的文本可能就是乱码，导致 JSON 解析失败或 Agent 得到错误信息。

类似问题也出现在使用 `curl.exe` 时：

```powershell
$result = & curl.exe -s "https://api.example.com/weather?city=北京"
$json = $result | ConvertFrom-Json   # 可能报错：Invalid JSON primitive
```

因为 `curl` 输出的是 UTF-8 字节，而 PowerShell 用 `[Console]::OutputEncoding` (GBK) 去解码，破坏二进制结构。

## 解决方案与工程化做法

### 1. 使用 PowerShell 7+ 并设置输出编码为 UTF-8

PowerShell Core (7+) 默认输出 UTF-8，这是最简单的升级。迁移后，配置 `$OutputEncoding`：

```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

如果必须在 Windows PowerShell 5.1 运行，可显式设置：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

但注意这对 cmdlet 的 `-Encoding` 参数不起全局作用，仍需在每个写文件操作中指定 `-Encoding UTF8`。

### 2. 确保 Invoke-RestMethod 使用 UTF-8 解码

有时 API 响应头不含 charset，在 Windows PowerShell 5.1 中可能被误认为 ISO-8859-1。可以从原始字节手动解码：

```powershell
$resp = Invoke-WebRequest -Uri $uri -Method Get
$encoding = [System.Text.Encoding]::UTF8   # 假设 UTF-8
$string = $encoding.GetString($resp.Content)
$json = $string | ConvertFrom-Json
```

或者设置 `$ProgressPreference = 'SilentlyContinue'` 避免进度条干扰，然后捕获 RawContentStream 处理。

### 3. 外部程序调用时强制使用 UTF-8 契约

在调用外部 curl 或 python 前设置控制台编码：

```powershell
$env:PYTHONIOENCODING = 'utf-8'  # 针对 Python 子进程
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$output = & curl.exe -s $uri
```

甚至可以在每次调用前临时切换编码，调用后恢复。注意 PowerShell 7 中，`&` 操作符解码外部输出时会参考 `[Console]::OutputEncoding`，修改该值立即影响后续解码。

### 4. 文件读写统一使用 UTF-8 No BOM

所有脚本内部生成的临时 JSON 文件、配置文件，写入时必须加 `-Encoding UTF8`。在 PowerShell 5.1 中 `-Encoding UTF8` 会生成带 BOM 的 UTF-8 文件，如果子进程不接受 BOM，需改用 `New-Item -Value ... -Encoding utf8NoBOM`（但 5.1 没有，需要 .NET 方法）。建议统一用 `[System.IO.File]::WriteAllText($path, $text, [System.Text.UTF8Encoding]::new($false))`，直接控制无 BOM UTF-8。

### 5. 在 MCP 工具层做防御性编码检查

Agent 侧（如 Python 调用的 MCP 服务器）捕获工具输出后，先对字符串尝试用 `utf-8` 和 `gbk` 重新编码/解码进行“复活”尝试。但这只是临时方案，根治仍需 Tool 侧保证 UTF-8 输出。

## 踩坑清单

- **PowerShell 5.1 的 `>>` 重定向默认用 UTF-16 LE**：如果用 `>>` 追加日志，文件将以 UTF-16 小端序编码，Windows 文本编辑器可能能打开，但给 Linux 侧的工具会直接报错。永远用 `Out-File -Append -Encoding UTF8` 代替 `>>`。
- **`ConvertTo-Json` 转中文成 Unicode 转义序列**：默认情况下 `ConvertTo-Json` 会把中文转成 `\uXXXX`，这不影响 JSON 合法性，但可读性差。设置 `-Compress` 后仍然如此。如果必须保留原始中文，使用 `ConvertTo-Json -Depth 10 | ForEach-Object { [System.Text.RegularExpressions.Regex]::Unescape($_) }` 来还原，但可能引入危险字符，慎用。
- **控制台字体和渲染不是编码问题**：有时候中文显示为问号，以为是编码，实际是控制台字体不支持中文字形。确保 PowerShell 使用 TrueType 中文字体（如“新宋体”或“Consolas+Microsoft YaHei”组合）。

## 可复用建议

1. **在团队代码库中创建一个 `encoding_init.ps1` 模块**，在每个工具脚本开头引入，统一设置：
   ```powershell
   [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
   $OutputEncoding = [System.Text.Encoding]::UTF8
   $PSDefaultParameterValues['*:Encoding'] = 'utf8'
   chcp 65001 > $null   # 仅对 cmd 子进程有效
   ```
2. **所有与 MCP/Agent 交互的 PowerShell 工具返回 JSON 时，通过 stdout 管道直接输出对象**，让父进程用 `ConvertTo-Json -Compress` 显式序列化，避免隐含编码。

3. **能不用 PowerShell 包装外部命令就尽量别用**。如果实在需要调用 curl，考虑直接用 PowerShell 原生的 `Invoke-RestMethod`，它对于 UTF-8 有更好的支持路径。

4. **建立端到端测试**，用 Python/Node 子进程调用工具脚本，传入中文参数、接收中文响应，断言返回的 JSON 中文字段 `"北京"` 不是乱码。一旦编码配置变更能被及时验证。

## 总结

Windows 中文 JSON API 调用中，PowerShell 的编码陷阱出现在输入输出边界——从 API 字节流到 .NET 字符串，再到进程间通信或文件落盘。核心问题是历史原因导致默认编码不是 UTF-8，以及新旧版本行为不一致。在 OpenClaw 这类 Agent 工具链中，自动化脚本的输出被 Agent 消费，编码问题会导致整个调用链中断。遵循“显式声明 UTF-8、使用 PowerShell 7、统一文件读写编码”三个原则，可以扫清 99% 的乱码障碍。下次你的 MCP 工具返回神秘问号时，不要先去检查 API，先检查脚本的 `[Console]::OutputEncoding` 这个隐形开关。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/6a9a0e0605adeb0f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/0e2b1baf414bfc70.png)

