---
title: 给 Agent 脚本加护城河：实现本地文件访问目录白名单
feedId: 31007
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景

在 OpenClaw 或者类似的 Agent 开发框架里，让大模型操作本地文件系统是一个常见的需求。MCP（Model Context Protocol）生态中，`@anthropic/mcp-server-filesystem` 之类的工具可以直接读写指定目录。

但一旦赋予 Agent 文件访问能力，“它能动哪些文件”立刻变成一个必须回答的安全问题。尤其在自动化流水线、本地开发辅助、运维脚本等场景中，我们往往只希望脚本停留在某个工作区目录，而不是一把梭访问整个 `$HOME` 甚至 `/`。

工程上最直接、最可控的方式就是：**在文件系统工具层加一层目录白名单校验，拒绝一切授权目录外的路径访问。**

## 问题拆解

假如你正在编写一个基于 MCP 的自定义文件工具（或者用 Plugin 方式挂在 OpenClaw 上），Agent 可能会调用：

- `readFile(path)`
- `writeFile(path, content)`
- `listDirectory(path)`
- `deleteFile(path)` 等

如果没有访问控制，恶意的或被污染的 prompt 可以传入 `/etc/passwd`、`~/.ssh/id_rsa`，甚至覆盖系统文件。我们需要确保：

1. 所有文件操作最终解析后的**真实绝对路径**必须在预设的目录白名单内。
2. 不能因符号链接、相对路径、路径遍历（`../`）等手段绕过。
3. 在 Windows 与 macOS/Linux 上行为一致。
4. 报错信息不能泄露服务器路径结构。

## 实现方案

以 Node.js 实现一个极简的可复用文件工具为例（使用 TypeScript，也适用于 `mcp-server` 的风格）。

### 1. 配置白名单

在环境变量或配置文件中定义允许访问的目录列表：

```env
ALLOWED_DIRS=/home/user/projects/safe-workspace,/tmp/agent-sandbox
```

加载后拆成数组并 resolve 为绝对路径：

```ts
const rawDirs = process.env.ALLOWED_DIRS?.split(',') || [];
const allowedDirs = rawDirs.map(dir => path.resolve(dir));
```

### 2. 路径校验核心函数

编写一个通用校验器，任何文件操作都先调用它：

```ts
import * as fs from 'fs/promises';
import * as path from 'path';

async function resolveSafePath(userInput: string): Promise<string> {
  // 1. 对用户输入做基础解析（支持 ~，环境变量可按需处理）
  const expanded = userInput.startsWith('~') 
    ? path.join(os.homedir(), userInput.slice(1)) 
    : userInput;

  // 2. 得到绝对路径
  const absolute = path.resolve(expanded);

  // 3. 跟随符号链接取得真实路径
  //    这一步是防绕过的关键
  let real: string;
  try {
    real = await fs.realpath(absolute);
  } catch {
    // 如果文件不存在，realpath 会失败，此时可以手动检查父目录是否允许
    // 或直接根据配置决定是否允许创建新文件
    // 这里为了安全，父目录必须已在白名单内
    const parent = path.dirname(absolute);
    const realParent = await fs.realpath(parent);
    if (!isAllowed(realParent)) throw new Error('Access denied');
    return absolute; // 允许创建，返回未跟随符号链接的绝对路径
  }

  // 4. 检查 real path 是否以任一 allowedDir 开头
  if (!isAllowed(real)) {
    throw new Error(`Access denied: path outside allowed directories`);
  }
  return real;
}

function isAllowed(resolvedPath: string): boolean {
  return allowedDirs.some(dir => {
    // 确保是子路径或完全相同，防止 /etc/abc 匹配 /etc/ab
    const relative = path.relative(dir, resolvedPath);
    return !relative.startsWith('..') && !path.isAbsolute(relative);
  });
}
```

### 3. 提供给 Agent 的工具函数

每个工具在上层调用 `resolveSafePath` 后再执行实际操作：

```ts
export async function readFile(userPath: string): Promise<string> {
  const safePath = await resolveSafePath(userPath);
  return fs.readFile(safePath, 'utf-8');
}

export async function writeFile(userPath: string, content: string): Promise<void> {
  const safePath = await resolveSafePath(userPath);
  await fs.writeFile(safePath, content, 'utf-8');
}
```

然后在 MCP Server 中把这些函数注册成工具，Agent 便能安全地访问文件。

## 踩坑记录

### 1. 符号链接绕过

如果只校验用户传入的路径，而没有解析 `realpath`，攻击者可以通过一个指向 `/etc/passwd` 的软链 `safe-dir/link` 绕过检查。必须先 `fs.realpath` 再比较白名单前缀。

但 `realpath` 要求路径存在，如果操作是创建新文件，路径不存在，`realpath` 会抛错。此时安全的做法是检查父目录的真实路径是否在白名单内，并且确保 `path.resolve` 的结果本身没有 `..` 突破父目录边界。

### 2. 路径遍历攻击

用户输入 `../../../etc/passwd`，经过 `path.resolve('/safe','../../../etc/passwd')` 会变成 `/etc/passwd`，而后继校验才会发现它不在白名单。所以顺序必须是：先 resolve 得到绝对路径，再 realpath，再 isAllowed。不要在 resolve 前就做前缀匹配。

### 3. Windows 兼容性

Windows 上 `path.resolve` 会处理盘符，`path.relative` 在跨盘符时可能返回绝对路径，需要额外判断。建议统一使用大小写敏感的路径比较，并在配置阶段将所有允许目录转为小写。另外，Windows 可能会用到 `\\?\` 长路径前缀，需要处理。

### 4. 错误信息泄露

不要在报错信息中暴露 `realpath` 的结果，只需给出模糊的 "Access denied" 或 "Invalid path"。可记录详细日志到服务端，但返回给 Agent 的信息要最少化。

## 可复用的工程建议

- **使用 `allowedDirectories` 模式**：如果你直接复用 `@anthropic/mcp-server-filesystem`，它本身就支持 `--directory` 参数配置白名单目录，内部已做了较完善的路径校验。自定义工具时参照它的思路，不要自己从头造轮子。
- **测试用例先行**：用单元测试覆盖符号链接绕过、`../` 穿透、绝对路径、Windows 盘符、不存在文件创建等场景。
- **结合容器化或沙盒环境**：白名单是应用层防线，更底层可以用 Docker/ Firecracker / macOS 沙盒限制进程能看到的文件系统范围，做到纵深防御。
- **只读与读写分离**：如果某些自动化流程只需要读取文件，直接屏蔽写、删操作，进一步减小风险。
- **审计日志**：记录每次文件操作的原始请求路径和解析后的真实路径，便于事后追溯。

## 总结

给 Agent 脚本加目录白名单，本质上是把“不应该访问的路径”在应用层硬编码拒绝。核心逻辑只有三步：解析、去符号链接、前缀匹配。但实现中到处是坑，尤其是在跨平台和创建新文件的场景下。遵循 `resolve + realpath + prefix check` 的黄金流程，并用充分的边界测试兜底，就可以在保持自动化便利性的同时，避免大部分常见的文件安全漏洞。

对于 OpenClaw 用户来说，不管是直接使用 MCP 文件服务器还是自己写 Plugin，都应该把目录白名单作为文件访问的默认配置项，没有白名单就不应该启动读写能力。这是工程上非常低廉但有效的安全护栏。

---

