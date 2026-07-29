# Paper Index

## 当前论文

| Status | Year | Title | Tags | Note | PDF |
|---|---:|---|---|---|---|
| reading | 2026 | OfficeQA Pro: An Enterprise Benchmark for End-to-End Grounded Reasoning | agent, benchmark, enterprise, grounded-reasoning, document-parsing, retrieval | [note](notes/2026_officeqa-pro-enterprise-benchmark-for-end-to-end-grounded-reasoning.md) | [arXiv PDF](https://arxiv.org/pdf/2603.08655) |
| reading | 2026 | Kimi K3: Open Frontier Intelligence | open-weight, MoE, long-context, multimodal, agentic-RL, systems | [note](notes/2026_kimi-k3-open-frontier-intelligence.md) · [专题精读](topics/kimi-k3/README.md) | [official PDF](https://github.com/MoonshotAI/Kimi-K3/blob/main/k3_tech_report.pdf) |
| reading | 2026 | LLM-as-a-Verifier: A General-Purpose Verification Framework | agent, verification, trajectory-reward-model, test-time-scaling, dense-reward | [note](notes/2026_llm-as-a-verifier-general-purpose-verification-framework.md) | [arXiv PDF](https://arxiv.org/pdf/2607.05391) |
| reading | 2026 | ScientistOne: Towards Human-Level Autonomous Research via Chain-of-Evidence | agent, autonomous-research, chain-of-evidence, provenance, verification | [note](notes/2026_scientistone-towards-human-level-autonomous-research-via-chain-of-evidence.md) | [arXiv PDF](https://arxiv.org/pdf/2605.26340) |
| reading | 2026 | OvisOCR2 Technical Report | document-parsing, OCR, multimodal, synthetic-data, on-policy-distillation | [note](notes/2026_ovisocr2-technical-report.md) | [arXiv PDF](https://arxiv.org/pdf/2607.13639) |
| reading | 2026 | Single-Rollout Asynchronous Optimization for Agentic Reinforcement Learning | agent, asynchronous-RL, single-rollout, value-model, off-policy | [note](notes/2026_single-rollout-asynchronous-optimization-for-agentic-reinforcement-learning.md) | [arXiv PDF](https://arxiv.org/pdf/2607.07508) |
| reading | 2026 | Are We Ready For An Agent-Native Memory System? | agent, memory, survey, benchmark, data-management | [note](notes/2026_are-we-ready-for-an-agent-native-memory-system.md) | [arXiv PDF](https://arxiv.org/pdf/2606.24775) |
| reading | 2026 | OpenThoughts-Agent: Data Recipes for Agentic Models | agent, data-curation, SFT, RL, tool-use | [note](notes/2026_openthoughts-agent-data-recipes-for-agentic-models.md) | [arXiv PDF](https://arxiv.org/pdf/2606.24855) |
| reading | 2026 | Privileged Information Distillation for Language Models | agent, distillation, RL, tool-use, privileged-information | [note](notes/2026_privileged-information-distillation-for-language-models.md) | [arXiv PDF](https://arxiv.org/pdf/2602.04942) |
| reading | 2026 | Scaling the Horizon, Not the Parameters: Reaching Trillion-Parameter Performance with a 35B Agent | agent, long-horizon, distillation, MoE, tool-use | [note](notes/2026_scaling-the-horizon-not-the-parameters-agents-a1.md) | [arXiv PDF](https://arxiv.org/pdf/2606.30616) |

## 方法论 / 工作流

| Status | Year | Title | Tags | Note | Source |
|---|---:|---|---|---|---|
| read | 2026 | Harvey LAB: Filesystem-First Benchmark for Realistic Legal Agent Work | legal-agent, agent-benchmark, professional-work, rubric-evaluation, sandbox | [note](notes/2026_harvey-lab-legal-agent-benchmark.md) | [GitHub](https://github.com/harveyai/harvey-labs) |
| read | 2026 | SearchOS: Structured Multi-Agent Search as Schema Completion | agentic-search, multi-agent, deep-research, evidence-grounding, orchestration | [note](notes/2026_searchos-structured-multi-agent-search-system.md) | [GitHub](https://github.com/antins-labs/SearchOS) |
| read | 2026 | Know Your Unknowns: HTML Artifacts for Surfacing Unknowns | human-agent-collaboration, agent-workflow, HTML-artifacts, prompt-engineering | [note](notes/2026_know-your-unknowns-html-effectiveness.md) | [web](https://thariqs.github.io/html-effectiveness/unknowns/) |

## 主题索引

- [Kimi K3 专题精读](topics/kimi-k3/README.md)
- [Agentic Model Training](topics/agentic-model-training.md)
- [Agent Memory Systems](topics/agent-memory-systems.md)
- [Agentic Search Systems](topics/agentic-search-systems.md)
- [Human-Agent Collaboration](topics/human-agent-collaboration.md)

## Skills

- [know-your-unknowns](skills/know-your-unknowns/SKILL.md): 面向 Codex / Claude 的三阶段 unknowns 暴露工作流。

## 待读问题

- K3 的 2.5x scaling efficiency 中，架构、数据、训练 recipe 各自贡献多少？百万上下文的收益又有多少来自 context management 而非 raw window？
- Agent memory 和普通 RAG 的边界在哪里？哪些能力必须作为持久、可更新的系统层来设计？
- Memory representation / extraction / retrieval / maintenance 哪一层最容易成为长程 agent 的瓶颈？
- 训练时给模型更多信息，和测试时不提供这些信息之间，能力到底如何迁移？
- 对 agent 来说，应该优先扩大模型参数，还是扩大可训练的长轨迹和工具交互过程？
- agent 训练数据里，task source、teacher quality、trajectory length 哪个才是最关键的瓶颈？
- 异步 agent RL 中，single-rollout 消除 group barrier 的收益能否覆盖额外 critic 的计算与显存成本？
- 多 teacher / privileged information / on-policy distillation 这些路线之间的共同抽象是什么？
- 什么时候应该让 agent 输出长期 Markdown 笔记，什么时候应该先生成一次性的 HTML 决策 artifact？
- Deep research 系统的主要瓶颈究竟是搜索策略、schema 设计、证据抽取，还是对 search state 的持久化与调度？
- Coverage Map 适合宽表和枚举任务；面对无法预先定义实体 / 属性的开放探索，应该怎样动态演化 schema？
- 自主研究系统的 claim provenance 应该只做来源链接，还是需要进一步保存版本、冲突、推导过程和可重放的执行环境？
- 专业工作 Agent 的 benchmark 应该如何同时校准 rubric 长度、all-pass 严格度、LLM judge 偏差与 synthetic-to-real distribution shift？
- 在固定 inference budget 下，应该把 compute 优先分给候选生成、评分粒度、重复验证、criteria decomposition，还是更大的 pivot set？
- 企业文档 Agent 应该如何显式建模统计口径、发布时间和后续修订关系，避免把第一个合理数值误当作最终证据？
- Parser 能否把 OCR / table topology 的不确定性传给 Agent，并触发有选择的原图复核，而不是输出一个看似确定但可能错误的文本现实？
