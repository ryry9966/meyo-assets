---
title: Windows PowerShell 调用中文 JSON API 总是乱码？深入字符集陷阱与工程化解法
feedId: 30775
source: 综合讨论
publishedAt: 2026-07-28
---

## 背景

在 Windows 上做 OpenClaw／Agent 自动化时，用 PowerShell 调用 REST API 几乎是标配。大部分情况返回的都是 JSON，里面经常夹带中文字段，例如用户名、任务描述、模型回复等。如果把这些 JSON 交给 MCP 工具或后续插件解析，一旦中文变成了 `????` 或者 `鍝堝搱`，整个链路就会崩塌。

更糟的是这类问题不是必现：同样一份脚本，在 PowerShell 5.1 的 CMD 窗口、PowerShell 7 的 Windows Terminal、又或者通过 Agent 子进程调用，结果完全不同。排查时很容易怀疑是 API 服务端的问题，实际上多数时候是 **客户端字符集处理不一致** 造成的。

## 问题：为什么中文会“被打坏”

假设你有一个典型的中文返回 API：

```http
GET /api/task?id=1
Content-Type: application/json; charset=utf-8

{
  "id": 1,
  "title": "处理中文订单",
  "status": "已完成"
}
```

用 PowerShell 最简单的调用方式：

```powershell
$resp = Invoke-RestMethod -Uri "http://localhost/api/task?id=1"
Write-Output $resp.title
```

在没有额外设置时，Windows PowerShell 5.1 很可能输出 `????` 或完全乱码的字符串。进入 `$resp.title` 检查原始字节，会发现它们已经被错误解码一次，之后就没救了。

根本原因在于 **PowerShell 的字符串解码管道用的是系统代码页（OEM code page）**。中文 Windows 的默认代码页是 CP936（GBK），而 API 返回的是 UTF-8 字节流。`Invoke-RestMethod` 在内部将响应字节流转成 .NET 字符串时，如果未正确识别 `charset`，会退回到 `$OutputEncoding`（默认为 ASCII/系统代码页），于是 UTF-8 被当作 GBK 解释，产生永久性损坏。

类似问题也出现在构造请求 Body 发送中文时：中文 .NET 字符串以默认编码转换，服务器端可能收到 GBK 而非预期的 UTF-8，出现入库乱码。

## 复现与排查步骤

以 Windows PowerShell 5.1 为例，搭建最小复现场景：

1. 准备一个返回中文 JSON 的测试 API（本地 Flask/FastAPI 即可）。
2. 打开 PowerShell（非 ISE，注意是 conhost 还是 Windows Terminal）。
3. 执行：

```powershell
$OutputEncoding; [Console]::OutputEncoding
```

通常前者显示 `System.Text.ASCIIEncoding`，后者为 GBK（`936`）。

4. 调用 API 并检查：

```powershell
$resp = Invoke-RestMethod -Uri "http://127.0.0.1:5000/task?id=1"
[System.Text.Encoding]::Default.GetString([System.Text.Encoding]::UTF8.GetBytes($resp.title))
```

如果二次编码后能看到正确中文，说明是解码错误。

5. 查看实际响应字节（使用 `Invoke-WebRequest`）：

```powershell
$raw = Invoke-WebRequest -Uri "http://127.0.0.1:5000/task?id=1" -UseBasicParsing
$raw.RawContent | Out-File -FilePath raw.txt -Encoding utf8
```

检查 raw.txt 中的字节序列，确认服务端返回无误。

## 工程化做法：持久修复

### 一、设置进程级 UTF-8 输入输出编码

在脚本最前面加入：

```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new()
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

- `$OutputEncoding` 控制管道、外部命令的字符串→字节转换，以及 Web cmdlets 的默认解码。
- `[Console]::OutputEncoding` 用于控制台显示（避免在终端看到乱码，但不影响 `$resp` 对象本身）。

> **重要**：如果使用 PowerShell 7+，默认 `$OutputEncoding` 已经是 UTF-8，但控制台编码仍可能受系统影响，所以这两行仍有必要放在自动化入口。

### 二、请求 Body 显式指定编码

构造 JSON 发请求时：

```powershell
$body = @{ title = "中文测试"; status = "进行中" } | ConvertTo-Json -Compress
$encodedBody = [System.Text.Encoding]::UTF8.GetBytes($body)

