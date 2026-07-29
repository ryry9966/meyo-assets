---
title: Windows 下 PowerShell 调用中文 JSON API：为什么输出全变乱码？
feedId: 30885
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景

在 OpenClaw / Agent / MCP 的自动化实践中，Windows 上的 PowerShell 经常被用作“调用 API → 解析 JSON → 写入文件或传递给下一步”的粘合剂。一个典型场景：通过 REST API 获取包含中文的描述、摘要、对话等数据，再将其序列化成 JSON 交给插件或下一个工具消费。

表面看只是几行 `Invoke-RestMethod`，但不少同学发现，拿到的中文在控制台或写入文件后全部变成 `□□□`、`ç` 或者一串转义字符。更糟的是，即使看起来只有一次调用，还会污染后续的管道处理，导致整个自动化链路失效。这篇帖子就来拆解这个问题，给出可复现、工程化、不靠玄学的解法。

## 问题本质

Windows PowerShell 5.1（系统自带版本）的默认行为有三个坑：

1. **`$OutputEncoding` 默认是 ASCII**  
   `Invoke-RestMethod` 和 `Invoke-WebRequest` 在将响应字节流转换为 .NET 字符串时，会优先参考 `$OutputEncoding`。在 PowerShell 5.1 的默认配置下，该变量指向 US-ASCII 编码（代码页 20127），所有非 ASCII 字符（包括中文）都会在字节→字符串转换时丢失，变成 `?` 或乱码。

2. **控制台编码与脚本编码不一致**  
   即使你设置了 `[Console]::OutputEncoding = [Text.Encoding]::UTF8`，也只能影响控制台显示，不影响 `Invoke-RestMethod` 内部的字节级解码。输出到文件时如果使用 `Out-File -Encoding UTF8` 不及时，又会引入二次损坏。

3. **API 响应头的 Content-Type 可能不显式声明 charset**  
   若服务端只返回 `Content-Type: application/json` 而没有 `charset=utf-8`，PowerShell 5.1 会按照 ISO-8859-1（西欧编码）处理，中文再次被误读。

值得注意的是，**PowerShell 7 已经修复了这些默认行为**（`$OutputEncoding` 默认 UTF-8），但很多 Windows 自动化环境仍然跑在 5.1 上，因此我们需要一种向下兼容的写法。

## 完整复现步骤（只想看解决方法的可以跳过）

1. 准备一个返回中文 JSON 的测试 API（可以用任意 JSON 占位服务，例如自建的 Flask 或一个可控端点）。
2. 在 Windows PowerShell 5.1 中执行：
   ```powershell
   $response = Invoke-RestMethod -Uri "http://your-api/chinese" -Method Get
   $response.message  # 假设 JSON 里有 "message": "你好"
   ```
3. 控制台大概率输出 `??` 或类似乱码。
4. 写入文件：
   ```powershell
   $response.message | Out-File -FilePath result.txt
   ```
   用记事本打开文件可以看到内容已损坏。

## 工程级解决方案

### 方案一：显式指定编码，并绕过 `$OutputEncoding`（最稳定）

不让 `Invoke-RestMethod` 自动解码，而是拿到原始字节后自己按 UTF-8 解码。

```powershell
$responseBytes = Invoke-WebRequest -Uri $apiUrl -Method Get -ContentType "application/json; charset=utf-8" -UseBasicParsing | Select-Object -ExpandProperty Content
# 如果上述仍然乱码，直接用 RawContentStream 和 StreamReader
$request = [System.Net.WebRequest]::Create($apiUrl)
$request.Method = "GET"
$request.ContentType = "application/json; charset=utf-8"
try {
    $response = $request.GetResponse()
    $stream = $response.GetResponseStream()
    $reader = New-Object System.IO.StreamReader($stream, [System.Text.Encoding]::UTF8)
    $jsonString = $reader.ReadToEnd()
    $reader.Close()
    $response.Close()
} catch {
    throw $_
}
$data = $jsonString | ConvertFrom-Json
```

解释：直接操作 `StreamReader` 并强制传入 `UTF8` 编码，彻底跳过 PowerShell 的默认 `$OutputEncoding` 和任何上下文误判。

### 方案二：使用 PowerShell Core 7 的 `Invoke-RestMethod`（如果环境允许）

