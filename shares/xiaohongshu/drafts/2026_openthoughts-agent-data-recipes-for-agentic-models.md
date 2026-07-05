# 训练 Agent，真正难的是数据 Recipe

- Paper: OpenThoughts-Agent: Data Recipes for Agentic Models
- Note: [../../../notes/2026_openthoughts-agent-data-recipes-for-agentic-models.md](../../../notes/2026_openthoughts-agent-data-recipes-for-agentic-models.md)
- PDF: [../../../papers/2026_openthoughts-agent-data-recipes-for-agentic-models.pdf](../../../papers/2026_openthoughts-agent-data-recipes-for-agentic-models.pdf)
- Target audience: AI 硕博 / 大厂 AI 工程师
- Draft status: draft
- Suggested format: 7-9 张图 + 正文
- Keywords: agent training, data recipe, SFT, RL, tool use, open data

## 0. 一句话定位

这篇论文最值得讲的不是又出了一个 agent 模型，而是它把 agent 训练数据拆成了可 ablation 的 recipe：task 从哪里来、怎么混、怎么筛、谁来生成轨迹、哪些轨迹值得保留、RL 数据源怎么选。

## 1. 标题候选

- 训练 Agent，真正难的是数据 Recipe
- OpenThoughts-Agent：开放 Agent 训练数据到底该怎么做？
- Stronger Teacher 不一定更好：Agent 数据管线的 6 个关键变量

## 2. 开头 Hook

很多 agent 论文会告诉你模型在 SWE-Bench 或 Terminal-Bench 上涨了多少，但很少清楚回答一个更基础的问题：

一个能泛化到多种 agent 任务的训练集，到底应该怎么构造？

OpenThoughts-Agent 这篇论文的价值就在这里。它不是只给一个最终数据集，而是把 SFT 数据管线拆成多个阶段做 controlled ablation，然后再研究 RL 数据源选择。

## 3. 问题定义

- 输入: task description 和可交互环境。
- 输出: agent trajectory，包括多轮工具调用、观察、修改和最终结果。
- 训练目标: 让 Qwen3 系列模型在多种 agent benchmark 上泛化，而不是只刷单一 benchmark。
- 核心变量:
  - task source
  - source mixing
  - task filtering
  - teacher model
  - rollout filtering
  - RL data source

旧方法的问题：

- 很多开放数据集围绕单个 benchmark，泛化性难判断。
- 数据构造细节不透明，别人很难复现和改进。
- 只说“用了更强 teacher”不够，因为 teacher 生成的轨迹未必最适合 student 学。

## 4. 方法拆解

### 4.1 总体框架

论文可以拆成两条线：

1. SFT 线：六阶段 data pipeline ablation。
2. RL 线：在 8B scale 上比较不同 RL data sources。

SFT 数据的基本单位是：

```text
training example = (task description, agent trajectory)
```

这意味着数据质量不只取决于任务本身，也取决于 agent 是如何一步步行动、观察和修正的。

### 4.2 六阶段 SFT Pipeline

```text
task sourcing
-> task mixing
-> task augmentation
-> task filtering
-> teacher rollout generation
-> rollout filtering
```

几个重要发现：

- task source 是最大变量，不同来源对 SWE、terminal、OOD 的收益非常不均匀。
- Top-4 / Top-8 混合比只用 Top-1 更稳，因为单一来源容易过拟合某类 benchmark。
- 普通 task augmentation 不一定有效，不能把“LLM 改写”当作默认增益。
- 更强 teacher 不一定更好，benchmark 最强模型可能生成 student 不好学的轨迹。
- 保留至少 5 turns 的 agent traces 有帮助，而且在 matched token budget 下仍然有收益。

### 4.3 RL 数据源

RL 部分的重点不是新算法，而是 data source ablation。论文在 8B scale 上固定训练管线，只改变 RL 数据来源。

结论很工程化：RL source 的 clean verifier、难度、任务形态和行为迁移性，都会影响最终 agent 能力。

## 5. 关键实验

| Claim | Evidence | My read |
|---|---|---|
| 数据 recipe 能训练出强 open-data agent | OpenThinkerAgent-32B 七个 benchmark 平均 44.8%，高于 Nemotron-Terminal-32B 的 40.9% | headline 结果支撑 recipe 有效 |
| task source 是核心变量 | 95 个 task generation strategies 的 ablation 显示跨度很大 | 训练集来源比很多小 trick 更重要 |
| stronger teacher 不一定更好 | GPT-5.3-Codex benchmark 强，但 teacher 效果反而差 | agent distillation 要看轨迹可学习性 |
| 长轨迹过滤有用 | `min turns >= 5` 在 matched token budget 下仍提升 | agent 训练需要真实多轮解决过程 |
| RL source matters | 8B RL 数据源 ablation 差异超过 run-to-run noise | RL 不是只要有 verifier 就够 |

