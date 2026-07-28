---
title: PowerShell 中文 JSON 乱码全解析：从控制台到文件输出的工程化修复
feedId: 30755
source: 综合讨论
publishedAt: 2026-07-28
---

## 背景：Agent 流水线里最让人头疼的“问号”

在 OpenClaw 社区，很多自动化实践都在 Windows 上用 PowerShell 串联一切：调用大模型 API、触发 MCP 工具、把中间结果写入 JSON 文件供下一环节消费。只要应答里出现中文，噩梦就开始了——控制台输出是“????”，重定向的 JSON 文件被下游工具报“非法 UTF-8 序列”，哪怕直接复制粘贴都是损坏的。

这不是某个 API 的问题，而是 PowerShell 在编码处理上的多重不一致。本文从一个可复现的最小场景出发，拆解乱码的真正成因，并给出跨版本、跨环境都能用的工程化修复。

## 问题复现：一个 API 调用，三种乱码

假设我们有一个最简单的 REST API，返回 `{"msg":"你好，OpenClaw"}`。用 PowerShell 5.1 在简中 Windows 下调用：

```powershell
$resp = Invoke-RestMethod -Uri http://127.0.0.1:5000/api/demo
$resp.msg   # 控制台可能显示正常，也可能显示 ??
$resp | ConvertTo-Json | Out-File -FilePath out.json
# 或重定向
$resp | ConvertTo-Json > out2.json
```

接下来你会遇到三类典型乱码：

1. **控制台乱码**：`¡Áåô` 这样的乱码，常见于 `[Console]::OutputEncoding` 仍为 OEM 代码页（如 936）时，直接输出宽字符。
2. **文件 BOM / 编码混乱**：`out.json` 实际是 UTF-16 LE 带 BOM，而很多解析器（jq、Python 默认读取）期待 UTF-8 without BOM，导致报错。
3. **JSON 中汉字被转义**：`ConvertTo-Json` 把中文变成了 `\uXXXX`，下游消费者看到的 `"msg":"\u4f60\u597d"` 失去了可读性，且增加了不必要的解析负担。

真正影响工程化可靠性的，是后两者——数据在流水线里被悄悄改写，但看起来似乎“写成功了”。

## 原因拆解：三个编码变量在打架

PowerShell 对编码的控制实际上分散在三个核心变量里，弄清它们才能定位问题：

- **`[Console]::OutputEncoding`**  
  控制台呈现代码。当 PowerShell 将字符串交给控制台宿主（conhost 或 Windows Terminal），会按此编码转码。默认是系统 OEM 代码页（简体中文 936，即 GBK）。只要 API 返回内容是 UTF-8 解码后的 .NET 字符串（内部是 UTF-16），输出时就会用 936 编码，遇上 936 不支持的字符就变成 `?`。

- **`$OutputEncoding`**  
  管道/重定向编码。PowerShell 把数据发给外部程序（例如 `python`、`jq`）或使用重定向运算符 `>` 时，会按 `$OutputEncoding` 编码。PS5.1 中这个变量默认是 ASCII (!)，导致 `>` 写入的文件里中文字节变成 `?`。即使你将 `[Console]::OutputEncoding` 改为 UTF-8，`>` 仍然可能损坏，因为两者独立。

- **`Out-File` / `Set-Content` 的默认 `-Encoding`**  
  PS5.1 里 `Out-File` 默认是 `Unicode`（即 UTF-16 LE），`Set-Content` 默认是 `Default`（ANSI）。如果你不显式指定 `-Encoding UTF8`，文件编码就是错误的。PS7 中 `Out-File` 默认改为 UTF8 without BOM，但为了兼容性，显式指定永远是安全的。

额外的坑：`Invoke-RestMethod` 返回的已经是解析好的对象，对象属性中的字符串是 .NET 原生 UTF-16，但 `ConvertTo-Json` 在 PS5.1 中会默认将所有非 ASCII 字符转义为 `\uXXXX`，且无参数可关闭。这在日志可读性、与 LLM 协作时很麻烦。

## 工程化修复步骤

