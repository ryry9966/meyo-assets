---
title: PowerShell 中文 JSON “打坏”实录：从自动化 Agent 踩坑到一劳永逸的编码修复
feedId: 30610
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景：当 Agent 工具收到一坨乱码

在 Windows 上跑 OpenClaw 或 MCP 插件时，很多同学习惯用 PowerShell 编写自动化脚本，调用中文 API 获取 JSON 结果，然后交给 Agent 做下一步推理。理想流程是这样的：

```
API 调用 → JSON 解析 → 提取字段 → 喂给 LLM
```

但现实常常变成：

```
API 调用 → 乱码 → 解析失败 → Agent 胡说八道
```

表现为：终端里显示 `é¸Ÿç»"`、`鍖椾含`，或者 `$json.name` 直接是 `??????`。同样的请求用 Python 或 curl 就完全正常。问题就锁定在 **PowerShell 的编码管线**。

这绝不是个例。表面是乱码，底层是 Windows 遗留编码、PowerShell 版本、脚本文件 BOM 三者叠加的系统性坑。如果不根治，你的自动化工具链随时会在凌晨告警里给你一串莫名其妙的“天书”。

---

## 问题复现：明明 API 返回正常，变量里却坏了

先用一个最简复现场景。假设有一个中文 API：

```
GET https://api.example.com/city
响应: {"code":0,"data":{"city":"北京","weather":"晴"}}
```

在 PowerShell 5.1（Win10/WinServer 预装版本）中这样调用：

```powershell
$resp = Invoke-WebRequest -Uri "https://api.example.com/city"
$json = $resp.Content | ConvertFrom-Json
$json.data.city   # 输出：??????
```

如果直接看 `$resp.Content`，字符串已经是乱码：`{"code":0,"data":{"city":"äº¬åŒ","weather":"æ™´"}}`。

原因出在 `Invoke-WebRequest` 对响应体的自动解码。服务器返回的 `Content-Type` 通常带着 `charset=utf-8`，但 Windows PowerShell 的 HTTP 客户端在某些情况下会忽略 charset，使用默认的 **ISO-8859-1** 解码，这就把原本的 UTF-8 字节序列强行映射为拉丁字符，产生了著名的“浣熊”乱码。

`Invoke-RestMethod` 也会碰到同样问题，因为底层共享连接。

---

## 为什么这么做会打坏中文：三股编码暗流

### 1. 响应流的罪魁：默认解码器

Windows PowerShell 5.1 的 `System.Net.HttpWebResponse` 获取字符串时，如果服务器未严格指明 charset，就会回退到 `[System.Text.Encoding]::Default`，也就是系统活动代码页（中文系统是 GBK，但 GBK 对 UTF-8 的字节序列解码会变成乱码）。即便指明了 UTF-8，旧版 .NET Framework 的某些实现仍会优先使用响应头里的规范名称，而 `Invoke-WebRequest` 早期版本在内部把响应体先当作字节数组再转字符串时选错了编码。

### 2. 控制台输出编码的二次伤害

即使变量内部已经是正确字符串（比如通过后面讲的方法修复了），直接用 `Write-Output` 打印中文也可能变乱码。因为控制台窗口有自己的编码 `[Console]::OutputEncoding`，默认与系统代码页一致。如果内部字符串是 UTF-16，控制台却用 GBK 渲染，就会再次损坏。

### 3. 脚本文件本身的编码地雷

很多新手用记事本保存 .ps1 文件，默认是 ANSI（GBK）。如果脚本里硬编码了中文字符串，例如 `$city = "北京"`，保存为 ANSI 后，PowerShell 读取时按 ANSI 解释，但在某些代码页切换的场景下会丢失信息。PowerShell 5.1 对无 BOM 的 UTF-8 支持很差，建议必须保存为 **UTF-8 with BOM**。

---

## 做法与步骤：从手动修复到一劳永逸

以下步骤在 Windows PowerShell 5.1 和 PowerShell 7+ 下分别给出最佳实践。

### 步骤 1：修复 API 响应解码

不再使用 `Invoke-WebRequest` 的字符串内容，直接拿到原始字节，用 UTF-8 解码：

```powershell
$resp = Invoke-WebRequest -Uri "https://api.example.com/city" -UseBasicParsing
$rawBytes = $resp.RawContentStream.ToArray()
$jsonStr = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$json = $jsonStr | ConvertFrom-Json
$json.data.city   # 正常输出“北京”
```

或者更简洁，用 `Invoke-RestMethod` 配合手动设置响应编码（PS5.1 需绕道）：

