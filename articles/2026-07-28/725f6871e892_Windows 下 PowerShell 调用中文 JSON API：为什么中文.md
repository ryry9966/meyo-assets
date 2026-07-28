---
title: Windows 下 PowerShell 调用中文 JSON API：为什么中文总是被“打坏”
feedId: 30812
source: 综合讨论
publishedAt: 2026-07-28
---

## 背景

在给 OpenClaw 写 Agent 或 MCP 插件时，经常需要让一个 PowerShell 脚本去调用外部的 JSON API——可能是内部工具、翻译接口，也可能是某个内部知识库。Windows 环境下很多同学会直接用 `Invoke-RestMethod` 或者 `Invoke-WebRequest` 来发请求、收响应。一旦请求体或响应里包含中文，就容易出现两种经典混乱：

1. 服务端收到的 JSON 字符串中文部分变成 `?`、`锟斤拷` 或者十六进制乱码；
2. 控制台上打印的 API 返回看起来是正常的中文，但写入文件或传给下游就乱码。

这类问题在开发 Agent 或 MCP 工具链时非常要命：本来应该是精确的指令文本，结果变成乱码，模型根本没法正确理解上下文。

## 问题本质：PowerShell 的编码魔法与默认值

很难简单的归咎为"PowerShell 不支持 UTF-8"。相反，它很支持，只不过不同版本、不同宿主、不同重定向方式下，默认行为差别很大。

- **Windows PowerShell 5.x** 的控制台通常使用系统默认代码页（简体中文 Windows 一般是 GBK/936）。对文本进行输出或重定向时，PowerShell 可能把它转换为 OEM 编码或 UTF-16 LE，这取决于输出目标。
- 调用 Web cmdlet 时，`Invoke-RestMethod` 如果不显式指定字符集，它会根据服务器的 `Content-Type` 响应头来解码；但**发送请求**时，如果你直接把中文字符串传给 `-Body`，它在内部编码时出现偏差，常常会变成 ASCII 范围外的字节被破坏。
- 管道、重定向运算符 `>` 以及 `Out-File` 默认使用 `Unicode`（UTF-16 LE）或 ASCII，不是 UTF-8，这会让保存的文件在非 Windows 工具下乱码。

PowerShell 7+ (`pwsh.exe`) 极大改善了这一情况：它默认使用 UTF-8 无 BOM，而且对 HTTP 请求体的编码也更规范。但很多存量脚本、员工电脑或 CI 环境仍然运行在 Windows PowerShell 5.1 上，因此下面的分析和解决方案会兼顾两者。

## 场景还原：一个中文 JSON 请求怎么被"打坏"的

假设你为 MCP 工具写了一个 PowerShell 脚本，调用一个翻译接口：

```powershell
$body = @{
    text = "你好，世界"
    target = "en"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "https://api.example.com/translate" `
                              -Method Post `
                              -Body $body `
                              -ContentType "application/json"
```

如果你在中文 Windows 10 上使用自带的 Windows PowerShell 5.1 运行这段，并且 API 没有做额外的字符集推断，它很大概率会变成 `"text":"?????"` 或者 `"text":"ä½ å¥½"`这种乱码。

**根本原因**：`Invoke-RestMethod` 的 `-Body` 参数在这里接收到的是一个 .NET 字符串 `$body`。当它准备将字符串编码为请求体字节时，如果没有显式提供 `-ContentType` 中的 `charset` 设定，它可能会使用与系统代码页有关的编码（例如 ISO-8859-1 或 Windows-1252）而非 UTF-8。即使写了 `-ContentType "application/json"`，但缺少 `charset=utf-8` 后缀，PowerShell 5.1 仍然会在某些内部路径里使用 `[System.Text.Encoding]::Default`，从而把中文变成乱码。

## 做法/步骤：三种可靠解法

下面是在生产级脚本里经过验证的三种方法，按推荐程度从高到低。

### 方案一：显式指定请求体编码（适用于 PS5.1 和 PS7）

不依赖 `ConvertTo-Json` 之后的字符串直接传递，而是手动将其转为 UTF-8 字节数组，配合 `Invoke-RestMethod` 的 `-Body` 重载。

```powershell
$bodyObject = @{
    text = "你好，世界"
    target = "en"
}
$jsonString = $bodyObject | ConvertTo-Json -Compress
$utf8Bytes  = [System.Text.Encoding]::UTF8.GetBytes($jsonString)

$response = Invoke-RestMethod -Uri "https://api.example.com/translate" `
                              -Method Post `
                              -Body $utf8Bytes `
                              -ContentType "application/json; charset=utf-8"
```

这里的关键是：不给 `-Body` 一个字符串，而是给一个 `[byte[]]`，PowerShell 会原样发送这些字节。同时我们在 `ContentType` 里明确声明了 `charset=utf-8`，服务端也能正确解码。该行为在 PS5.1 和 PS7 下完全一致，非常可靠。

