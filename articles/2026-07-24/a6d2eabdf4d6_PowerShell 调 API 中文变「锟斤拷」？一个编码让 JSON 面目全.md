---
title: PowerShell 调 API 中文变「锟斤拷」？一个编码让 JSON 面目全非
feedId: 30272
source: 综合讨论
publishedAt: 2026-07-24
---

## 背景

在 Windows 上用 PowerShell 写自动化脚本、调用各种 REST API，早已是 Agent、MCP 插件、OpenClaw 流水线里的常规操作。尤其是处理中文内容——模型返回、网页抓取、本地文件读取——经常要在脚本里拼一份 JSON，通过 `Invoke-RestMethod` 发出去。

但现实是：服务端收到的中文经常变成 `????`、`锟斤拷` 或者一串 `\u951f\xe6\x96\xa4` 之类的“不可描述”。更离谱的是，本地打印看起来正常，一发出去就坏，排查起来常常让人怀疑是不是网络中间层动了手脚。

这篇文章不会讲“学编码从 ASCII 开始”，也不会搬热点，只从工程化角度把原因、复现、修复、可复用方案讲清楚。

---

## 问题

假设你有一段脚本：

```powershell
$body = @{ content = "你好世界" } | ConvertTo-Json
Invoke-RestMethod -Uri "https://example.com/api/echo" -Method Post -Body $body
```

服务端收到的 `content` 字段变成乱码，或者直接被替换成问号。又比如，你从文件读了一条中文提示词：

```powershell
$prompt = Get-Content .\prompt.txt
Invoke-RestMethod -Uri "https://api.llm/v1/chat" -Method Post -Body ($prompt | ConvertTo-Json)
```

同样中文全损。控制台打印 `$body` 明明是对的，但一进网络就“掉字”。更隐蔽的情况：有些 API 会返回 `400` 或 `500`，日志里却看不到异常，因为 JSON 结构已被破坏。

---

## 原因

根本原因不是 PowerShell 不会发中文，而是 **编码协商链的每一环都可能使用不同的默认编码**。

### 1. 文件读入环节

`Get-Content` 在没有显式指定 `-Encoding` 时，使用系统当前的 ANSI 代码页（在中文 Windows 下通常是 GBK/CP936）。如果你的 `.txt` 文件实际是 UTF-8 编码，那么读进来的“你好世界”就已经被转成了错误的字符串，`ConvertTo-Json` 只是把错误的内容原样序列化。

### 2. 内存中的字符串

.NET 内部字符串是 UTF-16，`ConvertTo-Json` 输出的 JSON 文本同样是 UTF-16 字符串。当 `Invoke-RestMethod` 准备把 `-Body` 发送出去时，它需要将字符串转成字节流。此时，它会检查 `Content-Type`：

- 如果没有显式指定 `Content-Type`，默认行为是 `application/x-www-form-urlencoded`，并可能使用系统默认编码（ANSI）进行百分号编码，中文直接损坏。
- 如果指定了 `-ContentType "application/json"`，`Invoke-RestMethod` 会使用 **UTF-8 编码** 将 JSON 字符串转为字节流，这一步本身没有问题，但前提是 JSON 字符串中的中文字符是正确读入的——即源头就没对，编码再正确也无济于事。

### 3. 输出显示乱码的假象

有时调 API 返回的中文在控制台显示为乱码，这会和发送乱码的问题混淆。 `Invoke-RestMethod` 返回的结果是 .NET 对象，如果直接用 `Write-Host` 或直接输出，控制台的编码设置（`[Console]::OutputEncoding`）可能导致显示异常，但实际变量里的数据是正常的。这属于**展示问题**，不是数据损坏问题，需要区分。

---

## 复现步骤

在一个中文 Windows 10/11 的 PowerShell 5.1 环境，用几行脚本就能稳定复现：

1. 创建 UTF-8 编码的 `prompt.txt`，内容为 `你好，世界`。
2. 执行：
   ```powershell
   $body = @{ text = (Get-Content .\prompt.txt) } | ConvertTo-Json
   ```
3. 将 `$body` 发送到一个回显 JSON 的测试接口（或用 `nc` 监听），你会发现服务端收到的 `text` 已经变成 `��` 或 `????`。

