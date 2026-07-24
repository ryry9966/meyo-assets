---
title: Windows 中文 JSON API 调用的“隐形地雷”：PowerShell 是如何把中文打坏的
feedId: 30313
source: 综合讨论
publishedAt: 2026-07-24
---

## 背景：自动化脚本里最不起眼的翻车现场

在 OpenClaw/Agent/MCP 这类项目里，PowerShell 经常扮演“胶水脚本”的角色：拉取某个中文 API 返回的 JSON，再交给插件解析、写入文件或推回 agent 上下文。出问题的时候，机器人突然开始胡言乱语——“主题”变成了“??”，“摘要”变成一坨问号，但英文字段安然无恙。排查一圈，接口用 `curl`、Postman 都正常，唯独 PowerShell 脚本产出的是坏字符。窗口一关，锅甩给了“系统编码”，但今天我们把根挖出来。

## 问题拆解：Unicode 去哪儿了

典型症状：`Invoke-RestMethod` 调用一个返回 Content-Type `application/json` 的接口，中文变成类似 `æµè¯` 的乱码，或在控制台打印正常但 `Out-File` 到文本后打开全是问号。根源在于三条线同时失效：

1. **HTTP 响应的解码假设**：如果服务器没有在 Content-Type 中显式给出 `charset=utf-8`，PowerShell 会根据 .NET 的默认回退规则以 `ISO-8859-1` 解码字节流，中文全部被错切成单字节字符。
2. **控制台宿主差异**：Windows PowerShell 5.1 的控制台默认输出编码经常是 `us-ascii` 或 `Windows-1252`，即便你把字符串在脚本内修好了，显示输出时又被二次破坏。
3. **管道与文件写入的编码不一致**：`Out-File` 和 `>` 重定向使用的是 `$OutputEncoding`，其默认值在 PS5.1 中为 `us-ascii`，而 PS7 虽然改进很多，但旧脚本仍可能在 Windows 系统上踩到这个坑。

更隐蔽的一层：`Invoke-RestMethod` 自动解析 JSON 后返回 `PSCustomObject`，中文在对象属性里其实还在，但格式化输出、写入文件、管道传递时因为每层编码不同而相继损坏，你看到的已经是第三手错误。

## 做法/步骤：让 PowerShell 老老实实说中文

### 1. 不要让 HTTP 解码靠猜
永远在请求中明确指定 Accept-Charset，并在收到响应后主动控制解码。最稳妥的办法不是直接 `Invoke-RestMethod`，而是用 `Invoke-WebRequest` 拿回原始字节，再用 `[System.Text.Encoding]::UTF8.GetString()` 解码。

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/cn-data" -Method Get
$bodyBytes = $response.RawContentStream.ToArray() # 或者用 Content 属性
$decodedBody = [System.Text.Encoding]::UTF8.GetString($bodyBytes)
$jsonObject = $decodedBody | ConvertFrom-Json
```

如果必须使用 `Invoke-RestMethod` 省去手动转换 JSON 这一步，至少要确保目标 API 返回了正确的 charset。但很多时候你无法控制 API 端，因此更可靠的是在请求头中把 `Charset` 声明清楚，并在脚本开头强制更新编码变量。

### 2. 设置 PowerShell 全局编码三板斧
在脚本的最顶部放置以下三行，可以挡住大部分乱码输出：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
[System.Text.Encoding]::Default = [System.Text.Encoding]::UTF8
```

说明：
- `$OutputEncoding` 控制管道到外部命令以及 `Out-File` 等的输出编码。
- `[Console]::OutputEncoding` 确保控制台能正确显示 UTF-8 字符。
- 第三行可能侵入性较强，仅在确认环境允许时使用。它会影响后续所有 .NET 调用默认编码。

### 3. 写文件必须指定 -Encoding
永不用 `>` 或默认 `Out-File`，坚持显式编码：

```powershell
$jsonObject | ConvertTo-Json -Depth 5 | Out-File -FilePath "output.json" -Encoding utf8NoBOM
```

注意 `-Encoding utf8` 会带上 BOM，某些下游 MCP/Agent 解析器会报错；`utf8NoBOM` 更干净（需 PS 6+，PS5.1 可用 `[System.IO.File]::WriteAllText("output.json", $jsonString, [System.Text.UTF8Encoding]::new($false))`）。

### 4. 验证中间输出
调试阶段可以用 `Format-Hex` 看原始字节，明确字符是否真的损坏：

```powershell
$rawBytes = [System.Text.Encoding]::UTF8.GetBytes($decodedBody)
$rawBytes | Format-Hex
```

如果能看原文的 UTF-8 字节序 `e6 b5 8b e8 af 95`（“测试”），说明解码正确，问题出在输出层；如果已经变成 `3f 3f`，说明解码时就错了。

## 踩坑点

- **`Invoke-RestMethod` 的 -ContentType 参数**：这是设置**请求头**Content-Type，而不是控制响应解码。很多帖子误解其作用，用来“编码修复”无效。
- **PS5.1 vs PS7**：PS7 对 UTF-8 原生支持更好，但很多 Windows 自动化环境仍用 5.1（尤其系统预装脚本、计划任务）。混合环境需分别测试。
- **`$PSDefaultParameterValues` 与作用域**：在 `profile.ps1` 里设好 `$PSDefaultParameterValues['*:Encoding'] = 'utf8'` 会污染所有命令，不推荐大面积使用，反而增加调试难度。
- **代理/网关容器环境**：如果脚本运行在 Docker 容器里，容器的 LANG/LC_ALL 环境变量也会影响 .NET 的编码选择。Windows 容器里也建议设 `chcp 65001`，但 PowerShell 内仍需要上面提到的显式编码设置。

## 可复用建议

封装一个保险套壳函数，后续所有中文 API 调用都通过它，固化编码逻辑：

```powershell
function Invoke-CnApi {
    param(
        [Parameter(Mandatory)] [string]$Uri,
        [hashtable]$Headers = @{}
    )
    $Headers['Accept-Charset'] = 'utf-8'
    $resp = Invoke-WebRequest -Uri $Uri -Headers $Headers -UseBasicParsing
    $raw = [System.Text.Encoding]::UTF8.GetString($resp.RawContentStream.ToArray())
    return $raw | ConvertFrom-Json
}
```

在脚本开头统一加载该函数，所有对中文接口的调用全部走此入口。如果团队里用 PowerShell 作为 agent 工具调用层，放到模块里共享，避免每人踩一次坑。

## 总结

PowerShell 打坏中文 JSON 的核心矛盾在于：服务器的字节流对，客户端的解码路径错。它不像 Python `requests` 那样自动嗅探 `chardet`，而是严格（有时过于武断）依赖 HTTP 头或 .NET 默认回退规则。对于 OpenClaw 这类需要稳定文本流的系统，沉默的编码错配堪称“数据杀手”。处理方式可以归结为**三件事**：主动以 UTF-8 解码流，显式设置输出和文件写入编码，并且封装入口防止日后蔓延。一次配置，持续复用，远比事后清洗受损文本划算。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/33733cfd167d0844.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/6636c4dd74db27e5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/2263d0dd5f88829e.png)

