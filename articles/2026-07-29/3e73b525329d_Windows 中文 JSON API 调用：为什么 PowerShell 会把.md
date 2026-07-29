---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏，以及如何让 Agent 稳定对接
feedId: 30935
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景

在 Windows 上跑 OpenClaw 这类 Agent 流程时，经常会用本地脚本桥接外部 API——例如 PowerShell 调用某个 HTTP 接口，拿到 JSON 后传给下游的 MCP 工具或插件。一个反复出现的故障是：中文内容从 API 返回到 PowerShell 变量里还是正常的，可一旦 `ConvertTo-Json` 再输出，或者通过管道交给文件/其他进程，中文字符就可能变成乱码、Unicode 转义序列（`\uXXXX`），或者被莫名其妙地截断。

本文将还原这一问题在工程中的表现，给出原因和两种可落地的解法，并附带一套在 OpenClaw/MCP 场景下可复用的编码安全实践。

## 问题表现

典型场景：Agent 内的 PowerShell 脚本通过 `Invoke-RestMethod` 调用某 API，返回一个包含中文字段的对象。

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/items" -Method Get
$response.name  # 输出正常：张三
```

但当需要把整个对象序列化为 JSON 传给下一个工具时：

```powershell
$json = $response | ConvertTo-Json -Compress
# 或
$json = $response | ConvertTo-Json -Depth 3
Write-Output $json
```

得到的 JSON 里，`name` 变成了 `"\u5f20\u4e09"`，或者在日志文件里出现 `??`、`锟斤拷`。更有甚者，下游的 MCP server 因为解析到无效的 UTF-8 字节直接挂掉。

## 根因分析

问题出在 PowerShell 的“编码管道”约定与 Windows 系统默认代码页的冲突。

1. **`ConvertTo-Json` 默认转义非 ASCII 字符**  
   PowerShell 5.1 及更早版本的 `ConvertTo-Json` 默认会将所有非 ASCII 字符转义为 `\uXXXX` 形式。这是为了兼容古老的 JSON 解析器，但现代 API 和工具普遍期望直接使用 UTF-8 编码的原始字符。

2. **输出重定向与 Windows 代码页**  
   在 Windows 控制台（cmd、powershell.exe 的旧终端）中，输出默认使用系统活动代码页（如简体中文 GBK/936）。当脚本的输出被重定向到文件或被另一个进程通过标准输出捕获时，PowerShell 会将 Unicode 内容转换成该代码页，导致中文字符在非 Unicode 环境下损坏。即便终端是 Windows Terminal 使用 UTF-8，管道行为仍可能受 `[Console]::OutputEncoding` 影响。

3. **`Invoke-RestMethod` 与响应字符集的隐藏假设**  
   尽管 `Invoke-RestMethod` 能正确解析 UTF-8 的 JSON 响应，但一旦将结果交给字符串操作或文件写入，如果未显式指定编码，就会回退到默认的 `File` 编码模式——在 Windows PowerShell 中往往是 UTF-8 with BOM 或 ANSI，而不是纯 UTF-8，进一步埋雷。

这些问题叠加在一起，就表现为“明明单步调试时中文没问题，一放进链式脚本就坏掉”，尤其常见于用 PowerShell 作为 Agent 工具执行器的场景。

## 做法 / 步骤

下面给出两种工程解法，可根据环境选择。

### 方案一：强制使用 UTF-8 输出 + 禁用 JSON 转义（PowerShell 7 推荐）

在 PowerShell 7（pwsh.exe）中，`ConvertTo-Json` 新增了 `-EncodeAsString` 参数（实际参数名是 `-EscapeHandling`），可以避免转义中文，同时可通过全局变量控制输出编码。

```powershell
# 要求 PowerShell 7+
$response = Invoke-RestMethod -Uri "https://api.example.com/items" -Method Get

# 序列化，保留原始中文
$json = $response | ConvertTo-Json -Depth 10 -EscapeHandling EscapeNonAscii -Compress
# 错误，正确参数为： -EscapeHandling 只有 EscapeHtml 和 Default，没有 EscapeNonAscii。实际上在 PS 7 中需要：
# $json = $response | ConvertTo-Json -Depth 10 -Compress
# 然后通过设置 $OutputEncoding 或使用 [System.Text.Encoding]::UTF8 输出。
```

等等，这里需要纠正。PowerShell 7 中 `ConvertTo-Json` 默认**不再**将非 ASCII 字符转义为 `\uXXXX`，而是直接保留原始字符。因此，只需确保输出环境为 UTF-8：

```powershell
# PowerShell 7
$response = Invoke-RestMethod -Uri "https://api.example.com/items" -Method Get
$json = $response | ConvertTo-Json -Depth 10 -Compress

