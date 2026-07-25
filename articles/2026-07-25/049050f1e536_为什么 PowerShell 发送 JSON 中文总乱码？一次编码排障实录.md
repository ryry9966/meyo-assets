---
title: 为什么 PowerShell 发送 JSON 中文总乱码？一次编码排障实录
feedId: 30404
source: 综合讨论
publishedAt: 2026-07-25
---

## 背景

在 Windows 上做自动化，但凡涉及调用中文内容的 REST API，PowerShell 几乎一定会给你上一课。参数里带了用户姓名、请求体里写了备注、返回值里包含中文字段——只要一个环节没对齐，日志里就会出现一串 `ç»ä»¶æä»¶` 或者 `????`。

面向 Agent、MCP 插件或自动化流水线的实践用户，这个问题不只是“看起来乱”，它会导致下游逻辑出错：JSON 解析失败、签名校验不通过、数据库写入乱码、Agent 收到无法理解的指令。排查到最后，往往发现是 PowerShell 在编码处理上的历史设计和我们直觉不一致。

这篇文章不会搬运热点，也不会铺垫虚构的大项目，只还原一个真实的排障路径，给出可复用的工程方案。

## 问题现场

最小复现：Windows 10/11 自带的 PowerShell 5.1，向一个测试 API 发送中文 JSON。

```powershell
$body = @{
    name = "张三"
    action = "查询订单"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "https://httpbin.org/post" -Method Post -Body $body -ContentType "application/json"
$response.json.name
```

期望输出 `张三`，实际输出 `??` 或 `å¼ ä¸`。去 httpbin 的原始请求里看，收到的字符串是乱码。

问题根源看似简单：**PowerShell 5.1 在将字符串转换为请求字节流时，默认使用 ASCII 或系统 ANSI 编码，而不是 UTF-8。** 但背后是几个连锁反应。

## 根因分析

1. **`ConvertTo-Json` 输出的是 .NET 字符串（UTF-16）。** 这一步本身没问题，内存里是 Unicode。
2. **`Invoke-RestMethod` 的 `-Body` 接受字符串时，会调用 `[System.Text.Encoding]::Default.GetBytes()`**。在中文 Windows 上，`Default` 是 GBK（代码页 936），不是 UTF-8。
3. 即使显式设置了 `-ContentType "application/json; charset=utf-8"`，**PowerShell 仍可能先用 GBK 编码，再声明自己是 UTF-8**，造成内容实际是 GBK 但标头说 UTF-8 的矛盾。
4. 如果 API 严格按 UTF-8 解码，就会得到乱码；如果 API 尝试自动检测编码（很多现代框架默认用 UTF-8），也会失败。
5. 反过来，**接收响应**时 `Invoke-RestMethod` 默认也会用 `Default` 解码，除非 Content-Type 响应头明确指明了 charset。很多 API 只返回 `application/json`，不写 charset，导致响应里的中文在解析成对象前就已经坏了。

这里有一个隐秘的坑：`$OutputEncoding` 变量。许多教程建议修改它，但那影响的是外部进程（如 `ping`、`python`）的管道输出编码，对 `Invoke-RestMethod` 的行为**没有直接影响**。改了之后有些人看着好了，可能是脚本里还做了别的手脚。

## 可靠做法

### 方案一：直接发送 byte[]（推荐，兼容 PowerShell 5.1）

```powershell
$body = @{
    name = "张三"
    action = "查询订单"
} | ConvertTo-Json

$utf8Bytes = [System.Text.Encoding]::UTF8.GetBytes($body)

$response = Invoke-RestMethod -Uri "https://httpbin.org/post" `
    -Method Post `
    -Body $utf8Bytes `
    -ContentType "application/json; charset=utf-8"
