---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 28972
source: 综合讨论
publishedAt: 2026-07-14
---

## 背景

在 OpenClaw、各种 Agent 插件或自动化脚本里，我们经常需要从本地 Windows 环境调用一个返回中文 JSON 的 API。PowerShell 因为系统自带、无需额外安装，常被选作胶水语言。然而，几乎每个第一次用 PowerShell 处理中文 API 的工程师都会遇到同一个问题：**返回的 JSON 里中文变成了 `?`、乱码或者被截断**。

这个坑不是偶发的 bug，而是 Windows 上多种编码约定冲突的必然结果。理解了它，也就理解了在 MCP/插件管道里用 PowerShell 安全处理文本的核心原则。

## 问题复现

假设我们有一个 REST API，返回这样的 JSON：

```json
{"status":"成功","message":"你好，世界"}
```

用 PowerShell 5.x（Windows 10/11 自带版本）标准写法调用：

```powershell
$response = Invoke-RestMethod -Uri "http://localhost:8080/api/demo"
$json = $response | ConvertTo-Json
Write-Output $json
```

控制台输出可能变成：

```json
{"status":"??","message":"????"}
```

而如果直接用 `Invoke-WebRequest` 保存到文件再打开，文件内容可能会正确，但也可能出现半角乱码。更诡异的是，同样的脚本在 VS Code 的终端里正常，在系统 CMD 里乱码；或者在任务计划里跑乱码，手动跑又好了。

## 根因分析

所有问题的核心在于 PowerShell 处理文本时的 **三层编码不一致**：

1. **响应字节流的解码编码**  
   `Invoke-RestMethod` 和 `Invoke-WebRequest` 内部使用 HTTP 响应头中的 `Content-Type` 来决定如何把字节流转成字符串。如果 API 返回的 `Content-Type` 是 `application/json` 但**没有指定 charset**，Windows PowerShell 默认会用 **ISO-8859-1**（类似 ASCII 的超集）去解码，中文字节会被当作非法字符丢弃或转成 `?`。而 API 服务端很可能实际用的是 UTF-8。

2. **控制台输出编码（`[Console]::OutputEncoding`）**  
   即便已经正确解码为 .NET 字符串，PowerShell 在把字符串发送到控制台时，还会根据 `[Console]::OutputEncoding` 再一次转码。在中文 Windows 上，这个值默认是 `GBK`（代码页 936）。如果你的 PowerShell 字符串里包含了一些 GBK 不支持的字符（或终端字体不支持），输出仍然会乱码。更麻烦的是，如果某些自动化环境把标准输出重定向到文件或管道，编码行为又变了。

3. **文件输出编码**  
   `Out-File`、`Set-Content`、重定向 `>` 默认编码在 Windows PowerShell 中是 **UTF-16 LE** 或 **ASCII**（视情况而定），但绝不是 UTF-8 without BOM。这就导致落盘的数据被其他程序（如 Logstash、Agent）读取时出现编码级错乱。

三者叠加，让“打个中文 API”变成了编码排障马拉松。

## 稳定做法：以字节流为中心

在工程化脚本里，最可靠的做法是自己掌控字节 -> 字符串的转换，并显式指定一切编码。

### 1. 用 `Invoke-WebRequest` 拿原始字节

```powershell
$resp = Invoke-WebRequest -Uri "http://localhost:8080/api/demo" `
                           -UseBasicParsing
$rawBytes = $resp.Content.RawContent  # 或者 $resp.RawContentStream
```

若 API 总是 UTF-8，则直接解码：

```powershell
$utf8 = [System.Text.Encoding]::UTF8
$bodyString = $utf8.GetString($resp.Content)
```

若要自动依据响应头 charset，可以解析 `$resp.Headers['Content-Type']` 中的 charset 参数，并匹配对应 `Encoding`。

### 2. 用 .NET WebClient 或 HttpClient

如果愿意放弃 PowerShell 的高级抽象，直接使用 .NET 的 `HttpClient` 可以完全避开自动解码的歧义：

```powershell
$client = [System.Net.Http.HttpClient]::new()
$response = $client.GetAsync("http://localhost:8080/api/demo").Result
$bytes = $response.Content.ReadAsByteArrayAsync().Result
$jsonStr = [System.Text.Encoding]::UTF8.GetString($bytes)
```

随后 `$data = $jsonStr | ConvertFrom-Json` 即可得到正确的中文对象。

### 3. 控制台与文件输出强制 UTF-8

在需要控制台打印或输出到文件时，同时设置控制台编码并强制文件编码：

```powershell
# 控制台输出编码设为 UTF-8（需终端支持）
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

# 文件输出时显式指定编码
$jsonStr | Out-File -FilePath "result.json" -Encoding UTF8
```

注意：PowerShell 的 `-Encoding UTF8` 输出的是 **带 BOM** 的 UTF-8；如果下游工具（如 OpenClaw Agent 的 JSON 解析）不接受 BOM，可以用 `[IO.File]::WriteAllText("result.json", $jsonStr, [System.Text.UTF8Encoding]::new($false))` 输出无 BOM 的 UTF-8。

## 踩坑实录

- **`Invoke-RestMethod` 的 `-ContentType` 参数无效于解码**  
  这个参数只是设置 **请求头**的 Content-Type，不会影响响应解码逻辑。很多新手误以为加了这个就能解决问题。

- **PowerShell 5.x 与 7+ 行为不同**  
  PowerShell Core (pwsh) 在处理没有 charset 的 `application/json` 时会尝试按 UTF-8 猜测，乱码概率降低，但仍不绝对安全。如果环境允许，优先使用 pwsh 7+。

- **管道编码问题**  
  当 PowerShell 的输出被其他进程的 `stdin` 接收时（例如 `script.ps1 | python parse.py`），控制台编码不再适用，系统代码页会再次介入。推荐用 `Start-Process` 配合 `-RedirectStandardOutput` 和 `-NoNewWindow` 并指定编码，或者直接输出到文件。

- **IDE 终端透明化**  
  VS Code 的 PowerShell 集成终端可能覆盖了 `[Console]::OutputEncoding`，导致测试时一切正常，部署到系统任务后却乱码。验证时务必使用系统原生的 `powershell.exe -File script.ps1` 或 ISE。

## 可复用模块

将以上逻辑封装成一个安全函数，可在所有脚本里复用：

```powershell
function Invoke-Utf8Api {
    param([string]$Uri)
    $wc = [System.Net.WebClient]::new()
    $wc.Encoding = [System.Text.Encoding]::UTF8
    $jsonString = $wc.DownloadString($Uri)
    return $jsonString | ConvertFrom-Json
}
```

如果你的 API 始终 UTF-8，这段函数就够了。更健壮的版本可以检查 `Content-Type` 并自动选取 `Encoding`。

## 总结

在 Windows 上用 PowerShell 打中文 JSON API，**默认行为不可信，必须显式接管文本编码链**。三个关键点：

- 响应解码：接管字节流，显式指定 UTF-8 或从响应头推导 charset。
- 控制台输出：设置 `[Console]::OutputEncoding` 或在脚本中避免直接依赖控制台打印。
- 文件落盘：始终指定 `-Encoding UTF8`（或 UTF8NoBOM）。

在 OpenClaw/MCP 这类自动化流水线里，编码问题一旦在早期忽略，后续排查成本极高。把这几个固定模式写入团队的脚本模板，能从根源上杜绝“中文打坏”的幽灵问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/bcbf4515b0708670.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/76afe04a987b9b7c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/c77e38c778f578a3.png)

