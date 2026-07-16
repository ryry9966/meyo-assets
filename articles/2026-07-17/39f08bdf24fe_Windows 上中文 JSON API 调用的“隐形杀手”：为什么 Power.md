---
title: Windows 上中文 JSON API 调用的“隐形杀手”：为什么 PowerShell 总会把中文打坏
feedId: 29357
source: 综合讨论
publishedAt: 2026-07-17
---

## 背景

在 Windows 上通过 PowerShell 调用第三方中文 API、构建 MCP 工具或自动化插件时，经常会遇到一个令人抓狂的问题：API 返回的 JSON 明明在浏览器或 Postman 里完全正常，一到 PowerShell 里 `Invoke-RestMethod` 或 `Invoke-WebRequest` 拿出来就变成了一堆 `????` 或者看不懂的 Unicode 转义。即使你在代码里小心翼翼地指定了 `Content-Type: application/json; charset=utf-8`，问题依旧。

这不是 API 服务端的锅，也不是网络传输过程丢失了字节，而是 Windows 下 PowerShell 宿主与 .NET 运行时的默认编码机制“协作”出来的一种经典字符损坏案例。本文从 OpenClaw 工程师的实际踩坑出发，还原问题根因，并给出稳重、可复用的解决方案。

## 问题现场

假设你要调用一个中文分词 API，返回格式如下：

```json
{"word":"你好世界","pos":"ns"}
```

