---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 29704
source: 综合讨论
publishedAt: 2026-07-20
---

# Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏

## 背景

在 Windows 上使用 PowerShell 调用 JSON API，是 OpenClaw 社区搭建 Agent、MCP 服务、插件编排的常见操作。无论是拉取远程数据，还是与本地进程交互，脚本里总少不了 `Invoke-RestMethod` 或 `Invoke-WebRequest`。但不少人在处理中文响应时，发现日志里本该是“任务已下发”的地方变成了 `?????`，或者 `ConvertFrom-Json` 直接报 `Invalid JSON primitive`。

这个问题的根源不是 JSON 库有 bug，也不是 API 服务端作恶，而是**PowerShell 在处理 HTTP 响应时对文本编码的自动推断行为，在 Windows 系统上与 UTF-8 生态存在「假想默契」的偏差**。

---

## 问题复现

假设有一个最简单的 API（Flask、Node、Rust 均可），返回如下 JSON：

```json
{"status": "成功", "message": "你好，世界"}
```

在 Windows Terminal 中用 PowerShell 调用：

```powershell
$resp = Invoke-RestMethod -Uri http://127.0.0.1:5000/api/demo -Method Get
$resp.status
```

期望看到 `成功`，实际却输出 `æˆåŠŸ` 或 `????`，并且 `$resp.message` 也是一片乱码。当你把这组数据写入文件再被下游程序消费时，连锁报错就开始了。

---

## 真正的原因

### 1. PowerShell 的响应解码逻辑

`Invoke-RestMethod` 和 `Invoke-WebRequest` 内部会查看 HTTP 响应头中的 `Content-Type` 字段来决定解码方式。

如果服务端返回的是：

```
Content-Type: application/json
```

**没有声明 charset**，PowerShell 在 **Windows PowerShell 5.1** 中的默认行为是：退回到 **ISO-8859-1**（近似 Windows-1252）来解码字节流。UTF-8 编码的中文被按单字节拉丁编码解释，必然变成乱码。PowerShell 7 虽然默认改成了 UTF-8，但仍可能因为系统的 `[System.Text.Encoding]::Default` 设置而走入相同陷阱。

### 2. 输出重定向的二次伤害

即便你侥幸用别的手段拿到了正确的字符串，再用 `>` 重定向输出到文件，也可能将 UTF-8 数据毁掉。在 Windows PowerShell 中，`>` 操作符会调用 `Out-File`，默认编码是 **UTF-16 LE**。下游如果期望 UTF-8，就会看到肉眼不可见的 BOM 和双字节编码，再次形成乱码。

### 3. 请求端同样存在隐患

当你在请求体内发送中文时，如果只写了：

```powershell
$body = @{ text = "你好" } | ConvertTo-Json
Invoke-RestMethod -Uri ... -Method Post -Body $body -ContentType "application/json"
```

`ConvertTo-Json` 默认不会转义非 ASCII 字符，但 `-Body` 参数将字符串转换为字节流时，若无明确指定，可能会使用系统的 ANSI 编码（如 GBK），导致服务端收取到已损坏的字节。

---

## 做法与步骤

### 正确读取中文响应

**推荐方案：手动控制解码流程**

```powershell
$response = Invoke-WebRequest -Uri http://127.0.0.1:5000/api/demo -Method Get
$utf8 = [System.Text.Encoding]::UTF8
$bodyText = $utf8.GetString($response.RawContentStream.ToArray())
$data = $bodyText | ConvertFrom-Json
```

即使 `Content-Type` 头缺失，这里也能强制用 UTF-8 还原字符串。更稳妥的做法是封装成函数：

```powershell
function Get-UTF8Json {
    param($Uri)
    $resp = Invoke-WebRequest -Uri $Uri -Method Get
    $bytes = $resp.RawContentStream.ToArray()
    ([System.Text.Encoding]::UTF8.GetString($bytes)) | ConvertFrom-Json
}
```

### 请求时确保编码一致

显式控制 `-Body` 所使用的编码，避免交给 PowerShell 推断：

```powershell
$body = @{ text = "你好" } | ConvertTo-Json
$encodedBody = [System.Text.Encoding]::UTF8.GetBytes($body)
Invoke-RestMethod -Uri ... -Method Post -Body $encodedBody -ContentType "application/json; charset=utf-8"
```

将 `-Body` 指定为字节数组时，PowerShell 会原样传输，不再自作主张。

### 安全地写入文件

用 `Out-File` 或 `Set-Content` 时务必指定 `-Encoding UTF8`：

```powershell
$result | ConvertTo-Json | Out-File -FilePath data.json -Encoding UTF8
```

或者在 PowerShell 7 中使用重定向时，先设置 `$PSDefaultParameterValues` 或显式使用 `Set-Content`。对于跨平台消费，建议使用不带 BOM 的 UTF8：

```powershell
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllLines("data.json", $jsonString, $utf8NoBom)
```

---

## 踩坑点

1. **`Invoke-RestMethod` 的 `-ContentType` 参数只影响请求，不影响响应的解码方式**。很多人误以为加了 `charset=utf-8` 就能解决返回乱码，结果是请求发对了，但接收时依然用错误编码解码。

2. **`ConvertFrom-Json` 直接作用于 `$response.Content` 时内容已被破坏**，需要从字节流重新解码。

3. **PowerShell 7 并非万能保险**。若 Windows 系统的 `System Locale` 是中文（代码页 936），某些 .NET 的编码回退仍会干扰 `RawContentStream` 的使用方式，最好始终显式指定 `UTF8`。

4. **服务端忘记设置 `charset` 是乱码的导火索**，但对于运维脚本，你无法总要求所有 API 都规范，所以客户端防御是必须的。

---

## 可复用建议

- 在 OpenClaw 的自动化脚本中，**统一封装一个 `Invoke-RestMethodSafe` 函数**，内部使用 `Invoke-WebRequest` + 字节流 + UTF-8 解码，将所有 JSON 交互标准化。
- 所有输出文件的 cmdlet 都显式添加 `-Encoding UTF8`，并考虑使用 `$PSDefaultParameterValues['Out-File:Encoding']='utf8'` 来系统级预防。
- 在 Windows 环境下，若条件允许，**优先使用 PowerShell 7**，它的编码行为更符合网络标准，但仍需结合上述方法巩固。
- 对于关键流程，可在读取到字符串后加入验证：`if ($text -match '[^\x00-\x7F]') { ... }`，快速识别是否发生了乱码并触发重试或报错。

---

## 总结

PowerShell 把中文打坏，本质是**自动化编码推断**与**URL/JSON 生态的隐式约定**发生了冲突。Windows 历史包袱重，ANSI 代码页与 UTF-8 交替存在，再加上 `Invoke-RestMethod` 对响应解码的“自动猜测”，导致中文 API 调用极易踩坑。只要你在脚本层显式接管字节->字符串的转换过程，并固定文件写入编码为 UTF-8，就能彻底告别乱码问题。

在这样的工程化小工具上多花十分钟，能省去日后半天排查 OpenClaw agent 日志的无奈。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/2e87fe362a14fc60.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/cc0bf2581f998466.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/9940b7e28a3001f0.png)

