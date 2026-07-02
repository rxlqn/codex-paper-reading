# Agent Scaling 不只靠参数：这篇论文把 horizon 当成核心变量

- Paper: Scaling the Horizon, Not the Parameters: Reaching Trillion-Parameter Performance with a 35B Agent
- Note: [../../../notes/2026_scaling-the-horizon-not-the-parameters-agents-a1.md](../../../notes/2026_scaling-the-horizon-not-the-parameters-agents-a1.md)
- PDF: [../../../papers/2026_scaling-the-horizon-not-the-parameters-agents-a1.pdf](../../../papers/2026_scaling-the-horizon-not-the-parameters-agents-a1.pdf)
- Target audience: AI 硕博 / 大厂 AI 工程师
- Draft status: draft
- Suggested format: 6-9 张图 + 正文
- Keywords: long-horizon agent, MoE, multi-teacher distillation, tool use, scientific reasoning

## 0. 一句话定位

这篇论文的主张很直接：agent 能力的 scaling 不一定只靠更大的参数，也可以通过更长的知识-动作轨迹、更强的验证反馈和多域 teacher 蒸馏来扩大 horizon。

## 1. 标题候选

- Agent Scaling 不只靠参数：35B 模型如何逼近 1T 长程能力？
- Scaling the Horizon：长程 Agent 训练的新路线
- 不是更大模型，而是更长轨迹：Agents-A1 论文解读

## 2. 开头 Hook

我们通常把 LLM scaling 理解成参数、数据和算力的扩大。但 agent 任务里，模型不只是输出一段文本，而是要搜索、调用工具、观察反馈、修正计划、再行动。

这篇论文提出的视角是：对 agent 来说，真正需要 scale 的不只是 parameters，还有 horizon。

## 3. 问题定义

- 任务: 长程 agent 任务，包括搜索、科学推理、工程、工具调用、指令遵循。
- 输入: 用户目标、外部知识、工具环境、历史观察。
- 输出: 多步 action trajectory 和最终答案。
- 训练信号: knowledge-action trajectory、observations、execution results、verifier outcomes。
- 目标: 用 35B MoE agent 获得接近或超过 1T 模型的长程任务表现。

旧路线的问题：

- 只靠参数 scaling 成本高、复现难。
- 普通 SFT 很难覆盖长程任务里的行动、观察、验证、恢复过程。
- 单一领域优化很难统一搜索、科学、工程、工具调用等异构能力。

## 4. 方法拆解

### 4.1 总体框架

Agents-A1 的训练路线可以分成三层：

1. 建立 long-horizon knowledge-action infrastructure。
2. 通过 full-domain SFT 获得基础 agentic behavior。
3. 训练 domain-level teachers，再用 domain-routed on-policy distillation 合并到一个 student。

### 4.2 长程知识-动作基础设施

论文强调的不是孤立文本数据，而是把下面这些东西连成 trajectory：

- external knowledge
- actions
- observations
- verifier outcomes
- execution results

这使模型能学到“行动后观察，观察后修正”的过程。

### 4.3 多 teacher 蒸馏

核心思路：

```text
domain teachers -> domain-routed on-policy distillation -> unified student
```

每个 teacher 负责一个更专门的能力区域，student 通过 routing 和 on-policy distillation 统一吸收。

论文还引入 salient vocabulary alignment，用来提高不同 domain teacher 到 student 的知识迁移效率。

## 5. 关键实验

| Claim | Evidence | My read |
|---|---|---|
| 35B agent 可以达到很强长程表现 | SEAL-0、IFBench、HiPhO、FrontierScience-Olympiad、MolBench-Bind 等结果 | 支撑 horizon scaling 叙事 |
| 多域训练能统一异构能力 | 搜索、科学、工程、工具调用、指令遵循都进入 recipe | 价值在系统训练路线，而不只是单 benchmark |
| 不是只靠 SFT | 三阶段 recipe 包括 domain teachers 和 on-policy distillation | 关键在后两阶段能否稳定提升 |

## 6. Know-how

- 长程 agent 训练的核心资产不是单条问答，而是可验证的 action trajectory。
- trajectory 里必须保留 observation 和 verifier outcome，否则模型很难学到反馈闭环。
- 多域能力最好不要简单混在一起训练，需要 teacher / routing / alignment 机制。
- benchmark 对这种论文非常关键，要看是否真的测的是长程交互，而不是静态问答。
- 复现难点很可能不在模型结构，而在数据基础设施和 verifier 质量。

## 7. Takeaways

面向研究：

- Agent scaling 可以被定义为 horizon scaling，而不只是 parameter scaling。
- 长程 trajectory 是训练 agent planning / tool use / self-correction 的关键载体。
- 多 teacher distillation 是统一异构 agent 能力的一条实用路线。

面向工程：

- 如果要训练强 agent，先建设环境、工具、日志、verifier 和轨迹数据闭环。
- 不同任务域的 expert behavior 不一定要强行放进一个 teacher，可以先专门化再蒸馏统一。
- 评测必须覆盖长程和交互，否则会高估模型真实 agent 能力。

一句话总结：

> 这篇论文真正值得借鉴的不是“35B 打 1T”的标题，而是把 agent 训练重心从参数规模转向长程可验证轨迹。

## 8. 局限和我的判断

- 这条路线的数据和基础设施成本很高，小团队难以完整复现。
- 和 1T 模型的比较要细看评测设置，不能只看 headline numbers。
- salient vocabulary alignment 和 domain routing 的独立贡献需要看 ablation 后再下判断。
- 论文更像系统 recipe，很多效果可能来自工程整体，而非单一算法组件。

## 9. 配图规划

| 图 | 内容 | 目的 |
|---|---|---|
| 1 | 封面：Scaling horizon, not just parameters | 建立主张 |
| 2 | 参数 scaling vs horizon scaling | 讲清差异 |
| 3 | Knowledge-action trajectory | 解释数据资产 |
| 4 | 三阶段训练 recipe | 方法总览 |
| 5 | 多 teacher 到 student | 解释蒸馏 |
| 6 | Benchmark claim 表 | 支撑结果 |
| 7 | 复现 know-how | 输出实践价值 |
| 8 | Takeaways | 收束 |

## 10. 正文草稿

### 开头

这篇论文的标题很容易被读成“35B 打 1T”的模型榜单故事。但我觉得更值得关注的是它背后的训练视角：agent 的能力不只来自参数规模，也来自 horizon 的规模。

### 方法

论文中的 horizon 不是单纯上下文长度，而是模型在环境里连续行动、观察、验证、修正的过程。Agents-A1 通过 knowledge-action infrastructure 把这些过程组织成平均 45K tokens 的长程 trajectory。

训练上，它先做 full-domain SFT，再训练 domain-level teachers，最后通过 domain-routed on-policy distillation 把多个专门能力合并进一个 student。

### 实验

论文报告 Agents-A1 在多个长程 agent benchmark 上有很强表现，并在部分任务上超过 1T 级模型。阅读时我会重点看这些 benchmark 是否真正要求长程交互，以及 ablation 是否证明三阶段 recipe 的必要性。

### 我的 takeaways

如果要做 agent 系统，最该借鉴的是它的数据观：强 agent 不是只靠更多问答数据，而是需要 action、observation、verification 构成闭环。

### 结尾

这篇论文对工程实践的启发是：在考虑训练更强 agent 前，先问自己有没有可验证的长程环境、轨迹日志和 domain teacher 体系。没有这些，单纯堆 SFT 数据可能很难得到真正的长程能力。

