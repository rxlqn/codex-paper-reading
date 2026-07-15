# Single-Rollout Asynchronous Optimization for Agentic Reinforcement Learning

- Year: 2026
- Date: 2026-07-08
- Venue: arXiv / alphaXiv; under review
- Authors: Zhenyu Hou, Yujiang Li, Jie Tang, Yuxiao Dong
- Links:
  - alphaXiv: https://www.alphaxiv.org/abs/2607.07508
  - arXiv: https://arxiv.org/abs/2607.07508
  - PDF: https://arxiv.org/pdf/2607.07508
- Local PDF: [../papers/2026_single-rollout-asynchronous-optimization-for-agentic-reinforcement-learning.pdf](../papers/2026_single-rollout-asynchronous-optimization-for-agentic-reinforcement-learning.pdf)
- Tags: agent, reinforcement-learning, asynchronous-rl, single-rollout, value-model, off-policy, tool-use
- Status: reading
- Rating:
- One-line takeaway: SAO 用单 prompt 单 rollout 消除 GRPO 的组内等待，再用直接双边 token clipping、强化 critic 训练和跳过 observation 的 GAE 控制异步训练中的 policy lag 与高方差。

## Problem

长程 agent rollout 的长度差异很大。同步 RL 必须等一个 batch 中最慢的轨迹结束，容易让生成与训练资源闲置；异步 RL 虽然允许 rollout 一完成就进入训练，却会让轨迹来自不同版本的 rollout policy，产生更严重的 policy lag 和 off-policy drift。

GRPO 的 group-wise sampling 又引入了第二层矛盾：同一 prompt 的多条 rollout 必须凑齐后才能计算组内相对 advantage。长轨迹会拖住短轨迹，完成较早的样本在等待期间进一步变旧；而真实 online agent 场景通常只会为一个 prompt 返回一条轨迹，也不天然存在可用于组内归一化的多样本反馈。

论文因此关注的不是单纯提高异步系统吞吐，而是：如何在每个 prompt 只有一条 rollout、轨迹持续异步到达的条件下，仍然稳定且有效地训练长程 agent policy？

## Core Idea

论文提出 `Single-Rollout Asynchronous Optimization (SAO)`。每个 prompt 只生成一条 trajectory，完成后立即进入训练，不等待同组其他 rollout。这样更符合异步 actor-learner 系统和 online feedback 的数据到达方式，也减少了 group-wise waiting 带来的额外 staleness。

但 single-rollout 失去了 GRPO 的组内 baseline，梯度方差会明显增大，因此 SAO 重新引入 value model，并围绕异步 agent trajectory 做四类稳定化设计：

1. 用 rollout 时记录的 token log-probability 直接做 importance sampling，并屏蔽 trust region 外的 token；
2. critic 每次 policy update 前更新两次，使 value estimate 更快跟上 policy；
3. value model 冻结 attention，只更新 MoE projections，降低 critic 梯度不稳定；
4. token-level GAE 跳过环境 observation，从一个 model action 直接连接到下一个 model action。

## Method

### 1. Direct Double-Sided Importance Sampling

异步 rollout 可能跨越多个 policy version，精确维护所有历史 `old policy` 成本很高。SAO 直接把 rollout engine 保存的行为概率作为分母：

```text
r_t(theta) = exp(
  log pi_theta(a_t | s_t)
  - log pi_rollout(a_t | s_t)
)
```

再用双边 calibration function 限制 trust region：

```text
f(r_t) = r_t,  if 1 - epsilon_low < r_t < 1 + epsilon_high
         0,    otherwise
```

对应的 token-level objective 可以写成：

```text
L(theta) = E_t[
  f(r_t, epsilon_low, epsilon_high)
  * A_hat_t
  * log pi_theta(a_t | s_t)
]
```

它不是把极端 ratio 截到边界后继续训练，而是直接 mask 掉 trust region 外的 token。作者称其为 `Direct Double-Sided Importance Sampling (DIS)`。这个选择用可控的 off-policy bias 换取无需维护历史 policy ensemble 的实现复杂度。

### 2. Single-Rollout + Stronger Critic

- Sampling: 每个 prompt 只采样一条 rollout，完成即训练。
- Advantage: 使用 value-based critic，而不是 GRPO 的 group-relative baseline。
- Faster value update: 每次 actor update 对 critic 更新 `K=2` 次。
- Frozen attention: value model 冻结 attention 参数，只优化 MoE projections。
- Value pretraining: 扩大 value pretraining data，缓解 critic cold start。

这部分的核心不是“group size 设成 1”这么简单，而是要让 critic 足够快、足够稳地跟踪持续变化的异步 policy；否则 single-rollout 的高方差会直接破坏 actor update。

### 3. Skip-Observation Token-Level GAE

多轮 agent trajectory 的结构是：

```text
T = [action_0, observation_0, action_1, observation_1, ...]
```

observation 来自环境，不是 policy 生成的 token。标准 token-level GAE 如果沿 observation token 传播 advantage，会让 value model 对外部环境文本建模，并把这部分噪声混入 policy learning。

SAO 在 action 边界之间直接连接：

```text
delta = r_t + gamma * V(action_{i+1, first})
            - V(action_{i, last})

A_hat(action_{i, last}) = delta
  + gamma * lambda * A_hat(action_{i+1, first})
```

即跳过 observation token，只在模型自己生成的 action token 上估计 value 和 advantage。

## Experiments

### Setup

