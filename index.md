# Paper Index

## 当前论文

| Status | Year | Title | Tags | Note | PDF |
|---|---:|---|---|---|---|
| reading | 2026 | Are We Ready For An Agent-Native Memory System? | agent, memory, survey, benchmark, data-management | [note](notes/2026_are-we-ready-for-an-agent-native-memory-system.md) | [arXiv PDF](https://arxiv.org/pdf/2606.24775) |
| reading | 2026 | Privileged Information Distillation for Language Models | agent, distillation, RL, tool-use, privileged-information | [note](notes/2026_privileged-information-distillation-for-language-models.md) | [arXiv PDF](https://arxiv.org/pdf/2602.04942) |
| reading | 2026 | Scaling the Horizon, Not the Parameters: Reaching Trillion-Parameter Performance with a 35B Agent | agent, long-horizon, distillation, MoE, tool-use | [note](notes/2026_scaling-the-horizon-not-the-parameters-agents-a1.md) | [arXiv PDF](https://arxiv.org/pdf/2606.30616) |

## 方法论 / 工作流

| Status | Year | Title | Tags | Note | Source |
|---|---:|---|---|---|---|
| read | 2026 | Know Your Unknowns: HTML Artifacts for Surfacing Unknowns | human-agent-collaboration, agent-workflow, HTML-artifacts, prompt-engineering | [note](notes/2026_know-your-unknowns-html-effectiveness.md) | [web](https://thariqs.github.io/html-effectiveness/unknowns/) |

## 主题索引

- [Agentic Model Training](topics/agentic-model-training.md)
- [Agent Memory Systems](topics/agent-memory-systems.md)
- [Human-Agent Collaboration](topics/human-agent-collaboration.md)

## Skills

- [know-your-unknowns](skills/know-your-unknowns/SKILL.md): 面向 Codex / Claude 的三阶段 unknowns 暴露工作流。

## 待读问题

- Agent memory 和普通 RAG 的边界在哪里？哪些能力必须作为持久、可更新的系统层来设计？
- Memory representation / extraction / retrieval / maintenance 哪一层最容易成为长程 agent 的瓶颈？
- 训练时给模型更多信息，和测试时不提供这些信息之间，能力到底如何迁移？
- 对 agent 来说，应该优先扩大模型参数，还是扩大可训练的长轨迹和工具交互过程？
- 多 teacher / privileged information / on-policy distillation 这些路线之间的共同抽象是什么？
- 什么时候应该让 agent 输出长期 Markdown 笔记，什么时候应该先生成一次性的 HTML 决策 artifact？
