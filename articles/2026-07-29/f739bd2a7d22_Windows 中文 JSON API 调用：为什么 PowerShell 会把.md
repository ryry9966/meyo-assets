---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 30832
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景

在编写 MCP 工具或 OpenClaw 自动化 Agent 时，我们经常需要从 Windows 环境通过 PowerShell 脚本调用第三方 REST API，返回的 JSON 中携带中文字段，然后提取其中的数据做后续处理。一个很典型的场景：Agent 调起一份 PowerShell 脚本，脚本用 `Invoke-RestMethod` 或 `Invoke-WebRequest` 访问某内部接口，解析返回的 JSON 中的 `"姓名":"张三"`，再写入文件或传给下一个工具。很多开发者做完本地测试，推进到 Windows 的 Agent 环境后，发现中文全部变成了 `???` 或类似 `å¼ ä¸‰` 的乱码词。

这并非 PowerShell 天生“不会处理中文”，而是 Windows 上多套字符集默认值、HTTP 响应字节流与字符串转换规则相互拉扯的必然结果。下面把问题掰开，给出可直接复用的排障方法和编码规范。

## 问题现象

- 用 PowerShell 5.1 (`powershell.exe`) 调用 `Invoke-RestMethod` 请求一个返回中文 JSON 的 API，直接输出到控制台时中文显示正常，但赋值给变量再 `Write-Host` 或输出到文件，中文变成问号。
- 同一个脚本，在 PowerShell 7 (`pwsh.exe`) 下中文正常，但在 Agent 宿主只装着 PowerShell 5.1 的机器上必乱码。
- 即使 API 响应头明确声明 `Content-Type: application/json; charset=utf-8`，`Invoke-RestMethod` 解析后的对象中文字符仍损坏。
- 用 `Invoke-WebRequest` 再取 `.Content`，得到的字符串已经全部不可逆损坏，任何后续补救都无效。

根本原因并不在 JSON 解析一步，而是发生在 **HTTP 响应字节流到字符串的转换阶段**。

## 根因剖析

Windows 版的 PowerShell 5.1 使用 .NET Framework 4.x，其 `System.Net.HttpWebResponse` 在获取字符串时有一个摊还历史包袱：如果响应头没有提供 charset，或者 charset 虽然在 Content-Type 中但解析失败，它会回退到 **ISO-8859-1** 编码，而不是 UTF-8。而默认的 curl / .NET 5+（PowerShell 7 依托的 .NET Core）则遵循 WHATWG 规范，将未声明的 JSON 或文本响应假定为 UTF-8。这就是同一脚本在不同版本下表现迥异的根源。

更隐蔽的是，即使 API 返回了 `charset=utf-8`，PowerShell 5.1 有时也会因为 BOM 缺失、或响应流在中间件层的再编码而“错位”。此外，Windows 控制台的代码页（通常为 936 或 65001）和 `$OutputEncoding` 变量只影响打印到屏幕的显示，并不影响 API 调用时的字节解码方式。因此经常出现控制台看着正常、变量内已损坏的现象。

## 复现与排障步骤

### 1. 确认源头编码
用 `curl.exe`（Windows 10+ 自带）直接请求 API，将响应保存为文件，用能识别编码的编辑器（VS Code 底栏）查看是否是纯 UTF-8 无 BOM，以及头部的 `charset` 声明。

```powershell
curl.exe -s -H "Accept: application/json" https://api.example.com/data > resp.txt
```

### 2. 用 PowerShell 5.1 复现最小损坏
```powershell
$resp = Invoke-WebRequest -Uri "https://api.example.com/data"
$resp.Content   # 此时中文已乱
```

或者
```powershell
$data = Invoke-RestMethod -Uri "https://api.example.com/data"
$data.姓名   # 乱码
```

### 3. 确认二进制内容正确
通过读取 `RawContentStream` 来检查原始字节流是否完整：
```powershell
$stream = $resp.RawContentStream
$reader = [System.IO.StreamReader]::new($stream, [System.Text.Encoding]::UTF8)
$correctString = $reader.ReadToEnd()
$correctString   # 中文正常，说明只是解码方式错了
```
如果这一步也乱码，说明问题在更上游（如 API 本身返回非 UTF-8），需要从源头要求 UTF-8。

## 可靠的做法

### 方案一：强制让 PowerShell 以 UTF-8 解码请求体（推荐）

