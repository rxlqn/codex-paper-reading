# Human-Agent Collaboration

这个主题页用于整理 human-agent workflow、prompt engineering、agent artifact、代码协作、review readiness 和产品/设计决策相关材料。重点不是模型能力本身，而是人如何把不确定性、上下文和判断反馈给 agent。

## 当前材料

### Know Your Unknowns: HTML Artifacts for Surfacing Unknowns

- 笔记: [notes/2026_know-your-unknowns-html-effectiveness.md](../notes/2026_know-your-unknowns-html-effectiveness.md)
- 关注点: 用一次性 HTML artifact 在实现前、中、后暴露 unknowns，而不是用线性 Markdown 承载所有协作。
- 方法关键词: blindspot pass, domain vocabulary, design directions, throwaway mock, codebase-grounded brainstorm, architecture interview, semantics map, tweakable plan, implementation notes, buy-in doc, merge quiz.
- 和 agent 的关系: agent 不只是代码生成器，也可以生成临时交互界面，帮助人类做判断、表达偏好、确认语义和验证理解。

## 三阶段框架

| 阶段 | 推荐 artifact | 目标 |
|---|---|---|
| Pre-implementation | Blindspot pass | 先暴露代码库隐性坑、历史债和不可见边界 |
| Pre-implementation | Teach me my unknowns | 让用户先获得领域词汇和判断标准 |
| Pre-implementation | Four design directions / mock before you wire | 用可视化原型收集 taste、布局和交互偏好 |
| Pre-implementation | Brainstorm the intervention | 基于真实代码展开干预空间，而不是直接选一个解法 |
| Pre-implementation | Interview by blast radius | 优先澄清会改变架构、数据模型、权限或流程的问题 |
| Pre-implementation | Semantics map / tweakable plan | 在实现前确认参考语义和最该人类拍板的计划决策 |
| During implementation | Implementation notes | 保留计划偏差、代码发现、保守选择和后续人类判断 |
| Post-implementation | Buy-in doc / merge quiz | 提前回答 reviewer 问题，并验证 merge 前的系统理解 |

## 和现有主题的关系

- [Agentic Model Training](agentic-model-training.md): 关注训练更强 agent。
- [Agent Memory Systems](agent-memory-systems.md): 关注 agent 运行时如何保存、检索和维护状态。
- Human-Agent Collaboration: 关注人和 agent 的工作接口，尤其是如何把不确定性显性化。

## 后续可补充材料

- Agent coding workflow / Claude Code / Codex 使用经验。
- Prompt patterns for codebase investigation.
- PR review readiness and merge decision artifacts.
- Interactive artifacts for paper sharing and design exploration.
