---
title: 为什么 PowerShell 会把中文 JSON 打坏？——管道编码的隐性陷阱
feedId: 28982
source: 综合讨论
publishedAt: 2026-07-14
---

在 OpenClaw、Agent 编排以及 MCP 插件的日常开发中，我们经常需要在 Windows 上通过 PowerShell 调用各类 REST API，把返回的 JSON 数据落地给下一个工具消费。一个非常折磨人的场景是：用 `Invoke-RestMethod` 或 `curl` 拿到的中文明明在终端里看着正常，一旦写入文件或被下游程序读取，就变成 `??`、`锟斤拷` 甚至直接报 `Invalid UTF-8`。这篇文章从一次真实的 agent 排障出发，记录根因、复现、修复以及一套可复用的编码策略。

## 问题现场

我们在 Windows Server 2022 上使用 Windows PowerShell 5.1，通过 `Invoke-RestMethod` 调用一个内部 API，返回体包含中文城市名、备注等字段：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.internal.example.com/data?q=深圳"
$resp.city         # 终能正确显示 "深圳市"
$resp | ConvertTo-Json -Compress | Out-File result.json
```

打开 `result.json` 后，中文变成了 `æ·±å³å¸` 或 `????`。更糟的是，下游的 Python 脚本直接抛 `UnicodeDecodeError`。而如果把 `Invoke-RestMethod` 换成 `Invoke-WebRequest` 并获取 `$_.RawContent`，似乎又“正常”了，但一旦赋值给变量、放入管道再输出，乱码依旧。

## 根因：PowerShell 的“好心”转码

这里至少涉及三层编码的互相踩踏：

1. **API 响应的原始字节流**  
   绝大多数现代 JSON API 返回 `Content-Type: application/json; charset=utf-8`，字节流是合法 UTF-8。

2. **PowerShell 的内部字符串**  
   `Invoke-RestMethod` 会解析响应体，将字节流按响应头声明的 charset 解码为 .NET 的 `[string]` 对象。这一步本身没问题，因为 .NET 字符串是 Unicode。

3. **输出到管道 / 文件时的编码**  
   问题就出在这里。PowerShell 将字符串发送到管道或文件时，会使用 **编码转换器**，而转换器遵从的是：
   - 控制台输出（写入宿主）：受 `[Console]::OutputEncoding` 控制
   - 管道/重定向（> 、Out-File 等）：受 `$OutputEncoding` 变量控制
   - `Set-Content`、`Out-File` 自带 `-Encoding` 参数，**默认值往往是 Unicode（UTF-16 LE）或 ASCII**，具体版本间行为不同

在 Windows PowerShell 5.1 中：
- `$OutputEncoding` 默认会是系统的 OEM 代码页（简体中文版为 936 GBK），而非 UTF-8。
- 使用管道或重定向 `>` 等价于 `Out-File`，它继承了 `$OutputEncoding`。
- 未经指定的 `Out-File` 在 PS5.1 上默认使用 `Unicode`（UTF-16 LE），某些工具可能不识别 BOM。

因此，即使变量里完美保存了“深圳市”，当它穿过 `$OutputEncoding` 定义的窄编码，或通过 `Out-File -Encoding Unicode` 写入带 BOM 的 UTF-16 文件，下游按 UTF-8 读取就会乱码。而 `ConvertTo-Json` 生成的 JSON 字符串本身已经是 Unicode，在管道中被 GBK 化，所有高位字符被转成 `?` 或本地代码页的映射，信息彻底丢失。

## 复现最小案例

```powershell
# 模拟一个返回中文 JSON 的接口（httpbin）
$uri = "https://httpbin.org/anything?city=深圳"
$resp = Invoke-RestMethod -Uri $uri
# 检查内部值：$resp.args.city 应该是 "深圳"

