---
title: PowerShell 调用中文 JSON API：为什么你的脚本总是在打坏字符
feedId: 29609
source: 综合讨论
publishedAt: 2026-07-19
---

# 背景：自动化脚本中那个“薛定谔的中文”

在 Windows 上编写 OpenClaw 插件、MCP 工具或任何需要调用 REST API 的 Agent 时，最让人抓狂的时刻不是 API 鉴权失败，也不是 JSON 结构拼错，而是——一切接口返回 200，逻辑全对，可存入文本或发给模型的中文内容就是乱码。

典型场景：你用 PowerShell 封装了一个翻译或知识库检索的 API 调用，把中文查询词作为 JSON 请求体发出，结果对方收到的像是 `æç±ä½` 或者一堆问号。或者返回的 JSON 里中文明明在控制台打印正常，一旦 `Out-File` 保存，再用其他工具读取就成了无法解析的字节流。

这一切的根源，并不是你代码逻辑的问题，而是 **Windows 默认的编码人格分裂** 与 **PowerShell 5.1 的向后兼容债务** 联手埋下的坑。

# 问题复现：一行看似正确的请求

假设我们有一个接受中文搜索词的 API：

```
POST /api/search
Content-Type: application/json

{"keyword":"你好世界"}
```

用 PowerShell 5.1 最自然的写法：

```powershell
$body = @{ keyword = "你好世界" } | ConvertTo-Json
Invoke-RestMethod -Uri "http://127.0.0.1:8080/api/search" -Method Post -Body $body -ContentType "application/json"
```

在日志里看服务器接收到的请求体，就会发现 `你好世界` 四个字可能已经变成了 `??????`，或者直接是 UTF-8 字节被当作 Latin-1 解码后的“乱码美术字”。更隐蔽的情况下，控制台打印 `$response` 时中文显示正常（因为控制台恰巧用 UTF-8 解码了内存中正确的字符串），但随后的一次 `Set-Content`、`Out-File` 或通过管道传给下一个程序，乱码方才暴露出来。

# 根因：三层编码不一致

Windows 下的经典困局主要由三个独立但串联的环节构成：

1. **`$OutputEncoding` 不是 UTF-8**  
   PowerShell 5.1 在将字符串自动转为字节流（比如作为 `-Body` 参数发送）时，使用的是 `$OutputEncoding` 变量指定的编码。在大多数中文 Windows 系统上，该值默认为 `us-ascii` 或 `system default`（对应 GBK/CP936）。当你传入一个 .NET 字符串（本质是 UTF-16），PowerShell 会先将其转换为 `$OutputEncoding` 指定的字节序列，再交给 HTTP 请求。于是 UTF-8 服务端自然就收到了损坏的中文。

2. **`-ContentType` 的 charset 缺失**  
   即使你在 `-ContentType` 里写了 `application/json`，如果没有显式追加 `charset=utf-8`，服务端可能根据 RFC 默认采用 ISO-8859-1 或盲目猜测编码，导致二次误会。而 PowerShell 自身在发送请求时，若未指定 charset，内部可能会用 `$OutputEncoding` 去标注或省略标注，形成错上加错。

3. **响应的解码路径不可控**  
   `Invoke-RestMethod` 对响应体的解码严重依赖 HTTP 响应头里的 `charset`。如果服务端返回了 `Content-Type: application/json` 却没有带上 `charset=utf-8`，PowerShell 大概率会退回到 `ISO-8859-1` 进行解码，于是正确的 UTF-8 字节流再一次被错误地映射成乱码字符串。这个乱码字符串在内存里已经是损坏状态，即便你随后用 `[System.Text.Encoding]::UTF8.GetBytes()` 再编码回去也无济于事。

尴尬的是，这三个环节经常同时出现，形成“所有环节看上去都没报错，但数据就是坏的”的局面。

# 工程化修复方案（按推荐程度排序）

## 方案一：强制全局 UTF-8（简单粗暴，适合一次性脚本）

在脚本顶部加入：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

同时务必在 `-ContentType` 中补齐 charset：

```powershell
$body = @{ keyword = "你好世界" } | ConvertTo-Json
Invoke-RestMethod -Uri ... -Method Post -Body $body -ContentType "application/json; charset=utf-8"
```

**局限**：`$OutputEncoding` 只对当前会话有效，重启 PowerShell 后会恢复；控制台字体可能仍需调整为支持中文的字体（如 Consolas 或宋体）才能正确显示，但这不影响实际数据。

## 方案二：使用字节数组绕过字符串编码推断（推荐用于自动化插件）

直接绕过 PowerShell 的字符串编码猜测，用明确的 UTF-8 字节发送：

