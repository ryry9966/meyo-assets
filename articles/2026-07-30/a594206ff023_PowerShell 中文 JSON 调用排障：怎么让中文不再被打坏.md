---
title: PowerShell 中文 JSON 调用排障：怎么让中文不再被打坏
feedId: 30958
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景：Agent 脚本里的“天书”

在 Windows 上为 OpenClaw、MCP 或自动化插件编写 PowerShell 脚本时，只要调用带中文的 JSON API，十有八九会撞上乱码。比如把用户输入的“总结一下这篇文档”封装成 JSON 发出去，结果服务器收到的却是“æ??è¦?”，响应里的中文再写回日志文件时，又变成了“???”。更麻烦的是，乱码并非每次都出现：在 IDE 里单步跑脚本可能正常，通过 agent 调度或定时任务执行时就崩了。

这类问题根因不在业务逻辑，而在 PowerShell 的默认编码行为与 JSON / Web 生态的 UTF-8 预期不匹配。下文从工程视角逐层拆解，提供可复现的修复步骤，以及一套在自动化场景里行之有效的避坑方案。

## 问题：为什么 PowerShell 常常“打坏”中文？

核心矛盾出在两个地方：

1. **脚本文件自身的编码**  
   Windows PowerShell 5.1 默认使用系统代码页（通常是 GBK/936）读取脚本文件。如果没有 BOM，就把文件内容当作 ANSI 处理。即使你保存成 UTF-8 without BOM，PS 5.1 读到的字符串就已经错了。

2. **输出管道的编码**  
   PowerShell 5.1 在将字符串输出到控制台、重定向到文件、或交给 `Invoke-RestMethod` 的 body 时，默认使用的是 **UTF-16LE**。而 JSON 标准要求 UTF-8，绝大多数 Web API 也期望 body 是 `charset=utf-8`。两边编码错配，中文字节序列被错误解码，就出现了经典的“æ‰“å”现象。

此外，`Out-File` 默认输出 Unicode (UTF-16LE)，`Set-Content` 则是 ANSI，`Invoke-WebRequest` 的 `.Content` 属性也常因响应 `Content-Type` 里缺失 charset 而误判为系统代码页。

## 可复现的错误示例

下面脚本在 Windows PS 5.1 中大概率触发乱码（假设脚本保存为 UTF-8 without BOM）：

```powershell
$body = @{ prompt = "你好，请用中文回复" } | ConvertTo-Json
$response = Invoke-RestMethod -Uri http://localhost:11434/api/generate -Method Post -Body $body -ContentType "application/json"
$response.response | Out-File output.txt
```

结果：服务器收到的 `prompt` 乱码，`output.txt` 里也是乱码。即使显式指定了 `-ContentType "application/json; charset=utf-8"`，`$body` 在传出去之前就已经是错的。

## 做法/步骤：彻底修复中文 JSON 调用

### 1. 统一脚本文件编码为 UTF-8 with BOM

在 VSCode 里点击右下角编码，选择 **“Save with Encoding” → “UTF-8 with BOM”**。  
如果必须在无 BOM 的环境下运行（比如某些 CI 系统），则避免让 PS 5.1 直接解释脚本，改为用 `pwsh` 调用。

### 2. 构建 JSON body 时强制指定 UTF-8 字节流

最稳妥的方式是直接构造 UTF-8 字节数组，然后交给 `Invoke-RestMethod`：

```powershell
$payload = @{ prompt = "你好，请用中文回复" } | ConvertTo-Json
$utf8Bytes = [System.Text.Encoding]::UTF8.GetBytes($payload)
$response = Invoke-RestMethod -Uri http://localhost:11434/api/generate -Method Post -Body $utf8Bytes -ContentType "application/json; charset=utf-8"
```

这样避免了字符串在内部转换成另一种编码。

### 3. 正确保存响应中的中文

`Invoke-RestMethod` 自动反序列化的对象没有问题，但如果需要把原始文本落盘，用 `[System.IO.File]::WriteAllText` 替代 `Out-File`：