- Backbone: `Qwen3-30B-A3B-Thinking-2507`。
- Reasoning: 先在 GPT-OSS-120B 生成的 Tool-Integrated Reasoning 数据上做 3 epochs SFT，再初始化 policy 和 value model。
- Coding: 直接用 Qwen3-30B-A3B-Thinking-2507 训练。
- Batch size: 128；SAO group size 为 1。
- GRPO comparison: 每 batch 16 prompts，每个 prompt 8 rollouts，总样本数同为 128。
- Maximum length: 128K tokens。
- SWE-Bench scaffold: OpenHands，最多 300 interaction turns。
- Evaluation: AIME2025、BeyondAIME、HMMT Nov 2025、IMOAnswerBench、SWE-Bench Verified。

### Main Results

| Benchmark | SFT / Base | GRPO | SAO |
|---|---:|---:|---:|
| AIME2025 | 80.4 | 84.2 | **97.3** |
| BeyondAIME | 53.3 | 54.8 | **74.8** |
| HMMT Nov 2025 | 75.2 | 76.0 | **88.3** |
| IMOAnswerBench | 53.3 | 55.8 | **74.0** |
| SWE-Bench Verified | 23.0 | 27.0 | **29.8** |

标准 GRPO 在约 160 steps 后出现 collapse；加入 DIS 后可以稳定训练，但完整 SAO 在约 400 steps 后逐渐拉开差距，并稳定运行到约 1000 steps。

### Ablations

- `SAO`: AIME2025 97.3，BeyondAIME 74.8。
- critic 每次只更新一次: 95.0 / 69.8。
- value model 全参数更新: 90.6 / 74.5。
- Vanilla VAPO without DIS: 91.3 / 69.0。
- Running-mean baseline: 79.8 / 55.3。
- 400 steps 时，token-level value/action 的 89.8 / 66.8 高于 step-average 的 85.8 / 60.5 和 last-token 的 87.3 / 62.8。

### Online Learning Simulation

论文用不断切换写作风格偏好的模拟任务测试 non-stationary online learning。每个 prompt 只返回一条反馈，reward 由质量和风格是否匹配共同决定。SAO 在偏好切换后能较快改变 policy；使用最近 128 个 reward 均值作为 baseline 的方法适应更慢，因为历史窗口仍被旧 reward distribution 污染。

## Strengths

- 把异步 RL 的系统问题和优化问题连在一起：rollout 一完成就训练不仅影响吞吐，也改变了样本 staleness、baseline 和 advantage estimation。
- single-rollout 更贴近真实 online agent feedback，不依赖同一 prompt 同时获得多条轨迹。
- 专门处理了 agent action / environment observation 交错的结构，而不是把多轮轨迹当普通 token stream。
- ablation 覆盖 DIS、critic update frequency、frozen attention、running-mean baseline 和 action granularity，能较好定位稳定性来源。

## Weaknesses

- 论文以异步效率为主要动机，但实验集中在 accuracy 和 training stability，没有报告 wall-clock time、rollout throughput、GPU utilization 或 end-to-end training cost；因此不能从本文结果直接量化 SAO 的系统效率收益。
- SAO 需要额外 value model，并要求 rollout infrastructure 可靠保存 token-level behavior log-probabilities；它消除了 group barrier，但增加了 critic 的显存、训练和工程成本。
- 主要实验只覆盖 Qwen3-30B-A3B。结论是否适用于更小模型、dense model、短输出 RLHF 或 dense reward 环境仍不明确。
- GRPO 和 SAO 虽然 batch size 都是 128，但前者是 16 个 prompt 各 8 条 rollout，后者是 128 个 prompt 各 1 条 rollout；prompt diversity 与 advantage estimator 同时发生变化，不能把差异完全归因于异步等待。
- GLM-5.2 的部署只在摘要中作为应用结果出现，正文没有给出对应的大规模训练曲线、成本或对照实验。
- online learning 只是在受控的写作风格切换模拟中验证，不能直接外推到真实用户持续学习；真实部署还需要安全、监控和隐私机制。

## Useful For

- 设计长程 agent RL 系统时，区分三个边界：rollout 调度、off-policy correction、advantage estimation。
- 评估 GRPO 是否适合异步或 online 环境：如果每个 prompt 只有一次反馈，group-relative baseline 可能从结构上就不合适。
- 处理多轮 tool-use trajectory 时，把 model action token 和 environment observation token 分开，而不是默认对两者统一做 value propagation。
- 和 OpenThoughts-Agent 对照：后者研究 agent RL data source，SAO 研究 rollout 到达与 policy update 的优化协议。

## Questions

- 在严格 wall-clock / FLOPs matched 条件下，SAO 相比异步 GRPO 的收益还有多少？额外 critic 成本会抵消多少吞吐优势？
- 128 个单 rollout prompts 带来的任务多样性，和 single-rollout estimator 本身分别贡献了多少性能？
- frozen attention 是 Qwen3 MoE value model 的特定稳定化技巧，还是可迁移到 dense backbone 的一般规律？
- DIS 直接 mask 大 ratio token 会不会系统性丢掉稀有但重要的高收益行为？clip ratio 与最终探索能力如何权衡？
- skip-observation GAE 完全绕过 observation value，是否会损失“某类环境反馈预示后续成功”的状态信息？
- 在真实 online learning 中，如何同时处理 distribution shift、用户隐私、reward hacking 和灾难性遗忘？

## Share Notes

适合的分享主线：

1. 长程 agent RL 的 straggler 问题：同步 batch 为什么浪费，异步训练为什么又更 off-policy。
2. GRPO 的结构冲突：group-relative advantage 需要等齐一组，而 online feedback 往往每个 prompt 只有一条。
3. SAO 的主线：single rollout 立即训练 + DIS + 更快更稳的 critic + skip-observation GAE。
4. 实验：标准 GRPO collapse，DIS 先解决稳定性，完整 SAO 在约 400 steps 后继续拉开差距。
5. 工程判断：论文证明了优化有效性，但尚未给出完整的异步系统效率账本。
