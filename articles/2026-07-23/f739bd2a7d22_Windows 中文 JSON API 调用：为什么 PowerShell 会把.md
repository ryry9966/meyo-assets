---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 30127
source: 综合讨论
publishedAt: 2026-07-23
---

## 背景

在 Windows 上基于 PowerShell 做自动化时，我们经常需要通过 `Invoke-RestMethod` 或 `Invoke-WebRequest` 调用远程 JSON API，比如 OpenAI 兼容接口、自建中文知识库 API、或业务流程的 Agent 后端。只要返回的 JSON 中包含中文，就很可能出现终端打印乱码、文件存储成“锟斤拷”、管道传给其他工具时断裂。这类问题在中英文混合脚本环境中反复出现，根源并不是某个服务端返回了“坏字符”，而是 PowerShell 自己把字节序列“打坏”了。

## 问题场景

最简单的一个复现：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/items" -Method Get
$resp.title         # 应该输出“中文标题”，实际输出 "ä¸­æ–‡æ ‡é¢˜"
$resp | ConvertTo-Json | Out-File data.json
```

此时 `data.json` 里中文已经不可逆地变成了乱码。更隐蔽的是，如果你把 `$resp` 内容直接写入日志文件而不指定编码，再次读取回来时 JSON 解析直接失败，报错位置恰好是第一个中文字节。

## 为什么会这样

Windows PowerShell（5.1 及之前）运行在 .NET Framework 之上，它的编码处理与系统区域设置强绑定，而不是跟随 Unicode 规范。

- **控制台输出编码**：`[Console]::OutputEncoding` 在中文 Windows 上通常是 GBK（代码页 936），而非 UTF‑8。`Invoke-RestMethod` 返回的对象在终端显示时，字符串会被尝试用 OutputEncoding 编码，从而导致 UTF‑8 字节被错误解释。
- **偏好变量 `$OutputEncoding`**：这个变量影响 `>` 重定向和部分 cmdlet 的管道输出编码，默认也是 ASCII 或系统 ANSI 代码页，而不是 UTF‑8。当使用 `$resp | Out-File` 不显式指定编码时，文件以 UTF‑16 LE 保存（PowerShell 5.1 的默认），但更糟糕的是中间管道传输已经把中文转成了 ANSI 序列，再保存成 UTF‑16 也无法复原。
- **Invoke-RestMethod 本身的编码探测**：它会根据响应头 `Content-Type` 的 `charset` 来决定如何将字节流解码为字符串。如果服务端没有明确指定 `charset=utf-8`，PowerShell 可能回退到 ISO‑8859‑1，中文就彻底乱码。即使服务端使用了 UTF‑8 编码，若响应头里缺省了 charset，问题仍然存在。
- **JSON 解析器二次伤害**：`ConvertFrom-Json` 相对健壮，但如果你手动用字符串拼接构建 JSON 或者将乱码字符串传入，后续 `ConvertTo-Json` 也会输出乱码。

综上，这是“编码瀑布”：从网络字节流到内存字符串，再到屏幕、管道、文件的每一步都可能发生错误转换，只要有一个环节猜测编码不正确，中文就被打坏。

## 做法 / 步骤

### 1. 修复终端与控制台编码

在脚本顶部，强制把控制台和 PowerShell 内部编码统一为 UTF‑8：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

这两个变量缺一不可：前者决定 `Write-Host` 及命令返回对象的屏幕输出编码；后者决定 `>` 重定向、管道输入到外部命令（如 `findstr`、Python 脚本）时的编码。

### 2. 确保 API 响应按 UTF‑8 解码

最稳妥的办法是在请求中告知服务端，并在收到响应后手动处理字节流。对于 `Invoke-RestMethod`，可以这样做：

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/items" -Method Get
$utf8Bytes = [System.Text.Encoding]::UTF8.GetBytes($response.Content)
$correctString = [System.Text.Encoding]::UTF8.GetString($utf8Bytes)
$data = $correctString | ConvertFrom-Json
```

如果 API 是你可以控制的，务必在响应头里返回 `Content-Type: application/json; charset=utf-8`。这样 `Invoke-RestMethod` 就可以自动按 UTF‑8 处理，不需要上面这种原始解码工作。

### 3. 写入文件时显式指定编码

**永远**不要写 `> result.txt` 或 `Out-File result.json`，而是：

