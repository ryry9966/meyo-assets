---
title: 为什么 PowerShell 会把中文 JSON 打坏？Windows 编码链的一次深度排障
feedId: 29366
source: 综合讨论
publishedAt: 2026-07-17
---

# 为什么 PowerShell 会把中文 JSON 打坏？Windows 编码链的一次深度排障

## 背景

如果你在 Windows 上用 PowerShell 调过中文 API，十有八九碰到过这种灵异事件：服务端明明返回了正常的 JSON，但一到本地日志、输出面板，或者再转发出去的 payload 里，中文字段就变成了 `?`、`锟斤拷`，甚至直接消失。

OpenClaw 社区很多自动化流水线、MCP 插件、Agent 的本地 runtime 都跑在 Windows 上面。只要涉及 API 调用和中文内容处理，这个问题几乎必现。更坑的是，你用 Postman 或者 curl 验证过一切正常，偏偏切到 PowerShell 就炸——这往往会让人怀疑是 API 提供方的问题，排查方向完全跑偏。

## 问题的真实面貌

假设有这样一个极简调用：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/user/profile" -Method Get
$response.name
$response | ConvertTo-Json -Depth 3 | Out-File -FilePath "profile.json"
```

API 返回的原始 JSON 可能是：

```json
{"name": "张三", "bio": "OpenClaw社区成员"}
```

但控制台输出显示 `??`，写到文件的 profile.json 打开也是乱码。把文件拖到 VS Code 右下角一看——编码竟然是 `UTF-16 LE`，或者更离谱的 `Windows-1252`。

**问题不出在 API，不出在 .NET，甚至不全出在 PowerShell 本身。问题出在 Windows 编码链的「最后一公里」。**

## 根因拆解

PowerShell 5.x（Windows 内置版本）的默认行为有三层陷阱：

**第一层：响应解码阶段。** `Invoke-RestMethod` 和 `Invoke-WebRequest` 在底层使用的是 .NET 的 `HttpClient`。.NET 会尊重服务器返回的 `Content-Type` 头里的 charset。绝大多数现代 API 返回 `Content-Type: application/json; charset=utf-8`，这一步通常是对的，不会出错。

**第二层：管道与格式化阶段。** 当你把获取到的对象输出到控制台，或者 pipe 给 `ConvertTo-Json` 再输出，PowerShell 自身对 Unicode 字符串的内部表示是完好的。问题在于输出目标。

**第三层——也是真正的元凶：输出编码。** PowerShell 5.x 的 `Out-File` 和重定向运算符 `>` 默认使用 **UTF-16 LE** 编码。这不是 bug，是 Windows 的历史遗留设计。而很多外部工具、日志收集器、甚至部分文本编辑器，对 UTF-16 LE 的兼容性一塌糊涂，读到中文就变成乱码。

更隐蔽的情况是 `[Console]::OutputEncoding` 的影响。在中文 Windows 系统上，控制台默认代码页是 936（GBK）。当 PowerShell 试图将 Unicode 字符串输出到控制台时，会经过这个代码页的转换。如果一个字符在 GBK 中不存在，它就会变成 `?`——这个过程不可逆。

## 做法与步骤

### 1. 精确复现问题

先用 curl 对比验证，排除 API 侧的问题：

```powershell
curl.exe -H "Accept: application/json" "https://api.example.com/user/profile" -o curl_result.json
```

如果 curl_result.json 里的中文正常，说明问题确实在 PowerShell 侧。

### 2. 读取响应时干预编码

不要使用 `Invoke-RestMethod` 的默认行为，改用 `Invoke-WebRequest` 拿到原始字节后手动解码：

```powershell
$resp = Invoke-WebRequest -Uri "https://api.example.com/user/profile" -Method Get
$rawBytes = $resp.RawContentStream.ToArray()
$jsonString = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$obj = $jsonString | ConvertFrom-Json
```

这一步确保你拿到的是真正的 UTF-8 字符串，不经过任何 PowerShell 中间层的编码猜测。

### 3. 输出到文件时显式指定编码

**永远不要依赖默认输出编码。** 写文件用 `Out-File` 时加 `-Encoding utf8`：

```powershell
$obj | ConvertTo-Json -Depth 3 | Out-File -FilePath "profile.json" -Encoding utf8
```

如果是追加日志，`Add-Content` 同样需要 `-Encoding utf8`：

```powershell
Add-Content -Path "api.log" -Value $logLine -Encoding utf8
```

### 4. 控制台输出的临时修复

如果你只是调试时需要看控制台中文，可以临时修改代码页：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

但注意：**这不能修复文件输出的编码问题，只影响当前 session 的控制台显示。** 下次启动 PowerShell 会被重置。

## 踩坑点清单

- **ConvertTo-Json 的深度问题。** 默认 `-Depth` 是 2，嵌套深一点的返回对象直接给你截断，不报错，只是丢数据。写脚本时养成习惯显式指定 `-Depth 5` 或更高。

- **PowerShell 版本差异。** PowerShell 7+（跨平台 pwsh）已经将默认输出编码改为 UTF-8（无 BOM），行为与现代终端一致。如果你可以在环境里升级到 pwsh，这是最省事的方案。

- **BOM（Byte Order Mark）问题。** 很多人用了 `-Encoding utf8` 就以为万事大吉，但 PowerShell 5.x 的 `-Encoding utf8` 实际上是 **UTF-8 with BOM**。某些严格解析的 API 或解析器会因此炸掉。如果你需要无 BOM 的 UTF-8，用 `[System.IO.File]::WriteAllText()` 或者 `New-Item` + StreamWriter。

- **管道与重定向运算符 `>` 的行为。** `>` 在 PowerShell 5.x 里等同于 `Out-File`，所以也输出 UTF-16 LE。同样需要避免，替换为 `| Out-File -Encoding utf8`。

- **`$PSDefaultParameterValues` 的方案有瑕疵。** 有人建议全局设置默认编码参数，但这属于「隐式行为」，团队协作和 CI/CD 环境里很容易因为忘记这个全局配置而导致复现。建议每个文件输出都显式写明编码。

## 可复用建议

对于 OpenClaw 社区的插件和 MCP 开发者，建议在 Windows PowerShell 脚本模板里加入以下保底语句：

```powershell
# 脚本头部声明
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

