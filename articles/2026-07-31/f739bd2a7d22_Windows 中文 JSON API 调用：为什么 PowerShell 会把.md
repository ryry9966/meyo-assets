---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 31069
source: 综合讨论
publishedAt: 2026-07-31
---

# Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏

## 背景
在 Windows 上搭建 OpenClaw 自动化管线、Agent 工具链或 MCP 插件时，我们经常需要通过 PowerShell 脚本调用 HTTP API 获取中文 JSON。无论是查询天气、企业信息还是本地知识库，返回的中文字段常常在控制台或保存的文件里变成一堆问号或天书——哪怕用 `Invoke-RestMethod` 看起来已经解析完毕。这个坑一旦踩中，不仅调试体验极差，还会直接打断下游 Agent 的结构化理解，让一整条自动化链路中断。本文还原真实场景，解释编码链的断裂点，并给出工程化的可复用方案。

## 问题重现
典型的错误操作如下：
```powershell
# 调用一个返回中文 JSON 的测试 API
Invoke-RestMethod -Uri "https://httpbin.org/anything?msg=你好" |
    Out-File output.json
```
打开 `output.json`，原本的 `"msg": "你好"` 变成了 `"msg": "ä½ å¥½"` 或不可读字节。即便使用 `>` 重定向，结果依然乱码。

根本原因在于：**PowerShell 的字符串管道在写入文件时，没有强制使用与 API 响应一致的编码**。`Invoke-RestMethod` 内部已经将响应体按 UTF-8 解码为 .NET 的 Unicode 字符串，但 `Out-File`、`>` 重定向或 `Set-Content` 在 Windows PowerShell 5.x 中默认采用的操作系统编码（如简体中文的 GBK/ANSI）或 UTF-16 LE，与 UTF-8 字节码并不兼容。控制台代码页（chcp 936）也会参与捣乱，让流经 stdout 的数据再次被错误转码。

## 做法与步骤

### 1. 强制文件输出为 UTF-8
最直接的修复是显式指定 `-Encoding utf8`：
```powershell
Invoke-RestMethod -Uri "..." |
    ConvertTo-Json -Depth 10 |
    Out-File -Encoding utf8 output.json
```
或者使用 `Set-Content`（同样需要 `-Encoding utf8`，否则默认 ANSI）。建议始终加 `-Encoding` 参数，养成肌肉记忆。

### 2. 直接处理字节流（更彻底）
当 API 响应头没有正确声明 charset，或你想完全掌控解码过程时，应使用底层流：
```powershell
$response = Invoke-WebRequest -Uri "..." 
$reader   = [System.IO.StreamReader]::new(
                $response.RawContentStream,
                [System.Text.Encoding]::UTF8
            )
$json = $reader.ReadToEnd()
$reader.Close()
# 后续以 UTF-8 写入
[System.IO.File]::WriteAllText("output.json", $json, [System.Text.Encoding]::UTF8)
```
这能避免 `Invoke-WebRequest.Content` 根据响应头自动解码可能带来的偏差。

### 3. 确保跨进程传递中文不损坏
如果你的自动化脚本要将 JSON 通过管道传给命令行工具（例如 MCP 的语言服务器），必须设置输出编码：
```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)   # 无 BOM
$jsonString | & my-cli.exe
```
否则 PowerShell 可能用 OEM 代码页把字符串转回字节，再次引入乱码。

### 4. 升级至 PowerShell 7+
PowerShell 7 默认使用 UTF-8 无 BOM 进行文件输出和管道编码，`$OutputEncoding` 也自动设为 UTF-8。这能大幅减少此类乱码问题，推荐在 OpenClaw 的自动化环境中固定使用 pwsh.exe 而非默认的 Windows PowerShell。

## 踩坑点
- **`$response.Content` 不等于原始字节**：`Invoke-WebRequest` 的 `Content` 属性已按响应头声明的 charset 解码，如果服务器声明为 GBK 但实际是 UTF-8，就会被双重破坏。此时必须用 `RawContentStream`。
- **`Out-File` 与 `Set-Content` 差别**：在 PS5 中，`Out-File` 默认 Unicode (UTF-16 LE)，`Set-Content` 默认 ANSI，两者对中文都不友好。无论用哪个，都必须加 `-Encoding utf8`。
- **控制台显示 ≠ 文件编码**：即便在 ConEmu/Windows Terminal 里看到中文显示正常，保存到文件也可能乱码，这是两个独立的编码路径。
- **重定向 `>` 的编码绑定于 `$OutputEncoding`**：很多教程用 `> output.txt` 保存 API 结果，但在 PS5 中，`>` 等同于 `Out-File` 且不使用 `-Encoding` 参数，会跟随系统区域设置，导致依赖默认值出错。

## 可复用建议
为 OpenClaw 的 MCP 工具或自动化插件编写一个通用的安全请求函数：

```powershell
function Invoke-ApiJson {
    param([string]$Uri)
    $wr = [System.Net.HttpWebRequest]::Create($Uri)
    $wr.AutomaticDecompression = [System.Net.DecompressionMethods]::All
    try {
        $response = $wr.GetResponse()
        $stream   = $response.GetResponseStream()
        $reader   = [System.IO.StreamReader]::new($stream, [System.Text.Encoding]::UTF8)
        $json     = $reader.ReadToEnd()
        $reader.Close()
        $response.Close()
        return $json
    } finally {
        if ($reader) { $reader.Dispose() }
        if ($response) { $response.Dispose() }
    }
}
```
调用后统一用 `Set-Content -Encoding utf8` 写入文件，或直接返回字符串交由上层的 UTF-8 环境处理。同时，在脚本开头务必设置 `$OutputEncoding = [System.Text.Encoding]::UTF8`，并养成显式编码的习惯。

## 总结
Windows 中文环境下的 PowerShell 乱码，本质是 **API (UTF-8) → PowerShell 字符串 (Unicode) → 文件/管道 (ANSI/UTF-16) 这一多阶段编码瀑布中，最后一环没有锁定为 UTF-8**。修复并不神秘，只要在每一个 I/O 边界强制使用 UTF-8 编码，并优先选用 PowerShell 7，就能让中文 JSON 平稳地在自动化管线中流动。这条原则对 OpenClaw 场景下的 Agent 稳定性尤为重要——因为一旦中文损坏，LLM 接收到的就是噪声，后续决策和工具调用都会彻底失效。

---

