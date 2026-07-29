# Kimi K3 专题精读

Kimi K3 的技术报告横跨模型架构、预训练、agentic RL、环境合成、分布式系统与评测。把它当作一篇普通模型报告从头读到尾，很容易只记住“2.8T、104B active、1M context”。更有效的方式是把它拆成六个专题，每个专题回答一个独立问题。

## 阅读地图

| 顺序 | 专题 | 核心问题 | 报告章节 |
|---:|---|---|---|
| 1 | [架构](01-architecture.md) | K3 怎样沿 token、layer、channel 三个维度扩大信息流？ | §2，pp. 3-9 |
| 2 | [预训练与长上下文](02-pretraining-and-long-context.md) | 2.5x scaling efficiency 和 1M context 分别是怎样得到的？ | §3，pp. 10-11 |
| 3 | [Agentic Post-Training](03-agentic-post-training.md) | 3 个领域 × 3 档 effort 的专家如何训练并合并？ | §4.1，pp. 12-14 |
| 4 | [RL 环境](04-rl-environments.md) | 长程能力需要什么任务、harness、verifier 和持久状态？ | §4.2，pp. 14-17 |
| 5 | [基础设施与推理](05-infrastructure-and-serving.md) | 3T MoE、1M rollout 和 KDA serving 为什么必须协同设计？ | §5，pp. 18-25 |
| 6 | [评测与 SOTA](06-evaluation-and-sota.md) | “当前开放 SOTA”由哪些证据支持，又有哪些不可比因素？ | §6-7，pp. 26-33 |

## 三遍读法

### 第一遍：建立整机视角

先读摘要、Introduction、Figure 2 和 Conclusion。只回答三个问题：

- foundation scaling 和 test-time scaling 为什么要同时做？
- 1M context 在 K3 中是输入规格，还是训练/推理系统的工作尺度？
- K3 的能力提升能否被归因到单一模块？

### 第二遍：按研究兴趣选专题

- 做模型架构：精读专题 1 和 2。
- 做 agent training：精读专题 3 和 4，并与 [Agentic Model Training](../agentic-model-training.md) 对照。
- 做推理系统：精读专题 1 和 5。
- 做模型选型或行业分享：优先专题 6，避免把不同 harness 的数字直接横比。

### 第三遍：做可证伪检查

读每个专题末尾的“未回答问题”，把报告中的结论分成：

- 有公式或实现细节支撑；
- 有实验数字但缺少消融；
- 只有经验性描述；
- 需要独立第三方复现。

## 总判断

K3 最值得精读的不是“第一个开放 3T 模型”这个发布叙事，而是它把 frontier agent model 定义成一个跨层系统：

```text
大规模 multimodal base
  -> 多领域、多 effort RL experts
  -> on-policy consolidation
  -> persistent long-horizon rollout
  -> cache- and budget-aware serving
```

任何一层拿掉，另外几层都无法按报告描述的规模工作。这个共同设计思想，比单个 benchmark 第一更可复用。
