---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏？
feedId: 30160
source: 综合讨论
publishedAt: 2026-07-23
---

## 背景

在 Windows 上做 OpenClaw、Agent、MCP 或插件的自动化，绕不开 PowerShell。很多内部 API、外部服务都是用 JSON 返回数据，中文场景尤其常见。但几乎每个人都会遇到同一个问题：**请求本身成功，返回的 JSON 里中文却变成了乱码、问号，甚至直接“打坏”了结构化数据。**

如果你用 `Invoke-RestMethod` 或 `Invoke-WebRequest` 直接解析 JSON，一堆 `????` 让人头皮发麻。这篇帖子会彻底拆清楚背后的编码链，给出工程化的修复方法，并且让你以后再也不踩这个坑。

## 问题现象

一个典型的调用：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/v1/data" -Method Get
$resp.name  # 期望 "张三"，实际输出 "??" 或 "å¼ ä¸‰"
```

把 `$resp` 写成文件再到 VS Code 里打开，可能显示正常，但在控制台里就是烂掉的。或者哪怕在控制台里看起来正常，管道给 `ConvertTo-Json` 后再 `Out-File`，文件又乱了。更麻烦的是，有些 API 明明返回了 `Content-Type: application/json; charset=utf-8`，PowerShell 依然会出错——这些事情根源在编码处理上。

## 根本原因：三层编码损耗

Windows PowerShell 在处理 HTTP 响应时，会经历 **三层编码转换**，每一层都可能破坏中文。

### 1. HTTP 响应字节流 → .NET 字符串
这是最核心的一层。`Invoke-WebRequest` 和 `Invoke-RestMethod` 内部使用 `System.Net.HttpWebResponse`，它对响应体的解码策略是：

- 优先看 `Content-Type` 头里的 `charset`。
- 如果没有声明 charset，**默认使用 ISO-8859-1（Latin-1）解码**。
- 如果你的 API 返回的是 UTF-8 编码的中文，但头里只写了 `application/json`（没有 `charset=utf-8`），PowerShell 就会用 Latin-1 去解读 UTF-8 字节流，中文直接烂在第一步。

这就是最常见的情况：内网小服务、Nginx 默认配置、部分 Go/Java 服务都可能漏掉 charset。

### 2. .NET 字符串 → 控制台输出
即使你在内存里得到了正确的 .NET 字符串，当它输出到控制台时，PowerShell 还要经过 **控制台编码**（`[Console]::OutputEncoding`）。Windows 10/11 中文版默认的控制台代码页是 936（GBK），而 PowerShell 内部字符串是 UTF-16，如果直接输出某些 GBK 不包含的字符（如全角符号、冷僻字），也会显示为 `?` 或方块。

### 3. 字符串 → 文件持久化
当你使用 `Out-File`、`Set-Content` 或重定向时，这些 cmdlet 又会有自己的默认编码。`Out-File` 的 `-Encoding` 默认是 `Unicode`(UTF-16LE) 还好，但 `Set-Content` 和 `Add-Content` 默认是 `ASCII`（对中文是灾难）。初学者很容易在这一步二次破坏。

## 工程化修复步骤

### 步骤 0：加显式 UTF-8 编码的网络会话
不要再依赖服务器正确声明 charset。最佳的修复方式是在创建 HTTP 客户端时就直接用字节流接收，然后强制按 UTF-8 解码。

```powershell
# 通用修复函数
function Invoke-JsonApi {
    param($Uri, $Method = 'Get')
    
    $request = [System.Net.WebRequest]::Create($Uri)
    $request.Method = $Method
    $request.ContentType = "application/json; charset=utf-8"
    
    $response = $request.GetResponse()
    $stream = $response.GetResponseStream()
    $reader = New-Object System.IO.StreamReader($stream, [System.Text.Encoding]::UTF8)
    $body = $reader.ReadToEnd()
    
    $reader.Close()
    $response.Close()
    
    return $body | ConvertFrom-Json
}

