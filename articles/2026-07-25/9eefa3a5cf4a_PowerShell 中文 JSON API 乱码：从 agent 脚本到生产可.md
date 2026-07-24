---
title: PowerShell 中文 JSON API 乱码：从 agent 脚本到生产可用的工程实践
feedId: 30328
source: 综合讨论
publishedAt: 2026-07-25
---

## 背景

在 Windows 上写自动化工具、构建 OpenClaw 的插件或 MCP 服务的开发者，几乎绕不开 PowerShell。只要你的 agent 需要调用第三方中文 API（如翻译、文案生成、知识库检索），返回的 JSON 里夹杂着大段中文，很快就会遇到一个经典问题：**Invoke-RestMethod 拿到的是正常的对象，一落盘或者一进管道就变成乱码**；又或者控制台输出看着没问题，但用 `Out-File` 保存后，其他程序读到的全是问号或方块。

这个问题不是 bug，而是 Windows 生态下编码处理的必然结果。理清原因之后，其实三行代码就能根治。本文以实际 agent 工程为背景，还原问题、分析根因，并给出可复用的封装方案。

## 问题：看似能跑，实则已坏

最常见的场景是这样：我们用 PowerShell 调用一个返回中文 JSON 的 API。

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/zh/agent" -Method Post -Body $body
$response.data.text   # 控制台正常显示"这是一段中文"
```

看起来毫无问题，于是顺手写入文件供后续流程使用：

```powershell
$response.data.text | Out-File -FilePath "output.txt"
```

下游的 Python/Node.js 脚本一读，乱码。即使加上 `-Encoding UTF8`，仍然可能在某些机器上随机翻车。更隐蔽的情况是，agent 脚本通过管道把返回值传给另一个进程时，接收端看到的字符串已经是损坏的。

排查之后会发现，真正出问题的往往不是 API 本身的返回，而是 PowerShell 的 **输出编码**，以及文件写入时的 **BOM 与默认编码**。

## 根因拆解

### 1. 控制台编码 ≠ 管道/文件编码

PowerShell 控制台 (conhost / Windows Terminal) 默认使用当前系统区域设置的代码页，例如简体中文系统是 `936` (GBK)。**当你在控制台上打印字符串时，PowerShell 会将内部 UTF-16 字符串转换为活动的 OEM 代码页**。这就是直接看内容似乎正常的原因——控制台替你"反转"了一次。

但是，一旦你将结果通过管道输出给外部程序，或者用重定向写入文件，**编码就变成了 PowerShell 的 `$OutputEncoding` 变量所指定的编码**。这个变量默认是 `ASCII`（在 Windows PowerShell 5.1 中），于是所有非 ASCII 字符（中文）被强制转换为 `?`。

### 2. Out-File / Set-Content 的默认编码陷阱

`Out-File` 和 `Set-Content` 在 Windows PowerShell 5.1 中的默认编码是 **UTF-16 LE** 或 **带 BOM 的 UTF-8**，取决于文件系统和 PowerShell 版本。但很多跨语言工具（特别是 Unix 生态的工具链）期望的是 **无 BOM 的 UTF-8**。带 BOM 的文件交给 Python `open()` 默认模式，可能出现首字符异常；如果交给只接受纯 ASCII 的旧接口，BOM 会直接变成可见乱码。

### 3. Invoke-RestMethod 的"假象"

`Invoke-RestMethod` 会自动解析 JSON 并返回 PSCustomObject，这个过程是在内存里完成的，**对象中的字符串仍然是正确的 .NET 字符串**。所以访问属性时不会看到乱码。但当你将属性值输出到管道时，上述的编码转换就开始作祟。很多开发者误以为有了对象就没有编码问题，实际上编码问题只发生在**跨进程边界**和**持久化**的时候。

## 工程化解决方案与步骤

这里给出一个可直接嵌入 agent 脚本或 MCP 工具调用的安全函数，统一处理输出编码，避免散落各处的 `-Encoding` 参数。

```powershell
function Invoke-SafeApi {
    [CmdletBinding()]
    param(
        [string]$Uri,
        [hashtable]$Headers,
        [string]$Method = 'Get',
        [string]$Body
    )

    # 1. 强制调用时使用 UTF-8 请求体
    if ($Body) {
        $utf8NoBom = New-Object System.Text.UTF8Encoding $false
        $bodyBytes = $utf8NoBom.GetBytes($Body)
    }

    $params = @{
        Uri        = $Uri
        Method     = $Method
        Headers    = $Headers
    }
    if ($Body) { $params['Body'] = $bodyBytes }

    # 2. 获取原始响应字节流，避开自动解码
    $response = Invoke-WebRequest @params
    $rawBytes = $response.Content

    # 3. 无 BOM UTF-8 解码为字符串
    $utf8NoBom = New-Object System.Text.UTF8Encoding $false
    $text = $utf8NoBom.GetString($rawBytes)

    # 4. 解析为对象（可选）
    $obj = $text | ConvertFrom-Json

    return @{
        Text   = $text
        Object = $obj
    }
}
```

**核心思路**：跳过 `Invoke-RestMethod` 的自动解析，用 `Invoke-WebRequest` 获取原始字节，然后明确声明无 BOM UTF-8 解码。这样可以避免 `-ContentType` 被忽略、响应被错误解码的问题。

写入文件时，也必须显式指定编码为无 BOM UTF-8：

```powershell
$result = Invoke-SafeApi -Uri "https://api.example.com/zh/data"
[System.IO.File]::WriteAllText("output.txt", $result.Text, (New-Object System.Text.UTF8Encoding $false))
```

或使用 `Set-Content` 配合 `-Encoding` 参数（PowerShell Core 6+ 支持直接写 `UTF8NoBOM`）。

## 踩坑与可复用建议

- **不要在脚本顶部直接设置 `$OutputEncoding` 为 `[System.Text.UTF8Encoding]::new()` 就了事**。虽然这能缓解管道输出问题，但当脚本被别的进程以非交互方式调用时，控制台代码页可能又不一样，造成一些老旧工具接收参数时出错。建议**最小化作用域**，只在需要管道传递中文的函数内部临时修改。
- **跨平台兼容**：上述代码使用 .NET API，在 Windows PowerShell 5.1 和 PowerShell 7+ 上均可运行。如果你的 agent 运行在 PowerShell Core 上，`Set-Content -Encoding utf8NoBOM` 更简洁。
- **Agent 通信协议**：当通过 stdout 将数据交给另一个 agent 或 MCP 客户端时，务必配合 `[Console]::OutputEncoding` 统一。简单做法是启动进程时指定 `-Encoding UTF8`，并在 PowerShell 侧使用 `[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new()`。
- **日志落盘**：建议所有日志文件统一采用无 BOM UTF-8，且在脚本入口即调用一次 `$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'`（PS 5.1 中相当于带 BOM，但有总比 ASCII 好）。更推荐迁移到 `Add-Content` 配合 `Unicode` 或直接使用 System.IO.StreamWriter。

## 总结

在 Windows 上调用中文 API 时看到的乱码，本质是 PowerShell 的编码边界行为与跨平台工具链预期之间的错配。记住三件事就不会再掉坑：
1. **内存中永远是好的**，问题出在出进程和落盘。
2. **永远用字节流 + 显式 UTF-8 无 BOM**，不要依赖任何默认编码。
3. **封装一个安全函数**，让 agent 插件调用时不必再考虑编码细节。

这些处理不仅适用于 OpenClaw 开发，任何在 Win 下用 PowerShell 写自动化的场景都会受益。把编码的脏活抽象出来，是工程化的第一步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/a507d1eb043c50a0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/79d371444c7ab76c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/ba2ae93cf7f0fd87.png)

