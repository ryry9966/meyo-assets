---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 30230
source: 综合讨论
publishedAt: 2026-07-24
---

# 问题从哪里来
在 OpenClaw 的自动化里，用 PowerShell 作为工具调用方（比如 MCP 的 stdio server 或插件执行脚本）时，一个常见的任务是调用某个中文 API 并返回结果。API 返回的 JSON 里带着中文，本地测试看起来正常，但一旦被 Agent 当作工具调用的返回值读取，就变成了 `????` 或者一团乱码。更糟的是，你甚至不确定到底是请求阶段出了问题，还是输出阶段被重定向打坏了。

下面是一次典型的故障路径：  
- 用 `Invoke-RestMethod` 拉取包含中文的 JSON；  
- 在控制台里直接 `Write-Host` 可以正常显示中文字符；  
- 把返回值交给 OpenClaw 工具输出时，中文全部变成问号；  
- 用 `>` 重定向到文件，打开发现中文是乱码。

这几乎就是 Windows 下 PowerShell 的“编码地狱”标准剧情。

# 根因简介
Windows PowerShell（5.1）默认工作在旧版控制台编码体系下，涉及三套互不通信的编码设置：
1. **控制台输出编码** `[Console]::OutputEncoding`：影响控制台如何解释写入它的字节，默认通常是系统语言对应的 OEM 代码页，比如中文 Windows 的 `936 (GBK)`。  
2. **数据流输出编码** `$OutputEncoding`：影响 PowerShell 向外发送字符串时的编码（例如传给外部命令或通过管道重定向），默认同样是 `ASCII` 或系统代码页。  
3. **脚本文件本身编码**：如果脚本是 UTF-8 without BOM，Windows PowerShell 可能以系统代码页解析，把中文注释或字符串常量直接弄脏。  

当你调用中文 API 时，服务端返回 `Content-Type: application/json; charset=utf-8`，`Invoke-RestMethod` 能正确把字节解码为 .NET 字符串（内部 Unicode 表示）。问题发生在“把这个字符串送到外界”的时刻：  
- 若用 `Write-Host`，它直接写给控制台，`[Console]::OutputEncoding` 是 GBK，就可能出现“窄化”——字符能显示是因为控制台恰好输出了对应的字节序列，但外部进程抓取时却会按照自己的预设编码解析，产生乱码。  
- 若返回值被 OpenClaw 工具通过标准输出（stdio）读取，Agent 往往期望 UTF-8 字节流，而 PowerShell 的 `>` 重定向或管道输出会受到 `$OutputEncoding` 控制，把 Unicode 字符串转换成错误的编码字节。  
- 如果你还使用了 `ConvertTo-Json` 对结果二次加工，默认 `-Depth` 小于 2 时往往不会乱码，但一旦复合对象嵌套较深，`ConvertTo-Json` 会对某些标量做奇怪的编码处理，配合输出编码，更容易出现中文丢失。

听起来复杂，实际修复并不困难，关键是抓住“输出到外部”这一步。

# 实践步骤（可复现方案）
以下按推荐顺序给出三种解法，你可以根据运行环境选择。

### 方案一：切换到 PowerShell 7+（最推荐）
PowerShell 7 （pwsh） 默认使用 UTF-8 无签名作为输出编码，且对 `$OutputEncoding` 和控制台编码处理更合理。  
```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

$r = Invoke-RestMethod -Uri 'https://api.zhihu.com/something' `
                       -ContentType 'application/json; charset=utf-8'
# 直接返回，不要用 Write-Host
$r.data.title
```
如果你的 OpenClaw agent 通过 stdio 调用这个脚本，确保外部进程捕获的是 UTF-8。在 MCP server 配置里可以这样写：
```json
"args": ["pwsh", "-NoLogo", "-NoProfile", "-Command", "& './tool.ps1'"]
```
同时需要在脚本顶部设定 `$OutputEncoding`，因为 `pwsh` 的 stdio 输出虽然默认是 UTF-8，但保险起见显式设置总是更稳。

### 方案二：完全在 Windows PowerShell 5.1 中强转编码
如果无法升级，可以这样手动守住每一环：
```powershell
# 1. 确保脚本文件保存为 UTF-8 with BOM，或使用 UTF-8 无 BOM 并在脚本头添加：
#    [System.IO.File]::ReadAllText($PSCommandPath) 等等，但更简单：用带 BOM 的 UTF-8 保存。

