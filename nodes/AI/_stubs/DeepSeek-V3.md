---
name: DeepSeek-V3
field: AI
year: 2024
tags:
  - 大模型
  - MoE
  - MLA
  - MTP
  - FP8
  - DeepSeek-V2
  - DeepSeek-R1
  - H800
  - Transformer
desc: DeepSeek 2024-12 开源的 MoE 大模型，671B 总参 / 37B 激活，靠 MLA + MoE + FP8 把训练成本压到很低
layer: AI应用
params: 671B
---
DeepSeek-V3 是 DeepSeek 2024 年 12 月发布的开源 MoE 大模型：总参数 671B、每个 token 激活 37B，上下文 128K。它的意义在于用很低的训练成本做到了和当时闭源一线模型接近的水平。

## 为什么需要它
稠密模型每个 token 都要过全部参数，参数量一上去训练和推理成本线性往上涨。V3 用 MoE 让每个 token 只走一小部分专家，再配一组省显存、省通信的工程设计，把成本压下来——技术报告给的训练开销约 278.8 万 H800 GPU 小时。

## 怎么运作
- **MLA**（Multi-head Latent Attention，V2 引入）：把 K/V 压成低维 latent 再缓存，KV Cache 大幅缩小
- **DeepSeekMoE**：细粒度专家 + 共享专家
- **无辅助损失的负载均衡**：靠给每个专家加可调 bias 平衡路由，不再往 loss 里加惩罚项
- **MTP**（Multi-Token Prediction）：训练时多预测后面几个 token，增加训练信号
- **FP8 混合精度训练**，预训练约 14.8T token

## 容易搞混的
- **V3 里没有 Engram。** Engram 是 DeepSeek 2026 年 1 月论文里提出的另一个模块（哈希 n-gram 查表记忆），和 V3 是两回事。
- 和 DeepSeek-R1：R1（2025-01）是在 V3 基座上用强化学习训出来的推理模型。

## 我的理解
待补：……

## 线头
- 架构基础：Transformer、MoE
- 注意力：MLA，从 MHA → MQA → GQA 那条线下来
- 前代：DeepSeek-V2（2024，引入 MLA）
- 后继：DeepSeek-R1（2025，RL 推理）
- 硬件：H800，出口管制下的算力约束
- Engram：DeepSeek 2026 的条件记忆模块，不属于 V3

## 关系
