# 专题 4：RL 任务合成与长程 Agent 环境

## 核心判断

K3 把 agent RL environment 看成可组合的训练分布，而不是单一 benchmark wrapper。真正的训练对象包含：

```text
task + harness + tools + context policy + memory + sandbox state + verifier
```

因此，模型能力的泛化不仅取决于 task diversity，也取决于交互协议和环境状态是否多样。

## 1. Unified White-Box RL Environment

如果始终在一个固定 harness 中训练，模型可能过拟合：

- tool schema；
- system prompt；
- context management；
- memory protocol；
- interaction format；
- 某个框架的错误恢复习惯。

K3 把 harness 拆成可配置模块：

- tool interfaces；
- system prompts；
- context management strategies；
- skills；
- memories；
- subagents；
- 其他协议组件。

通过组合这些模块，环境可实例化 Kimi Code、Claude Code、Codex、OpenClaw、Hermes，也可生成新配置。不同 task group 在 RL 中动态使用不同组合。

这与 OpenThoughts-Agent 的 data recipe 互补：

- OpenThoughts 更强调 task source、teacher、trajectory filtering；
- K3 更强调在线 RL 时的 harness distribution 与环境可恢复性。

## 2. Knowledge-Graph-Guided Task Synthesis

任务合成从自演化的分层 knowledge graph 开始。

### Graph expansion

- 以粗粒度 seed nodes 开始；
- agent 对每个节点进行多次 web search；
- 新建节点前先探索现有 graph，复用等价或相关概念；
- 边从 coarse concept 指向 fine concept；
- 新节点继续分配给 agent 递归探索；
- 当概念足够 atomic 时停止分支。

### Material retrieval

系统从不同层级采样单个或相关节点，结合 ancestor context 构造 web query，抓取真实公开材料，再由 synthesis agent 按目标 task type 生成任务。

相比简单 topic list，graph 的优势是：

- 控制 coverage 和 granularity；
- 发现 underrepresented concepts；
- 可通过 node/edge 追踪任务来源；
- 相关节点组合能产生跨概念任务。

潜在风险：

- graph expansion agent 的偏差会递归放大；
- web availability 不等于真实工作频率；
- atomic concept 覆盖不等于 compositional task 覆盖；
- 合成任务可能可验证但不自然。

## 3. Verifiable Agentic Problems

K3 的代表性任务包括：

- 多步复杂搜索：规划、逐步取证、给出可验证答案；
- 专业知识工作：投行、数据分析、法律，在 sandbox 中操作领域工具并交付产物；
- 多步视觉推理：STEM、visual puzzle、chart understanding；
- 视觉工具循环：用 Python crop、zoom、transform、compute、verify，再把生成图像作为新 observation。

关键是 outcome 能被验证。长程任务若只有自然语言 judge，很容易把 reward 变成风格或篇幅偏好。

## 4. 专项环境

### Kernel Optimization

模型需要分析现有 kernel、运行 profiler、改写实现并对数值正确性和性能共同优化。它提供：

- 确定性的 correctness test；
- 可重复的 performance measurement；
- 长达数小时的真实优化循环；
- 高价值、低 reward ambiguity 的 coding RL。

### Personal Assistant

环境模拟长期持续的 assistant workflow。任务不是一次性问答，而是在事件流中维护约束和状态；单条轨迹可能包含数百到数千次 tool calls、累计数百万 context tokens。

它检验：

- memory 与当前 instruction 的冲突处理；
- 延迟事件；
- 多轮 follow-up；
- 长期状态一致性。

### Autonomous Execution Tasks

AET 训练模型在给定目标和可操作系统中自主探索、执行、验证和调整。相比固定步骤任务，它更接近黑盒系统复制和开放式工程交付。

### Web Development

通过代码、浏览器截图和交互验证形成 vision-in-the-loop。静态 code judge 不足以评价布局、视觉一致性和交互行为，因此环境需要渲染和视觉 feedback。

## 5. 为什么 sandbox 必须可恢复

Partial rollout 会把一条 trajectory 跨多个 RL iteration。只保存文本 transcript 不够，因为真实状态还包括：

- 文件系统；
- 进程；
- 安装依赖；
- 浏览器或服务状态；
- 中间构建产物；
- 外部工具产生的缓存。

K3 使用可恢复 microVM，把 model state 与 environment state 都视为 trajectory 的一部分。这个系统条件决定了算法能否真正训练数百步的 agent。

## 6. 推荐的环境分析框架

阅读任何 agent RL 工作时，可用以下表格检查：

| 维度 | 检查问题 |
|---|---|
| Task source | 来自真实任务、公开材料还是纯合成？ |
| Harness | 是否只训练单一工具协议？ |
| State | transcript 之外的环境状态如何保存？ |
| Verifier | outcome、过程还是 LLM judge？ |
| Horizon | 步数、token、wall time 分布怎样？ |
| Failure recovery | timeout、crash、stale policy 如何处理？ |
| Leakage | benchmark task 或实现是否进入合成图谱？ |
| Auditability | source、rubric、tool log、artifact 能否回放？ |

## 未回答问题

- 知识图谱和合成任务是否公开？如何排查 benchmark contamination？
- 各类 RL environment 的数据量与 sampling ratio 是多少？
- 动态 harness 配置的实际组合空间多大？
- 训练中的 skills、memory、subagent 模块是否与部署版本一致？
- 专业知识工作中的 verifier 是确定性规则、rubric judge 还是人工复核？
- microVM 恢复失败、外部网站变化和非确定性工具输出如何进入 reward？
