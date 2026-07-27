---
title: PowerShell 调用 JSON API 时中文被“打坏”的根本原因与工程化修复
feedId: 30654
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景：为什么你的 Agent 插件突然处理不了中文

在 Windows 上为 OpenClaw 或 MCP 自动化链路编写插件时，很多开发者会直接用 PowerShell 调用内部或外部 API，获取 JSON 数据后再交给下游处理。一个非常常见的错误场景是：API 返回的 JSON 中包含中文，但下游 Agent 收到的却是乱码、问号甚至抛出解析异常。这种问题在 **Invoke-RestMethod** 或 **Invoke-WebRequest** 与 `Out-File` 管道结合时尤其高发。

根源并非 API 的恶意篡改，而是 PowerShell 在 Windows 环境下对字符编码的“自以为是”行为。

## 问题本质：双重编码失控

PowerShell 在 Windows 上的默认行为有两个关键编码设置：

1. **控制台输出编码 (`[Console]::OutputEncoding`)**：默认会被映射到当前系统区域设置（如简体中文 Windows 是 `GB2312` 代码页 936），用于解释控制台中写入的字节流。
2. **管道输出编码 (`$OutputEncoding`)**：当 PowerShell 将字符串通过管道传给其他原生命令（如 `Out-File`、`Set-Content`，或者重定向到文件），会使用此编码将字符转换为字节。这个变量在 Windows PowerShell 5.1 中默认是 `ASCII`（即代码页 20127），而不是 UTF-8。

当 API 返回的 JSON 本身就是 UTF-8 编码（绝大多数现代 API 如此），而你又用如下类似方式写入文件：

```powershell
Invoke-RestMethod -Uri "https://api.example.com/data" | ConvertTo-Json | Out-File result.json
```

此时发生的事情是：

- `Invoke-RestMethod` 正确地将 UTF-8 字节流解码为 .NET 字符串（内存中为 Unicode，正确）。
- `ConvertTo-Json` 输出的是 UTF-8 字符串，默认也会正确表示中文。
- 但在管道进 `Out-File` 时，`$OutputEncoding` 被设置为 ASCII，所有非 ASCII 字符（中文）被替换为“?”。
- 如果不用 `Out-File` 而直接用重定向 `>`，则受 `[Console]::OutputEncoding` 影响，中文可能被转码为 GB2312，此时如果下游预期 UTF-8，乱码同样会发生。

更隐蔽的问题是：即便你显式使用了 `-Encoding UTF8`（Windows PowerShell 5.1），`Out-File` 输出的 UTF-8 文件也会被插入 **BOM**（字节顺序标记）。如果下游解析器（比如某些 JSON 解析库、Node.js 的 `fs.readFileSync('file','utf-8')` 不剥离 BOM）把这个 BOM 当作有效数据的一部分，会导致 JSON 解析失败或在字符串头部出现一个不可见的 `\uFEFF`。

## 可靠复现步骤

1. 在简体中文 Windows 的 PowerShell 5.1 中执行以下命令：
   ```powershell
   $json = '{"name": "你好世界"}'
   $json | Out-File -Encoding UTF8 test.json
   Get-Content test.json -Encoding UTF8
   ```
   表面上看文件内容正常，但用十六进制查看会发现文件开头有三个字节 `EF BB BF`（BOM）。如果将这个文件传给 Node.js 脚本用 `JSON.parse()` 处理，可能直接报错。

2. 模拟真实的 API 调用场景：
   ```powershell
   $resp = Invoke-WebRequest -Uri "https://httpbin.org/anything" -Body '{"msg":"测试"}' -ContentType "application/json" -Method POST
   $resp.Content | Out-File -FilePath raw.json
   ```
   控制台可能显示中文正常，但 `raw.json` 中中文全部变为乱码或者 `?`。

## 工程化修复：从临时钩子到永久方案

### 方案一：在 Windows PowerShell 5.1 中修补（临时）
如果你暂时无法迁移到 PowerShell 7，可以通过在脚本头部显式设置编码来避免问题：

```powershell
# 设置控制台输出编码为 UTF-8，能改善重定向操作
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
# 设置外部命令/管道输出编码为 UTF-8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

然后使用 `Out-File` 时，为了去掉 BOM，不要用原生的 `Out-File`，而是要写一个显式的 .NET 写入：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/data"
$jsonString = $response | ConvertTo-Json -Compress
[System.IO.File]::WriteAllText("result.json", $jsonString, [System.Text.UTF8Encoding]::new($false))
```

