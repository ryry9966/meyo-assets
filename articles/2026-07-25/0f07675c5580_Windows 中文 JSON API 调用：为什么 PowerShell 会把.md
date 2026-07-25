---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏，以及如何可靠修复
feedId: 30432
source: 综合讨论
publishedAt: 2026-07-25
---

## 背景

在 OpenClaw 生态里，很多自动化环节不可避免地要用到 PowerShell——它随 Windows 预装、能直接操控系统对象，常被用作 MCP 工具或 Agent 插件的胶水脚本。然而，一旦脚本里需要向 API 发送含中文的 JSON，不少同学立刻撞上同一堵墙：服务端收到的中文是乱码，Agent 拿到的解析结果直接爆炸，排查起来又苦又绕。

这个问题不是 Unicode 时代该有的遗留症，但 PowerShell 5.1（Windows 10/11 默认版本）的默认编码行为，刚好与当今主流 API 假设的 UTF-8 不兼容。下面我会把原因和工程化解法讲清楚，让你下一次写 PowerShell API 调用时不再踩坑。

## 问题复现

假设你有一个 Go/Node/Python 写的 REST 服务，期望接收 UTF-8 编码的 JSON 请求体。你用 PowerShell 写了这样一个调用：

```powershell
$body = @{
    title = "今日待办"
    content = "检查 OpenClaw 插件状态"
} | ConvertTo-Json -Compress

Invoke-RestMethod -Uri "http://localhost:8080/api/task" `
    -Method Post `
    -Body $body `
    -ContentType "application/json; charset=utf-8"
```

看着一切正常，但抓包或服务端日志里收到的却是：

`{"title":"ä»Šæ—¥å¾…åŠž","content":"æ£€æŸ¥ OpenClaw æä»¶çŠ¶æ€"}`

这是典型的中文被当作 ISO-8859-1 字节流解释后再编码的结果。

## 为什么 PowerShell 会“打坏”中文

根源在于 `Invoke-RestMethod` 和 `Invoke-WebRequest` 的 `-Body` 参数处理方式：

- 当 `-Body` 接收到一个 `[string]` 对象时，PowerShell 会先用 **当前系统的默认 ANSI 代码页**（中文 Windows 通常是 GBK/CP936）将字符串转换成字节数组，然后再发送。
- 虽然你在 `ContentType` 里声明了 `charset=utf-8`，但只要 `-Body` 是字符串，这个声明 **不会** 改变 PowerShell 对字符串的编码方式——它仍然会用 ANSI 去编码。
- 于是，你内存中的 .NET 字符串（底层是 UTF-16）先被错误地按 GBK 编码成字节，然后接收端再按 UTF-8 解码，自然变成乱码。

PowerShell 7 开始，`-Body` 对字符串的编码行为有所调整，但在 Windows 上为避免破坏向后兼容性，依然有可能沿用系统代码页，所以根本解法不是换版本，而是改变传参方式。

## 可靠修复方案

### 1. 手动以 UTF-8 字节数组传递 Body

将 JSON 字符串显式转换为 UTF-8 字节数组，直接传给 `-Body`，这是最稳健的做法：

```powershell
$bodyJson = @{
    title = "今日待办"
    content = "检查 OpenClaw 插件状态"
} | ConvertTo-Json -Compress

$utf8Bytes = [System.Text.Encoding]::UTF8.GetBytes($bodyJson)

Invoke-RestMethod -Uri "http://localhost:8080/api/task" `
    -Method Post `
    -Body $utf8Bytes `
    -ContentType "application/json; charset=utf-8"
```

显式指定 `charset=utf-8` 是双保险，能让接收端更明确地按 UTF-8 解码。

### 2. 封装成可复用函数

自动化脚本里反复写编码转换太啰嗦，可以抽一个函数放进模块或 profile：

```powershell
function Invoke-Utf8JsonPost {
    param(
        [string]$Uri,
        [hashtable]$Body
    )
    $json = $Body | ConvertTo-Json -Compress
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($json)
    Invoke-RestMethod -Uri $Uri -Method Post -Body $bytes `
        -ContentType "application/json; charset=utf-8"
}
```

后续所有 POST 都用这个函数，编码问题一次性根治。

### 3. 如果必须用 PowerShell 5.1，建议同时设置 `$OutputEncoding`

`$OutputEncoding` 会影响 `Invoke-RestMethod` 对字符串的默认编码吗？通常不会，但有时在重定向或管道中也会掺和一脚。在脚本开头统一设定无害：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
```

仅作为防御性措施，核心仍然是用字节数组。

## 踩坑清单与排查思路

- **误以为 `-ContentType "charset=utf-8"` 就足够了**  
  它只是 HTTP 头，不改变 PowerShell 对字符串 Body 的编码方式。只设置这个头而不改 Body 编码方式，问题依旧。

- **脚本文件本身的编码陷阱**  
  用 ISE 或记事本保存 `.ps1` 文件时，如果选择“UTF-8 无 BOM”，PowerShell 5.1 可能无法正确解析其中的非 ASCII 字符串。建议脚本保持为 UTF-8 with BOM 或使用 Visual Studio Code 并明确指定编码。否则字符串在脚本加载时就已经是坏的。

- **`curl` 别名耽误排查**  
  PowerShell 中 `curl` 默认指向 `Invoke-WebRequest`，参数行为与真实 curl 不同。如果在 Agent 里把外部 curl 与 PowerShell 混用，可能出现一致性问题。建议在脚本里使用完整命令，或 `Remove-Alias curl` 后调用真实 `curl.exe`。

- **`ConvertTo-Json` 的深度与特殊字符**  
  复杂嵌套对象转 JSON 时，`-Depth` 参数不足会截断；某些 Unicode 字符可能被转义。建议用 `-Depth 10` 起步，必要时用 `[System.Web.HttpUtility]::JavaScriptStringEncode` 做额外转义，但注意不要双重编码。

- **代理与重定向干扰**  
  如果 OpenClaw 插件通过本地代理访问 API，代理层也可能再编码一次。排查时先用 `Invoke-WebRequest -OutFile` 保存原始响应用于比对。

## 总结与可复用建议

Windows 上做中文 API 调用，编码问题本质上是 PowerShell 将 Unicode 字符串错误地绑定到系统 ANSI 代码页的遗留行为。只要通过**字节数组 + UTF‑8 头**的方式传递 Body，就能在 5.1 和 7+ 版本上获得一致行为。

几点工程化建议：

- 编写任何面向 API 的 PowerShell 工具函数时，**从第一行就采用 `[System.Text.Encoding]::UTF8.GetBytes()`**，并把它沉淀到团队公共模块。
- 如果你的 MCP 服务器或 Agent 插件允许用 Python/Node 替代，在非 Windows 系统管理的场景优先考虑它们，避免无谓的编码消耗。
- 在 CI/CD 管道中运行 PowerShell 脚本时，统一指定 `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8`，能让日志和错误信息中的中文不混乱。
- 排查时优先用 `Invoke-WebRequest` 代替 `Invoke-RestMethod` 查看原始响应，配合 Fiddler/Wireshark 观察实际发出的字节流。

把编码搞定，你的 Agent 工具链才能真正稳定地处理中文内容，而不是每天都在怀疑“是不是网络又挂了”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/f51f3b1521dcb6e3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/2a10e3e4f9b3ed3f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/fda220998c78456c.png)

