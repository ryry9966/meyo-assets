---
title: Why PowerShell Corrupts Chinese JSON on Windows – And How to Fix It for Reliable API Automation
feedId: 30762
source: 综合讨论
publishedAt: 2026-07-28
---

# Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏

你刚刚给 OpenClaw 开发了一个 MCP 工具，返回的 JSON 内容包含中文书籍摘要。本地调试一切正常，部署到 Windows 服务器后，Agent 开始把“《红楼梦》”识别成“《çº¢æ¥¼æ¢¦》”。不是模型的问题，不是 prompt 的问题，而是 PowerShell 在你和 API 之间安插了一道编码毒化层。

这类问题在 Windows 原生环境中几乎必然出现，尤其当你用 Invoke-RestMethod 或 Invoke-WebRequest 消费任何返回中文 JSON 的 API 时。本文将完整拆解原因，给出可复用的工程化修复方案，并提炼出适用于 Agent/MCP/插件自动化的编码铁律。

## 背景与典型触发场景

Windows 10/11 默认预装的 PowerShell 版本是 5.1，其底层输出编码基于传统 OEM 代码页（例如国区简体中文为 CP936）。而绝大多数现代 API（无论是自建的 FastAPI、Flask 服务，还是公网接口）都使用 UTF-8 返回 JSON。当你写下类似这样的代码：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/books/1" -Method Get
Write-Host $response.title
```

看到的是乱码，甚至部分字符直接消失。更隐蔽的是，如果通过 `ConvertTo-Json` 将对象管道传递给另一个进程或文件，二次伤害还会叠加——Windows 控制台宿主、文件重定向与 .NET 字符串内部的编码转换会在三个环节分别做一次错误假定。

常见触发组合：
- 使用 `Invoke-RestMethod` 直接获取 JSON 响应，然后输出到控制台/日志文件
- 通过 `Start-Job` 或 `ForEach-Object -Parallel` 并发请求后序列化结果
- 把返回内容写入临时文件供 Agent 工具链读取
- 在 CICD runner（Azure DevOps 或 GitHub Actions 的 Windows runner）上调用 PowerShell 脚本处理中文元数据

如果你在用 OpenClaw 的 ExecuteShell 插件调用本地 PowerShell 脚本去拉取配置或知识库内容，几乎必然踩中这个坑。

## 问题根源：多层次的编码错配

PowerShell 5.1 对 Web cmdlet 的编码处理可以拆解为三个错配层：

**1. HTTP 响应流解码假设**
`Invoke-WebRequest` 和 `Invoke-RestMethod` 底层依赖 .NET 的 `HttpWebResponse`。如果没有在请求中明确指定 `Content-Type` 的 charset，或者 API 虽然声明了 `charset=utf-8` 但实现不严格，.NET Framework 4.x 会用系统默认代码页去解码字节流。PowerShell 5.1 寄生在 .NET Framework 4.x 上，即使用 UTF-8 作为字节传输，也会被错误解码成 CP936，导致多字节字符断裂。

**2. 控制台输出编码**
即使你在内存里得到了正确的 .NET 字符串，`Write-Host` 或默认输出到控制台时，PowerShell 5.1 使用 `[Console]::OutputEncoding` 进行转换，该属性默认与 OEM 代码页相同，而非 UTF-8。UTF-8 字符串被强行转码后输出，自然是乱码。

**3. 管道与文件重定向**
`Out-File`、`Set-Content` 甚至 `>` 重定向默认使用 `Unicode`（UTF-16LE）或 `Default`（取决于 PowerShell 版本），但你若显式指定 `-Encoding UTF8`，则可能写入带 BOM 的 UTF-8，而下游 Linux 工具或解析器（如 jq）会因 BOM 崩溃。

这三层只要错配一层，中文就会坏。更棘手的是，在 PowerShell Core 6+ 中问题已大幅缓解（默认 UTF-8 且无 BOM），但很大一部分生产环境仍然挂着 5.1。

## 工程化修复步骤

**步骤 1：强制使用 .NET 的 HttpClient 获取原始字节，自行解码**

放弃 `Invoke-RestMethod` 对自动解析 JSON 的便利，改用更底层可控的方式：

```powershell
$uri = "https://api.example.com/books/1"
$req = [System.Net.WebRequest]::Create($uri)
$req.Method = "GET"
$req.ContentType = "application/json; charset=utf-8"
$res = $req.GetResponse()
$stream = $res.GetResponseStream()
$reader = New-Object System.IO.StreamReader($stream, [System.Text.Encoding]::UTF8)
$jsonStr = $reader.ReadToEnd()
$reader.Close()
$res.Close()
$obj = $jsonStr | ConvertFrom-Json
```

这里的关键是 `StreamReader` 强制使用 UTF-8 解码，完全绕过 .NET Framework 的字符集自动检测。

**步骤 2：统一控制台/文件输出编码**

在脚本入口加入以下代码（可通过 Profile 或脚本头固化）：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

`$OutputEncoding` 影响 PowerShell 的管道传递给外部程序的编码，`[Console]::OutputEncoding` 影响控制台宿主。两者都设成 UTF-8，可以防止打印字符串时被再次破坏。

**步骤 3：写出供 Agent/MCP 工具消费的文件时，明确不带 BOM**

```powershell
$jsonStr | Out-File -FilePath "response.json" -Encoding utf8NoBOM
# 或者在 PowerShell 5.1 中，用 .NET 方法直接写出：
[System.IO.File]::WriteAllText("response.json", $jsonStr, [System.Text.UTF8Encoding]::new($false))
```

参数 `$false` 用于创建不带 BOM 的 UTF-8 编码。所有期望纯 JSON 的工具都应该接收 BOM-less UTF-8。

**步骤 4：封装可复用的安全请求函数**

对于需要在多个 Agent 工具脚本里复用的请求，可封装函数：

```powershell
function Invoke-Utf8Api {
    param($Uri, $Method = 'Get', $Body)
    $request = [System.Net.WebRequest]::Create($Uri)
    $request.Method = $Method
    $request.ContentType = "application/json; charset=utf-8"
    if ($Body) {
        $bytes = [System.Text.Encoding]::UTF8.GetBytes($Body)
        $request.ContentLength = $bytes.Length
        $stream = $request.GetRequestStream()
        $stream.Write($bytes, 0, $bytes.Length)
        $stream.Close()
    }
    $response = $request.GetResponse()
    $reader = New-Object System.IO.StreamReader($response.GetResponseStream(), [System.Text.Encoding]::UTF8)
    $content = $reader.ReadToEnd()
    $reader.Close()
    $response.Close()
    return $content | ConvertFrom-Json
}
```

这个函数会返回正确解析的中文 JSON 对象，完全不受系统代码页影响。

## 踩坑点与可复用建议

- **不要相信 `-ContentType` 参数**：`Invoke-WebRequest` 的 `-ContentType` 仅仅设置请求头，不会影响响应解码。很多教程误导读者以为加上就能解决，实则无效。
- **不要指望 `chcp 65001` 能彻底修复**：修改控制台代码页只能缓解显示问题，管道、文件输出编码依旧混乱，且副作用影响其他命令。
- **注意 `ConvertFrom-Json` 的深度解析**：对于深层嵌套的长文本，使用 `ConvertFrom-Json -Depth 10` 以避免截断。中文截断会导致 JSON 解析失败。
- **跨平台兼容**：编写适用于 Windows/Linux 的插件脚本时，用 `if ($PSVersionTable.PSVersion.Major -lt 6)` 判断，分别采用 .NET 方法（5.1）或原生 `-Encoding utf8NoBOM`（6+）。
- **Agent 工具返回格式**：如果 MCP 工具指定返回纯文本，但内部模拟了 JSON 输出，请在 PowerShell 末尾显式写入命名管道或标准输出时使用 UTF-8 BOM-less，并在调用侧（如 Python subprocess）指定 `encoding='utf-8'`。

## 总结

Windows 上 PowerShell 5.1 的编码行为是遗留系统与现代 API 世界之间的定时炸弹。面向 Agent 的自动化流程一旦涉及命令行调用和数据交换，中文乱码会让整个工具链的可靠性急剧下降。解决办法不是绕过 PowerShell，而是用 StreamReader 显式接管解码、统一输出编码为 UTF-8 无 BOM，并把这种安全请求模式固化到团队的基础脚本库中。

下次当你的 Agent 输出“ç¥¨æ”时，你可以平静地打开控制台，把 `[Console]::OutputEncoding` 和 `StreamReader(stream, UTF8)` 两行代码甩给报错 —— 然后继续构建真正可靠的自动化节点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/eaad27f62693f4dc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/608a874725c9f8f8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/0f140e0a252da07e.png)

