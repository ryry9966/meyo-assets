---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文「打坏
feedId: 30732
source: 综合讨论
publishedAt: 2026-07-28
---

你给 OpenClaw 搭了一个本地 Agent，用 PowerShell 写了个脚本去请求某个返回 JSON 的 API，一切看起来都很正确，直到终端里本应出现的“处理成功”变成了“??????”或者一串 `\u6210\u529f` 的 unicode 转义。一通排查后发现 —— API 本身没问题，curl 正常，Python 正常，唯独 PowerShell 把中文打坏了。

这个场景在 Windows 的 Agent / MCP / 插件自动化中极其常见，本质不是某个服务的 bug，而是 PowerShell 和 Windows 控制台编码长期存在的“水土不服”。我会尽量克制地讲清楚根因，并给出可工程化复用的解法。

## 背景：一次最简的“翻车”复现

假设你有一个 Agent API，返回内容为：

```json
{"status": "成功", "message": "任务已下发"}
```

最简单的 PowerShell 调用：

```powershell
$resp = Invoke-RestMethod -Uri http://127.0.0.1:8149/api/echo
$resp.message
```

Windows 终端（PowerShell 5.1，默认代码页 936/GBK）中直接输出，大概率显示为 `???` 或某些不可见字符。如果进一步将结果导出：

```powershell
$resp | ConvertTo-Json | Out-File result.json
```

得到的 `result.json` 里中文也成了问号或乱码，后续再被 Agent 解析就会崩溃。

## 问题根因：三条线交叉破坏

本质上，有三条编码线在博弈：

1. **API 响应的真实编码**：绝大多数现代 API 使用 UTF-8，响应头中可能携带 `Content-Type: application/json; charset=utf-8`。
2. **PowerShell 管道内的字符串处理**：`Invoke-RestMethod` 内部会将响应流转成 .NET 字符串，这个过程信赖了被“污染”的解码路径。在 Windows PowerShell 5.1 中，当未明确指定字符集时，它可能回退到 `[System.Text.Encoding]::Default`（即系统活动的 ANSI 代码页，943/936 等）。于是 UTF-8 字节被当成 GBK 解释，中文损坏。
3. **输出编码**：即使内存中的字符串已经是正确的 .NET 字符串（Unicode），当写入控制台时，PowerShell 主机又会试图将 Unicode 映射到当前控制台代码页。如果代码页是 936，不能表示的字符就会被替换为问号。同样的，`Out-File` 默认使用 UTF-16 LE 或带 BOM 的 UTF-8？实际上，在没有指定 `-Encoding` 时，PowerShell 5.1 的 `Out-File` 默认使用 Unicode（UTF-16 LE），这倒不会丢字符，但如果用了 `>>` 重定向，那是 `Out-File -Append` 的别名，编码可能变成 ASCII 或 OEM，导致进一步损毁。最坑的是 `Set-Content` 和 Write-Output 重导向到文件，它们会使用目标文件系统或环境的 ANSI 编码。

结果就是：从网络到内存，再从内存到文件/屏幕，任何一个环节采用了非 UTF-8 的假定，中文就“打坏”了。

## 可复现的修复步骤

### 1. 确认当前的编码陷阱

在脚本开头打印当前代码页：

```powershell
chcp
```

通常返回 `936`（简体中文 GBK）。除非你在终端属性里勾选了“使用 Unicode UTF-8 提供全球语言支持”（Windows 10 1903+），否则默认都不是 UTF-8。

### 2. 从源头控制 HTTP 调用的解码

最稳妥的办法：不要依赖 `Invoke-RestMethod` 的自动解码，而是手动用 `Invoke-WebRequest` 获取原始字节，再显式用 UTF-8 解码。

```powershell
$response = Invoke-WebRequest -Uri http://127.0.0.1:8149/api/echo -UseBasicParsing
$rawBytes = $response.RawContentStream.ToArray()
$jsonString = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$data = $jsonString | ConvertFrom-Json
$data.message   # 正确显示中文
```