以下步骤在 PowerShell 5.1 和 7.x 下均可生效，且覆盖控制台、重定向、文件输出三种路径。

### 1. 在脚本开头锁定 UTF-8 编码环境

```powershell
# 控制台输出为 UTF-8，大多数终端（如 Windows Terminal 的 WSL 风格）能正确渲染
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

# 管道和重定向编码设为 UTF-8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

这两行必须放在脚本最顶部。对于交互式环境，可以在 profile 里设置，但不强制推荐，因为可能影响其他依赖 GBK 输出的老旧工具。

### 2. 拿到原始 JSON 字符串，避免重新序列化

如果你只需要把 API 返回的 JSON 原样落盘，不要用 `Invoke-RestMethod`（它二次解析），而用 `Invoke-WebRequest` 并取 `Content` 属性：

```powershell
$response = Invoke-WebRequest -Uri $apiUrl
$rawJson = $response.Content   # 已是正常的 UTF-8 字符串
$rawJson | Set-Content -Path response.json -Encoding UTF8
```

`Set-Content` 配合 `-Encoding UTF8` 在 PS5.1 和 PS7 中都会生成 UTF-8 without BOM 文件，对下游最友好。

### 3. 如果必须使用对象再序列化，强制关闭转义

当你需要对数据做变换后输出 JSON，使用 `ConvertTo-Json` 时需注意：

- **PowerShell 7+**：使用 `-EscapeHandling EscapeNonAscii`（或直接默认的 `EscapeHtml`，但后者也只转义 HTML 敏感字符；不转义中文在 PS7 已是默认行为，但为了明确，建议使用 `EscapeHandling` 明确控制）。
  
  但 PS7 默认就不会转义中文了。为安全写成：
  ```powershell
  $obj | ConvertTo-Json -EscapeHandling Default
  ```

- **PowerShell 5.1 无此参数**：只能换用 `Newtonsoft.Json` 或直接在 .NET 中处理。简化做法：如果目标只是带中文的 JSON 落盘，尽量用原始字符串方案，避免 `ConvertTo-Json`。

### 4. 统一文件写入行为

在脚本中设置默认参数，避免每次带 `-Encoding`：

```powershell
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['Set-Content:Encoding'] = 'utf8'
```

注意：`>` 和 `>>` 仍然受到 `$OutputEncoding` 控制，已经在前置步骤中设为 UTF-8。如果仍不放心，完全禁用重定向，改用 `Out-File -FilePath ... -Encoding utf8`。

### 5. 跨 PowerShell 版本的保障策略

- 所有 CI/CD 环境尽量用 **PowerShell 7+**，其默认编码行为更现代。
- 遗留环境（如 Windows Server 2016 自带 PS5.1）必须严格执行上述所有设置，并通过 smock 测试验证。

## 可复用封装建议

可以将上述设置打包成一个函数，在每个自动化脚本开头引入：

```powershell
function Initialize-PsUtf8 {
    [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
    $global:OutputEncoding = [System.Text.Encoding]::UTF8
    $global:PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
    $global:PSDefaultParameterValues['Set-Content:Encoding'] = 'utf8'
    Write-Verbose "PowerShell encoding chain set to UTF-8"
}
```

在 OpenClaw 的 Agent 或插件启动脚本中直接调用 `Initialize-PsUtf8`，就能杜绝大多数中文乱码。

## 总结

PowerShell 的编码并非“把一个设置改好就万事大吉”，而是由控制台输出、管道/重定向、文件写入三大独立变量共同决定。针对中文 JSON API 调用的场景，核心原则是：

- 始终显式指定 UTF-8，不依赖任何默认值；
- 优先保留原始 JSON 字符串，避免不必要的序列化与转义；
- 用 `$PSDefaultParameterValues` 降低犯错概率；
- 在 PS5.1 遗留环境里做全量验证。

做到这几点之后，你的 Windows 自动化流水线将再也不会因为一个中文字符而中断。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/e7f04e9f4f26e452.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/e0b53578bdfe5006.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/503c5f054c1a9640.png)

