---
title: Agent 文件访问护栏：给自动化脚本加本地目录白名单
feedId: 30334
source: 综合讨论
publishedAt: 2026-07-25
---

## 背景

在 OpenClaw、Agent 和 MCP 生态里，让自动化脚本读写文件几乎是一个“刚需”——生成报告、转换数据、持久化上下文，都绕不开磁盘操作。问题在于：一旦把文件系统的能力暴露给 Agent，就等于打开了一扇后门。一次 prompt 注入、一次上下文污染，都可能让 Agent 尝试写入 `~/.ssh/authorized_keys`，或者把 `/etc/passwd` 打包带走。而默认的 Tool 实现（比如基于 Node.js 的 `fs.readFile` / `fs.writeFile`）通常不会替你检查路径边界。

与其赌模型“不会做坏事”，不如提前设置一道轻量的工程护栏：**本地目录白名单**。

## 问题抽象

我们希望 Agent 通过 MCP Tool 调用文件操作时，**只能访问预先声明的目录列表**，例如 `/app/data` 和 `/tmp/agent-workspace`。对于任何试图读取或写入白名单之外路径的请求，一律拒绝并记录告警。

这个需求简单明确，但落地时会遇到几个工程细节：

- 相对路径怎么处理？
- 符号链接会不会绕过检查？
- 文件还不存在时，如何判断它“将会”落在哪个目录？
- 并发场景下会不会有 TOCTOU 问题？

下面是一个在 MCP Server 里可复用的实现模式。

## 实现思路与关键代码

假设使用 Node.js 构建 MCP Server，我们定义一个 `safePath` 函数，作为所有文件工具的前置审查层。

```js
const path = require('path');
const fs = require('fs');

const ALLOWED_DIRS = (process.env.AGENT_ALLOWED_DIRS || '/app/data,/tmp/agent-workspace')
  .split(',')
  .map(d => path.resolve(d.trim()));

function safePath(userPath) {
  // 1. 解析成绝对路径
  const resolved = path.resolve(userPath);
  // 2. 基础校验，拒绝包含空字符的路径
  if (resolved.indexOf('\0') !== -1) {
    throw new Error('Invalid path: null byte found');
  }

  let realPath;
  try {
    // 3. 解析符号链接，拿到真实路径
    realPath = fs.realpathSync(resolved);
  } catch (err) {
    // 文件尚不存在：对父目录解析真实路径，再拼回文件名
    const parent = path.dirname(resolved);
    const parentReal = fs.realpathSync(parent);
    realPath = path.join(parentReal, path.basename(resolved));
  }

  // 4. 前缀白名单检查
  const allowed = ALLOWED_DIRS.some(dir => {
    const rel = path.relative(dir, realPath);
    // 安全的相对路径应当不以 '..' 开头
    return rel !== '' && !rel.startsWith('..') && !path.isAbsolute(rel);
  });

  if (!allowed) {
    // 记录日志（注意只暴露安全信息给日志，避免信息泄漏回传 Agent）
    console.warn(`[guard] blocked access to: ${realPath}`);
    throw new Error(`Access denied: outside of allowed directories`);
  }

  return realPath;
}
```

随后，在 MCP Tool 定义里用 `safePath` 包裹原始的读写操作：

```js
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  if (name === 'read_file') {
    const safeAbsPath = safePath(args.path);
    const content = await fs.promises.readFile(safeAbsPath, 'utf-8');
    return { content: [{ type: 'text', text: content }] };
  }
  // write_file 同理
});
```

## 踩坑记录

- **符号链接绕过**：如果不调用 `realpathSync`，攻击者可以通过在合法目录内创建一个指向 `/etc` 的符号链接，轻易越权。必须对已存在文件解析真实路径；对未存在文件则解析父目录的真实路径，再做拼接。
- **TOCTOU 问题**：检查与真正 `open` 之间存在时间窗口，文件系统可能被篡改。对于非高安全场景可接受；若要求更高，建议结合 `openat2` 的 `RESOLVE_NO_SYMLINKS` 标志（Linux 5.6+），或者直接将 Agent 进程放入容器/隔离文件系统。
- **Windows 分隔符**：`path.relative` 在 Windows 上可能返回反斜杠，检查 `startsWith('..')` 时要先统一成正斜杠或直接用 `path.parse` 的方式。示例代码已用平台原生方法，基本安全。
- **文件不存在的父目录**：如果连父目录都不合法，`realpathSync(parent)` 会抛错，应该直接用 try/catch 转化为拒绝。按默认拒绝原则处理。
- **向上级目录穿越 `../`**：`path.resolve` 可以消除掉大部分穿越，但为了可靠性，白名单检查仍然用 `path.relative` 并判断是否以 `..` 开头。这是双重保险。

## 可复用建议

1. **收敛为独立模块**：将 `safePath` 和对应的读写函数封装成一个小工具库，所有需要文件能力的 MCP Server 或插件统一引用，避免重复实现与疏漏。
2. **白名单集中配置**：通过环境变量或配置文件注入，启动时加载并 `resolve` 成绝对路径。支持动态热加载时，注意多线程读写，可简单地加读写锁或重启服务器生效。
3. **增强可观测性**：对每次拒绝操作记录请求的 Tool 名、原始路径（脱敏后）和时间戳，并接入告警。短时间内大量拒绝可能是恶意探测。
4. **纵深防御**：目录白名单只是一道应用层护栏，建议配合 Docker/容器限制、AppArmor/SELinux 等系统级措施，形成多层防御。
5. **测试用例**：用路径穿越 payload（`../../etc/passwd`）、符号链接跳转、空字节注入等场景做自动化测试，保证每次变更护栏不被破坏。

## 总结

Agent 文件访问的目录白名单，本质上是一个无侵入的中间件，它不改变模型行为，但能有效收缩爆炸半径。实现成本极低，却能避免大量低级但后果严重的安全失误。在生产环境里，每一次“先检查，再打开”都能让系统更健壮——这比事后靠审计日志亡羊补牢可靠得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/ca7b74d3d7a8760b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/2060c3aaa00f3ba4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/8527303ad1c6dcb6.png)

