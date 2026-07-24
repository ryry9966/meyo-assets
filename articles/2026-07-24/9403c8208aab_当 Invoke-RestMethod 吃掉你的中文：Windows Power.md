---
title: 当 Invoke-RestMethod 吃掉你的中文：Windows PowerShell 下的编码陷阱
feedId: 30302
source: 综合讨论
publishedAt: 2026-07-24
---

## 背景

在 OpenClaw 或自定义 MCP 插件的实践中，Windows 用户经常用 PowerShell 脚本调用本地 API、远程 JSON 接口，或充当 Agent 与外部服务通信的“胶水”。常见的场景是：`Invoke-RestMethod` 发送带中文参数的请求，或解析返回的中文响应，然后将结果传给下游。但现实是，你可能会看到服务端日志里收到的是 `???` 或 `æ±‰å­—`，终端窗口里的中文也显示为乱码。更糟糕的是，Agent 在解读这些损坏的字符串时会消耗宝贵的 token，甚至做出错误决策。

这并非 PowerShell 本身的“bug”，而是 Windows 环境中字符编码传递链条的必然结果。本文不打算泛泛而谈 UTF-8 的重要性，而是从工程角度还原一根编码管线上可能断裂的每一环，并给出可复用的修复方案。

## 问题：中文到底怎么被“打坏”的？

假定你发出了这样一段看似正确的代码：

```powershell
$body = @{ message = "你好" } | ConvertTo-Json
Invoke-RestMethod -Uri http://localhost:8080/echo -Method Post -Body $body -ContentType "application/json; charset=utf-8"
```

在服务端用 Wireshark 抓包，你可能看到请求体竟然不是 `{"message":"你好"}`，而是 `{"message":"??"}` 或某种编码字符串。根本原因有三个层级：

1. **内存中的字符串编码**：PowerShell 5.1 底层基于 .NET Framework，字符串是 UTF-16 LE。`ConvertTo-Json` 默认会把非 ASCII 字符转义为 `\uXXXX`，但有些版本或场景下它又直接输出原始字节。当 `-Body` 参数传递一个字符串时，`Invoke-RestMethod` 必须将其转为字节流，此时它依赖 `[System.Text.Encoding]::Default` 或 `$OutputEncoding` 变量的值。

2. **`$OutputEncoding` 的陷阱**：这个变量决定了 PowerShell 将字符串发送给外部进程或 Web 请求时所使用的编码。在 Windows PowerShell 5.1 中，它的默认值通常与控制台代码页一致（如简体中文的 GBK/936）。于是，哪怕你声明了 `charset=utf-8`，实际写入网络流的还是 GBK 字节，服务端按 UTF-8 解析必然失败。

3. **响应解码**：即使请求正确，返回的 JSON 流是 UTF-8，`Invoke-RestMethod` 在反序列化时会用 `$OutputEncoding` 解码吗？不一定。它可能尊重响应头的 Content-Type，但如果返回体没有明确 charset，.NET 的 `HttpWebResponse` 可能会根据系统代码页猜测，导致中文变成乱码。更常见的是，你在终端直接 `Write-Host` 一个中文对象，而 PowerShell 控制台的字体和代码页不支持渲染，也会显示问号，但这只是显示问题，数据结构本身可能仍是正确的。

## 做法与步骤

### 1. 诊断编码链

- **抓包验证**：使用 `Invoke-WebRequest` 替换，并保存 `$request.Content` 的字节到文件，观察实际发送内容。
- **检查变量**：查看 `[System.Text.Encoding]::UTF8.GetBytes($body)` 是否与你预期的 UTF-8 字节一致。
- **确认控制台**：在 PowerShell 5.1 中运行 `chcp`，若输出 936，说明默认代码页为 GBK。但这不直接影响 API 调用，仅影响显示。

### 2. 强制 UTF-8 输出（PowerShell 5.1）

