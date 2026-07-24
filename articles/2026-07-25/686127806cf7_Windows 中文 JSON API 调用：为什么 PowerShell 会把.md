---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏？
feedId: 30350
source: 综合讨论
publishedAt: 2026-07-25
---

## 背景
在基于 Agent、MCP 或自定义插件的自动化工作流中，我们经常需要通过 HTTP API 下发结构化数据，比如向 LLM 传递中文提示词、写入带有中文 commit 信息或更新工单描述。当 Windows 成为运行环境时，PowerShell 几乎是默认的胶水语言。然而，许多开发者在 Windows 上使用 `Invoke-RestMethod` 或 `Invoke-WebRequest` 发送 JSON 后，发现服务端收到的中文变成了 `????`、`\uXXXX` 转义序列，甚至直接报编码错误。更诡异的是，同样的脚本在 PowerShell 7 (pwsh) 上正常，在 Windows PowerShell 5.1 上就“打坏”中文。这不是偶然，而是一系列历史编码设定叠加的结果。

## 问题根因：管道与 cmdlet 的编码战争
Windows PowerShell 5.1 基于 .NET Framework 4.x，内部字符串全部以 UTF-16LE 存储，但它的外部输入/输出（文件、管道、Web 请求）默认编码并不统一。

- **`$OutputEncoding`**：决定了 PowerShell 向外部程序发送数据时使用的编码。Windows PowerShell 默认值是 **ASCII**（确切地说是与系统活动代码页相关的编码，英文系统为 Windows-1252）。当你将字符串通过管道传给 `curl` 或其它外部命令时，中文会被强制转成单字节编码，高位字符直接丢失，变成 `?`。
- **Web cmdlet 的 `-Body` 行为**：在 Windows PowerShell 中，如果直接传入一个 `[string]` 作为 `-Body`，cmdlet 会使用 **ISO-8859-1**（Latin-1）编码将字符串转为字节数组。这对于中文是致命的，因为中文码点在 ISO-8859-1 中无效，全部被替换为 `?`（0x3F）。即使你指定了 `-ContentType "application/json; charset=utf-8"`，也于事无补，损坏发生在转换阶段。
- **控制台输出与重定向**：即使 API 返回了正确的 UTF-8 数据，`Write-Host` 或默认输出到控制台时，还会受到 `[Console]::OutputEncoding` 的影响。如果控制台代码页是 936 (GBK) 或 437，而终端字体不支持中文，你看到的仍然是乱码或空白方块。
- **`ConvertTo-Json` 转义**：低版本 PowerShell 的 `ConvertTo-Json` 默认会将非 ASCII 字符转义为 `\uXXXX`，虽然服务器能解析，但可读性差，且在某些严格校验 Unicode 转义的服务端可能引发意外。

这些因素累加，使得“简单”的 POST 一个中文 JSON 成为踩坑合集。

## 步骤：打造一份“打不坏”的中文请求
下面给出在 Windows PowerShell 5.1 和 PowerShell 7 中都能稳定工作的方案。优先推荐使用 PowerShell 7，如果环境不可变，则按第三点执行。

### 1. 构建 JSON 并转为 UTF-8 字节数组
不要直接把 `[string]` 传给 `-Body`，统一使用 `[byte[]]`。这样绕过了 cmdlet 的文本编码步骤。

```powershell
$payload = @{
    prompt = "你好，世界"
    user   = "agent-alpha"
} | ConvertTo-Json -Depth 3

# 将 JSON 字符串以 UTF-8 编码转换为字节序列
$bodyBytes = [System.Text.Encoding]::UTF8.GetBytes($payload)
```

### 2. 发起请求并指定内容类型
```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/generate" `
                              -Method Post `
                              -Body $bodyBytes `
                              -ContentType "application/json; charset=utf-8"
```
此时 `-Body` 接收的是字节数组，cmdlet 不再进行额外编码，直接发送原始字节，服务端能正确接收 UTF-8 中文。

### 3. 安全读取响应内容
如果响应体包含中文，且你需要以文本形式处理（例如保存或显示），确保读取时使用 UTF-8 解码：

```powershell
# 获取原始内容（会进行自动解码，通常正确）
$content = $response.Content

