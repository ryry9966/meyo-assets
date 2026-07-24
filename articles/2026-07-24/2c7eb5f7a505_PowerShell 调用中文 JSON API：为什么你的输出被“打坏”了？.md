---
title: PowerShell 调用中文 JSON API：为什么你的输出被“打坏”了？
feedId: 30287
source: 综合讨论
publishedAt: 2026-07-24
---

## 背景

在 OpenClaw 的自动化流程里，Agent 经常需要调用第三方 API 拉取中文数据，再做解析或入库。很多同学习惯在 Windows 上用 PowerShell 写轻量脚本，因为不依赖额外运行环境。但一条简单的 `Invoke-RestMethod` 返回的 JSON，只要包含中文，就可能在控制台变成方框、问号或彻底乱码；重定向到文件，下游解析直接失败。

这不是 API 的锅，而是 **Windows 下 PowerShell 的编码历史债务** 在作祟。理解并修好它，才能让 Agent 链路可靠运行。

## 问题根源：三层编码错配

一个典型的中文 JSON API 响应链路是这样的：

```
服务器 (UTF-8 字节流) → .NET HttpClient (已解码为字符串) → PowerShell 对象 → 控制台 / 文件
```

错配发生在最后两步：

1. **`Invoke-RestMethod` / `Invoke-WebRequest` 其实拿到了正确的 Unicode 字符串**。因为底层 `HttpClient` 会根据响应头 `charset` 自动解码，API 通常声明 `Content-Type: application/json; charset=utf-8`，所以变量里的中文是好的。

2. **控制台渲染时，编码跟字符不匹配**。Windows 传统控制台（conhost）默认代码页是系统 ANSI 编码，中文 Windows 下是 GBK（936）；而 PowerShell 5.1 的 `$OutputEncoding` 默认也是 ASCII。当对象被打印到控制台，PowerShell 会用 `$OutputEncoding` 编码后发送给控制台，控制台再按自己的代码页解码，中文就这样被“打坏”了。

3. **输出到文件时隐式转码**。`>` 重定向会使用 `$OutputEncoding` 写入文件；`Set-Content` 默认用 `Default`（ANSI）；`Out-File` 默认用 Unicode (UTF-16 LE)，如果不指定 `-Encoding`，都可能破坏中文。

图示一下错配链：

```text
API 返回 UTF-8 → PS 字符串正常
                ↓ 打印到控制台
          $OutputEncoding (ASCII) 编码
                ↓
        控制台代码页 (936) 解码 → 乱码
                ↓ 写入文件
          Out-File -Encoding Unicode → UTF-16 LE
          Set-Content → ANSI (GBK)   → 原 UTF-8 内容错位
```

## 修复步骤

### 1. 立即生效：设置控制台与输出编码

在脚本开头加上：

```powershell
# 控制台代码页改为 UTF-8
chcp 65001 > $null
# PowerShell 输出编码设为 UTF-8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

对 `Invoke-RestMethod` 返回的对象，直接打印就能正常显示中文。

### 2. 持久化到文件：显式指定 UTF-8

```powershell
$response = Invoke-RestMethod -Uri 'https://api.example.com/data' -ContentType 'application/json; charset=utf-8'
# 保存为 UTF-8 without BOM（多数 Linux 工具链偏好）
$response | ConvertTo-Json -Depth 10 | Out-File -FilePath 'data.json' -Encoding utf8NoBOM
```

如果希望带 BOM（某些 Windows 编辑器识别），改用 `-Encoding utf8`。

### 3. 避免隐式转码的坑

- **不要用 `>` 重定向**，它会受 `$OutputEncoding` 影响，且编码不可控。
- **避免 `Set-Content` 不加 `-Encoding`**，它会使用系统默认 ANSI，中文 Windows 下变成 GBK。
- **管道里的 `ConvertTo-Json`**：如果对象中字符串正常，序列化后仍是正确的 Unicode，但输出到文件时必须配合 `Out-File -Encoding`。

### 4. 终极方案：切换到 PowerShell 7

PowerShell 7 (Core) 在所有平台上默认使用 UTF-8，且 `$OutputEncoding` 默认为 UTF-8。控制台代码页也会自动适配。大多数乱码问题在 PS7 上自然消失。Windows 下可通过 `winget install Microsoft.PowerShell` 安装，然后在脚本中用 `pwsh.exe` 执行。

## 踩坑记录

- **“变量里中文没问题，但控制台就是乱码”**  
  这是因为控制台代码页未改为 UTF-8。还有些终端（如旧版 cmd）不支持 UTF-8，即使 `chcp 65001` 也没用。建议用 Windows Terminal + 支持的字型（如 Cascadia Code）。

- **`Invoke-WebRequest` 返回的 `Content` 是字节数组？**  
  `Invoke-WebRequest` 的 `Content` 属性是原始字节，如果你手动将其转字符串，必须用 `[System.Text.Encoding]::UTF8.GetString($resp.Content)`，否则会按 `Default` 解码。

- **`ConvertTo-Json` 转义了特殊字符**  
  有的 API 返回的 Unicode 字符（如 emoji）被转成 `\uXXXX`，如果想保留原字符，可用 `-EscapeHandling` 参数（PS 7 支持），或在后续处理中自行反转义。

- **在 Agent 的 `shell` 步骤中乱码**  
  如果 OpenClaw Agent 通过 `shell` action 调用 PowerShell 脚本，务必在脚本内显式设置编码，因为 Agent 启动的子进程可能没有继承当前终端的 UTF-8 设置。

## 可复用建议

封装一个安全函数，自动处理编码和保存：

```powershell
function Invoke-SafeJsonApi {
    param(
        [string]$Uri,
        [string]$OutFile,
        [int]$Depth = 10
    )
    # 强制设置编码
    $prevOutputEncoding = $OutputEncoding
    $prevConsoleCP = [Console]::OutputEncoding
    try {
        chcp 65001 > $null
        $OutputEncoding = [System.Text.Encoding]::UTF8
        [Console]::OutputEncoding = [System.Text.Encoding]::UTF8

        $result = Invoke-RestMethod -Uri $Uri -ContentType 'application/json; charset=utf-8'
        if ($OutFile) {
            $result | ConvertTo-Json -Depth $Depth | Out-File -FilePath $OutFile -Encoding utf8NoBOM
        }
        return $result
    } finally {
        $OutputEncoding = $prevOutputEncoding
        [Console]::OutputEncoding = $prevConsoleCP
    }
}
```

在 Agent 脚本或计划任务里统一使用此类函数，能避免大量潜在崩溃。

## 总结

Windows 下 PowerShell 处理的不是中文本身有问题，而是 **默认编码假设与 UTF-8 时代脱节**。只要在交互式运行或脚本入口显式设定三层编码——控制台代码页、`$OutputEncoding`、文件输出编码，整个链路就能保持中文无损。对自动化实践来说，这一点稳定性的提升远比反复 debug 控制台乱码划算。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/477d69ed611225ab.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/068c45d29ffec1b6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/09e09aea474a6aef.png)