用 PowerShell 最直观的写法：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/tokenize" -Method Post -Body $jsonPayload -ContentType "application/json; charset=utf-8"
Write-Host $response.word
```

结果控制台输出 `????` 或者 `ä½ å¥½ä¸�ç`?。

更隐蔽的情况是：你用 `Out-File` 或 `>>` 重定向到文件，然后打开文件发现中文完好，但屏幕上就是乱码；或者 `ConvertTo-Json` 后再 `echo` 到命令行，中文变成了 `\uXXXX` 转义序列。各种不一致现象让排查方向容易跑偏。

## 根因分析

问题的核心在于 **PowerShell 控制台的输出编码**与**数据在内存中的实际编码**不匹配。拆开来看，主要有三层错位：

1. **Web cmdlet 的行为**  
   `Invoke-RestMethod` 和 `Invoke-WebRequest` 底层使用 `System.Net.Http.HttpClient`，它能正确解析服务器返回的 UTF-8 字节流并反序列化为 .NET 字符串。在内存中，这些字符串是 Unicode，完全正确。所以问题不在这里。

2. **PowerShell 的输出编码**  
   当你在控制台或 ISE 中用 `Write-Host`、直接输出对象、或进行管道操作时，PowerShell 会将 .NET 字符串转换为控制台可显示的字节流。这个转换使用的编码由 `[Console]::OutputEncoding` 决定。在中文版 Windows 中，该属性默认通常是 **GBK (代码页 936)**。于是，Unicode 字符被按 GBK 编码转换，GBK 无法覆盖所有字符时就变成问号，或者在字体不支持时显示为方块。

3. **JSON 序列化器的默认转义**  
   如果使用 `ConvertTo-Json` 把对象转成 JSON 字符串，PowerShell 5.1 默认会使用 `\uXXXX` 转义非 ASCII 字符，导致中文变成 Unicode 转义序列，这在某些日志或管道传递时看起来像“打坏了”，但其实是合规的 JSON。后期如果不处理，写入文件或传给下一个 API 时可能正常，但人眼查看就很不友好。

另外，当使用重定向运算符 `>` 或 `Out-File` 时，文件的编码默认是 UTF-16 LE（PowerShell 5.1）或 ASCII（某些版本），与预期 UTF-8 不一致，进一步加剧混乱。

## 稳定修复步骤

### 1. 对齐控制台输出编码
在脚本开头强制设定控制台输出编码为 UTF-8：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

设置后，`Write-Host` 或直接输出字符串便可以正确显示中文。如果使用 Windows Terminal，这一设置会立即生效；传统 conhost 可能需要配合字体调整，但编码层面已正确。

### 2. 使用 `-Encoding` 参数读写文件
调用 `Invoke-RestMethod` 后如需保存到文件，显式指定 UTF-8：

```powershell
$response | ConvertTo-Json -Depth 3 | Out-File -FilePath "result.json" -Encoding utf8
```

或者用 `Set-Content` 配合 `-Encoding utf8`。注意 `Out-File` 的默认编码是 UTF-16 LE，不指定就会在文件头生成 BOM，影响其他工具读取。

### 3. 避免 JSON 序列化时中文被转义
在 Windows PowerShell 5.1 中，`ConvertTo-Json` 没有直接参数控制转义行为，但可以通过一个简单函数覆盖序列化器：

```powershell
function ConvertTo-JsonUtf8NoEscape($object) {
    $json = $object | ConvertTo-Json -Depth 10
    # 将 \uXXXX 还原回文字符
    [System.Text.RegularExpressions.Regex]::Replace($json, "\\u([0-9a-fA-F]{4})", {
        param($m)
        [char][int]::Parse($m.Groups[1].Value, [System.Globalization.NumberStyles]::HexNumber)
    })
}
```

在 PowerShell Core (7+) 中，可使用 `-EscapeHandling EscapeNonAscii` 参数更优雅地控制。

### 4. 底层保底方案——直接操作字节流
如果上述设置后仍然偶发问题，可以绕过 PowerShell 的自动编码转换，直接使用 .NET 的 `HttpClient` 并操作 UTF-8 字节流：

```powershell
$client = [System.Net.Http.HttpClient]::new()
$response = $client.GetByteArrayAsync($url).Result
$jsonString = [System.Text.Encoding]::UTF8.GetString($response)
```

这样可以保证从网络字节到字符串的转换无任何中间层干扰，适合对可靠性要求极高的 MCP 数据管道。

## 踩坑实录

- **坑1：改变 `[Console]::OutputEncoding` 后 VS Code 内置终端仍乱码**  
   VS Code 的终端可能覆盖了编码设置，需要在 VS Code 的 `settings.json` 中设定 `"terminal.integrated.shellArgs.windows"` 或直接在终端内部重新执行一次编码设置命令。

- **坑2：`Invoke-WebRequest` 获取的 `.Content` 属性乱码**  
   `Invoke-WebRequest` 返回的 `.Content` 是已解码的字符串，若服务器未在响应头中声明 charset，.NET 会尝试自动检测，可能误判为 ISO-8859-1。这时需要手动获取原始字节并解码：

   ```powershell
   $bytes = (Invoke-WebRequest -Uri $url).RawContentStream.ToArray()
   $text = [System.Text.Encoding]::UTF8.GetString($bytes)
   ```

- **坑3：`-ContentType` 参数不会强制 PowerShell 按 UTF-8 解释响应**  
   此参数只影响请求头的 Content-Type，不影响响应解码方式。响应解码依赖服务器响应头或 BOM。所以即使写了 `charset=utf-8`，仍需关注响应端。

## 可复用建议

1. **模版化你的 HTTP 调用**：将 Invoke-RestMethod 封装成一个内部函数 `Invoke-Api`，在函数内部统一设置 `[Console]::OutputEncoding` 和 UTF-8 解码细节。
2. **所有文件写入操作带上 `-Encoding utf8`**：形成肌肉记忆，避免默认行为。
3. **在 OpenClaw 插件输出前做一次编码归一化**：无论是返回给 Agent 的文本还是写入 MCP 结果集，都显式使用 `Encoding.UTF8.GetBytes()` 然后转成 Base64 或在 JSON 中安全传输。
4. **日志不要只在控制台看**：将返回内容先写入 UTF-8 日志文件，再通过 `Get-Content -Encoding utf8` 读取显示，可快速定位是显示问题还是数据问题。

## 总结

PowerShell 把中文“打坏”的根源不是坏在数据上，而是坏在 Windows 控制台的旧时代编码遗产与现代化工具间的错配。作为自动化实践者，我们需要做的是理解 .NET 字符串是干净的，但通往屏幕和文件的路径上每一步都可能发生编码转换，必须主动接管这些转换，强制使用 UTF-8。遵循“统一输出编码 + 显式文件编码 + 底层字节流兜底”三原则，以后无论在 OpenClaw 的哪个场景下调用中文 JSON API，都能稳稳地拿到可读、可传递的正确结果。

---

