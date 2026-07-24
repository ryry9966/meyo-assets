---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏——写给自动化脚本与 Agent 开发者
feedId: 30281
source: 综合讨论
publishedAt: 2026-07-24
---

## 背景

在 Windows 上搭建 OpenClaw 这类 Agent 系统时，经常会用 PowerShell 作为“胶水”来调用远程 API：可能是本地的 MCP 服务、第三方大模型接口，也可能是你自己写的插件后端。一切都很顺利，直到请求体或响应里出现了中文——客户端收到的乱码让 JSON 解析直接炸掉，Agent 的输出把“你好”变成了“ä½ å¥½”，整条链路瘫痪。

这个问题在 Windows 中文环境下的 PowerShell 5.1 中几乎必现，并且排查起来远比想象中隐蔽。本文将从一个真实发生过的调试场景出发，把原因、修复步骤和可工程化的建议一次性讲清楚。

## 问题：中文在 PowerShell 与 JSON API 之间“被打坏”的两种典型表现

**表现 1：请求体中文乱码，服务端收到损坏数据**

你用 `Invoke-RestMethod` 发送一个包含中文的 JSON 请求体：

```powershell
$body = @{ message = "你好，世界" } | ConvertTo-Json
Invoke-RestMethod -Uri $apiUrl -Method Post -Body $body `
    -ContentType "application/json"
```

服务端日志显示收到的却是 `ä½ å¥½ï¼Œä¸–ç•Œ`（典型的 UTF-8 字节被按 Latin-1 解释后的结果）。API 返回 400，因为 JSON 本身已经非法。

**表现 2：响应体中文乱码，脚本处理出错**

API 返回的 JSON 中包含中文字段，例如 `{"status": "已完成"}`，但你在控制台看到的 `"status"` 值是 `"å·²å®Œæˆ"`。不仅日志不可读，后续用 `ConvertFrom-Json` 解析出来的对象也是乱码，Agent 据此做出的决策自然完全错乱。

## 核心原因：PowerShell 在 Windows 上的编码堆栈不一致

PowerShell 5.1 在 Windows 上处理字符串时，会涉及多个不同的编码设置，而且它们的默认值高度依赖系统区域（System Locale）。中文版 Windows 的“非 Unicode 程序语言”是简体中文（代码页 936）。

关键变量：

- `[System.Text.Encoding]::Default`：这是 .NET 框架在没有显式指定编码时的 fallback。在中文 Windows 上是 GBK（CP936）。
- `$OutputEncoding`：PowerShell 向外部进程（例如 Python、curl）发送管道内容时使用的编码。默认通常是 ASCII（US-ASCII）或跟随控制台代码页。
- `Invoke-RestMethod` / `Invoke-WebRequest` 的 `-ContentType`：当你只写 `-ContentType "application/json"` 而没有 `charset=utf-8` 时，底层会使用 `[System.Text.Encoding]::Default` 去编码请求体。
- 控制台字体：只是显示问题，但也容易干扰判断。

当 PowerShell 执行 `$body = @{ message = "你好，世界" } | ConvertTo-Json` 时，字符串本身是 .NET 的 Unicode 字符串，一切正常。问题出现在 `Invoke-RestMethod -Body $body` 这一步：cmdlet 将字符串转换为字节流时，默认编码是 `[System.Text.Encoding]::Default`——在中文 Windows 上就是 GBK。于是你的 UTF-8 友好的 JSON 字符串被重新编码成了 GBK 字节流，再被服务端按 UTF-8 解释，直接乱码。响应乱码也是同理：HTTP 响应字节流被按默认编码解码，而不是按服务器返回的 `Content-Type` 中声明的 `charset=utf-8`。

## 具体修复步骤（已验证可复现）

### 1. 强制使用 UTF-8 编码发送请求体

最简单的方法是将 JSON 字符串先转为 UTF-8 字节数组，再用 `-Body` 传递，同时明确告知服务器编码：

```powershell
$json = @{ message = "你好，世界" } | ConvertTo-Json -Compress
$utf8Bytes = [System.Text.Encoding]::UTF8.GetBytes($json)
Invoke-RestMethod -Uri $apiUrl -Method Post -Body $utf8Bytes `
    -ContentType "application/json; charset=utf-8"
```

如果你的 PowerShell 版本支持 `-Encoding` 参数（PS 6+），也可以直接：

