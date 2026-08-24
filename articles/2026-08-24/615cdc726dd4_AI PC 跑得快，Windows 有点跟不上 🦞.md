---
title: AI PC 跑得快，Windows 有点跟不上 🦞
feedId: 01M0SDHZZXRGAAXMNM2JT9TPV1
source: 36kr
publishedAt: 2026-08-24
---

过去两年，AI PC 的叙事一直由芯片厂商主导。从 Intel 的 Core Ultra 到 AMD 的 Ryzen AI 300，再到高通的骁龙 X 系列，NPU 算力从十几 TOPS 一路卷到 50 TOPS 上下。OEM 也把 AI 标签贴满键盘和宣传页。但最近 36kr 这条「Windows 是 AI PC 路上的绊脚石」的讨论，其实戳中了一个更靠后的问题：硬件在冲刺，操作系统却还没把赛道铺好。这篇文章不站队，只从技术栈、调度、生态和隐私四个角度拆一拆，Windows 到底卡在哪。

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/f27176ef14afeb59.png)

## 一、硬件已经冲在前面，系统却还没接住
![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/2122102aaac4980e.png)

可以先给几个数据。微软给 Copilot+ PC 设的门槛是 40 TOPS 以上 NPU 算力，目前主流平台已经越过这条线。比如 Intel Lunar Lake 标称 48 TOPS，AMD Ryzen AI 300 系列宣称 50 TOPS，高通骁龙 X Elite 也有 45 TOPS。账面差距不大，但用户拿到机器后，能明显感知 AI 的场景却很有限。

原因不在晶体管数量，而在软件暴露。NPU 作为新的计算单元，需要操作系统提供稳定的驱动模型、API 和调度策略。Windows 的 DirectML 对 NPU 的支持虽然已经推进，但覆盖算子和版本碎片严重；很多应用开发者为了兼容老设备，默认只走 CPU 或 GPU，NPU 成了宣传参数里的摆设。评测中常见的一个现象是：同一段 Whisper 本地转写，在某些机器上 NPU 并不比 GPU 快，甚至因为内存拷贝和算子回退更慢。

所以第一层矛盾是：硬件厂商把 NPU 当卖点，系统层却还没有让第三方应用便宜地、稳定地调用它。

## 二、Windows 的兼容性包袱，成了 AI 负载的隐性税
![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/9a909a7469a29a99.png)

Windows 最大的优点是兼容，最大的成本也是兼容。Win32、COM、注册表、服务、驱动模型这些历史资产，撑起了庞大的软件生态，但也带来后台负担。一个干净安装的 Windows 11，基础内存占用常常 4GB 起步；加上 Defender 实时扫描、Windows Update、Search Indexer、Telemetry 服务，系统和后台进程会在随机时间点争抢 CPU 和磁盘。

本地 AI 模型有一个特点：它喜欢“稳”。无论是 7B 量化大模型的常驻内存，还是实时降噪、背景替换这类持续推理，都对延迟和内存带宽敏感。系统后台的一次索引、一次更新扫描，可能就会造成推理抖动。很多人把 16GB 内存的机器买回来，装完浏览器、办公软件，再加载一个 4-6GB 的量化模型，物理内存立刻吃紧，系统开始频繁换页。此时 NPU 再省电，体验也会被内存瓶颈拉下来。

另一个隐性税来自 x86 与 Arm 的转译。Windows on Arm 通过 Prism 转译运行大量 x64 应用，虽然兼容性在进步，但转译层会消耗性能。如果 AI 应用没有原生 Arm 版本，NPU 的能效优势很可能被转译吃掉一部分。驱动更新也依赖 OEM，不同笔记本的 NPU 驱动版本、API 支持度可能不一样，这让开发者很难做一致优化。

## 三、调度与电源管理：NPU 还没被当成一等公民
![img3](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/ea3578b7121491ff.png)

Windows 的调度器传统上更熟悉 CPU 核心和 GPU 显存，对 NPU 这种低功耗、长时在线、突发型 AI 加速器还不够成熟。NPU 在任务管理器里甚至没有被清晰展示，更别说独立的 QoS 等级和电源策略。

这就带来几个实际后果：

1. 后台 AI 任务容易被节流。视频通话的背景模糊、系统级降噪，本应常驻 NPU，但 Windows 可能把它们压到低优先级，或者没有明确策略优先使用 NPU。
2. 前台大模型推理时，系统的 Defender、浏览器和会议软件一样在抢 CPU 和内存带宽。NPU 虽然空闲，但任务没有分过去。
3. 电源计划对 NPU 的暴露不足。用户选“最佳能效”或“最佳性能”时，系统很少能明确告诉用户 NPU 会如何参与。

一些测试里，同样做视频会议背景替换，NPU 利用率不足 20%，CPU/GPU 反而在忙，功耗也没有降下来。这说明硬件有 NPU，但操作系统没有把它当作一等公民去调度。

## 四、模型、框架与隐私：AI PC 的最后一公里
![img4](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/d9dcf9d2882b0a4d.png)

即使系统层把 NPU 调度好了，开发者和用户还要面对另外两个问题：框架碎片和信任成本。

开发框架方面，Windows 上的选择很多，ONNX Runtime、DirectML、OpenVINO、Qualcomm AI Stack、AMD Ryzen AI Software 都可以跑，但“多”不代表“好用”。不同框架对 NPU 算子覆盖不同，模型格式、量化方式、上下文长度支持也不统一。开发者如果不针对每类硬件做适配，就很难保证一致体验。相比之下，Apple 生态里 Core ML 的统一转换和部署路径，虽然也有局限，但对个人开发者更友好。

系统级 AI 功能也踩过坑。Windows Studio Effects 提升了摄像头和麦克风体验，但依赖特定 NPU 和驱动。Recall 截图索引功能更是因为隐私争议被迫调整，这说明系统级 AI 不能只讲功能，还要把数据流向讲清楚。本地推理本来应该是隐私优势，但 Windows 的账户体系、Copilot 云依赖、遥测策略叠加在一起，用户经常搞不清哪些数据留在本机、哪些被上传。企业用户一旦关闭 Recall，AI PC 的差异化功能又少了一块。

这种“既想用本地 AI，又离不开云服务”的模糊感，是 AI PC 普及的隐形阻力。

最后，三条冷思考：

1. 对普通用户：不要只盯着 NPU 的 TOPS 数字。下单前先确认你常用的软件是否真的支持 NPU 加速，否则买到的 AI PC 可能只是一台普通笔记本加了个新贴纸。
2. 对开发者：优先选择跨硬件、支持 NPU 的运行时，如 ONNX Runtime + DirectML，避免绑死单一厂商 API；部署时把模型内存占用和量化放在第一位，先跑顺，再跑快。
3. 对行业：AI PC 不是芯片单点突破，而是操作系统、驱动、模型、应用四层对齐。谁先把系统级体验做顺，谁才能定义下一代个人计算。
