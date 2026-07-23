---
title: 中文 JSON 在 Windows PowerShell 里是怎么被“打坏”的？
feedId: 30172
source: 综合讨论
publishedAt: 2026-07-23
---

## 背景

在 Windows 上使用 PowerShell 调用带中文的 JSON API，是 OpenClaw 插件、Agent 工具链、自动化脚本中常见的场景。无论是调用内部服务的 REST 接口，还是与第三方 MCP 服务器交互，只要数据里出现中文字符，看似简单的 `Invoke-RestMethod` 或 `ConvertTo-Json` 就会开始“打坏”中文——终端看到问号、服务端拿到乱码、文件里的 JSON 根本无法被其他工具解析。

这种现象不是玄学，而是 Windows PowerShell 默认编码行为与 UTF-8 之间的一系列冲突。本文把这些问题拆解成可复现、可修复的工程步骤，避免你每次都要重复踩坑。

## 问题：两种最常见的中文乱码

### 场景1：服务端返回的中文 JSON 在终端变成乱码

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/items"
$resp.name  # 期望“中文名称”，实际输出“???????”
```

另存为文件后，用 VS Code 打开发现是乱码，或者用 `Out-File` 写出的文件在其他工具里全是问号。

### 场景2：发送含中文的 JSON 请求，服务端看到被损坏的数据

```powershell
$body = @{ remark = "备注信息" } | ConvertTo-Json
Invoke-RestMethod -Uri "https://api.example.com/items" -Method Post -Body $body -ContentType "application/json"
```

服务端日志里收到的 `remark` 变成 `????`，或者更隐蔽的：JSON 里中文字符被转成了 `\u5907\u6CE8\u4FE1\u606F`，虽然合法但可读性极差，且下游系统如果不支持 Unicode 转义就会出现语义丢失。

## 根本原因：PowerShell 的编码陷阱地图

### 1. `$OutputEncoding` 默认不是 UTF-8，而是 ASCII

Windows PowerShell 中，`$OutputEncoding` 决定了与外部程序（包括 HTTP 响应解码、管道传递）交互时的字符编码，**默认值为 US-ASCII**。  
当 `Invoke-RestMethod` 返回的响应体需要被解码为字符串时，它会参考 `$OutputEncoding`。一旦响应中包含非 ASCII 字符，ASCII 解码就会将这些字节替换为 `?`。  
所以中文在结果对象里直接变成 `?`，表面看是控制台问题，实质是解析阶段就已损坏。

> 注意：`[Console]::OutputEncoding` 只影响控制台显示（比如 Write-Host），**不影响**管道、重定向或 HTTP 响应的字符串转换。很多人改了控制台编码仍然乱码，就是因为搞错了对象。

### 2. `>` 与 `Out-File` 默认编码是 UTF-16 LE

在 Windows PowerShell 中：

- `Some-Command > file.txt` 实际上等价于 `Some-Command | Out-File -Encoding Unicode`（即 UTF-16 LE）
- 直接使用 `Out-File -FilePath file.json` 不指定编码，也是 UTF-16 LE

当你把 API 返回的 JSON 用 `>` 存到文件，然后用 Python、Node.js 或 `jq` 读取时，这些工具预期的是 UTF-8，结果一个两字节的 BOM 和双字节编码瞬间让解析器崩溃。

### 3. `ConvertTo-Json` 默认转义非 ASCII 字符

PowerShell 的 `ConvertTo-Json` 为了兼容性，会将所有非 ASCII 字符转义为 `\uXXXX`。  
这在技术上是合法的 JSON，但会把“备注”变成 `\u5907\u6CE8`，体积膨胀，而且某些弱 JSON 解析器或日志系统可能将其当作普通字符串直接存储，导致后续处理混乱。

## 工程化修复步骤

### Step 1：统一编码为 UTF-8

在脚本最顶部（或 OpenClaw Agent 的启动块）强制设置：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

这两行分别解决**字符串**和**显示**的编码问题。`$OutputEncoding` 是修复乱码的钥匙。

如果你的自动化环境不能修改全局 Profile，就在每个脚本模块的 `begin` 块中设置。

### Step 2：发送 JSON 请求时，显式指定 UTF-8 字节数组

最稳妥的方式是不让 PowerShell 自动编码，而是自己控制字节流：

```powershell
$bodyHashtable = @{ remark = "备注信息" }
$jsonString = $bodyHashtable | ConvertTo-Json -Compress
$jsonBytes = [System.Text.Encoding]::UTF8.GetBytes($jsonString)