```powershell
$jsonString = @{ keyword = "你好世界" } | ConvertTo-Json
$utf8Bytes  = [System.Text.Encoding]::UTF8.GetBytes($jsonString)

Invoke-RestMethod -Uri ... -Method Post -Body $utf8Bytes -ContentType "application/json; charset=utf-8"
```

这样做后，PowerShell 不再对 `-Body` 做额外的编码转换，而是将字节数组原样发送。这是最确定、最不会出错的方式，特别适合需要严格保证字符完整性的 OpenClaw 工具链。

## 方案三：直接使用 .NET HttpClient 接管一切

当需要大量 API 交互时，完全绕过 PowerShell 的命令层，用 C# 风格的代码控制每一个细节：

```powershell
$http = [System.Net.Http.HttpClient]::new()
$content = [System.Net.Http.StringContent]::new(
    $jsonString,
    [System.Text.Encoding]::UTF8,
    "application/json"
)
$response = $http.PostAsync("http://127.0.0.1:8080/api/search", $content).Result
$responseBody = $response.Content.ReadAsStringAsync().Result
```

这种方式完全不依赖 `$OutputEncoding`，编码和解码都由你自己指定的 UTF-8 处理，适合封装成稳定复用的函数或模块。

## 持久化注意：文件写出也要 UTF-8

修复了 API 调用，不代表万事大吉。当你把返回结果用 `Out-File` 或 `Set-Content` 保存时，在 Windows PowerShell 5.1 中默认会写入 UTF-8 with BOM 或甚至 UCS-2 LE。为了避免下游 MCP 工具读取出错，务必显式指定编码：

```powershell
$response | Out-File -FilePath result.json -Encoding utf8NoBOM
```

如果 PowerShell 版本不支持 `utf8NoBOM`，回到 .NET 方式：

```powershell
[System.IO.File]::WriteAllText("result.json", $response, [System.Text.UTF8Encoding]::new($false))
```

# 踩坑与排障清单

- **Body 为哈希表而非 JSON 字符串**：`Invoke-RestMethod` 会自动序列化为 JSON，但编码仍走 `$OutputEncoding`，容易忘记。始终手动 `ConvertTo-Json` 并转为字节数组更安全。
- **控制台输出正常≠数据正常**：控制台可能因为 `[Console]::OutputEncoding` 巧合性地显示正确，实际内存中早已损坏。用 `[System.Text.Encoding]::Default.GetBytes()` 检查字节序列是最可靠的验证手段。
- **PowerShell 版本差异**：PowerShell Core 7+ 已默认将 `$OutputEncoding` 设为 UTF-8，很多莫名其妙的问题会自动消失。如果你的 Agent 环境可以升级，直接换用 pwsh 是最省事的解法。
- **代理与重定向干扰**：部分公司网络在中间代理或 VPN 层对 HTTP 头进行改写，甚至移除 charset。记得抓包比对原始响应头，排除环境因素。

# 可复用建议：内置防损函数

在 OpenClaw 插件或自动化 Agent 的 PowerShell 脚本中，我习惯立刻声明一个安全 API 调用函数：

```powershell
function Invoke-Utf8ApiPost {
    param($Uri, $Data)
    $json = $Data | ConvertTo-Json -Compress
    $bytes = [Text.Encoding]::UTF8.GetBytes($json)
    $resp = Invoke-RestMethod -Uri $Uri -Method Post -Body $bytes -ContentType "application/json; charset=utf-8"
    return $resp
}
```

后续所有涉及中文的 API 调用统一走这个入口，既避免重复踩坑，也让代码意图更清晰。

# 总结

Windows 中文 API 调用被“打坏”的本质，是 PowerShell 的字符串与字节转换路径上存在太多向下兼容的编码假设。没有报错、没有异常，数据却悄悄损坏，这正是编码问题最危险的地方。

工程化的解决思路只有一条：**在每一个字节边界上，由你自己明确指定 UTF-8，而不是信任默认行为**。无论是 `$OutputEncoding`、`-ContentType`、`-Body` 的编码，还是文件写出，都强制统一为 UTF-8。如果代码量稍大，用 .NET 的直接 API 替换掉 PowerShell 的命令糖，是减少隐性 Bug 的长远之计。

在中文 AI Agent 工具链不断丰富、本地 MCP 插件越来越多的今天，字符编码问题看似古老，却依然会在你不经意间阻塞整个自动化管道。花半小时加固编码处理，换来未来无数次不掉坑，很值得。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/8c1de126760c1370.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/b51b02eca7041a4e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/0d234c6b8b653994.png)