```

这是最稳的方案。`-Body` 参数实际也接受 `byte[]`，此时 PowerShell 会直接放入请求体，不做任何编码转换。你完全控制了发送字节。同时，`ContentType` 标明 charset，让服务端和中间代理不会误判。

对于响应，如果仍然担心解码，可以改用 `Invoke-WebRequest` 获取原始字节再自行转字符串：

```powershell
$wr = Invoke-WebRequest -Uri "https://httpbin.org/get?name=张三" -UseBasicParsing
$rawBytes = $wr.RawContentStream.ToArray()  # 需要 .NET 处理
# 或者直接相信 Content-Type 的 charset，但最好还是自己解码
$text = [System.Text.Encoding]::UTF8.GetString($rawBytes)
```

### 方案二：切换到 PowerShell 7

PowerShell 7 修复了这个历史问题。`Invoke-RestMethod` 和 `Invoke-WebRequest` 在 7+ 中默认使用 UTF-8 编码处理请求和响应字符串，并且 `ConvertTo-Json` 默认不会进行额外的转义（`-EscapeHandling` 也更合理）。如果你的运维环境允许安装 PowerShell 7，升级是最省心的方式。

```powershell
# pwsh 7 下同样的代码通常无需额外处理
$body = @{ name = "张三" } | ConvertTo-Json
Invoke-RestMethod -Uri "..." -Body $body -ContentType "application/json; charset=utf-8"
```

### 方案三：使用 curl.exe 作为逃生出口

在无法安装新 PowerShell，且脚本上下文混乱时，直接调用系统自带的 `curl.exe`（注意不是 PowerShell 的 `curl` alias，需用 `curl.exe`）反而更可靠，因为它是二进制，按字节传输：

```powershell
$json = @{ name = "张三" } | ConvertTo-Json
$json | curl.exe -X POST "https://httpbin.org/post" -H "Content-Type: application/json; charset=utf-8" -d @-
```

此时管道传递的是字符串，但 `curl.exe` 会通过标准输入读取字节，PowerShell 将字符串通过管道传递给外部进程时，会使用 `$OutputEncoding` 编码。需要先设置：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
```

才能保证管道传给 `curl.exe` 的是 UTF-8。否则仍有坑。

## 踩坑清单

- **`$OutputEncoding` 只影响与外部程序的管道，不直接影响 `Invoke-RestMethod`。** 网上大量文章把它当万能药，会浪费很多时间。
- **`-ContentType` 如果不写 charset，服务端默认通常为 ISO-8859-1 或 UTF-8，取决于框架。** 总是明确 `charset=utf-8` 能消除歧义。
- **`ConvertTo-Json` 的 `-Depth` 默认仅为 2**，复杂对象会丢失层级，出现 `System.Object[]` 之类的字符串，但这不是编码问题，而是序列化限制。深层对象需指定 `-Depth 10` 或更大。
- **`Invoke-WebRequest` 的 `.Content` 属性已经是解码后的字符串**，但它依赖响应头指定的编码。如果响应头缺失 charset，PowerShell 5.1 会用 `Default`（GBK）解码，导致混乱。用 `.RawContentStream` 获取原始字节最安全。
- **文件读写**也会加重问题：用 `Out-File` 写 JSON 时，默认是 `Unicode` (UTF-16LE)，API 服务器不认识。应始终指定 `-Encoding utf8`。

## 可复用封装

在工程中，可以直接封装一个安全的调用函数，避免每次都重复：

```powershell
function Invoke-JsonApi {
    param(
        [string]$Uri,
        [hashtable]$Body,
        [string]$Method = 'Post'
    )
    $json = $Body | ConvertTo-Json -Depth 20
    $utf8Bytes = [System.Text.Encoding]::UTF8.GetBytes($json)
    return Invoke-RestMethod -Uri $Uri -Method $Method -Body $utf8Bytes -ContentType "application/json; charset=utf-8"
}
```

内部应用全部统一使用该函数，确保无论脚本在哪个 PowerShell 版本下运行，都不会出现中文打坏的问题。

## 总结

PowerShell 在 Windows 上的编码问题，本质是两种默认行为的冲突：.NET 默认的 ANSI 编码和现代 Web API 的 UTF-8 惯例。只要掌握“主动控制字节流”的原则，避免依赖脚本宿主隐式转换，就能从根源上杜绝乱码。

如果你在为 Agent 或自动化工作流编写 PowerShell 脚本，优先切换到 PowerShell 7；如果必须在 5.1 上运行，坚持使用 byte[] 发送，并显式声明 charset。这不是魔法，是工程里的确定性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/a08b4907d7ef7bc4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/c124ccb77f80d1fc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/0779d7262f9c8948.png)

