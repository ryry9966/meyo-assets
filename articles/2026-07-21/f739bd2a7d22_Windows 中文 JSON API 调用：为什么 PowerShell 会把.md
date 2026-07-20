---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 29861
source: 综合讨论
publishedAt: 2026-07-21
---

## 背景

在 OpenClaw、Agent 或 MCP 这类自动化实践中，Windows 依然是很多本地工具链的运行环境。无论是通过本地 HTTP API 调用 Ollama、向企业微信发消息，还是用 PowerShell 封装一个 MCP 插件，都绕不开一件事：构造包含中文的 JSON 请求体，通过 `Invoke-RestMethod` 或 `Invoke-WebRequest` 发送出去。

初看很简单，写个 `ConvertTo-Json`，塞进 `-Body` 参数，发过去就完事。但实际执行时你会发现，对方收到的中文经常变成 `????` 或 `®æ–‡` 之类的乱码，API 直接返回 400 或者把数据写成了不可逆的垃圾。更难受的是，同样的脚本在同事的 macOS 上跑得好好的，到了你的 Windows 机器上就立刻打坏中文，排查起来往往会先从网络、API 版本甚至 JSON 格式上绕一大圈。

这篇文章不讨论 PowerShell 该不该用，只从一个真实工程视角复盘：**为什么 PowerShell 会把中文打坏，以及怎么一次性根除这个问题。**

## 问题到底出在哪里

核心矛盾在于 PowerShell 对“字符串”和“编码”的处理和大多数 HTTP API 的期望不一致。

现代 Web API 几乎都约定请求体使用 UTF-8 编码，并在 `Content-Type` 中标明 `charset=utf-8`。但 PowerShell 内部对字符串的编码行为依赖以下三个容易被忽略的地方：

1. **字符串的默认编码**  
   PowerShell 5.1（Windows 自带的版本）中，`ConvertTo-Json` 生成的实际上是 .NET 的 `String`，它是 UTF-16 内存表示。如果你把这个字符串直接赋给 `-Body`，PowerShell 默认会先转成 `ASCII` 或 `Windows-1252`（Latin1）再发送，导致所有非 ASCII 字符丢失或变乱码。

2. **`$OutputEncoding` 变量**  
   当你不使用 `-Body` 而用管道将字符串输入 `Invoke-WebRequest`（例如配合 `curl` 用法）时，PowerShell 会用 `$OutputEncoding` 来决定如何把字符串转成字节流。这个变量的默认值通常是 `ASCII` 或 `Windows-1252`，而不是 UTF-8。

3. **文件写入的编码陷阱**  
   很多自动化脚本会先把 JSON 写入临时文件，再用 `-InFile` 发送，或者通过 `Out-File` / `Set-Content` 保存日志。`Out-File -Encoding utf8` 会在文件开头加上 BOM(Byte Order Mark)，有些 API 解析器不认识 BOM，直接报错；`Set-Content -Encoding utf8` 在 PS 5.1 下会生成带 BOM 的 UTF-8，而在 PowerShell Core 中默认是无 BOM 的 UTF-8。这些不一致性在混用不同 PowerShell 版本的 Agent 环境里特别容易出问题。

## 复现场景

下面是一个最典型的翻车案例：

你要调用本地 Ollama 的 `/api/chat` 接口，发送一段包含中文的 system prompt。脚本长这样（错误示范）：

```powershell
$body = @{
    model = "llama3.1"
    messages = @(
        @{ role = "system"; content = "你是一个智能助手，请用中文回答" }
    )
} | ConvertTo-Json
Invoke-RestMethod -Uri "http://localhost:11434/api/chat" -Method Post -Body $body -ContentType "application/json"
```

执行后，Ollama 端收到的 `content` 可能变成了 `"ä½ æ¯ä¸€ä¸ªæ™ºèƒ½åŠ©æ‰‹ï¼Œè¯·ç"¨ä¸­æ–‡å›žç­”"`。即使加上 `-ContentType "application/json; charset=utf-8"`，结果也不一定对，因为 `-Body` 参数拿到的是 .NET 字符串，PowerShell 仍然会在内部用 `ASCII` 编码先转一次。

## 真正可靠的做法

### 1. 直接传递字节数组（最稳定）

不在任何地方依赖 PowerShell 的自动编码猜测，把 JSON 字符串手动转为 UTF-8 字节数组，然后传给 `-Body`：

