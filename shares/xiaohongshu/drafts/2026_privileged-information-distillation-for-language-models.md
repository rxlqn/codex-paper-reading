# 没有 CoT，怎么蒸馏 Frontier Agent？

- Paper: Privileged Information Distillation for Language Models
- Note: [../../../notes/2026_privileged-information-distillation-for-language-models.md](../../../notes/2026_privileged-information-distillation-for-language-models.md)
- PDF: [../../../papers/2026_privileged-information-distillation-for-language-models.pdf](../../../papers/2026_privileged-information-distillation-for-language-models.pdf)
- Target audience: AI 硕博 / 大厂 AI 工程师
- Draft status: draft
- Suggested format: 6-8 张图 + 正文
- Keywords: privileged information, agent distillation, RL, tool use, no CoT

## 0. 一句话定位

这篇论文讨论的是一个很现实的 agent 训练问题：frontier model 可能只给你 action trajectory，不给完整 CoT，那么还能不能把它的长程工具使用能力蒸馏给小模型？

## 1. 标题候选

- 没有 CoT，怎么蒸馏 Frontier Agent？
- 训练时有答案提示，推理时没有：LLM 如何迁移 privileged information？
- Agent 蒸馏的新问题：我们看得到动作，却看不到推理

## 2. 开头 Hook

很多 agent 蒸馏方法默认我们能拿到专家模型的 reasoning trace。但真实 API 或产品环境里，frontier model 往往只暴露工具调用和最终动作，不暴露完整 Chain-of-Thought。

这就产生了一个训练矛盾：

- 训练时，我们可以从成功 trajectory 里构造额外提示或 privileged information。
- 测试时，student policy 又不能依赖这些额外信息。

这篇论文的核心问题就是：如何让模型在训练时借助 PI 成功探索，同时把能力迁移到测试时无 PI 的 student。

## 3. 问题定义

- State: 当前多轮交互上下文 `s`。
- Action/output: 模型输出 `o`，包括 reasoning tokens 和 action tokens。
- Privileged information: 训练时可见的额外信息 `I`，例如 frontier trajectory 派生出的工具调用序列、工具名、hint。
- Teacher: `pi_T(o | s, I)`，训练时看得到 PI。
- Student: `pi_S(o | s)`，测试时看不到 PI。
- 目标: student 在没有 `I` 的情况下也能保留 teacher 借助 `I` 学到的高奖励行为。

关键难点：

- PI 会改变 teacher 的采样分布，teacher 越会用 PI，和 student 的分布差距可能越大。
- 只做先 teacher 后 distill 的两阶段流程，容易遇到 checkpoint 选择、off-policy、不稳定和算力浪费问题。

## 4. 方法拆解

### 4.1 总体框架

`pi-Distill` 的设计是共享参数 teacher-student：

1. 用同一个模型参数 `theta` 表示 teacher 和 student。
2. teacher 输入 `(s, I)`，负责借助 PI 产生高 reward trajectory。
3. student 只输入 `s`，负责学习 teacher 的行为。
4. teacher 和 student 联合训练，而不是先后训练。

### 4.2 关键公式

Teacher 目标可以理解为：

```text
J_teacher = E_{o ~ pi_T(.|s,I)}[R(o,s)]
            - beta * KL(pi_T(.|s,I) || stopgrad(pi_S(.|s)))
```

直觉：

- 第一项鼓励 teacher 借助 PI 找到高奖励行为。
- 第二项限制 teacher 不要跑到 student 完全学不到的分布。
- `stopgrad` 表示这里把 student 当作 reference，不直接更新它。

Student 目标可以理解为：

```text
J_student = learn from trajectories sampled by pi_T(.|s,I)
```

也就是 student 在无 PI 条件下拟合 teacher 的高奖励行为。

最终：

```text
J = alpha * J_teacher + (1 - alpha) * J_student
```

`alpha` 控制训练重心：偏 teacher 探索，还是偏 student 蒸馏。

### 4.3 OPSD

论文还提出 OPSD：student 自己 on-policy 采样，然后用和 PI-conditioned teacher 的 reverse KL 作为 dense signal。