```powershell
Invoke-RestMethod -Uri $apiUrl -Method Post -Body $json `
    -ContentType "application/json; charset=utf-8" -Encoding UTF8
```

但在 Windows PS 5.1 中 `-Encoding` 参数不起作用或不存在，所以字节数组方案是最稳妥的。

### 2. 修复响应解码

对于响应乱码，我们可以手动获取原始字节流，再用 UTF-8 解码：

```powershell
$response = Invoke-WebRequest -Uri $apiUrl -Method Get
$rawBytes = $response.RawContentStream.ToArray()
$jsonString = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$obj = $jsonString | ConvertFrom-Json
```

### 3. 全局设置（治本手段）

将下面三行放在脚本或 `$PROFILE` 的最前面，可以避免大部分自动化任务中的编码坑：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
[System.Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

- `$OutputEncoding` 影响向外部进程发送数据的编码，尤其当你用管道将 JSON 传给 `curl` 或 `python` 时。
- `[System.Console]::OutputEncoding` 确保控制台输出本身是 UTF-8（仍需字体支持）。
- `$PSDefaultParameterValues` 让所有支持 `-Encoding` 的 cmdlet（如 `Out-File`、`Set-Content`）默认使用 UTF-8。

## 踩坑点

**坑 1：PowerShell ISE 与普通控制台的行为不同**

ISE 内部使用的是 WPF 文本框，其对 UTF-8 的渲染会略好于传统 conhost，但编码处理逻辑完全一样。很多开发者在 ISE 里看着没问题的中文，一部署到真实 agent 后台的 PowerShell 窗口就乱码。**一切以无头（headless）运行环境为准**，不要依赖 ISE 的显示“假象”。

**坑 2：`Invoke-RestMethod` 的自动反序列化可能掩盖问题**

如果你的 API 返回的是简单的 JSON 对象，`Invoke-RestMethod` 会自动将其转为 `PSCustomObject`。这个对象里的字符串已经经历了从字节到 .NET 字符串的解码过程，如果这一步已经错了，你再复制、打印、传参都是错的。**不要只是看终端输出**，要用 `-OutFile` 保存原始响应文件或检查 `RawContentStream`。

**坑 3：Agent 链路中的多处编码转换**

在 OpenClaw 或 MCP 自动化场景中，经常是“Agent → MCP Client → PowerShell 脚本 → API”。如果客户端或中间件有自己的一套字符串处理逻辑（例如 Node.js 的 `Buffer` 或 Python 的 `subprocess`），可能会引入第二次编码破坏。出现中文乱码时，要在链路的每一段分别抓包确认，通常第一处破环就在 PowerShell 的这端。

## 可复用建议

1. **永远不要信任默认编码。** 在 Windows 上写任何与外部系统交互的 PowerShell 脚本，开头就设置 `$OutputEncoding` 和 `$PSDefaultParameterValues`，形成团队规范。
2. **优先使用 `curl.exe`（Windows 10+/Server 2019+ 内置的真正的 curl）替代 `Invoke-RestMethod`。** `curl.exe` 内部处理 UTF-8 比 PS 更直接，且与 Linux 行为一致，尤其适合跨平台 MCP 工具。
3. **在 JSON 中保留 BOM？千万不要。** PowerShell 的 `Out-File -Encoding Unicode` 等选项会写 BOM，绝大多数 JSON 解析器会崩溃。只使用 `-Encoding UTF8NoBOM`。
4. **为 Agent 的脚本编写 check-health 子命令。** 启动时调用一个已知返回中文的端点，校验返回值，如果检测到乱码立即报警并中止，避免静默产生错误决策。
5. **文档化你的编码策略。** 在插件或 MCP 工具的 README 中明确声明脚本要求 UTF-8 环境，并提供上述的配置代码片段，降低使用者的踩坑成本。

## 总结

PowerShell 在 Windows 上把中文 JSON 打坏的根本原因是：**自动化脚本默认的编码路径与 JSON 标准所要求的 UTF-8 不兼容**。这既是 Windows 历史包袱，也是我们写跨语言自动化时必须面对的工程现实。只需三行全局设置加上字节数组传输这个小习惯，就足以让 90% 的中文乱码问题消失。

对于正在构建 Agent、插件或 MCP 服务的你，今天花 15 分钟把编码问题彻底收拾干净，未来就能避免无数次凌晨排查乱码的痛苦。

---

## 配图

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/cb37c704b7a00eaf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/4abe6e8ae2f38014.png)

