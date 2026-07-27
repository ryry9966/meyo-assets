---
title: 为什么 PowerShell 会把中文 JSON 打坏？Windows 下 API 调用的编码陷阱与工程化修复
feedId: 30698
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景

在 Windows 上使用 PowerShell 调用中文 API、处理返回的 JSON 是许多自动化、OpenClaw 插件或 Agent 工作流的常态。一个简单的任务是：获取某个中文 API 的 JSON 响应，保存为 JSON 文件，供后续工具读取。但经常出现一种诡异现象：API 返回时中文看起来正常，保存到文件、再读取时却变成了一堆“锟斤拷”或乱码方块。排查一圈发现——不是 API 的问题，而是 PowerShell 自己把中文“打坏”了。

## 问题复现

假设我们调用一个返回中文内容的 API：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/v1/items/zh"
$resp.data.name  # 终端输出可能正常显示“中文测试”
```

如果直接使用 `Out-File` 或 `Set-Content` 保存 JSON：

```powershell
$resp | ConvertTo-Json -Depth 5 | Out-File -FilePath "data.json"
```

再次用编辑器打开 `data.json`，或使用 `Get-Content` 再 `ConvertFrom-Json` 解析，中文全部变成乱码。

更隐蔽的情况是，通过重定向 `>` 保存时也会损坏：

```powershell
$resp | ConvertTo-Json -Depth 5 > data.json
```

## 根本原因

PowerShell 5.1（Windows 预装版本）的文件输出行为与多数 Unix 工具完全不同：

1. **`Out-File` 的默认编码是 Unicode (UTF-16LE)**，而不是 UTF-8。
2. **`Set-Content` 的默认编码取决于系统 ANSI 代码页**（如简体中文是 GBK），保存 UTF-8 内容时会用 ANSI 编码器重新编码，中文自然被破坏。
3. **重定向运算符 `>` 本质上是 `Out-File` 的别名**，同样输出 UTF-16LE。
4. `ConvertTo-Json` 生成的是 UTF-8 字符串，但一旦流入这些默认管道，就会被二次编码，形成错误字节序列。后续用 `Get-Content` 读取时，若未指定 `-Encoding UTF8`，会按默认编码解读，展示出乱码。

PowerShell 的 `[Console]::OutputEncoding` 和 `$OutputEncoding` 也会影响字符串在控制台和管道间的传输，但文件输出的编码决策主要来自 cmdlet 的 `-Encoding` 参数和默认值。如果脚本需要跨工具、跨平台读取（例如 Linux 上的 OpenClaw 或后续 Node.js Agent），UTF-16LE 或 GBK 文件基本无法使用。

## 修复步骤

### 1. 显式指定文件编码为 UTF-8

最直接的方法是在写出文件时使用 `-Encoding UTF8` 或带 BOM 的 `Default`（在 PowerShell Core 7+ 中 `-Encoding UTF8NoBOM` 更合适）：

```powershell
$resp | ConvertTo-Json -Depth 5 | Out-File -FilePath "data.json" -Encoding UTF8
```

或使用 `Set-Content`：

```powershell
$jsonStr = $resp | ConvertTo-Json -Depth 5
Set-Content -Path "data.json" -Value $jsonStr -Encoding UTF8
```

### 2. 全局修改输出编码（仅限 PowerShell 5.1）

如果不希望每个命令都加 `-Encoding`，可以在脚本顶部强制将文件输出重定向行为改为 UTF-8。注意这会影响所有管道输出：

```powershell
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['*:Encoding'] = 'utf8'  # 对多个 Write-* cmdlet 生效
```

但这样做可能在某些环境下引入 BOM，且 `Set-Content` 等仍可能不适用，需谨慎测试。

### 3. PowerShell Core 7+ 的默认行为

在 PowerShell 7+ 中，`Out-File` 默认编码已改为 UTF-8（无 BOM），而 `Set-Content` 的默认仍然依赖 ANSI。因此统一使用 `Out-File -Encoding UTF8NoBOM` 或直接写 `[System.IO.File]::WriteAllText("data.json", $jsonStr, [System.Text.UTF8Encoding]::new($false))` 更为可靠。

## 踩坑点

- **BOM 问题**：Windows 版 PowerShell 的 `-Encoding UTF8` 会写入 UTF-8 BOM（字节序标记）。许多 Unix 工具和 JSON 解析器对 BOM 敏感，可能导致解析失败。如果目标消费方在 Linux，推荐使用 `UTF8NoBOM`（PS Core）或 .NET 方法手动控制。
- **重定向陷阱**：脚本中若出现 `something > file`，很容易忘记它仍使用旧编码。可改成 `something | Out-File -FilePath file -Encoding utf8`。
- **$OutputEncoding 的误导**：这个变量影响的是与外部程序（如 Python、curl）管道交互时的字符串编码，不改变 PowerShell 内部文件输出。曾经有人在排查文件乱码时被误导去修改它，问题仍未解决。
- **Invoke-RestMethod 可能已正确解码**：API 返回的 UTF-8 JSON 经 `Invoke-RestMethod` 会自动解析为对象，此时中文已经正确存储在 .NET 字符串中。乱码仅发生在文件写出或再次编码环节，因此不必怀疑获取过程。
- **跨平台 CI/CD 环境**：在 GitHub Actions 的 Windows runner 上，默认 PowerShell 为 5.1，若未指定编码，产出的 JSON 文件在后续 Linux runner 中读取必然乱码。必须显式设定 UTF8。

## 可复用建议

1. **在脚本开头统一编码配置**：
   ```powershell
   $OutputEncoding = [System.Text.Encoding]::UTF8
   $PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
   ```
   并注意区分 PowerShell 版本，在 7+ 中推荐 `utf8NoBOM`。

2. **使用 .NET 静态方法处理关键文件**：对于最终被 Agent 或 OpenClaw 读取的 JSON，采用 `[System.IO.File]::WriteAllText` 写入，并明确指定 `UTF8Encoding` 无 BOM 实例，这能完全脱离 PowerShell 的编码假设。

3. **验证文件编写后的完整性**：在脚本中加入检查步骤，读取刚写出的 JSON 并解析，确保 `name` 等字段中文可读。

4. **在文档和团队公约中强调“永远不要使用 `>` 保存文本数据”**，尤其是需要跨平台共享时。

## 总结

Windows PowerShell 的默认文件编码是自动化场景中最隐蔽的“中文杀手”。多数问题集中在 `Out-File` 和重定向的 Unicode 遗留行为，以及 `Set-Content` 的 ANSI 策略。通过在每次写出 JSON 时显式指定 UTF-8 编码，即可彻底解决。面向 Agent、OpenClaw 等跨平台消费场景，建议直接使用 .NET 方法输出无 BOM 的 UTF-8 文件，并做好编码验证。把编码当成基础设施级别的强制约定，远比出现乱码后回头排查成本更低。

---