# 2. 设置输出编码
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

# 3. 获取 API 响应并避免后续编码污染
$resp = Invoke-WebRequest -Uri 'https://api.zhihu.com/something' -UseBasicParsing
$rawJson = [System.Text.Encoding]::UTF8.GetString($resp.RawContentStream.ToArray())
# 或者直接取 Content 属性（其内部已经是正确的 Unicode，但同样需要控制输出）
$jsonObj = $resp.Content | ConvertFrom-Json
# 此时如果需要返回子属性字符串，直接把它传给 Write-Output
Write-Output $jsonObj.message
```
关键点：不用 `Write-Host`，用 `Write-Output`，因为 `Write-Host` 绕过成功管道，其输出不被 `$OutputEncoding` 约束，在 Agent 捕获时完全可能成为乱码的源头。

### 方案三：摆脱 PowerShell 的高层封装，用 curl.exe 配合字节处理
如果只是为了做一个简单调用，可以直接用系统自带的 `curl.exe`（注意不是 `Invoke-WebRequest` 的别名），然后捕获原始输出：
```powershell
$json = & curl.exe -s -H 'Accept: application/json' 'https://api.zhihu.com/xxx'
# curl.exe 会把 stdout 字节直接输出，PowerShell 接收后转为字符串时需注意
# 在 PowerShell 5.1 中，外部命令的输出会被 $OutputEncoding 解码
# 所以必须先设置好 $OutputEncoding
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
$cleanJson = & curl.exe -s -H 'Accept: application/json' 'https://api.zhihu.com/xxx'
```
这样绕过了 `Invoke-RestMethod` 的内部处理，但文件 API 调用不再是一个 Json 对象，需要你自行 `ConvertFrom-Json`。

# 踩坑点与可复用清单
- **ConvertTo-Json 吃掉中文**：如果对象深度超过默认的 2，又有中文字符串，直接用 `ConvertTo-Json -Depth 5` 有时会在控制台输出转义，但不是乱码。确保输出时仍然用 `Write-Output`，且 `$OutputEncoding` 为 UTF-8。  
- **文件重定向符号 `>`**：它受 `$OutputEncoding` 影响。若要写入 UTF-8 文件，务必用 `$result | Out-File -Encoding utf8 data.json`，或 `Set-Content -Encoding utf8`。  
- **BOM 与 Linux Agent 兼容性**：有些 Agent 运行在 Linux 宿主上，通过 stdio 接收 PowerShell 输出时，若脚本用带 BOM 的 UTF-8 保存，可能会导致第一行出现一个不可见字符。建议用 utf8NoBOM 保存，并显式设置 `$OutputEncoding` 为无 BOM。  
- **控制台字体**：在调试时，即使编码正确，若 PowerShell 控制台字体不支持中文字符集，显示也会是方框，但这并不代表数据损坏，可以先将输出写到文件再用支持 UTF-8 的编辑器查看。

# 融入 OpenClaw/Agent 的建议
如果你正在为 Agent 编写“工具函数”（例如 MCP server 或插件脚本），可以将以下模板函数放在脚本最顶部：
```powershell
function Initialize-OutputEncoding {
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
}
Initialize-OutputEncoding
```
任何有中文输出的脚本都调用 `Initialize-OutputEncoding`。Agent 工具描述中，若工具返回的中文可能被截断或乱码，可以在描述里注明 `"encoding": "utf-8"` 或要求运行时环境使用 pwsh 7。实测在 OpenClaw 的工具调用链中，强制 stdio 模式使用 pwsh，并配合上面的初始化函数，中文 JSON API 调用就再也没出现过 `????`。

# 总结
PowerShell 在 Windows 上的中文乱码问题不是魔法，而是对输出管道编码缺乏全局管控带来的副作用。在 Agent 自动化场景中，输出被外部进程捕获，编码错误会被成倍放大。解决路径只有一条：**确保 Unicode 字符串离开 PowerShell 进程时被编码为确定且一致的 UTF-8 字节**。一旦你把这四件事对齐——脚本文件保存为 utf8NoBOM、设置 `$OutputEncoding` 与 `[Console]::OutputEncoding`、用 `Write-Output` 而非 `Write-Host`、并且在 Agent 侧声明期望 UTF-8——中文 JSON 调用就会像静默的管道一样可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/217f674dd8156a57.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/8b8b9b2e2fa827df.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/62cb2cbaec1c4fec.png)

