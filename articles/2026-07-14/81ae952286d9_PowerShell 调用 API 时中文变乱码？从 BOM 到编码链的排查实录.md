---
title: PowerShell 调用 API 时中文变乱码？从 BOM 到编码链的排查实录
feedId: 29081
source: 综合讨论
publishedAt: 2026-07-14
---

## 背景：一条“简单”的 JSON 请求怎么就不通？

在 Windows 上为 Agent 或 MCP 插件编写调用外部 API 的脚本时，PowerShell 常常是第一选择——无需安装额外依赖，直接 `Invoke-RestMethod` 就能发起 HTTP 请求。尤其是面向国内服务的 JSON API，传入中文参数（如发送消息、创建文档）几乎是刚需。

问题来了：**一切看起来正常的请求，服务端报 400、签名校验失败或“参数格式错误”，日志里中文却显示为一串 `????` 或类似 `æ±‰å­—` 的乱码。** 同样的内容用 Python、curl 甚至浏览器都正常，唯独 PowerShell 总会“打坏”中文。

这不是偶然。它在工程里反复出现，根源在于 Windows 上 PowerShell 5.1 默认的字符编码行为，与 UTF-8 世界的预期形成了断裂。

## 问题定位：不只是编码，是编码链

多数人的第一反应是“设置编码”，于是会在脚本里加上 `chcp 65001` 或 `[Console]::OutputEncoding`。但这只能解决控制台输出显示，并不能解决 HTTP 请求体的编码。

真正的问题是 **PowerShell 将字符串转换为请求体字节流时，并未使用 UTF-8，而服务端假设接收的是 UTF-8 编码的 JSON**。

在 Windows PowerShell 5.1 中，当你构建一个 hashtable 并自动序列化为 JSON（`ConvertTo-Json`），或者直接把中文字符串当作请求体发送时，命令行环境、.NET 的 `[Text.Encoding]` 默认值、以及 `Invoke-RestMethod` 的实际行为串成了一条非 UTF-8 的链条：

- **PowerShell 会话编码**：`$OutputEncoding` 默认是 ASCII-based 的代码页（如 `us-ascii`），影响与外部程序交互和部分命令的输出。
- **`ConvertTo-Json` 的转义行为**：PowerShell 5.1 内置的 `ConvertTo-Json` 会将非 ASCII 字符转义为 `\uXXXX` 形式，这本身是合法 JSON，但如果深度设置不当或结合其他编码步骤，容易造成二次转码。
- **`Invoke-RestMethod` 和 `Invoke-WebRequest` 的字符集选择**：当传递一个字符串作为 `-Body` 时，这两个 cmdlet 会使用 `[Text.Encoding]::Default` 来编码字符串。在中文 Windows 上，默认编码往往是 `GB2312` 或 `GBK` 的代码页（例如 936），而非 UTF-8。
- **媒体类型缺失**：没有指定 `Content-Type: application/json; charset=utf-8`，部分服务端会回退到自行猜测编码，进一步放大错误。

这些环节叠加，导致发给服务端的字节流是 GBK 编码的“JSON”，而服务端按 UTF-8 解析，中文自然沦为乱码。签名校验失败则是因为原始字符串与签名用的 UTF-8 字节不一致。

## 做法与步骤：让 PowerShell 真正发出 UTF-8 JSON

在工程中，我们不可能要求所有服务端兼容 GBK。更务实的做法是强制让 PowerShell 发送 UTF-8 编码的请求体。

### 方案一：显式指定 UTF-8 字节流（推荐，兼容性最好）

不再传递字符串，而是构造出 UTF-8 编码的字节数组，交给 `Invoke-RestMethod`：

```powershell
# 1. 构建请求体对象
$body = @{
    msg = "你好，世界"
    ts  = 1710000000
} | ConvertTo-Json -Compress

# 2. 将 JSON 字符串显式转为 UTF-8 字节数组
$utf8Bytes = [System.Text.Encoding]::UTF8.GetBytes($body)

# 3. 发送请求，-Body 接受字节数组，并带上 Content-Type
$response = Invoke-RestMethod -Uri "https://api.example.com/echo" `
    -Method Post `
    -Body $utf8Bytes `
    -ContentType "application/json; charset=utf-8"
