---
title: Windows PowerShell 处理中文 JSON API 的编码陷阱与工程化解法
feedId: 30750
source: 综合讨论
publishedAt: 2026-07-28
---

## 背景

在 Windows 上开发 OpenClaw 插件或 MCP 工具时，我们经常需要用 PowerShell 调用返回中文 JSON 的 API，然后解析、落盘或转发。很多同学会遇到这样的现象：控制台里看着一切正常，哪怕 `Write-Host` 的中文也正确，但只要把响应体写入文件，或者在管道中作为字符串传递给下游，中文就变成了乱码，甚至直接损坏 JSON 结构，导致后续解析失败。

这不是 PowerShell“不擅长处理中文”，而是 Windows 上两种 PowerShell 版本的默认编码行为与 UTF-8 生态之间存在一系列不一致，而这些不一致在自动化 / Agent 管线中会集中爆发。

## 问题分析：为什么会“打坏”中文？

核心矛盾在于 **PowerShell 的默认字符编码假设与 JSON 协议的编码假设不同**。

1. **Invoke-RestMethod / Invoke-WebRequest 的返回内容**  
   这两个 cmdlet 在收到响应后会根据响应头的 `charset` 自动解码为 .NET 字符串（内存中是 UTF-16）。  
   因此 `$response.content` 在内存里字符串本身是正常的，中文并没有丢失。

2. **输出到文件时编码选择错误**  
   典型的破坏点在于：
   ```powershell
   $r = Invoke-RestMethod -Uri "https://api.example.com/data"
   $r | Out-File -FilePath "data.json"
   ```
   **问题：** Windows PowerShell 5.1 中，`Out-File` 的默认编码是 `Unicode`（UTF-16 LE），而 **不是 UTF-8**。即便你用 `Set-Content`，默认编码也可能是 `ASCII`（在 5.1 中）或 `ANSI`（遵循系统代码页）。如果后续工具链期望 UTF-8 无 BOM 的 JSON，就会出现乱码或 U+FEFF 导致 JSON 非法。

3. **重定向运算符 `>` 的编码陷阱**  
   `$r > output.json` 在 Windows PowerShell 5.1 中同样使用默认的 `Unicode` (UTF-16 LE)。这会让类 Unix 工具（如 jq、Python 标准库 `json.load`）直接扑街。

4. **控制台代码页与管道编码**  
   控制台宿主（conhost 或 Windows Terminal）会使用当前代码页（通常是 936 简体中文 GBK）渲染文本，但这种渲染只影响屏幕显示。  
   真正影响管道和文件写入的是 `$OutputEncoding` 变量，它决定了 PowerShell 与外部程序交互时使用的编码，而不是控制台显示。  
   很多人被屏幕上的正常显示欺骗，忽略了文件写出时的编码参数。

5. **PowerShell Core (7+) vs Windows PowerShell (5.1) 行为差异**  
   - PowerShell Core 中，`Out-File` 和 `Set-Content` 默认使用 `utf8NoBOM`，这是现代化的选择。  
   - Windows PowerShell 5.1 因为历史兼容，仍沿用传统默认值。  
   目前在 Windows 服务器和很多企业环境中，PowerShell 5.1 依然在役，这也是问题高发区。

## 工程化解决方案与步骤

以下方案以 Windows PowerShell 5.1 为主要目标，兼顾跨版本一致性。

### 步骤 1：强制以 UTF-8 保存 API 响应体

推荐使用显式编码参数，不依赖任何默认值：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/zh-cn/data" -Method Get
$jsonString = $response | ConvertTo-Json -Depth 10 -Compress

# 写法一：Out-File 指定 utf8（有 BOM）
$jsonString | Out-File -FilePath "data.json" -Encoding utf8

# 写法二：使用 .NET 写入器获得无 BOM UTF-8
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText("$PWD\data.json", $jsonString, $utf8NoBom)
```

### 步骤 2：统一脚本环境编码设置

在自动化脚本头部加入：

```powershell
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['*:Encoding'] = 'utf8'  # 对 Set-Content 等常见 cmdlet 生效
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::InputEncoding  = [System.Text.Encoding]::UTF8
```

这样管道到 `Out-File`、`Set-Content` 等命令时会默认使用 UTF-8 with BOM。对于不需要 BOM 的场景，仍然建议使用 `[System.IO.File]::WriteAllText`。

### 步骤 3：读取外部 JSON 文件时避免乱码

当你的 PowerShell 脚本需要读取其他工具生成的 UTF-8 文件时，务必指定编码：

```powershell
$data = Get-Content -Path "input.json" -Encoding UTF8 | ConvertFrom-Json
```

如果不加 `-Encoding UTF8`，Windows PowerShell 5.1 可能按 ANSI 代码页错误解码，造成中文字符损坏。

## 实际踩坑点与排查路径

- **症状**：`ConvertFrom-Json` 报错 “Invalid JSON primitive”，但人眼看文件中文正常。  
  **原因**：文件开头有 BOM（`U+FEFF`）。用 `Out-File -Encoding utf8` 写入的 UTF-8 带 BOM，但对于标准 JSON 解析器（如 `JSON.parse`、Python `json.load`），BOM 是非法的。  
  **解法**：改用无 BOM 写入方式，若已存在则用 `[System.IO.File]::ReadAllText` 配合 UTF8 encoding 去除 BOM，或利用 `Get-Content -Encoding UTF8` 输出字符串（它会去除 BOM）。

- **症状**：日志、通知里中文显示正常，但传给 MCP server 就变成 `????`。  
  **原因**：你可能用了 `Write-Output $response` 然后通过管道传给另一个程序，而 `$OutputEncoding` 仍是系统默认代码页，与目标程序的期望（一般是 UTF-8）不匹配。  
  **排查**：在管道前设置 `$OutputEncoding = [System.Text.Encoding]::UTF8`。

- **环境漂移**：开发机用 PowerShell 7 一切正常，部署到 Windows Server 2019 上的 PowerShell 5.1 就直接崩。  
  **建议**：提前用 `$PSVersionTable.PSVersion` 判断版本，对 5.1 单独启用严格的编码参数，或者统一使用 .NET API 做 I/O，屏蔽版本差异。

## 可复用建议

1. **内部工具库封装**：为你的 OpenClaw 项目编写一个内部函数 `Write-Utf8File`，内部调用无 BOM 的 .NET 方法，所有脚本统一使用这个函数输出文件。
2. **编码检查清单**：每次写文件、读文件、管道传值时，都明确指定 `-Encoding` 或手动设置输出编码，杜绝默认值依赖。
3. **CI 验证**：在 CI 流程中加入编码验证步骤，例如用 Python 判断文件是否以 UTF-8 编码且不含 BOM，若不满足则报错。
4. **跨脚本接口约定**：如果你的 PowerShell 脚本会被其他 Agent 或 MCP 调用，文档里必须清晰写清楚输入输出的编码假设，并给出示例代码。

## 总结

Windows 上 PowerShell 打坏中文，根源不在语言本身，而在于新旧编码默认值和 JSON 生态 UTF-8 惯例之间的错配。这个问题在自动化、Agent 管线、插件开发中特别致命，因为中间环节多，错误会放大。  
工程上解决并不复杂：强制所有 I/O 点使用显式 UTF-8 编码，写入文件时优先无 BOM，管道交互时设置 `$OutputEncoding`，并把这一习惯固化为团队规范或公共函数库。做到这几条，你的 PowerShell 脚本就能稳定地运行在中文 JSON API 的调用链路中，不再给下游解析器埋雷。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/03066ec3018e6c98.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/84554722e6ff80e6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/ba10b0752c177cfa.png)

