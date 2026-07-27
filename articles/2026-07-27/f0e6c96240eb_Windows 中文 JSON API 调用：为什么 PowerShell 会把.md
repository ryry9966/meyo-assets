---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏？一次工程排障实录
feedId: 30685
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景：一个看似完美的 MCP 工具为何返回乱码

我在 Windows 上写了一个简单的 OpenClaw MCP 工具，用 PowerShell 脚本封装了一个内部系统的 JSON API。脚本很简单：`Invoke-RestMethod` 请求后端，解析返回的 JSON，提取中文摘要字段，输出给 OpenClaw。在终端直接运行脚本时，控制台能正常显示中文，但一旦通过 OpenClaw 的 MCP server 调用，返回内容里的中文字符就变成了 `????` 或形如 `ä½ å¥½` 的乱码。

直觉告诉我这是编码问题，但第一反应是去查 OpenClaw 的进程编码，结果发现根源在于 **PowerShell 5.1 在管道和文件重定向场景下对 UTF-8 的处理极度不可靠**——而这个坑在 Windows 中文环境里几乎必踩。

## 问题定位：两次编码转换，一次都没猜对

缩到最小复现步骤：在 PowerShell 5.1 中执行以下命令，并把输出重定向到文件。

```powershell
$resp = Invoke-RestMethod -Uri "https://httpbin.org/anything" -Body '{"msg":"你好"}' -Method POST
$resp.json.msg | Out-File -FilePath result.txt
```

用记事本打开 `result.txt`，看到的是 `ä½ å¥½`。但在控制台直接 `Write-Host $resp.json.msg` 又显示正常。

原因分两层：

1. **`Invoke-RestMethod` 的返回值编码取决于 `$OutputEncoding`**。在 PowerShell 5.1 中，`$OutputEncoding` 默认是 `Windows-1252`（代码页 1252）。即使后端返回的 JSON 是 UTF-8，PowerShell 在将字节流转换为 .NET 字符串时，依赖的是 `$OutputEncoding`，导致中文字节被错误地按 Latin-1 解释，生成了错误的字符串对象。

2. **`Out-File` 默认使用 `Unicode` (UTF-16LE) 编码，但管道传入的已经是损坏的字符串**。于是损坏的内容又被按 UTF-16LE 写入文件，记事本再按 UTF-8 打开时，就看到了经典的“烫烫烫”或乱码。

至于控制台能显示，是因为 `Write-Host` 直接输出到控制台宿主，而 PowerShell 控制台的 `[Console]::OutputEncoding` 默认是系统 OEM 代码页（中文 Windows 为 936），恰好能按 GBK 解释损坏的字节，有时看起来“似乎正常”。这也是最具迷惑性的地方。

## 修复尝试：你设置的可能还不够

### 尝试一：只改 `$OutputEncoding`（失败了一半）
```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
$resp = Invoke-RestMethod -Uri ... -Method POST
```
这时 `$resp.json.msg` 在内存中已经是正确的字符串，但用 `Out-File` 或 `>` 重定向仍然可能出错，因为文件写入编码没匹配。

### 尝试二：加 `-ContentType` 或直接处理字节（可靠但不优雅）
```powershell
$resp = Invoke-RestMethod -Uri ... -ContentType "application/json; charset=utf-8" -Method POST
```
仅加 `-ContentType` 并不能改变 `$OutputEncoding` 的作用，仍然依赖于会话的编码设置。真正起作用的是在请求时显式处理响应字节：
```powershell
$respBytes = Invoke-WebRequest -Uri ... -Method POST
$jsonString = [System.Text.Encoding]::UTF8.GetString($respBytes.Content)
$obj = $jsonString | ConvertFrom-Json
```
这略过了 `$OutputEncoding`，但代码变丑，且丢失了 `Invoke-RestMethod` 的自动解析优势。

### 最终可用方案：组合拳

在我的 MCP 工具脚本头部统一加上了这几行：

```powershell
# 强制会话编码为 UTF-8，影响 Invoke-RestMethod 和外部命令
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

# 调用 API
$resp = Invoke-RestMethod -Uri "https://api.internal.example.com/data" -Method GET

# 显式指定输出编码为 UTF-8 with BOM（兼容记事本）或纯 UTF-8
$resp.summary | Out-File -FilePath $env:TEMP\mcp_result.json -Encoding utf8
```

注意：`-Encoding utf8` 在 Windows PowerShell 5.1 中输出的是 **带 BOM 的 UTF-8**；而在 PowerShell 7 中，`-Encoding utf8NoBOM` 才是无 BOM 的 UTF-8。如果要写入 JSON 文件供其他程序读取，建议使用 `utf8NoBOM`（需 PS7）或用 `[System.IO.File]::WriteAllText($path, $jsonString, [System.Text.UTF8Encoding]::new($false))` 绕过 cmdlet。

## 踩坑点记录

- **`Out-File` 和 `Set-Content` 的默认编码不同**：`Out-File` 在 PS5.1 默认是 `Unicode`（UTF-16LE），`Set-Content` 默认是 `Default`（系统 ANSI 代码页），两者都会破坏已修复的中文。
- **管道传给外部程序也不可靠**：如果脚本里把中文用管道传给 `python` 或 `node`，需要保证 `$OutputEncoding` 匹配外部程序期望的输入编码，否则同样会乱码。
- **OpenClaw MCP server 侧**：当我用 `child_process` 调用 PowerShell 脚本时，其 `stdout` 默认被 Node.js 按 UTF-8 解码。如果 PowerShell 输出的是 UTF-16 或 GBK 字符串，Node 侧就会收到乱码。解决办法是在 PowerShell 脚本中统一输出 UTF-8，并让 Node 侧不额外转码。

## 可复用建议

1. **所有面向自动化、被外部进程消费的 PowerShell 脚本，顶部固定设置**：
   ```powershell
   $OutputEncoding = [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
   ```
2. **优先使用 `Invoke-RestMethod`，但记得先改 `$OutputEncoding`**；若追求极致稳定，改用 `Invoke-WebRequest` + 手动解码。
3. **输出到文件时，显式指定 `-Encoding utf8NoBOM`（PS7）或使用 .NET API**，避免依赖 cmdlet 的默认编码。
4. **在自动化链路上统一 UTF-8**：OpenClaw Agent → MCP Server（Node/ Python）→ 调用 PowerShell 子进程，每一层都强制使用 UTF-8，减少猜测。
5. **PowerShell 7 是更友好的选择**：其默认 `$OutputEncoding` 已经是 UTF-8，`Out-File` 提供 `utf8NoBOM`，且与 .NET Core 的编码行为一致，值得升级。

## 总结

PowerShell 在 Windows 上的编码行为是一场由历史包袱和向后兼容共同塑造的迷局。中文开发者在做 JSON API 调用时，很容易被“控制台看着正常”的假象欺骗。只要记住：**一旦输出离开 PowerShell 的显示宿主，进入管道、文件或外部进程，编码就从头开始失控**。好在通过显式的编码控制，我们可以把这场灾难扼杀在脚本开头三行。

这类问题看似简单，但在 Agent 与 MCP 交织的自动化环境里，往往会在凌晨两点突然爆发。希望这份排障记录能帮你少熬一夜。

---

