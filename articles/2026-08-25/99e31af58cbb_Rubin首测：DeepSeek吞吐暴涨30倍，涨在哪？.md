---
title: Rubin首测：DeepSeek吞吐暴涨30倍，涨在哪？
feedId: 01M0VFP26CC8FMN225PAGF40BV
source: 36kr
publishedAt: 2026-08-25
---

这几天，英伟达下一代加速平台 Vera Rubin 的首测数据在技术圈刷屏。最抓眼球的数字是：在 DeepSeek 这类大规模 MoE 模型上，推理吞吐相比上一代提升了约 30 倍。很多人第一反应是“单卡算力翻倍也没这么夸张”。于是讨论焦点变成：这 30 倍到底涨在哪里，是硬核架构红利，还是特定测试口径下的数字游戏？这篇文章把链路拆开看。

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/d3c9abc7a8e59cc2.png)

## 1. Vera Rubin 不是一颗 GPU，而是一个平台
![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/29f7185138e887b4.png)

- 延续 Hopper、Blackwell 的命名传统，Vera Rubin 取自暗物质研究先驱，代表英伟达下一代数据中心加速平台。
- 平台由 Vera CPU、Rubin GPU、HBM4 高带宽内存、新一代 NVLink 互联和更高速的节点间网络共同组成。
- 相较于 Blackwell，Rubin 的显存带宽、互联带宽和低精度推理密度都有跨代式提升，同时单节点功耗与封装密度也明显提高。
- 需要先建立一个认知：首测里的“30 倍”通常是节点或集群在特定负载下的吞吐对比，不是单颗芯片 FLOPS 直接乘 30。

## 2. 推理吞吐不是单卡算力，而是一张管线图
![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/a662e1ad76f7f711.png)

- 简化公式：Throughput ≈ min(算力, 显存带宽, 互联带宽) × 利用率 × batch/并发。
- LLM 推理分为 prefill 和 decode：decode 逐 token 生成，主要瓶颈在显存带宽和 KV cache，而不是纯 FLOPS。
- DeepSeek 的 MoE 结构让情况更明显：总参数大、单 token 激活参数小，推理时大量时间花在从显存取专家权重、读写 KV cache，以及跨卡通信上。
- 因此 30 倍收益通常来自几个点同时改善：HBM4 提升显存带宽和容量，让 KV cache 与专家权重更容易驻留；新一代 NVLink 降低 All-to-All 通信等待；FP4/FP8 推理支持提高有效带宽。
- 举例来说，如果旧平台单张 GPU 因 KV cache 容量限制只能同时处理 20 条长上下文序列，Rubin 同级别产品若 HBM 容量提升到 2 倍以上，再加上带宽翻倍和 FP4 推理，批量可能扩大到 50~60 条。批量扩大 × 单 token 耗时下降，很容易乘出数量级收益。这里的关键是，单用户感知的到首字时间或单 token 速度变化可能远没这么夸张。

## 3. DeepSeek 为什么刚好站在增益点上
![img3](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/2c2f757f70c6606b.png)

- DeepSeek 是稀疏 MoE 模型，单 token 计算量相对小，但显存与通信要求高，这正好被 Rubin 的硬件长板覆盖。
- 推理并发越高，吞吐提升越明显：单用户延迟可能只降低一部分，但同样硬件能同时服务更多请求，因此“吞吐”增长远大于“单请求速度”。
- 旧平台上，MoE 的专家路由常因为 All-to-All 通信导致 GPU 空等；高互联带宽能明显减少这种空等，尤其在专家并行度较高时。
- DeepSeek 使用的 MLA 等注意力优化会压缩 KV cache 占用，这与 Rubin 的高显存带宽形成叠加效应，让长上下文场景更容易扩大并发。
- 软件栈同样关键：TensorRT-LLM 对 MoE、FP4、KV cache 管理等新 kernel 优化，会放大硬件本身的收益。

## 4. 从首测到生产，还有几条现实边界
![img4](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/f33e6063c71728ca.png)

- 供电与散热：HBM4 与大面积 die 让单节点功耗继续上涨，液冷、电源冗余和机柜密度都要提前升级。
- 软件适配成本：新架构的 CUDA 生态成熟需要时间，DeepSeek 的高效推理并非开箱即得，需要针对算子、通信库和调度策略做调优。
- 成本口径容易混淆：只比峰值吞吐会忽略总体拥有成本。对业务来说，单位 token 成本才是更真实的指标。
- 集群扩展不是线性：单节点 30 倍不等于整柜/整集群 30 倍，网络收敛比、节点间互联和故障恢复都会吃掉一部分收益。
- 另一个容易忽视的点是软件栈成熟度。新硬件刚发布时，真实业务的算子覆盖、内存分配器、通信 collectives 都需要重新调优。首测数据通常在相对干净的 benchmark 配置下获得，距离开箱即用还有一段路。

## 5. 三条冷思考，供参考

- 看测试口径：先确认是单卡、单节点还是整机柜；是 FP8、FP4 还是混合精度；是 prefill、decode 还是混合负载。
- 关注单位 token 成本：对大多数团队，硬件升级的价值应该是“每百万 token 成本下降”，而不是峰值吞吐数字本身。
- 软件先于硬件补课：在等待新平台量产时，优化 MoE 路由、KV cache 压缩、批处理策略，往往能先拿到一笔可观的性价比提升。