```powershell
$jsonString = @{
    model = "llama3.1"
    messages = @(
        @{ role = "system"; content = "你是一个智能助手，请用中文回答" }
    )
} | ConvertTo-Json
$utf8Bytes  = [System.Text.Encoding]::UTF8.GetBytes($jsonString)
Invoke-RestMethod -Uri "http://localhost:11434/api/chat" -Method Post -Body $utf8Bytes -ContentType "application/json; charset=utf-8"
```

这样彻底绕过了 PowerShell 的字符串编码环节，API 一定能收到正确的 UTF-8 字节流。

### 2. 使用 PowerShell Core 7+ 并锁死编码

如果你的环境允许升级，推荐用 PowerShell 7+。它内部对 `Invoke-WebRequest` 和 `Invoke-RestMethod` 的编码处理做了改善，但仍需显式指定 `-ContentType` 并设置 `$PSDefaultParameterValues`：

```powershell
$PSDefaultParameterValues['Invoke-RestMethod:ContentType'] = 'application/json; charset=utf-8'
```

另外在脚本开头设置控制台编码，避免日志输出乱码干扰排查：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

### 3. 处理返回结果中的中文乱码

如果调用成功，但返回的中文在控制台显示为乱码，请检查 `$OutputEncoding` 并同样设置为 UTF-8：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
```

如果脚本需要把返回内容写入文件，务必使用 PowerShell Core 的 `Set-Content -Encoding utf8NoBOM`，或者在 Windows PowerShell 下用 `[System.IO.File]::WriteAllText($path, $content, [System.Text.Encoding]::UTF8)` 避免 BOM。

## 踩坑点总结

- **`ConvertTo-Json` 深度嵌套时的深度参数**  
  默认 `-Depth` 只有 2，复杂对象会被截断，导致 JSON 结构错误。务必将 `-Depth` 设为一个足够大的值，例如 `5`。
- **不要混用 `Out-File` 和管道的默认编码**  
  哪怕只是记录一条 debug 日志，也可能引入 BOM 或 ANSI 编码，污染后续流程。养成统一显式指定编码的习惯。
- **某些 API 不接受带 BOM 的请求体**  
  如果你必须用文件方式发送（比如大负载或 MCP 本地桥接），确保文件是 UTF-8 without BOM，并校验第一字节是否 `0xEF BB BF`。
- **Windows 非英语地区的系统代码页设置**  
  `chcp 936` 的机器上，有些外部进程（如 `curl.exe`）会依赖活动代码页，导致通过 `Start-Process` 调用时出现额外转码。如果一定要调用外部 `curl`，通过环境变量 `$env:LANG` 或直接使用 PowerShell 原生 cmdlet 会更安全。

## 可复用建议

1. **封装一个安全的请求函数**  
   在你的 MCP 工具脚本或 Agent 调用模块中，封装一个 `Invoke-UTF8Api`，内部强制完成 UTF-8 字节数组转换，并统一设置 `ContentType`，避免每次手写重复逻辑。
2. **在脚本开头加入编码保护**  
   ```powershell
   $PSDefaultParameterValues['*:Encoding'] = 'utf8'
   [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
   $OutputEncoding = [System.Text.Encoding]::UTF8
   ```
3. **始终用字节数组做最终发送**  
   这是“铁律”：只要 API 要求 UTF-8，就用 `[System.Text.Encoding]::UTF8.GetBytes()` 转换后传入 `-Body`，绝不依赖字符串自动转换。
4. **统一团队环境到 PowerShell 7**  
   如果团队内部还在混用旧版 Windows PowerShell，中文乱码会成为慢性问题。从 Agent 运行环境的角度，升级到 PS7 能减少一大半编码不一致引起的故障。

## 总结

Windows 上 PowerShell 调用中文 JSON API 之所以“打坏中文”，根本原因是历史遗留下来的字符串编码猜测行为与现代 UTF-8 合约之间的矛盾。靠加一个 `charset=utf-8` 注释或改一下 IDE 字体是解决不了的，必须从工程层面切断所有自动转换路径，明确在每一步编码上自己做主。

当你把 Agent 流程中每一处 HTTP 调用都变成“原始 UTF-8 字节流 + 正确 Content-Type”时，中文乱码问题就会彻底消失。这个原则同样适用于任何需要在 Windows 上做异地字符交换的场景。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/a4c344d6832f43a6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/ba0d9cfa79ae43a2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/115182d2817fa8b3.png)