$response = Invoke-WebRequest -Uri "http://127.0.0.1:5000/task" `
  -Method Post `
  -ContentType "application/json; charset=utf-8" `
  -Body $encodedBody
```

这里用字节数组绕过自动编码，确保服务器收到 UTF-8。

### 三、写入文件或日志时强制 UTF-8

`Out-File`、`Set-Content`、`Add-Content` 默认编码是 `Unicode` (UTF-16LE) 或系统代码页，必须显式指定：

```powershell
$resp | ConvertTo-Json -Depth 3 | Out-File -FilePath data.json -Encoding utf8
```

如果使用重定向 `>`，它在 Windows PowerShell 5.1 下是 UTF-16LE，需要用 `| Out-File -Encoding utf8` 代替。

### 四、整合为一个安全调用函数

自动化里建议封装一个 `Invoke-RestMethodUtf8`：

```powershell
function Invoke-RestMethodUtf8 {
    param(
        [Parameter(Mandatory)]$Uri,
        $Method = 'Get',
        $Body,
        $Headers
    )
    $prevOutput = $OutputEncoding
    try {
        $OutputEncoding = [System.Text.UTF8Encoding]::new()
        $params = @{ Uri = $Uri; Method = $Method }
        if ($Body) { $params.Body = $Body }
        if ($Headers) { $params.Headers = $Headers }
        Invoke-RestMethod @params
    } finally {
        $OutputEncoding = $prevOutput
    }
}
```

## 踩坑点

1. **PowerShell 版本差异**  
   PowerShell 7+ 虽然默认 `$OutputEncoding` 为 UTF-8，但某些模块（如 `Az`）可能修改它，导致间歇性乱码。自动化脚本应始终主动设置，而不是依赖默认值。

2. **curl.exe 与 Invoke-WebRequest 的别名陷阱**  
   Windows 10 开始 `curl` 实际上是 PowerShell 的别名，指向 `Invoke-WebRequest`，使用相同的编码逻辑。如果直接调用 `curl.exe`（外部命令），则需要额外设置环境变量 `$Env:LC_ALL='en_US.UTF-8'` 或通过 `-ContentType` 指定字符集。

3. **Agent 子进程环境**  
   当 Agent 通过 `Start-Process` 或 System.Management.Automation 创建 PowerShell 运行空间时，可能未加载 Profile，`$OutputEncoding` 依旧是 ASCII。必须把编码设置写在调用代码块或脚本正文中，不能依赖 Profile。

4. **文件内容读写**  
   `Get-Content -Encoding utf8` 读取含 BOM 的文件可能留下特殊字符，建议使用 `[System.IO.File]::ReadAllText("path", [System.Text.Encoding]::UTF8)` 更精确控制。

## 可复用建议

- **入口脚本模板**：所有自动化 PowerShell 脚本头部放置：
  ```powershell
  [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
  $OutputEncoding = [System.Text.UTF8Encoding]::new()
  $PSDefaultParameterValues['*:Encoding'] = 'utf8'
  ```

- **容器 / CI 环境**：若在 Windows 容器或 CI Agent 中运行，在 `Dockerfile` 或启动命令中设置：
  ```
  $env:LANG='zh_CN.UTF-8'
  ```
  并确保 `-Encoding utf8` 贯穿所有文件操作。

- **优先使用原生指令而非重定向**：结合 `Tee-Object` 或 `Out-File` 保留 UTF-8 输出，避免 `>` 重定向变形中文字符。

## 总结

PowerShell 的字符集行为是“继承+默认”的产物，在英文环境下往往不会被触发，一旦处理中文 API，就会暴露。面向 OpenClaw／Agent／MCP 的自动化用户，这类编码不一致带来的静默错误可能浪费大量排查时间。从根源上固定 `$OutputEncoding` 为 UTF-8、所有 IO 操作显式指定编码、封装安全调用函数，是成本最低、收益最大的工程化实践。记住：**在 Windows 做自动化，编码永远不要相信默认值。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/220d2b01a0e507ba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/e00e619ad18f948d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/6e6b34a23cf749c0.png)