请求时保存原始字节，再手动转换为字符串，完全绕过自动解码。

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/data" -UseBasicParsing
$bytes = $response.RawContentStream.ToArray()
$utf8String = [System.Text.Encoding]::UTF8.GetString($bytes)
$obj = $utf8String | ConvertFrom-Json
```

对于 `Invoke-RestMethod`，可以先获取字节流再解析 JSON，但需要自己处理 `Invoke-WebRequest` 后再反序列化，可封装成函数：

```powershell
function Invoke-Utf8RestMethod {
    param([string]$Uri, [string]$Method = 'GET')
    $resp = Invoke-WebRequest -Uri $Uri -Method $Method -UseBasicParsing
    $raw = $resp.RawContentStream.ToArray()
    $json = [System.Text.Encoding]::UTF8.GetString($raw)
    return $json | ConvertFrom-Json
}
```

### 方案二：统一配置控制台和输出编码（辅助）

当必须使用 `Invoke-RestMethod`，且确认返回 charset 正确却仍乱码时，可以在脚本开头设置全局参数，并确保控制台代码页为 UTF-8：

```powershell
$PSDefaultParameterValues['Invoke-RestMethod:ContentType'] = 'application/json; charset=utf-8'
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

但这只能保证**输出到控制台**时不再次损坏，不能修复已损坏的字符串对象。所以它更重要的价值是当脚本外部调起（如从 Agent 进程）时，避免父管道再踩一次编码转换的坑。

### 方案三：优先使用 PowerShell 7

如果 Agent 宿主环境允许，直接将执行器指向 `pwsh.exe`（PowerShell 7）。PowerShell 7 的 `Invoke-RestMethod` 和 `Invoke-WebRequest` 默认以 UTF-8 处理响应，99% 的中文问题会自动消失。在不允许切换的遗留 Windows（如 Windows Server 2012 R2）上，再手动降级使用方案一。

## 踩坑点与经验

- **BOM 幻觉**：有些 API 返回 UTF-8 with BOM，PowerShell 5.1 解码会多出 `ï»¿` 字符，导致 JSON 解析失败。字节方案可同时规避。
- **`-UseBasicParsing` 的重要性**：在 Windows Server Core 无 IE 引擎时，不加此参数会因 DOM 解析器缺失而直接报错，和编码问题无关。
- **不要迷信控制台输出**：`Write-Host` 看到的正常可能是当前 `[Console]::OutputEncoding` 的撞大运结果，实际对象已损坏。务必转入文件或用 `-f` 格式打印来判断。
- **管道传递中的二次转换**：脚本运行在外部进程（如 Agent 调用的子进程）时，使用 `powershell -File script.ps1` 输出到 stdout 经过 `$OutputEncoding` 编码再回到父进程，如果父进程不是 UTF-8，会二次乱码。建议脚本内始终将结果写入文件，或直接输出二进制流。

## 可复用清单

- [ ] 用 `curl.exe` 样本确认 API 原始编码为 UTF-8。
- [ ] 在脚本头部设置 `$PSDefaultParameterValues` 和输出编码（增强可移植性）。
- [ ] 当运行环境为 PowerShell 5.1 时，统一使用 `Invoke-WebRequest -UseBasicParsing` + `RawContentStream` + 手动 UTF-8 解码。
- [ ] 若可升级，Agent 配置使用 `pwsh.exe` 绕过历史编码包袱。
- [ ] 结果传递时，避免依赖 stdout，优先写入临时文件并设置 UTF-8 编码输出。
- [ ] 所有与外部交互的 JSON 字符串在内存中始终保持原样，不做任何 `Out-File`、管道操作前不做隐式编码假设。

## 总结

PowerShell 在 Windows 上处理中文 JSON 的问题不是一个“Bug”，而是一系列合理但有冲突的默认值叠加的结果：.NET Framework 的 ISO-8859-1 回退、控制台代码页、输出编码变量各管一摊。对于 Agent 与自动化流程来说，最安全的策略是**永远不要信任字符串自动转换，自己掌控字节流到字符串的编码**。这会多几行代码，但再也不会在半夜因为中文乱码掉链子。遇到问题时按步骤检查字节流保真、解码编码、管道传递三层，八成都能迅速定位并修妥。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/f5c5e0056adb3b6b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/0463e04cd04af4dd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/a3d5257d873e42cb.png)

