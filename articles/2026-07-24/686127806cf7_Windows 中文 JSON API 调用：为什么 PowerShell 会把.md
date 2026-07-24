---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏？
feedId: 30289
source: 综合讨论
publishedAt: 2026-07-24
---

# Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏？

## 背景

在 Windows 上构建 OpenClaw、MCP 插件、自动化脚本时，通常需要用 PowerShell 调用返回中文内容的 JSON API。常见操作是 `Invoke-RestMethod` 或 `Invoke-WebRequest`，拿到数据后：
- 写入文件供下游消费
- 在控制台输出供人工调试
- 通过标准输出传递给另一个进程（例如 Python、Node）

但很多开发者会发现：明明 API 返回的响应体在浏览器或 Postman 里中文显示正常，一到 PowerShell 脚本里就变成 `????`、`ç±³é¥­`（这类莫哈维乱码），或者写入文件后用记事本打开是一堆乱码。问题不复杂，但如果不搞清楚 Windows 控制台和 PowerShell 的编码模型，就会反复踩坑。

## 问题：到底哪里打坏了

典型场景：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/items" -Method Get
$resp.name  # 本应是“你好”，输出却是“??”或“ä½ å¥½”
$resp | ConvertTo-Json | Out-File data.json
# data.json 里中文全乱
```

或者用 `>` 重定向：

```powershell
Invoke-WebRequest -Uri ... | Select-Object -Expand Content > output.txt
# output.txt 中文变乱码
```

根本原因并不是 `Invoke-RestMethod` 本身不会处理 UTF-8，而是**后续步骤的编码假设与数据本身的编码不一致**。

## 根因分析

### 1. PowerShell 控制台输出编码

Windows PowerShell 5.1 的控制台默认输出编码由 `[Console]::OutputEncoding` 决定，通常是系统 OEM 代码页（如简体中文 GBK 936 或英文系统 437）。当你在控制台打印字符串时，PowerShell 会将 .NET 字符串（内部 UTF-16）按 **OutputEncoding** 编码成字节流写入控制台。如果该编码不支持某些 Unicode 字符，就会变成问号。

更隐蔽的是，当使用 `>` 重定向时，PowerShell 使用的是 **`$OutputEncoding`** 变量（默认为 ASCII），而不是控制台编码。这就导致即使控制台能显示中文，文件里仍然是 ASCII 损坏。

### 2. Web cmdlet 的自动解码

`Invoke-RestMethod` 和 `Invoke-WebRequest` 会自动根据响应的 `Content-Type` 头部（如 `charset=utf-8`）将字节流解码为 .NET 字符串，该字符串本身是正确的。但如果响应缺少 charset，或者服务器错误声明为 `charset=ISO-8859-1`，cmdlet 可能按错误编码解码，造成原始字节层面的损坏。此时即使后续编码正确也无济于事。

### 3. Out-File / Set-Content 的默认编码

`Out-File` 和 `Set-Content` 在 Windows PowerShell 5.1 里默认使用 **UTF-16 LE**（带 BOM），而不是 UTF-8。如果你期望下游工具读取 UTF-8，就会乱码。`>` 重定向用的是 `$OutputEncoding`（默认 ASCII），更糟糕。

### 4. 直接输出对象的行为

当把对象直接输出到控制台，PowerShell 的格式化系统会调用 `.ToString()` 等方法，仍受到 OutputEncoding 约束。如果数据中文字符在代码页范围外，显示为 `?`。

## 做法与步骤

### 步骤一：确保 API 响应被正确解码

优先信任服务器返回的 charset。如果服务器缺失或错误，可手动获取原始字节并强制按 UTF-8 解码：

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/items" -Method Get
$rawBytes = $response.RawContentStream.ToArray()
$correctString = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$data = $correctString | ConvertFrom-Json
```

或者使用 `-ContentType` 提示（但该参数主要用于发送请求，并非强制解码）。

### 步骤二：控制输出编码

**方案 A：设置 OutputEncoding 与重定向编码**

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

