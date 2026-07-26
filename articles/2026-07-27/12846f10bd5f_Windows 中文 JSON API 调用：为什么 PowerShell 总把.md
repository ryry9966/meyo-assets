---
title: Windows 中文 JSON API 调用：为什么 PowerShell 总把中文打坏？一处绕不开的编码陷阱
feedId: 30604
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景

在构建面向 OpenClaw、Agent 或 MCP 插件的自动化流程时，不少开发者会直接在 Windows 上使用 PowerShell 调用外部 HTTP API，然后解析返回的 JSON。一旦返回值包含中文，往往会出现令人头疼的现象：控制台输出变成问号、写入文件后变成不可读的字符，或者看到的是一串 `\uXXXX` 转义序列。更棘手的是，同样的脚本在本地测试“好像正常”，放到自动化调度里就乱码。

这一问题的本质，不完全是某个命令的 bug，而是 PowerShell 在 Windows 控制台、管道、重定向和 .NET 类型互操作中，对“字符串的编码管理”存在多层隐式转换。工程上想彻底避开，需要理解这些层次，并固化为可复用的编码约定。

## 问题：中文 JSON 为什么被打坏？

典型的暴雷场景如下：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/data" -Method Get
Write-Host $response.name   # 输出乱码
$response | ConvertTo-Json | Out-File result.json  # 文件内中文损坏
```

表面看是中文损坏，实际上至少有三个层面的编码在同时作用：

1. **控制台输出编码** – Windows 控制台（特别是简体中文系统）默认代码页为 936 (GBK)，而 API 返回的 JSON 通常是 UTF-8。`Write-Host` 依赖 `[Console]::OutputEncoding` 将 .NET 字符串（UTF-16 内部）转换到控制台代码页，如果该编码不是 UTF-8，中文字符就会在转换中丢失。
2. **HTTP 响应处理编码** – `Invoke-RestMethod` 会自动反序列化 JSON，但前提是响应头中的 `Content-Type` 包含正确的 `charset=utf-8`。如果服务端未明确声明，或者响应经过 gzip 压缩导致编码探测失败，PowerShell 可能回退到默认编码，从而引入乱码。
3. **文件写入编码** – `Out-File` 和重定向操作符 `>` 在 PowerShell 5.1 中默认使用 **UTF-16 LE**，而多数外部工具期望 UTF-8。一旦不加 `-Encoding` 参数，其他程序再读取该文件时就会出现乱码。

一个容易被忽视的子问题：有些 API 会将中文转义成 `\uXXXX`，`Invoke-RestMethod` 自动将其解码为正确的 .NET 字符串对象。但如果开发者随后使用 `ConvertTo-Json` 再序列化，且未指定 `-Compress` 或 `-EscapeHandling`，该字符串会再次被转义，导致文件里看到的仍是转义码而非文字。

## 做法/步骤：如何稳定处理中文 JSON

下面是一套在 Windows 下可复用的处理流程，能在交互式窗口和自动化脚本中保持一致。

### 1. 设置控制台与输出编码

在执行任何外部调用前，显式设置：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
```

第一行保证 `Write-Host` 等命令能正确将 .NET 字符串输出到终端。第二行让脚本中所有 `Out-File` 默认使用 UTF-8，避免逐条指定。对于 PowerShell 7+，控制台已默认使用 UTF-8，但仍建议显式设置以保证脚本向下兼容。

### 2. 安全地获取并解码 API 响应

推荐使用 `Invoke-WebRequest` 获取原始字节，手动解码，再转换自 JSON，这样可以完全绕过自动编码探测：

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/data" -Method Get
$rawBytes = $response.Content -as [byte[]]
$utf8String = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$data = $utf8String | ConvertFrom-Json
```

如果确认服务端总是返回正确的 `Content-Type: application/json; charset=utf-8`，且不需要处理压缩，也可以直接使用 `Invoke-RestMethod`，但要加一层保护——将结果转为 JSON 字符串后立即以 UTF-8 重新读取，确保编码无偏差：

```powershell
$data = Invoke-RestMethod -Uri "..." -Method Get
$jsonString = $data | ConvertTo-Json -Compress -Depth 10
$jsonString | Out-File -Encoding utf8 "result.json"
```

这里的 `-Compress` 会移除不必要的空白，并保持非 ASCII 字符以直接字符形式输出（PowerShell 7 行为更稳定）。在 PowerShell 5.1 中，可能需要额外指定 `-EscapeHandling EscapeNonAscii` 的逆操作，但那是另一话题，通常直接使用 UTF-8 文件写入即可。

### 3. 将结果传递给下游自动化

如果要将 JSON 字符串传递给其他程序（如 curl 再发送、写入管道），必须确保管道传输的是字节流而非 .NET 字符串。例如通过 `[System.IO.File]::WriteAllText("result.json", $jsonString, [System.Text.Encoding]::UTF8)` 直接写出，绝对不要使用 `>` 重定向，除非显式开启了 `$PSDefaultParameterValues`。

## 踩坑点：三个被反复踩过的坑

- **ISE 环境与脚本环境表现不同**  
  PowerShell ISE 的交互窗口对输出编码处理更宽松，部分乱码在 ISE 中不出现，但一旦保存为 `.ps1` 由 `powershell.exe -File` 执行，就会暴露。排查时务必在脚本执行环境下验证。

- **API 返回 Gzip 压缩时 Content 类型被篡改**  
  `Invoke-WebRequest` 在自动解压后，`Content` 属性可能是字符串，但 `RawContent` 仍包含原始压缩字节。如果手动解码时误用 `RawContent`，会读取到压缩数据而非真实 JSON。安全做法是使用 `$response.Content` 并再次转换为字节数组进行 UTF-8 解码。

- **混合使用不同版本的 PowerShell**  
  如果自动化流程中同时调用 `powershell.exe`（版本 5.1）和 `pwsh.exe`（版本 7+），编码默认值会突变。建议在脚本开头检查 `$PSVersionTable.PSVersion`，并据此条件化编码设置，或者统一推广到 PowerShell 7 运行自动化任务。

## 可复用建议

把这些设置封装成脚本模板，所有涉及中文 API 交互的自动化脚本统一引用：

```powershell
# encoding-bootstrap.ps1
[CmdletBinding()]
param()
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['*:Encoding'] = 'utf8'   # 影响部分命令的 -Encoding 默认值
function Get-ApiJson {
    param($Uri)
    $resp = Invoke-WebRequest -Uri $Uri -Method Get
    $bytes = $resp.Content -as [byte[]]
    $jsonStr = [System.Text.Encoding]::UTF8.GetString($bytes)
    return $jsonStr | ConvertFrom-Json
}
```

在真正的业务脚本开头 `. .\encoding-bootstrap.ps1` 即可。对于需要更高性能的场景，也可以直接调用 `curl.exe`（Windows 10+ 自带），它使用 UTF-8 处理管道，彻底绕开 PowerShell 的编码魔数。

## 总结

PowerShell 处理中文 JSON 出现乱码，根因在于 Windows 控制台与 .NET 字符串之间存在多重隐式编码转换，而默认值往往不与 API 的 UTF-8 对齐。只要在脚本中显式锁定三条防线——控制台输出编码、HTTP 字节解码方式、文件写入编码——就能稳定地让中文 JSON 进出 PowerShell，不再“被打坏”。这一处置方法对任何依赖 PowerShell 做 HTTP 粘合层的 OpenClaw / Agent / 自动化工程都适用，成本极低，却能消除一类让人浪费半天排查时间的隐蔽 bug。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/2618f2e0750b626e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/d88b742963588a33.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/3deffb342d463a19.png)

