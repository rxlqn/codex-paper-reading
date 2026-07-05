# Agentic Model Training

这个主题页用于整理和 agent 训练相关的论文，尤其关注长程任务、工具使用、蒸馏、强化学习、privileged information 和多 teacher 训练。

## 当前论文

### Privileged Information Distillation for Language Models

- 笔记: [notes/2026_privileged-information-distillation-for-language-models.md](../notes/2026_privileged-information-distillation-for-language-models.md)
- 关注点: 训练时使用 privileged information，测试时让 student 在没有 privileged information 的条件下执行。
- 方法关键词: pi-Distill, OPSD, shared-parameter teacher-student, action-only distillation, RL.
- 和 agent 的关系: 目标场景是多轮、长程、工具调用环境；尤其针对 frontier agent 不暴露完整 CoT，只能观察 action trajectory 的情况。

### Scaling the Horizon, Not the Parameters

- 笔记: [notes/2026_scaling-the-horizon-not-the-parameters-agents-a1.md](../notes/2026_scaling-the-horizon-not-the-parameters-agents-a1.md)
- 关注点: 不只扩大参数量，而是扩大 agent 的长程交互轨迹和异构能力训练。
- 方法关键词: Agents-A1, 35B MoE, long-horizon knowledge-action infrastructure, domain teachers, domain-routed on-policy distillation.
- 和 agent 的关系: 试图把搜索、科学推理、工程、工具调用、指令遵循等异构能力统一到一个可部署 agent 模型里。

### OpenThoughts-Agent

- 笔记: [notes/2026_openthoughts-agent-data-recipes-for-agentic-models.md](../notes/2026_openthoughts-agent-data-recipes-for-agentic-models.md)
- 关注点: 开放 agentic model training data recipe，尤其是 task source、mixing、filtering、teacher model、trajectory filtering 和 RL source selection。
- 方法关键词: OpenThoughts-Agent-v2, OpenThinkerAgent-32B, six-stage SFT pipeline, controlled ablation, GLM-4.7-AWQ teacher, min-turns filtering, RLOO.
- 和 agent 的关系: 把训练强 agent 的关键从“单个 benchmark 训练集”转向“可系统 ablation 的 task + trajectory 数据管线”。

## 初步对比

| 维度 | Privileged Information Distillation | Scaling the Horizon | OpenThoughts-Agent |
|---|---|---|---|
| 核心问题 | 如何把训练时额外信息迁移到测试时无额外信息的 student | 如何用长程轨迹和多域能力训练 35B agent，逼近 1T 模型表现 | 如何系统化设计开放 agent SFT/RL 数据 recipe |
| 训练信号 | Frontier trajectory 派生的 privileged information | 长程 knowledge-action trajectory、verifier outcome、多域 teacher | task descriptions、agent trajectories、teacher rollouts、RL verifier rewards |
| 蒸馏/训练结构 | 同参数 teacher-student，teacher 有 PI，student 无 PI | 多 teacher 到统一 student，domain-routed on-policy distillation | 六阶段 SFT ablation + 8B RL data-source study |
| 适合分享的主线 | “没有 CoT 时还能不能有效蒸馏 agent？” | “Agent scaling 的关键可能不是参数，而是 horizon” | “训练 agent，数据 recipe 比单个数据集名字更重要” |

## 后续可补充论文

- Tool-use RL / GRPO for agents.
- Long-horizon agent benchmarks.
- Multi-teacher distillation.
- Chain-of-thought distillation and CoT hiding policy对训练的影响。
