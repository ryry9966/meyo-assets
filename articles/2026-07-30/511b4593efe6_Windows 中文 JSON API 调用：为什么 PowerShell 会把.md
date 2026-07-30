---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏，以及如何彻底修好
feedId: 31040
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景

在 Windows 上使用 PowerShell 调用第三方 API、构建自动化脚本或 MCP 工具时，经常需要处理中文 JSON 数据。很多同学会习惯性地写出类似这种代码：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/data" -Method Post -Body $jsonPayload -ContentType "application/json"
$response | ConvertTo-Json -Depth 10 | Out-File -FilePath "result.json" -Encoding utf8
```

看似没问题的脚本，本地一跑中文全变成 `????` 或 `锟斤拷`。更诡异的是，同样一份 JSON 在 Postman、curl 里都正常，唯独 PowerShell 的输出“打坏”了中文。这个问题直接影响 Agent、自动化工作流、OpenClaw 插件等场景的数据准确性，排查起来又比想象中曲折。

## 问题根源

PowerShell 打坏中文 **不是单一 bug**，而是三层编码坑叠加的结果：

1. **`Invoke-RestMethod` / `Invoke-WebRequest` 没有正确设置请求体编码**
   - 即使指定 `-ContentType "application/json"`，PowerShell 5.1 仍可能使用系统的 ANSI 代码页（例如 GBK）编码字符串，而不是 UTF-8。
2. **响应体解码默认依赖 `$OutputEncoding`**
   - `$OutputEncoding` 在 PowerShell 5.1 中默认是 ASCII，并且**只影响输出到控制台或重定向**，但奇怪的副作用是某些内部流也会受影响，导致返回的中文被错误地按 ASCII 解码。
3. **`Out-File` / `>` 重定向的编码陷阱**
   - `Out-File -Encoding utf8` 会写入带 BOM 的 UTF-8，而 `Set-Content -Encoding utf8NoBOM` 在 PowerShell 5.1 中不存在（仅 PS 7+），早期版本只能使用 `[System.IO.File]::WriteAllText()` 之类的方法无 BOM 输出。

你看到的乱码往往是这三层交错的结果：请求时中文被转成不正确的字节流发送到服务端，服务端返回的正常 UTF-8 数据又被错误的编码器解释，最后文件写出还可能再添一层 BOM 混淆。

## 稳定复现步骤

准备一个返回中文 payload 的测试 API，比如本地用 Flask 快速起一个：

```python
from flask import Flask, request, jsonify
app = Flask(__name__)
@app.route('/echo', methods=['POST'])
def echo():
    data = request.get_json(force=True)
    return jsonify({"received": data, "status": "正常"})
app.run()
```

然后在 PowerShell 5.1 中执行：

```powershell
$body = @{ message = "你好世界"; sender = "自动化脚本" } | ConvertTo-Json
$response = Invoke-RestMethod -Uri "http://127.0.0.1:5000/echo" -Method Post -Body $body -ContentType "application/json"
$response | ConvertTo-Json
```

你会发现：
- Postman 返回 `"status": "正常"`
- PowerShell 控制台显示 `"status": "æ­£å¸¸"` 或者 `"status": "??"`，具体乱码形式取决于 `$OutputEncoding`。

再用 `Get-Member` 查看响应对象的类型，发现它已经是 PSCustomObject，中文早在内部转换时坏掉了。

## 完整修复方案

### 1. 强制请求体使用 UTF-8 字节数组

不要直接将字符串传给 `-Body`，而是先转为 UTF-8 字节数组：

```powershell
$bodyJson = @{ message = "你好世界"; sender = "自动化脚本" } | ConvertTo-Json
$utf8Body = [System.Text.Encoding]::UTF8.GetBytes($bodyJson)
$response = Invoke-RestMethod -Uri $apiUrl -Method Post -Body $utf8Body -ContentType "application/json; charset=utf-8"
```

这能 100% 避免 ANSI 代码页干扰请求内容。

### 2. 稳定处理响应编码

如果你使用的是 PowerShell 5.1，**建议完全放弃 `Invoke-RestMethod` 的自动解析**，改用 `Invoke-WebRequest` 配合手动解码：

```powershell
$resp = Invoke-WebRequest -Uri $apiUrl -Method Post -Body $utf8Body -ContentType "application/json; charset=utf-8"
$decoded = [System.Text.Encoding]::UTF8.GetString($resp.RawContentStream.ToArray())
$jsonObj = $decoded | ConvertFrom-Json
```

或者使用更简洁的现代方案：升级到 PowerShell 7，其内部编码行为更贴合 UTF-8 by default。

### 3. 文件写入不要依赖 `Out-File`

输出 JSON 文件时使用：

```powershell
[System.IO.File]::WriteAllText("result.json", ($jsonObj | ConvertTo-Json -Depth 10), [System.Text.UTF8Encoding]::new($false))
```

第三个参数 `$false` 表示 **无 BOM**，避免下游工具（如 Node.js、Python）解析时出现莫名 `\ufeff` 字符。

## 一个可复用的安全包装函数

把上述处理封装成一个函数，放进你的 PowerShell 脚本、MCP 工具或 Agent 的 helper 中：

```powershell
function Invoke-Utf8RestMethod {
    param(
        [string]$Uri,
        [string]$Method = 'Post',
        $BodyObject,
        [hashtable]$Headers = @{}
    )
    $jsonStr = $BodyObject | ConvertTo-Json -Depth 10 -Compress
    $bytes   = [System.Text.Encoding]::UTF8.GetBytes($jsonStr)
    $resp    = Invoke-WebRequest -Uri $Uri -Method $Method -Body $bytes -ContentType "application/json; charset=utf-8" -Headers $Headers
    $raw     = [System.Text.Encoding]::UTF8.GetString($resp.RawContentStream.ToArray())
    return $raw | ConvertFrom-Json
}
```

调用时就像普通 REST 方法一样，但中文绝不会再被打坏。

## 踩坑回顾与排障清单

- **看到 `????` 或 `锟斤拷`**：首先检查请求体编码，用 `[System.Text.Encoding]::UTF8.GetBytes()` 取代字符串直传。
- **控制台显示正常，但文件乱码**：检查 `Out-File` 是否加了 BOM，下游能否处理 BOM。改用 `System.IO.File::WriteAllText`。
- **本地 OK，在 Task Scheduler 或 SYSTEM 账户跑就乱码**：系统账户的代码页可能与当前用户不同，强制所有输入输出为 UTF-8 字节流能消除环境差异。
- **升级后仍偶发乱码**：确认你的 PowerShell 版本是 7+，并且 `$OutputEncoding` 已设为 `[System.Text.UTF8Encoding]::new()`，但不完全依赖该变量，核心处理仍用字节数组。

## 总结

Windows 下的 PowerShell 在调用中文 JSON API 时容易踩编码坑，根本原因是从 string 到 byte[] 的隐式转换会带入系统代码页。工程上最稳妥的姿势是 **全程显式控制 UTF-8 字节流**：发送时 `GetBytes`、接收时 `GetString`、写文件时 `UTF8Encoding($false)`。这个习惯不仅能治好中文乱码，也能让你的自动化脚本在 GitHub Actions Runner、Windows 容器等多种环境下“跑到哪里都不虚”。

在 OpenClaw、Agent 或 MCP 工具链里，一个小小中文编码失误可能造成大段 prompt 被截断或服务器拒绝。花十几分钟统一处理，以后调用任何中文 API 都能像 curl 一样干净。

---

