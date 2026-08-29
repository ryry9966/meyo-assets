---
title: 英伟达129亿收购Hugging Face，贵在哪？🦞
feedId: 01M15KWJSPD2BPE9TS9H9F8SYZ
source: 36kr
publishedAt: 2026-08-29
---

一条消息在开发者圈刷屏：英伟达被曝拟以 129 亿美元收购 Hugging Face，而早期投资人 Kevin Durant 的账面回报达到 240 倍。是否最终落地仍待官方确认，但它把 Hugging Face 从开发者熟悉的“模型仓库”推到了 AI 基础设施并购的聚光灯下。比起 240 倍的财富故事，更值得拆解的是：为什么一个以开源模型托管为主的平台，会被估值到百亿美元级？

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/0708b67d80a2f82b.png)

## 一、Hugging Face 到底在“卖”什么

Hugging Face 并不直接生产大模型，它做的是模型与数据集的“分发和协作层”。大模型不是一个简单文件，而是 weights、config、tokenizer、推理脚本等组合起来的复杂工程包。Hugging Face 把这个工程包标准化了。

- 核心入口：Hub 上公开模型数量已超过百万，数据集数十万，Spaces 应用数十万；Transformers 库几乎成为多模态模型加载的默认入口。
- 开发者体验：一行 `pipeline` 或 `from_pretrained` 就能加载模型，不用自己处理权重分片、tokenizer 对齐和推理优化。
- 商业模式：免费托管为基础，私有仓库、Inference Endpoints、AutoTrain、Enterprise Hub、专用 GPU 实例等收费。
- 典型场景：团队微调 LLaMA 或 Mistral 后，可以直接通过 HF 部署成 HTTP 推理 API，不必自建 MLOps 平台。

## 二、英伟达为什么愿意出高价：从 GPU 到开发者入口

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/cd32079639289c45.png)

英伟达的核心壁垒从来不只是 GPU 芯片，而是以 CUDA 为核心的软件生态。过去十年它靠 CUDA、cuDNN、TensorRT 把硬件优势变成开发者依赖。收购 Hugging Face，本质上是买下“模型从哪来、在哪里跑、由谁部署”的入口。

1. 闭环形成：GPU/DPU 硬件、CUDA 软件栈、TensorRT/Triton/NIM、Hugging Face Hub、企业推理服务，形成一条完整链路。
2. 防御价值：如果模型分发层保持中立，开发者在 HF 上可能选择 AMD、Intel 或云厂商自研芯片；如果 HF 被英伟达控制，默认会更优先走 NVIDIA 加速栈。
3. 类比：微软 2018 年以 75 亿美元收购 GitHub，买的是开发者关系和云端入口；英伟达想复制这条路径。
4. 早期绑定：英伟达此前已参与 HF 融资，也把 TensorRT-LLM、NIM 等工具集成进 HF 生态。

## 三、129 亿美元到底贵不贵



从财务收入看，这个价格不便宜。Hugging Face 在 2023 年 D 轮融资时估值约 45 亿美元，129 亿对价意味着约 2.8~3 倍溢价。市场估测其年化收入在 1 亿美元上下，EV/Revenue 远高于传统软件公司。

但如果用平台公司逻辑算，指标会不一样：

- 开发者规模与模型数量：类似 GitHub 被收购时的 2800 万开发者逻辑，HF 的模型仓库规模和下载调用量已经是 AI 基础设施级资产。
- 战略溢价：对英伟达而言，这不仅是一宗普通 SaaS 收购，而是防止竞争对手从模型层绕开 CUDA 的防御性动作。
- 杜兰特 240 倍：具体入股成本未披露，但只有极早期进入才可能产生这种倍率。它属于风险投资幂律分布的尾部样本，不能代表普遍收益。

## 四、交易若落地，会怎样影响开发者与云厂商

- 开发者：短期可能获得更流畅的 NVIDIA 集成；长期需关注平台锁定和商务政策变化。
- 云厂商：AWS、GCP、Azure 都曾与 HF 合作；若平台被英伟达控制，可能加速自建模型花园，或扶持 ModelScope、Replicate、CivitAI 等替代品。
- 模型发布方：头部实验室可能把 HF 当作镜像而非唯一渠道；部分社区可能转向自托管或去中心化存储。
- 监管：百亿美元级科技并购可能触发反垄断审查，尤其在 AI 基础设施被视为战略领域时。

## 五、三条冷思考

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/a4e483709da21bb2.png)

1. 工具链不要绑定单一平台。镜像模型权重、保留 dataset 和推理配置的可迁移性，平台并购不会总是对开发者有利。
2. 关注“分发层”而不是只关注模型本身。模型会越来越商品化，谁掌握发现、托管、部署和评估入口，谁就掌握定价权。
3. 别被 240 倍叙事带偏。早期投资是幂律游戏，一个杜兰特 240 倍背后，可能有大量失败项目没有声音；普通人先保住本金与现金流。

英伟达与 Hugging Face 的交易如果落地，意味着 AI 竞争从“算力有多强”转向“算力 + 分发 + 开发者入口”的综合战争。模型仓库的估值不只是代码和参数，而是生态话语权。
