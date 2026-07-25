---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏，怎么稳住它
feedId: 30419
source: 综合讨论
publishedAt: 2026-07-25
---

## 背景

在 Windows 上用 PowerShell 做自动化——无论是给 OpenClaw 写插件、搭 MCP 服务桥接、还是用 Agent 调内部 API，都绕不开一件事：请求返回 JSON，里面夹着中文。正常来说，服务器返回 `{"msg":"任务已创建"}`，你期望在 PowerShell 里看到一模一样的汉字。但现实是，控制台印出一堆 `???` 或 `ä½ å¥½`，写到文件后被下游工具当乱码丢掉。

这个问题在中文 Windows 环境里实在太容易踩到，根因不在 PowerShell “烂”，而在于你面对的不止一款编码，而是一整条编码链路。下面我用一次真实的磁盘清理脚本开发经历，把这条链路拆开，给出能复用的工程化做法。

## 问题链路：到底是谁打坏了中文

假设你要调用公司内部的审批接口，返回一段 JSON，包含申请人的中文姓名。

用最自然而然的写法：

```powershell
$resp = Invoke-RestMethod -Uri "http://api.internal/approval?id=42"
$resp.name
```

在某些机器上，`$resp.name` 输出正常；另一些机器上却是乱码。同一个脚本，为什么时好时坏？因为链路里有三个容易分开失控的点：

1. **服务器响应的 Content-Type 没有 charset**  
   如果服务器只返回 `Content-Type: application/json`，没有 `charset=utf-8`，PowerShell 的 `Invoke-RestMethod` 会尝试自动检测编码，在中文 Windows 上很可能退回 `ISO-8859-1`，中文直接打成单字节乱码。

2. **控制台编码和 PowerShell 内部编码不一致**  
   PowerShell 字符串是 UTF-16 的，但控制台窗口用的代码页可能是 `936`（GBK）。当对象输出到控制台时，PowerShell 要把 UTF-16 转成 GBK。如果字符串里含有 GBK 无法表示的字符，或者转换过程不干净，就会出现 ? 或空白。

3. **文件写入的默认编码不是 UTF-8**  
   习惯性地用 `$resp | Out-File result.json` 或重定向 `>`，得到的文件实际上是 UTF-16 LE 带 BOM。后端服务、Python 脚本、jq 命令如果不认识这个编码，直接读成乱码。很多人查了半天发现是文件编码的坑。

踩坑点的本质：在 Windows 上，PowerShell 5.1 和 PowerShell 7 行为有差异，`$OutputEncoding` 变量的作用范围很容易被忽略，`> ` 重定向符走的是 `Out-File` 的默认参数，而默认编码跟着 `$PSDefaultParameterValues` 不一定是你想要的 UTF-8。

## 从乱码到干净的步骤

下面是一套管用的做法，覆盖请求、控制台、文件三个环节。

### 1. 先让控制台稳定显示中文

脚本开头直接设三样东西，避免环境差异：

