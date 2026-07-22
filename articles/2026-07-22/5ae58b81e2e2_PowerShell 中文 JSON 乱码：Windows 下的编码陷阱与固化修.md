---
title: PowerShell 中文 JSON 乱码：Windows 下的编码陷阱与固化修复
feedId: 30092
source: 综合讨论
publishedAt: 2026-07-22
---

## 背景：当 Agent 脚本开始说中文

在 OpenClaw‑CN 社区里，用 PowerShell 调用 REST API 已是日常操作。不管是 MCP 插件拉取远端配置，还是作为 Agent 离线工具执行批处理后回传 JSON，中文字段出现的场景越来越多。可一旦中文进了 JSON，Windows 把字符串打坏的问题就接踵而来——有人看到 `æ‰“å°`，有人收到解析异常，也有人发现服务端根本没收到预期字符。这不是玄学，是 Windows 控制台、管道与 HTTP 处理链的三方编码不统一。

## 问题呈现：一个必然翻车的例子

最简情形：用 `Invoke-RestMethod` 请求一个返回中文 JSON 的端点，不加任何特殊处理。

```powershell
$resp = Invoke-RestMethod -Uri "http://localhost:8080/api/echo"
$resp.message
```

如果服务端正确返回 `{"message":"你好"}`，PowerShell 控制台上可能直接显示 `??` 或 `ä½ å¥½`。更隐蔽的是，把 `$resp` 通过管道传给 `ConvertTo-Json` 再写入文件，文件里中文同样是乱码。根源在于：**PowerShell 5.1（Windows 内置）在处理 HTTP 响应时，默认采用系统 ANSI 代码页（中文系统为 CP936 / GBK），而非 UTF-8**。而 JSON 标准要求 UTF-8 编码，这种不匹配会彻底毁掉多字节字符。

`curl.exe`（Windows 10 1803+ 内置）似乎更接近底层，但也逃不过管道陷阱：

```bash
curl.exe -s http://localhost:8080/api/echo | Out-File result.txt
```

默认情况下，`Out-File` 使用 `Unicode`（UTF-16LE）编码，且 `curl` 输出的字节流未经转换直接落盘，中文依旧会损坏。就算在 `cmd.exe` 里能看到中文，一旦流入 PowerShell 的文件管道，编码就被重新解释并写坏。

## 核心机制：为什么 PowerShell 会做这种“翻译”

1. **响应解码**：`Invoke-WebRequest` / `Invoke-RestMethod` 在收到字节流后，依赖 HTTP 头部的 `Content-Type` 来解码。如果响应头缺失 `charset` 字段，或者写成了 `charset=utf-8` 但 PowerShell 认为“不必强制”，它会退回默认代码页。很多内部 API 没有严谨标注 `charset`，于是中枪。

2. **控制台输出编码**：PowerShell 控制台本身的输出编码是 `[Console]::OutputEncoding`，默认值与系统区域有关。即使内存中字符串是正确的 UTF‑16（.NET 内部），当打印到控制台时，如果控制台不支持宽字符，就会显示为 `?`。

3. **文件写入**：`Out-File`、`Set-Content`、重定向 `>` 都沿用 PowerShell 的 `$OutputEncoding` 变量。该变量默认也是 ASCII（代码页 20127）或系统 ANSI，而非 UTF-8。这导致写出的文件在字节层面就错了。

4. **BOM 隐雷**：部分强迫式的 UTF-8 做法会带上 BOM（字节顺序标记），而下游 JSON 解析器（如 Python 的 `json.loads` 或 Node.js 的 `JSON.parse`）通常无法容忍 BOM，直接报错。

## 可重现的修复步骤

### 1. 修复 HTTP 响应的解码（推荐方案）

直接指定 `-ContentType` 并显式读取原始字节：

```powershell
$response = Invoke-WebRequest -Uri $uri -ContentType "application/json; charset=utf-8"
$rawBytes = $response.RawContentStream.ToArray()
$cleanJson = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$obj = $cleanJson | ConvertFrom-Json
$obj.message   # 输出“你好”
```