# 如果仍然乱码，可强制从字节流读取
$responseRaw = Invoke-WebRequest -Uri ... -Body $bodyBytes -ContentType ...
$utf8String = [System.Text.Encoding]::UTF8.GetString($responseRaw.RawContentStream.ToArray())
```

### 4. 输出到文件时显式指定 UTF-8
```powershell
# 推荐使用 Set-Content 或 Out-File 的 -Encoding 参数
Set-Content -Path "result.json" -Value $content -Encoding UTF8
```
**避免使用重定向运算符 `>`**，因为它使用 `Out-File` 的默认 Unicode (UTF-16LE) 编码，会产出不兼容的宽字节文件。

### 5. 修正控制台显示（可选）
如果你需要在屏幕上看到正确的中文，运行以下两条命令：
```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```
同时，将控制台字体设为支持 CJK 的等宽字体（例如新宋体、SimSun-ExtB 或更纱黑体）。

## 踩坑点
1. **`Invoke-RestMethod` 自动解析 JSON 时截断中文**  
   极少数 API 返回的 Content-Type 不包含 `charset=utf-8`，但实际是 UTF-8，cmdlet 可能按 ISO-8859-1 解码，导致 `$response` 对象的属性出现乱码。此时可先 `Invoke-WebRequest` 拿到原始流，手动解码。

2. **`ConvertTo-Json` 深度限制与转义**  
   对于多层嵌套对象，务必设置 `-Depth` 大于默认的 2。若需生成无转义中文字符的 JSON，可在 PowerShell 7 中使用 `ConvertTo-Json -EscapeHandling EscapeNonAscii` 的综合控制，或在 Windows PowerShell 中使用模块 `JSON.net`。

3. **Windows PowerShell ISE 与 VSCode 终端的差异**  
   ISE 的内置窗格使用奇怪的编码重定向，会导致与独立终端表现不同。调试编码问题应始终在独立 `powershell.exe` 或 `pwsh.exe` 控制台中进行。

4. **`Content-Type` 大小写与 BOM**  
   部分强类型 API 对 `charset` 大小写敏感，建议全小写 `utf-8`。另外，不要给 POST 请求体添加 UTF-8 BOM（字节顺序标记），因为 JSON 规范禁止 BOM，可通过 `$bodyBytes` 直接构造避免。

## 可复用建议
- **封装请求函数**：将上述字节数组构造、请求和响应解码打包为模块函数 `Invoke-ApiUtf8`，在所有自动化脚本中统一调用，避免重复犯错。
- **环境检测与降级**：在脚本开头判断 `$PSVersionTable.PSEdition`，如果是 `Desktop`（Windows PowerShell），则自动设置全局编码变量并引入安全请求模板。
- **在 `$PROFILE` 中预设编码**：为所有交互式会话添加 `$OutputEncoding = [Console]::OutputEncoding = [System.Text.Encoding]::UTF8`，虽然不能解决 Web cmdlet 的 `-Body` 字符串问题，但能减少管道和控制台乱码。
- **使用 curl.exe 替代**：在极度顽固的环境下，可以考虑直接在 PowerShell 中调用系统的 `curl.exe`，配合 `--data-binary @file.json` 和 `-H "Content-Type: application/json; charset=utf-8"`，将 PowerShell 的编码干扰降到最低。

## 总结
PowerShell 把中文“打坏”，本质是历史遗留的默认编码与现代 UTF-8 生态的冲突。Windows PowerShell 为了向后兼容，选择了保守的 ASCII/ISO-8859-1 行为，而运维自动化中的 Agent、插件、OpenClaw 应用早已全面拥抱 UTF-8。理解了这一错配，并有意识地使用字节数组、显式声明编码、升级到 PowerShell 7，就能彻底告别中文乱码的噩梦。稳定的字符传输是自动化管道中不起眼却至关重要的一环，值得为它留下这份防坑笔记。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/9579bca7779b1556.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/764120eb64109754.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/c69a9ca20889f1a0.png)

