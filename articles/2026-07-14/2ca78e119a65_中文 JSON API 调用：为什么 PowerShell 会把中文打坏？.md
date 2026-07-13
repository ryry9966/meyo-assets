---
title: 中文 JSON API 调用：为什么 PowerShell 会把中文打坏？
feedId: 28957
source: 综合讨论
publishedAt: 2026-07-14
---

## 背景

在 OpenClaw 的 Agent、MCP 插件、自动化脚本链中，经常会用 PowerShell 调用各类 HTTP API 获取中文数据——可能是知识库检索结果，也可能是大模型返回的文本。站点、服务器清一色走 UTF-8，可一旦在 Windows 上 `Invoke-RestMethod` 打印或落盘，中文就成了一堆乱码，排查起来比想象中耗时。这篇文章不玩段子，只把根因、复现、治法和工程经验摊开讲。

## 问题：看起来是 JSON，读出来是“锟斤拷”

最常见场景：一个返回中文 JSON 的接口，用 Postman 正常，用 curl 正常，甚至浏览器直接打开也无恙。但在 Windows PowerShell 5.1 中执行：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/data" -Method Get
$resp.title
```

控制台输出变成类似 `"�Һ�"` 或 `"娴嬭瘯"` 的不可读字符。重定向到文件 `$resp | Out-File result.txt` 再用记事本打开，可能显示成 UTF‑16 文件、也可能依旧破损。

## 根因：三层编码不一致

问题并不是 PowerShell“不支持中文”，而是 **默认字符串流、控制台输出、文件重定向三者使用的编码不完全对齐，再加上 `Invoke-RestMethod` 的自动解析策略过于乐观**。

1. **响应体解码**  
   当 API 返回头只写 `Content-Type: application/json` 而缺少 `charset=utf-8` 时，`Invoke-RestMethod` 会退回使用默认 ANSI 代码页（简体中文系统是 GBK/CP936）对原始字节进行解码。UTF-8 多字节序列被错误地按单字节解释，中文当场损坏。

2. **控制台输出**  
   Windows 控制台默认输出编码是系统 OEM 代码页（如 CP437 或 CP936），并不一定是 UTF-8。即便字符串内部是正确 Unicode，`Write-Output` 时若显示字体不支持，也会看到“豆腐块”。

3. **文件重定向**  
   `>` 操作符和 `Out-File` 在不同 PowerShell 版本中默认编码不同：PowerShell 5.1 的 `>` 会产生 UTF-16 LE 文件，`Out-File -Encoding UTF8` 需要显式指定。混用极易产生二次乱码。

而 **PowerShell 7+ (Core)** 已大幅改善，默认以 UTF-8 处理，遇到此类问题的概率低得多。但很多 Windows 生产环境依旧绑定 PowerShell 5.1，这就必须自己兜底。

## 做法与步骤

### 1. 稳定复现环境

写一个小脚本即可触发：

```powershell
# 模拟返回中文 JSON 的服务（实际部署时可替换为真实 API）
$url = "https://httpbin.org/anything"
$body = @{ name = "中文测试" } | ConvertTo-Json -Compress
$resp = Invoke-RestMethod -Uri $url -Method Post -Body $body -ContentType "application/json; charset=utf-8"
$resp.json.name   # 很可能乱码
```

### 2. 根治方案：原始字节解码 + 格式化输出

放弃由 `Invoke-RestMethod` 自动解析，改用 `Invoke-WebRequest` 拿原始流，显式指定 UTF-8 解码，再将 JSON 字符串转成对象：

```powershell
$req = Invoke-WebRequest -Uri $url -Method Post -Body $body -ContentType "application/json; charset=utf-8" -UseBasicParsing
$rawBytes = $req.RawContentStream.ToArray()
$jsonString = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$obj = $jsonString | ConvertFrom-Json
$obj.json.name   # 正确显示 "中文测试"
```

这一招**绕过了所有自动编码推断**，无论响应头是否写 `charset`，都强制 UTF-8 解析。

### 3. 控制台输出不乱

在当前会话顶部显式设置控制台输出编码和字体：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

并确保控制台属性中字体选择“Consolas”或“Microsoft YaHei Mono”等能显示汉字的等宽字体。这一步对脚本内的 `Write-Host`、`Write-Output` 到控制台的流生效。

### 4. 落盘文件编码可控

任何需要持久化文本的地方，一律显式指定 UTF-8 或 UTF-8 BOM：

```powershell
$obj | ConvertTo-Json -Depth 10 | Out-File -FilePath "result.json" -Encoding UTF8
```

如果需要追加日志，可以使用 `Add-Content -Encoding UTF8`。切勿依赖 `>`，除非已全局设置 `$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'`（只适用于 PowerShell 5.1 部分场景）。

## 踩坑点

1. **看似修好但管道又坏**  
   即使 `$obj` 内部正确，`$obj | Select-Object name | Export-Csv` 仍可能写入 GBK 编码的 CSV。务必给 `Export-Csv` 也加 `-Encoding UTF8`。

2. **PowerShell ISE 与 VS Code 差异**  
   ISE 的输出编码与普通控制台不同，测试时请在目标运行环境（通常是 `powershell.exe` 或计划任务）中验证。

3. **Invoke-WebRequest 的字符集假设**  
   `$req.Content` 属性是已解码的字符串，同样受自动检测影响，所以直接使用 `RawContentStream` 最安全。

4. **Start-Transcript 日志乱码**  
   如果脚本使用 `Start-Transcript` 记录执行日志，也需要在开始前设置 `[Console]::OutputEncoding`，并将 BOM 写入文本头或接受 UTF‑8。实际上更推荐用 `Tee-Object` 配合显式文件编码。

5. **API 端真的返回了 UTF-8 吗？**  
   少数老旧服务声称 UTF-8 却输出混合编码。增加防御：先检测 BOM，若无 BOM 则尝试 UTF-8，失败后回退到 GBK 解码，但此种复杂度较高，一般通过监控源端修复。

## 可复用工程建议

在团队项目中，建议封装一个 `Invoke-Utf8JsonApi` 函数，固定处理流程：

```powershell
function Invoke-Utf8JsonApi {
    param([string]$Uri, [string]$Method = 'Get', $Body)
    $reqParams = @{
        Uri             = $Uri
        Method          = $Method
        UseBasicParsing = $true
        ContentType     = 'application/json; charset=utf-8'
    }
    if ($Body) { $reqParams.Body = $Body }
    $response = Invoke-WebRequest @reqParams
    $json = [System.Text.Encoding]::UTF8.GetString($response.RawContentStream.ToArray())
    return ($json | ConvertFrom-Json)
}
```

将该函数写入模块或 profile，所有脚本统一调用，编码问题一次解决。

另外，在 CI/CD 或 Agent 调度环境中，尽量使用 PowerShell 7+，可以规避 90% 的这类编码陷阱。如果短期无法切换，则把上述显式 UTF-8 解码写成注释过的模板，贴到每一个 HTTP 调用的上方。

## 总结

PowerShell 打坏中文 JSON，本质是编码转换链上出现了 UTF-8→ANSI/GBK 的误转。因为 HTTP 的 `charset` 缺失或因历史原因，PowerShell 5.1 的自动推断不够“信任 UTF-8”。解法并不高深：**接管原始字节流、强制 UTF-8 解码、显式指定所有输出编码**。在 OpenClaw 这种强调稳定性的自动化链路里，编码问题往往是小概率但高阻塞的卡点，早做防御，后期排查成本会低一个数量级。

---

## 配图

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/93919ecfe9a48c31.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/9608620d61b2dc45.png)