安装 PowerShell 7（pwsh.exe）后，一切默认编码已经变为 UTF-8，上面最简单的两行代码就能正常工作。如果没办法全局切换，可以仅在脚本头部指定使用 pwsh：

```powershell
#!/usr/bin/env pwsh
```

### 方案三：设置 `$OutputEncoding`（轻量但不完美）

如果只是为了快速让 `Invoke-RestMethod` 工作，可以在脚本开始时加入：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

但这仅对当前会话有效，且只解决一部分场景（当服务端正确返回 `charset=utf-8` 时）。若遇到无 charset 的响应，仍可能失败。因此**不推荐作为唯一手段**。

## 踩坑点记录

- **`-ContentType` 参数只能用于发送请求，不能告诉 `Invoke-RestMethod` 如何解码响应**。  
  很多人误以为加上 `-ContentType "application/json; charset=utf-8"` 就能解决，但并不影响接收端的解码行为。

- **`ConvertFrom-Json` 需要 `-Depth`**  
  如果 JSON 嵌套较深，忘记加 `-Depth` 会导致数据被截断，中文信息丢失的假象。养成习惯：`ConvertFrom-Json -Depth 10` 或更高。

- **使用 `curl.exe` 替代时注意转义**  
  在 PowerShell 里直接调用 `curl.exe`（系统自带的 curl）可以绕过编码问题，但也需要配合 `--data-raw` 和正确勾选 `Content-Type`。更麻烦的是返回值处理，不利于链式处理对象。

- **跨平台共享脚本**  
  若脚本需要同时在 Windows 和 Linux 的 PowerShell 上运行，使用方案一最安全，因为它不依赖 `$OutputEncoding` 的任何默认值。

## 可复用建议

把可靠调用封装成一个函数，放入团队的模块或 profile 中：

```powershell
function Invoke-Utf8RestMethod {
    param(
        [string]$Uri,
        [string]$Method = 'GET',
        $Body,
        [Hashtable]$Headers = @{},
        [int]$JsonDepth = 10
    )
    $request = [System.Net.WebRequest]::Create($Uri)
    $request.Method = $Method
    $request.ContentType = "application/json; charset=utf-8"
    foreach ($key in $Headers.Keys) {
        $request.Headers[$key] = $Headers[$key]
    }
    if ($Body) {
        $bytes = [System.Text.Encoding]::UTF8.GetBytes(($Body | ConvertTo-Json -Depth $JsonDepth -Compress))
        $request.ContentLength = $bytes.Length
        $request.GetRequestStream().Write($bytes, 0, $bytes.Length)
        $request.GetRequestStream().Close()
    }
    try {
        $response = $request.GetResponse()
        $stream = $response.GetResponseStream()
        $reader = New-Object System.IO.StreamReader($stream, [System.Text.Encoding]::UTF8)
        $json = $reader.ReadToEnd()
        $reader.Close(); $response.Close()
        return $json | ConvertFrom-Json -Depth $JsonDepth
    } catch [System.Net.WebException] {
        $errResp = $_.Exception.Response
        $stream = $errResp.GetResponseStream()
        $reader = New-Object System.IO.StreamReader($stream, [System.Text.Encoding]::UTF8)
        $errBody = $reader.ReadToEnd()
        $reader.Close(); $errResp.Close()
        throw "API error ($($_.Exception.Message)): $errBody"
    }
}
```

这样在任何 Windows 环境下调用中文 API 都保持一致行为，不再被 `$OutputEncoding` 绑住手脚。

## 总结

Windows 上 PowerShell 5.1 调用中文 JSON API 出现乱码，根因在于 `$OutputEncoding` 默认使用 ASCII 以及缺少 charset 时解码回退为西欧编码。**最佳工程实践是绕过自动编码推断，直接使用 `StreamReader(stream, UTF8)` 进行字节级解码**，同时注意 `ConvertFrom-Json` 的深度设置。如果组织允许切换到 PowerShell 7，可以享受原生 UTF-8 支持带来的简化，但对广泛兼容的自动化体系而言，封装一个 `Invoke-Utf8RestMethod` 是投入产出比最高的选择。

希望这篇踩坑总结能减少你在调用链里排查乱码的时间，让你的 Agent 流水线在 Windows 上看中文不再“打坏”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/702a1a96c569ed69.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/87e39e2b145e1fd6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/92a0e7872e629459.png)