```powershell
# 在脚本顶部设置
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

这样，当 `-Body` 参数传递字符串时，`Invoke-RestMethod` 会用 UTF-8 编码发送。但注意，`ConvertTo-Json` 在 PS5.1 中可能仍会转义字符，导致中文变成 `\u4f60\u597d`。但这其实是安全的 JSON 表达，服务端如果支持 Unicode 转义也能正常解析。如果你希望直接发送原始多字节 UTF-8，需确保 `-Body` 是字节数组，而不是字符串：

```powershell
$jsonString = $body | ConvertTo-Json -Compress
$utf8Bytes = [System.Text.Encoding]::UTF8.GetBytes($jsonString)
Invoke-RestMethod -Uri ... -Body $utf8Bytes -ContentType "application/json; charset=utf-8"
```

### 3. 处理响应

```powershell
$response = Invoke-RestMethod ... -ContentType "application/json; charset=utf-8"
# 如果仍需字符串操作，显式转换
[System.Text.Encoding]::UTF8.GetString([System.Text.Encoding]::UTF8.GetBytes($response.property))
```

在 PowerShell 7（跨平台版）中，默认编码已统一为 UTF-8，但仍有坑：某些内部 API 仍然续用旧行为，所以显式指定总是更安全。

### 4. 写入文件的安全路径

当你要将 JSON 存盘给 Agent 读取时，必须避免 BOM 问题。在 PS5.1 中：

```powershell
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllLines($path, $jsonString, $utf8NoBom)
```

在 PS7 可直接用 `Out-File -Encoding utf8NoBOM`。

## 踩坑点

- **管道混用**：`$OutputEncoding` 的改变也会影响 `|` 传递给外部命令的行为，可能破坏其它脚本。建议在函数内部本地化设置，或者使用模块隔离。
- **PowerShell ISE 的欺骗性**：ISE 使用 WPF 渲染，显示中文字符可能正常，但后台编码策略与普通控制台不同，导致调试通过的脚本在计划任务或 CI 中运行失败。
- **双 Hop 编码**：如果先 `ConvertTo-Json`，又把结果赋值给变量，再通过 `-Body` 以字符串传递，而`$OutputEncoding`还是 GBK，那么原先 JSON 中的 `\uXXXX` 转义可能被当作纯文本，二次编码后彻底变形。所以最好的习惯是传递字节数组。
- **Windows 的“Beta” UTF-8 支持**：在区域设置中勾选“Beta: 使用 Unicode UTF-8 提供全球语言支持”后，系统默认代码页会变为 65001，但很多旧应用会崩溃。不建议为了脚本而去修改系统级设置。

## 可复用建议

在编写面向 OpenClaw 环境的自动化函数时，建议创建一个统一的请求模块：

```powershell
function Invoke-SafeJsonApi {
    param($Uri, $Method, $BodyObject)
    $json = $BodyObject | ConvertTo-Json -Compress
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($json)
    Invoke-RestMethod -Uri $Uri -Method $Method -Body $bytes -ContentType "application/json; charset=utf-8"
}
```

将该模块放入 Agent 的 tools 目录，所有外部调用全部走此通道，可规避 90% 的编码问题。同时，在调试时用 `$bytes | Format-Hex` 观察字节序列，事半功倍。

## 总结

Windows PowerShell 的字符编码并非不可预测，而是严格遵循 .NET 的流编码规则。当你把中文 JSON 交给 `Invoke-RestMethod` 时，它只不过是在执行一条由 `$OutputEncoding`、请求头声明、系统代码页共同决定的编码路径。理解并显式控制这条路径，就能确保中文在管道、网络和文件系统中完整无损。在 Agent 自动化场景下，编码的确定性远比看似“智能”的默认行为更重要——毕竟，AI 不怕你给它规则，就怕你给它乱码。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/8f7aeefb4d397cad.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/4ba4c8378f86af3a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/1f074d31fba018d1.png)

