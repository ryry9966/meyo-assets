---
title: 根治 PowerShell 调 JSON API 时中文被打坏的编码顽疾
feedId: 29978
source: 综合讨论
publishedAt: 2026-07-21
---

## 背景：Windows 上的自动化怪圈

在 OpenClaw、MCP 插件或各类 Agent 自动化里，我们经常需要用最顺手的脚本语言快速拉取 RESTful JSON API。PowerShell 在 Windows 生态下几乎是最“原生”的选择——无需额外安装运行时，可直接调用 .NET 网络栈，还能无缝对接任务计划、CI Runner。但一旦 API 返回的中文开始出现在脚本输出里，事情就变得诡异起来：控制台打印正常，写入文件却变成乱码；又或者文件用记事本打开是乱码，换成 VSCode 才正常；更有甚者，把 JSON 传给下一步的 MCP 工具直接解析失败。这些问题的根源，十有八九指向 PowerShell 的编码处理。

## 问题复现：中文是怎么“坏”掉的

一个最常见的写法：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/data" -Method Get
$resp | ConvertTo-Json -Depth 5 > result.json
```

当 API 返回的 JSON 里包含 `"name":"小明"` 时，你会发现 `result.json` 用 UTF-8 解析器读出来是 `??` 或者诡异符号，用记事本打开是乱码，但用 `type result.json` 在控制台看又正常。另一个场景：

```powershell
$body = @{ text = "你好世界" } | ConvertTo-Json
Invoke-RestMethod -Uri "https://api.example.com/echo" -Method Post -Body $body -ContentType "application/json; charset=utf-8"
```

服务端收到的 `text` 字段可能变成 `ä½ å¥½ä¸–ç•Œ`，或者直接变成问号。这些问题看起来像是网络传输错误，但抓包会发现发送的数据本身就是坏的。

## 根因：PowerShell 的编码“多重宇宙”

PowerShell 在 Windows 上有好几套并行的编码机制，默认行为在不同版本间还有差异，这是造成混乱的主因。

1. **输出重定向 `>` 操作符**  
   `>` 实际上调用的是 `Out-File`，但它的默认编码在 Windows PowerShell 5.1 中是 **UTF-16 LE**（Unicode），而非 UTF-8。当你把一个包含中文字符的字符串通过 `>` 写入文件时，文件被保存为 UTF-16 LE，且带有 BOM。如果下游工具默认按 UTF-8 读取，就会看到乱码。

2. **控制台输出编码**  
   控制台能正常显示中文，是因为 PowerShell 自动匹配了系统代码页（例如简体中文 GBK 页 cp936），或者现代终端（如 Windows Terminal）的默认 UTF-8 设置。但这与文件写入编码是两码事。

3. **`Invoke-RestMethod` 的字符处理**  
   当使用 `Invoke-RestMethod` 获取 JSON 时，它内部使用 `System.Net.Http.HttpClient`，默认会按响应头里的 `charset` 解码字节流。如果 API 正确返回了 `Content-Type: application/json; charset=utf-8`，那么 `$resp` 对象中的中文字符在内存中是正常的 .NET 字符串（UTF-16 内部表示）。但一旦你用 `ConvertTo-Json` 再通过管道输出，编码就容易踩坑。

4. **`[Console]::OutputEncoding` 的影响**  
   在 PowerShell 5.1 中，该值通常等于系统代码页，会影响某些命令向管道写入文本时的编码选择。当你使用 `Out-File -Encoding Default` 时，“Default” 指的是系统 ANSI 代码页，中文系统上就是 GBK，这与 UTF-8 又不兼容。

PowerShell 7+ 在一定程度上改良了默认编码：`>` 在 PS7 里已经变成 UTF-8 无 BOM。但很多 Windows 服务器和 CI 环境仍然内置 PS5.1，而 OpenClaw 或 MCP 宿主的脚本环境很可能是 PS5.1。因此必须写出兼容方案，而不是指望版本升级。

## 正确做法：显式且无 BOM 的 UTF-8

### 1. 安全地将 JSON 写入文件

永远不要用 `>` 写入中文字符内容，改用 `Out-File` 或 `Set-Content` 并显式指定 UTF-8。

**推荐写法 (PS5.1 兼容)：**

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/data" -Method Get
$json = $resp | ConvertTo-Json -Depth 10
$json | Out-File -FilePath "result.json" -Encoding UTF8
```

此处的 `-Encoding UTF8` 在 Windows PowerShell 中生成 **带 BOM 的 UTF-8**。BOM 有时会导致下游 JSON 解析器报错（如 Python 的 `json.load` 默认不忽略 BOM）。更干净的做法是转换为字节流写入：

```powershell
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText("$PWD\result.json", $json, $utf8NoBom)
```

