# Agent Memory Systems

这个主题页用于整理 agent long-term memory、memory system、RAG 与 context engineering 边界、动态更新、长程稳定性、survey 和系统评测相关论文。

## 当前论文

### Are We Ready For An Agent-Native Memory System?

- 笔记: [notes/2026_are-we-ready-for-an-agent-native-memory-system.md](../notes/2026_are-we-ready-for-an-agent-native-memory-system.md)
- 关注点: survey + benchmark 视角下，把 agent memory 定义成持久、可更新、可维护的数据管理系统，而不是普通 RAG 组件。
- 方法关键词: representation and storage, extraction, retrieval and routing, maintenance, dynamic update, long-horizon stability.
- 和 agent 的关系: agent 的长期能力不仅依赖训练和工具调用，也依赖 memory layer 是否能正确保存、更新、检索和维护历史信息。

## 初步框架

| 维度 | 关注问题 | 常见失败模式 |
|---|---|---|
| Representation / Storage | memory object 如何表示，存在哪类后端里 | 表示过平，无法表达实体关系和时间信息 |
| Extraction | 原始交互如何转成 memory | 过度抽取丢细节，schema 太弱导致事实不稳定 |
| Retrieval / Routing | query 来时如何找相关 memory | 只靠 embedding similarity，忽略时间、结构和任务意图 |
| Maintenance | 如何处理冲突、版本、遗忘和合并 | append-only 返回旧事实，global rewrite 成本高 |

## 和现有主题的关系

- [Agentic Model Training](agentic-model-training.md): 关注如何训练更强 agent。
- Agent Memory Systems: 关注 agent 部署和运行时如何长期管理状态。

两者可以连起来看：训练侧需要长程 trajectory 和 verifier，运行侧需要可靠 memory lifecycle。

## 后续可补充论文

- MemGPT / Letta.
- Mem0.
- Zep / temporal knowledge graph memory.
- A-MEM and hybrid memory.
- LongMemEval / LoCoMo / memory benchmark papers.