Invoke-RestMethod -Uri "https://api.example.com/items" `
    -Method Post `
    -Body $jsonBytes `
    -ContentType "application/json; charset=utf-8"
```

关键点：
- `-ContentType` 中明确 `charset=utf-8`，部分服务端会据此选择解码器。
- `-Body` 直接给 `byte[]`，PowerShell 不再碰编码，原样发送。

### Step 3：安全写出 JSON 文件

使用 `Set-Content` 或 `Out-File` 时显式指定 `utf8NoBOM`（PowerShell 6+ 可直接使用 `-Encoding utf8NoBOM`；Windows PowerShell 5.1 中需要变通）：

```powershell
$response | ConvertTo-Json -Depth 5 | Set-Content -Path "result.json" -Encoding UTF8
```

如果你还在用 Windows PowerShell 5.1，最可靠的办法是：

```powershell
[System.IO.File]::WriteAllText("$PWD\result.json", $jsonString, [System.Text.UTF8Encoding]::new($false))
```

第三个参数指定是否带 BOM，`$false` 即无 BOM，更安全。

### Step 4：控制 ConvertTo-Json 的转义行为

如果你必须让 JSON 中的中文保持可读，可以用 `-EscapeHandling EscapeNonAscii`（PowerShell 6+ 才有效，Windows PowerShell 不支持）。  
对于 Windows PowerShell，可以使用 `Newtonsoft.Json` 库：

```powershell
Add-Type -Path "Newtonsoft.Json.dll"
[Newtonsoft.Json.JsonConvert]::SerializeObject($bodyHashtable, [Newtonsoft.Json.Formatting]::Indented)
```

但引入外部依赖会增加维护复杂度，请按需选择。对于纯 API 调用，**不转义并非必须**，只要保证字节流正确即可。

## 踩坑点总结

- **`$OutputEncoding` vs `[Console]::OutputEncoding`**：前者管管道/HTTP解码，后者只管终端显示，别只改后者。
- **Windows PowerShell 5.1 不支持 `utf8NoBOM` 枚举值**：直接 `Set-Content -Encoding utf8` 会产生 BOM，某些下游工具（如 `jq`、Node.js 的某些 parser）会失败。请用 .NET 方法写出无 BOM 文件。
- **手动构造 JSON 字符串**：不要使用字符串拼接（如 `"{""remark"":""$remark""}"`），一旦 `$remark` 中含双引号或特殊字符就会非法。始终通过 `ConvertTo-Json` 或序列化库构建。
- **`-ContentType "application/json"` 不等于 `charset=utf-8`**：如果服务端严格遵循标准，没有 charset 时可能回退到 ISO-8859-1 或 ASCII，保险起见总写明 UTF-8。

## 可复用模板

为了方便在 OpenClaw、Agent 或自动化脚本里直接复制，这里提供一个最小可运行模板：

```powershell
# 编码初始化
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

# 安全 POST JSON 函数
function Invoke-JsonPost {
    param([string]$Uri, [hashtable]$BodyData)
    $json = $BodyData | ConvertTo-Json -Compress
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($json)
    Invoke-RestMethod -Uri $Uri -Method Post -Body $bytes `
        -ContentType "application/json; charset=utf-8"
}

# 安全读文件无 BOM
function Save-Utf8NoBom {
    param([string]$Path, [string]$Content)
    [System.IO.File]::WriteAllText($PWD.Resolve($Path), $Content, 
        [System.Text.UTF8Encoding]::new($false))
}
```

在你的插件或 Agent 脚本里引用这个函数库，可以避免大部分因编码引发的非确定性故障。

## 总结

Windows PowerShell 的“中文打坏”问题，本质上是历史遗留的默认编码与现代 UTF-8 实践之间的矛盾。在自动化流水线、OpenClaw 插件与 Agent 通信等对可靠性要求较高的场景中，必须显式控制以下三点：

1. **管道与响应解码编码**（`$OutputEncoding`）
2. **请求体字节编码**（byte[] 发送）
3. **文件输出编码**（无 BOM UTF-8）

遵循这三条原则后，中文 JSON 调用才会像其他语言一样可靠，不会再出现“测试环境正常、生产环境一堆问号”的诡异现象。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/c540c2b908b7cf4d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/a853c4772b0d73d1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-23/423e234e9044c48b.png)