第三个参数 `new UTF8Encoding($false)` 明确生成 **无 BOM** 的 UTF-8 编码。

### 方案二：迁移到 PowerShell 7 (pwsh) —— 推荐
PowerShell 7 做了两个关键改进：
- 默认将所有输出编码设为 UTF-8（包括 `$OutputEncoding` 和 `[Console]::OutputEncoding`）。
- `Out-File` 新增了 `-Encoding utf8NoBOM` 参数，直接生成无 BOM 的 UTF-8 文件。

因此，相同的逻辑在 pwsh 中可以写成最自然的形式，且不会产生任何乱码或 BOM 困扰：

```powershell
Invoke-RestMethod -Uri "https://api.example.com/data" | ConvertTo-Json | Out-File -Encoding utf8NoBOM result.json
```

对于 OpenClaw 或 MCP 插件，如果必须依赖 PowerShell 作为执行器，将脚本执行器指定为 `pwsh.exe` 通常是最彻底的办法。

## 踩坑点清单

- **BOM 是隐蔽的 JSON 杀手**：许多跨平台工具（例如 Linux 上的 `jq`、Golang 的 `encoding/json`）期望 JSON 文件以 `{` 开头，BOM 会导致解析失败，错误信息却极不直观（往往只是 `invalid character`）。
- **`Set-Content` 默认也会用 `$OutputEncoding`**，而且 `-Encoding UTF8` 同样会带 BOM。
- **控制台显示的“假正常”**：因为控制台本身可能用 GBK 渲染，你看到的是二次解码后的结果，误以为数据没问题，等文件传出去才暴露。
- **管道中混用原生命令**：将 PowerShell 输出通过 `|` 传递给 `cmd.exe` 或 `python.exe` 时，$OutputEncoding 决定了传递的字节编码，必须设置为 UTF-8 才能与 Python 端期望一致。

## 可复用建议：在自动化链路中固化编码策略

1. **统一使用 pwsh 7+**，并在脚本首行添加 `#requires -Version 7.0` 确保执行环境无误。
2. **文件写入统一采用 `[System.IO.File]::WriteAllText(path, content, [System.Text.UTF8Encoding]::new($false))`** 这种 .NET 方法，而不是依赖 Power​​Shell cmdlet 的 `-Encoding` 参数，因为后者在 Windows PowerShell 中行为永远不一致。
3. **在 OpenClaw 的 MCP 工具配置中，如果工具由 PowerShell 脚本实现**，检查是否设置了正确的编码管道：可以在脚本开头直接硬编码 `[Console]::OutputEncoding = [System.Text.UTF8]::new($false); $OutputEncoding = [System.Text.UTF8]::new($false)`。
4. **对 JSON 输出做 BOM 防御**：在消费端（例如 Node.js）读取文件时，先使用 `.replace(/^\uFEFF/,'')` 去掉可能存在的 BOM，或统一用 `strip-bom` 这样的库处理。
5. **测试时用十六进制确认**：不要只看文本，使用 `Format-Hex .\result.json` 检查开头是否有 `EF BB BF`。

## 总结

Windows 上的 PowerShell 在处理中文 JSON 时的“打坏”行为，本质是**默认编码假设与 UTF-8 世界不兼容**。这不是 API 的问题，也不是 JSON 的问题，而是数据管道中编码协商失败。对于依赖 PowerShell 做基础自动化的 OpenClaw 或 MCP 用户，最稳妥的做法是升级到 PowerShell 7，并强制所有输出为 UTF-8 无 BOM。在暂时无法升级的环境中，掌握 `[Console]::OutputEncoding` 与 `$OutputEncoding` 的组合设置，并采用 .NET 直接写文件，可以稳定地绕过这个持续了十多年的巨坑。

字符集从来都是基础设施中最容易忽视、却最容易引发级联错误的组件。在 Agent 和自动化链路的每一个环节，显式声明“我们使用 UTF-8 无 BOM”是工程化的底线。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/c6e58d8f7244fc7f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/53a84e7fa03ce671.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/587a01662c7362bf.png)

