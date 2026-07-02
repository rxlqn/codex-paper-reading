# Privileged Information Distillation for Language Models

- Year: 2026
- Date: 2026-02-16
- Venue: arXiv / alphaXiv
- Authors: Emiliano Penaloza, Dheeraj Vattikonda, Nicolas Gontier, Alexandre Lacoste, Laurent Charlin, Massimo Caccia
- Links:
  - alphaXiv: https://www.alphaxiv.org/abs/2602.04942
  - arXiv: https://arxiv.org/abs/2602.04942
  - PDF: https://arxiv.org/pdf/2602.04942
- Local PDF: [../papers/2026_privileged-information-distillation-for-language-models.pdf](../papers/2026_privileged-information-distillation-for-language-models.pdf)
- Tags: agent, distillation, reinforcement-learning, privileged-information, tool-use, no-cot
- Status: reading
- Rating:
- One-line takeaway: 训练时让 teacher 看到 privileged information，再把能力迁移到测试时看不到这些信息的 student，可以在没有完整 CoT 的 agent 蒸馏场景里优于常规 SFT+RL 路线。

## Problem

长程 agent 任务里，训练时的 privileged information 可以显著提高成功率，但部署时 policy 往往不能依赖这些额外信息。论文关注的问题是：如何把训练时借助 privileged information 学到的能力迁移给测试时不含 privileged information 的 policy。

这个问题在 frontier agent 蒸馏里尤其明显：很多强模型只暴露最终动作或工具调用轨迹，不暴露完整 Chain-of-Thought。传统蒸馏依赖 reasoning trace，而这里能观察到的是 action trajectory。

## Core Idea

论文提出两个方法：

- `pi-Distill`: 一个共享参数的 teacher-student 框架。teacher 在训练时 conditioned on privileged information，student 不看 privileged information；二者联合优化，让 teacher 学会用 PI，同时让 student 学会复现 teacher 的高奖励行为。
- `OPSD`: On-Policy Self-Distillation。student 采样 on-policy trajectory，同时用 reverse KL penalty 约束 student 靠近 PI-conditioned teacher。

关键点不是简单先训练一个带 PI 的 teacher 再离线蒸馏，而是在共享参数和联合目标下减少 teacher/student 分布差距。

## Method

- 任务形式: 多轮、长程、工具调用 agent 环境。
- Privileged information 来源: 从 frontier model 的成功轨迹中派生，而不是假设能拿到完整 CoT。
- PI 类型:
  - Tool calls and arguments.
  - Tool calls only.
  - Self-generated hints.
- 主要训练目标:
  - teacher 目标: 最大化 reward，同时通过 KL 约束靠近 student。
  - student 目标: 在 teacher 采样出的高奖励轨迹上学习无 PI policy。
  - OPSD: student on-policy 采样，使用和 PI-conditioned teacher 的 reverse KL 作为 dense token-level signal。

## Experiments

- Benchmarks:
  - tau-Bench retail / airline.
  - Travel Planner.
  - GEM tool-use environments.
- Baselines:
  - SFT without CoT + RL.
  - SFT with CoT + RL.
  - 其他 self-distillation / PI 使用方式。
- Main results:
  - `pi-Distill` 在多个设置中优于标准 SFT+RL baseline。
  - 在部分条件下，`OPSD` 也有竞争力。
  - 对 OOD tool-use environments 有泛化收益。

## Strengths

- 问题定义非常贴近真实 agent 蒸馏：只能看到动作，看不到完整 CoT。
- 把 privileged information 的“训练时有、测试时无”这个矛盾写成了清晰的 teacher-student 目标。
- 对不同 PI 信息密度做了分析，有助于判断什么类型的轨迹信号值得收集。

## Weaknesses

- 方法依赖可构造有效 PI；如果轨迹本身质量差或 PI 与任务 reward 弱相关，收益可能下降。
- shared-parameter teacher-student 的优化稳定性和超参敏感性需要细读实验部分。
- 目前主要在 agent benchmark 上验证，迁移到其他类型任务还需要额外证据。

## Useful For

- 思考没有 CoT 的 frontier agent 蒸馏。
- 设计“训练时有额外证据/轨迹/答案，推理时不能给”的训练框架。
- 和 long-horizon agent RL、tool-use imitation learning、trajectory distillation 论文做对比。

## Questions

- PI 的最佳形式是否取决于 base model 已有能力？弱模型是否更需要高信息密度 PI？
- `pi-Distill` 和多 teacher distillation 能否结合？
- 如果 privileged information 是检索证据、环境状态或 verifier trace，而不是工具调用轨迹，目标函数是否需要改？
- KL penalty 控制 teacher-student gap 的机制是否可以被更稳定的 trust-region 或 curriculum 替代？

## Share Notes

适合的分享主线：

1. Frontier agent 蒸馏的现实限制：有 action，没有完整 CoT。
2. 训练时 privileged information 的价值和部署时不可用之间的矛盾。
3. `pi-Distill`: 共享参数 teacher-student 如何同时学会“用 PI”和“不用 PI”。
4. 和常规 SFT+RL / CoT distillation 的实验对比。
5. 对自己项目的启发：哪些中间轨迹、工具调用、检索证据可以作为训练时 PI？

