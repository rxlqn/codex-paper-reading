# Scaling the Horizon, Not the Parameters: Reaching Trillion-Parameter Performance with a 35B Agent

- Year: 2026
- Date: 2026-06-29
- Venue: arXiv / alphaXiv
- Authors: Lei Bai, Zongsheng Cao, Yang Chen, Zhiyao Cui, Shangheng Du, Yue Fan, Shiyang Feng, Zijie Guo, Haonan He, Liang He, Xiaohan He, Shuyue Hu, Yusong Hu, Songtao Huang, Yichen Jiang, Hao Li, Xin Li, Dahua Lin, Weihao Lin, Fenghua Ling, Dongrui Liu, Zhuo Liu, Runmin Ma, Chunjiang Mu, Haoyang Peng, Tianshuo Peng, Jinxin Shi, Luohe Shi, Boyuan Sun, Zelin Tan, Shengji Tang, Qianyi Wang, Yiming Wu, Yi Xie, Xiangchao Yan, Jingqi Ye, Peng Ye, Fangchen Yu, Jiakang Yuan, Bihao Zhan, Bo Zhang, Chen Zhang, Shufei Zhang, Shuaiyu Zhang, Wenlong Zhang, Yiqun Zhang, Junpeng Zhao, Zhijie Zhong, Bowen Zhou, Yuhao Zhou
- Links:
  - alphaXiv: https://www.alphaxiv.org/abs/2606.30616
  - arXiv: https://arxiv.org/abs/2606.30616
  - PDF: https://arxiv.org/pdf/2606.30616
- Local PDF: [../papers/2026_scaling-the-horizon-not-the-parameters-agents-a1.pdf](../papers/2026_scaling-the-horizon-not-the-parameters-agents-a1.pdf)
- Tags: agent, long-horizon, MoE, distillation, tool-use, scientific-reasoning
- Status: reading
- Rating:
- One-line takeaway: 论文主张 agent scaling 不一定只靠扩大参数，也可以通过长程知识-动作轨迹、多域 teacher 和 on-policy distillation，把 35B MoE agent 训练到接近或超过 1T 模型的长程任务表现。

## Problem

现实 agent 任务往往是长程的：模型需要搜索信息、调用工具、观察结果、验证中间步骤，并持续修正策略。单纯扩大参数能提升能力，但成本高且难以复现。论文提出另一条路线：扩大 agent 的 horizon，把知识获取、动作执行、观察解释、验证反馈这些中间过程显式化，并转化成可训练信号。

## Core Idea

论文提出 `Agents-A1`，一个 35B Mixture-of-Experts agentic model。核心不是只做更大的模型，而是构建长程 knowledge-action infrastructure，生成平均长度约 45K tokens 的 agentic trajectories，并通过三阶段 recipe 训练：

1. full-domain supervised fine-tuning，让 base model 对齐广泛 agentic behaviors；
2. domain-level teacher training，让不同领域 teacher 学到专门能力；
3. multi-teacher domain-routed on-policy distillation，加上 salient vocabulary alignment，把六类异构能力统一到一个可部署 student model。

## Method

- Infrastructure:
  - 连接 external knowledge、actions、observations、verifier outcomes。
  - 支持长程 trajectory 生成和反馈闭环。
- Training recipe:
  - Full-domain SFT: 覆盖搜索、科学研究、工程、工具调用、指令遵循等多域数据。
  - Domain teachers: 针对不同领域训练 specialized teachers。
  - Domain-routed OPD: 根据 domain routing 让 student 从多个 teacher 中吸收能力。
  - Salient vocabulary alignment: 提高跨领域知识迁移效率。
- Model:
  - 35B MoE agentic model。

## Experiments

- Benchmarks mentioned:
  - SEAL-0.
  - IFBench.
  - HiPhO.
  - FrontierScience-Olympiad.
  - MolBench-Bind.
  - SciCode.
  - HLE.
  - BrowseComp.
- Main claim:
  - Agents-A1 在多个长程 agent benchmark 上达到强结果。
  - 与 Kimi-K2.6、DeepSeek-V4-pro 等 1T 级模型相比，在部分 benchmark 上领先，在其他 benchmark 上保持竞争力。

## Strengths

- 把“agent 能力来自长程交互过程”这件事工程化为 infrastructure + trajectory + verifier + distillation。
- 对多域 agent 能力的统一训练给出完整 recipe，不只是单 benchmark 优化。
- 对“参数 scaling 之外的 agent scaling”给出清晰叙事，适合做论文分享。

## Weaknesses

- 训练基础设施和数据管线很重，复现成本可能高。
- 多域 teacher、routing、salient vocabulary alignment 的具体贡献需要结合 ablation 细读。
- 和 1T 模型的比较需要关注评测设置、模型版本、采样参数和 benchmark 是否完全公平。

## Useful For

- 设计长程 agent 数据生产和训练系统。
- 对比 parameter scaling 与 horizon scaling。
- 思考如何把搜索、工具调用、科学推理、工程任务放进同一个 agent 训练 recipe。
- 和 privileged-information distillation 论文一起讨论：训练时额外轨迹/teacher/环境反馈如何迁移到部署模型。

## Questions

- 长程 trajectory 的平均 45K tokens 是否主要提升规划能力，还是提升了工具/检索/验证的格式遵循？
- domain-routed distillation 是否需要显式 domain label？如果线上任务 domain 混合，该如何 routing？
- salient vocabulary alignment 的收益来自知识迁移，还是来自减少 teacher/student 输出空间错配？
- infrastructure 中 verifier outcome 的质量对最终模型有多敏感？

## Share Notes

适合的分享主线：

1. 传统 scaling 叙事：扩大参数。
2. 这篇论文的反向主张：扩大 horizon，而不是只扩大参数。
3. 长程 knowledge-action infrastructure 如何把 agent 过程变成训练数据。
4. 三阶段 recipe：full-domain SFT -> domain teachers -> multi-teacher OPD。
5. 讨论：这条路线对自己的 agent 系统有什么借鉴，哪些部分最难复现？