```powershell
# 控制台输出编码转成 UTF-8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
# 管道、重定向等交给外部程序时的编码
$OutputEncoding = [System.Text.Encoding]::UTF8
# 强制 Out-File / Set-Content 等 cmdlet 默认用 UTF-8
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

这三行能消灭一半的显示乱码。尤其 `$OutputEncoding` 会影响 PowerShell 把字符串交给外部命令（比如 `curl.exe`、`python`）时的编码方式，很多人只改控制台却忘了它。

### 2. 确保 API 响应能正确解码

最稳妥的办法是不依赖自动编码检测，直接拿原始字节流，自己用 UTF-8 解码。

```powershell
$response = Invoke-WebRequest -Uri "http://api.internal/approval?id=42" -UseBasicParsing
$rawStream = $response.RawContentStream
$reader = [System.IO.StreamReader]::new($rawStream, [System.Text.Encoding]::UTF8)
$bodyString = $reader.ReadToEnd()
$reader.Close()
$obj = $bodyString | ConvertFrom-Json
```

`Invoke-WebRequest` 的 `RawContentStream` 是网络流原始字节，你可以精确控制解码。如果嫌弃每次写这么多，可以封装成一个函数：

```powershell
function Invoke-Utf8RestMethod {
    param([string]$Uri)
    $resp = Invoke-WebRequest -Uri $Uri -UseBasicParsing
    $utf8 = [System.Text.Encoding]::UTF8
    $bodyStr = $utf8.GetString($resp.RawContentStream.ToArray())
    return $bodyStr | ConvertFrom-Json
}
```

如果你想继续用 `Invoke-RestMethod`，那就必须要求后端 API 在响应头里带上 `charset=utf-8`，否则就把锅甩给服务端。工程里我通常同时做两件事：API 强制声明 UTF-8，客户端侧加防线。

### 3. 安全地写文件

永远避免裸用 `Out-File` 和 `>`。

推荐写法：

```powershell
$obj | ConvertTo-Json -Depth 10 | Set-Content -Path result.json -Encoding UTF8
```

`Set-Content -Encoding UTF8` 会生成 **无 BOM 的 UTF-8** 文件，几乎所有现代工具都能正确识别。如果你必须保留 BOM（比如某些古董 Windows 程序），使用 `-Encoding UTF8BOM`。

## 踩坑实录：一个难以复现的间歇性乱码

有一次我写一个 MCP 工具，在 Windows 11 上返回给 Agent 的中文一直正常，但部署到一台 Windows Server 2019 上后，Agent 收到的 JSON 里的中文变成了类似 `ç ³è¯` 的字符。

排查过程：
- 在 Server 上用 `curl.exe` 重放请求，结果正常，排除服务端问题。
- 在 PowerShell 中用 `$resp.Content` 看，控制台正常，说明 `Invoke-WebRequest` 的解码没完全坏。
- 但一旦通过 `ConvertTo-Json` 转成 JSON 字符串，再用 `Out-File` 保存，文件编码变成 UTF-16 LE，最终传到 Agent 的 Python 环境直接解码失败。

修复就是在文件写入环节加了 `-Encoding UTF8`，同时在脚本顶部设置了那三行全局变量。后来我把这个检查写进团队的基础效能模板，再没踩过同类坑。

## 可复用建议

- **脚本模板固化编码设置**：所有涉及中文输出的 PowerShell 脚本，顶部粘贴那三行编码变量，成本极低。
- **API 契约带上 charset**：无论是自己写服务还是要求下游，强制 `Content-Type: application/json; charset=utf-8`，省去无穷的自动检测烦恼。
- **文件只用 Set-Content -Encoding UTF8**：忘掉 `Out-File` 和重定向 `>`，除非你清楚知道它们在该环境下的默认编码。
- **跨平台项目加 UTF-8 BOM 检查**：如果你的 PowerShell 脚本会被 Git 管理、被 CI 运行，可以在 CI 里加一步 `powershell -Command "if ((Get-Content script.ps1 -Encoding Byte -TotalCount 3) -join ',' -ne '239,187,191') { throw '脚本未以UTF-8 BOM保存' }"`，避免保存时的编码变异。
- **MCP/Agent 开发者特别注意**：Agent 与工具之间的数据交换往往经过多层序列化，中文经一次编码错误可能变成永久损坏。工具返回的 JSON 字符串，最好用明确 UTF-8 的方式构造，而不是依赖 PowerShell 的隐式转换。

## 总结

Windows 中文 JSON API 调用里，“中文被打坏”几乎从来不是 Windows 本身的锅，而是编码意识没跟上链路。把控制台编码、管道输出编码、文件写入编码、响应解码编成几个固定的防御动作，就能把 PowerShell 从“乱码制造机”变成可靠的中文处理工具。在 Agent 和自动化场景里，这些防御动作写得越早，后面省下的排障时间越多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/8285f1d700d23730.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/12680cc0a68b8858.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/a93621efc8f4f592.png)