## 6. Know-how

- 设计 agent 数据集时，先列清楚 task source inventory，不要直接混成一个大 blob。
- task mix 要服务泛化，不要只按单 benchmark 得分排序。
- teacher model 的选择要看 trajectory style、可学习性、tool-use pattern，而不是只看 teacher 自己的榜单分。
- rollout filtering 要关注 multi-turn supervision，不只是过滤明显失败样本。
- RL 数据源要看 verifier 是否可靠、任务难度是否适中、行为是否能迁移到目标 benchmark。

## 7. Takeaways

面向研究：

- agent training data 应该被当成实验对象，而不是论文附录里的实现细节。
- data source 和 trajectory quality 可以比模型小改动更关键。
- SFT 和 RL 数据选择需要一起设计，否则 RL 可能只是修补一个不合适的 SFT 起点。

面向工程：

- 如果要做自己的 agent 训练，先搭建可重复的数据管线和评测 harness。
- 记录每条 trajectory 的来源、teacher、turn 数、timeout、verifier outcome，后续才有 ablation 空间。
- 不要默认“更强 teacher + 更多数据 = 更强 student”。

一句话总结：

> 这篇论文的核心启发是：训练 agent 的关键资产不是一个数据集名字，而是一套能被 ablation、扩展和复现的数据 recipe。

## 8. 局限和我的判断

- Base model 主要固定在 Qwen3 系列，recipe 与 base pretraining 的关系还没完全拆开。
- RL 主要在 8B scale 验证，不能直接推断到 32B。
- 复现成本不低，因为 sandbox、harness、verifier 和数据生成环境都是系统工程。
- 部分结论可能依赖具体 benchmark，比如 SWE / terminal tasks 对 trajectory length 的偏好。

## 9. 配图规划

| 图 | 内容 | 目的 |
|---|---|---|
| 1 | 封面：训练 Agent，真正难的是数据 Recipe | 建立主张 |
| 2 | 从单一数据集到 data recipe | 解释问题重构 |
| 3 | 六阶段 SFT pipeline | 方法总览 |
| 4 | task source / mix / filter 的变量表 | 展示可 ablation 的设计空间 |
| 5 | stronger teacher 不一定更好 | 强调反直觉发现 |
| 6 | long traces filtering | 解释为什么多轮轨迹重要 |
| 7 | SFT + RL 数据源关系 | 连接 RL 部分 |
| 8 | Know-how checklist | 给工程复用价值 |

## 10. 正文草稿

### 开头

这篇论文回答的是一个很实在的问题：如果我们想训练一个开放的 agentic model，训练数据到底应该怎么做？

很多工作会给出一个数据集或者一个 benchmark 分数，但 OpenThoughts-Agent 更像是把 agent 数据工程摊开来做实验。

### 方法

它把 SFT 数据管线拆成六个阶段：任务来源、任务混合、任务增强、任务过滤、teacher 轨迹生成、轨迹过滤。每个阶段都做 ablation，然后用这些实验结果组装最终的 100K agentic traces。

这比“收集更多数据”更细，因为 agent 数据不是普通问答。每条样本里都包含 task 和 trajectory，而 trajectory 的 teacher、turn 数、工具调用和失败模式都会影响 student 能学到什么。

### 实验

最终 `OpenThinkerAgent-32B` 在七个 agent benchmark 上平均 44.8%，超过已有强 open-data baseline。更有意思的是 ablation：task source 影响巨大，top source 混合更稳，强 teacher 不一定是好 teacher，保留更长 multi-turn traces 有帮助。

### 我的 takeaways

如果我要借鉴这篇论文，第一件事不是照搬它的数据源，而是照搬它的实验组织方式：把数据 recipe 的每个决策点都变成可比较变量。

### 结尾

对 agent 训练来说，数据不应该只是“我们用了多少条”。更重要的问题是：任务从哪里来、轨迹谁生成、失败怎么过滤、哪些行为能迁移。OpenThoughts-Agent 这篇论文的价值，就在于把这些问题变成了可讨论、可复现、可继续优化的 recipe。
