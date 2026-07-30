---
title: Windows PowerShell 调用中文 JSON API：为什么你的自动化总被编码“打坏”
feedId: 31032
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景：Agent 与自动化管线里的中文“隐形炸弹”

在 OpenClaw 生态里，很多自动化节点都绕不开 PowerShell：拉取 REST API、解析 JSON、再交给下游插件或 MCP 服务器。只要 API 返回的 JSON 里带有中文字段——用户名、地址、评论——不少开发者就会碰上这样的场景：

- 控制台打印正常，但 `Set-Content` 写入文件后用记事本打开全是乱码
- `Invoke-RestMethod` 解析出的对象里，中文变成了 `?` 或 `ç` 之类
- 管道传给 Python 脚本后直接报 `UnicodeDecodeError`

在这些表象背后，几乎都是同一个根因：**PowerShell 在不同版本、不同宿主间的自动编码决策，与 JSON API 实际返回的 UTF-8 串发生了错配。**

## 问题还原：一次典型的中文 API 调用故障

以一个返回用户信息的 REST API 为例，假设 `GET https://api.example.com/user` 返回如下 JSON（Content-Type: `application/json; charset=utf-8`）：

```json
{
  "name": "张三",
  "bio": "喜欢写自动化脚本"
}
```

在 Windows 10 自带的 Windows PowerShell 5.1 下执行：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/user"
$resp.name   # 输出：??
$resp | ConvertTo-Json | Out-File user.json
```

打开 `user.json` 会发现中文全部变成问号或乱码。即使直接在控制台输出 `$resp.name`，也可能显示为 `???`，但更隐蔽的情况是：控制台看似正常，写入文件或传给其他进程后却彻底损坏。

## 为什么会这样？编码管线里的三处断裂

### 1. Invoke-RestMethod 的内部解码策略

在 Windows PowerShell 5.1 里，`Invoke-RestMethod` 收到响应字节流后，并没有严格遵循响应头里的 `charset=utf-8`。它依赖 .NET Framework 的 `HttpWebResponse`，后者在某些路径下会使用 `[System.Text.Encoding]::Default`。对于简体中文 Windows，这个默认编码是 **GBK（代码页 936）**。当服务器返回的 UTF-8 字节被按 GBK 解释时，多字节组合被拆开解析，就生成了不可恢复的乱码。这是最隐蔽的坑——它不会报错，只是把字符串静默毁掉。

### 2. PowerShell 的控制台输出与管道编码

即使对象内的字符串侥幸没被破坏，当它要显示在控制台时，PowerShell 会使用 `$OutputEncoding` 控制字符发送到管道的编码。在 PS 5.1 下，默认为 `US-ASCII`（！），任何非 ASCII 字符都会被转换为 `?`。更令人困惑的是，控制台宿主（如 `conhost.exe`）如果开启了 `[Console]::OutputEncoding` 并设为 UTF-8，可能让中文在屏幕上正确显示，但管道下游或重定向到文件依然使用 `$OutputEncoding`，导致“看着正常、实际已损坏”。

### 3. Out-File / Set-Content 的默认编码

在 Windows PowerShell 5.1 中，`Out-File` 和 `Set-Content` 的默认编码是 `Unicode (UTF-16LE)`，而不是 UTF-8。这意味着即使字符串本身正确，写入文件后也会变成 UTF-16 编码。如果用只认 UTF-8 的工具读取，同样会乱码；即使用文本编辑器打开，如果没有 BOM，也可能识别错误。同时别忘了，如果字符串在前面步骤已被破坏，任何文件操作都无法拯救。

## 工程修复步骤：让中文 JSON 稳稳落地

以下方法均在 Windows PowerShell 5.1 下验证，同样适用于更高版本。

### 方法一：接管原始字节流，显式按 UTF-8 重建

放弃 `Invoke-RestMethod` 的自动解析，手动获取字节数组并转为字符串，再解析 JSON：

```powershell
$req = Invoke-WebRequest -Uri "https://api.example.com/user" -UseBasicParsing
$rawBytes = $req.Content.ReadAsByteArray()
$utf8String = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$data = $utf8String | ConvertFrom-Json
$data.name   # 正确输出“张三”
```

这里的关键是 `Invoke-WebRequest` 的 `Content` 属性是字符串，但 .NET 内部仍可能误用编码，所以立刻拿到 `RawContentStream` 或通过 `ReadAsByteArray()` 确保按字节处理。`-UseBasicParsing` 避免在无 IE 环境触发 COM 问题。

### 方法二：强制全局编码为 UTF-8（适用于脚本化环境）

如果需要在同一脚本里多次调用 API，可以先设置相关编码变量：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

这会让 `Invoke-RestMethod` 的输出、管道编码以及所有 cmdlet 的 `-Encoding` 参数默认使用 UTF-8。注意：该方法在 PS 5.1 下对 `Invoke-RestMethod` 内部解码的改善有限，你可能仍需要结合方法一；但在 PowerShell 7 中，这三行足以根治绝大多数中文问题。

### 方法三：写入文件时显式指定编码

永远不要依赖 cmdlet 的默认编码，尤其是在跨工具链传递文本时：

```powershell
$data | ConvertTo-Json -Depth 5 |
  Set-Content -Path user.json -Encoding UTF8
```

## 踩坑点与可复用建议

- **API 响应头不可信时**：某些服务端宣称 `charset=utf-8`，但实际发送的 JSON 里含有非法的字节序列。在手动解码前，用 `$rawBytes` 检查前几个字节是否为 `0xEF 0xBB 0xBF`（UTF-8 BOM），如果有则需要跳过或使用 `StreamReader` 自动处理。
- **控制台字体缺失**：即使编码正确，控制台也可能因缺少中文字体显示为方框。可在 ConPTY 环境（如 Windows Terminal）下测试，避免被字体问题误导。
- **跨平台共用脚本**：如果脚本可能在 Linux/macOS 上的 PowerShell 7 运行，添加条件判断 `if ($IsWindows)` 来控制编码修复，避免在其他平台画蛇添足。
- **管道传给外部程序**：使用 `| & python.exe script.py` 时，务必确保 `$OutputEncoding` 与 Python 期望的 stdin 编码一致（通常设置为 UTF-8）。对于二进制数据，直接使用 `Start-Process` 或 `.NET` 的 `Process` 类绕过编码转换。

**可封装的安全 API 函数**，可在团队内复用：

```powershell
function Get-ApiJsonUtf8 {
    param([string]$Uri)
    $req = Invoke-WebRequest -Uri $Uri -UseBasicParsing
    $raw = $req.Content.ReadAsByteArray()
    $str = [System.Text.Encoding]::UTF8.GetString($raw)
    return $str | ConvertFrom-Json
}
```

## 总结

Windows 上的 PowerShell 中文 JSON 调用问题，本质是 90 年代的默认编码设计延续至今，与现代 API 普遍使用的 UTF-8 之间的矛盾。在 OpenClaw/Agent 自动化管线中，任何文本被悄悄转换成 `?` 都可能引爆下游节点。与其迷信某个 cmdlet 的“智能识别”，不如在关键路径上接管字节→字符串的转换过程，并固化编码声明。

当你的自动化脚本里出现令人费解的中文乱码时，请回溯三个检查点：**API 响应的原始解码方式、PowerShell 的管道输出编码、文件写入的编码参数**。把这三个环节紧固为 UTF-8，就能让中文无损地穿过 PowerShell 这座桥梁。

---