再用另一种方式构造：
```powershell
$body = '{"text":"你好，世界"}'   # 直接手写 JSON
Invoke-RestMethod -Uri ... -Body $body  # 没有 -ContentType
```
服务端同样乱码，因为默认的 `Content-Type` 没有指定 JSON，编码规则使用了 ANSI。

---

## 做法 / 修复步骤

### 1. 文件读入时必须指定编码

```powershell
$prompt = Get-Content .\prompt.txt -Encoding UTF8
```

这是最容易被忽略但最致命的一环。

### 2. 构建 JSON 时使用对象序列化，而不是手拼字符串

```powershell
$payload = @{
    model = "gpt-4"
    messages = @(
        @{ role = "user"; content = $prompt }
    )
}
$jsonBody = $payload | ConvertTo-Json -Compress
```
`ConvertTo-Json` 会自动将 .NET 对象转为 UTF-16 字符串，这一步中文保持完整。

### 3. 调用时明确指定 Content-Type

```powershell
Invoke-RestMethod -Uri "https://api.example.com/chat" `
    -Method Post `
    -Body $jsonBody `
    -ContentType "application/json; charset=utf-8"
```
`-ContentType` 告诉 `Invoke-RestMethod` 使用 UTF-8 字节序列化，与服务端正确协商。

### 4. （可选）控制台输出乱码的处理

仅当返回值在终端显示为乱码时，可设置：
```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```
但这**不会影响发送出去的请求内容**，只是让肉眼看着舒服。

---

## 踩坑点

- **Get-Content 的默认编码陷阱**  
  很多教程不写 `-Encoding`，在纯英文环境没问题，一到中文就直接掉坑。

- **认为设置 `$OutputEncoding` 就能解决一切**  
  `$OutputEncoding` 影响的是与外部程序管道通信的编码，对 `Invoke-RestMethod` 几乎无影响。不要病急乱投医。

- **用 `[System.Text.Encoding]::UTF8.GetBytes($str)` 手动处理 Body**  
  这本身思路正确，但容易和 `-Body` 参数的行为冲突。如果传入 Byte[]，`Invoke-RestMethod` 会原样发送，不会再做编码转换。但此时 `-ContentType` 仅作为 Header 发送，必须确保发送的字节流与 `charset` 一致，否则服务端会解析错误。更稳妥的方式是用字符串 + `-ContentType`，让 Cmdlet 自己处理编码。

- **PowerShell ISE 的显示误导**  
  ISE 的编码处理与普通控制台不一致，有时看起来正常，一旦部署到任务计划或 CI 中就会报废。始终用 `powershell.exe -NoProfile -File script.ps1` 的方式测试。

---

## 可复用建议

将发送 JSON 的操作封装成一个可靠的函数，避免每次踩坑：

```powershell
function Invoke-JsonApi {
    param(
        [string]$Uri,
        [hashtable]$Body,
        [string]$Method = 'Post'
    )
    $json = $Body | ConvertTo-Json -Compress -Depth 10
    Invoke-RestMethod -Uri $Uri `
        -Method $Method `
        -Body $json `
        -ContentType "application/json; charset=utf-8"
}
```

所有文件读取统一使用 `-Encoding UTF8`，或者在脚本头部约定：

```powershell
$PSDefaultParameterValues = @{
    'Get-Content:Encoding' = 'utf8'
    'Out-File:Encoding'    = 'utf8'
}
```

---

## 总结

PowerShell 把中文打坏，不是因为“PowerShell 对中文不友好”，而是因为 **从文件到内存、从内存到网络，每一步的默认编码都无法保证与 UTF-8 对齐**，而大多数现代 API 偏偏只认 UTF-8。

搞清楚整条路径：**源编码（文件）→ 内存（.NET 字符串）→ 序列化（JSON）→ 网络传输（Content-Type + 字节编码）**，就能把问题范围缩小到一两步。记住三板斧：

1. **文件读入强制 UTF-8**；
2. **用 `ConvertTo-Json` 构造 JSON，别手拼**；
3. **`-ContentType "application/json; charset=utf-8"` 写死**。

OpenClaw 的自动化流程里，这类细节决定稳定性。希望这篇可以帮你省下半天查 `?` 的时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/7d97ff07854bb4a7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/7a72308341863d2a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/03ebc99aa5830a71.png)

