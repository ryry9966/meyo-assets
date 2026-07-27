---
title: Windows 中文 JSON API 调用的隐形杀手：为什么 PowerShell 会把中文打坏
feedId: 30743
source: 综合讨论
publishedAt: 2026-07-28
---

# Windows 中文 JSON API 调用的隐形杀手：为什么 PowerShell 会把中文打坏

> 面向 OpenClaw / Agent / MCP / 自动化工程师的编码排障实录

---

## 1. 背景：当 MCP 脚本突然“失语”

很多工程师在 Windows 上用 PowerShell 构建 MCP 插件或自动化流水线时，都会遇到同一个诡异现象：明明 API 返回了正常的 JSON，用 `curl.exe` 或 Postman 看中文完好无损，可一到 PowerShell 里 `Invoke-RestMethod` 或 `Invoke-WebRequest`，再将结果 `Write-Host` 输出，或者用 `Out-File` 保存，中文字符就变成了“锟斤拷”“烫烫烫”“\\uXXXX”乱码，甚至直接变成问号。对于依赖多语言内容的 Agent 系统，这足以让整个工具链“失语”。

**问题根源根本不在 API，而在 PowerShell 的编码管道。** 这篇文章会用工程化的方式，把这条管道的“淤塞点”逐个挖出来，并给出可复用、可落到 OpenClaw 自动化实践里的修复方案。

---

## 2. 问题定位：UTF-8 的“迷路”旅程

### 2.1 看似正确的响应，却被偷偷转码

假设你的 MCP 工具调用了某个 AI 服务：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/translate" -Method Post -Body $body
Write-Host $response.data.text
```

终端上输出的中文变成了乱码。但如果你直接用 `$response` 回车查看对象，PowerShell 自己显示却是正常的——因为 PowerShell 格式化视图使用的是 `[Console]::OutputEncoding` 之外的另一套内部处理。问题往往出在“输出到文件/管道/重定向”这一步。

常见的三个淤塞点：

- **[Console]::OutputEncoding 非 UTF-8**  
  默认情况下，Windows PowerShell 5.1 的控制台输出编码是系统代码页（936 简体中文 GBK 或 65001 UTF-8，但很多机器仍是 437 OEM）。当 `Write-Host` 或脚本重定向输出时，字符串会按 `[Console]::OutputEncoding` 编码写入标准输出。如果 API 返回的是 UTF-8 字符串，而 `[Console]::OutputEncoding` 是 ASCII/GBK，那些不在其字符集的字符就会被替换为 `?`。

- **Out-File / Set-Content 默认 UTF-16 LE**  
  PowerShell 5.1 的 `Out-File` 和 `Set-Content` 默认编码是 `Unicode`（即 UTF-16 LE），而非 UTF-8。如果你用 `$response.Content | Out-File result.json` 保存 JSON，文件头会带上 BOM，并且内容编码变为 UTF-16，许多 JSON 解析器（包括 OpenClaw 自己的文本处理器）会直接报错或读出乱码。

- **Invoke-RestMethod 的字符解码与管道二次编码**  
  `Invoke-RestMethod` 或 `Invoke-WebRequest` 会尝试根据响应头 `Content-Type` 中的 charset 自动解码（通常是 UTF-8），返回的 `.Content` 属性已经是 .NET 字符串。但当这个字符串经过 PowerShell 管道，再通过 `>` 或 `|` 传递给外部程序（如 `python`、`findstr`、`Out-File`）时，PowerShell 又会用 `$OutputEncoding` 变量（默认为 ASCII！）对其进行编码，中文就此“打坏”。

### 2.2 在 OpenClaw 场景中，问题被放大

在开发 OpenClaw 插件时，我们常需要将 API 返回的 JSON 直接写入本地文件，或者作为消息块推送给 Agent。如果中间任何一步发生编码变体，Agent 收到的就是“损坏的中文”，轻则输出乱码，重则 JSON 解析失败，导致整个工作流中断。

---

## 3. 可行的修复与工程化建议

### 3.1 一次性根治三个关键编码变量

将下面四行放在脚本最顶部，适用于 **Windows PowerShell 5.1** 环境：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['*:Encoding'] = 'utf8'  # 对 Set-Content/Add-Content 等生效
```

如果使用 **PowerShell 7+**，前两行依旧有效，但 `$PSDefaultParameterValues` 设置会稍有差异，可直接在具体命令上加 `-Encoding utf8NoBOM`。

### 3.2 安全保存 JSON 的最佳实践

永远不要用 `Out-File` 或 `>` 处理 API 响应。建议直接操作流：

