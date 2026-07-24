---
title: PowerShell 调用中文 JSON API 的编码陷阱：为什么 Invoke-RestMethod 把你的中文打得面目全非
feedId: 30291
source: 综合讨论
publishedAt: 2026-07-24
---

## 背景

在做 OpenClaw 插件的自动化测试时，我们需要在 Windows 上通过 PowerShell 调用一个返回中文 JSON 的 API。看起来是标准操作：`Invoke-RestMethod` 或 `Invoke-WebRequest` 拿到响应，然后解析。但实际得到的中文经常变成 `æµè¯` 之类的乱码，或者直接被替换成 `????`，导致下游的 Agent 逻辑完全中断。

更糟糕的是，这种问题在 PowerShell 5.1（Windows 自带）和 PowerShell 7 中表现不同，且很多时候只在脚本文件里出现，在交互式控制台中反而正常。这对于需要稳定运行在计划任务、CI/CD 或 OpenClaw 调度器里的自动化流程来说，是一个典型的“环境敏感型”坑。

本文只聚焦 Windows 平台，因为 macOS/Linux 的 PowerShell 默认使用 UTF-8，编码冲突较少。

## 问题根因

整个调用链路可能发生三次编码损伤：

1. **请求体编码**：如果 API 是 POST 且带中文 `Content-Type: application/json; charset=utf-8`，但 `Invoke-RestMethod -Body` 在将 .NET 字符串转换成字节流时，可能被 PowerShell 的 `$OutputEncoding` 绑架。
2. **响应解码**：服务端正确返回了 UTF-8 字节，但 `Invoke-RestMethod` 会按照 HTTP 响应头的 `charset` 来解码。如果 API 实现不规范，缺少 charset，PowerShell 会退到 `[System.Text.Encoding]::Default`，即系统活动代码页（中文 Windows 上是 GBK/CP936），导致 UTF-8 被错误解释。
3. **控制台输出编码**：即使内存中的字符串已经正确，输出到文件或管道时，`Out-File`、`>` 重定向使用 `$OutputEncoding`（在 PS5.1 中默认为 ASCII!），再次损坏汉字。

典型症状：在 ISE 或 VS Code 终端里看到正确中文，但 `>.log` 文件乱码；或者直接 `Invoke-RestMethod` 返回的 PSObject 属性中文就是坏的。

## 复现环境与简化示例

假设我们有一个 API `https://api.example.com/greet`，返回：
```json
{"message": "你好，世界！"}
```

最简单的调用：
```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/greet" -Method Get
$response.message  # 乱码: "?????" 或 "ä½ å¥½ï¼Œä¸–ç•Œï¼"
```

检查变量：
```powershell
[System.Text.Encoding]::Default.EncodingName  # 中文系统: "Chinese Simplified (GB2312)"
$OutputEncoding  # PS5.1: ASCIIEncoding; PS7: UTF8Encoding
```

## 做法与步骤：可靠调用中文 JSON API

下面的步骤以 PowerShell 7+ 为主，但也会给出 PS5.1 的兼容写法。我们的目标是在任何执行上下文（脚本文件、计划任务）都能正确获取中文。

**1. 显式指定请求和响应的编码**