```powershell
$resp = Invoke-WebRequest -Uri "https://api.example.com/city" -UseBasicParsing
$str = [System.Text.Encoding]::UTF8.GetString($resp.RawContentStream.ToArray())
$json = $str | ConvertFrom-Json
```

在 PowerShell 7 中，可以直接使用 `-SkipHttpErrorCheck` 等参数，且默认编码已改为 UTF-8，大部分情况无需额外处理。

### 步骤 2：统一控制台与文件输出编码

在脚本开头强制设置：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

这样 `Write-Output` 输出中文不会变乱码，`Out-File`、`Set-Content` 默认就是 UTF-8。

如果需要写入日志文件，直接用 `Add-Content -Path $log -Value $line -Encoding UTF8`。

### 步骤 3：永久性环境配置（可选）

在 `$PROFILE` 文件中加入上述编码设置，每个新会话自动生效。PowerShell 7 用户基本无需操心，因为默认已经是 UTF-8。

---

## 踩坑点：几个让你怀疑人生的细节

1. **控制台修复了，但重定向到文件又坏了**  
   单独运行脚本 OK，用 `.\script.ps1 > result.txt` 重定向，文件内容乱码。这是因为重定向操作符 `>` 实际上调用 `Out-File`，编码由 `$PSDefaultParameterValues` 控制。如果没有预设，则用 Unicode（UTF-16 LE）。所以务必添加 `$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'`，或者直接改用 `| Out-File -Encoding UTF8`。

2. **Invoke-RestMethod 的魔法与失灵**  
   `Invoke-RestMethod` 会尝试自动将 JSON/XML 解析为对象，但当响应头缺失 charset 时，解析失败会直接报错，而不是返回乱码。这反而是一种保护。但如果强行用 `-ContentType` 参数，并不能改变解码行为，那是设置请求头的。要根治还是得从原始字节入手。

3. **Windows 计划任务中的脚本执行环境**  
   很多 Agent 工具由计划任务或服务触发，此时 PowerShell 环境可能没有加载 `$PROFILE`，控制台编码设置也不会生效。建议在每个自动化脚本内部，显式写入编码设置，不要依赖外部配置。

4. **PowerShell 5.1 与 7 并存时的默认切换**  
   有些系统默认 `powershell.exe` 是 5.1，`pwsh.exe` 是 7。如果你用 5.1 运行 7 的脚本（有 `#requires -Version 7` 会报错，但无声明时会静默失败），需确保运行环境与脚本期望一致。推荐 Agent 启动命令明确使用 `pwsh.exe -File script.ps1`。

---

## 可复用建议：给自动化插件的编码安全层

基于上述踩坑，可以抽象一个通用的安全调用函数，在你的 OpenClaw 插件或 MCP 工具脚本里复用：

```powershell
function Invoke-Utf8RestMethod {
    param($Uri, $Headers)
    $resp = Invoke-WebRequest -Uri $Uri -Headers $Headers -UseBasicParsing
    $jsonStr = [System.Text.Encoding]::UTF8.GetString($resp.RawContentStream.ToArray())
    return $jsonStr | ConvertFrom-Json
}
```

对于 Agent 工具描述，建议明确提示“所有 API 响应均以 UTF-8 处理”，并将输出字段统一重新转为 JSON 字符串传递给 LLM 时，也用 `ConvertTo-Json -Compress` 确保编码一致。

如果你的 Agent 管道里还有 Python 或 curl 节点，建议中间数据全部使用 UTF-8 编码的 JSON 文件传递，PowerShell 通过 `Get-Content -Encoding UTF8` 读取，从根本上避免进程间编码错配。

---

## 总结

PowerShell 在 Windows 上处理中文 JSON 的乱码问题，本质是 **HTTP 响应解码默认行为 + 控制台输出编码 + 脚本文件编码** 三座大山叠加的结果。解决之道不是零散地改一两处，而是建立一个编码安全脚手架：

- 永远从原始字节流用 UTF-8 主动解码 API 响应；
- 脚本开头显式设置控制台、文件输出编码；
- 保存脚本时必须选 UTF-8 with BOM（PS5.1）；
- 自动化任务中不自带环境，请内部显式配置。

完成这四步后，你会发现那些曾经让你抓狂的“鸟语”彻底消失了，Agent 拿到的城市名终于变回“北京”。对于以 OpenClaw 为核心的自动化栈，这种底层稳定性比炫技功能重要得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/0890909ecb9be239.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/37455a61a596ed93.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/bbddd0b00f8342b6.png)