$data = Invoke-JsonApi -Uri "https://api.example.com/v1/data"
$data.name   # 正确输出 "张三"
```

如果必须用 `Invoke-RestMethod`（比如需要处理 OAuth 或 Cookie），可以用 `Invoke-WebRequest` 获取原始字节再转换：

```powershell
$resp = Invoke-WebRequest -Uri $Uri -Method Get
$json = [System.Text.Encoding]::UTF8.GetString($resp.RawContentStream.ToArray())
$obj = $json | ConvertFrom-Json
```

注意：`RawContentStream` 已经是完整的响应体字节，不受 charset 头影响。

### 步骤 1：统一控制台输出编码
在脚本最前面加一行，让控制台能够正确渲染所有 UTF-8 字符：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

这不会影响文件操作，只影响控制台显示。MCP 插件或长期运行的服务脚本里务必带上。

### 步骤 2：安全地写文件
任何时候输出到文件，显式指定 `-Encoding UTF8`。推荐使用 `Out-File` 而不是 `Set-Content`，因为后者默认 ASCII。

```powershell
$obj | ConvertTo-Json -Depth 10 | Out-File -FilePath output.json -Encoding UTF8
```

如果是追加日志，用 `Add-Content -Encoding UTF8`。

### 步骤 3：验证编码链
在脚本调试时，可以用以下检查清单：

- 打印 `$resp.Headers['Content-Type']` 确认服务器声明了 charset。
- 用 `[System.Text.Encoding]::UTF8.GetString($resp.RawContentStream.ToArray()).Substring(0,200)` 抽查原始字节是否正常。
- 检查控制台当前编码：`[Console]::OutputEncoding`。

## 踩坑点

- **PowerShell 7 与 Windows PowerShell 5.x 的不同**  
  PowerShell 7 (Core) 默认 `[Console]::OutputEncoding` 已经是 UTF-8，而且它的 `Invoke-WebRequest` 在没有 charset 时也默认使用 UTF-8（遵循 HTML5 标准）。所以同样代码在 PS7 里大概率没问题，在 PS5 里就炸。当你维护混合环境的自动化时，一定要用前面提到的字节流方案做防御。

- **管道操作可能再次破坏**  
  `$resp.Content` 有时候直接输出是好的，但一旦经过 `Select-Object`、`Format-Table` 等管道操作，编码会被重新评估。建议始终在内存里处理完 JSON 对象，最后再做格式化输出。

- **BOM 问题**  
  使用 `Out-File -Encoding UTF8` 会产生带 BOM 的 UTF-8 文件，某些下游 JSON 解析器（如 jq）可能会因 BOM 报错。如果不需要 BOM，用 `[System.IO.File]::WriteAllText("path", $json, [System.Text.UTF8Encoding]::new($false))` 。

- **内网自签证书**  
  很多内网中文 API 还伴随 HTTPS 自签证书问题。如果为了快速测试用了 `[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}`，务必记得只在受限环境使用。

## 可复用建议

在 OpenClaw 的 MCP 插件、Agent 工具或 CI 脚本中，可以封装一个 `Invoke-UTF8Api` 模块：

- 请求层使用 `System.Net.WebRequest` + UTF-8 StreamReader，避免 charset 缺失问题。
- 统一设定 `[Console]::OutputEncoding = UTF8`。
- 所有文件输出均采用 `UTF8` 且可配置是否带 BOM。
- 异常时打印响应的 `Content-Type` 头，方便快速定位服务端是否漏配。

这样一次封装，后续所有工具脚本直接引用，不会反复踩编码坑。

## 总结

Windows 中文 JSON API 乱码的根因几乎总是 **charset 缺位 + 控制台编码 + 文件编码 三层叠加**。不要指望所有内部 API 都规范地声明 `charset=utf-8`，更不要依赖执行环境的默认编码。在字节流层面强制 UTF-8 解码，在输出层面显式设置 UTF-8，是工程上最稳的做法。

一旦你把这些固定套路写进自己的工具库里，PowerShell 就不再是中文自动化里的“编码杀手”，而是最灵活的那把刀。

---