最直接的办法：放弃 `Invoke-RestMethod` 的自动解码，用 `Invoke-WebRequest` 拿到原始字节，自己按 UTF-8 解码。

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/greet" -Method Get -ContentType "application/json; charset=utf-8"
$rawBytes = $response.RawContentStream.ToArray()  # 获取原始字节（PS7+ 可用）
# 如果 PS5.1，用 $response.Content 可能已经被错误解码，所以需要操作 RawContentStream
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
$jsonString = $utf8NoBom.GetString($rawBytes)
$obj = $jsonString | ConvertFrom-Json
$obj.message  # 正确输出
```

如果 API 确实在响应头里包含 `charset=utf-8`，且你用的是 PS7，实际上 `Invoke-RestMethod` 会遵守该头。那么只需强制脚本文件自身保存为 UTF-8 with BOM（PS5.1 识别 BOM）或 UTF-8 without BOM（PS7 默认）。编辑器另存为 UTF-8 with BOM 往往能解决很多脚本编码派生问题。

**2. 统一会话编码环境**

在脚本顶部强制设置编码相关变量：

```powershell
# PS7 推荐
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
```

对于 PS5.1，`$PSDefaultParameterValues` 对新版 cmdlet 有限，所以主要改 `$OutputEncoding`：
```powershell
$OutputEncoding = New-Object System.Text.UTF8Encoding $false
[Console]::OutputEncoding = New-Object System.Text.UTF8Encoding $false
```

但要注意，`[Console]::OutputEncoding` 只会影响控制台宿主，对于后台作业（如计划任务）无控制台，这条无效。后台作业主要依赖 `$OutputEncoding` 和 cmdlet 自身的 `-Encoding` 参数。

**3. 保存结果到文件时的正确姿势**

绝对避免 `$result > file.txt` 或 `Out-File` 不带 `-Encoding`。使用：

```powershell
$result | Out-File -FilePath "output.json" -Encoding utf8NoBOM
# 或直接 Set-Content -Path "output.json" -Value $result -Encoding UTF8NoBOM
```

PS5.1 的 `Out-File` 不支持 `utf8NoBOM`，只能用 `UTF8`（带 BOM）或 `-Encoding utf8`。对于 JSON，BOM 可能导致部分解析器报错，所以如果目标读取方对 BOM 敏感，用 .NET 方法写入：

```powershell
[System.IO.File]::WriteAllText("output.json", $result, (New-Object System.Text.UTF8Encoding $false))
```

**4. POST 中文请求体的安全构造**

当你需要发送中文 JSON 时，不要依赖字符串插值后让 PowerShell 自动转换。构造一个哈希表，转成 JSON，然后用 `-Body` 传入字节数组或确保编码正确。

```powershell
$body = @{
    name = "张三"
    query = "中文搜索"
} | ConvertTo-Json

# 安全方法：显式获取 UTF-8 字节
$bodyBytes = [System.Text.Encoding]::UTF8.GetBytes($body)
Invoke-RestMethod -Uri "..." -Method Post -Body $bodyBytes -ContentType "application/json; charset=utf-8"
```

或者使用 PS7 的 `-Body` 直接传对象配合 `-ContentType`，它会自动序列化并遵守 UTF-8，但为了跨版本安全，字节数组是最稳定的。

## 踩坑点

- **ISE 的障眼法**：Windows PowerShell ISE 的交互窗口使用了不同的输出编码，可能显示正确，但脚本保存为 ANSI 或无 BOM 的 UTF-8，运行时字符串字面量本身就是坏的。永远以实际运行环境为准。
- **`-ContentType` 不影响响应解码**：很多人以为请求时带上 `charset` 能让响应也按这个来，实际上服务器决定的 charset 头才是标准。若无 charset，就得自己截获字节流。
- **Git 换行符和编码转换**：如果脚本通过 Git 在 Linux 和 Windows 之间同步，可能被 `core.autocrlf` 改变编码，导致 BOM 丢失或字符破坏。应在 `.gitattributes` 中声明 `*.ps1 text eol=crlf working-tree-encoding=UTF-8`（针对 PS5.1 可设 UTF-8-BOM）。
- **计划任务跑在 SYSTEM 账户下**：此时活动代码页可能不同，`[System.Text.Encoding]::Default` 不再是 GBK，可能引发其他解码问题。务必将所有编码都显式指定为 UTF-8。

## 可复用建议

1. **建立统一的 Bootstrap 模块**：在插件入口脚本最前面 `dot-source` 一个 `encoding-init.ps1`，内容就是上面的 `$OutputEncoding` 和 `$PSDefaultParameterValues` 设置，以及覆盖 `Out-File` 的代理函数确保默认 UTF-8。
2. **使用 `Invoke-WebRequest` + 手动解码模式封装函数**：比如写 `Invoke-Utf8Api`，内部一律处理 `RawContentStream`，供所有 OpenClaw 插件共用。
3. **CI 上做编码回归检查**：用一个简单的中文 API 调用测试，验证输出文件的十六进制是否符合 UTF-8 期望，防止环境变更导致回归。
4. **转向 PowerShell 7**：PS7 的默认编码更接近现代期望，`$OutputEncoding` 默认为 UTF-8，很多隐式问题自然消失。如果必须留在 PS5.1，就务必将上述防御式编码规则固化为团队规范。

## 总结

Windows 上 PowerShell 处理中文 JSON API 的核心矛盾，是遗留的 ANSI/系统活动代码页与现代 UTF-8 万事之间的冲突。自动化流程不能依赖“看起来正常”的交互式环境，必须显式地在请求、响应、输出三个环节强制使用 UTF-8。通过掌握字节数组手动解码、全局编码变量设置、文件 I/O 的 Encoding 参数，可以彻底消灭随机乱码，让 OpenClaw 插件在 Windows 上也稳定输出正确的中文数据。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/bc22ab792d126708.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/1897dc4d84729961.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/caa8ddc568048c33.png)

