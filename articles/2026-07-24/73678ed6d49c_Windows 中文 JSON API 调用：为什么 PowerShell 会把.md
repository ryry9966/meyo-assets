---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏 — 根源、修复与工程实践
feedId: 30265
source: 综合讨论
publishedAt: 2026-07-24
---

## 背景：你的“自动化助手”突然不会说中文了

在 MCP 服务器、OpenClaw 插件或任何需要从 Windows 脚本调用外部 API 的自动化场景里，我们常常会用 `Invoke-RestMethod` 或 `Invoke-WebRequest` 获取 JSON 数据。一切看似正常，直到响应中包含中文字符：控制台打印正常，但一旦把结果写入文件、传入下游工具，或者作为 MCP 返回值，中文就变成了 `????`、`涓枃` 之类的天书。

更令人困惑的是：同一个脚本，在同事的 Windows Terminal 里就跑得好好的，到自己机器上就“打坏中文”。问题根源并不在 API 本身，而在 PowerShell 与 Windows 控制台对编码的默认假设。

## 问题定位：三层编码冲突

要理解“为什么打坏”，必须先看清数据流：

1. **API 返回的原始字节**：绝大多数现代 API 返回 `Content-Type: application/json; charset=utf-8`，响应体是标准的 UTF-8 字节流。
2. **PowerShell 的解码行为**：`Invoke-RestMethod` 会根据响应的 `charset` 自动将字节流解码为 .NET 字符串。这一步通常没问题，变量里已经保存了正确的 Unicode 字符串。
3. **输出到控制台或文件**：当字符串离开进程——例如打印到控制台、重定向到文件、通过管道传给外部程序——就会触发 **输出编码转换**。

真正的问题就出在第三步。PowerShell 5.1（Windows 内置版本）的 `[Console]::OutputEncoding` 默认是系统代码页，简体中文 Windows 通常是 GBK（936）。如果 API 返回的字符串被复制到控制台缓冲区时，系统会尝试用 GBK 进行编码，而 GBK 无法表达某些 Unicode 字符，于是触发“最佳匹配”替换，造成不可逆的乱码。即使只是用 `> result.json` 重定向到文件，也可能因为 `$OutputEncoding` 或 `Out-File` 的默认编码导致损坏。

更隐蔽的是，在某些终端（如自带的 `conhost.exe`）中看起来“正常”，是因为终端的渲染层做了修正，但实际写入文件的数据已经损坏。当你把同一个内容通过标准输出传到另一个进程（例如 MCP 的 stdio 传输），下游收到的就是损坏的字节流。

## 做法：一步一步根除乱码

以下步骤适用于 **PowerShell 5.1** 和 **PowerShell 7+**，在 Windows 10/11 上验证通过。

### 1. 根治控制台输出编码
在脚本最开头强制设置输出编码为 UTF-8：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

- `[Console]::OutputEncoding` 控制控制台显示的编码和 `>` 重定向输出时的编码。
- `$OutputEncoding` 控制管道传递给外部程序时的编码（例如 `| Out-File`）。

在 PowerShell 7 中，还可以同时设置环境变量建议外部程序使用 UTF-8：

```powershell
$env:LC_ALL = 'C.UTF-8'
```

但最关键的仍是前两项。

### 2. 保持原始字节流，避免中间转换
如果需要将 API 响应直接保存为 JSON 文件，**不要** 先解码为字符串再写回，应直接保存原始字节：

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/data" -UseBasicParsing
[System.IO.File]::WriteAllBytes("result.json", $response.Content.Raw)
```

或直接使用 `Invoke-RestMethod` 的 `-OutFile` 参数：

```powershell
Invoke-RestMethod -Uri "https://api.example.com/data" -OutFile "result.json"
```

这样做可以完全绕过输出编码转换，文件内容与服务器返回的原始 UTF-8 字节一致。

### 3. 显式指定文件写入编码
如果必须对字符串做处理后再保存，请始终显式指定 UTF-8 编码：

```powershell
$data | ConvertTo-Json -Depth 10 | Out-File -FilePath "output.json" -Encoding utf8NoBOM
```

避免使用 `-Encoding utf8`（会带 BOM，某些 JSON 解析器可能报错），推荐使用 `utf8NoBOM`。如果你使用的 PowerShell 版本低于 6.0，则使用：

```powershell
$Utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllLines("output.json", $data, $Utf8NoBom)
```

### 4. 在 MCP / Agent 场景中强制 stderr / stdout 编码
如果脚本作为 MCP 服务器的子进程运行，父进程读取的是标准输出流。此时尤其需要确保 `[Console]::OutputEncoding` 已设为 UTF-8，因为 PowerShell 会将 .NET 字符串按此编码写入 stdout。在入口脚本的开头就设置，并在写入 JSON 时直接用 `Write-Output` 而不是 `Write-Host`。

如果你的 MCP 客户端使用 Node.js 的 `child_process.spawn`，也应在 Windows 上传递 `stdio: ['pipe', 'pipe', 'pipe', 'utf8']` 选项。

## 踩坑点清单

- **PowerShell ISE 的幽灵行为**：ISE 环境可能无视 `[Console]::OutputEncoding` 的修改，因为它不是真正的控制台窗口。务必在 `pwsh.exe` 或 Windows Terminal 中测试。
- **不完整的修复**：仅设置 `$OutputEncoding`，而忘记 `[Console]::OutputEncoding`，会导致 `>` 重定向仍输出 GBK 损坏数据。
- **BOM 困扰**：`Out-File -Encoding utf8` 会带 BOM，某些 JSON Schema 校验会失败。坚持使用 `utf8NoBOM` 或无 BOM 的字节写入。
- **`Invoke-WebRequest` 和 `Invoke-RestMethod` 差异**：前者返回原始字节更方便保存；后者自动解析对象，但在序列化回去时也可能引入转换。对直接保存的场景优先使用 `-OutFile`。

## 可复用建议：封装一个防乱的 API 调用函数

工程中可以把最佳实践固化：

```powershell
function Get-ApiJsonSafe {
    param([string]$Uri, [string]$OutFile)
    if ($OutFile) {
        Invoke-RestMethod -Uri $Uri -OutFile $OutFile
    } else {
        $response = Invoke-WebRequest -Uri $Uri -UseBasicParsing
        $Utf8NoBom = [System.Text.UTF8Encoding]::new($false)
        $json = $Utf8NoBom.GetString($response.Content.Raw)
        return $json | ConvertFrom-Json
    }
}
```

在脚本或模块中统一调用该函数，杜绝手动编码处理。

## 总结

PowerShell 把中文 JSON 打坏的根因，是 Windows 控制台系统默认使用旧代码页输出文本，而现代 API 清一色使用 UTF-8。三层编码不匹配最终在输出边界爆发，而非 API 解析阶段。解决之道不是到处修补，而是**主动将输出环境设置为 UTF-8，并尽量直接操作原始字节流**。

在 OpenClaw 这类以自动化流转为核心的工具链里，任何一处字符串损坏都可能传递到整个 Agent 管线，导致下游解析失败、日志不可读。从源头把编码问题约束在“脚本最开始 3 行”，是投入产出比最高的工程习惯。

记住这组最小动作：
```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
Invoke-RestMethod -OutFile data.json  # 直接存文件
```

此后，中文再也不会在你的 API 调用中凭空消失。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/fefcc7af19ec5153.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/e0580f3283e72f58.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/6f640f42acd73e43.png)