这条路线更 on-policy，但效果取决于 teacher 提供的信息密度和 student 当前探索能力。

## 5. 关键实验

| Claim | Evidence | My read |
|---|---|---|
| action-only PI 可以有效蒸馏 agent | tau-Bench、Travel Planner、GEM tool-use 上有提升 | 对 API 不给 CoT 的现实场景很重要 |
| `pi-Distill` 优于常规 SFT+RL | 多个设置下超过 SFT w/o CoT + RL，部分超过 SFT w/ CoT + RL | 说明联合 teacher-student 不是简单工程 trick |
| PI 类型影响很大 | tool calls + args、tool calls only、self-generated hints 的效果不同 | PI 不是越多越好，还要看 utility 和 distribution shift |

## 6. Know-how

- 如果只有 action trajectory，也不要急着认为无法蒸馏；可以把轨迹转换成训练时 PI。
- PI 的设计要同时考虑信息量和 student-teacher gap。
- 对长程 agent，teacher 能成功探索很重要，但 student 能否继承同样重要。
- KL 约束不是装饰项，它在控制 teacher 不要变成 student 学不到的专家。
- 复现时要重点看 `alpha`、`beta`、PI 格式和 trajectory 质量。

## 7. Takeaways

面向研究：

- “训练时有、测试时无”的信息不一定浪费，可以通过 PI distillation 转化成能力。
- CoT 不可见会迫使蒸馏从 reasoning trace 转向 action trajectory。
- teacher-student 分布差距是 agent 蒸馏里的核心变量。

面向工程：

- 日志里的工具调用、检索结果、环境反馈都可能变成训练时 PI。
- 如果线上不能暴露 CoT，仍然可以围绕 action trace 设计学习框架。
- 要把“成功轨迹收集”和“student 可学习性”一起设计。

一句话总结：

> 这篇论文的价值在于，它把“没有 CoT 怎么蒸馏 agent”这个现实问题，重新写成了 privileged information transfer 问题。

## 8. 局限和我的判断

- PI 的构造质量决定上限，垃圾轨迹不会因为叫 privileged information 就有用。
- 方法主要在 agent/tool-use benchmark 上验证，是否迁移到一般推理任务还要谨慎。
- shared-parameter teacher-student 的稳定训练需要更多工程细节支撑。

## 9. 配图规划

| 图 | 内容 | 目的 |
|---|---|---|
| 1 | 封面：没有 CoT，怎么蒸馏 Frontier Agent？ | 建立问题 |
| 2 | 旧蒸馏 vs 现实 API：CoT 可见性差异 | 解释痛点 |
| 3 | PI teacher / no-PI student 框架 | 讲清方法 |
| 4 | `J_teacher` 和 `J_student` 公式 | 展示技术核心 |
| 5 | 三种 PI 类型对比 | 讲 know-how |
| 6 | 实验结论表 | 支撑主张 |
| 7 | Takeaways | 收束 |

## 10. 正文草稿

### 开头

如果你在做 agent 蒸馏，一个很现实的问题是：强模型真的会把它的完整推理过程给你吗？

很多时候不会。你能拿到的是工具调用、参数、环境返回、最终结果，而不是完整 Chain-of-Thought。这篇论文讨论的正是这个限制下的训练问题。

### 方法

论文把这个问题定义成 privileged information transfer：训练时可以给 teacher 看额外信息 `I`，但测试时 student 只能看普通上下文 `s`。

`pi-Distill` 的关键是让 teacher 和 student 共享参数并联合训练。teacher 负责借助 PI 找高奖励行为，student 负责在无 PI 条件下继承这些行为。

### 实验

作者在 tau-Bench、Travel Planner 和 GEM tool-use environments 上验证，发现 `pi-Distill` 在多个设置中优于常规 SFT+RL baseline。

### 我的 takeaways

对我来说，这篇论文最有启发的是：agent 蒸馏不一定要执着于 CoT。只要系统日志里有成功 trajectory，就可以思考怎么把它转换成训练时 PI。

### 结尾

这不是一篇单纯提出新 loss 的论文。它更重要的地方在于，把 frontier agent 蒸馏中的一个产品现实限制，变成了一个可以被优化的问题定义。