```powershell
$response = Invoke-WebRequest -Uri $uri -Method Post -Body $body
# 直接从 RawContentStream 写入，避免字符串转码
$fileStream = [System.IO.File]::Create("output.json")
$response.RawContentStream.CopyTo($fileStream)
$fileStream.Close()
```

或者用 `Invoke-RestMethod` 先得到对象，再通过 `ConvertTo-Json` 显式控制编码：

```powershell
$result = Invoke-RestMethod -Uri $uri -Method Post -Body $body
$json = $result | ConvertTo-Json -Depth 10
[System.IO.File]::WriteAllText("output.json", $json, [System.Text.UTF8Encoding]::new($false))
```

这里 `UTF8Encoding` 构造参数 `$false` 表示不加 BOM，兼容性最好。

### 3.3 验证编码是否生效（实用一行命令）

你可以快速验证环境状态：

```powershell
[Console]::OutputEncoding.EncodingName ; $OutputEncoding.EncodingName
```

如果输出不是 `Unicode (UTF-8)`，说明修复未完整生效。必要时可通过 `chcp 65001` 切换当前终端代码页，但仅对 CMD 窗口有效，PowerShell 仍必须依靠前文变量设置。

---

## 4. 踩坑实录与排障清单

1. **白色恐怖：ISE 与 VS Code 终端不一致**  
   Windows PowerShell ISE 拥有独立的内部宿主，`[Console]::OutputEncoding` 的设置可能被其覆盖，测试时务必使用普通终端或 VS Code 的 PowerShell 集成终端重现。

2. **BOM 的地雷**  
   即使强制 UTF-8，某些命令（如早期 `Out-File -Encoding utf8`）仍会写入 BOM。下游工具如果使用 C++ 的 `ifstream` 或 Python 的 `json.load()` 默认模式，可能会报错。使用 `utf8NoBOM`（PS 7+）或手动写入 Stream 可避免。

3. **管道到外部程序仍然可能崩溃**  
   即使设置了 `$OutputEncoding = UTF-8`，`| python script.py` 时，PowerShell 会按 `$OutputEncoding` 将字符串编码为字节流传送。但要注意，某些外部程序期望接收当前控制台的 OEM 编码而不是 UTF-8。此时可以将数据保存为临时文件再传入，或使用 Start-Process 配合重定向。

4. **JSON 反序列化的双重编码**  
   部分 API 返回的中文已经是 `\uXXXX` 转义格式，`Invoke-RestMethod` 会自动解码成可读中文。但如果你错误地做了 `ConvertTo-Json` 或再次 `Out-File`，中文可能又变成 `\uXXXX`，这并非乱码，但可读性差。正确做法是用 `-EscapeHandling` 参数控制转义。

---

## 5. 可复用模板：为 OpenClaw 定制的安全 API 调用函数

下面是一个可直接用在 MCP 插件里的骨架：

```powershell
function Invoke-SafeApiCall {
    param([string]$Uri, [hashtable]$Body)
    # 强制编码
    [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
    $OutputEncoding = [System.Text.Encoding]::UTF8

    $response = Invoke-RestMethod -Uri $Uri -Method Post -Body ($Body | ConvertTo-Json) -ContentType "application/json; charset=utf-8"

    # 安全写入 JSON
    $utf8NoBom = [System.Text.UTF8Encoding]::new($false)
    $json = $response | ConvertTo-Json -Depth 10
    [System.IO.File]::WriteAllText("$PSScriptRoot\response.json", $json, $utf8NoBom)

    return $response
}
```

如果要在日志里打印中文用于调试，请用：

```powershell
Write-Host ([System.Text.Encoding]::UTF8.GetString([System.Text.Encoding]::UTF8.GetBytes($yourString)))
```

虽然看起来滑稽，但能确保字符串先被固化到 UTF-8 字节再正确还原，绕过控制台可能存在的编码干扰。

---

## 6. 总结

Windows 上 PowerShell 的“中文乱码”绝不是玄学，它是一条包含三四个变量（`[Console]::OutputEncoding`、`$OutputEncoding`、文件输出默认编码、终端代码页）的编码管道。理解这点后，只要在脚本头部做好统一设置、在文件输出时明确编码，就能彻底终结乱码。对于 OpenClaw 或任何 Agent 自动化实践者来说，这应当是每个工程化脚本的第一段代码。

**记住：JSON 是无辜的，坏掉的只是管道**。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/ef88948d51e88d9d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/b00bd68018a44670.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/643d4ed615c965d5.png)