# 强制标准输出使用 UTF-8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
Write-Output $json
```

如果在 Agent 中通过标准输出捕获结果，还需要在启动 PowerShell 时设置编码：

```powershell
# 启动命令
pwsh -NoProfile -Command "[Console]::OutputEncoding=[Text.Encoding]::UTF8; .\script.ps1"
```

或者直接写入文件并显式指定编码：

```powershell
$json | Out-File -FilePath "result.json" -Encoding utf8NoBOM
# 如果仍是 Windows PowerShell 5.1，则需要：
$json | Out-File -FilePath "result.json" -Encoding UTF8
# 注意：Windows PowerShell 的 -Encoding UTF8 带 BOM，可能影响下游解析，可在后续用 .NET 去除 BOM 或使用 StreamWriter。
```

### 方案二：Windows PowerShell 5.1 兼容方案

很多企业内部环境仍使用 Windows PowerShell 5.1，`ConvertTo-Json` 会转义中文。可以通过 .NET 的 `JsonSerializer` 绕过，或者用 `Newtonsoft.Json`（如果预装）。

使用 .NET 的 `System.Text.Json`：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/items" -Method Get
$utf8 = [System.Text.Encoding]::UTF8
$json = [System.Text.Json.JsonSerializer]::Serialize($response, [System.Text.Json.JsonSerializerOptions]@{
    Encoder = [System.Text.Encodings.Web.JavaScriptEncoder]::UnsafeRelaxedJsonEscaping
    WriteIndented = $false
})
# 输出到文件，保证无 BOM
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText("result.json", $json, $utf8NoBom)
```

`UnsafeRelaxedJsonEscaping` 会阻止对中文等多字节符的转义，输出人类可读的 JSON。如果环境不支持 `System.Text.Json`（.NET Framework 4.x 默认无），可用 `Newtonsoft.Json`：

```powershell
# 需确保 Newtonsoft.Json.dll 可用
Add-Type -Path "Newtonsoft.Json.dll"
$json = [Newtonsoft.Json.JsonConvert]::SerializeObject($response)
```

无论哪种方式，显式写入文件都使用无 BOM 的 UTF-8，这是跨平台工具链最安全的格式。

## 踩坑点

1. **终端编码 vs 管道编码分离**  
   即使在 Windows Terminal 中看到中文正常，也不代表通过标准输出传递时不会损坏。Agent 捕获的通常是进程的标准输出流，其编码由 `[Console]::OutputEncoding` 决定。务必在脚本开头就设定。

2. **BOM 造成的解析异常**  
   Windows PowerShell 的 `Out-File -Encoding UTF8` 默认带 BOM，许多 JSON 解析器会将 BOM 视为非法字符，导致 `Unexpected token` 错误。因此强烈建议使用无 BOM 的 UTF-8 写入，或使用 `[System.IO.File]::WriteAllText` 控制编码。

3. **`-Depth` 参数未指定时截断数据**  
   `ConvertTo-Json` 默认序列化深度为 2，超过深度的部分会变成字符串 `"..."` 或直接丢失，这在复杂 API 响应中会导致数据损坏，与中文问题无关但容易被误认为乱码。务必根据实际响应深度设置 `-Depth`。

4. **混合使用不同 PowerShell 版本**  
   OpenClaw 或其他 Agent 可能自动检测 `powershell.exe` 而非 `pwsh.exe`，导致使用了带有转义问题的旧版序列化器。可以在脚本开头检查版本，若为 5.1 则自动切换为 .NET 方式，确保行为一致。

## 可复用建议

- **将所有 PowerShell 工具脚本的编码策略标准化**：在脚本顶部添加以下代码块，根据脚本宿主统一输出编码。

```powershell
# 统一输出编码为 UTF-8 无 BOM
if ($PSVersionTable.PSVersion.Major -ge 7) {
    [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
    $outputEncoding = [System.Text.Encoding]::UTF8
} else {
    # Windows PowerShell: 使用 .NET 确保无 BOM 写入
    $PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
    # 同时设定标准输出编码为 UTF-8（可能需要 chcp 65001 配合）
    $outputEncoding = [System.Text.Encoding]::UTF8
    [Console]::OutputEncoding = $outputEncoding
}
```

- **对 JSON 序列化进行封装**：创建工具函数 `ConvertTo-JsonSafe`，内部判断 PowerShell 版本，选择不转义中文的序列化方式，统一提供给所有 Agent 脚本调用。
- **在 MCP 配置中明确 PowerShell 参数**：如果使用 MCP server 启动子进程，确保命令行包含 `-NoProfile -OutputFormat Text` 并设置环境变量 `$env:LC_ALL='C.UTF-8'` 或类似措施，避免交互式 Shell 的文化信息干扰。
- **自动化测试中加入编码验证**：在 CI 流程中，用 Python 脚本调用 PowerShell 工具脚本，解析输出 JSON 并断言中文字段无转义序列、无乱码，可防止回归。

## 总结

Windows 上的 PowerShell 处理中文 JSON 时容易“打坏”，本质是历史遗留的编码假设与现今 UTF-8 通用实践之间的错配。这种错配在 Agent、MCP、插件链等自动化场景中被放大，因为数据要流经多个进程边界。

解决路径很清晰：要么统一迁移到 PowerShell 7 并控制输出编码，要么在 5.1 中用 .NET 的现代 JSON API 接管序列化。无论哪种，最终都要以“无 BOM UTF-8 写入”作为交付标准，让下游无论是什么语言、什么平台，都能稳定消费。

对 OpenClaw/Agent 开发者来说，这个经验还可以上升为一条原则：当你代理一个外部脚本执行器时，永远不要相信其输出流的默认编码；显式指定、显式验证，是自动化可靠性的基石。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/edd62bbc536ad730.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/a284fe2107e16101.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/91e866f53a6bab18.png)