或者用 `Set-Content`（PS5.1 的 `-Encoding UTF8NoBOM` 不可用，此参数是 PS7 专有，所以在 5.1 下推荐上面的 .NET 写法）。

### 2. 发送含中文的 JSON 请求体

构造请求体时，务必确保字符串被正确编码为 UTF-8 字节，且 `Content-Type` 指明字符集。

```powershell
$body = @{ text = "你好世界" } | ConvertTo-Json
$utf8 = [System.Text.Encoding]::UTF8
$bytes = $utf8.GetBytes($body)
Invoke-RestMethod -Uri "https://api.example.com/echo" -Method Post -Body $bytes -ContentType "application/json; charset=utf-8"
```

直接将 `$body` 字符串传给 `-Body` 参数时，PowerShell 会将它转化为字节流。在 PS5.1 中，这个转化默认使用 `[System.Text.Encoding]::Default`，也就是系统 ANSI 编码（GBK），导致 UTF-8 服务端解码失败。显式构造字节数组可以完全避开这个不可靠的默认转化。

### 3. 配置全局编码以减轻心智负担

对于自动化脚本，可以在脚本开头强制设置控制台编码（这会影响到后续从管道读取的外部命令）：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

第一条确保外部程序（如 `curl.exe`）输出中文不挂；第二条让 `Out-File` 和 `Set-Content` 系列命令默认使用 UTF-8。注意，在 PS5.1 中 `*:Encoding` 的值只能是 `string` 格式的已知编码名，且不带 `NoBOM` 选项，所以它生成的是带 BOM 的 UTF-8。如果下游对 BOM 敏感，仍需在最终写入环节用字节写入覆盖。

## 踩坑实录与排障经验

- **坑1：IDE 编码混淆**  
  在 VSCode 中新建 `.ps1` 文件并保存成 UTF-8 无 BOM 后，脚本内 hardcode 的中文可能正常；但用 ISE 编辑保存的脚本默认是 UTF-16 LE，有时会造成执行时的奇怪字符。建议统一用 VSCode 编辑，并在 `.vscode/settings.json` 中设置 `"files.encoding": "utf8"`。

- **坑2：`Out-File -Append` 累积 BOM**  
  若多次追加写入，带 BOM 的 UTF-8 文件会在每次追加时重复写入 BOM，导致文件开头出现多个 `ï»¿`，JSON 解析必然失败。改用 `[System.IO.File]::AppendAllText` 配合无 BOM UTF-8 常量。

- **坑3：管道内编码接力棒**  
  当使用 `| ConvertTo-Json | Out-File` 时，`ConvertTo-Json` 输出的纯字符串在管道中传输不涉及文件编码，但 `Out-File` 决定了最终编码。曾遇到同事在管道中先 `Set-Content` 截断再 `Out-File -Append`，结果前一段是 UTF-8 无 BOM，后一段是带 BOM GBK，文件彻底不可读。

- **坑4：Invoke-WebRequest 的 RawContent**  
  如果想要原始响应内容，`Invoke-WebRequest` 的 `Content` 属性是已解析的字符串（内部已按响应编码解码），但 `RawContent` 包含 HTTP 头，其编码与原始字节不同。要精确获取原始字节，应使用 `Invoke-WebRequest` 的 `-OutFile` 或直接操作 `HttpClient`。

## 可复用建议：构建编码健壮的工具函数

将常用的安全写入和请求打包为 PowerShell 函数，可以有效避免团队中反复犯同样的错误：

```powershell
function Write-Utf8NoBom {
    param([string]$Path, [string]$Content)
    $utf8NoBom = New-Object System.Text.UTF8Encoding $false
    [System.IO.File]::WriteAllText((Resolve-Path $Path).Path, $Content, $utf8NoBom)
}

function Invoke-RestMethodUtf8 {
    param(
        [string]$Uri,
        [hashtable]$Body,
        [string]$Method = 'Post'
    )
    $json = $Body | ConvertTo-Json -Compress
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($json)
    Invoke-RestMethod -Uri $Uri -Method $Method -Body $bytes -ContentType "application/json; charset=utf-8"
}
```

将这些函数放入 `profile.ps1` 或模块中，所有自动化脚本均使用它们，编码问题基本可以杜绝。

## 总结

PowerShell 在 Windows 上处理中文 JSON 时容易打坏，并非语言缺陷，而是编码默认值向后兼容与多版本演变带来的认知成本。根本战法只有一条：**永远显式指定 UTF-8，并清楚你当前版本中 `-Encoding UTF8` 是否带 BOM**。在面向 OpenClaw 或 MCP 的自动化里，数据流一旦跨进程、跨语言，任何隐藏的编码偏差都会在集成时放大。养成使用字节流写入和显式编码的习惯，比花一整天排查“为什么 Python 读这个 JSON 崩了”要划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/b3c25bab3f3814fa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/5019151acbc6317a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/fb3376cbc6417495.png)

