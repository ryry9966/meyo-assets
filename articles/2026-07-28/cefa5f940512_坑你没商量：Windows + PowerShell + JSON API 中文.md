---
title: 坑你没商量：Windows + PowerShell + JSON API 中文为何“打坏”，及可复现修复方案
feedId: 30719
source: 综合讨论
publishedAt: 2026-07-28
---

## 背景

在构造 OpenClaw 模型调用、Agent 工具链或 MCP 插件时，很多人会在 Windows 上用 PowerShell 测试 API：发起 `Invoke-RestMethod` 或 `Invoke-WebRequest`，向模型传中文 prompt，或让模型返回中文 JSON。然而打开终端，看到的不一定是正常中文，而是一串 `???`、`锟斤拷` 或 `\uXXXX` 转义堆。另一些时候，服务端收到的直接是乱码，导致插件处理参数出错。这种“打坏”并非偶发，根本原因在于 PowerShell 在处理字符编码、管道输出、重定向时的默认行为与多数 API 期待的不一致。

## 问题：中文怎么坏掉的？

结合多次踩坑经验，破坏点主要集中在三处：

1. **PowerShell 发送请求体时的编码**  
   `Invoke-RestMethod -Body` 默认会把字符串转换成 `ISO-8859-1` (Windows-1252) 编码，而不是 UTF-8。于是请求体中的中文字节被错误编码，服务端解出来就是乱码。

2. **控制台输出编码**  
   PowerShell 5.1 控制台默认代码页通常是 936 (GBK) 或 437，而 API 返回的是 UTF-8 JSON。`Write-Host` 或直接输出字符串到控制台时，中文字符可能按当前代码页解释，导致显示乱码。

3. **文件重定向与管道编码**  
   使用 `>` 或 `Out-File` 默认编码是 UTF-16 LE (Unicode) 或 ASCII（取决于 PowerShell 版本），与多数后续处理工具期待的 UTF-8 不符。当 Agent 把 API 响应写入文件再传给下一个插件时，很容易出现文件内容损坏。

如果你的工具链是这样的：
```
PowerShell 脚本调用 API → 返回 JSON → 解析中文 → 传给 MCP 插件 → 插件判断失败
```
那八成是这里出了问题。

## 做法/步骤：让中文安全通过每一环

下面给出一个可复现、无坑的调用写法框架。

### 1. 确保请求体以 UTF-8 发送

```powershell
$body = @{
    model = "gpt-4o-mini"
    messages = @(
        @{ role = "user"; content = "解释什么是‘百发失一，不足谓善射’？" }
    )
} | ConvertTo-Json -Depth 10

$response = Invoke-RestMethod -Uri $uri -Method Post `
    -ContentType "application/json; charset=utf-8" `
    -Body ([System.Text.Encoding]::UTF8.GetBytes($body))
```

关键点：不直接传字符串 `$body`，而是将其转为 UTF-8 字节数组。`-ContentType` 显式指出 `charset=utf-8`，虽然很多服务端会忽略，但有助于避免一些反向代理的猜测错误。

### 2. 正确解析返回的 JSON，保证内部字符串是 .NET 的 `System.String`

`Invoke-RestMethod` 返回的对象已自动反序列化，中文内容在内存中是正常的。此时不要直接用 `Write-Host $response.choices[0].message.content`，因为控制台渲染可能会把字打坏。如果需要预览，改用：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$response.choices[0].message.content
```

建议只在开发调试时设置 `[Console]::OutputEncoding`，脚本开头加上判断，避免在无人值守环境下非预期修改。

### 3. 持久化输出时强制 UTF-8

若需将 API 返回结果保存为 JSON 文件供其他 Agent 或 MCP 插件读取：

```powershell
$response | ConvertTo-Json -Depth 10 | Out-File -FilePath "result.json" -Encoding utf8
```

注意：是 `utf8`（带 BOM），如果下游工具要求不带 BOM，可改用：

```powershell
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText("result.json", ($response | ConvertTo-Json -Depth 10), $utf8NoBom)
```

### 4. 用 Invoke-WebRequest 处理原始字节流

如果你需要更精细的控制（例如自行解析响应体），用 `Invoke-WebRequest` 获取字节数组并转成 UTF-8 字符串：

```powershell
$resp = Invoke-WebRequest -Uri $uri -Method Post -Body $utf8Bytes -ContentType "application/json; charset=utf-8"
$rawJson = [System.Text.Encoding]::UTF8.GetString($resp.Content)
```

## 踩坑点

- **PowerShell 5.1 与 7+ 差异**：  
  PowerShell Core (7+) 默认使用 UTF-8 作为 `Out-File` 等命令的编码，但仍需注意 `$OutputEncoding` 变量。部分背景作业中，UTF-8 设置可能不继承，导致子任务继续用默认编码，最好在脚本开头显式 `$OutputEncoding = [System.Text.Encoding]::UTF8`。

- **curl 别名陷阱**：  
  在 Windows 上，PowerShell 的 `curl` 是 `Invoke-WebRequest` 的别名，不是真正的 cURL。参数行为不同，遇到网上复制来的 `curl` 命令要转换成 PowerShell 风格，虽然这不直接影响中文编码，但会让调试过程雪上加霜。

- **API Key 或 Headers 中的中文？**  
  一般不会，但如果你的插件需要传递包含中文的 Header 值（如自定义元数据），同样需要编码处理，虽然多数 API 要求 Header 值仅限 ASCII。

- **VS Code 终端与外部终端不一致**：  
  VS Code 集成终端可能已配置为 UTF-8，而在双击 `.ps1` 脚本弹出的原生窗口中仍为 GBK 编码。请统一在脚本中通过 `[Console]::OutputEncoding` 设定。

## 可复用建议

- **封装一个 Invoke-Api 函数**：在 OpenClaw 项目或插件里提供一个标准化函数，内部统一处理 UTF-8 请求体生成、字节发送、响应解码、文件输出，避免团队中每个人各自调试编码问题。
- **为 PowerShell 脚本添加编码检测断言**：在请求发送前，检查 `$OutputEncoding` 是否为 UTF-8；保存文件后，用 `Get-Content -Encoding byte` 检查首几个字节是否为 BOM 或合法 UTF-8，若不满足则抛出明确错误，避免静默传递乱码。
- **利用 Webhook 调试器对比**：先使用 Postman 或 CURL（原生）确认 API 本身返回正确，然后对比 PowerShell 结果，差异点即编码问题所在。

## 总结

Windows 上 PowerShell 的中文 JSON API 乱码，绝大部分是编码设置与 API 预期不一致造成，发送、输出、文件三环缺一不可，必须逐环用 UTF-8 贯通。记住三点：发送字节数组、控制台输出设置 UTF-8、文件保存显式指定 UTF-8（或 UTF-8 no BOM）。这样一来，OpenClaw Agent 或多插件链路的稳定性会明显提升，避免把时间花在反复排查“锟斤拷”上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/a874e7fb139a8afb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/84456a7396b05eed.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/df8068e8d676c440.png)