# 尝试输出到文件 —— 必现乱码
$resp | ConvertTo-Json -Depth 2 > broken.json
Get-Content broken.json
```

查看 `broken.json` 的十六进制头会发现，文件编码是 GBK 或者 UTF-16，但中文已经损坏。

## 修复步骤

**核心思路：绕过 PowerShell 的默认管道编码，直接以 UTF-8 无 BOM 写入文件。**

### 方法一：配置 `$OutputEncoding` 为纯 UTF-8（适用于持续输出的脚本）
```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)  # $false 表示无 BOM
$resp | ConvertTo-Json -Depth 2 | Out-File -Encoding UTF8 fixed.json
```
注意 `Out-File -Encoding UTF8` 会写入 BOM（不可省略 `-Encoding`），结合管道用 UTF-8 输出就可以保证无 BOM。更彻底的做法是用 `[System.IO.File]::WriteAllText`。

### 方法二：直接用 .NET 方法写文件（推荐）
```powershell
$jsonStr = $resp | ConvertTo-Json -Depth 2 -Compress
[System.IO.File]::WriteAllText("fixed.json", $jsonStr, [System.Text.UTF8Encoding]::new($false))
```
这条路径完全自管理编码，不经过 PowerShell 管道的隐形转码。

### 方法三：使用 PowerShell Core (pwsh 7+)
pwsh 7 在启动时会设置 `$OutputEncoding` 为 UTF-8 无 BOM，并让 `>` 重定向和 `Out-File` 默认使用 UTF-8。迁移到 pwsh 可以从根本上避免大部分乱码。对于 Windows 10 1809+ 和 Windows Server 2019+，建议将 Agent 运行时默认 Shell 改为 pwsh。

## 其他踩坑点

- **Invoke-WebRequest 的 RawContent**：看似没乱码，实际是它将响应体存为字节数组，而你在终端输出时恰好与 `[Console]::OutputEncoding` 匹配显示，但将其以字符串形式转存会同样踩编码坑。
- **`Set-Content -Encoding`**：默认会使用 ASCII 或 ANSI，对中文直接丢弃高位字节，绝对不要用于存 JSON。
- **BOM 的隐性危害**：许多 JSON 解析器（包括 Python 的 `json.load`）默认支持带 BOM 的 UTF-8，但一些严格的嵌入式系统或命令行工具会因开头的 `\xEF\xBB\xBF` 报错，因此统一使用无 BOM 更安全。
- **字符集声明的缺失**：如果 API 未返回 `charset`，`Invoke-RestMethod` 会尝试根据 BOM 或 ISO-8859-1 解码。这时需要手动干预，通过 `Invoke-WebRequest` 获取原始字节再用 `[System.Text.Encoding]::UTF8.GetString()` 转换。

## 可复用建议

在需要稳定处理中文 JSON 的 Windows Agent 工作流中，建议将所有 PowerShell 脚本（.ps1）的头部都带上下列初始化：

```powershell
# 强制输出编码为UTF-8无BOM
$OutputEncoding = [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)

# 所有生成JSON文件的模块统一使用函数
function Write-JsonFile {
    param($InputObject, $Path)
    $json = $InputObject | ConvertTo-Json -Depth 10 -Compress
    [System.IO.File]::WriteAllText($Path, $json, [System.Text.UTF8Encoding]::new($false))
}
```

另外务必让下游工具（Python/Node/Go）也以 UTF-8 读取文件。如果团队使用 `curl.exe` 而非 PowerShell cmdlet，则可以通过 `curl -s | Out-File` 的管道同样受上述配置保护；或者直接重定向 `curl -o`，但不建议通过 PowerShell 包裹。

## 总结

PowerShell 把中文 JSON 打坏的根因，是 Windows 遗留编码习惯与 UTF-8 生态之间的冲突，尤其体现在管道输出和文件重定向的隐性转码。只要在脚本头部固定 `$OutputEncoding`，并采用 .NET 方法显式编码写入文件，就能在 Windows PowerShell 5.1 上稳定工作。最佳实践是尽快迁移到 pwsh 7，让默认行为与现代 UTF-8 体系对齐。编码一致性是自动化管线能否“静默运行”的基础，值得一开始就做对。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/630a82d179cca57b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/4ef9a2561f9d9659.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/4e52b8d4b8695e5d.png)