`Invoke-RestMethod` 也可用 `-OutFile` 绕开控制台，但它默认缺少 `-Encoding` 参数，无法保证 UTF‑8。更稳健的做法是用 `Invoke-WebRequest` 的 `RawContentStream`，纯手工控制字节到字符串的转换。

### 2. 固化环境编码配置

在脚本开头或 `$PROFILE` 中设置：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

这两行会改变所有 `>` 重定向、`Out-File` 以及控制台显示的编码行为。但要留意，如果某些 cmdlet 独立使用 `-Encoding` 参数，该设置无效；另外，改变 `[Console]::OutputEncoding` 会影响整个会话中所有控制台应用的输出显示。

### 3. 安全写出 UTF‑8（无 BOM）文件

如果需要将 API 结果保存为 JSON 文件：

```powershell
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText("result.json", $jsonStr, $utf8NoBom)
```

或使用 PowerShell Core 6+ 的 `Out-File -Encoding utf8NoBOM`。对于 Windows PowerShell 5.1，只能用 .NET 方法来保证无 BOM。

### 4. curl + native 管道安全处理

若必须用 `curl.exe`，可配合 `cmd` 重定向临时文件，再读入：

```bash
cmd /c "curl -s http://localhost:8080/api/echo > temp.json"
$json = Get-Content -Raw temp.json -Encoding UTF8
```

`Get-Content -Encoding UTF8` 会正确读取 UTF‑8 文件，即使用于生成文件的程序没有写入 BOM。

## 踩坑实录

- **Invoke-RestMethod 自动解析后再输出**：`$resp | ConvertTo-Json -Depth 3` 看上去在 PowerShell 控制台中文正常，实际是因为控制台渲染兼容了 UTF‑16，但写入文件时 `$OutputEncoding` 仍是 ANSI，导致保存到 `result.json` 后乱码。
- **PowerShell ISE 与普通控制台行为不一致**：ISE 内部输出编码不同，测试通过却在真机脚本中翻车。
- **设置 `$OutputEncoding` 后旧脚本受影响**：如果其他地方依赖默认 ANSI 文件输出（例如生成批处理文件），改变该变量可能导致其他脚本错误。建议仅在新会话或模块作用域内设定。
- **某些 REST API 返回的 `Content-Type` 带 `charset=utf-8`，但仍被错误解码**：这是因为响应流在传输过程中已被底层网络层转码过一次，例如某些代理或 .NET 的 `HttpClient` 缓存。此时强制用 `RawContentStream` 再按 UTF‑8 解码最可靠。

## 可复用建议

1. **封装一个 `Invoke-Utf8RestMethod` 函数**，内部统一通过 `Invoke-WebRequest` 获取原始字节、用 UTF‑8 解码、反序列化 JSON，外部只需关心返回值。
2. **在所有写文件的自动化任务中，显式指定编码**，绝对不使用 PowerShell 默认行为。
3. **为 Agent/MCP 插件编写自检步骤**：发送一个中文字符串到已知 echo 端点，校验返回结果，如果不一致则抛出清晰错误，提示环境编码问题。
4. **在 CI/CD 的 Windows 步骤中，执行 `chcp 65001` 并设置上述编码变量**，固化 UTF‑8 环境。
5. **对 JSON 文件增加 BOM 扫描**，避免下游工具因 BOM 中断。

## 总结

PowerShell 对中文 JSON 的破坏，表面上是“乱码”，本质是 Windows 生态中三个编码层面各自为政：HTTP 响应解码、控制台输出、文件写入。只要抓住 **UTF‑8 字节流不依赖任何默认转换** 这条原则，用 `RawContentStream` + 显式 UTF‑8 解码，配合无 BOM 写入，就能在整个自动化链路上告别“打坏”。在 OpenClaw 的 Agent 与 MCP 实践中，这种底层编码一致性的固执，远比多加一个 try/catch 更有价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/26a9b759b8df2a29.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/0c570adceea0bcb9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/0691a4e8281ef4b2.png)