此后 `>` 重定向和 `Out-File -Encoding utf8` 均可正常输出 UTF-8。注意：`$OutputEncoding` 只影响 `>`，不影响 `Out-File`，后者需要显式指定 `-Encoding utf8`。

**方案 B：显式指定编码写入文件**

```powershell
$data | ConvertTo-Json -Depth 3 | Out-File -FilePath "data.json" -Encoding UTF8
# 或使用 Set-Content -Encoding UTF8
```

如果要求无 BOM 的 UTF-8，可以用 .NET 方法：

```powershell
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText("data.json", ($data | ConvertTo-Json), $utf8NoBom)
```

### 步骤三：发送中文请求体

发送 JSON 带中文时，必须明确指定 charset：

```powershell
$body = @{ name = "你好" } | ConvertTo-Json
Invoke-RestMethod -Uri "..." -Method Post -Body $body -ContentType "application/json; charset=utf-8"
```

`-ContentType` 会写入请求头，确保服务端按 UTF-8 解析。

### 步骤四：在 MCP / Agent 管道中传递

如果是通过标准输出传递 JSON 给 MCP 服务器或 Agent 进程，务必确保进程启动时的编码设置：

- 在 PowerShell 脚本开头加入 `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8`
- 如果使用 `Start-Process` 或管道，检查接收端是否默认 UTF-8 解码

一个可复用的安全函数：

```powershell
function Get-ApiJsonUtf8 {
    param($Uri)
    $resp = Invoke-WebRequest -Uri $Uri
    $raw = $resp.RawContentStream.ToArray()
    [System.Text.Encoding]::UTF8.GetString($raw) | ConvertFrom-Json
}
```

## 踩坑点复盘

- **直接 `>` 输出到文件**：在未修改 `$OutputEncoding` 前，等于用 ASCII 破坏中文。调了半天还以为是 API 问题。
- **只在控制台编码下手，忘了文件编码**：设置 `[Console]::OutputEncoding = UTF8` 后控制台正常了，但 `Out-File` 仍是 UTF-16，下游 Python 读取报错。
- **`ConvertTo-Json` 深度不够截断数据**：默认 `-Depth 2` 可能丢失嵌套对象，先确认 `-Depth` 足够，别误以为是编码问题。
- **BOM 干扰**：某些工具（如 jq）对 UTF-8 BOM 敏感，需要显式生成无 BOM 文件。
- **PowerShell Core 6+ 行为不同**：PowerShell 7+ 中 `Out-File` 和 `Set-Content` 默认编码已改为 UTF-8 无 BOM，`>` 仍受 `$OutputEncoding` 影响，但默认也变为 UTF-8。看环境定策略。

## 可复用建议

1. **入口处统一编码声明**：在脚本或模块顶部写入：

   ```powershell
   [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
   $OutputEncoding = [System.Text.Encoding]::UTF8
   $PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
   ```

2. **API 调用获取原始流后再解码**：当服务器 Content-Type 不可信时，强制按 UTF-8 解码。
3. **文件写入一律显式 `-Encoding utf8`**，不要依赖默认值。
4. **测试字符**：用“我爱你𠮷”这样的字符验证，包含 BMP 外字符（如“𠮷”），能暴露更多编码缺陷。
5. **跨进程传递 JSON 使用 Base64 避险**：如果管道编码实在难统一，可以先 Base64 编码传输，接收端解码，彻底规避二进制流编码问题。

## 总结

Windows PowerShell 处理中文 JSON 的混乱，本质是多个环节编码假设不一致导致：
- 网络字节流 → .NET 字符串（自动解码，可能出错）
- .NET 字符串 → 控制台/文件（OutputEncoding / 文件编码不匹配）
- 文件编码 → 下游工具（UTF-16 vs UTF-8，BOM）

把每个环节的编码显式设置一致，问题就消失了。尤其是对 OpenClaw、Agent 插件这类数据要经过多个进程交接的场景，编码的鲁棒性直接影响自动化流水线的可靠性。不要再让中文“打坏”成为半夜排查的元凶。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/da32c4c820d7437d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/21cb310564e9c222.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/1e9337de812820cc.png)