# 统一文件写入函数
function Write-Utf8File {
    param([string]$Path, [string]$Content)
    [System.IO.File]::WriteAllText($Path, $Content, [System.Text.UTF8Encoding]::new($false))
}
```

如果你的插件需要在 CI/CD 的 PowerShell 环境下跑，务必在测试矩阵里加上 Windows Server 中文版 + PowerShell 5.x 的组合验证，否则很容易出现开发机（通常是 VSCode + UTF-8 终端）正常、生产机乱码的经典场景。

此外，对于需要转发 API 返回内容给下游的 OpenClaw Agent，建议在拿到响应后立即执行一次编码规范化：读取 -> 转为 UTF-8 字节流 -> 重建字符串。这一步虽然看起来多余，但能有效阻断上游的编码污染。

## 总结

Windows 中文 PowerShell 打坏 JSON 编码这件事，本质上是 PowerShell 5.x 默认输出编码（UTF-16 LE）与控制台代码页（GBK）共同作用的结果。API 返回和 .NET 解析本身都没问题，问题出在「字符串离开 PowerShell 管道的时刻」。

解决路径很清晰：**所有输出操作都显式指定 `-Encoding utf8`（或根据需要选择 utf8NoBOM）；关键路径上不要依赖 PowerShell 的自动编码推断；生产环境能升 pwsh 就升。**

排查过程本身就是一次对 Windows 编码链的深度理解，对于经常在 Windows 上做 API 集成和自动化开发的 OpenClaw 用户来说，这个坑值得踩一次、记一辈子。

---