**踩坑点**：`Invoke-WebRequest` 的 `.Content` 属性已经是被解码后的字符串，用的编码可能依旧不对。必须使用 `.RawContentStream` 拿到原始字节。

### 3. 设置全局输出编码（仅作为辅助）

可以将整个脚本的输出编码全部切到 UTF-8：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

这能缓解控制台显示问题，但无法解决 `Invoke-RestMethod` 解码阶段的问题（因为那时还没到输出环节）。在 PowerShell 5.1 中，要彻底根治，最好改用 PowerShell 7+，它默认将所有外部命令和文件读写视为 UTF-8，并且控制台交互的编码配置更为一致。

### 4. 安全写入文件

需要持久化 JSON 时，强制使用 UTF-8 无 BOM：

```powershell
$data | ConvertTo-Json -Compress | Out-File -FilePath clean.json -Encoding utf8NoBOM
```

或者使用 `[System.IO.File]::WriteAllText("clean.json", $data | ConvertTo-Json, [System.Text.UTF8Encoding]::new($false))`，可以完全避免 BOM 和 PowerShell 管道的二次破坏。

## 踩坑汇总

- **`Invoke-RestMethod` 直接当成对象用很方便，但会隐藏编码问题**。一旦 API 没有明确写 `charset` 或是老旧的 Windows 服务，中文就坏在半路上。
- **`>>` 和 `Set-Content` 的默认编码是 OEM/ANSI**，在中文 Windows 上是 GBK，会将已正确的 Unicode 字符串再次“降级”为 GBK 并丢失字符。
- **`ConvertFrom-Json` 本身是安全的**，前提是传入的字符串已经是正确的 Unicode。问题往往出在获取字符串的步骤上。
- **PowerShell 5.1 与 7 的行为差异巨大**。如果你的 Agent 脚本可以强制指定用 `pwsh.exe` 运行，会省去很多麻烦。团队内部可以约定脚本的第一行是 `#Requires -Version 7`。

## 可复用建议

给所有需要从 HTTP API 获取中文 JSON 的自动化脚本写一个稳定的“安全读取”模块（示意）：

```powershell
function Get-JsonUtf8 {
    param([string]$Uri)
    $req = Invoke-WebRequest -Uri $Uri -UseBasicParsing -SkipCertificateCheck
    $json = [System.Text.Encoding]::UTF8.GetString($req.RawContentStream.ToArray())
    return $json | ConvertFrom-Json
}
```

在任何 Agent 调用处，不使用原生的 `Invoke-RestMethod`，统一通过这个函数。即使未来迁移到 Unix/Linux 也无需修改，因为 UTF-8 作为唯一显式声明，不与平台代码页产生关联。

另外，**在调试阶段请一定先用 `curl.exe` 做对比验证**（注意是 `curl.exe`，不是 PowerShell 的 `curl` 别名，`curl.exe` 直接把字节流写入 stdout，不会被二次编码），这样能第一时间判断问题是否出在客户端。

## 总结

“中文打坏”看似是某一个字符集的小问题，但在 Windows + PowerShell 的自动化管线里会绕过你所有的测试，直到 Agent 输出的 JSON 被下一个 MCP 节点消费时莫名其妙出错。根因就三条：HTTP 解码依赖了 ANSI 代码页、控制台输出降级、文件写入使用了非 UTF-8。修复的核心是**显式地抓住原始字节流，并在每一步强制 UTF-8**，同时尽快迁移到 PowerShell 7 以获得更现代的编码默认值。

把这条链路固化到团队的代码规范和共享模块里，可以避免 90% 的“怎么中文又乱了”的工单。剩下的 10%，排期换个系统终端也许更快乐。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/94ce83e248c0b0d1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/32906bffb4533738.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/2e3d9cca6bd26cb6.png)