```

关键点：
- `ConvertTo-Json` 生成的是 .NET 字符串，默认是 Unicode 内存表示，没问题。
- `GetBytes` 显式指定 UTF-8，从根本上避免了 Windows 默认代码页的影响。
- `-ContentType` 必须包含 `charset=utf-8`，让服务端明确编码。
- 直接传字节数组时，`Invoke-RestMethod` 不会再次编码，直接使用该字节流。

### 方案二：设置会话级的输出编码（适合简单调用）

如果只是快速调试，也可以通过修改 `$OutputEncoding` 变量来影响某些 cmdlet 的行为，但**不够彻底**：

```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false) # 无 BOM 的 UTF-8
$body = @{ msg = "测试" } | ConvertTo-Json
Invoke-RestMethod -Uri "https://api.example.com/echo" -Method Post -Body $body -ContentType 'application/json'
```

注意，这里 `-Body` 仍然是字符串，`Invoke-RestMethod` 在某些 PowerShell 版本中受 `$OutputEncoding` 影响，但在 PowerShell 5.1 中该变量主要影响控制台和管道输出，对 HTTP body 编码的影响并不可靠。因此**方案一更稳健**。

### 方案三：升级到 PowerShell 7+

PowerShell 7 (Core) 已在多平台统一 UTF-8 无 BOM 作为默认编码。在 PowerShell 7 中，`$OutputEncoding` 默认为 UTF-8，且 `Invoke-RestMethod` 在传递字符串 body 时也会使用 UTF-8（除非显式覆盖）。如果你的工具链允许，直接切换到 PowerShell 7 是最彻底的解决方案。

## 踩坑点：那些你以为改了但没改的地方

### 1. `chcp 65001` 只救控制台

`chcp 65001` 改变的是控制台代码页，影响的是屏幕上显示的中文和向控制台程序写入/读取的字节解释，**不影响** `Invoke-RestMethod` 的内部编码选择。大量教程让人执行 `chcp 65001` 后测试中文显示好转，然后就以为问题解决了，其实请求体仍是 GBK。

### 2. `[Console]::OutputEncoding` 的幻觉

设置 `[Console]::OutputEncoding = [Text.Encoding]::UTF8` 同样只处理控制台输出，不触及 HTTP 请求编码。曾有人在脚本开头设置后看到控制台不乱码，误以为 API 调用也会跟着正常，结果浪费了大半天排查服务端签名。

### 3. `-ContentType` 只写 `application/json` 不够

有些服务端严格校验 `charset` 参数。若只写 `application/json`，服务端可能会回退到默认编码（如 ISO-8859-1）并错误解析 UTF-8 字节流。务必带上 `; charset=utf-8`。

### 4. BOM 的灵魂纠缠

Windows 上的记事本或部分工具保存 UTF-8 时会默认添加 BOM。如果在 PowerShell 中读取文件传入请求体，务必用 `-Encoding UTF8`（PowerShell 7 的 `-Encoding utf8NoBOM`）读入。否则 JSON 开头的 BOM 字节 `EF BB BF` 会导致服务端 JSON 解析失败。

## 可复用建议

- **凡涉及中文 JSON 请求，统一使用字节数组方式**，封装成一个函数 `Invoke-RestMethodUtf8`，团队内部复用，避免每次都踩坑。
- **脚本内明确定义 HTTPS 客户端 Encoding**：可以用 `[System.Net.ServicePointManager]::ServerCertificateValidationCallback` 等方式确保全局编码预期，但更推荐字节数组这种局部显式控制。
- **在 CI/CD 或 Agent 的环境中，强制设置 `[System.Environment]::SetEnvironmentVariable('DOTNET_SYSTEM_GLOBALIZATION_INVARIANT', 'false')` 等变量，避免因环境差异导致非 UTF-8 默认行为。**
- **编写自动化脚本时，注释醒目标记 “使用 UTF-8 字节流，勿改为字符串传 Body”**，防止后续维护者因为简洁而改回字符串，悄悄引入乱码回归。

## 总结

Windows 上 PowerShell 默认编码与 UTF-8 的错位，是中文 API 集成中常见的暗坑。它不抛出异常，只在服务端留下一串问号或签名错误，让人往更远的方向排查。解决的核心在于**不再信任默认编码，而是显式将 JSON 字符串转为 UTF-8 字节数组再交给 HTTP 客户端**。

对于正在构建 OpenClaw 插件、Agent 工具链或自动化管道的工程师，这条经验或许能帮你节省一个下午的沮丧调试。当你下次看到 `Invoke-RestMethod` 返回 “参数非法” 时，记得检查从内存到网卡的编码路径，而不是急着怀疑 API 文档。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/4c1783cd03b3ca66.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/88e40b20a3377e87.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/80b98e23662aaf4d.png)

