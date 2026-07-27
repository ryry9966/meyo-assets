---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏——编码陷阱与工程化修复
feedId: 30656
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景：当 LLM 返回的中文 JSON 在 PowerShell 里变成天书

在 OpenClaw 这类 Agent 和 MCP 自动化实践中，用 PowerShell 调用 REST API 获取中文内容非常普遍。典型场景：用 `Invoke-RestMethod` 拉取 LLM 生成的摘要、翻译结果或结构化 JSON，再喂给下一个插件、写入文件或通过管道传给其他进程。可一旦中文出现在 JSON 里，Windows 下的 PowerShell 经常让它们变成形如 `ä½ å¥½` 的乱码，不仅破坏可读性，还会导致下游解析失败。

这并不是玄学，而是 PowerShell 与外部世界交互时，编码协商不充分的必然结果。

## 问题复现：看似合理的调用，输出却一塌糊涂

在 PowerShell 5.1 控制台里执行下面脚本：

```powershell
$response = Invoke-RestMethod -Uri "https://echo.example.com/data" -Method Post -Body '{"text":"你好世界"}'
$response | ConvertTo-Json -Depth 3 | Out-File output.json
cat output.json
```

打开 `output.json`，看到的很可能是 `"{ \"message\": \"ä½ å¥½ä¸–ç•Œ\" }"` 之类的乱码。更糟糕的是，直接把 `$response` 通过管道传给另一个进程（比如 `curl`、`python`）时，接收方也拿到损坏的字节流。

## 根本原因：三层编码不一致叠加

PowerShell 本身托管在 .NET 框架上，内部字符串都是 Unicode（UTF-16 LE）。问题主要出在三个环节：

1. **控制台输出编码**  
   PowerShell 控制台的 `[Console]::OutputEncoding` 默认是系统 OEM 代码页（简体中文 Windows 通常是 `cp936`）。当字符串通过管道或标准输出离开 PowerShell 进程时，会按此编码转换。中文字符在 GBK/CP936 里有对应映射，但如果接收方硬解析为 UTF-8，就会乱码。

2. **文件写入时的默认编码**  
   PowerShell 5.1 的 `Out-File`、`>` 重定向操作符默认使用 **UTF-16 LE with BOM**。如果另一个工具（如 VS Code、Python 脚本）按 UTF-8 打开，会看到每个字符间夹杂 `\x00` 或直接报错。而 `Invoke-WebRequest` 的响应内容可能按服务器返回的编码自动转成字符串，但 `Content` 属性本身是 `[byte[]]` 时更容易出错。

3. **`$OutputEncoding` 变量的误导**  
   `$OutputEncoding` 仅影响将字符串通过管道传给**外部原生命令**时的编码，而不影响文件重定向或 `Out-File`。很多人误以为设置它就万事大吉，结果依然乱码。

另外，PowerShell 7+ 改变了部分默认行为（如 `Out-File` 默认 UTF-8 without BOM），但在 Windows Server 2016 等环境里仍以 PowerShell 5.1 为主，问题集中爆发。

## 工程化做法：显式控制每一个边界

### 1. 读取 API 响应，立即转换到正确的 .NET 字符串

`Invoke-RestMethod` 已经做了 JSON 解析，返回 PSCustomObject，内部字符串是正常的。所以重点是写出时的控制。若遇到 `Invoke-WebRequest`，需要手动解码：

```powershell
$resp = Invoke-WebRequest -Uri $uri
$json = [System.Text.Encoding]::UTF8.GetString($resp.Content)
$obj = $json | ConvertFrom-Json
```

确保不要用 `$resp.Content` 直接当字符串。

### 2. 写入文件一律指定 UTF-8 (无 BOM 或有 BOM)

在脚本开头全局强制：

```powershell
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['Set-Content:Encoding'] = 'utf8'
```

或每次显式指定：

```powershell
$response | ConvertTo-Json -Depth 5 | Out-File -Encoding utf8 output.json
```

如需兼容旧版 Windows 记事本，加 BOM：

```powershell
$utf8Bom = New-Object System.Text.UTF8Encoding($true)
[System.IO.File]::WriteAllText("output.json", $json, $utf8Bom)
```

### 3. 管道传给外部程序时，对齐编码预期

假设要把 JSON 通过管道传给 Python 脚本：

```powershell
$json | python detect.py
```

这时必须确保 PowerShell 传出的字节流是 UTF-8：

```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
[Console]::OutputEncoding = $OutputEncoding
```

同时 Python 脚本也必须以 UTF-8 读取 `sys.stdin`（`PYTHONIOENCODING=utf-8`）。

## 踩坑点与可复用建议

- **不要用 `>` 重定向写文件**  
  `>` 等同于 `Out-File` 且无法追加 `-Encoding` 参数。用 `Out-File -Encoding utf8` 或 `Set-Content -Encoding utf8` 替代。

- **`$OutputEncoding` 与 `[Console]::OutputEncoding` 不是一回事**  
  前者用于管道到外部命令，后者用于控制台显示（主机输出）。两个都要设。

- **PowerShell 7+ 用户并非高枕无忧**  
  PWSH 的 `Out-File` 默认 utf8NoBOM，但控制台编码仍受系统区域影响。在非英文 Windows 或 SSH 远程会话中，`[Console]::OutputEncoding` 可能仍是 cp850 之类。建议始终显式设置。

- **JSON 序列化时建议带上 `-Compress` 避免额外换行干扰**  
  `ConvertTo-Json -Depth 10 -Compress` 并结合 `[System.Text.Encoding]::UTF8.GetBytes()` 写入文件，可以绕过 Pipe 的编码陷阱。

- **统一团队模板**  
  在 Agent 或插件的入口脚本 `.ps1` 文件头部插入以下代码段，可避免 90% 的编码问题：

```powershell
$Global:PSDefaultParameterValues = @{
    'Out-File:Encoding' = 'utf8'
    'Set-Content:Encoding' = 'utf8'
}
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
[Console]::OutputEncoding = $OutputEncoding
[Console]::InputEncoding = $OutputEncoding
```

## 总结

PowerShell 中文 JSON 乱码的本质是**隐式编码转换**在 Windows 遗产设置下踩坑。自动化工程师必须认识到：任何跨越 PowerShell 进程边界的字符串传输，都需要显式锁定 UTF-8。这不仅是为了中文正确性，更是为 Agent 流水线里多种语言、工具协同提供确定性。下次你的 PowerShell 脚本对接 OpenClaw 组件时，花 5 行代码做编码约束，可能省下半天排查乱码的时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/f3e909e279137f00.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/c169b3493941f53e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/fda09fb781961c9b.png)

