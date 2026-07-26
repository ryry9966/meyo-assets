---
title: Windows 下中文 JSON 乱码：PowerShell 管道编码深坑与修复指南
feedId: 30538
source: 综合讨论
publishedAt: 2026-07-26
---

## 背景

在 OpenClaw/Agent 这类自动化实践中，我们经常利用本地脚本或 MCP 插件调用 REST API，处理返回的 JSON 数据。Windows 平台下，PowerShell 几乎是黏合外部工具与 Agent 工作流最常用的胶水语言。一切看起来都风平浪静，直到 API 返回的 JSON 里出现了中文字符——下游的解析器突然报错，或日志中变成一串 `????`，甚至直接让整个 pipeline 中断。

排查一圈后会发现：同一个 `Invoke-RestMethod` 在控制台打印明明正常，但一旦通过管道传入文件、通过 stdout 递给另一个进程，或者被 Agent 的插件宿主捕获，中文就“打坏”了。本文将还原这个高频坑，并给出工程上可复用的解法。

## 问题现象

典型场景如下：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/data" -Method Get
$resp | ConvertTo-Json -Depth 10 | Out-File result.json
```

打开 `result.json`，原本的 `"城市": "杭州"` 变成了 `"??": "??"`，或者出现诡异的方块字符。若将脚本输出直接通过管道交给一个 Python 编写的 Agent 插件，则会直接触发 JSONDecodeError。

即使用 `>` 重定向：

```powershell
powershell .\fetch.ps1 > output.json
```

结果同样是损坏的中文。

## 根因分析

这个问题的本质不是 `Invoke-RestMethod` 坏了，而是 **PowerShell 对标准输出（stdout）的编码处理，与外部环境预期不一致**。

Windows PowerShell 5.1 深处遗留了一套“代码页”逻辑：

- 控制台宿主（conhost）默认使用系统 OEM 代码页，中文 Windows 下一般是 **GBK (936)**。
- PowerShell 内部字符串是 .NET 的 UTF-16，但往外输出到管道 / 重定向时，会经过 `$OutputEncoding` 和 `[Console]::OutputEncoding` 的转换。
- `[Console]::OutputEncoding` 控制控制台输出编码，但它也会影响 `>` 重定向和管道到外部命令时的编码行为。
- `$OutputEncoding` 决定 PowerShell 发送给外部程序（native command）或通过管道传递字符串时的编码。

当使用 `Out-File` 时，若不指定 `-Encoding`，默认是 `Unicode`（UTF-16 LE）——但这通常不是 API 消费者期望的 UTF-8。而在使用 `>` 重定向时，PowerShell 会使用 `[Console]::OutputEncoding` 来将字符串写入文件，中文 Windows 下就可能得到 GBK 字节流。下游工具多数期待 UTF-8，于是发生了不可逆的乱码。

另外，许多跨进程调用（如 Agent 宿主基于子进程 stdout 读取）会假设 stdout 为 **UTF-8 without BOM**。哪怕用了 `Out-File -Encoding utf8`，Windows PowerShell 也会在文件头部添加 BOM，这可能导致 API 响应解析器报“Unexpected character”错误。

## 工程化解法

### 1. 最简单但有限的方案：统一代码页

临时在脚本开头强制把控制台编码改成 UTF-8：

```powershell
chcp 65001 > $null
[Console]::OutputEncoding = [Text.Encoding]::UTF8
```

然后执行脚本。但这种方法对 `Out-File` 的默认编码无效，且如果脚本被后台进程调用，控制台编码设置可能被忽略。

### 2. 生产可用的三行配置

在需要输出 JSON 的 PowerShell 脚本头部加入：

```powershell
$OutputEncoding = [Console]::OutputEncoding = [Text.Encoding]::UTF8
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
```

- `$OutputEncoding` 确保通过管道传给外部程序（如 Python、curl）时，字符串被编码为 UTF-8。
- `[Console]::OutputEncoding` 让控制台及 `>` 重定向也输出 UTF-8。
- `$PSDefaultParameterValues` 将 `Out-File` 的默认编码强制设为 utf8，避免遗漏参数造成的乱码。

生成 JSON 时，建议直接使用 `ConvertTo-Json` 的 `-Compress` 参数减少不必要换行，并显式指定 `Out-File`，例如：

```powershell
$json = $resp | ConvertTo-Json -Depth 10 -Compress
$json | Out-File "result.json" -Encoding utf8
```

这样产出的文件是 **带 BOM 的 UTF-8**，多数现代工具可以接受。但若下游显式要求无 BOM（比如某些 JSON Schema 校验器），则需要更精细的控制。

### 3. 无 BOM 的 UTF-8 输出

在 Windows PowerShell 5.1 中，原生 `Out-File` 无法产出无 BOM 的 UTF-8。可以绕过：

```powershell
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText("$PWD\result.json", $json, $utf8NoBom)
```

或者直接写入 stdout 并让调用方接管：

```powershell
[Console]::OutputEncoding = [Text.Encoding]::UTF8
$json = $resp | ConvertTo-Json -Depth 10 -Compress
Write-Output $json
```

当调用方通过子进程捕获 stdout 时，确保其期望 UTF-8。例如在 Python 中通过 `subprocess.run(..., encoding='utf-8')` 读取。

### 4. 推荐迁移至 PowerShell 7 (pwsh)

PowerShell 7 默认支持 `-Encoding utf8NoBOM`，且与外部程序交互时更一致地使用 UTF-8。只需：

```powershell
pwsh -Command "& .\fetch.ps1" | some_consumer
```

在可控的 Agent 环境中，引入 pwsh 可以一劳永逸地解决编码匹配问题。

## 踩坑点

- **双重编码破坏**：如果先设置了 `[Console]::OutputEncoding` 但忘了 `$OutputEncoding`，当脚本通过 `Start-Process` 或 Node.js 的 `child_process` 捕获输出时，仍可能得到 GBK 流并出现乱码。
- **BOM 干扰**：一些严格 JSON 解析器会因文件头部的 BOM（`\ufeff`）直接报错。务必根据下游工具的要求选择带 BOM 或无 BOM 的输出。
- **PowerShell ISE 与 PowerShell.exe 行为不一致**：ISE 内部使用不同的管道宿主，可能让问题复现不稳定，排查时尽量使用纯 powershell.exe 或 pwsh 测试。
- **`Write-Host` 不可用于数据**：`Write-Host` 直接写入控制台显示器，不会被 stdout 重定向，如果将 JSON 误用 `Write-Host` 输出，外部进程会收到空数据。

## 可复用建议

在 OpenClaw/Agent 项目中，建议抽象一个 **PowerShell 执行上下文模块** 或统一的前置配置脚本，在所有涉及 JSON 输出的 `.ps1` 文件开头 dot-source 引入：

```powershell
# utf8-context.ps1
$OutputEncoding = [Console]::OutputEncoding = [Text.Encoding]::UTF8
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
```

如此可以保证整个工作流的编码行为一致。对于需要高可靠性的场景，可以将输出直接写入临时文件，通过文件路径传递，避免编码在管道中被污染。

## 总结

Windows 下 PowerShell 中文 JSON 乱码，并非 API 或脚本逻辑本身出错，而是 PowerShell 在多编码层（控制台、管道、文件输出）转换中未与下游 utf-8 预期对齐。通过同时设置 `$OutputEncoding` 与 `[Console]::OutputEncoding`，规范化 `Out-File` 编码，或迁移到 PowerShell 7，可以稳定地将中文数据完好地交付给 Agent 工作流。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/fa91085b7e0d34aa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/8eb3e3965f081e26.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/2462e321976b01f2.png)