### 方案二：改用 `Invoke-WebRequest` 并精细控制 Header

如果你需要使用流式响应或其他高级特性，可以用 `Invoke-WebRequest` 配合 `-Headers` 和 `-Body` 的字节数组方式，同方案一基本一样。或者直接把 JSON 数据写入请求流：

```powershell
$request = [System.Net.WebRequest]::Create("https://api.example.com/translate")
$request.Method = "POST"
$request.ContentType = "application/json; charset=utf-8"

$bytes = [System.Text.Encoding]::UTF8.GetBytes($jsonString)
$request.ContentLength = $bytes.Length
$stream = $request.GetRequestStream()
$stream.Write($bytes, 0, $bytes.Length)
$stream.Close()
```

这看起来冗长，但对于需要精细控制超时、证书或 Keep-Alive 的场景是一种很稳的做法，且编码完全透明。

### 方案三：只升 PowerShell 7+，但注意输出重定向

如果你可以全面切换到 PowerShell 7，那么最简单的办法就是升级：

- `Invoke-RestMethod -Body $jsonString -ContentType "application/json; charset=utf-8"` 通常会正常工作。
- 但输出到文件时仍然要小心，因为 `>` 重定向在 PS7 中用 UTF-8 无 BOM，而 `Out-File` 默认依然是 UTF-16 LE。因此写脚本时统一养成习惯：`$response | Out-File -FilePath result.json -Encoding utf8`。

## 踩坑点清单

1. **`ConvertTo-Json` 的深度与转义**  
   - 默认 `-Depth` 为 2，复杂对象会被截断，DeepL 等 API 的请求体常会因此不完整。必要时指定 `-Depth 10` 或更大。
   - `ConvertTo-Json` 会把 `<`, `>`, `&` 等字符转义为 Unicode 转义序列，服务端不一定喜欢，可考虑改用 `Newtonsoft.Json` 或.NET 的 `System.Text.Json`。

2. **控制台显示正常但文件乱码**  
   - 这是典型的编码落差。控制台通过宿主渲染可能正确显示了 GBK 字符，但写入文件时用的 UTF-16 或 ANSI。检查 `$OutputEncoding` 和 `[Console]::OutputEncoding`，但最好的习惯是统一使用 `Out-File -Encoding utf8` 或 `Set-Content -Encoding utf8`。

3. **响应正文中文乱码**  
   - 检查 API 响应的 `Content-Type` 是否缺少 `charset` 或声明错误。如果是，可以手动解码：  
     `$raw = Invoke-WebRequest -Uri ...; $text = [System.Text.Encoding]::UTF8.GetString($raw.RawContentStream.ToArray())`
   - 如果服务端返回的是 GBK，那么在 PS5.1 中 `Invoke-RestMethod` 可能自动解码正确，但 PS7 默认当作 UTF-8 就会错，此时需要明确转换。

4. **CI/CD 或远程会话中的编码**  
   - SSH 进入 Windows 时 PowerShell 宿主的编码可能与本地不同，即使脚本里写了 UTF-8 也可能被终端模拟器搅乱。验证环境变量 `$env:LANG` 和 `$OutputEncoding`。如果通过 OpenClaw 远程调用，必须做端到端测试。

## 可复用建议

把你验证过的可靠模式封装成一个脚本函数，这个函数允许你传入任意 PSObject 并安全地调用 JSON API：

```powershell
function Invoke-JsonApi {
    param(
        [string]$Uri,
        [PSObject]$BodyObject,
        [int]$Depth = 10
    )
    $json = $BodyObject | ConvertTo-Json -Depth $Depth -Compress
    $bytes = [Text.Encoding]::UTF8.GetBytes($json)
    Invoke-RestMethod -Uri $Uri -Method Post -Body $bytes -ContentType "application/json; charset=utf-8"
}
```

同时在每个脚本头部统一设置编码偏好（不会完全解决重定向问题，但值得加）：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
```

## 总结

PowerShell 在 Windows 下调用中文 JSON API 的乱码问题，根源在于字符串编码的路径隐式假设与 Web 服务期待的 UTF-8 不一致。解决办法并不复杂：*始终显式将 JSON 字符串转为 UTF-8 字节数组再发送，并在请求和文件输出时明确声明 UTF-8 编码*。当这种模式固化成团队脚本规范后，你会发现中文乱码彻底消失，Agent、MCP 工具链的健壮性也显著提升。如果条件允许，尽量迁移到 PowerShell 7，编码行为更符合现代 Web 开发预期，但仍要保留显式 UTF-8 操作，避免在暗角翻车。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/b3aa0d2f3e131697.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/029aadb8ca8bb8c8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/cf8c0291472e0204.png)