```powershell
$data | ConvertTo-Json -Depth 5 | Out-File -FilePath result.json -Encoding utf8NoBOM
```

在 Windows PowerShell 5.1 中，`utf8NoBOM` 可用（需要 PowerShell 版本 ≥ 5.1），否则使用 `utf8`（会带 BOM，但许多 JSON 解析器可以兼容）。如果你需要 byte-for-byte 的无 BOM UTF‑8，可以用 `[System.IO.File]::WriteAllLines` 手动控制。

### 4. 管道传给外部工具

如果你要将 JSON 字符串通过管道交给 Python、Node.js 或 `curl`，记得在调用前已正确设置了 `$OutputEncoding`。跨平台建议用 PowerShell Core（7+），它对 UTF‑8 支持更原生。

## 踩坑点

- **Out-File 默认编码**：Windows PowerShell 5.1 使用 `Unicode`（UTF‑16 LE），而不是 UTF‑8。不写参数会得到双字节编码的文件，极易造成后续处理工具误读。
- **`>` 重定向的编码与 `$OutputEncoding`**：许多教程会教你用 `>` 写文件，这在英文场景下勉强可用，一旦涉及中文，重定向会按 `$OutputEncoding` 写出 ANSI 字节，而你在 IDE 里打开时又默认以 UTF‑8 解读，看到的是混乱字符。
- **控制台本身字体问题**：如果终端字体不支持中文（例如某些旧版 PowerShell 控制台用 “Consolas”），即使编码正确也会显示方框。这时候需切换到 “新宋体” 或 “Cascadia Code” 等支持中文字形的字体。
- **BOM 争论**：有些接收 JSON 的 REST 客户端严格要求不能有 BOM。如果服务端使用了 .NET Core 的 `System.Text.Json`，它会默认拒绝 BOM。此时必须使用 `utf8NoBOM` 写文件，或 `[System.Text.Encoding]::new UTF8Encoding($false)`。
- **PowerShell Core vs Windows PowerShell**：PowerShell 7+ 已经默认将 `[Console]::OutputEncoding` 设为 UTF‑8，`$OutputEncoding` 也是 UTF‑8，很多隐式行为改善。但当你需要在 Windows 上维护旧脚本时，仍需要手动确保兼容。

## 可复用建议

1. 在所有自动化脚本的顶部加入一个 **编码初始化块**：
   ```powershell
   # 初始化 UTF-8 环境
   $script:OriginalOutputEncoding = [Console]::OutputEncoding
   $script:OriginalPowershellEncoding = $OutputEncoding
   try {
       [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
       $OutputEncoding = [System.Text.Encoding]::UTF8
   } catch {
       Write-Warning "无法设置 UTF-8 编码，中文可能显示异常"
   }
   ```
   这样即使脚本中途失败，也不会污染同一个会话的后续命令。

2. 建立一个内部编码规范：**网络请求 → 字符串 → 文件写入**，全程锁定 UTF‑8，方法就是显式用 `Out-File -Encoding utf8NoBOM`，或统一通过 `[System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($false))` 处理。

3. 在 OpenClaw 这类自动化管线中，Agent 可能通过 PowerShell 子进程执行脚本。调用方应当强制指定编码，比如启动进程时设置 `$env:LC_ALL='en_US.UTF-8'`，或为 PowerShell 加 `-Encoding utf8` 参数。

4. 如果经常需要调试 API 返回内容，可以快速检查原始字节：
   ```powershell
   $bytes = [System.Text.Encoding]::UTF8.GetBytes($rawContent)
   $bytes[0..20] -join ' '
   ```
   判断是否以常见的 UTF‑8 BOM `EF BB BF` 开头，快速排除服务端添加 BOM 的问题。

## 总结

Windows 中文 JSON API 调用的乱码，本质是 PowerShell 在多个隐式编码环节上的不一致。修复并不复杂，但需要开发者清楚每一段字符串的来龙去脉，养成显式编码的习惯。把控制台、网络响应、文件输入输出全部收敛到 UTF‑8，就能彻底杜绝“锟斤拷”这类让工程停滞的细节问题。在配置 OpenClaw、MCP 插件等自动化框架时，提前在宿主脚本中固化这些设置，远比事后到处修补要划算。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/19e1bb8a5e8b94e1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/2dee70af06d1a6e1.png)

