---
title: PowerShell 调 API 中文变问号？根本原因与根治方案
feedId: 30219
source: 综合讨论
publishedAt: 2026-07-23
---

## 背景

在 Windows 上做 OpenClaw 插件或 Agent 自动化时，常会通过 PowerShell 调用 HTTP API。例如用 `Invoke-RestMethod` 拿数据、写入 JSON 文件，再传给下游的 MCP Server。一旦 API 返回内容包含中文，很多人会发现文件里的中文变成了 `???` 或一堆乱码。即便在终端看似正常输出，一落地文件就崩。这不是偶然，而是 Windows 下默认字符编码与 PowerShell 行为叠加导致的经典坑。

## 问题根因

PowerShell 5.1（Windows 自带版本）在处理字符串输出时，默认使用系统 OEM 代码页（如 GBK 936），而不是 UTF-8。当 API 返回的 UTF-8 文本经过转换后写入文件，如果 `Out-File` 或 `>` 重定向没有显式指定编码，就会使用 `Unicode` (UTF-16LE) 或 OEM 编码，导致中文被错误转码。更深一层：`Invoke-RestMethod` 自动解析 JSON 成对象，在序列化输出时依赖控制台的 `$OutputEncoding` 变量，该变量往往被设为 ASCII 或 OEM 编码。即使对象内部字符串正确，一旦流向文件，编码就不对。

如果运行环境是 PowerShell Core（7.x），默认编码为 UTF-8 without BOM，情况好很多，但部分 Windows Server 或老旧环境仍用 5.1。所以这不是“PS坏了”，而是“PS在Window上的默认值太老”。

## 完整复现

```powershell
# 假设 API 返回 {"name":"你好世界"}
$response = Invoke-RestMethod -Uri 'https://api.example.com/hello' -Method Get
$response.name  # 控制台可能显示正常
$response | ConvertTo-Json | Out-File -FilePath 'output.json'
```
结果 `output.json` 里 `name` 变成 `? ? ?`。

如果直接使用 `Invoke-WebRequest` 取原始字节：
```powershell
$raw = Invoke-WebRequest -Uri '...' -ContentType 'application/json; charset=utf-8'
$raw.Content | Out-File 'output.txt'
```
同样遭殃，除非在 `Out-File` 加 `-Encoding utf8`。

## 工程化解决步骤

### 1. 强制所有环节使用 UTF-8

最稳妥的做法：脚本顶部设定输出编码，并显式声明读写编码。

```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false) # 无 BOM
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
[Console]::InputEncoding  = [System.Text.UTF8Encoding]::new($false)
```

这三行确保控制台交互、管道传递都是以 UTF-8 编解码。

### 2. 写入文件时明确编码

```powershell
$response | ConvertTo-Json -Depth 10 | Out-File -FilePath 'output.json' -Encoding utf8NoBOM
```
或使用 `Set-Content`：
```powershell
$response | ConvertTo-Json | Set-Content -Path 'output.json' -Encoding UTF8
```
`Out-File` 在 PowerShell 5.1 中 `-Encoding utf8` 默认带 BOM，可能影响某些 JSON 解析器。建议用 `utf8NoBOM`（PS Core 支持）或调用 .NET 方法。

### 3. 使用 .NET 方法做稳健写入

当 PS 版本不确定时，直接用 `System.IO.File` 写 UTF-8：

```powershell
$json = $response | ConvertTo-Json
[System.IO.File]::WriteAllText("$PWD\output.json", $json, [System.Text.UTF8Encoding]::new($false))
```
完全绕开 PS 的编码猜测，行为确定。

### 4. 读取文件也显式编码

如果下游用 `Get-Content` 读 JSON 再反序列化，必须同样指定编码：

```powershell
$json = Get-Content -Path 'output.json' -Raw -Encoding UTF8
$obj = $json | ConvertFrom-Json
```
忽略 `-Encoding` 就可能再次踩坑。

## 踩坑实录

- **`Console.OutputEncoding` 只影响屏幕输出**，不影响文件写入。很多人改了控制台编码就以为万事大吉，但文件依然错。
- **`Out-File` 的默认编码在 PS 5.1 是 `Unicode`（UTF-16LE）**，不是 OEM。所以文件出现的中文不是 GBK 乱码，而是被转成了 UTF-16，但用 UTF-8 打开就显示两字节变一个问号。务必去掉“记事本以 UTF-8 打开就正常”的幻想。
- **`ConvertTo-Json` 默认深度只有 2**，嵌套对象会丢失。与编码问题叠加，排查时易混淆。
- **某些 API 返回 Content-Type 不含 charset**，`Invoke-RestMethod` 可能按 `ISO-8859-1` 解析。可用 `Invoke-WebRequest` 获取原始字节，手动按 UTF-8 解码：
```powershell
$bytes = Invoke-WebRequest -Uri '...' -Method Get | Select-Object -ExpandProperty Content
$utf8 = [System.Text.Encoding]::UTF8.GetString($bytes.RawContentStream.ToArray())
```
- **PowerShell 脚本文件本身的编码**：若 `.ps1` 文件保存为 UTF-8 with BOM，PS 5.1 能正确解析中文；若为无 BOM，中文字符串可能乱码。建议脚本文件统一用 UTF-8 with BOM（Windows PowerShell）或 UTF-8（PS Core）。

## 可复用建议

1. 在插件或 Agent 初始化脚本中，**固定执行一次编码环境设定**，作为模版头部。
2. 对关键 JSON 文件读写，封装成两个工具函数：`Write-Utf8Json` 和 `Read-Utf8Json`，内部统一用 .NET `File` API，屏蔽版本差异。
3. CI/CD 流水线若在 Windows Runner 上执行，一定检查 runner 的 PS 版本。推荐使用 `pwsh` (PowerShell Core) 避免 5.1 的历史包袱。
4. 对于 MCP Server 通过子进程调用 PS 的场景，注意进程间输入/输出管道重定向编码，可在 `Start-Process` 中设定 `StandardOutputEncoding`。
5. 避免使用 `>` 重定向符，它背后也是 `Out-File`，且同样依赖编码设置。用 `Set-Content` 或 `Out-File -Encoding` 显式控制。

## 总结

Windows 中文 JSON API 调用被“打坏”，本质是微软旧编码遗产与 PowerShell 5.1 默认值不合时宜的结合。不要寄望于控制台视觉正常，那是一层假象。只要数据落盘或跨进程，必须显式声明 UTF-8 通道，并且最好直接使用 .NET 文件 API 做写入。对 Agent 自动化流程来说，早期固定编码策略比事后修补成本低得多。记住那三行 `$OutputEncoding` 设置和 `.WriteAllText` 的组合，就能从根源上避免中文变成问号。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/71c91b382143d701.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/7266dc50f8c4e2bc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/a948d016683bf54e.png)