```powershell
$result = $response.response
[System.IO.File]::WriteAllText("output.txt", $result, [System.Text.Encoding]::UTF8)
```

如果必须用 `Out-File`，记得带上 `-Encoding utf8`：

```powershell
$response.response | Out-File -FilePath output.txt -Encoding utf8
```

### 4. 修正控制台输出（可选）

如果脚本需要向控制台打印中文日志，在 PS 5.1 里可以尝试：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

但在非交互式 Shell 或远程调用中，该设置影响有限，**不能作为主要依赖**。

### 5. 换用 PowerShell 7+（推荐一步到位）

`pwsh` (PowerShell 7) 默认编码即为 UTF-8 without BOM，且 `Out-File`、`Set-Content` 等 cmdlet 也都默认使用 UTF-8，不再需要手工处理编码。唯一需要注意的是 **脚本文件编码保持一致**，保存为 UTF-8 without BOM 即可。此外，pwsh 的 `[Console]::OutputEncoding` 默认为 UTF-8，控制台中文输出也大大改善。

自动化场景下，直接指定解释器为 `pwsh.exe`：

```powershell
pwsh -File .\script.ps1
```

或者通过 OpenClaw 的 MCP 配置将 `command` 设定为 `pwsh`。

## 踩坑点：这些细节最容易栽跟头

- **PowerShell ISE 与外部执行环境不一致**：ISE 的编码处理更为宽松，脚本可能 “碰巧” 成功，但放到 agent 环境就失败。务必在目标运行环境下测试。
- **`ConvertTo-Json` 深度限制与转义**：嵌套对象中有特殊字符时，`ConvertTo-Json` 可能错误转义，配合 `-Compress` 使用效果更稳定。
- **Ollama / LM Studio 等本地 API 的 charset**：部分本地模型服务在返回头里不指定 `charset`，`Invoke-RestMethod` 可能会按 ISO-8859-1 解码，此时应直接读取字节流：`$raw = Invoke-WebRequest ...; $text = [System.Text.Encoding]::UTF8.GetString($raw.RawContentStream.ToArray())`。
- **`$PSDefaultParameterValues` 的副作用**：全局设置 `Out-File` 默认编码虽然方便，但可能影响其他模块，建议只在脚本块内局部设置。

## 可复用建议

1. **自动化脚本头部加入编码强制声明**（pwsh 兼容）：  
   ```powershell
   $PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
   $PSDefaultParameterValues['*:Encoding'] = 'utf8'   # 影响广泛，谨慎使用
   ```

2. **封装一个安全 JSON 调用函数**，统一处理编码：
   ```powershell
   function Invoke-Utf8JsonApi {
       param($Uri, $Payload)
       $bytes = [Text.Encoding]::UTF8.GetBytes(($Payload | ConvertTo-Json -Compress -Depth 10))
       Invoke-RestMethod -Uri $Uri -Method Post -Body $bytes -ContentType "application/json; charset=utf-8"
   }
   ```

3. **在团队里推广 pwsh**，废除 Windows PowerShell 5.1 执行自动化任务，从源头消除编码陷阱。

## 总结

Windows 上 PowerShell 的中文 JSON 乱码，本质是 **Windows 遗留编码习惯（UTF-16LE、ANSI 代码页）与现代 Web 生态（UTF-8 everywhere）之间的冲突**。在自动化流水线、MCP 插件等场景下，编码的入侵性远比我们想象的大。最快捷的解决路径是 **切换到 PowerShell 7，统一使用 UTF-8 编码保存和输出**；若必须留在 PS 5.1，则需要显式构造 UTF-8 字节流、强制文件写入 UTF-8，并避免依赖默认管道编码。

把这层编码习惯固化下来，中文在 JSON API 调用里就不再是“随机出现的天书”，而成为可控、可调试的数据。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/d0bf863080864c87.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/4a73f49a184537f6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/3b7b193eb1633533.png)

